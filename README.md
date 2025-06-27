Bitcoin Core integration/staging tree
=====================================

https://bitcoincore.org

For an immediately usable, binary version of the Bitcoin Core software, see
https://bitcoincore.org/en/download/.

## Bitcoin Core Rust Migration Analysis

This repository contains a comprehensive analysis of migrating Bitcoin Core from C++ to Rust. After an extensive architectural review examining all 8 major subsystems (~320K lines of code), we evaluated migration strategies for both Go and Rust languages.

**Key Findings:**
- **Architecture Analysis**: Detailed examination of consensus validation, networking, mempool, storage, RPC, wallet, mining, and GUI subsystems
- **Language Comparison**: Comprehensive evaluation of Go vs Rust for Bitcoin Core requirements
- **Final Recommendation**: **Rust** selected as the optimal choice due to zero-cost abstractions, memory safety without garbage collection, and superior performance characteristics critical for consensus-critical systems
- **Migration Strategy**: 72-month phased migration plan with detailed work breakdown structure and project management framework

### Analysis Documentation

The complete analysis is available in the [`ai_docs/`](ai_docs/) directory:

- **[Architecture Overview](ai_docs/bitcoin-architecture-overview.md)** - Comprehensive technical analysis of Bitcoin Core's 8 major subsystems
- **[Architecture Diagrams](ai_docs/architecture-diagrams.md)** - Visual system diagrams and component interactions
- **[Detailed Flow Diagrams](ai_docs/detailed-flow-diagrams.md)** - Process flows, state machines, and data flows
- **[System Integration Diagrams](ai_docs/system-integration-diagrams.md)** - Complete system-level integration views
- **[Rust Migration Plan](ai_docs/rust-migration-plan.md)** - Comprehensive 72-month Rust migration strategy
- **[Go Migration Plan](ai_docs/go-migration-plan.md)** - Complete Go migration approach for comparison
- **[Language Comparison](ai_docs/language-comparison-analysis.md)** - Detailed Go vs Rust analysis across 8 critical factors
- **[Final Recommendation](ai_docs/final-recommendation.md)** - Executive recommendation with comprehensive justification
- **[Project Framework](ai_docs/rust-migration-project-framework.md)** - Complete project planning and execution framework
- **[Work Breakdown Structure](ai_docs/detailed-work-breakdown-structure.md)** - Detailed task breakdown for systematic implementation
- **[Task Templates](ai_docs/task-specification-templates.md)** - Standardized templates for task specification and execution
- **[Team Structure](ai_docs/project-management-team-structure.md)** - Project management approach and organizational structure

**Executive Summary**: Rust emerges as the clear choice for Bitcoin Core migration, offering memory safety without garbage collection overhead, deterministic behavior essential for consensus systems, and performance characteristics that can match or exceed C++. The migration would require 40-60 engineers over 6 years with an estimated budget of $26-39M, but would result in a more secure, maintainable, and performant Bitcoin Core implementation.

What is Bitcoin Core?
---------------------

Bitcoin Core connects to the Bitcoin peer-to-peer network to download and fully
validate blocks and transactions. It also includes a wallet and graphical user
interface, which can be optionally built.

Further information about Bitcoin Core is available in the [doc folder](/doc).

License
-------

Bitcoin Core is released under the terms of the MIT license. See [COPYING](COPYING) for more
information or see https://opensource.org/license/MIT.

Development Process
-------------------

The `master` branch is regularly built (see `doc/build-*.md` for instructions) and tested, but it is not guaranteed to be
completely stable. [Tags](https://github.com/bitcoin/bitcoin/tags) are created
regularly from release branches to indicate new official, stable release versions of Bitcoin Core.

The https://github.com/bitcoin-core/gui repository is used exclusively for the
development of the GUI. Its master branch is identical in all monotree
repositories. Release branches and tags do not exist, so please do not fork
that repository unless it is for development reasons.

The contribution workflow is described in [CONTRIBUTING.md](CONTRIBUTING.md)
and useful hints for developers can be found in [doc/developer-notes.md](doc/developer-notes.md).

Testing
-------

Testing and code review is the bottleneck for development; we get more pull
requests than we can review and test on short notice. Please be patient and help out by testing
other people's pull requests, and remember this is a security-critical project where any mistake might cost people
lots of money.

### Automated Testing

Developers are strongly encouraged to write [unit tests](src/test/README.md) for new code, and to
submit new unit tests for old code. Unit tests can be compiled and run
(assuming they weren't disabled during the generation of the build system) with: `ctest`. Further details on running
and extending unit tests can be found in [/src/test/README.md](/src/test/README.md).

There are also [regression and integration tests](/test), written
in Python.
These tests can be run (if the [test dependencies](/test) are installed) with: `build/test/functional/test_runner.py`
(assuming `build` is your build directory).

The CI (Continuous Integration) systems make sure that every pull request is tested on Windows, Linux, and macOS.
The CI must pass on all commits before merge to avoid unrelated CI failures on new pull requests.

### Manual Quality Assurance (QA) Testing

Changes should be tested by somebody other than the developer who wrote the
code. This is especially important for large or high-risk changes. It is useful
to add a test plan to the pull request description if testing the changes is
not straightforward.

Translations
------------

Changes to translations as well as new translations can be submitted to
[Bitcoin Core's Transifex page](https://explore.transifex.com/bitcoin/bitcoin/).

Translations are periodically pulled from Transifex and merged into the git repository. See the
[translation process](doc/translation_process.md) for details on how this works.

**Important**: We do not accept translation changes as GitHub pull requests because the next
pull from Transifex would automatically overwrite them again.
