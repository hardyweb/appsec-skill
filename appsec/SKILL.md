---
name: appsec
description: AppSec secure coding ΓÇö IDOR, XSS, CSRF, SSRF, SQLi, XXE, JWT, upload, path traversal
---
[APPSEC]
Cherry-picked from VibeSec-Skill (BehiSecc). Complements /skill security (infra) and /skill laravel (framework).
Scope: web app code. Infra hardening -> security skill. Laravel idioms -> laravel skill.

[ACCESS CONTROL]
- Verify ownership at data layer on every request, not just route/middleware
- Multi-tenant: check org membership, not only user role
- Check parent resource ownership (comment -> post -> user)
- Return 404 not 403 on foreign resources (prevents enumeration)
- UUIDv4 over sequential IDs (unless user explicitly wants sequential)
- Mass assignment: filter fields explicitly; never accept whole request body
- Privilege change: validate server-side, re-validate after change
- Account removal/deactivation: revoke all tokens + sessions immediately
- Short-lived tokens + refresh, or revocation list (jti blacklist)

[CLIENT-SIDE]
- XSS: encode by context (HTML/JS/URL/CSS), never one generic escape
- Indirect XSS sources: URL params, hash, Referer/User-Agent if rendered, third-party API data, WebSocket msgs, postMessage, localStorage if rendered
- Overlooked: error messages echoing input, PDF/HTML generators, email templates, admin log viewers, SVG upload, Markdown with HTML allowed
- Rich text: whitelist tags/attrs (DOMPurify)
- CSP: default-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; no unsafe-inline/unsafe-eval for scripts; nonce/hash for inline
- Headers: X-Content-Type-Options: nosniff, Referrer-Policy: strict-origin-when-cross-origin, HSTS, Cache-Control: no-store on sensitive pages
- CSRF: token tied to session, validated on all state-changing requests (POST/PUT/PATCH/DELETE), missing token = reject
- CSRF also on pre-auth: login, signup, password reset/change, email verify, OAuth callback
- Cookies: Secure; HttpOnly; SameSite=Lax or Strict
- Never state-change on GET
- CSRF token in header (X-CSRF-Token) not URL; regenerate on login
- JSON content-type does NOT prevent CSRF; validate Origin/Referer too
- Secrets: never in JS bundles, source maps, HTML comments, hidden fields, data attrs, localStorage, SSR hydration state, NEXT_PUBLIC_*/REACT_APP_* envs
- No stack traces, internal IPs, DB schema, or versions in prod responses

[OPEN REDIRECT]
- Prefer indirect refs: ?next=dashboard -> lookup map
- Else allowlist hostnames; or relative paths only (starts with /, no //)
- Block: userinfo @ trick, subdomain abuse, javascript:/data: schemes, %252f double encode, backslash, null byte, tab/newline, protocol-relative //, fragment #@
- IDN homograph: convert to Punycode before validate, or block non-ASCII for sensitive redirects

[PASSWORDS]
- Min 8 (12+ recommended), max 128, allow all chars, no forced character classes
- Hash: Argon2id / bcrypt / scrypt. Never MD5, SHA1, plain SHA256

[SSRF]
- Any server-side fetch of user URL: webhooks, URL preview, PDF-from-URL, image fetch, import-from-URL, RSS, proxy
- Preferred: allowlist domains
- Else: scheme http/https only, resolve DNS, reject private/loopback/link-local, pin resolved IP (no re-resolve), block redirects or validate each hop
- Block cloud metadata: 169.254.169.254, metadata.google.internal
- Bypass formats to reject: decimal 2130706433, octal 0177.0.0.1, hex 0x7f.0.0.1, [::1], [::ffff:127.0.0.1], [::], 127.1, IPv6 scope %25eth0, DNS rebinding, CNAME to internal, parser confusion
- Timeout + response size limit + network isolation

[FILE UPLOAD]
- Allowlist extension (single extension only), verify magic bytes, then parse with real library
- Server-side max size; also set proxy/server limits
- Rename to random UUID, discard original filename
- Store outside webroot or separate domain; non-executable perms
- Serve with Content-Disposition: attachment + nosniff
- Reject SVG with script or sanitize; reject polyglots
- Archives: validate extracted paths (ZIP slip)
- DOCX/XLSX/SVG are XML: disable external entities in parser
- Magic bytes: JPEG FFD8FF, PNG 89504E470D0A1A0A, GIF 47494638, PDF 25504446, ZIP/DOCX 504B0304

[SQL INJECTION]
- Parameterized queries / ORM binding = primary defense
- Cannot parameterize: ORDER BY, LIMIT/OFFSET, table/column names, IN lists -> allowlist
- LIKE: escape % and _
- DB user least privilege; never expose SQL errors

[XXE]
- Disable DTD + external entities + XInclude
- PHP: libxml_disable_entity_loader(true) / XMLReader with LIBXML_NONET
- Python: lxml XMLParser(resolve_entities=False, no_network=True) or defusedxml
- Java: disallow-doctype-decl=true, external-general/parameter-entities=false, setExpandEntityReferences(false)
- .NET: DtdProcessing.Prohibit, XmlResolver=null
- XML sources: SOAP, XML-RPC, RSS, SAML, SVG, DOCX/XLSX/PPTX, PDF XFA

[PATH TRAVERSAL]
- Avoid user input in paths; use indirect reference map
- Else canonicalize (realpath) and verify commonpath == base dir
- Reject .., absolute paths, null bytes; allowlist chars
- Validate extension

[JWT]
- Verify with explicit algorithm allowlist; never derive alg from token; reject none
- Secret >= 256-bit random (not a passphrase)
- Always set + validate exp; short-lived (15m) access token
- jti for revocation
- Store in HttpOnly+Secure+SameSite=Strict cookie, never localStorage
- Refresh token rotation, invalidate old on use

[GRAPHQL]
- Disable introspection in production
- Depth limit (~10), cost limit, cap operations per request

[AUTHZ CHECKLIST]
- [ ] Ownership verified at data layer
- [ ] Org/tenant membership checked
- [ ] Role permissions validated server-side
- [ ] Field allowlist on writes
- [ ] 404 on foreign resources
- [ ] Tokens revoked on deactivation

[AUDIT MODE  READ-ONLY]
- For existing/previously-built apps: audit only. Do NOT edit, refactor, or "fix" any code unless user explicitly asks after reading the report.
- Read-only actions allowed: read files, grep/rg, git log/diff/show, dependency manifests, config review, framework version check.
- No writes, no migrations, no composer/npm install, no service restart, no DB writes, no running the app.
- Never run active exploitation against live/prod (sqlmap, ZAP, nmap, curl payloads) without explicit approval. Static review first.
- Report format, grouped by severity (Critical/High/Medium/Low), each finding:
  file:line ΓÇö vuln class ΓÇö why exploitable ΓÇö fix suggestion (one line, no patch applied)
- End report with: what was checked, what was NOT checked (needs approval), and a prioritized fix list.
- Laravel-specific greps: `$request->all()`/`->update($request`, `DB::raw`, `whereRaw`, `orderByRaw`, `{!! !!}` in Blade, `->withoutMiddleware`, `verify` disabled, `APP_DEBUG=true`, `APP_KEY` in git, missing `authorize()` in FormRequest/Policy, `Storage::` with user path, `Http::get($userUrl)`, `unserialize`, `eval`, `include $`.
- Deliverable is the report. Code changes only on explicit follow-up request.

[SECRET SCAN [TRADITIONAL, NO DEPENDENCIES]
Use plain grep/rg + git. No gitleaks/trufflehog required (mention only if already installed).

1. Working tree (exclude vendor/node_modules/.git):
   rg -n --hidden -g '!vendor' -g '!node_modules' -g '!.git' \
     -e 'AKIA[0-9A-Z]{16}' \
     -e 'ASIA[0-9A-Z]{16}' \
     -e 'gh[pousr]_[A-Za-z0-9]{36,}' \
     -e 'github_pat_[A-Za-z0-9_]{82}' \
     -e 'sk_live_[0-9A-Za-z]{24,}' \
     -e 'sk_test_[0-9A-Za-z]{24,}' \
     -e 'rk_live_[0-9A-Za-z]{24,}' \
     -e 'xox[baprs]-[0-9A-Za-z-]{10,}' \
     -e 'AIza[0-9A-Za-z_-]{35}' \
     -e 'ya29\.[0-9A-Za-z_-]{20,}' \
     -e 'SG\.[0-9A-Za-z_-]{22}\.[0-9A-Za-z_-]{43}' \
     -e '-----BEGIN (RSA |EC |OPENSSH |PGP )?PRIVATE KEY-----' \
     -e 'eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}' \
     -e '(api|access|secret|private|auth)[_-]?key["'"'"' :=]+[A-Za-z0-9_\-]{16,}' \
     -e '(password|passwd|pwd)["'"'" :=]+[^ "'"'"'$]{8,}' \
     -e '(token|bearer)["'"'" :=]+[A-Za-z0-9_.\-]{20,}' \
     -e '://[^/:@[:space:]]+:[^/@[:space:]]+@' \
     -e '(mongodb|postgres|postgresql|mysql|redis|amqp)://[^[:space:]]*:[^[:space:]@]+@'

2. Built assets shipped to browser (highest risk ΓÇö public by definition):
   rg -n -e 'sk_live|sk_test|AKIA|AIza|xox[baprs]-|gh[pousr]_|BEGIN .*PRIVATE KEY' public/ dist/ build/ static/ .next/ 2>/dev/null
   rg -n 'NEXT_PUBLIC_|REACT_APP_|VITE_|PUBLIC_' .env* 2>/dev/null   # inspect values, names alone are fine

3. Git history (deleted secrets still live here):
   git log --all --oneline -- .env .env.* '*.pem' '*.key' '*.p12' '*.pfx' '*.jks' id_rsa* 2>/dev/null
   git log --all -p -S 'AKIA' --oneline | head -50
   git log --all -p -S 'sk_live_' --oneline | head -50
   git log --all -p -S 'BEGIN RSA PRIVATE KEY' --oneline | head -50
   git rev-list --all | head -200 | xargs -I{} git grep -l -E 'AKIA[0-9A-Z]{16}|sk_live_' {} 2>/dev/null | head
   git log --all --diff-filter=D --name-only --pretty=format: | sort -u | grep -E '\.env|\.pem|\.key|\.p12|id_rsa'

4. Config / infra files:
   rg -n -e 'APP_KEY=base64:' -e 'APP_DEBUG=true' -e 'DB_PASSWORD=' -e 'MAIL_PASSWORD=' .env* 2>/dev/null
   rg -n --hidden -g '!vendor' -g '!node_modules' -e 'password\s*[:=]\s*["'"'"'][^"'"'"']{6,}' -e 'secret\s*[:=]\s*["'"'"'][^"'"'"']{6,}' -e 'apiKey\s*[:=]' -e 'token\s*[:=]\s*["'"'"'][^"'"'"']{10,}'
   rg -n --hidden -g '!vendor' -g '!node_modules' -e 'AKIA' -e 'BEGIN .*PRIVATE KEY' ~/.aws ~/.ssh ~/.config 2>/dev/null   # only if user points at it

5. Reduce noise before reporting:
   - Skip matches in: .env.example, *.md, *.lock, test fixtures, docs, vendor/, node_modules/, *.min.js
   - Skip obvious placeholders: changeme, your-key-here, xxx+, example, dummy, test, REDACTED, <...>, ${...}, process.env.*, env(
   - High-entropy heuristics: flag any 32+ char string mixing upper/lower/digit in a quoted literal assigned to a name containing key/secret/token/pass
   - Never print full secret in report. Mask: first 6 + last 4 chars, e.g. AKIAIOSF...MPLE

6. Report each hit as:
   file:line ΓÇö secret type ΓÇö masked value ΓÇö exposed? (in git history / in built asset / local only) ΓÇö action
   Action ladder: (a) in built asset or git history -> ROTATE NOW, then purge history; (b) local .env only and gitignored -> ok, verify .gitignore; (c) committed to repo -> rotate + purge.
   Rotation is the fix. Deleting the line is not. Assume any secret in git history or a public bundle is already compromised.

7. Also check .gitignore covers: .env, .env.*, *.pem, *.key, *.p12, storage/*.key (Laravel), node_modules, vendor.

 
[STATE]
- Approval gates apply to destructive verification steps (scanning prod, active exploitation tests) ΓÇö ask first.
- Verify with /skill verification; record results in walkthrough.
