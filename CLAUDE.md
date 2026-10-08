# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**mod_tile** is a high-performance tile serving system with two components:

- **mod_tile**: Apache 2 HTTP module that serves map tiles
- **renderd**: Daemon that renders tiles using Mapnik

## Build System

CMake is the primary build system; Autotools still works but is deprecated (`configure.ac` reads the version from `CMakeLists.txt`). The project requires C99 and C++14 (C++17 for Mapnik 4+).

### Ubuntu dependencies

```sh
sudo apt --no-install-recommends --yes install \
  apache2 apache2-dev cmake curl g++ gcc git \
  libcairo2-dev libcurl4-openssl-dev libglib2.0-dev \
  libiniparser-dev libmapnik-dev libmemcached-dev librados-dev
```

### CMake Presets (quickest)

```sh
cmake --preset dev && cmake --build --preset dev && ctest --preset dev   # Debug + tests + warnings
cmake --preset asan && cmake --build --preset asan && ctest --preset asan  # ASan + UBSan
cmake --preset tsan && cmake --build --preset tsan && ctest --preset tsan  # ThreadSanitizer (unit tests only)
```

Presets build into `build/<preset>/`. `config.h` is generated in the build tree (`<build>/includes/config.h`), so multiple build directories can coexist.

### CMake Build (manual)

```sh
export CMAKE_BUILD_PARALLEL_LEVEL=$(nproc)

cmake -B /tmp/mod_tile_build -S . \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_LOCALSTATEDIR=/var \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DCMAKE_INSTALL_RUNSTATEDIR=/run \
  -DCMAKE_INSTALL_SYSCONFDIR=/etc \
  -DENABLE_TESTS=ON

cmake --build /tmp/mod_tile_build

# Run all tests
cd /tmp/mod_tile_build && ctest

# Run a single test by name
cd /tmp/mod_tile_build && ctest -R <test_name> --output-on-failure

# Install
sudo cmake --install /tmp/mod_tile_build --strip
```

### Autotools Build

```sh
./autogen.sh
./configure
make
sudo make install
```

### Key CMake Options

| Option                                       | Default | Description                                                                                            |
| -------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------ |
| `ENABLE_TESTS`                               | OFF     | Build test suite (Catch2)                                                                              |
| `USE_CAIRO`                                  | ON      | Cairo composite backend                                                                                |
| `USE_CURL`                                   | ON      | HTTP proxy storage backend                                                                             |
| `USE_MEMCACHED`                              | ON      | Memcached storage backend                                                                              |
| `USE_RADOS`                                  | ON      | Ceph RADOS storage backend                                                                             |
| `MALLOC_LIB`                                 | libc    | Memory allocator: libc/jemalloc/mimalloc/tcmalloc                                                      |
| `ENABLE_WARNINGS`                            | OFF     | `-Wall -Wextra` (unused-parameter suppressed)                                                          |
| `ENABLE_WERROR`                              | OFF     | `-Werror` (with `ENABLE_WARNINGS`)                                                                     |
| `ENABLE_SANITIZERS`                          | ""      | e.g. `address,undefined` or `thread`                                                                   |
| `TILE_LOAD_DIRECTORY` / `TILE_LOAD_FILENAME` | auto    | Override the distro auto-detection of where the Apache `tile.load` goes (for packaging / cross builds) |

Shared sources are compiled once into internal OBJECT libraries in `src/CMakeLists.txt` (`common_objs`, `config_objs`, `protocol_objs`, `submit_queue_objs`, `store_objs`, `store_file_utils_objs`, `renderd_core_objs`) and reused by every target. `renderd.c` is the exception: `gen_tile_test` recompiles it with `MAIN_ALREADY_DEFINED`.

### Linting / Static Analysis

CI runs `flawfinder` for security scanning (`.github/workflows/flawfinder-analysis.yml`), a lint workflow (`.github/workflows/lint.yml`: astyle, cmakelint, prettier) and `.github/workflows/sanitizers-and-warnings.yml`:

- **Warnings** — GCC and Clang with `ENABLE_WARNINGS` + `ENABLE_WERROR`; the code base builds warning-free, keep it that way. `g_logger()` is declared with `G_GNUC_PRINTF`, so format/argument mismatches are compile errors.
- **ASan + UBSan** — full suite including the Apache/renderd integration tests. `tests/CMakeLists.txt` puts the ASan runtime in `LD_PRELOAD` so the system `httpd` can load the instrumented `mod_tile.so`. Services write sanitizer reports to `build/tests/logs`, which the job also checks.
- **TSan** — unit test executables only (TSan cannot be preloaded into an uninstrumented `httpd`). `tests/tsan.supp` suppresses a lock-order report inside GDAL.

## Architecture

### Communication Flow

```
HTTP Client → mod_tile (Apache module)
                  ├── Storage backends (tile cache hit) → Response
                  └── Unix socket → renderd daemon
                                        └── Request queue (5 priority levels)
                                              └── Thread pool → Mapnik → Metatile storage
```

### Key Components

**`src/mod_tile.c`** — Apache module: request handling, cache expiry heuristics, delay pool rate limiting, statistics. Config structures: `tile_config_rec` (per-directory), `tile_server_conf` (server-wide).

**`src/renderd.c`** — Rendering daemon main loop; manages worker threads and Unix socket listener.

**`src/gen_tile.cpp`** — Mapnik tile generation. Called by renderd worker threads.

**`src/request_queue.c`** — Priority queue with hash-indexed deduplication. 5 levels: Normal, Priority, Low, Bulk, Dirty.

**`src/renderd_config.c`** — Parses `renderd.conf` (INI format) into `renderd_config` / `xmlconfigitem` structs.

**Storage backends** (pluggable via `includes/store.h` function-pointer interface):

- `store_file.c` — filesystem (default), stores 8×8 metatiles
- `store_memcached.c` — Memcached
- `store_rados.c` — Ceph RADOS
- `store_ro_http_proxy.c` — HTTP proxy (read-only)
- `store_ro_composite.c` — composite read-only (requires Cairo)
- `store_null.c` — no-op

### Protocol

mod_tile and renderd communicate over a Unix socket (default: `/run/renderd/renderd.sock`, TCP fallback: `localhost:7654`) using the protocol defined in `includes/protocol.h`. Protocol version is v3. Commands: `cmdRender`, `cmdDirty`, `cmdRenderPrio`, `cmdRenderLow`, `cmdRenderBulk`, `cmdDone`, `cmdNotDone`.

### Metatile Format

Tiles are stored in 8×8 metatile bundles (`METATILE = 8`) in a hashed directory structure. File format: `"META"` magic header + index of per-tile offsets/sizes. `includes/metatile.h` defines the layout; `src/metatile.cpp` is the C++ wrapper.

### Important Constants (`includes/render_config.h`, `includes/mod_tile.h`)

- `MAX_ZOOM = 20`
- `HASHIDX_SIZE = 2213` (request deduplication hash)
- Default tile directory: `/var/cache/renderd/tiles`
- `MAX_LOAD_OLD = 16`, `MAX_LOAD_MISSING = 50` — re-render thresholds

## Tests

Tests use **Catch2** (v3.16.0 amalgamated build, vendored in `tests/catch/`; see its README to update) and live in `tests/`. Matchers use the v3 names, e.g. `Catch::Matchers::ContainsSubstring`. The main suites:

- `gen_tile_test.cpp` — Mapnik rendering pipeline (largest suite)
- `renderd_config_test.cpp` — configuration parsing
- `renderd_test.cpp` — daemon core
- `render_expired_test.cpp`, `render_list_test.cpp`, `render_old_test.cpp` — utility programs
- `render_speedtest_test.cpp` — performance tool

Test infrastructure uses `tests/httpd.conf.in` and `tests/renderd.conf.in` templates to spin up live Apache + renderd for integration tests. `tests/tiles.sha256sum` holds expected checksums for tile output validation.

## Known Issues / Technical Debt

- **`src/request_queue.c:request_queue_close`** — queued render items are not freed on shutdown (items in all five priority lists leak). The TODO comment is present in the source. Safe in practice because renderd only shuts down at process exit, but should be fixed for clean valgrind runs.
- **`src/renderd.c` / `src/mod_tile.c` — `bzero` usage** — several files use the deprecated `bzero()` instead of `memset(..., 0, ...)`. Functionally equivalent on Linux but not strictly portable.
- **`src/gen_tile.cpp:render_thread` — startup `strndup`/`malloc` leaks** — `output_format`, `xmlfile`, `xmlname` (`strndup`), `prj` (`malloc`), and `store` (`init_storage_backend`) are allocated once per thread at startup and never freed. The render thread runs in an infinite loop and never exits, so these do not accumulate in practice.
- **`src/store_ro_composite.c` — `connection_string_secondary` (strdup) not freed on late error paths** — after the `strdup` on the secondary connection string, several subsequent error paths (store_primary init failure, store_secondary init failure) free it, but if `store_secondary` init succeeds and a later step fails the pointer may still leak depending on the code path. Low risk: composite storage is rarely used and init failures abort the process.
- **`src/renderd.c` TCP listener binds the wildcard address** — it listens on all addresses (dual-stack IPv6, falling back to IPv4 when IPv6 is unavailable) regardless of `iphostname`, which clients use to connect. Binding to `iphostname` would be a behaviour change for existing configs.
- **Process-lifetime allocations** — LeakSanitizer reports small one-shot leaks at exit in `init_storage_*` and the `render_*` utilities' `main()`; harmless but noisy. Run the asan preset with `detect_leaks=0` (the test preset does this).

## Parameterized rendering cache

When `parameterize_style` is configured (e.g. for multilingual maps), `render()` in `src/gen_tile.cpp` maintains a per-`xmlmapconfig` cache (`parameterized_map_cache`) mapping options strings to pre-built `mapnik::Map` copies. This prevents Mapnik's PostGIS datasource plugin from creating a new PostgreSQL connection pool entry on every render, which previously caused rapid memory exhaustion. The cache is per render-thread and unbounded in size; in practice the number of distinct options values is small (e.g. one per requested language).

## Repository Layout

```
includes/       — public headers (protocol, store interface, config structs)
src/            — C/C++ source for mod_tile, renderd, storage backends, utilities
tests/          — Catch2 test suites + fixtures
cmake/          — custom Find*.cmake modules
docs/build/     — per-distro build instructions
docs/man/       — man pages
etc/            — example Apache and renderd config files
utils/          — example map data
.github/        — CI workflows (build-and-test, lint, flawfinder, docker)
```
