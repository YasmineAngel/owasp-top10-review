# Findings: melihcolpan/flask-restful-login @ 38d97a8

| | |
|---|---|
| Target | [melihcolpan/flask-restful-login](https://github.com/melihcolpan/flask-restful-login) @ `38d97a8` |
| Date | 2026-10-08 |
| Method | Manual source review with [`checklist/owasp-top10-api-checklist.md`](../checklist/owasp-top10-api-checklist.md), live verification on a local instance, `pip-audit` |
| Environment | Windows, Python 3.13, dependencies from the pinned `requirements.txt` (Werkzeug 3.1.9, Flask 3.1.3) |
| Pull request | [#374](https://github.com/melihcolpan/flask-restful-login/pull/374) (F-01, F-03) |

## Summary

| ID | Title | OWASP 2025 | Severity | Status |
|----|-------|------------|----------|--------|
| F-01 | Debug mode and Werkzeug debugger always enabled | A02 | Medium | Fixed, PR #374 merged |
| F-02 | Role checks bypassed: any user gets admin access | A01 | High | Disclosed privately, fixed upstream |
| F-03 | Empty and short passwords accepted | A07 | Medium | Fixed, PR #374 merged |

Plus 10 lower-severity observations (O-01 to O-10) from code review.

---

## F-01: Debug mode and Werkzeug debugger always enabled

- **OWASP:** A02:2025 Security Misconfiguration (checklist A02.1)
- **Severity:** Medium
- **Location:** `main.py`, `app.run(...)` call
- **Description:** `create_app()` reads `DEBUG` from the environment (default `False`),
  but `app.run()` hardcodes `debug=True` and `use_reloader=True`, overriding it.
  The README states debug defaults to `False`.
- **Evidence:** Started with no `DEBUG` variable set (confirmed with
  `Get-ChildItem env:`); server printed `Debug mode: on` and `Debugger is active!`.
- **Impact:** The Werkzeug debugger allows arbitrary Python execution from the error
  page (PIN-protected). Binding to `localhost` limits exposure, but anyone changing
  `host` or putting a proxy in front would expose it.
- **Fix:** Pass `app.config['DEBUG']` to `app.run()` for both `debug` and `use_reloader`.
- **Fix validation:**

  | Run | Before | After |
  |-----|--------|-------|
  | No `DEBUG` set | Debug mode on, debugger active | Debug mode off |
  | `DEBUG=true` | On (setting ignored) | On, as requested |

- **Status:** Fixed in merged [PR #374](https://github.com/melihcolpan/flask-restful-login/pull/374), commit `ef26cef`

## F-02: Role checks bypassed, any authenticated user gets admin and super-admin access

- **OWASP:** A01:2025 Broken Access Control (checklist A01.2, A10.3)
- **Severity:** High
- **Location:** `api/roles/role_required.py`, `permission()` decorator
- **Description:** The role check only runs when `request.authorization is None`.
  Since Werkzeug 2.3, `request.authorization` also parses Bearer tokens, so with the
  pinned Werkzeug 3.1.9 it is never `None` for a Bearer request. The check is skipped
  and the decorator falls through to the handler. It also fails open: any request
  that doesn't enter the `if` block is allowed.
- **Evidence:** Logged in as `alice` (role `user`); `GET /data_admin`,
  `GET /data_super_admin` and `GET /users` all returned data.
- **Impact:** Vertical privilege escalation; any registered user can list other
  users' usernames and emails and reach every admin-only route.
- **Fix:** Read the token from `request.authorization.token`, deny by default
  (401 if missing or invalid, 403 if the role is too low).
- **Fix validation (local only, not pushed):**

  | Request | Before | After |
  |---------|--------|-------|
  | `user` → `/data_admin`, `/data_super_admin`, `/users` | Data returned | Permission denied |
  | `admin` → `/data_admin` | Allowed | Allowed |
  | `admin` → `/data_super_admin` | Allowed (bug) | Permission denied |

- **Status:** Reported privately to the maintainer by email on 2026-10-08, with
  reproduction steps, root cause and a tested patch. The maintainer reproduced it and fixed it
  the same day in [`328333f`](https://github.com/melihcolpan/flask-restful-login/commit/328333f):
  the decorator now reads the token from `request.authorization` and denies by default
  (401 for a missing or invalid token, 403 for an insufficient role), with regression tests
  for each role. The same decorator had been copied into the maintainer's
  `flask-login-example` project, where it exposed the user list; fixed there in `9a95f70`.
  Both commits credit the reporter. Published with the maintainer's agreement.

## F-03: Empty and short passwords accepted at registration and password reset

- **OWASP:** A07:2025 Authentication Failures (checklist A06.2, A05.4)
- **Severity:** Medium
- **Location:** `api/handlers/UserHandlers.py`, `Register.post()` and `ResetPassword.post()`
- **Description:** Register inputs are `.strip()`-ed, then compared to `None`. An empty
  string is not `None`, so empty username, email and password all pass. There is no
  minimum length. `ResetPassword` doesn't validate `new_pass` at all.
- **Evidence:** `POST /v1/auth/register` with `"password": ""` returned
  `registration completed.`
- **Impact:** Accounts with empty or trivially guessable passwords; combined with the
  lack of rate limiting (O-03), these are easy to brute-force.
- **Fix:** Reject empty fields; enforce a minimum password length of 8 (NIST SP 800-63B)
  on both registration and password reset. Four regression tests added.
- **Fix validation:** Test suite went from 11 to 15 tests, all passing. New tests cover
  empty password, 5-character password, whitespace-only username, and short password on reset.
- **Status:** Fixed in merged [PR #374](https://github.com/melihcolpan/flask-restful-login/pull/374), commit `73868f2`

---

## Lower-severity observations

Identified by reading the code; not live-tested unless stated.

| ID | Observation | OWASP 2025 | Severity |
|----|-------------|------------|----------|
| O-01 | `Logout` only blacklists the refresh token; the access token stays valid for up to 1 hour. It also doesn't check that the refresh token belongs to the caller. | A07 | Low |
| O-02 | Changing the password doesn't invalidate existing access or refresh tokens. | A07 | Low |
| O-03 | No rate limiting or lockout on `/login` and `/register` (acknowledged in the README). | A06 | Low |
| O-04 | User enumeration: `Login` only runs bcrypt when the email exists, so response time differs; `Register` returns `409 Already exists` for known emails. | A07 | Low |
| O-05 | `RefreshToken` doesn't check the user still exists, and always issues a role-0 token, so admins lose their role after a refresh. | A07 | Low |
| O-06 | Some errors use HTTP status `999` (`NOT_ADMIN`, `HEADER_NOT_FOUND`), which is not a valid status code. | A10 | Low |
| O-07 | `db_initializer.py` contains hardcoded credentials (`sa_password`, `admin_password`); not called by the app at this commit, but easy to wire in by mistake. | A02 | Info |
| O-08 | No `Cache-Control: no-store` on responses that return tokens; no security headers. | A02 | Info |
| O-09 | `verify_auth_token` checks `if "email" and "admin" in data`, which only tests `"admin"`. Harmless because tokens are signed, but the intent is wrong. | A10 | Info |
| O-10 | `UsersData` uses `print()` to log query parameters (emails) to stdout instead of the logger. | A09 | Info |

## Checks passed

- **A05 Injection:** all database access goes through the SQLAlchemy ORM (`filter_by`, `in_`, `between`); no raw SQL, `eval`, `pickle` or shell calls.
- **A04 Cryptographic Failures:** passwords hashed with bcrypt (`gensalt()`), verified with `checkpw`; access and refresh tokens signed with separate secrets and time-limited (1h / 2h).
- **A02 Security Misconfiguration:** secrets loaded from the environment, app refuses to start without them; server binds to `localhost`.
- **A01 Broken Access Control:** `UserSchema` excludes the password hash from `/users` responses.
- **A03 Software Supply Chain Failures:** all dependencies pinned with `==`; `pip-audit -r requirements.txt` reported no known vulnerabilities. Note: Flask-RESTful 0.3.10 receives little maintenance.