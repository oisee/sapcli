# RFC Function Modules Catalog in sapcli

**Created:** 2025-11-30
**Project:** oisee/sapcli - Command Line Interface for SAP products
**Total unique RFC modules:** 19

---

## Table of Contents

1. [User Management](#1-user-management)
2. [Security and SSL Management (STRUST)](#2-security-and-ssl-management-strust)
3. [Arbitrary RFC Execution](#3-arbitrary-rfc-execution)
4. [BAPI Response Handling](#4-bapi-response-handling)
5. [Summary Tables](#5-summary-tables)

---

## 1. User Management

**File:** `sap/rfc/user.py`
**CLI Command:** `sapcli user`
**Module Count:** 6

### 1.1 BAPI_USER_CREATE1

**Purpose:** Create a new ABAP user

**Invocation:** `UserManager.create_user()` (line 427)

**Input Parameters:**
- `USERNAME` - user login name
- `ADDRESS` - structure with data:
  - `FIRSTNAME` - first name
  - `LASTNAME` - last name
  - `E_MAIL` - email address
- `PASSWORD` - structure:
  - `BAPIPWD` - password (for Service type) or dummy password
- `ALIAS` - structure:
  - `USERALIAS` - alias for HTTP authentication
- `LOGONDATA` - structure:
  - `USTYP` - user type (A=Dialog, B=Service, C=Communications Data, S=System)
  - `CLASS` - user group
  - `GLTGV` - valid from date (format YYYYMMDD)
  - `GLTGB` - valid to date (format YYYYMMDD)

**Output Parameters:**
- `RETURN` - BAPIRET2 table with operation results

**Features:**
- For Dialog (A) and Communications Data (C) users, `SUSR_USER_CHANGE_PASSWORD_RFC` is called after creation
- Default valid from = today, valid to = 20991231

---

### 1.2 BAPI_USER_CHANGE

**Purpose:** Modify an existing ABAP user

**Invocation:** `UserManager.change_user()` (line 443)

**Input Parameters:**
- `USERNAME` - user name
- `PASSWORD` - new password (optional)
- `PASSWORDX` - structure with flag `BAPIPWD='X'` (if password is being changed)
- `ALIAS` - new alias (optional)
- `ALIASX` - structure with flag `BAPIALIAS='X'` (if alias is being changed)

**Output Parameters:**
- `RETURN` - BAPIRET2 table

**Features:**
- Before invocation, `BAPI_USER_GET_DETAIL` is executed to determine user type
- For Dialog and Communications Data types, `SUSR_USER_CHANGE_PASSWORD_RFC` is additionally called

---

### 1.3 BAPI_USER_GET_DETAIL

**Purpose:** Retrieve detailed information about a user

**Invocation:** `UserManager.fetch_user_details()` (lines 420-422)

**Input Parameters:**
- `USERNAME` - user name

**Output Parameters:**
- `LOGONDATA` - structure with authorization data (including `USTYP` - user type)
- `ADDRESS` - address data
- `RETURN` - BAPIRET2 table
- Many other structures with detailed information

**Usage:**
- Retrieve user type (`USTYP`) before password change

---

### 1.4 BAPI_USER_ACTGROUPS_ASSIGN

**Purpose:** Assign roles (Activity Groups) to a user

**Invocation:** `UserManager.assign_roles()` (line 460)

**Input Parameters:**
- `USERNAME` - user name
- `ACTIVITYGROUPS` - table with roles:
  - `AGR_NAME` - role name
  - `FROM_DAT` - valid from date (default = today)
  - `TO_DAT` - valid to date (default = 20991231)

**Output Parameters:**
- `RETURN` - BAPIRET2 table

---

### 1.5 BAPI_USER_PROFILES_ASSIGN

**Purpose:** Assign authorization profiles to a user

**Invocation:** `UserManager.assign_profiles()` (line 470)

**Input Parameters:**
- `USERNAME` - user name
- `PROFILES` - table with profiles:
  - `BAPIPROF` - profile name

**Output Parameters:**
- `RETURN` - BAPIRET2 table

---

### 1.6 SUSR_USER_CHANGE_PASSWORD_RFC

**Purpose:** Change productive password for user (for Dialog and Communications Data types)

**Invocation:** `UserManager._call_user_change_password()` (line 385)

**Input Parameters:**
- `BNAME` - user name
- `PASSWORD` - current password (or dummy password)
- `NEW_PASSWORD` - new password
- `USE_BAPI_RETURN` - flag (1 = use BAPI response format)

**Output Parameters:**
- `RETURN` - BAPIRET2 structure

**Features:**
- Workaround is used: dummy password is set during creation, then changed to real password
- Dummy password is taken from environment variable `SAPCLI_ABAP_USER_DUMMY_PASSWORD` (default: `DummyPwd123!`)
- Applied only for types: A (Dialog), C (Communications Data)
- For type B (Service), password is set directly via BAPI_USER_CREATE1/CHANGE

---

## 2. Security and SSL Management (STRUST)

**File:** `sap/rfc/strust.py`
**CLI Command:** `sapcli strust`
**Module Count:** 13

### 2.1 SSFR_PSE_CHECK

**Purpose:** Check existence and status of PSE (Personal Security Environment) storage

**Invocation:** `SSLCertStorage.exists()` (lines 148-151)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - identification structure:
  - `PSE_CONTEXT` - PSE context (e.g., 'SSLS', 'SSLC')
  - `PSE_APPLIC` - PSE application (e.g., 'DFAULT', 'ANONYM')

**Output Parameters:**
- `ET_BAPIRET2` - table with check results

**Return Codes:**
- `TYPE='E', NUMBER='031'` - PSE does not exist
- `TYPE='S'` - PSE exists and is OK
- Other errors - PSE is corrupted

---

### 2.2 SSFR_PSE_CREATE

**Purpose:** Create a new PSE file with cryptographic keys

**Invocation:** `SSLCertStorage.create_pse()` (line 185)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification
- `IV_ALG` - encryption algorithm:
  - `R` = RSA
  - `S` = RSAwithSHA256
  - `D` = DSA
  - `G` = GOST_R_34.10-94
  - `H` = GOST_R_34.10-2001
  - `X` = CCL (Custom Crypto Library)
- `IV_KEYLEN` - key length in bits (default: 2048)
- `IV_REPLACE_EXISTING_PSE` - overwrite flag ('X' = overwrite, '-' = error if exists)
- `IV_DN` - Distinguished Name (optional)

**Output Parameters:**
- `ET_BAPIRET2` - results table

---

### 2.3 SSFR_PSE_REMOVE

**Purpose:** Delete PSE file from STRUST storage

**Invocation:** `SSLCertStorage.remove()` (lines 215-218)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification

**Output Parameters:**
- `ET_BAPIRET2` - results table

---

### 2.4 SSFR_PSE_UPLOAD_XSTRING

**Purpose:** Upload PSE file from binary data (XSTRING)

**Invocation:** `SSLCertStorage.upload()` (lines 236-239)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification
- `IV_PSE_XSTRING` - binary PSE file content
- `IV_REPLACE_EXISTING_PSE` - overwrite flag ('X' or '-')
- `IV_PSEPIN` - PSE file password (optional)

**Output Parameters:**
- `ET_BAPIRET2` - results table

**Usage:**
- Import PSE file created by external tools (OpenSSL, etc.)

---

### 2.5 SSFR_IDENTITY_CREATE

**Purpose:** Create a new STRUST Application Identity (register PSE in the system)

**Invocation:** `SSLCertStorage.create_identity()` (lines 196-200)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - structure:
  - `PSE_CONTEXT` - context
  - `PSE_APPLIC` - application
  - `PSE_DESCRIPT` - description (optional)
  - `SPRSL` - SAP language code (optional)
- `IV_REPLACE_EXISTING_APPL` - overwrite flag ('X' or '-')

**Output Parameters:**
- `ET_BAPIRET2` - results table

---

### 2.6 SSFR_GET_ALL_STRUST_IDENTITIES

**Purpose:** Retrieve list of all existing STRUST identities in the system

**Invocation:** `list_identities()` (line 358)

**Input Parameters:** none

**Output Parameters:**
- `ET_STRUST_IDENTITIES` - table of all registered PSE identities
- `ET_BAPIRET2` - results table

**Usage:**
- Get list of all available PSE storages for administration

---

### 2.7 SSFR_GET_OWNCERTIFICATE

**Purpose:** Retrieve the system's own certificate from PSE

**Invocation:** `SSLCertStorage.get_own_certificate()` (lines 278-281)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification

**Output Parameters:**
- `EV_CERTIFICATE` - X.509 certificate in Base64 format
- `ET_BAPIRET2` - results table

**Usage:**
- Export own certificate for sharing with partners

---

### 2.8 SSFR_GET_CERTIFICATELIST

**Purpose:** Retrieve list of trusted certificates from PSE storage

**Invocation:** `SSLCertStorage.get_certificates()` (lines 292-295)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification

**Output Parameters:**
- `ET_CERTIFICATELIST` - table of X.509 certificates in binary format

**Usage:**
- View all imported CA certificates

---

### 2.9 SSFR_PUT_CERTIFICATE

**Purpose:** Add X.509 certificate to the trusted certificates list

**Invocation:** `SSLCertStorage.put_certificate()` (lines 252-256)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification
- `IV_CERTIFICATE` - binary X.509 certificate data

**Output Parameters:**
- `ET_BAPIRET2` - results table

**Error Codes:**
- `TYPE='E', NUMBER='522'` - certificate already exists

**Usage:**
- Import CA certificates to establish trust

---

### 2.10 SSFR_PUT_CERTRESPONSE

**Purpose:** Upload signed Identity Certificate (response from Certificate Authority)

**Invocation:** `SSLCertStorage.put_identity_cert()` (lines 327-333)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification
- `IT_CERTRESPONSE` - table of strings (80 characters each) with certificate content
- `IV_CERTRESPONSE_LEN` - total certificate length in bytes
- `IV_PSEPIN` - PSE password (usually empty)

**Output Parameters:**
- `ET_BAPIRET2` - results table

**Features:**
- Certificate must be split into 80-character strings
- `PKCResponseABAPData` class is used for data formatting

**Workflow:**
1. Create PSE with `SSFR_PSE_CREATE`
2. Get CSR with `SSFR_GET_CERTREQUEST`
3. Sign CSR at Certificate Authority
4. Upload signed certificate with `SSFR_PUT_CERTRESPONSE`

---

### 2.11 SSFR_GET_CERTREQUEST

**Purpose:** Retrieve Certificate Signing Request (CSR) for sending to Certificate Authority

**Invocation:** `SSLCertStorage.get_csr()` (lines 311-314)

**Input Parameters:**
- `IS_STRUST_IDENTITY` - PSE identification

**Output Parameters:**
- `ET_CERTREQUEST` - table of strings with CSR in PEM format
- `ET_BAPIRET2` - results table

**Usage:**
- Generate CSR to obtain signed certificate from CA

---

### 2.12 SSFR_PARSE_CERTIFICATE

**Purpose:** Parse and extract information from X.509 certificate

**Invocations:**
- `SSLCertStorage.parse_certificate()` (lines 301-304)
- `iter_storage_certificates()` (line 372)

**Input Parameters:**
- `IV_CERTIFICATE` - binary X.509 certificate data

**Output Parameters:**
- Structure with certificate details:
  - Issuer
  - Subject
  - Serial Number
  - Validity dates
  - Public key info
  - Extensions

**Usage:**
- View certificate details before import
- Certificate validation

---

### 2.13 ICM_SSL_PSE_CHANGED

**Purpose:** Notify ICM (Internet Communication Manager) about PSE changes

**Invocation:** `notify_icm_changed_pse()` (line 352)

**Input Parameters:** none

**Output Parameters:** none

**Usage:**
- MANDATORY call after any PSE changes
- Forces ICM to reload PSE without system restart
- Critical for activating new certificates in SSL connections

---

### Standard PSE Identities

Defined in `IDENTITY_MAPPING`:

| Constant | PSE_CONTEXT | PSE_APPLIC | Purpose |
|----------|-------------|------------|---------|
| `server_standard` | SSLS | DFAULT | SSL Server (standard HTTPS server) |
| `client_anonymous` | SSLC | ANONYM | Anonymous SSL Client |
| `client_standard` | SSLC | DFAULT | Standard SSL Client |

---

## 3. Arbitrary RFC Execution

**File:** `sap/cli/startrfc.py`
**CLI Command:** `sapcli startrfc <FM_NAME>`

### 3.1 Dynamic Invocation of Any RFC Module

**Invocation:** `startrfc()` (line 116)

**Capabilities:**
- Invoke **any** RFC-enabled function module by name
- Pass parameters in JSON format
- Support for additional parameter types via CLI flags

**Parameter Formats:**

1. **JSON (primary format):**
```bash
sapcli startrfc RFC_SYSTEM_INFO '{}'
sapcli startrfc BAPI_USER_GET_DETAIL '{"USERNAME": "DEVELOPER"}'
```

2. **Read from stdin:**
```bash
echo '{"QUERY": "SELECT * FROM MARA"}' | sapcli startrfc MY_RFC_FM -
```

3. **String parameters (-S):**
```bash
sapcli startrfc BAPI_USER_CREATE1 '{}' -S USERNAME:TESTUSER -S ADDRESS.FIRSTNAME:John
```

4. **Integer parameters (-I):**
```bash
sapcli startrfc Z_CUSTOM_FM '{}' -I MAX_ROWS:1000 -I CLIENT:100
```

5. **File parameters (-F):**
```bash
sapcli startrfc Z_UPLOAD_FILE '{}' -F FILE_CONTENT:/path/to/file.bin
```

**Result Check Modes (-c):**

1. **raw (default)** - output raw response without checks
2. **bapi** - check RETURN structure and return error code on TYPE='E'

**Output Formats (-o):**

1. **human (default)** - pretty-printed format
2. **json** - JSON format (bytes converted to Base64)

**Usage Examples:**

```bash
# Get system information
sapcli startrfc RFC_SYSTEM_INFO '{}'

# Create user with BAPI response check
sapcli startrfc BAPI_USER_CREATE1 \
  '{"USERNAME":"TEST001","PASSWORD":{"BAPIPWD":"Init1234"},"LOGONDATA":{"USTYP":"B"}}' \
  -c bapi

# Output to JSON file
sapcli startrfc RFC_READ_TABLE \
  '{"QUERY_TABLE":"USR02","DELIMITER":"|"}' \
  -o json -R /tmp/response.json

# Upload file
sapcli startrfc Z_UPLOAD_BINARY '{}' \
  -S FILENAME:document.pdf \
  -F BINARY_DATA:/path/to/document.pdf
```

---

## 4. BAPI Response Handling

**File:** `sap/rfc/bapi.py`

### 4.1 BAPIReturn Class

Universal handler for BAPI responses (RETURN structure).

**BAPI Message Types (BAPI_MTYPE):**

| Code | Type | Description |
|------|------|-------------|
| S | Success | Successful execution |
| E | Error | Error |
| W | Warning | Warning |
| I | Info | Information message |
| A | Abort | Critical error (abort) |

**Methods:**

1. **`__init__(bapi_return)`** - accepts dict or list[dict]
2. **`is_error`** - property, True if TYPE='E' exists
3. **`is_empty`** - property, True if no messages
4. **`error_message`** - property, first error message
5. **`message_lines()`** - list of all messages in readable format
6. **`contains(msg_class, msg_number)`** - check for specific message
7. **`raise_for_error()`** - static method, raises BAPIError on errors

**Message Format:**
```
{TYPE}({ID}|{NUMBER}): {MESSAGE}
```

Example:
```
Error(SU|001): User TEST001 already exists
Success: User created successfully
```

**Usage:**

```python
# In UserManager._call_bapi_method()
resp = connection.call('BAPI_USER_CREATE1', **params)
BAPIError.raise_for_error(resp['RETURN'], resp)  # Raises exception on error
return resp

# In strust.py
bapiret = BAPIReturn(stat['ET_BAPIRET2'])
if bapiret.is_error:
    raise BAPIError(bapiret, stat)
```

---

## 5. Summary Tables

### 5.1 By Category

| Category | Count | File | CLI Command |
|----------|-------|------|-------------|
| User Management | 6 | sap/rfc/user.py | `sapcli user` |
| STRUST/SSL | 13 | sap/rfc/strust.py | `sapcli strust` |
| **Total Fixed** | **19** | - | - |
| Dynamic Invocation | ∞ | sap/cli/startrfc.py | `sapcli startrfc` |

### 5.2 Alphabetical List of All Modules

| № | RFC Module | Category | Purpose |
|---|-----------|----------|---------|
| 1 | BAPI_USER_ACTGROUPS_ASSIGN | User | Assign roles to user |
| 2 | BAPI_USER_CHANGE | User | Modify user |
| 3 | BAPI_USER_CREATE1 | User | Create user |
| 4 | BAPI_USER_GET_DETAIL | User | Get user details |
| 5 | BAPI_USER_PROFILES_ASSIGN | User | Assign profiles to user |
| 6 | ICM_SSL_PSE_CHANGED | STRUST | Notify ICM of PSE changes |
| 7 | SSFR_GET_ALL_STRUST_IDENTITIES | STRUST | List all PSE identities |
| 8 | SSFR_GET_CERTIFICATELIST | STRUST | List trusted certificates |
| 9 | SSFR_GET_CERTREQUEST | STRUST | Get CSR |
| 10 | SSFR_GET_OWNCERTIFICATE | STRUST | Get own certificate |
| 11 | SSFR_IDENTITY_CREATE | STRUST | Create PSE identity |
| 12 | SSFR_PARSE_CERTIFICATE | STRUST | Parse X.509 certificate |
| 13 | SSFR_PSE_CHECK | STRUST | Check PSE existence |
| 14 | SSFR_PSE_CREATE | STRUST | Create PSE file |
| 15 | SSFR_PSE_REMOVE | STRUST | Remove PSE file |
| 16 | SSFR_PSE_UPLOAD_XSTRING | STRUST | Upload PSE from binary |
| 17 | SSFR_PUT_CERTIFICATE | STRUST | Add trusted certificate |
| 18 | SSFR_PUT_CERTRESPONSE | STRUST | Upload signed Identity Certificate |
| 19 | SUSR_USER_CHANGE_PASSWORD_RFC | User | Change productive password |

### 5.3 By Usage Frequency

| RFC Module | Code Invocations | Criticality |
|-----------|------------------|-------------|
| BAPI_USER_CREATE1 | 1 | High |
| BAPI_USER_CHANGE | 1 | High |
| BAPI_USER_GET_DETAIL | 1 | Medium |
| SUSR_USER_CHANGE_PASSWORD_RFC | 1 | High |
| SSFR_PSE_CREATE | 1 | High |
| SSFR_PUT_CERTIFICATE | 1 | High |
| SSFR_PARSE_CERTIFICATE | 2 | Medium |
| ICM_SSL_PSE_CHANGED | 1 | **Critical** |
| Others | 1 | Medium |

---

## 6. Important Notes

### 6.1 What Does NOT Use RFC

1. **Transport Management (CTS)** - uses **ADT HTTP API**:
   - `/sap/bc/adt/cts/transportrequests/*`
   - `/sap/bc/adt/cts/transports/*/newreleasejobs`

2. **gCTS (Git-enabled CTS)** - uses **REST API**:
   - `/sap/bc/cts_abapvcs/repository`
   - `/sap/bc/cts_abapvcs/repository/{rid}/*`

3. **All development operations** - use **ADT protocol** (HTTP/XML):
   - ABAP Classes, Programs, Includes, Interfaces
   - Function Groups, Function Modules
   - Data Definitions (CDS Views)
   - Packages
   - ABAP Unit (aunit)
   - ATC (Code Inspector)
   - Data Preview
   - BSP Applications
   - Fiori Launchpad

### 6.2 When RFC Is Used

RFC is used **only** in the following cases:

1. **Administrative tasks without ADT API:**
   - User management
   - Role and profile assignment

2. **System security:**
   - PSE and SSL certificate management
   - STRUST operations

3. **Legacy functions:**
   - Functions historically available only via RFC
   - Functions without modern ADT endpoints

4. **Specific tasks on demand:**
   - Via `startrfc` command to invoke custom FMs

### 6.3 Architectural Principles

1. **Preference for ADT over RFC:**
   - Where possible, ADT HTTP API is used
   - RFC only when ADT is unavailable

2. **Modular organization:**
   - Each RFC call category in a separate file
   - Clear separation: User / STRUST / Custom

3. **Error handling:**
   - All BAPI calls checked via `BAPIError.raise_for_error()`
   - Unified BAPIRET2 structure handling

4. **Dependency Injection:**
   - All RFC calls receive connection as parameter
   - No global connection objects

---

## 7. Complete Workflow Examples

### 7.1 Creating User with Roles

```bash
# 1. Create Dialog user
sapcli user create DEV001 \
  --first-name John \
  --last-name Developer \
  --email john.developer@company.com \
  --type A \
  --password SecurePass123! \
  --password-productive

# Internally executes:
# - BAPI_USER_CREATE1 (with dummy password)
# - SUSR_USER_CHANGE_PASSWORD_RFC (set real password)

# 2. Assign roles
sapcli user assign-roles DEV001 Z_DEVELOPER SAP_BC_BASIS_ADMIN

# Internally executes:
# - BAPI_USER_ACTGROUPS_ASSIGN
```

### 7.2 SSL Certificate Setup

```bash
# 1. Create PSE
sapcli strust createpse server_standard \
  --algorithm RSAwithSHA256 \
  --key-length 4096 \
  --dn "CN=myserver.company.com,O=My Company,C=US"

# Internally executes:
# - SSFR_IDENTITY_CREATE
# - SSFR_PSE_CREATE

# 2. Get CSR
sapcli strust getcsr server_standard > server.csr

# Internally executes:
# - SSFR_GET_CERTREQUEST

# 3. (Sign CSR at CA - external process)

# 4. Upload signed certificate
sapcli strust putcertresponse server_standard server-signed.crt

# Internally executes:
# - SSFR_PUT_CERTRESPONSE

# 5. Add CA certificate to trust list
sapcli strust putcertificate server_standard ca-root.crt

# Internally executes:
# - SSFR_PUT_CERTIFICATE

# 6. Notify ICM
sapcli strust icm-notify

# Internally executes:
# - ICM_SSL_PSE_CHANGED
```

---

**Document prepared based on source code analysis of oisee/sapcli project**
**All RFC calls found and documented**
**Last updated:** 2025-11-30
