# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

sapcli is a command-line interface tool for SAP products, primarily focused on ABAP Development Tools (ADT) and SAP NetWeaver RFC. It enables CI/CD automation for SAP systems through HTTP/REST APIs and RFC protocols.

## Development Commands

### Testing
```bash
# Run all unit tests
make test
# OR
python -m unittest discover -b -v -s test/unit

# Run specific test pattern
python -m unittest discover -b -v -s test/unit -k test-name-pattern
```

### Linting
```bash
# Run all linters (pylint, flake8, mypy)
make lint

# Individual linters
pylint --rcfile=.pylintrc sap
PYTHONPATH=$(pwd):$PYTHONPATH pylint --rcfile=.pylintrc bin/sapcli
flake8 --config=.flake8 sap
PYTHONPATH=$(pwd):$PYTHONPATH flake8 --config=.flake8 bin/sapcli
```

### Code Coverage
```bash
# Generate coverage report
make report-coverage

# Generate HTML coverage report
make report-coverage-html
# Report at: .htmlcov/index.html

# Full check (lint + coverage)
make check
```

## Architecture

### Layered Design

The codebase follows a strict separation of concerns with three main layers:

1. **Core APIs** (`sap/adt/`, `sap/rest/`, `sap/rfc/`, `sap/odata/`)
   - Protocol-level implementations
   - ADT (ABAP Development Tools) - HTTP/XML based
   - REST (gCTS) - RESTful APIs for Git-enabled CTS
   - RFC - SAP NetWeaver Remote Function Calls
   - OData - For BSP and Fiori Launchpad services
   - **IMPORTANT**: These modules MUST remain independent from CLI and each other

2. **CLI Layer** (`sap/cli/`)
   - Command implementations using the core APIs
   - Connection factory functions to create connections from CLI args
   - Command registration system via `CommandsCache` in `sap/cli/__init__.py`
   - **Keep business logic MINIMAL** - delegate to core APIs

3. **Entry Point** (`bin/sapcli`)
   - Argument parsing
   - Connection credential handling (supports env vars)
   - Command dispatch

### Connection Types

Four connection types are created from CLI args:

- `adt_connection_from_args()` - ADT/HTTP connections (most commands)
- `rfc_connection_from_args()` - RFC connections (user, strust, startrfc)
- `gcts_connection_from_args()` - REST connections for gCTS
- `odata_connection_from_args()` - OData services (BSP, FLP)

### Adding New Commands

When adding a new command:

1. Implement core functionality in `sap/adt/` or appropriate API module
2. Create CLI wrapper in `sap/cli/` that uses the core API
3. Register command in `sap/cli/__init__.py` in `CommandsCache.commands()`
4. Add tests in `test/unit/`

The command class pattern:
- Create a `CommandGroup` class with `name` attribute
- Implement `install_parser(parser)` to add arguments
- Implement command handler methods that accept `(connection, args)`

### Testing Framework

- Uses Python's built-in `unittest` (no pytest dependency)
- Tests located in `test/unit/` mirroring source structure
- Fixtures in `test/unit/fixtures_*.py` files
- Mock helpers in `test/unit/mock.py` and `test/unit/infra.py`
- System tests available in `test/system/`

### Configuration

Connection parameters can be provided via:
- CLI arguments (highest priority)
- Environment variables (SAP_ASHOST, SAP_CLIENT, SAP_USER, SAP_PASSWORD, etc.)
- Interactive prompts for missing credentials
- See `sap.cli.resolve_default_connection_values()` for full list

### Key Dependencies

The project minimizes dependencies:
- `requests` - HTTP communication for ADT/REST/OData
- `pyodata` - OData protocol support
- `PyYAML` - YAML parsing
- `PyRFC` - Optional, for RFC functionality (requires SAP NetWeaver RFC SDK)

### Coding Practices

- **Minimize dependencies** - avoid adding new external packages
- **Document through tests** - tests are primary documentation
- **Separation of concerns** - keep CLI separate from API implementations
- **Lazy loading** - CLI modules import API modules only when needed
- **Environment variable support** - use `SAP_*` prefix for all SAP-related env vars
- **Logging** - use `sap.get_logger()` for debug output, NOT for user-facing messages
- Use imperative, present tense in commit messages
- Amend commits when fixing review feedback (avoid fixup commits)

### Important Modules

- `sap/adt/core.py` - Base ADT Connection class with HTTP handling
- `sap/adt/objects.py` - Base classes for ABAP objects (ADTObject, etc.)
- `sap/adt/marshalling.py` - XML serialization/deserialization
- `sap/rest/gcts/` - Git-enabled CTS implementation
- `sap/cli/__init__.py` - Command registration and connection factories
