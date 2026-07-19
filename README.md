# HR SaaS Backend (Multi-Tenant)

Spring Boot backend for a multi-tenant HR management SaaS. Any company can
register its own workspace ("tenant"), get an Admin account, and manage its
own Employees — leave requests, attendance, departments — completely
isolated from every other company's data.

## Decisions I made that you should know about

You asked "Mongo or Postgres, Postgres stresses me out" — I went with
**Postgres**. HR data is relational by nature (employees → departments,
leave requests → employees → reviewers). Postgres + Spring Data JPA is
the more standard, better-documented combination for this domain, and
Flyway migrations (included, in `src/main/resources/db/migration/`) mean
you never write raw SQL by hand day-to-day — just run the app and the
schema builds itself. If this still feels heavy, tell me and I can convert
it, but I'd genuinely recommend sticking with Postgres here.

You asked for **Java 20** — Java 20 is no longer distributed (it was a
short-term release, EOL'd months after launch). I used **Java 21 (LTS)**
instead, since it's the direct, safe successor and Spring Boot 3.3 targets
LTS versions. If you specifically need 20 for a course requirement, let me
know and I'll adjust the `pom.xml`, but I'd advise against it for anything
real.

I could not run `mvn` in this environment (no Maven/network access to
Maven Central here), so **this project has not been compiled or run in
this sandbox**. I wrote it carefully, but treat the first build on your
machine as the real test — see Troubleshooting below if something doesn't
compile.

## Multi-Tenancy Model

Approach: **shared database, shared schema, tenant_id column** on every
tenant-scoped table (`companies.id` is the tenant id). This is the
simplest, most common multi-tenant SaaS pattern and avoids the operational
overhead of schema-per-tenant or database-per-tenant.

Every tenant-scoped query is manually filtered by `company_id` in the
repository/service layer (see `TenantContext` + service classes) — nothing
relies on "the query happens to be scoped correctly," each repository
method takes `companyId` explicitly, sourced from the authenticated user's
JWT, never from client input.

### Tenant Registration Flow

1. `POST /api/auth/register-company` — a company submits its name +
   details + the first admin's name/email/password, in one request.
2. Backend creates a `Company` row, generates a unique URL-safe `slug`
   from the company name (e.g. "Acme Inc" → `acme-inc`, with `-1`, `-2`
   suffixes on collision).
3. Backend creates the first `User` with role `ADMIN`, status `ACTIVE`,
   tied to that company.
4. A welcome email is sent (async, via Gmail SMTP) confirming the
   workspace and slug.
5. Backend returns a JWT immediately — the admin is logged in right away,
   no separate login step needed after registration.

The `slug` doubles as the tenant identifier employees/admins must supply
at login (`POST /api/auth/login` takes `companySlug` + `email` +
`password`), so two different companies can each have a user with the
same email address without collision.

### Employee Onboarding Flow

Employees do **not** self-register (a stranger shouldn't be able to join
a random company). Instead:

1. Admin calls `POST /api/admin/employees` with the employee's details.
2. Backend creates a `User` with role `EMPLOYEE`, status `PENDING` (no
   password yet), and an `Invitation` record with a random token,
   expiring in 72 hours (configurable via `INVITE_EXPIRATION_HOURS`).
3. An email is sent to the employee with a link:
   `{FRONTEND_BASE_URL}/accept-invite?token=...`
4. Employee's frontend calls `POST /api/auth/accept-invitation` with the
   token + a chosen password. Backend sets the password hash and flips
   status to `ACTIVE`.
5. Employee can now log in normally.

**You'll need to build the frontend page that reads the `token` query
param and posts it** — this backend only exposes the API.

## Tech Stack

- Java 21, Spring Boot 3.3.4
- Spring Web, Spring Security (JWT, stateless), Spring Data JPA
- PostgreSQL + Flyway migrations
- Spring Mail (Gmail SMTP)
- Lombok
- `spring-dotenv` to load `.env` automatically (no extra tooling needed)

## Project Structure

```
src/main/java/com/hrsaas/
  config/       Spring configuration (Security, CORS)
  controller/   REST controllers (AuthController, AdminController, EmployeeController)
  dto/          Request/response payloads
  entity/       JPA entities
  enums/        Role, UserStatus, LeaveType, LeaveStatus, AttendanceStatus
  exception/    ApiException + global handler
  repository/   Spring Data JPA repositories
  security/     JwtService, JwtAuthenticationFilter, AuthenticatedUser
  service/      Business logic (AuthService, EmployeeService, LeaveService, AttendanceService, DepartmentService, MailService)
  tenant/       TenantContext (thread-local, holds current tenant + user id)
src/main/resources/
  application.yml
  db/migration/V1__init_schema.sql
```

## Setup

### 1. Start PostgreSQL

Easiest option — Docker Compose is included:

```bash
docker compose up -d
```

This starts Postgres on `localhost:5432` with database `hr_saas`,
user `postgres`, password `postgres` (matches `.env.example` defaults).

Or, if you already have Postgres installed natively:

```bash
createdb hr_saas
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Then edit `.env`:

- `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` — your Postgres connection.
- `MAIL_USERNAME` — your Gmail address.
- `MAIL_APP_PASSWORD` — **not your normal Gmail password.** Go to your
  Google Account → Security → 2-Step Verification (must be enabled) →
  App Passwords → generate one for "Mail". Paste the 16-character code
  here.
- `JWT_SECRET` — any long random string, at least 32 characters. Generate
  one with `openssl rand -base64 48`.
- `CORS_ALLOWED_ORIGINS` — comma-separated list of frontend origins
  allowed to call this API, e.g. `http://localhost:3000,https://app.yoursite.com`.
- `FRONTEND_BASE_URL` — used to build the invitation link sent in emails.

### 3. Run

```bash
mvn spring-boot:run
```

Flyway will automatically create all tables on first run
(`ddl-auto: validate` means Hibernate never silently changes your schema —
Flyway migrations are the only source of schema truth, which is safer for
production).

### 4. Stopping / resetting the local database

```bash
docker compose down          # stop, keep data
docker compose down -v       # stop and wipe data (fresh start)
```

## API Overview

### Public

- `POST /api/auth/register-company` — tenant registration
- `POST /api/auth/login` — requires `email`, `password`, `companySlug`
- `POST /api/auth/accept-invitation` — requires `token`, `password`
- `POST /api/auth/refresh` — requires `refreshToken`; rotates it (old one
  is revoked, a new access + refresh token pair is returned)
- `POST /api/auth/forgot-password` — requires `email`, `companySlug`;
  always returns 200 regardless of whether the account exists, to avoid
  leaking which emails are registered
- `POST /api/auth/reset-password` — requires `token`, `newPassword`;
  invalidates all of that user's existing refresh tokens on success

### Admin only (`ROLE_ADMIN`)

- `POST /api/admin/employees` — create + invite an employee
- `GET /api/admin/employees` — paginated list
- `GET/PUT /api/admin/employees/{id}`
- `PATCH /api/admin/employees/{id}/deactivate` / `/reactivate`
- `POST /api/admin/departments`, `GET /api/admin/departments`, `DELETE /api/admin/departments/{id}`
- `GET /api/admin/leave-requests` — all leave requests in the company
- `PATCH /api/admin/leave-requests/{id}/review` — `{ "approve": true/false, "note": "..." }`
- `GET /api/admin/attendance` — company-wide attendance

### Employee (`ROLE_ADMIN` or `ROLE_EMPLOYEE`)

- `GET /api/employee/me`
- `POST /api/employee/leave-requests`
- `GET /api/employee/leave-requests`
- `PATCH /api/employee/leave-requests/{id}/cancel`
- `POST /api/employee/attendance/check-in`
- `POST /api/employee/attendance/check-out`
- `GET /api/employee/attendance`

All authenticated routes require `Authorization: Bearer <accessToken>`.

## Things I was not sure about / assumptions I made

1. **Refresh token rotation is now implemented** (`POST /api/auth/refresh`):
   each use revokes the old refresh token and issues a fresh
   access+refresh pair. If you'd rather keep a refresh token valid across
   multiple uses until it naturally expires (simpler, slightly less
   secure), tell me and I'll switch the strategy.
2. **Password reset is now implemented** (`POST /api/auth/forgot-password`
   → email link → `POST /api/auth/reset-password`), same token pattern as
   invitations, 1-hour expiry. Resetting a password revokes all of that
   user's existing refresh tokens (forces re-login on other devices) —
   let me know if you'd rather not do that.
3. **Docker Compose file is now included** (`docker-compose.yml`) for a
   local Postgres instance — run `docker compose up -d` before your first
   `mvn spring-boot:run`.
4. **Employees cannot self-register**, only admins invite them. If you
   actually want open self-signup with email-domain matching (e.g.
   `@acme.com` auto-joins the Acme tenant), that's a different, riskier
   flow — let me know if you want it instead.
5. **No payroll module.** You said "HR management" broadly — I built
   Employees, Departments, Leave Requests, and Attendance as the core,
   since payroll is a much bigger, jurisdiction-specific domain (tax
   rules, currencies, pay stubs). Happy to scope that out separately.
6. **Rate limiting / brute-force protection on login** is not implemented.
   For a real production SaaS you'll want this (e.g. Bucket4j or a
   reverse-proxy-level limiter). Flag if you want it built in.
7. I did **not** run this code — no Maven binary was available in the
   sandbox I built it in. Double check the first `mvn clean install` on
   your machine; if you hit compile errors, paste them back to me and
   I'll fix immediately.

## Troubleshooting

- **Flyway checksum error on restart**: if you edit `V1__init_schema.sql`
  after already running it once, Flyway will refuse to run. Either drop
  the database and recreate, or add a new `V2__...sql` migration instead
  of editing V1.
- **Mail fails silently**: check `MAIL_APP_PASSWORD` is an App Password,
  not your login password — Gmail rejects normal passwords for SMTP.
- **401 on every request**: confirm `Authorization: Bearer <token>` header
  is present and the JWT hasn't expired (`JWT_ACCESS_EXPIRATION_MS`,
  default 1 hour).
