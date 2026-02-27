# React Security — MANG Level Interview Guide

> XSS · CSP · Authentication · Authorization · CSRF · Dependency Security · Secrets · MANG Scenarios

---

## Table of Contents

1. [XSS (Cross-Site Scripting)](#1-xss-cross-site-scripting)
2. [`dangerouslySetInnerHTML` — When & How](#2-dangerouslysetinnerhtml)
3. [Content Security Policy (CSP)](#3-content-security-policy)
4. [Authentication Patterns](#4-authentication-patterns)
5. [Authorization & Route Protection](#5-authorization--route-protection)
6. [CSRF (Cross-Site Request Forgery)](#6-csrf)
7. [Secure Data Handling](#7-secure-data-handling)
8. [Dependency Security](#8-dependency-security)
9. [Server-Side Rendering Security](#9-ssr-security)
10. [React Server Components Security](#10-react-server-components)
11. [Security Headers & Best Practices](#11-security-headers)
12. [MANG Cross-Questions & Scenarios](#12-mang-cross-questions--scenarios)

---

## 1. XSS (Cross-Site Scripting)

### Q: How does React protect against XSS?

**A:** React **auto-escapes** all values embedded in JSX before rendering them to the DOM. Any string rendered with `{}` is treated as plain text — never as HTML.

```tsx
// User input with malicious script
const userComment = '<script>alert("hacked")</script>';

// ✅ React auto-escapes — renders as TEXT, not executed
<p>{userComment}</p>
// DOM output: <p>&lt;script&gt;alert("hacked")&lt;/script&gt;</p>

// ✅ Also safe — attributes are escaped too
<div title={userComment} />
// DOM output: <div title="&lt;script&gt;alert(&quot;hacked&quot;)&lt;/script&gt;"></div>
```

### Q: Where React does NOT protect you (XSS attack vectors)

**Vector 1: `dangerouslySetInnerHTML`**
```tsx
// ❌ Bypasses React's escaping — raw HTML injected into DOM
<div dangerouslySetInnerHTML={{ __html: userContent }} />
// If userContent contains <script> or <img onerror="...">, it EXECUTES
```

**Vector 2: `href` with `javascript:` protocol**
```tsx
// ❌ User-controlled URLs can execute JS
const userUrl = 'javascript:alert("XSS")';
<a href={userUrl}>Click me</a> // EXECUTES on click!

// ✅ Validate protocol
function SafeLink({ href, children }) {
  const isSafe = /^https?:\/\//.test(href) || href.startsWith('/');
  return <a href={isSafe ? href : '#'}>{children}</a>;
}
```

**Vector 3: Dynamic `src` attributes**
```tsx
// ❌ User controls iframe/script src
<iframe src={userProvidedUrl} />    // can load malicious page
<script src={userProvidedUrl} />    // can load malicious script

// ✅ Whitelist allowed domains
const ALLOWED_DOMAINS = ['youtube.com', 'vimeo.com'];
function SafeEmbed({ url }) {
  const { hostname } = new URL(url);
  if (!ALLOWED_DOMAINS.includes(hostname)) return <p>Blocked embed</p>;
  return <iframe src={url} sandbox="allow-scripts" />;
}
```

**Vector 4: CSS injection via `style` prop**
```tsx
// ❌ Older browsers: CSS expressions can execute JS
<div style={{ background: userInput }} />
// userInput = "url('javascript:alert(1)')"  — some browsers execute this

// ✅ Validate/sanitize CSS values
const safeBg = /^#[0-9a-f]{3,8}$/i.test(userColor) ? userColor : '#ffffff';
<div style={{ background: safeBg }} />
```

**Vector 5: `eval()` and dynamic code execution**
```tsx
// ❌ NEVER do this
eval(userInput);
new Function(userInput);
setTimeout(userInput, 1000); // string form executes as code

// ✅ Use data-driven approaches, never eval
```

### Cross-question: *"If React escapes everything, why do XSS vulnerabilities still happen in React apps?"*

Because developers bypass protections:
1. Using `dangerouslySetInnerHTML` without sanitizing
2. Allowing `javascript:` in `href` / `src`
3. Injecting user data into `<script>` tags during SSR
4. Using third-party libraries that render raw HTML
5. Server-side rendering without escaping data in the HTML template

---

## 2. `dangerouslySetInnerHTML`

### Q: When is `dangerouslySetInnerHTML` acceptable?

| ✅ Acceptable | ❌ Not acceptable |
|---|---|
| CMS/blog content from **trusted** admin dashboard | User-submitted comments/reviews |
| Markdown rendered by **you** on the server | HTML from URL query parameters |
| Email templates rendered in preview | Any untrusted external API response |
| Static HTML generated at build time | User profile bios |

### Q: How do you sanitize HTML before rendering?

Use **DOMPurify** — the industry-standard HTML sanitizer:

```bash
npm install dompurify @types/dompurify
```

```tsx
import DOMPurify from 'dompurify';

function SafeHTML({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li', 'h1', 'h2', 'h3', 'img'],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'class'],
    ALLOW_DATA_ATTR: false,
  });

  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// What DOMPurify removes:
// <script>alert('XSS')</script>          → removed entirely
// <img src=x onerror="alert('XSS')">    → removes onerror attribute
// <a href="javascript:alert('XSS')">    → removes href
// <div onclick="alert('XSS')">          → removes onclick
```

### Cross-question: *"Can you sanitize on the client side, or must it be server side?"*

**Both, but server-side is preferred:**
- **Server-side** sanitization happens once at storage time → every consumer gets clean data
- **Client-side** sanitization is a fallback → but if an API returns unsanitized data to a non-React client (mobile app, email), it's still vulnerable

**Best practice:** Sanitize on **write** (server) AND on **read** (client) — defense in depth.

---

## 3. Content Security Policy (CSP)

### Q: What is CSP and how does it prevent XSS?

**A:** CSP is a browser security mechanism delivered via HTTP header. It tells the browser **which sources of content are allowed** — scripts, styles, images, etc. If an injected script doesn't match the policy, the browser **blocks it**.

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' https: data:;
  connect-src 'self' https://api.example.com;
  font-src 'self' https://fonts.googleapis.com;
  frame-src 'none';
  object-src 'none';
  base-uri 'self';
```

| Directive | Controls | Example |
|---|---|---|
| `script-src` | JavaScript sources | `'self'` = only your domain |
| `style-src` | CSS sources | `'unsafe-inline'` = allow inline styles (React needs this) |
| `img-src` | Image sources | `https: data:` = any HTTPS + data URIs |
| `connect-src` | Fetch/XHR/WebSocket targets | `https://api.example.com` |
| `frame-src` | iframe sources | `'none'` = no iframes allowed |
| `object-src` | Plugins (Flash, Java) | `'none'` = always block |

### CSP and React — key issues

```tsx
// Problem 1: React inline styles
// style-src 'unsafe-inline' is needed for JSX style={{}} props
// ❌ Without it, all inline styles are blocked

// Problem 2: Next.js inline scripts
// Next.js injects inline <script> for hydration data
// Need a nonce: <script nonce="abc123">

// Solution: Use nonce-based CSP
// Header: Content-Security-Policy: script-src 'nonce-abc123'
// Next.js config:
const cspHeader = `script-src 'nonce-${nonce}' 'strict-dynamic'`;
```

### Cross-question: *"What is `'strict-dynamic'` and why use it?"*

`'strict-dynamic'` tells the browser: "trust any script loaded by a trusted script." So if your main bundle (loaded with a nonce) dynamically loads chunks, those chunks are also trusted — without needing to whitelist every CDN URL.

---

## 4. Authentication Patterns

### Q: Where should you store JWT tokens in a React app?

| Storage | XSS Safe? | CSRF Safe? | Verdict |
|---|---|---|---|
| `localStorage` | ❌ Any JS can read it | ✅ Not sent automatically | Avoid for sensitive tokens |
| `sessionStorage` | ❌ Any JS can read it | ✅ Not sent automatically | Slightly better (cleared on tab close) |
| **HttpOnly cookie** | ✅ JS can't access it | ❌ Sent automatically (needs CSRF protection) | **Recommended** |
| In-memory (variable) | ✅ Wiped on refresh | ✅ Never persisted | Good for short sessions |

### Recommended: HttpOnly cookie + refresh token pattern

```
┌──────────┐   POST /login          ┌──────────┐
│  React   │ ─────────────────────→ │  Server  │
│   App    │                        │          │
│          │ ←── Set-Cookie:        │          │
│          │     access=<JWT>;      │          │
│          │     HttpOnly; Secure;  │          │
│          │     SameSite=Strict;   │          │
│          │     Max-Age=900        │          │
│          │                        │          │
│          │ ←── Set-Cookie:        │          │
│          │     refresh=<token>;   │          │
│          │     HttpOnly; Secure;  │          │
│          │     Path=/auth/refresh │          │
└──────────┘                        └──────────┘
```

```tsx
// React side — cookies are sent automatically, no manual token management
async function fetchProfile() {
  const res = await fetch('/api/profile', {
    credentials: 'include', // send cookies cross-origin
  });
  return res.json();
}

// Token refresh — happens transparently
async function fetchWithAuth(url, options = {}) {
  let res = await fetch(url, { ...options, credentials: 'include' });
  
  if (res.status === 401) {
    // Access token expired — try refresh
    const refreshRes = await fetch('/auth/refresh', {
      method: 'POST',
      credentials: 'include',
    });
    
    if (refreshRes.ok) {
      // New access token set via cookie — retry original request
      res = await fetch(url, { ...options, credentials: 'include' });
    } else {
      // Refresh failed — redirect to login
      window.location.href = '/login';
    }
  }
  
  return res;
}
```

### Cross-question: *"If you must use localStorage (e.g., third-party auth), how do you mitigate XSS risk?"*

1. **Strict CSP** — prevent injected scripts from running
2. **Short-lived tokens** (15 min) + refresh tokens in HttpOnly cookies
3. **Token binding** — include a fingerprint hash in the JWT that's verified server-side
4. **Subresource Integrity (SRI)** — ensure CDN scripts aren't tampered with
5. **Regular dependency audits** — compromised npm package = XSS

---

## 5. Authorization & Route Protection

### Q: How do you protect routes in a React SPA?

```tsx
// Protected route component
function ProtectedRoute({ children, requiredRole }) {
  const { user, isLoading } = useAuth();

  if (isLoading) return <Spinner />;
  if (!user) return <Navigate to="/login" replace />;
  if (requiredRole && user.role !== requiredRole) return <Navigate to="/forbidden" replace />;

  return children;
}

// Usage
<Routes>
  <Route path="/login" element={<LoginPage />} />
  <Route path="/dashboard" element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  } />
  <Route path="/admin" element={
    <ProtectedRoute requiredRole="admin">
      <AdminPanel />
    </ProtectedRoute>
  } />
</Routes>
```

### Cross-question: *"Is client-side route protection sufficient?"*

**Absolutely not.** Client-side guards are **UX features**, not security features. A determined attacker can:
1. Disable JavaScript
2. Call API endpoints directly (curl/Postman)
3. Modify React code in the browser

**Every API endpoint must validate authentication and authorization server-side.** Client-side guards prevent accidental access and improve UX — they don't provide security.

```
Client-side: "Should I show this page?" → UX
Server-side: "Should I return this data?" → SECURITY
```

---

## 6. CSRF (Cross-Site Request Forgery)

### Q: What is CSRF and how do you prevent it?

**A:** CSRF exploits the browser's automatic cookie sending. A malicious site can make a request to your API, and the browser includes your auth cookies — the server thinks it's you.

```html
<!-- Attacker's site: evil.com -->
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker" />
  <input name="amount" value="10000" />
</form>
<script>document.forms[0].submit();</script>
<!-- Browser sends bank.com cookies automatically! -->
```

### Prevention strategies

**1. `SameSite` cookies (primary defense)**
```
Set-Cookie: token=abc; SameSite=Strict; Secure; HttpOnly
```

| `SameSite` value | Behavior |
|---|---|
| `Strict` | Cookie never sent from cross-origin requests |
| `Lax` | Sent on top-level GET navigations only (default in modern browsers) |
| `None` | Always sent (must include `Secure`) |

**2. CSRF tokens (defense in depth)**
```tsx
// Server generates a CSRF token per session
// React sends it in a custom header (can't be set by cross-origin forms)
const csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;

fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken, // custom headers can't be sent cross-origin without CORS
  },
  credentials: 'include',
  body: JSON.stringify({ to: 'alice', amount: 100 }),
});
```

**3. Check `Origin` / `Referer` header server-side**

### Cross-question: *"Do SPAs need CSRF protection if they only use JSON APIs?"*

**Yes, but reduced risk.** Browsers block cross-origin `application/json` POST requests via CORS preflight. However:
- Some servers accept `application/x-www-form-urlencoded` (which bypasses preflight)
- Flash/Java plugins could send arbitrary requests (legacy risk)
- `SameSite=Lax` + CSRF token is still recommended

---

## 7. Secure Data Handling

### Q: How do you handle sensitive data in React?

**Secrets in frontend code:**

```tsx
// ❌ NEVER put in frontend code — visible in bundle
const API_KEY = 'sk-secret-abc123';
const DB_PASSWORD = 'admin123';

// ❌ .env files in React are NOT secret (bundled into JS)
// REACT_APP_API_KEY, VITE_API_KEY — these are PUBLIC
console.log(import.meta.env.VITE_API_KEY); // visible in browser Sources tab

// ✅ Keep secrets on the server — proxy through your API
// Frontend: fetch('/api/weather?city=Delhi')
// Server: fetch('https://weather.api.com?key=SECRET&city=Delhi')
```

**Sensitive data in state:**

```tsx
// ❌ Storing credit card number in React state
const [cardNumber, setCardNumber] = useState('');
// Problem: React DevTools can read state, browser memory dump exposes it

// ✅ Use iframes from payment provider (Stripe Elements)
// Card data never touches your React code
<Elements stripe={stripePromise}>
  <CardElement /> {/* rendered in Stripe's iframe */}
</Elements>
```

**Preventing data leakage:**

```tsx
// ❌ Logging sensitive data
console.log('User login:', { email, password }); // visible in browser console

// ❌ Sending sensitive data in URL params
navigate(`/reset-password?token=${resetToken}`);
// Token appears in: browser history, referrer headers, server logs

// ✅ Send via POST body or fragment (#)
navigate('/reset-password');
// Token sent via POST body or stored in sessionStorage temporarily
```

### Cross-question: *"What happens if someone opens React DevTools on your production app?"*

They can see:
1. All component state (passwords if stored in state)
2. Redux store contents
3. Context values
4. Props passed between components

**Mitigation:**
- Never store passwords/tokens in React state (use HttpOnly cookies)
- Redact sensitive fields in Redux DevTools: `configureStore({ devTools: false })` in production
- Use `devToolsEnhancer({ features: { dispatch: false } })` to limit DevTools capabilities

---

## 8. Dependency Security

### Q: How do you secure npm dependencies?

**1. Audit regularly**
```bash
npm audit                    # check for known vulnerabilities
npm audit fix                # auto-fix compatible updates
npm audit fix --force        # fix with breaking changes (review carefully)
npx better-npm-audit audit   # stricter auditing
```

**2. Lock versions**
```json
// package-lock.json — commit this! It pins exact versions
// Without it: npm install may fetch newer (potentially compromised) versions

// Use exact versions in package.json
"dependencies": {
  "react": "18.2.0"         // exact
  // vs "react": "^18.2.0"  // allows 18.x.x — risky
}
```

**3. Subresource Integrity (SRI) for CDN scripts**
```html
<!-- If script is tampered on CDN, browser blocks it -->
<script
  src="https://cdn.example.com/react.production.min.js"
  integrity="sha384-abc123..."
  crossorigin="anonymous"
></script>
```

**4. Use `npm-provenance` and `socket.dev` for supply chain security**
```bash
npx socket optimize         # detect suspicious packages
npx lockfile-lint --path package-lock.json --type npm --allowed-hosts npm
```

**5. Monitor for malicious packages**

| Attack Type | Example | Defense |
|---|---|---|
| **Typosquatting** | `react-dom` → `reactdom` | Review package names carefully |
| **Dependency confusion** | Public pkg named like internal pkg | Use npm scoped packages (`@company/pkg`) |
| **Compromised maintainer** | Popular package taken over | Pin versions, review changelogs |
| **Install scripts** | `postinstall` runs arbitrary code | Use `--ignore-scripts` for untrusted pkgs |

---

## 9. SSR Security

### Q: What security risks are unique to Server-Side Rendering?

**Risk 1: HTML injection during hydration**

```tsx
// ❌ Server renders user data directly into HTML
function Page({ userBio }) {
  return <div dangerouslySetInnerHTML={{ __html: userBio }} />;
}
// If userBio = '<script>steal(document.cookie)</script>'
// HTML is served with the script — executes before React hydrates!

// ✅ Always sanitize before rendering
import DOMPurify from 'isomorphic-dompurify'; // works on server + client
const clean = DOMPurify.sanitize(userBio);
```

**Risk 2: Leaking server data to client**

```tsx
// ❌ Next.js getServerSideProps — don't return internal data
export async function getServerSideProps() {
  const user = await db.query('SELECT * FROM users WHERE id = 1');
  return { props: { user } }; // ❌ Sends entire DB row (including password_hash)!
}

// ✅ Return only what the client needs
export async function getServerSideProps() {
  const user = await db.query('SELECT id, name, avatar FROM users WHERE id = 1');
  return { props: { user } };
}
```

**Risk 3: SSRF (Server-Side Request Forgery)**

```tsx
// ❌ Server fetches a URL provided by the user
export async function getServerSideProps({ query }) {
  const res = await fetch(query.url); // user controls the URL!
  // They can access: http://169.254.169.254/metadata (AWS secrets)
  // Or: http://localhost:5432 (internal database)
}

// ✅ Whitelist allowed URLs
const ALLOWED_HOSTS = ['api.example.com'];
const url = new URL(query.url);
if (!ALLOWED_HOSTS.includes(url.hostname)) throw new Error('Blocked');
```

---

## 10. React Server Components

### Q: What security considerations exist for React Server Components?

```tsx
// Server Component — runs ONLY on the server
// ✅ Safe to use secrets here
async function UserDashboard() {
  const data = await db.query('SELECT * FROM dashboard', { apiKey: process.env.SECRET_KEY });
  return <DashboardView data={data} />;
  // SECRET_KEY is NEVER sent to the client ✅
}

// ⚠️ But be careful what you pass to Client Components
'use client';
function DashboardView({ data }) {
  // 'data' is serialized and sent to the browser
  // Don't pass: tokens, API keys, internal IDs the user shouldn't see
}
```

**Server Actions security:**

```tsx
// Server Action — function that runs on the server, called from client
'use server';

async function deleteUser(formData: FormData) {
  const userId = formData.get('userId');
  
  // ❌ Missing: authentication check
  // Anyone can call this function with any userId!
  
  // ✅ Always validate auth in Server Actions
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');
  if (session.user.role !== 'admin') throw new Error('Forbidden');
  
  // ✅ Validate input
  const parsed = z.string().uuid().parse(userId);
  
  await db.deleteUser(parsed);
}
```

### Cross-question: *"Can a user call a Server Action with tampered arguments?"*

**Yes.** Server Actions are POST endpoints under the hood. A user can call them with any arguments using fetch/curl. **Always validate authentication, authorization, and input on every Server Action** — treat them exactly like API endpoints.

---

## 11. Security Headers & Best Practices

### Essential HTTP headers

```
# Prevent clickjacking (page loaded in iframe)
X-Frame-Options: DENY
# Modern version:
Content-Security-Policy: frame-ancestors 'none'

# Prevent MIME-type sniffing
X-Content-Type-Options: nosniff

# Enable HTTPS everywhere
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# Control what info is sent in Referer header
Referrer-Policy: strict-origin-when-cross-origin

# Restrict browser features
Permissions-Policy: camera=(), microphone=(), geolocation=(self)
```

### React-specific best practices checklist

```
✅ Never use dangerouslySetInnerHTML without DOMPurify
✅ Validate href/src attributes (block javascript: protocol)
✅ Use HttpOnly + Secure + SameSite cookies for auth
✅ Keep secrets on the server (VITE_* env vars are PUBLIC)
✅ Sanitize data on write (server) AND read (client)
✅ Client-side route guards = UX. Server-side auth = Security.
✅ Pin dependency versions, audit regularly
✅ Set CSP headers with nonces for inline scripts
✅ Disable React DevTools in production for sensitive apps
✅ Use HTTPS everywhere
✅ Validate all Server Action inputs (zod/yup)
✅ Never log sensitive data in console statements
✅ Use iframe sandbox for third-party embeds
✅ Implement rate limiting on auth endpoints (server)
✅ Enable SRI for CDN-loaded scripts
```

---

## 12. MANG Cross-Questions & Scenarios

---

### Scenario 1: XSS in a Rich Text Editor

> **"Your app has a rich text editor (like blog posts). Users write HTML content. Other users view it. How do you prevent stored XSS?"**

**Answer:**

```
Write flow:  User → Editor → DOMPurify (server) → Database
Read flow:   Database → DOMPurify (client, defense in depth) → dangerouslySetInnerHTML
```

```tsx
// Server (on save)
import createDOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';

const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window);

app.post('/api/posts', (req, res) => {
  const cleanContent = DOMPurify.sanitize(req.body.content, {
    ALLOWED_TAGS: ['h1', 'h2', 'h3', 'p', 'b', 'i', 'em', 'strong', 'a', 'img', 'ul', 'ol', 'li', 'blockquote', 'code', 'pre'],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'class'],
    FORBID_ATTR: ['style', 'onerror', 'onload', 'onclick'],
  });
  db.saveBlogPost({ ...req.body, content: cleanContent });
});

// Client (on render — defense in depth)
function BlogPost({ content }) {
  const clean = useMemo(() => DOMPurify.sanitize(content), [content]);
  return <article dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### Cross-question: *"What if the attacker bypasses DOMPurify?"*

Defense in depth:
1. **CSP** blocks inline scripts even if injected: `script-src 'nonce-...'`
2. **HttpOnly cookies** — even if JS executes, it can't steal tokens
3. **Monitor** for CSP violation reports: `Content-Security-Policy-Report-Only`
4. **Update DOMPurify** regularly — new bypass techniques are patched

---

### Scenario 2: Securing a Multi-Tenant SaaS Dashboard

> **"Users from Company A must never see data from Company B. How do you enforce this in a React + API architecture?"**

**Answer framework:**

```
Layer 1: API (primary enforcement)
- Every request includes tenant_id from the JWT
- Database queries ALWAYS include WHERE tenant_id = jwt.tenant_id
- Never trust tenant_id from the client request body

Layer 2: React (secondary, UX-level)
- Auth context contains tenant info
- Route guards check tenant permissions
- React Query keys include tenant: ['users', tenantId]
  (cache is scoped per tenant — switching tenants invalidates cache)
```

```tsx
// Server middleware — strict tenant isolation
function tenantMiddleware(req, res, next) {
  const tenantId = req.jwt.tenantId; // from validated JWT
  req.tenantId = tenantId;           // attach to request
  
  // Override any client-supplied tenantId
  if (req.body?.tenantId && req.body.tenantId !== tenantId) {
    return res.status(403).json({ error: 'Tenant mismatch' });
  }
  next();
}

// Database — every query is tenant-scoped
const users = await db.query('SELECT * FROM users WHERE tenant_id = $1', [req.tenantId]);
```

### Cross-question: *"What if a developer forgets the tenant filter in a query?"*

1. **Row-Level Security (RLS)** in PostgreSQL — tenant filtering at the database level:
   ```sql
   ALTER TABLE users ENABLE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON users
     USING (tenant_id = current_setting('app.tenant_id')::uuid);
   ```
2. **ORM middleware** that auto-injects `WHERE tenant_id = ?`
3. **Code review checklist** requiring tenant filter on every query
4. **Integration tests** that attempt cross-tenant data access

---

### Scenario 3: JWT Token Theft — Incident Response

> **"A user reports their account was accessed by someone else. You suspect JWT theft. Walk through investigation and mitigation."**

**Investigation:**
1. Check server logs — was the JWT used from a different IP/user-agent?
2. Check if the user's session was created via legitimate login
3. Look for XSS evidence — CSP violation reports, unusual script loads
4. Check for compromised dependencies (`npm audit`)

**Immediate mitigation:**
```tsx
// 1. Invalidate all sessions for the affected user
await db.deleteAllSessions(userId);

// 2. Rotate the JWT secret (invalidates ALL tokens company-wide) — nuclear option
// Better: use per-user token versioning
const jwt = sign({ userId, tokenVersion: user.tokenVersion }, SECRET);
// On compromise: user.tokenVersion++ → all old tokens fail validation

// 3. Force re-login
```

**Long-term prevention:**
- Short-lived access tokens (15 min)
- Refresh token rotation (single-use refresh tokens)
- Token binding (fingerprint in JWT verified against TLS connection)
- Anomaly detection (login from new country → require 2FA)

---

### Scenario 4: Preventing Prototype Pollution

> **"A team member uses `Object.assign` with user input. Explain the risk and fix."**

```js
// ❌ Prototype pollution via user input
const userInput = JSON.parse('{"__proto__": {"isAdmin": true}}');
const config = Object.assign({}, defaults, userInput);

// Now EVERY object in the app has isAdmin === true!
console.log({}.isAdmin); // true — polluted!
```

**In React context:**
```tsx
// ❌ Merging user preferences into defaults
const prefs = { ...defaultPrefs, ...userInput };
// If userInput has __proto__ or constructor.prototype, it's dangerous

// ✅ Validate input shape with Zod
const PrefsSchema = z.object({
  theme: z.enum(['light', 'dark']),
  fontSize: z.number().min(12).max(24),
});
const safePrefs = PrefsSchema.parse(userInput); // strips unknown keys
```

### Cross-question: *"Does this affect React specifically?"*

Not directly — React itself isn't vulnerable. But if prototype is polluted before React renders, it could affect:
- Component props (inherited `isAdmin: true`)
- Context values
- Object spread operations

**Fix:** Always validate/parse user input with a schema validator. Never blindly merge untrusted objects.

---

### Scenario 5: Security Audit Checklist for React App

> **"You're asked to audit a React app's security before launch. What do you check?"**

```
AUTHENTICATION
[ ] Tokens stored in HttpOnly cookies (not localStorage)
[ ] Access tokens are short-lived (< 15 min)
[ ] Refresh tokens are single-use with rotation
[ ] Password reset tokens expire and are single-use
[ ] Rate limiting on login endpoint
[ ] Account lockout after N failed attempts

AUTHORIZATION
[ ] API validates permissions on EVERY endpoint
[ ] Client-side route guards are NOT the only check
[ ] Admin endpoints have role-based access control
[ ] Multi-tenant data isolation verified

XSS PREVENTION
[ ] No dangerouslySetInnerHTML without DOMPurify
[ ] href/src attributes validated (no javascript: protocol)
[ ] CSP header configured with nonces
[ ] No eval() or new Function() with user input
[ ] Third-party scripts loaded with SRI

CSRF PREVENTION
[ ] SameSite=Strict on auth cookies
[ ] CSRF token on state-changing requests (POST/PUT/DELETE)
[ ] Origin header validated server-side

DATA SECURITY
[ ] No secrets in frontend code (VITE_* vars are public!)
[ ] Sensitive data not logged to console
[ ] Secrets not in URL parameters
[ ] Payment data handled by PCI-compliant provider (Stripe)
[ ] React DevTools disabled in production

DEPENDENCIES
[ ] npm audit passes with no critical/high vulnerabilities
[ ] package-lock.json committed
[ ] No known malicious dependencies
[ ] CI pipeline runs audit on every build

HEADERS
[ ] Strict-Transport-Security (HSTS)
[ ] X-Content-Type-Options: nosniff
[ ] X-Frame-Options: DENY
[ ] Referrer-Policy set
[ ] Permissions-Policy restricts unused APIs

SSR / SERVER COMPONENTS
[ ] User data escaped in server-rendered HTML
[ ] Server Actions validate auth + input
[ ] No SSRF via user-controlled URLs
[ ] Internal data not leaked to client props
```

---

### Scenario 6: Open Redirect Vulnerability

> **"Your login page redirects to `?returnUrl=/dashboard` after login. An attacker crafts `?returnUrl=https://evil.com`. How do you prevent this?"**

```tsx
// ❌ Naive redirect — open redirect vulnerability
function LoginPage() {
  const searchParams = useSearchParams();
  const navigate = useNavigate();

  const handleLogin = async (credentials) => {
    await login(credentials);
    navigate(searchParams.get('returnUrl') || '/'); // attacker controls URL!
  };
}

// ✅ Validate redirect URL
function LoginPage() {
  const searchParams = useSearchParams();
  const navigate = useNavigate();

  const handleLogin = async (credentials) => {
    await login(credentials);
    const returnUrl = searchParams.get('returnUrl') || '/';
    
    // Only allow relative URLs (no external redirects)
    const isRelative = returnUrl.startsWith('/') && !returnUrl.startsWith('//');
    navigate(isRelative ? returnUrl : '/');
  };
}
```

### Cross-question: *"Why check for `//` as well?"*

`//evil.com` is a **protocol-relative URL** — the browser resolves it as `https://evil.com`. Simply checking `startsWith('/')` would allow it through. The double-slash check blocks this vector.

---

*Last updated: February 2026 | React 18 + React 19 · Next.js 14/15 · OWASP Top 10*
