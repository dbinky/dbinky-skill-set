# React Security Review Rules

## XSS via dangerouslySetInnerHTML

- Flag every use of `dangerouslySetInnerHTML`. Require a comment justifying it and proof that input is sanitized.
- If `dangerouslySetInnerHTML` is necessary, verify the value is sanitized with DOMPurify or equivalent. Flag raw user input.
- Flag template literals injected into `dangerouslySetInnerHTML` — interpolated values may contain scripts.
- Flag `innerHTML` set via refs (`ref.current.innerHTML = ...`) — this bypasses React's XSS protection entirely.
- React auto-escapes JSX expressions (`{userInput}`) — but flag `href`, `src`, and `action` attributes with user input (e.g., `href={userInput}` allows `javascript:` URLs).
- Flag `javascript:` protocol in any dynamic URL. Validate URL schemes explicitly.

## Client-Side Auth Token Storage

- Flag tokens stored in `localStorage` — accessible to any JS on the page (XSS = full compromise).
- Prefer `httpOnly` cookies for auth tokens. Flag client-side JS that reads auth cookies.
- If `localStorage` is used for tokens, flag missing XSS mitigation across the app.
- Flag tokens in URL query parameters or fragments — they leak via referrer headers and browser history.
- Flag long-lived tokens without refresh rotation.
- Verify tokens are cleared on logout (`localStorage.removeItem`, cookie expiry).

## Dependency Supply Chain Attacks

- Flag new dependencies added without justification. Fewer dependencies = smaller attack surface.
- Flag dependencies with very low download counts or single maintainers for critical functionality.
- Verify `package-lock.json` or `yarn.lock` is committed — flag if missing.
- Flag `npm install` with `--ignore-scripts` disabled when adding unknown packages.
- Flag postinstall scripts in new dependencies — review what they execute.
- Flag wildcard version ranges (`*`, `>=`) in `package.json`. Use exact or caret ranges.
- Verify `npm audit` or `yarn audit` runs in CI. Flag if not configured.

## Server-Side Rendering Security

- Flag SSR code that injects user-controlled data into the initial HTML without escaping.
- Verify `serialize-javascript` or equivalent is used (not `JSON.stringify`) when embedding state in SSR HTML — `JSON.stringify` doesn't escape `</script>` tags.
- Flag SSR hydration mismatches that could indicate injected content.
- Flag server-side `fetch` calls using user-supplied URLs — SSRF risk.
- Verify SSR frameworks (Next.js) have CSRF protection on API routes.

## Environment Variable Exposure

- Flag any `REACT_APP_` or `VITE_` variable containing secrets — these are embedded in the client bundle.
- API keys in client-side env vars are public. Flag if the key grants write access or costs money.
- Flag `.env` files committed to source control. Verify `.env` is in `.gitignore`.
- Flag `process.env` usage outside of configuration modules — centralize env access.
- Verify `.env.example` exists and contains no real values.

## API Security

- Flag API calls without authentication headers (missing `Authorization` header or cookie).
- Flag API base URLs hardcoded as HTTP — enforce HTTPS.
- Verify error responses from APIs are not displayed raw to users (may contain internal details).
- Flag fetch calls without timeout or abort controller — can hang indefinitely.
- Flag client-side API calls that send more data than necessary (over-posting).

## CSRF Protection

- If using cookie-based auth, verify CSRF tokens are sent with state-changing requests.
- Flag `SameSite=None` on auth cookies without understanding the CSRF implications.
- For SPAs with JWT in headers: CSRF is not applicable — but verify tokens are not in cookies without CSRF protection.
- Flag forms that POST to external URLs without CSRF consideration.

## Content Security Policy

- Flag inline scripts and styles if CSP is enforced — they require `unsafe-inline` which weakens CSP.
- Verify `nonce` or `hash` based CSP is used with SSR frameworks.
- Flag `eval()`, `new Function()`, and `setTimeout(string)` — they require `unsafe-eval` in CSP.
- Flag missing CSP headers in the deployment configuration.

## Data Exposure

- Flag components that render sensitive data (SSN, full credit card, passwords) without masking.
- Flag Redux/Zustand stores containing sensitive data accessible via browser devtools.
- Flag `console.log` of auth tokens, user PII, or API keys.
- Verify React DevTools are disabled in production builds.
- Flag source maps deployed to production — they expose source code.

## Third-Party Script Security

- Flag `<script>` tags loading third-party code without `integrity` (SRI) attributes.
- Flag third-party analytics or tracking scripts with access to the full DOM.
- Verify iframes use `sandbox` attribute to restrict capabilities.
- Flag `postMessage` listeners without origin validation: `window.addEventListener('message', handler)` must check `event.origin`.

## File Upload Security

- Flag file uploads that rely solely on client-side type checking — always validate server-side.
- Flag image preview via `URL.createObjectURL` without type validation — malicious files can be rendered.
- Flag missing file size limits on upload inputs.
