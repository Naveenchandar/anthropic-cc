# Authentication — Developer Guide

This document explains how authentication works end-to-end in UIGen so new team members can understand it quickly.

---

## Overview

UIGen supports two modes:

| Mode | Description |
|---|---|
| **Anonymous** | Use the app without logging in. Work is saved in `sessionStorage` only. |
| **Authenticated** | Sign in / sign up to persist projects to the database (SQLite via Prisma). |

Anonymous work is **not lost** on sign-in — it is automatically migrated to a new project.

---

## Tech Stack

| Concern | Library / Approach |
|---|---|
| Session token | JWT (`jose`) |
| Token storage | HTTP-only cookie (`auth-token`, 7-day expiry) |
| Password hashing | `bcrypt` (10 rounds) |
| DB access | Prisma ORM → SQLite |
| Anonymous tracking | `sessionStorage` (`uigen_anon_data`) |
| Route protection | Next.js middleware |

---

## File Map

```
src/
├── lib/
│   ├── auth.ts                  # JWT create/read/delete session helpers
│   └── anon-work-tracker.ts     # sessionStorage helpers for anonymous work
├── actions/
│   └── index.ts                 # Server Actions: signUp, signIn, signOut, getUser
├── hooks/
│   └── use-auth.ts              # Client hook: wraps server actions + post-login routing
├── components/auth/
│   ├── AuthDialog.tsx           # Modal toggling between sign-in and sign-up views
│   ├── SignInForm.tsx           # Email + password form → calls useAuth().signIn()
│   └── SignUpForm.tsx           # Email + password + confirm → calls useAuth().signUp()
└── middleware.ts                # Protects /api/projects and /api/filesystem routes
```

---

## Step-by-Step: Sign Up Flow

```
User fills SignUpForm
        │
        ▼
useAuth().signUp(email, password)          [src/hooks/use-auth.ts]
        │
        ▼
signUp() Server Action                     [src/actions/index.ts]
  1. Validate: email + password present, password ≥ 8 chars
  2. prisma.user.findUnique() — reject if email already exists
  3. bcrypt.hash(password, 10)
  4. prisma.user.create({ email, hashedPassword })
  5. createSession(user.id, user.email)    [src/lib/auth.ts]
       → Signs a JWT (HS256, 7d expiry)
       → Sets HTTP-only cookie "auth-token"
  6. Returns { success: true }
        │
        ▼
handlePostSignIn()                         [src/hooks/use-auth.ts]
  • If sessionStorage has anonymous work →
      createProject(anonWork) → redirect to new project
  • Else if user has existing projects →
      redirect to most recent project
  • Else →
      createProject(empty) → redirect to new project
```

---

## Step-by-Step: Sign In Flow

```
User fills SignInForm
        │
        ▼
useAuth().signIn(email, password)          [src/hooks/use-auth.ts]
        │
        ▼
signIn() Server Action                     [src/actions/index.ts]
  1. prisma.user.findUnique({ email })
  2. bcrypt.compare(password, user.password)
  3. On match → createSession(user.id, user.email)
       → Signs JWT, sets HTTP-only cookie
  4. Returns { success: true }
        │
        ▼
handlePostSignIn()                         [same as sign-up above]
```

---

## Step-by-Step: Session Verification

Every request passes through `src/middleware.ts`:

```
Incoming Request
        │
        ▼
middleware()
  • Reads "auth-token" cookie from request
  • jwtVerify(token, JWT_SECRET)           [src/lib/auth.ts → verifySession()]
  • If path is /api/projects or /api/filesystem AND no valid session
      → 401 { error: "Authentication required" }
  • Otherwise → NextResponse.next()
```

Server Actions and API routes that need the current user call `getSession()`:

```ts
// src/lib/auth.ts
export async function getSession(): Promise<SessionPayload | null>
// Reads cookie store, verifies JWT, returns { userId, email, expiresAt } or null
```

---

## Anonymous Work Migration

When a user works without logging in, `setHasAnonWork()` is called (from `src/lib/anon-work-tracker.ts`) with the current chat messages and file system state. This is stored in `sessionStorage` under two keys:

| Key | Value |
|---|---|
| `uigen_has_anon_work` | `"true"` |
| `uigen_anon_data` | JSON `{ messages, fileSystemData }` |

On sign-in or sign-up success, `handlePostSignIn()` in `use-auth.ts`:
1. Checks `getAnonWorkData()` — if data exists, creates a new project with it
2. Calls `clearAnonWork()` to remove both `sessionStorage` keys
3. Redirects the user to the newly created project

This means **no anonymous work is lost** when the user decides to create an account.

---

## Sign Out Flow

```ts
// src/actions/index.ts
export async function signOut() {
  await deleteSession();   // deletes "auth-token" cookie
  revalidatePath("/");
  redirect("/");
}
```

---

## JWT Secret

The JWT is signed with `process.env.JWT_SECRET`. If this env var is absent, it falls back to `"development-secret-key"`. **Always set `JWT_SECRET` in production.**

---

## AuthDialog Component

`AuthDialog` is a controlled modal (Radix UI `Dialog`) that:
- Accepts `open`, `onOpenChange`, and `defaultMode` (`"signin"` | `"signup"`) props
- Renders `SignInForm` or `SignUpForm` based on internal `mode` state
- Provides a toggle link so users can switch between modes
- Calls `onOpenChange(false)` on successful authentication to close itself

---

## Key Invariants

- Passwords are **never** stored in plaintext — always bcrypt-hashed.
- The JWT cookie is **HTTP-only** — JavaScript cannot read it.
- `secure: true` is only set in production (`NODE_ENV === "production"`).
- Anonymous projects have `userId: null` in the database — they are not associated with any account until migrated.
