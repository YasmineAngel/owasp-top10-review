# OWASP Top 10:2025 Manual Review: flask-restful-login

A manual security review of [melihcolpan/flask-restful-login](https://github.com/melihcolpan/flask-restful-login),
a small Flask REST API with token authentication and role-based access control, against the
**OWASP Top 10:2025**. Findings were verified live, fixed where safe to do so, and contributed
back upstream.

**Pull request:** [Security hardening: respect DEBUG setting and enforce password policy](https://github.com/melihcolpan/flask-restful-login/pull/374)

## Scope

| | |
|---|---|
| Target | `melihcolpan/flask-restful-login` @ `38d97a8` |
| Stack | Flask 3.1, Flask-RESTful, Flask-HTTPAuth, SQLAlchemy 2.1, itsdangerous, bcrypt, Werkzeug 3.1.9 |
| Method | Manual source review against a custom checklist, live verification on a local instance, dependency audit with `pip-audit` |
| Date | 8 October 2026 |

## Results

| ID | Finding | OWASP 2025 | Severity | Status |
|----|---------|------------|----------|--------|
| F-01 | Debug mode and Werkzeug debugger always enabled, ignoring the `DEBUG` setting | A02 Security Misconfiguration | Medium | Fixed in PR |
| F-02 | Access control issue (details withheld) | A01 Broken Access Control | High | Reported privately to the maintainer, fix validated locally |
| F-03 | Empty and short passwords accepted at registration and password reset | A07 Authentication Failures | Medium | Fixed in PR |

Additional observations (lower severity, documented in [`findings/findings.md`](findings/findings.md)):
logout only revokes the refresh token, password changes don't invalidate existing tokens,
no rate limiting on login, response timing can reveal registered emails, non-standard HTTP status
codes (`999`) on some errors.

Checks that **passed**: parameterized queries through the ORM (no SQL injection), bcrypt password
hashing, secrets loaded from the environment with fail-fast startup, separate signing keys for access
and refresh tokens, password hashes excluded from API responses, pinned dependencies with no known
CVEs (`pip-audit`).

## Highlights

**A dependency upgrade silently disabled a security control (F-02).** The vulnerable code was
correct when written. A change in a library's behavior made the check stop running, with no error
and no failing test. A scanner looking for known-bad patterns would not flag it; it only shows up by
reading the logic and testing it live. The check also failed *open*, so when it stopped running,
requests were allowed instead of denied.

**The configuration looked right but wasn't applied (F-01).** The code read `DEBUG` correctly and
defaulted to `False`, matching the README, then a later line overrode it. Verified by starting the
server with no `DEBUG` set and observing the debugger come up.

**Responsible disclosure.** The project has no `SECURITY.md` and private vulnerability reporting is
disabled, so F-02 was reported to the maintainer by email with reproduction steps, root cause, a
tested patch and a 90-day disclosure window. Only the two low-risk hardening fixes went into the
public pull request. Full details of F-02 will be added here once a fix is published.

## Fix validation

| Finding | Before | After |
|---------|--------|-------|
| F-01, no `DEBUG` set | Debug mode on, debugger active | Debug mode off |
| F-01, `DEBUG=true` | On (setting ignored) | On, as requested |
| F-02, `user` role on admin routes | Data returned | Permission denied (admin access unaffected) |
| F-03, empty / 5-character password | Account created | `422 Invalid input` |

Test suite: 11 tests before, 15 after (4 new regression tests), all passing.

## Repository contents

- [`checklist/owasp-top10-api-checklist.md`](checklist/owasp-top10-api-checklist.md): reusable 37-item
  OWASP Top 10:2025 checklist for Python JSON APIs, each item with where to look in the code
- [`findings/findings.md`](findings/findings.md): detailed findings with evidence, impact and fixes

## What I'd check next

Rate limiting and account lockout on `/login`, token revocation on logout and password change,
and a review of the token format (itsdangerous signed payloads vs. standard JWTs with a `type` claim).
