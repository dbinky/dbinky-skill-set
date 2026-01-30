# Angular Security Review Rules

## Built-in XSS Protection and Bypasses

- Angular sanitizes all interpolated values in templates by default. Flag code that bypasses this.
- Flag every use of `bypassSecurityTrustHtml()`, `bypassSecurityTrustScript()`, `bypassSecurityTrustUrl()`, `bypassSecurityTrustResourceUrl()`, and `bypassSecurityTrustStyle()`. Each must have a justifying comment and verified safe input.
- Flag `[innerHTML]` with user-controlled content — Angular sanitizes it but not perfectly. Prefer text interpolation.
- Flag dynamic `<script>` injection via `Renderer2` — Angular strips scripts from `innerHTML` but `Renderer2.createElement('script')` bypasses this.
- Flag `ElementRef.nativeElement.innerHTML = ...` — completely bypasses Angular's sanitization.
- Verify `DomSanitizer` is only used to mark known-safe static content, never unsanitized user input.

## HTTP Interceptor Auth Patterns

- Verify a centralized `HttpInterceptor` attaches auth tokens to outgoing requests. Flag per-service token attachment.
- Flag interceptors that attach tokens to ALL requests including third-party URLs — tokens leak to external services.
- Verify the interceptor checks the request URL before attaching credentials.
- Flag interceptors that silently retry on 401 without token refresh logic.
- Verify 401 responses redirect to login or trigger token refresh, not silent failure.
- Flag interceptors that modify the request but don't clone it first (`req.clone()`).

## Route Guards

- Verify protected routes use `CanActivate` / `canActivate` (functional guards in Angular 15+).
- Flag routes with sensitive data that lack guards — client-side guards are UX, not security, but they prevent accidental exposure.
- Flag guard logic that only checks a boolean flag without validating the token's expiry.
- Verify child routes inherit parent guards or have their own. Flag unprotected child routes under protected parents.
- Flag `CanDeactivate` guards that expose form data in the confirmation dialog.
- Flag guards that call async validation but don't handle failures (default behavior allows navigation).

## Content Security Policy

- Flag inline styles and scripts — they require `unsafe-inline` which weakens CSP.
- Angular CLI supports CSP nonces via `ngCspNonce` attribute (Angular 16+). Verify it's configured.
- Flag `eval()` usage — Angular's JIT compiler requires `unsafe-eval`. Verify AOT compilation is used in production.
- Flag missing CSP `style-src` for Angular's component styles — they are injected as `<style>` tags.
- Verify CSP headers are set at the server/CDN level, not just as `<meta>` tags.

## Angular's DomSanitizer

- Flag injection of `DomSanitizer` — review every call site.
- Verify sanitized values are not constructed from user input without validation.
- Flag `bypassSecurityTrust*` methods called in pipes — pipes rerun on every change detection, widening the attack surface.
- Flag sanitizer bypasses in shared/library components — they affect all consumers.
- Verify URLs passed to `bypassSecurityTrustUrl` are validated against an allowlist of schemes (`https:`, `mailto:`).

## Input Validation

- Flag template-driven forms without validation directives. Use reactive forms with `Validators` for consistent validation.
- Verify server-side validation exists for all inputs — client validation is bypassable.
- Flag `pattern` validators using incomplete regex (missing `^` and `$` anchors).
- Flag file upload components without type and size validation.
- Flag `@Input()` properties used in `[innerHTML]` or URL bindings without sanitization.

## API Security

- Flag HTTP calls using `http://` — enforce HTTPS.
- Verify sensitive endpoints require authentication tokens via interceptors.
- Flag API keys embedded in Angular source code — they ship to the browser.
- Flag `withCredentials: true` on cross-origin requests without understanding CORS implications.
- Verify error interceptors don't expose server error details to end users.

## Session and Token Management

- Flag auth tokens stored in `localStorage` — XSS accessible. Prefer `httpOnly` cookies.
- If tokens must be in `localStorage`, verify the app has strong XSS protection (CSP, sanitization).
- Flag missing token expiry checks on the client. Verify token refresh logic exists.
- Flag tokens passed in URL query parameters.
- Verify logout clears all stored tokens and session data.

## Data Exposure

- Flag Angular components that render sensitive data (PII, tokens, secrets) in the DOM.
- Flag `console.log` of auth tokens or sensitive user data.
- Verify source maps are NOT deployed to production — they expose TypeScript source.
- Flag Angular DevTools access in production (cannot be fully prevented but source maps make it worse).
- Flag services that cache sensitive data in memory without clearing on logout.

## Dependency and Build Security

- Flag `angular.json` with `optimization: false` in production — unminified code is easier to reverse engineer.
- Verify `budgets` are configured to catch unexpectedly large bundles (may indicate injected code).
- Flag third-party Angular libraries that require `bypassSecurityTrust*` in their setup instructions.
- Verify `ng update` is run regularly to patch Angular security fixes.
- Flag `scripts` array in `angular.json` loading unvetted third-party scripts.

## CSRF Protection

- Angular's `HttpClientXsrfModule` provides CSRF token handling. Verify it's imported when using cookie auth.
- Flag custom CSRF implementations when Angular's built-in module works.
- Verify the CSRF cookie name and header name match the server's expectations.
- Flag state-changing GET requests — CSRF protection only applies to POST/PUT/DELETE.
