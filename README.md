# Web-Based Report Management Application

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend-009688?logo=fastapi&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL%20Server-Stored%20Procedures-CC2927?logo=microsoftsqlserver&logoColor=white)
![pyodbc](https://img.shields.io/badge/pyodbc-ODBC%20Driver%2017-0078D4)
![JWT](https://img.shields.io/badge/Auth-JWT%20(HS256)-000000?logo=jsonwebtokens&logoColor=white)
![bcrypt](https://img.shields.io/badge/Passwords-bcrypt-4B8BBE)
![Fernet](https://img.shields.io/badge/Secrets-Fernet%20(AES)-6A5ACD)
![pytest](https://img.shields.io/badge/Tests-104%20passed-0A9EDC?logo=pytest&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

> 🚀 **Secure Backend Project**: Config → Logging & Errors → SQL Server Stored-Procedure Layer → Security Core → Authentication → RBAC → User Management  
> 🔒 **Security-First Design**: bcrypt hashing, JWT auth, Fernet-encrypted credentials, least-privilege DB access, parameterized calls only  
> 🔧 **Quick Start**: See [Quick Start](#-quick-start) | [Project Documents](#-documentation) | [API Reference](#-api-reference)

A **centralized, configuration-driven report management platform**. An administrator defines a report once (name, SQL query, target database, schedule, output format, recipients) and non-technical users can then log in, browse the reports they are allowed to see, run them and download the results (Excel / PDF / CSV), without writing SQL or touching the database directly. Built by a **9-member team** (Frontend, Database, Backend/Security) in a one-month timeline, with this repository holding the **FastAPI backend**.

> ℹ️ **Current status:** Backend **Phases 0 – 4** are implemented and tested (scaffolding, DB access layer, security core, authentication & user management, RBAC). Report execution, export and scheduling (Phases 5 – 10) are still on the roadmap. See [Project Status & Roadmap](#-project-status--roadmap).

## 📁 Project Structure

```
Web-Based-Report-Management-Application-/
├── app/
│   ├── main.py                      # FastAPI entrypoint: routers, exception handlers, GET /health
│   ├── config.py                    # Settings (pydantic-settings) loaded from env / .env, cached singleton
│   │
│   ├── api/
│   │   ├── deps.py                  # get_current_user: decodes the Bearer JWT → CurrentUser(user_id, role)
│   │   └── routes/
│   │       ├── auth.py              # POST /auth/login
│   │       └── users.py             # POST/PUT/DELETE /users (admin only)
│   │
│   ├── core/
│   │   ├── security.py              # bcrypt hash/verify + JWT create/decode
│   │   ├── crypto.py                # Fernet encrypt/decrypt + startup key validation
│   │   ├── rbac.py                  # require_role / require_permission dependencies
│   │   ├── errors.py                # AppError hierarchy + clean JSON exception handlers
│   │   └── logging_config.py        # JSON-line structured logging
│   │
│   ├── db/
│   │   ├── mssql.py                 # pyodbc connection lifecycle, timeout + bounded retry
│   │   └── proc.py                  # call_proc / call_proc_scalar / query_function (only place that builds SQL)
│   │
│   ├── repositories/
│   │   └── users_repo.py            # 1:1 wrappers around the user/auth stored procedures
│   │
│   ├── services/
│   │   ├── auth_service.py          # authenticate(): verify credentials → JWT → audit log
│   │   └── user_service.py          # create / update / set password / deactivate + audit log
│   │
│   └── schemas/
│       └── user.py                  # Pydantic request/response models (never expose PasswordHash)
│
├── tests/                           # 16 test files, 105 tests (pytest)
│   ├── conftest.py                  # Test env vars (no real DB or secrets needed)
│   └── test_*.py                    # config, logging, errors, health, mssql, proc, security, crypto,
│                                    # deps, rbac, schemas, users_repo, auth/user services & routes
│
├── connectivity_check.py            # Manual script: verifies DB login, procedure access, and DENY grants
├── extract_pdf.py                   # Helper that prints the text of the two project PDFs
├── SRS_RE_1.pdf                     # Software Requirements Specification (15 pages)
├── PROJEC_1.PDF                     # Project status report: database & backend progress (5 pages)
├── PHASE2_AGENT_GUIDE.md            # Implementation guide used for Phase 2 (Security Core)
├── requirements.txt                 # Python dependencies
├── .env.example                     # Template for required environment variables
├── .gitignore
└── README.md
```

## 📊 Project at a Glance

| Item | Value |
|------|-------|
| **Purpose** | Self-service reporting layer between users and the databases |
| **Backend** | Python · FastAPI · pyodbc (no ORM) |
| **Database** | Microsoft SQL Server (`ReportManagementDB`) |
| **DB design** | 10 tables · 26 stored procedures · 6 functions |
| **Auth** | Username/password → JWT access token (HS256) |
| **Roles** | `admin` (full control) · `viewer` (execute & download reports) |
| **Platform scale** | Hub for 26 applications (data sources and consumers) |
| **Planned DB engines** | SQL Server · Oracle · DB2 |
| **Planned outputs** | Excel (`.xlsx`) · PDF · CSV |
| **Tests** | 105 collected: **104 passed, 1 skipped** (opt-in integration test needing a live DB) |
| **Team / Timeline** | 9 members · 1 month (Version 1.0, July 2026) |

> ℹ️ The database itself (schema, stored procedures, functions and security grants) is deployed from three SQL scripts that are maintained by the Database team and are **not included in this repository**. The backend only needs a reachable `ReportManagementDB` that exposes the procedures listed below.

## 🎯 Features Implemented

### 🔐 Authentication (`POST /auth/login`)
- Verifies a username/password pair against the bcrypt hash stored in the DB
- Returns `{"access_token": "...", "token_type": "bearer"}`
- Same generic `"Invalid credentials."` message for unknown user, wrong password **and** inactive account → **no user enumeration**
- Every attempt (success or failure) is written to the audit log (`LOGIN_SUCCESS` / `LOGIN_FAILED`)

### 👥 User Management (admin only)
- Create, update, change password and deactivate users
- Partial updates: unset fields stay unchanged
- Passwords are hashed **before** they reach the repository layer (plaintext never touches the DB)
- Every action is audit-logged (`USER_CREATE`, `USER_UPDATE`, `USER_SET_PASSWORD`, `USER_DEACTIVATE`)

### 🛡️ Role-Based Access Control
- Server-side enforcement through FastAPI dependencies, never relying on the frontend
- Mirrors the DB rule `dbo.ufn_IsActionAllowed`

| Action | Admin | Viewer |
|--------|:-----:|:------:|
| `manage_users` | ✅ | ❌ |
| `manage_reports` | ✅ | ❌ |
| `manage_connections` | ✅ | ❌ |
| `manage_schedules` | ✅ | ❌ |
| `execute_report` | ✅ | ✅ |
| `download_report` | ✅ | ✅ |

### 🔑 Security Core
- **Passwords:** bcrypt with a fresh salt per hash; inputs over 72 bytes are rejected instead of being silently truncated
- **JWT:** signed with `JWT_SECRET`, embeds `sub` (numeric UserId), `role` and `exp`; invalid, tampered or expired tokens all raise the same generic `AuthError`
- **Credential encryption:** Fernet (AES) for database credentials stored as `VARBINARY`; `CRED_ENCRYPTION_KEY` is validated at import time so a bad key **fails fast at startup**, not at the first report run

### 🗄️ Stored-Procedure-Only Data Access
- The app **never** runs raw table SQL: every call goes through `call_proc`, `call_proc_scalar` or `query_function`
- Values are always bound parameters; only validated identifiers (procedure and parameter names, checked against a strict regex allowlist) are placed in SQL text
- 5-second connect timeout and one bounded retry, only on transient SQLSTATEs (`08001`, `08S01`, `HYT00`, `HYT01`)
- Connections are always closed (context manager)

### 🧯 Error Handling & Logging
- `AppError` hierarchy: `NotFoundError` (404) · `AuthError` (401) · `PermissionError` (403) · `ValidationError` (422) · `DatabaseConnectionError` (503)
- A catch-all handler logs full details server-side and returns a generic 500: no stack traces, DB errors or secrets are leaked to clients
- JSON-line logs on stdout (`time`, `level`, `logger`, `message`) with the level driven by `LOG_LEVEL`

## 🚀 Quick Start

### 0. Clone and Install

```bash
git clone https://github.com/DiaaSalah57/Web-Based-Report-Management-Application-.git
cd Web-Based-Report-Management-Application-

python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

**Prerequisites**
- Python 3.11+
- [Microsoft ODBC Driver 17 for SQL Server](https://learn.microsoft.com/sql/connect/odbc/download-odbc-driver-for-sql-server) (on Linux/macOS also `unixODBC`)
- A deployed `ReportManagementDB` with the stored procedures and the restricted `app_role` login (to run the real app; not needed to run the tests)

### 1. Configure the Environment

```bash
cp .env.example .env        # Windows: copy .env.example .env
```

| Variable | Required | Description |
|----------|:--------:|-------------|
| `APP_DB_CONN` | ✅ | ODBC connection string for `ReportManagementDB`, using the restricted `app_role` login |
| `JWT_SECRET` | ✅ | Long random secret used to sign/verify JWTs |
| `JWT_EXPIRE_MINUTES` | ✅ | Token lifetime in minutes (e.g. `60`) |
| `CRED_ENCRYPTION_KEY` | ✅ | Fernet key (32-byte url-safe base64) for encrypting stored DB credentials |
| `LOG_LEVEL` | ❌ | `DEBUG` · `INFO` (default) · `WARNING` · `ERROR` · `CRITICAL` |

Generate the secrets:

```bash
# Fernet key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# JWT secret
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

> ⚠️ `.env` is git-ignored. Never commit real secrets.

### 2. Verify Database Connectivity (optional but recommended)

```bash
# macOS/Linux
export APP_DB_CONN="Driver={ODBC Driver 17 for SQL Server};Server=localhost;Database=ReportManagementDB;UID=app_role;PWD=...;"
# Windows PowerShell:  $env:APP_DB_CONN="..."

python connectivity_check.py
```

**Expected Output:**
```
Installed ODBC drivers: [...]
Connected OK.  Database = ReportManagementDB   Login = app_role
ufn_GetDueReports() returned N due report(s).
Good: direct table access is DENIED (security model is working).
RESULT: the backend can reach the database through stored procedures. ✔
```

The script checks three things: the login works, the app can execute database functions, and **direct `SELECT` on `dbo.Users` is denied** (proving the least-privilege grants are in place).

### 3. Run the API

```bash
uvicorn app.main:app --reload --port 8000
# Interactive docs: http://localhost:8000/docs
```

```bash
# Health check
curl http://localhost:8000/health
# {"status":"ok"}

# Log in
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "your-password"}'
# {"access_token":"<jwt>","token_type":"bearer"}

# Create a user (admin token required)
curl -X POST http://localhost:8000/users \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"username": "sara", "password": "S3cure-Pass!", "role": "viewer"}'
# {"user_id": 7, "username": "sara", "role": "viewer", "is_active": true}
```

### 4. Run the Tests

```bash
pytest
```

**Expected Output:**
```
104 passed, 1 skipped
```

> The tests need **no database and no real secrets**: `tests/conftest.py` injects safe dummy environment variables, and the DB layer is mocked. The single skipped test is an opt-in integration test that requires a live SQL Server.

## 📖 API Reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/health` | None | Liveness probe → `{"status": "ok"}` |
| `POST` | `/auth/login` | None | Exchange credentials for a JWT access token |
| `POST` | `/users` | Admin | Create a user → `201` + `UserOut` |
| `PUT` | `/users/{user_id}` | Admin | Update username / role / `is_active` (all optional) → `204` |
| `PUT` | `/users/{user_id}/password` | Admin | Set a new password → `204` |
| `DELETE` | `/users/{user_id}` | Admin | Deactivate the user (soft delete) → `204` |

### Request / Response Models (`app/schemas/user.py`)

| Model | Fields & Validation |
|-------|---------------------|
| `LoginRequest` | `username` (1–100 chars) · `password` (1–72 chars) |
| `TokenResponse` | `access_token` · `token_type = "bearer"` |
| `UserCreate` | `username` (1–100) · `password` (**8–72** chars) · `role` (`admin` or `viewer`) |
| `UserUpdate` | `username?` · `role?` · `is_active?` (every field optional) |
| `SetPasswordRequest` | `password` (8–72 chars) |
| `UserOut` | `user_id` · `username` · `role` · `is_active` (**never** contains `PasswordHash`) |

### Error Format

All errors share one clean JSON shape:

```json
{ "error": { "message": "You do not have permission to perform this action." } }
```

| Status | Meaning |
|--------|---------|
| `401` | Missing/invalid/expired token, or bad credentials (always `"Invalid credentials."`) |
| `403` | Authenticated, but the role is not allowed to perform the action |
| `404` | Resource not found |
| `422` | Invalid input (Pydantic validation or application-level) |
| `503` | Database temporarily unavailable |
| `500` | Unexpected error (details only in the server log) |

## 🏗️ Architecture Details

### Three-Tier Architecture

```
┌──────────────────────────────┐
│  FRONTEND (React)            │  Login · Report List · Dashboard · Downloads
└──────────────┬───────────────┘
               │  REST API (JSON + Bearer JWT)
┌──────────────▼───────────────┐
│  BACKEND (FastAPI)           │  Authentication & RBAC · Report Execution · PDF/CSV/Excel Export
└──────────────┬───────────────┘
               │  pyodbc → stored procedures / functions only
┌──────────────▼───────────────┐
│  DATABASE (SQL Server)       │  ReportManagementDB: 10 tables · 26 procs · 6 functions
└──────────────────────────────┘
```

### Backend Layering

```
HTTP request
    ↓
api/routes        ← thin handlers, request validation (Pydantic schemas)
    ↓   Depends(require_role / require_permission) → get_current_user → decode JWT
services          ← business rules: hash password, issue token, write audit log
    ↓
repositories      ← 1 function = 1 stored procedure; knows parameter names, nothing else
    ↓
db/proc.py        ← builds `EXEC proc @p = ?` from validated names, binds values
    ↓
db/mssql.py       ← pyodbc connection, timeout, retry, always closed
    ↓
SQL Server        ← usp_* procedures / ufn_* functions (ownership chaining)
```

### Login Flow

```
POST /auth/login {username, password}
    ↓
usp_GetUserForLogin(@Username)      → UserId, PasswordHash, Role, IsActive (or no row)
    ↓
unknown user / inactive / wrong password?  → usp_Log_Add(LOGIN_FAILED) → 401 "Invalid credentials."
    ↓
usp_Log_Add(LOGIN_SUCCESS)
    ↓
create_access_token(sub=UserId, role=Role, exp=now+JWT_EXPIRE_MINUTES)  → 200 + token
```

### Admin-Only Request Flow

```
PUT /users/5  + Authorization: Bearer <jwt>
    ↓
get_current_user   → verify signature & expiry, read sub + role   (401 on failure)
    ↓
require_role("admin")                                            (403 if role ≠ admin)
    ↓
user_service.update_user → users_repo.update_user → usp_User_Update
    ↓
usp_Log_Add(USER_UPDATE, actor = current user)
    ↓
204 No Content
```

### Database Design (`ReportManagementDB`)

| Table | Purpose |
|-------|---------|
| `Users` | Accounts, roles (`admin`/`viewer`), bcrypt hashes, status |
| `Applications` | Registry of the 26 systems (source / consumer / both) |
| `Connections` | Target databases: `connection_type` (`sqlserver`/`oracle`/`db2`) + **encrypted** credentials |
| `Reports` | Central report definition: name, SQL query, connection, owner |
| `ReportFormats` | Output formats per report (`xlsx` / `pdf` / `csv`) |
| `User_Reports` | Many-to-many: which users may access which report |
| `Schedules` | Periodicity (daily / weekly / monthly / cron) and `next_run` |
| `Recipients` | Report-to-email mapping for automated dispatch |
| `ReportRuns` | Execution audit: status, timing, row count, file path, errors |
| `Logs` | User-action audit (login / create / edit / delete / download / execute) |

### Stored Procedures Used by the Backend Today

| Procedure | Parameters | Returns |
|-----------|------------|---------|
| `usp_GetUserForLogin` | `@Username` | 0–1 row: `UserId, Username, PasswordHash, Role, IsActive` |
| `usp_User_Create` | `@Username, @PasswordHash, @Role` | scalar `NewUserId` |
| `usp_User_Update` | `@UserId, @Username, @Role, @Status` | none (`@Status` = `'active'`/`'inactive'`, translated from `is_active`) |
| `usp_User_SetPassword` | `@UserId, @PasswordHash` | none |
| `usp_User_Delete` | `@UserId` | none (sets `Status='inactive'`) |
| `usp_Log_Add` | `@Action, @UserId, @EntityType` | none (`@UserId` nullable for unknown-user failures) |

Other objects already deployed and reserved for later phases: CRUD procedures for applications, connections, reports, schedules and recipients; `usp_GetDueReports` / `ufn_GetDueReports`, `usp_ReportRun_Start`, `usp_ReportRun_Complete`, `usp_AdvanceSchedule`; `ufn_IsActionAllowed`, `ufn_UserHasReportAccess`, `ufn_GetUserReports`, `ufn_GetReportFormats`, `ufn_GetReportRecipients`; `usp_Connection_SetSecret` / `usp_Connection_GetSecret`.

## 🔒 Security Model

| Layer | Control |
|-------|---------|
| **Passwords** | bcrypt hashes only, salted per hash; 72-byte limit enforced explicitly |
| **Tokens** | Signed JWT (HS256) with `sub`, `role`, `exp`; any invalid token → generic 401 |
| **Authorization** | Server-side RBAC on every protected route (`require_role`, `require_permission`) |
| **Enumeration** | Identical error for unknown user / wrong password / inactive account |
| **Stored credentials** | Fernet-encrypted `VARBINARY`; decrypted only at connection time; key validated at startup |
| **SQL injection** | No ORM and no raw SQL: bound parameters only; procedure/parameter names checked against an identifier allowlist |
| **Least privilege** | `app_role` and `automation_role` can only `EXECUTE`; direct `SELECT/INSERT/UPDATE/DELETE` on base tables is `DENY`-ed (ownership chaining lets procedures work) |
| **Error leakage** | Clients get generic messages; details are logged server-side only |
| **Secret hygiene** | Tests assert that passwords, tokens, keys and ciphertext are never written to logs or error messages |
| **Audit** | Login attempts and every user-management action recorded through `usp_Log_Add` |

## 🧪 Testing

**105 tests · 104 passed · 1 skipped** across 16 test files:

| Area | Test files | Tests |
|------|-----------|:-----:|
| Config & logging | `test_config.py`, `test_logging_config.py` | 6 |
| Errors & health | `test_errors.py`, `test_health.py` | 7 |
| DB connectivity & proc caller | `test_mssql.py`, `test_proc.py` | 17 |
| Security core | `test_security.py`, `test_crypto.py` | 16 |
| Auth dependency & RBAC | `test_deps.py`, `test_rbac.py` | 17 |
| Schemas | `test_user_schemas.py` | 10 |
| Repository | `test_users_repo.py` | 10 |
| Services | `test_auth_service.py`, `test_user_service.py` | 10 |
| Routes | `test_auth_routes.py`, `test_users_routes.py` | 12 |

```bash
pytest                                   # whole suite
pytest tests/test_security.py -v         # one module
pytest -k rbac                           # by keyword
```

## 📖 Documentation

### Project Documents
- **[SRS_RE_1.pdf](SRS_RE_1.pdf)**: Software Requirements Specification (15 pages): objectives, three-tier architecture, functional and non-functional requirements, tech stack, schema & RBAC matrix, team structure, optional enhancements, security considerations and the one-month timeline
- **[PROJEC_1.PDF](PROJEC_1.PDF)**: Project status report (5 pages): database design (10 tables, 26 procedures, 6 functions), security model, backend Phases 0–1, test status and the remaining roadmap
- **[PHASE2_AGENT_GUIDE.md](PHASE2_AGENT_GUIDE.md)**: Step-by-step guide for Phase 2 (Security Core) with required functions, architectural rules, tests and completion checklists

### Helper Scripts
- **[connectivity_check.py](connectivity_check.py)**: DB connection, procedure access and `DENY`-grant check
- **[extract_pdf.py](extract_pdf.py)**: Prints the text of the two PDFs (needs `pypdf`)

## 🛠️ Tech Stack

| Layer / Concern | Technology | Purpose |
|-----------------|-----------|---------|
| Language | Python 3.11+ | Backend |
| Web framework | FastAPI + Uvicorn | REST API, dependency injection, OpenAPI docs |
| Validation & settings | Pydantic · pydantic-settings | Request/response models, env-based config |
| Database | Microsoft SQL Server | Users, reports, schedules, audit |
| DB driver | pyodbc (ODBC Driver 17) | Stored-procedure calls |
| Password hashing | bcrypt | Secure password storage |
| Tokens | python-jose (HS256) | JWT creation/verification |
| Encryption | cryptography (Fernet) | Encrypting stored DB credentials |
| Export (planned) | openpyxl · reportlab · csv | Excel / PDF / CSV generation |
| Testing | pytest · httpx | Unit and route tests |
| Frontend (planned) | React | UI, consumes the REST API |

**`requirements.txt`:** `fastapi` · `uvicorn[standard]` · `pydantic` · `pydantic-settings` · `pyodbc` · `python-jose[cryptography]` · `bcrypt` · `cryptography` · `openpyxl` · `reportlab` · `pytest` · `httpx`

> `openpyxl` and `reportlab` are already listed for the upcoming report-generation phase; they are not used by the current code.

## 🗺️ Project Status & Roadmap

| Phase | Focus | Status |
|-------|-------|:------:|
| **0** | Scaffolding: config, logging, error handling, `GET /health` | ✅ Done |
| **1** | SQL Server connectivity & stored-procedure caller | ✅ Done |
| **2** | Security core: bcrypt, JWT, Fernet, startup key validation | ✅ Done |
| **3** | Authentication & user management (login, JWT issuance, user CRUD) | ✅ Done |
| **4** | Authorization / RBAC enforcement | ✅ Done for auth & user routes<sup>†</sup> |
| **5** | Applications & Connections registry (encrypted credentials) | ⏳ Planned |
| **6** | Report & configuration management (reports, formats, recipients, schedules) | ⏳ Planned |
| **7** | Multi-database execution engine (SQL Server / Oracle / DB2) | ⏳ Planned |
| **8** | Report generation (Excel / PDF / CSV) | ⏳ Planned |
| **9** | Automation script, scheduling & PowerShell email dispatch | ⏳ Planned |
| **10** | Cross-cutting hardening, logging/audit, end-to-end integration | ⏳ Planned |

<sup>†</sup> `require_permission` already defines the full action matrix (`manage_reports`, `manage_connections`, `manage_schedules`, `execute_report`, `download_report`); it will be applied to those routes as they are built.

### Planned Enhancements (from the SRS)
- 📈 Dashboard with charts (Chart.js)
- ⏰ Scheduled reports with email delivery
- 🔎 Parameterised reports (safe inputs such as date range / region)
- 📝 Audit-log viewer
- 📄 Search, sort and pagination on the report list · Favourites · Excel export · Dark mode · Docker

## 🔧 Troubleshooting

**`ImportError: libodbc.so.2: cannot open shared object file` (Linux):**
```bash
sudo apt-get install -y unixodbc      # then re-run pytest / uvicorn
```

**`ValidationError` for `Settings` on startup (missing field):**
```bash
# APP_DB_CONN, JWT_SECRET, JWT_EXPIRE_MINUTES and CRED_ENCRYPTION_KEY are all required
cp .env.example .env     # then fill in real values
```

**`ValueError: Invalid encryption key` on import:**
```bash
# CRED_ENCRYPTION_KEY must be a valid Fernet key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

**API returns `503 The database is temporarily unavailable`:**
```bash
# Check the driver name, server, credentials and that SQL Server allows SQL authentication + TCP/IP
python connectivity_check.py
```

**`connectivity_check.py` prints "WARNING: direct SELECT on dbo.Users SUCCEEDED":**
```text
# The DENY grants are not applied to the app login: re-run the roles & grants SQL script.
# The app account must only be able to EXECUTE procedures/functions.
```

**`401 Invalid credentials.` for a valid user:**
```text
# The message is intentionally generic: it is returned for unknown user, wrong password
# and inactive accounts. Check the server log / Logs table (LOGIN_FAILED) for the attempt.
```

**`422` when creating a user:** the password must be **8–72 characters** and the role must be exactly `admin` or `viewer`.

## 🤝 Contributing

This is a team project (Frontend, Database and Backend/Security sub-teams). To contribute:
1. Read the [SRS](SRS_RE_1.pdf) and the [status report](PROJEC_1.PDF)
2. Follow the layering rules: routes → services → repositories → `db/proc.py`
3. **Never** write raw table SQL or interpolate values into SQL text; add a stored procedure and a repository wrapper instead
4. Add tests for every new behavior and keep `pytest` green
5. Open an issue or pull request on GitHub

## 📜 License

Released under the [MIT License](LICENSE).

## 👥 Authors

- **Diaa Salah**: [@DiaaSalah57](https://github.com/DiaaSalah57)
- Project team: 9 members across the Frontend (3), Database (2) and Backend/Security (4) sub-teams

**Date:** July 2026  
**Version:** 1.0 (Backend Phases 0–4)

---

## 🎉 Project Highlights

✅ **Security-first backend**: bcrypt, JWT, Fernet-encrypted credentials, generic error messages  
✅ **Stored-procedure-only data access**: no ORM, no raw table SQL, bound parameters only  
✅ **Least-privilege database model**: `EXECUTE`-only roles with `DENY` on base tables  
✅ **Server-side RBAC**: admin/viewer permissions enforced on every protected route  
✅ **Full audit trail**: logins and user-management actions logged in the database  
✅ **Well-tested**: 104 passing tests that need no database or real secrets  
✅ **Clean layering**: routes → services → repositories → DB layer  

---

**For detailed requirements, see:**
- [Software Requirements Specification](SRS_RE_1.pdf)
- [Project Status Report](PROJEC_1.PDF)
- [Phase 2 Implementation Guide](PHASE2_AGENT_GUIDE.md)

**Quick Start:** `pip install -r requirements.txt` → `cp .env.example .env` → `uvicorn app.main:app --reload --port 8000`
