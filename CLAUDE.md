# Research Hub — Architecture Reference

## What This App Is

Research Hub is a data dashboard for Eduversal's research and academic staff. It displays university placements, scholarship records, Olympiad results, teacher databases, and related analytics. It is a **vanilla HTML/CSS/JS application** (no React, no bundler framework). Pages are plain `.html` files with inline scripts that load Firebase via CDN.

**Deployment:** Vercel (build output in `dist/`).

---

## Monorepo Structure

```
Eduversal Web/                    ← monorepo root (not a deployed app)
├── Academic Hub/                 ← analytics dashboards (Vercel)
├── Central Hub/                  ← admin control panel (Vercel)
│   ├── firestore.rules           ← ⚠️ ONLY Firestore rules file — deploy from here
│   └── firebase.json             ← firebase deploy config
├── Teachers Hub/                 ← teacher tools (Vercel)
├── Research Hub/                 ← THIS app (Vercel)
└── keys/                         ← service account JSON keys (gitignored)
```

Each app has its **own GitHub repository** and its **own deployment target**, but all share the single Firebase backend `centralhub-8727b`.

---

## Shared Firebase Backend

**Project ID:** `centralhub-8727b`

| Field                | Value                                      |
|----------------------|--------------------------------------------|
| authDomain           | centralhub-8727b.firebaseapp.com           |
| projectId            | centralhub-8727b                           |
| storageBucket        | centralhub-8727b.firebasestorage.app       |
| messagingSenderId    | 244951050014                               |
| apiKey / appId       | gitignored — see Firebase Console          |

**SDK:** Firebase modular v10 (`10.7.1`), loaded from the CDN:
```
https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js
https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js
https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js
```
Do NOT use the compat SDK (`firebase/app`, `firebase.firestore()` namespace style). Always use modular imports.

---

## Firebase Config Pattern

**`firebase-config.js`** (gitignored) sets `window.ENV` at page load:
```js
window.ENV = {
  FIREBASE_API_KEY: "...",
  FIREBASE_AUTH_DOMAIN: "centralhub-8727b.firebaseapp.com",
  // ...
};
```

**Local development:** HTML pages include `<script src="firebase-config.js"></script>` plus an inline fallback:
```html
<script>
  if (!window.ENV) window.ENV = { FIREBASE_API_KEY: "...", ... };
</script>
```

**Production (Vercel):** `build.js` replaces `__FIREBASE_*__` placeholders and strips the `<script src="firebase-config.js">` tag. The `firebase-config.js` source file is NOT deployed.

**Template:** `firebase-config.example.js` — copy to `firebase-config.js` and fill in `apiKey` and `appId`.

---

## Auth Pattern

Every protected page loads `auth-guard.js` as a module:
```html
<script type="module" src="auth-guard.js"></script>
```

`auth-guard.js` (modular SDK v10):
1. Hides `document.body` immediately (prevents flash of content).
2. Initialises Firebase (guards against double-init with `getApps()`).
3. Listens on `onAuthStateChanged`. If no user → redirects to `index.html`.
4. Fetches (or creates) Firestore profile. If missing, creates it and assigns `role_researchhub: 'research_user'` automatically.
5. **Domain check** — only `@eduversal.org` Google SSO accounts are allowed. Email/password accounts bypass this check. Fails → `index.html?error=domain`.
6. Role check — `role_researchhub` must be in `['research_user', 'research_admin']`. Fails → `index.html?error=access`.
7. Exposes globals and dispatches `authReady`.

**`index.html`** (login page) handles auth inline and does NOT use `auth-guard.js`. It applies the same `eduversal.org` domain restriction via a local `isDomainAllowed()` function.

**Globals exposed after `authReady`:**
| Global               | Value                                |
|----------------------|--------------------------------------|
| `window.firebaseApp` | FirebaseApp instance                 |
| `window.auth`        | Auth instance                        |
| `window.db`          | Firestore instance                   |
| `window.currentUser` | firebase.User object                 |
| `window.userProfile` | Firestore `users/{uid}` document     |
| `window.firestoreOps`| `{ doc, getDoc, setDoc, serverTimestamp }` |

**Listening for auth in page scripts:**
```js
document.addEventListener('authReady', ({ detail: { user, profile } }) => {
  // safe to use window.db, window.currentUser, window.userProfile here
});
```

---

## Role System

Research Hub uses `role_researchhub` as its Firestore role field.

| Field              | Values                                          |
|--------------------|-------------------------------------------------|
| `role_researchhub` | `'research_user'` (default) \| `'research_admin'` |

**Allowed roles:** `['research_user', 'research_admin']`

First login automatically assigns `research_user`. `research_admin` must be set manually via Central Hub's `console.html`.

**isAdmin check pattern:**
```js
const isAdmin = profile?.role_researchhub === 'research_admin';
```

**Access is restricted to `@eduversal.org`** — unlike Academic Hub and Teachers Hub which allow 15 partner school domains, Research Hub is internal-only.

---

## Firestore Collections

**Firestore rules** live **exclusively** in `Central Hub/firestore.rules` — the single source of truth for all apps.

⚠️ **Always deploy rules from the `Central Hub/` directory:**
```bash
cd "Eduversal Web/Central Hub"
firebase deploy --only firestore:rules --project centralhub-8727b
```
Research Hub does NOT have its own `firestore.rules`. Never create one — it would overwrite the shared rules.

---

## Build & Deployment

**Platform:** Vercel
**Build command:** `node build.js`
**Output directory:** `dist/`
**Clean URLs:** enabled (`vercel.json` has `cleanUrls: true`)

### Pages:
| File                  | Purpose                            |
|-----------------------|------------------------------------|
| `index.html`          | Login / home (no auth guard)       |
| `placement.html`      | University placement dashboard     |
| `scholarships.html`   | Scholarship records                |
| `olympiad.html`       | Olympiad medal records             |
| `universities.html`   | University data                    |
| `teachers.html`       | Teacher database                   |

### Vercel environment variables required:
```
FIREBASE_API_KEY
FIREBASE_AUTH_DOMAIN
FIREBASE_PROJECT_ID
FIREBASE_STORAGE_BUCKET
FIREBASE_MESSAGING_SENDER_ID
FIREBASE_APP_ID
```

---

## Key Files

| File                         | Purpose                                                    |
|------------------------------|------------------------------------------------------------|
| `auth-guard.js`              | Auth + role + domain gate for protected pages              |
| `build.js`                   | Vercel build script — placeholder replacement              |
| `firebase-config.js`         | Local dev config (gitignored)                              |
| `firebase-config.example.js` | Template for firebase-config.js                            |
| `vercel.json`                | Vercel deployment config (cleanUrls, build cmd)            |
| `dist/`                      | Build output (not committed)                               |
| `resources/`                 | Source data files (xlsx, json)                             |

---

## Important Conventions

- **No React, no npm bundler.** All JS runs directly in the browser via CDN ESM imports.
- **Always use modular SDK v10.** Never use the compat namespace (`firebase.firestore()` etc.).
- **`createdAt` not `timestamp`** for all Firestore timestamp fields.
- **Never commit `firebase-config.js`.** It is in `.gitignore`.
- **Auth guard goes first.** On protected pages, `auth-guard.js` must be the first `<script type="module">` tag.
- **Use `authReady` event** to gate all Firestore reads — never call `window.db` before the event fires.
- **`@eduversal.org` only** — do not expand the allowed domain list to partner school domains. Research Hub is internal staff only.
