# Spec: Registration

## Overview
Turn the static `/register` page into a working sign-up flow. A visitor submits name, email and password; the app validates the input, stores a new row in `users` with a hashed password, and sends them to the login page. This is the first write path into the data layer built in Step 1 and is required before login/logout (Step 3) and profile (Step 4) can exist.

## Depends on
- Step 1 — Database setup (`users` table, `get_db()`, `init_db()`)

## Routes
- `GET /register` — render the empty registration form — public (already exists)
- `POST /register` — validate input, create the user, redirect to `/login` on success; re-render the form with an error message on failure — public

The existing `/register` route gains `methods=["GET", "POST"]`. All other routes are unchanged.

## Database changes
No database changes. The `users` table already has `name`, `email` (UNIQUE), `password_hash` and `created_at`.

New helpers in `database/db.py` (DB logic must not live in routes):
- `create_user(name, email, password)` — hashes the password with `generate_password_hash`, inserts the row with a parameterised query, returns the new user id; lets `sqlite3.IntegrityError` propagate on a duplicate email
- `get_user_by_email(email)` — returns the matching `sqlite3.Row` or `None`

## Templates
- **Create:** none
- **Modify:** `templates/register.html`
  - form `action` must use `{{ url_for('register') }}` instead of the hardcoded `/register`
  - re-populate `name` and `email` inputs from the submitted values after a failed attempt (never the password)
  - add `minlength="8"` to the password input

## Files to change
- `app.py` — accept POST on `/register`, validate, call the new db helpers, redirect or re-render with `error`
- `database/db.py` — add `create_user()` and `get_user_by_email()`
- `templates/register.html` — changes listed above

## Files to create
- None

## New dependencies
No new dependencies.

## Rules for implementation
- No SQLAlchemy or ORMs
- Parameterised queries only
- Passwords hashed with werkzeug (`generate_password_hash`); never store or log plaintext
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- Reuse the existing `.auth-error` block in `register.html` to show errors; no new CSS unless unavoidable
- Validation (server side, in this order), each failure re-renders the form with HTTP 200 and an `error` string:
  - name, email and password are all present after stripping whitespace
  - email contains `@` and a `.` after it (basic format check)
  - password is at least 8 characters
  - email is not already registered (catch `sqlite3.IntegrityError` rather than only pre-checking, to avoid a race)
- Normalise email to lowercase and strip whitespace before saving and comparing
- No session or login on registration; do not add `secret_key` or `session` in this step
- Do not implement the `/login` POST handler or any other stub route
- Route function stays thin: parse form, call db helpers, render or redirect

## Definition of done
- [ ] `GET /register` still renders the form with status 200
- [ ] The form posts via `url_for('register')` (no hardcoded URL in the template)
- [ ] Submitting valid details creates a row in `users` and redirects (302) to `/login`
- [ ] The stored `password_hash` is a werkzeug hash, not the plaintext password
- [ ] Registering with an email that already exists (including different casing, e.g. `Demo@Spendly.com`) shows an error and creates no extra row
- [ ] Empty fields, an invalid email, or a password under 8 characters each show a clear error and create no row
- [ ] After a failed attempt the name and email fields keep their values and the password field is empty
- [ ] The seeded demo user (`demo@spendly.com`) is unaffected
- [ ] The app starts without errors and the other pages (`/`, `/login`, `/terms`, `/privacy`) still return 200
