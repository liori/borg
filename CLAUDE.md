# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BorgBackup (Borg) is a deduplicating backup program with compression and authenticated encryption. It's written in Python with performance-critical components in Cython/C for chunking, compression, and encryption operations.

## Development Commands

### Initial Setup

```bash
# Create and activate virtual environment (recommended)
python3 -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows

# Install in development mode (this compiles Cython extensions)
pip install -e .

# Install development dependencies
pip install -r requirements.d/development.txt

# Install pre-commit hooks
pre-commit install
```

### Running Tests

Tests use pytest through tox. Most tests require `fakeroot` for permission testing.

```bash
# Run all tests on all Python versions
fakeroot -u tox

# Run tests on specific Python version
fakeroot -u tox -e py38

# Run specific test module
fakeroot -u tox borg.testsuite.locking

# Run specific test by name pattern
fakeroot -u tox borg.testsuite -- -k "test_name"

# Run with verbose output
fakeroot -u tox borg.testsuite -- -v

# Run tests excluding certain patterns
fakeroot -u tox borg.testsuite.locking -- -k '"not Timer"'

# Run with parallel execution (adjust XDISTN for worker count)
XDISTN=4 fakeroot -u tox

# Direct pytest (without tox, for single test iteration)
pytest -v -n auto --pyargs borg.testsuite.module_name
```

**Important:** When using `--` to pass options to pytest, you MUST also specify `borg.testsuite[.module]`.

### Linting

```bash
# Run flake8 via tox
tox -e flake8

# Run flake8 directly
flake8 src scripts conftest.py

# Run all pre-commit hooks
pre-commit run --all-files
```

Code style follows PEP 8 with 120 character line length. See `setup.cfg` for specific flake8 configuration.

### Building Documentation

```bash
# Install documentation dependencies
pip install -r requirements.d/docs.txt

# Build HTML documentation
cd docs/
make html

# View at docs/_build/html/index.html

# Rebuild command usage docs (after changing CLI)
python setup.py build_usage
python setup.py build_man
```

### Other Commands

```bash
# Clean compiled Cython files
python setup.py clean2

# Verify tox.ini changes
fakeroot -u tox --recreate
```

## Architecture Overview

### Entry Point and CLI

- **`src/borg/archiver.py`** (302KB): Main CLI entry point. Defines all subcommands (create, extract, list, mount, etc.) using argparse. Entry point is `borg.archiver:main`.

### Core Backup Components

- **`src/borg/archive.py`**: Archive creation, extraction, and checking. Handles filesystem traversal, metadata collection, and chunk processing.
- **`src/borg/repository.py`**: Low-level repository operations, manifest management, and transaction handling (PUT/DELETE/COMMIT).
- **`src/borg/cache.py`**: Local cache management for chunk index and file metadata. Critical for performance.
- **`src/borg/remote.py`**: SSH remote repository operations using `RemoteRepository` and `RepositoryServer`.

### Deduplication Pipeline

1. **Chunking** (`src/borg/chunker.pyx`): Content-defined chunking using Buzhash rolling hash algorithm.
2. **Hashing** (`src/borg/hashindex.pyx`): Hash index for chunk deduplication lookup.
3. **Compression** (`src/borg/compress.pyx`): LZ4, zstd, zlib, lzma compression.
4. **Encryption** (`src/borg/crypto/low_level.pyx`): AES-256 encryption and HMAC-SHA256 authentication.

### Crypto Subsystem

Located in `src/borg/crypto/`:

- **`low_level.pyx`**: Cython wrapper for OpenSSL crypto primitives (AES, HMAC, PBKDF2).
- **`key.py`**: Key management classes (RepoKey, PassphraseKey, etc.) and key file handling.
- **`keymanager.py`**: Key creation, loading, and encryption mode selection.
- **`nonces.py`**: Nonce management for authenticated encryption.

All encryption happens client-side. Repository servers never see unencrypted data.

### Platform Abstraction

`src/borg/platform/` contains OS-specific code:

- **`posix.pyx`**: POSIX common functionality (xattrs, ACLs).
- **`linux.pyx`**: Linux-specific features (ACLs via libacl, sync_file_range).
- **`darwin.pyx`**: macOS-specific features.
- **`freebsd.pyx`**: FreeBSD-specific features (file flags).
- **`windows.pyx`**: Windows support (experimental).

Platform selection happens at build time based on `sys.platform`.

### Helpers and Utilities

`src/borg/helpers/` contains utility modules:

- **`errors.py`**: Exception definitions.
- **`fs.py`**: Filesystem utilities.
- **`manifest.py`**: Repository manifest handling.
- **`msgpack.py`**: MessagePack serialization wrapper.
- **`parseformat.py`**: Parsing and formatting for archive names, patterns, etc.
- **`progress.py`**: Progress indicators.

### Algorithms

`src/borg/algorithms/`:

- **`checksums.pyx`**: xxHash and CRC32 implementations in Cython.

Bundled C implementations of LZ4, zstd, and xxHash are in `src/borg/algorithms/` (fallback if system libraries unavailable).

### Testing

`src/borg/testsuite/` contains 31 test modules covering unit and integration tests. Key test fixtures in `conftest.py` set up isolated test environments (XDG_CONFIG_HOME, XDG_CACHE_HOME).

## Build System Details

### Cython Compilation

Performance-critical modules are written in Cython (`.pyx` files) and compiled to C extensions:

- `chunker.pyx`
- `compress.pyx`
- `hashindex.pyx`
- `item.pyx`
- `crypto/low_level.pyx`
- `algorithms/checksums.pyx`
- Platform-specific: `platform/{posix,linux,darwin,freebsd,windows}.pyx`

Extensions are compiled automatically by `pip install -e .` or `python setup.py build_ext`.

### Dependency Management

**Core dependencies:**
- `msgpack` (0.5.6 to 1.1.1, exact version critical)
- `packaging`

**Optional:**
- `pyfuse3` or `llfuse` for `borg mount` functionality

**Build-time:**
- `Cython`
- `setuptools_scm` (version from git tags)
- `pkgconfig` (for locating system libraries)

### System Libraries

By default, setup.py attempts to use system libraries. Override with environment variables:

- `BORG_USE_BUNDLED_LZ4=YES` - Use bundled LZ4 instead of system library
- `BORG_USE_BUNDLED_ZSTD=YES` - Use bundled zstd
- `BORG_USE_BUNDLED_XXHASH=YES` - Use bundled xxHash
- `BORG_OPENSSL_PREFIX=/path` - Specify OpenSSL location (required, not bundled)
- `BORG_LIBLZ4_PREFIX=/path` - Specify LZ4 location
- `BORG_LIBZSTD_PREFIX=/path` - Specify zstd location

## Code Organization Patterns

### Command Implementation

Commands in `archiver.py` follow this pattern:

1. Argument parsing with `argparse` decorators
2. Command function decorated with `@with_repository`, `@with_archive`, etc.
3. Business logic delegates to classes in other modules (Archive, Repository, Cache)
4. Progress output via topic loggers (controlled by `--stats`, `--list`, etc.)

### Error Handling

- User-facing errors inherit from `Error` class in `helpers/errors.py`
- Exit codes: 0 (success), 1 (warning), 2 (error), 128+ (signals)
- Critical errors in import block cause `sys.exit(2)` to ensure proper exit code

### Logging

- Use `logger.debug()`, `logger.info()`, `logger.warning()`, `logger.error()`
- Direct user interaction (Y/N prompts) goes to stderr, not logging
- Topic loggers for `--stats` and `--list` flags (see `_setup_implied_logging()` in archiver.py)

## Development Workflow

### Branching Model

- Main development on `master` branch
- Maintenance branches: `1.0-maint`, `1.1-maint`, `1.2-maint`, etc.
- PRs target `master` unless fixing maintenance-only issues
- Backports labeled with `backport/x.y-maint`

### Contribution Guidelines

- Run tests before submitting PR (`fakeroot -u tox`)
- Ensure flake8 compliance (`tox -e flake8`)
- Add tests and docs for new features
- Clean commits focused on single topics
- Refer to issues in commit messages
- Update `CHANGES.rst` for user-facing changes (maintainers handle this during release)

### Pre-commit Hooks

Project uses pre-commit for automated linting. After `pre-commit install`, hooks run on every commit. Force-run with `pre-commit run --all-files`.
