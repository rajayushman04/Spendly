# Spec: Login and Logout

## Overview
Turn the static `/login` page into a working sign-in flow and replace the `/logout` stub with a real sign-out. A visitor submits email and password; the app checks them against the hashed password stored in `users`, starts a Flask session, and redirects them into the app. Logout clears the session. This is the first use of sessions in Spendly and is required before the profile page (Step 4) and the expense routes (Steps 7–9) can be restricted to logged-in users.

## Depends on
- Step 1 — Database setup (`users` table, `get_db()`)
- Step 2 — Registration (`create_user()`, `get_user_by_email()`, redirect to `/login`)

## Routes
- `GET /login` — render the sign-in form; if already logged in, redirect to `/profile` — public (already exists)
- `POST /login` — validate credentials, store the user in the session, redirect to `/profile` on success; re-render the form with an error on failure — public
- `GET /logout` — clear the session and redirect to `/` — logged-in (a logged-out visitor is simply redirected to `/`)

The existing `/login` route gains `methods=["GET", "POST"]`. `/profile` stays a stub (Step 4); it is only the redirect target. All other routes are unchanged.

## Database changes
No database changes. The `users` table already has `email` (UNIQUE) and `password_hash`.

No new helpers are needed: `get_user_by_email()` already exists and returns a `sqlite3.Row` or `None`. Password verification uses werkzeug's `check_password_hash` in the route layer (or a thin helper in `database/db.py` if preferred); do not assume other db helpers exist.

## Templates
- **Create:** none
- **Modify:**
  - `templates/login.html`
    - form `action` must use `{{ url_for('login') }}` instead of the hardcoded `/login`
    - re-populate the `email` input from the submitted value after a failed attempt (never the password)
  - `templates/base.html`
    - navbar shows "Sign in" / "Get started" when logged out, and "Sign out" (linking to `url_for('logout')`) when `session.user_id` is set
    - footer links to `/terms` and `/privacy` should use `url_for()`

## Files to change
- `app.py` — set `app.secret_key`, accept POST on `/login`, implement `/logout`, import `session` and `check_password_hash`
- `templates/login.html` — changes listed above
- `templates/base.html` — conditional navbar links

## Files to create
- None

## New dependencies
No new dependencies. Flask sessions and `werkzeug.security.check_password_hash` are already available.

## Rules for implementation
- No SQLAlchemy or ORMs
- Parameterised queries only
- Passwords hashed with werkzeug; verify with `check_password_hash`, never compare plaintext, never log passwords
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- Reuse the existing `.auth-error` block in `login.html` for errors; no new CSS unless unavoidable (a style for the "Sign out" nav link, if needed, goes in `static/css/style.css` using existing variables)
- Secret key: read from the `SECRET_KEY` environment variable with a clearly labelled dev-only fallback; never commit a real secret
- Normalise email (strip, lowercase) before lookup
- Use one generic error for both unknown email and wrong password ("Invalid email or password.") so accounts cannot be enumerated; re-render with HTTP 200
- Require both fields to be non-empty after stripping; otherwise show an error
- On success store only `session["user_id"]` (and optionally `session["user_name"]`); never store the hash
- Logout uses `session.clear()` and redirects with `url_for('landing')`
- Do not implement `/profile` or any other stub route; do not add a login-required decorator beyond what these routes need
- Do not open-redirect: the post-login redirect target is fixed (`url_for('profile')`), not taken from a query parameter
- Route functions stay thin: parse form, call db helper, set session, render or redirect

## Definition of done
- [ ] `GET /login` renders the form with status 200 when logged out
- [ ] The form posts via `url_for('login')` (no hardcoded URL in the template)
- [ ] Logging in as `demo@spendly.com` / `demo123` redirects (302) to `/profile` and sets a session cookie
- [ ] Email matching is case-insensitive (`Demo@Spendly.com` also works) and surrounding whitespace is ignored
- [ ] A wrong password and an unknown email both show the same "Invalid email or password." error, with no session set
- [ ] Empty fields show an error and no session is set
- [ ] After a failed attempt the email field keeps its value and the password field is empty
- [ ] A user registered through `/register` can log in with the password they chose
- [ ] Visiting `/login` while logged in redirects to `/profile`
- [ ] `GET /logout` clears the session and redirects (302) to `/`; afterwards the navbar shows "Sign in" again
- [ ] The navbar shows "Sign out" when logged in and "Sign in" / "Get started" when logged out
- [ ] The app starts without errors and `/`, `/register`, `/terms`, `/privacy` still return 200
