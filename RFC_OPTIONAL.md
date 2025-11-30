# RFC is Optional in sapcli

## Summary

**RFC (PyRFC) is NOT required** to use most of sapcli functionality. It is an **optional dependency** needed only for specific administrative commands.

---

## Installation Options

### Option 1: Basic Installation (ADT only - No RFC)

Most users need only this:

```bash
pip install requests pyodata PyYAML
```

**Available commands:** All ABAP development commands via ADT protocol
- ✅ program, include, interface, class, ddl
- ✅ functiongroup, functionmodule
- ✅ aunit, atc, datapreview
- ✅ package, cts, gcts
- ✅ checkout, checkin, activation
- ✅ abapgit, bsp, flp, rap
- ✅ table, structure, dataelement, badi
- ✅ adt (ADT discovery)

**Total: 25+ commands work WITHOUT RFC**

---

### Option 2: Full Installation (ADT + RFC)

Only needed if you require user management or SSL certificate operations:

```bash
# Install base dependencies first
pip install requests pyodata PyYAML

# Install SAP NetWeaver RFC SDK (see https://sap.github.io/PyRFC/install.html)
# Then install PyRFC
pip install pyrfc
```

**Additional commands with RFC:**
- ❌ user (User Management: create, change, assign roles/profiles)
- ❌ strust (SSL Certificate Management)
- ❌ startrfc (Arbitrary RFC function module execution)

**Total: Only 3 commands require RFC**

---

## Dependency Architecture

### Core Dependencies (Required)

Listed in `setup.py` `install_requires`:
- **requests** - HTTP communication for ADT/REST/OData
- **pyodata** - OData protocol support (BSP, FLP)
- **PyYAML** - YAML parsing

### Optional Dependencies

**NOT** listed in `install_requires`:
- **pyrfc** - SAP NetWeaver RFC SDK Python bindings
  - Requires native SAP NetWeaver RFC SDK libraries
  - Complex installation process
  - Only needed for 3 specific commands

---

## Runtime Behavior

### Without PyRFC installed

```bash
$ sapcli user create TEST001 --first-name Test --last-name User
Exception (SAPCliError):
  RFC functionality is not available(enabled)
```

**Clear error message** - no crash, graceful degradation

### With PyRFC installed

All 28 commands work normally.

---

## Architecture Implementation

From `sap/rfc/core.py`:

```python
def rfc_is_available():
    """Returns true if RFC can be used"""
    if SAPRFC_MODULE is None:
        _try_import_pyrfc()
    return SAPRFC_MODULE is not None

def _try_import_pyrfc():
    global SAPRFC_MODULE
    try:
        import pyrfc
    except ImportError as ex:
        mod_log().info('Failed to import the module pyrfc')
        mod_log().debug(str(ex))
    else:
        SAPRFC_MODULE = pyrfc
```

**Key points:**
- Graceful import handling
- No hard dependency
- Commands register regardless of PyRFC availability
- Error only at execution time if PyRFC needed but missing

---

## Use Cases

### You DON'T need RFC if you:

- ✅ Work with ABAP code (classes, programs, includes, interfaces)
- ✅ Run ABAP Unit tests
- ✅ Run ATC checks
- ✅ Manage packages and transports
- ✅ Use gCTS (Git-enabled CTS)
- ✅ Deploy BSP applications
- ✅ Work with Fiori Launchpad
- ✅ Manage CDS Views, tables, structures
- ✅ Use data preview
- ✅ Activate ABAP objects

**→ Basic installation is sufficient**

### You DO need RFC if you:

- ❌ Create/modify ABAP users
- ❌ Assign roles and profiles to users
- ❌ Manage SSL certificates (STRUST)
- ❌ Need to call custom RFC function modules
- ❌ Work with PSE (Personal Security Environment)

**→ Full installation required**

---

## Recommended Approach

### For CI/CD Pipelines

Use **basic installation** (no RFC):
- Faster setup
- No SAP NetWeaver RFC SDK required
- No native library dependencies
- Simpler Docker images

### For System Administration

Use **full installation** (with RFC):
- Covers all administrative tasks
- User management capabilities
- Security operations

---

## Command Classification

### ADT Protocol (25 commands - No RFC needed)

| Command | Purpose |
|---------|---------|
| program | ABAP Reports/Programs |
| include | ABAP Includes |
| interface | ABAP OO Interfaces |
| class | ABAP OO Classes |
| ddl | CDS Views (Data Definitions) |
| functiongroup | Function Groups |
| functionmodule | Function Modules |
| aunit | ABAP Unit Tests |
| atc | ABAP Test Cockpit / Code Inspector |
| datapreview | SQL Query Preview |
| package | Package Management |
| cts | Change Transport System |
| checkout | Download ABAP objects |
| checkin | Upload ABAP objects |
| activation | Activate ABAP objects |
| adt | ADT discovery |
| abapgit | abapGit operations |
| rap | RAP Business Services |
| table | DDIC Tables |
| structure | DDIC Structures |
| dataelement | DDIC Data Elements |
| badi | BAdI Enhancement Implementations |

### REST Protocol (1 command - No RFC needed)

| Command | Purpose |
|---------|---------|
| gcts | Git-enabled Change Transport System |

### OData Protocol (2 commands - No RFC needed)

| Command | Purpose |
|---------|---------|
| bsp | BSP Application Management |
| flp | Fiori Launchpad |

### RFC Protocol (3 commands - PyRFC required)

| Command | Purpose | RFC Modules Used |
|---------|---------|------------------|
| user | User Management | 6 BAPIs (BAPI_USER_*) |
| strust | SSL Certificates | 13 FMs (SSFR_*, ICM_SSL_PSE_CHANGED) |
| startrfc | Custom RFC calls | Any user-specified FM |

---

## Migration Path

If you're currently using RFC commands and want to minimize dependencies:

1. **Check your usage:**
   ```bash
   grep -r "sapcli user\|sapcli strust\|sapcli startrfc" ./
   ```

2. **Consider alternatives:**
   - User management → Manual via SAP GUI or use other tools
   - Certificate management → Manual via STRUST transaction
   - Custom RFC calls → Evaluate if ADT/REST alternatives exist

3. **Hybrid approach:**
   - Use lightweight sapcli for CI/CD (no RFC)
   - Use full sapcli on admin workstations (with RFC)

---

## FAQ

**Q: Why isn't PyRFC in requirements.txt?**
A: It's optional and has complex native dependencies. Users should install it only if needed.

**Q: Will my scripts break if I don't install PyRFC?**
A: No, if you don't use `user`, `strust`, or `startrfc` commands. All other commands work fine.

**Q: Can I install PyRFC later?**
A: Yes, install it anytime. No need to reinstall sapcli.

**Q: How do I know if I need RFC?**
A: Check if you use any of these commands: `user`, `strust`, `startrfc`. If not, you don't need RFC.

**Q: Does the Docker image need RFC?**
A: Only if you need user management or certificate operations in containerized environments.

---

## Performance Impact

### Basic Installation
- Installation time: < 1 minute
- Docker image size: +20 MB (Python + deps)
- No native compilation

### Full Installation with RFC
- Installation time: 5-15 minutes
- Docker image size: +200+ MB (RFC SDK + PyRFC)
- Requires native compilation
- Complex dependency chain

**→ Use basic installation when possible**

---

**Last updated:** 2025-11-30
**Related:** See RFC_MODULES_CATALOG.md for detailed RFC module documentation
