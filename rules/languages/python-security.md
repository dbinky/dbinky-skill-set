# Python Security Review Rules

## Input Validation

- Use Pydantic models for all API request validation in FastAPI/Django REST Framework.
- Never trust `request.GET`/`request.POST` values without validation and type coercion.
- Validate file uploads: check MIME type, file extension, and file size. Never trust `Content-Type` header alone.
- Use `bleach` or `nh3` to sanitize HTML input. Never allow raw HTML from users.
- For Django forms, always use `form.is_valid()` before accessing `cleaned_data`.
- Limit request body size at the ASGI/WSGI server level (gunicorn, uvicorn config).

## Authentication/Authorization

- Django: use `@login_required` and `@permission_required` decorators, never manual session checks.
- FastAPI: use `Depends()` with security dependencies for every protected route.
- Never store passwords in plaintext. Use Django's `make_password`/`check_password` or `bcrypt`/`argon2-cffi`.
- JWT: validate expiration, issuer, audience. Use `PyJWT` with explicit algorithm specification (`algorithms=["RS256"]`).
- Flag: `algorithm="none"` or `algorithms=["HS256", "none"]` in JWT verification.
- Session cookies: set `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE`.

## SQL/NoSQL Injection

- Always use ORM query methods or parameterized queries. Never use f-strings or `%` formatting in SQL.
- Flag: `cursor.execute(f"SELECT ... WHERE id = {user_id}")` or any string interpolation in SQL.
- Django ORM: use `.filter()`, `.exclude()`, never `.raw()` or `.extra()` with user input.
- SQLAlchemy: use bound parameters `text("SELECT * FROM t WHERE id = :id").bindparams(id=val)`.
- For MongoDB (pymongo): never pass `json.loads(user_input)` directly as a query filter.

## XSS Prevention

- Django templates auto-escape by default. Flag any use of `|safe` or `mark_safe()` with user data.
- Jinja2: ensure `autoescape=True` is set. Flag `Markup()` wrapping user content.
- Set `Content-Security-Policy` headers via middleware.
- For API responses, always return `application/json` content type, never `text/html`.
- Sanitize user content before storing, not just before rendering.

## Secrets Management

- Never hardcode secrets. Use environment variables with `os.environ.get()` or `python-decouple`.
- Flag: any string literal matching patterns like API keys, passwords, tokens, or connection strings.
- Use `.env` files for local development only; ensure `.env` is in `.gitignore`.
- In Django, move `SECRET_KEY` to environment variables; never commit the default.
- Use `secrets` module for generating tokens, not `random`.

## Dependency Vulnerabilities

- Run `pip-audit` or `safety check` in CI against the locked dependency set.
- Pin all dependencies with exact versions in lock files.
- Review `requirements.txt` or `poetry.lock` diffs in PRs for unexpected package additions.
- Avoid packages with no maintenance activity, very low download counts, or typo-squatting names.
- Use `--require-hashes` with pip for supply chain integrity.

## Deserialization

- Never use `pickle.loads()` or `pickle.load()` on untrusted data. It enables arbitrary code execution.
- Flag: `yaml.load()` without `Loader=yaml.SafeLoader`. Use `yaml.safe_load()`.
- Flag: `eval()`, `exec()`, `compile()` with any user-controlled input.
- For JSON, use `json.loads()` with subsequent Pydantic/dataclass validation.
- Flag: `marshal.loads()` or `shelve.open()` with untrusted data.

## File System Access

- Use `pathlib.Path.resolve()` and verify the result is under the allowed root directory.
- Never use user input directly in `open()`, `os.path.join()`, or `shutil` operations.
- Flag: `os.path.join(base, user_input)` without path traversal check (leading `/` bypasses join).
- Set `MEDIA_ROOT` and `STATIC_ROOT` in Django; validate uploads go only to those directories.
- Use `tempfile.mkstemp()` or `tempfile.TemporaryDirectory()` for temporary files.

## Cryptography

- Use `cryptography` library, not `pycrypto` (unmaintained) or `pycryptodome` for new projects.
- Hash passwords with `argon2-cffi` or `bcrypt`. Never use `hashlib` for passwords.
- Use `secrets.token_urlsafe()` for secure random tokens.
- For encryption, use `Fernet` (symmetric) from `cryptography` or `RSA-OAEP` (asymmetric).
- Never use MD5 or SHA1 for anything security-sensitive.

## Common CVE Patterns

- Server-Side Template Injection (SSTI): never render user input as a Jinja2 template string.
- SSRF: validate URLs before `requests.get(user_url)`. Block internal IP ranges.
- ReDoS: avoid user-controlled regex patterns or validate complexity. Use `re2` for untrusted patterns.
- XML External Entity (XXE): use `defusedxml` instead of `xml.etree.ElementTree` for untrusted XML.
- Zip bomb: limit extraction size and file count when using `zipfile` with untrusted archives.
- `subprocess` injection: always pass command as a list, never a string with `shell=True`.
