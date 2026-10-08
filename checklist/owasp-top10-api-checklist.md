# OWASP Top 10:2025 Manual Review Checklist (Flask / JSON API)

Reusable checklist for a manual source-code review of a small Python web API.
Each item: **what to check**, **where to look** (adapt per project), and a status column.

Status values: `PASS` · `FAIL` (finding) · `PARTIAL` · `N/A`

Reference: https://owasp.org/Top10/2025/

---

## A01:2025 Broken Access Control (includes SSRF)

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A01.1 | Every protected route has an auth decorator; no route is accidentally public | `routes.py`, each handler class | | |
| A01.2 | Role checks are enforced server-side (user / admin / super admin), not trusted from the request | role decorators, token payload parsing | | |
| A01.3 | A user cannot change their own role (mass assignment on register / update) | register handler, model constructor | | |
| A01.4 | Users can only read/modify their own records (no IDOR via user ID or email in the request) | handlers taking an ID / email parameter | | |
| A01.5 | Admin search endpoints don't leak more fields than needed (password hashes, internal flags) | user search handler, serializers | | |
| A01.6 | No server-side request to a URL supplied by the user (SSRF) | any `requests` / `urllib` call | | |

## A02:2025 Security Misconfiguration

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A02.1 | Debug mode off by default; cannot be enabled accidentally in prod | `main.py`, env var handling | | |
| A02.2 | Server binds to `127.0.0.1` by default, not `0.0.0.0` | `app.run(...)` | | |
| A02.3 | No hardcoded secrets or default keys; app fails if secrets are missing | config / `auth.py` | | |
| A02.4 | Security headers set (`X-Content-Type-Options`, `Cache-Control: no-store` on token responses) | after_request hooks | | |
| A02.5 | CORS is not wide open (`*`) with credentials | CORS setup, if any | | |

## A03:2025 Software Supply Chain Failures

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A03.1 | Dependencies pinned to exact versions | `requirements.txt` | | |
| A03.2 | No known CVEs in pinned versions (`pip-audit -r requirements.txt`) | `requirements.txt` | | |
| A03.3 | No abandoned / deprecated libraries in security-critical paths | auth and token libraries | | |

## A04:2025 Cryptographic Failures

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A04.1 | Passwords hashed with a slow, salted algorithm (bcrypt / argon2) | user model | | |
| A04.2 | Password comparison uses the library's verify function (constant-time) | login handler, model | | |
| A04.3 | Token signing secrets are long and random; access and refresh secrets differ | `auth.py` | | |
| A04.4 | Tokens have an expiry and expiry is actually checked on use | token serializer / verify function | | |
| A04.5 | Access and refresh tokens can't be swapped for each other (separate keys or a `type` claim) | refresh + auth verify functions | | |

## A05:2025 Injection

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A05.1 | All DB access goes through the ORM or parameterized queries; no f-strings / `%` / `+` in SQL | every `.filter`, `text(...)`, `execute(...)` | | |
| A05.2 | Search filters built from user input (lists, dates) are validated before use | user search handler | | |
| A05.3 | No `eval`, `exec`, `pickle.loads`, `yaml.load` or shell calls on user input | `grep -rn "eval\|exec\|pickle\|subprocess\|os.system"` | | |
| A05.4 | Input is type- and length-checked (username, email, password) | register / login handlers | | |

## A06:2025 Insecure Design

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A06.1 | Rate limiting / lockout on login and register | login handler, extensions | | |
| A06.2 | Password policy: minimum length, rejects empty passwords | register + password reset handlers | | |
| A06.3 | Password change requires the old password | password reset handler | | |
| A06.4 | Changing the password invalidates existing tokens / sessions | password reset handler, blacklist | | |

## A07:2025 Authentication Failures

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A07.1 | Login error doesn't reveal whether the email exists (same message + similar timing) | login handler | | |
| A07.2 | Register doesn't reveal existing emails / usernames more than necessary | register handler | | |
| A07.3 | Logout actually invalidates the token(s) it claims to | logout handler, blacklist model | | |
| A07.4 | Blacklisted / revoked tokens are rejected everywhere they're checked | auth verify + refresh handler | | |
| A07.5 | Refresh token can't be reused after it's been rotated or revoked | refresh handler | | |

## A08:2025 Software or Data Integrity Failures

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A08.1 | Token signature always verified; no "none" algorithm or unsigned fallback | token loading code | | |
| A08.2 | Data from the client isn't trusted for integrity-critical fields (role, user ID) | handlers, token payload | | |

## A09:2025 Security Logging and Alerting Failures

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A09.1 | Failed logins and permission denials are logged | login handler, role decorators | | |
| A09.2 | Logs never contain passwords or full tokens | all `logging` / `print` calls | | |

## A10:2025 Mishandling of Exceptional Conditions

| # | Check | Where to look | Status | Notes |
|---|-------|---------------|--------|-------|
| A10.1 | Exceptions don't return stack traces or internal details to the client | error handlers, bare `except` | | |
| A10.2 | Malformed JSON / missing fields return 400, not 500 | every handler reading `request.json` | | |
| A10.3 | Auth code fails closed: an exception in a check means "deny", never "allow" | role decorators, token verify | | |
| A10.4 | No bare `except: pass` that hides security-relevant errors | `grep -rn "except"` | | |

---

## Summary (fill in after the review)

| Severity | Count |
|----------|-------|
| High | |
| Medium | |
| Low | |
| Info | |

Reviewed project: ______ · Commit: ______ · Date: ______ · Reviewer: ______
