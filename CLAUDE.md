# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

### Basic Build
```bash
# Configure the build
cmake -B build

# Build with parallel jobs (replace N with number of cores)
cmake --build build -j N

# Install (optional)
cmake --install build
```

### Common Build Configurations
```bash
# Developer mode with all features enabled
cmake --preset=dev-mode
cmake --build build_dev_mode

# Debug build
cmake -B build -DCMAKE_BUILD_TYPE=Debug

# Release build with debug info (default)
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo

# Build with clang instead of gcc
cmake -B build -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang
```

## Test Commands

### Unit Tests
```bash
# Run all unit tests
ctest --test-dir build

# Run with verbose output
ctest --test-dir build --output-on-failure

# Run specific test suite
build/bin/test_bitcoin --run_test=getarg_tests

# Run with full logging
build/bin/test_bitcoin --log_level=all --run_test=getarg_tests

# Run single test case
build/bin/test_bitcoin --run_test=getarg_tests/doubledash
```

### Functional Tests
```bash
# Run all functional tests
build/test/functional/test_runner.py

# Run specific test
build/test/functional/test_runner.py feature_block.py

# Run with parallel jobs
build/test/functional/test_runner.py -j 4

# Run tests matching pattern
build/test/functional/test_runner.py wallet_*
```

### Fuzz Tests
```bash
# Build for fuzzing
cmake --preset=libfuzzer
cmake --build build_fuzz

# Run a specific fuzzer
build_fuzz/bin/test_bitcoin_fuzzer -max_total_time=60 block_deserialize
```

## Lint Commands

### Run All Linters
```bash
# Using Docker (recommended for CI consistency)
DOCKER_BUILDKIT=1 docker build -t bitcoin-linter --file "./ci/lint_imagefile" ./ && docker run --rm -v $(pwd):/bitcoin -it bitcoin-linter

# Or using the test runner (requires Rust toolchain)
( cd ./test/lint/test_runner/ && cargo fmt && cargo clippy && RUST_BACKTRACE=1 cargo run )
```

### Individual Linters
```bash
# Python linting
test/lint/lint-python.py

# Shell script linting
test/lint/lint-shell.py

# Spelling check
test/lint/lint-spelling.py

# C++ formatting check
contrib/devtools/clang-format-diff.py
```

## High-Level Architecture

### Core Components

1. **Consensus Layer** (`src/consensus/`)
   - Transaction and block validation rules
   - Script verification
   - Consensus-critical constants

2. **Networking** (`src/net*`)
   - P2P protocol implementation
   - Connection management
   - Message handling

3. **Node Components** (`src/node/`)
   - Block and transaction processing
   - Chain state management
   - Mempool

4. **Wallet** (`src/wallet/`)
   - Key management
   - Transaction creation and signing
   - Balance tracking

5. **RPC Server** (`src/rpc/`)
   - JSON-RPC API implementation
   - Command handlers
   - Authentication

6. **GUI** (`src/qt/`)
   - Qt-based graphical interface
   - Separate repository for GUI-only changes

### Key Design Patterns

1. **Interface-based Design**
   - Abstract interfaces in `src/interfaces/`
   - Allows modular testing and alternative implementations

2. **Validation Architecture**
   - Separation between consensus rules and policy
   - Mempool acceptance vs block validation

3. **Threading Model**
   - Main validation thread
   - Network message handling threads
   - RPC server threads
   - GUI thread (when enabled)

4. **Data Persistence**
   - LevelDB for chain state and block index
   - SQLite for descriptor wallets
   - Custom format for block storage

### Important Files and Conventions

1. **Naming Conventions**
   - Classes: `UpperCamelCase`
   - Functions/Methods: `UpperCamelCase`
   - Variables: `snake_case`
   - Member variables: `m_` prefix
   - Global variables: `g_` prefix
   - Constants: `ALL_CAPS`

2. **Code Style**
   - Enforced by clang-format
   - 4-space indentation
   - Opening braces on new lines for functions/classes

3. **Testing Philosophy**
   - Every change should include tests
   - Unit tests for individual components
   - Functional tests for integration
   - Fuzz tests for security-critical code

### Development Workflow

1. **Before Making Changes**
   - Read relevant code and understand existing patterns
   - Check for related issues/PRs
   - Run existing tests to ensure clean baseline

2. **While Developing**
   - Follow existing code style in the file
   - Add tests for new functionality
   - Keep commits focused and atomic

3. **Before Submitting**
   - Run unit tests: `ctest --test-dir build`
   - Run functional tests for affected areas
   - Run linters if modifying C++ or Python code
   - Ensure compilation with different configurations

4. **Common Development Tasks**
   - Adding RPC command: Implement in `src/rpc/`, add tests
   - Modifying consensus: Extreme care needed, extensive testing
   - GUI changes: Consider using bitcoin-core/gui repository
   - Adding dependencies: Update build system and documentation