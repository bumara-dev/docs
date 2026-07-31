---
title: "Authentication Flow"
description: "Complete sign-in, MFA, and session management for the backoffice."
---

The backoffice uses a custom sign-in implementation with Clerk hooks (not pre-built components) to provide full control over the authentication experience.

---

## Sign-In Flow Overview

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Step 1     │     │   Step 2     │     │   Step 3     │     │   Complete   │
│   Email      │────▶│   Password   │────▶│   2FA Code   │────▶│   Dashboard  │
│              │     │              │     │  (if enabled) │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                │
                                                ▼ (if MFA not setup)
                                          ┌──────────────┐
                                          │  /setup-mfa  │
                                          │  (mandatory) │
                                          └──────────────┘
```

---

## Custom Sign-In Form

**File:** `apps/backoffice/components/auth/sign-in-form.tsx`

The sign-in form uses Clerk's `useSignIn` hook for a three-step authentication process.

### Step 1: Email Entry

```tsx
const handleIdentifierSubmit = async (e: FormEvent) => {
  e.preventDefault();
  
  const result = await signIn.create({
    identifier: identifier.trim(),
  });

  if (result.status === "needs_first_factor") {
    setStep("password");
  } else if (result.status === "complete") {
    await setActive({ session: result.createdSessionId });
    router.push("/");
  }
};
```

### Step 2: Password Entry

```tsx
const handlePasswordSubmit = async (e: FormEvent) => {
  e.preventDefault();
  
  const result = await signIn.attemptFirstFactor({
    strategy: "password",
    password,
  });

  if (result.status === "complete") {
    // No MFA required
    await setActive({ session: result.createdSessionId });
    router.push("/");
  } else if (result.status === "needs_second_factor") {
    // MFA required - move to 2FA step
    setStep("2fa");
  }
};
```

### Step 3: Two-Factor Authentication

```tsx
const handle2FASubmit = async (e?: FormEvent, codeOverride?: string) => {
  e?.preventDefault();
  
  const codeToSubmit = codeOverride || code;
  
  const result = await signIn.attemptSecondFactor({
    strategy: useBackupCode ? "backup_code" : "totp",
    code: codeToSubmit,
  });

  if (result.status === "complete") {
    await setActive({ session: result.createdSessionId });
    router.push("/");
  }
};
```

### Auto-Submit on 6 Digits

The TOTP code auto-submits when 6 digits are entered:

```tsx
const handleCodeChange = (value: string) => {
  setCode(value);
  // Pass value directly to avoid React state race condition
  if (value.length === 6 && !useBackupCode && !isLoading) {
    handle2FASubmit(undefined, value);
  }
};
```

---

## MFA Enforcement

Backoffice requires mandatory MFA when `REQUIRE_MFA=true` (default).

### MfaGuard Component

**File:** `apps/backoffice/components/shared/auth/mfa-guard.tsx`

```tsx
export async function MfaGuard({ children, redirectTo = "/setup-mfa" }) {
  // Skip if not required by environment
  if (!env.REQUIRE_MFA) {
    return <>{children}</>;
  }

  // Exempt paths to prevent redirect loops
  const MFA_EXEMPT_PATHS = [
    "/setup-mfa",
    "/sign-in",
    "/sign-out",
    "/access-denied",
  ];

  const pathname = getPathnameFromHeaders(headersList);
  
  const isExemptPath =
    pathname === "" || // Can't determine path, don't redirect
    MFA_EXEMPT_PATHS.some((exemptPath) => pathname.startsWith(exemptPath));

  if (isExemptPath) {
    return <>{children}</>;
  }

  const user = await currentUser();

  if (!user?.twoFactorEnabled) {
    redirect(redirectTo);
  }

  return <>{children}</>;
}
```

### Path Detection

The guard uses multiple fallback methods to detect the current path:

```tsx
function getPathnameFromHeaders(headersList: Headers): string {
  // Try custom header (set by proxy.ts)
  const xPathname = headersList.get("x-pathname");
  if (xPathname) return xPathname;

  // Try Next.js internal header
  const xInvokePath = headersList.get("x-invoke-path");
  if (xInvokePath) return xInvokePath;

  // Try referer URL
  const referer = headersList.get("referer");
  if (referer) {
    try {
      return new URL(referer).pathname;
    } catch {}
  }

  return "";
}
```

---

## MFA Setup Page

**File:** `apps/backoffice/app/(mfa-setup)/setup-mfa/page.tsx`

The MFA setup is a dedicated page outside the main authenticated layout to prevent redirect loops.

### Route Group Structure

```
apps/backoffice/app/
├── (mfa-setup)/           # Separate route group
│   ├── layout.tsx         # Has auth guards but NO MfaGuard
│   └── setup-mfa/
│       └── page.tsx       # MFA setup wizard
```

### Setup Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Intro     │     │   Scan QR    │     │ Verify Code  │     │ Backup Codes │
│  (Get Ready) │────▶│   (or manual)│────▶│  (6 digits)  │────▶│  (Save them) │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                      │
                                                                      ▼
                                                              ┌──────────────┐
                                                              │   Complete   │
                                                              │  → Dashboard │
                                                              └──────────────┘
```

### Step 1: Introduction

Explains that MFA is mandatory for backoffice access and what's needed.

### Step 2: Scan QR Code

```tsx
const handleStartSetup = async () => {
  await user.reload();
  
  const totp = await user.createTOTP();
  
  setQrCodeUri(totp.uri ?? null);
  setTotpSecret(totp.secret ?? null);
  setStep("scan");
};
```

Displays QR code and manual entry option for the secret key.

### Step 3: Verify Code

```tsx
const handleVerifyCode = async (codeToVerify?: string) => {
  const code = codeToVerify || verificationCode;
  
  const result = await user.verifyTOTP({ code });
  
  // Try to create backup codes (optional)
  try {
    const backupCodeResource = await user.createBackupCode();
    setBackupCodes(backupCodeResource.codes ?? []);
    setStep("backup");
  } catch {
    // MFA is still enabled, skip backup codes
    setStep("complete");
  }
};
```

### Step 4: Backup Codes

Users can copy or download backup codes. If backup code generation fails (e.g., session verification required), MFA setup still completes.

### Session Verification Handling

If Clerk requires session reverification:

```tsx
if (errorCode === "session_step_up_verification_required") {
  // Prompt user to sign out and back in
  setStep("password");
  setError("Your session needs to be refreshed. Please sign out and sign back in.");
}
```

---

## Backup Codes

Backup codes provide account recovery if the authenticator is lost.

### Usage During Sign-In

The sign-in form allows toggling between TOTP and backup codes:

```tsx
<button onClick={toggleBackupCode} type="button">
  {useBackupCode ? "Use authenticator app" : "Use a backup code"}
</button>
```

### Backup Code Format

```
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
XXXX-XXXX
```

Each code can only be used once.

---

## Session Management

### Session Creation

After successful authentication:

```tsx
await setActive({ session: result.createdSessionId });
router.push("/");
```

### Session Claims

The Clerk JWT includes organization claims:

```typescript
type ClerkSessionClaims = {
  sub?: string;           // User ID
  sid?: string;           // Session ID
  org_id?: string;        // Organization ID
  org_role?: string;      // Role (e.g., "org:admin")
  org_slug?: string;      // Organization slug
  org_permissions?: string[]; // Permission strings
};
```

### Context Extraction

The API middleware extracts claims:

```typescript
export const attachAuthToContext = createMiddleware<AppBindings>(
  async (c, next) => {
    const auth = getAuth(c);
    const claims = auth?.sessionClaims;

    c.set("userId", auth?.userId || claims?.sub);
    c.set("orgId", auth?.orgId || claims?.org_id);
    c.set("orgRole", auth?.orgRole || claims?.org_role);
    c.set("orgPermissions", auth?.orgPermissions || claims?.org_permissions);

    await next();
  }
);
```

---

## Error Handling

### Sign-In Errors

| Error Code | Message | Resolution |
|------------|---------|------------|
| `form_identifier_not_found` | No account found | Check email address |
| `form_password_incorrect` | Incorrect password | Re-enter password |
| `form_code_incorrect` | Invalid code | Check authenticator |
| `session_step_up_verification_required` | Session needs refresh | Sign out and back in |

### Error Display

```tsx
{error && (
  <Alert variant="destructive">
    <AlertCircle className="size-4" />
    <AlertDescription>{error}</AlertDescription>
  </Alert>
)}
```

---

## Security Considerations

### Prevent Timing Attacks

The code uses consistent error messages to prevent username enumeration.

### State Race Condition Fix

When auto-submitting TOTP, the code is passed directly to avoid React state update delays:

```tsx
// WRONG: State may not be updated yet
if (value.length === 6) {
  handle2FASubmit(); // Uses stale state
}

// CORRECT: Pass value directly
if (value.length === 6) {
  handle2FASubmit(undefined, value);
}
```

### Session Refresh for Sensitive Operations

Clerk may require password reverification for sensitive operations like enabling MFA. The setup page handles this gracefully.

---

## Next Steps

- [Authorization](/backoffice/backoffice-auth/authorization) - Role and permission details
- [Troubleshooting](/backoffice/backoffice-auth/troubleshooting) - Common authentication issues

