---
title: "Troubleshooting"
description: "Common issues and solutions for backoffice authentication and authorization."
---

## Access Denied Errors

### "Access denied. Backoffice organization membership required."

**Cause:** User's active organization is not the backoffice org.

**Solutions:**

1. **Verify user is member of backoffice org:**
   - Go to Clerk Dashboard → Organizations → Select backoffice org
   - Check if user is listed as member
   - If not, add them with appropriate role

2. **Ensure user has switched to backoffice org:**
   - Users may be in a personal or tenant org context
   - Use organization switcher to select backoffice org

3. **Check environment configuration:**
   ```bash
   # Verify CLERK_INTERNAL_ORG_ID matches Clerk dashboard
   echo $CLERK_INTERNAL_ORG_ID
   ```

---

### "Staff account deactivated"

**Cause:** The `is_active` flag in `back_office_agents` is `false`.

**Solutions:**

1. **Check staff record:**
   ```sql
   SELECT * FROM back_office_agents 
   WHERE clerk_user_id = 'user_xxx';
   ```

2. **If intentional:** User needs admin to reactivate them.

3. **If unintentional:** Reactivate the account:
   ```sql
   UPDATE back_office_agents 
   SET is_active = true 
   WHERE clerk_user_id = 'user_xxx';
   ```

---

### "Staff registration required"

**Cause:** User has Clerk account but no `back_office_agents` record.

**Solutions:**

1. **Create staff record manually:**
   ```sql
   INSERT INTO back_office_agents (
     id, clerk_user_id, role, is_active, created_at, updated_at
   ) VALUES (
     gen_random_uuid(),
     'user_xxx',
     'member',
     true,
     now(),
     now()
   );
   ```

2. **Or trigger the webhook** that auto-creates staff records when users are added to the backoffice organization.

---

### "Email not from allowed domain"

**Cause:** User's primary email doesn't match `CLERK_ALLOWED_BACKOFFICE_DOMAINS`.

**Solutions:**

1. **Check allowed domains:**
   ```bash
   # Should be comma-separated, e.g., @bumara.com,@bumara.co.zm
   echo $CLERK_ALLOWED_BACKOFFICE_DOMAINS
   ```

2. **Verify user's primary email in Clerk Dashboard**

3. **Add domain to allowlist if legitimate:**
   ```bash
   CLERK_ALLOWED_BACKOFFICE_DOMAINS=@bumara.com,@bumara.co.zm,@newdomain.com
   ```

---

## MFA Issues

### Infinite Redirect Loop

**Cause:** MFA guard redirects to setup page, which also requires MFA.

**Solutions:**

1. **Verify `/setup-mfa` is in exempt paths:**
   
   Check `apps/backoffice/proxy.ts`:
   ```typescript
   const isOrgSelectRoute = createRouteMatcher([
     "/org-select(.*)",
     "/sign-in(.*)",
     "/access-denied(.*)",
     "/setup-mfa(.*)",  // Must be included
   ]);
   ```

2. **Verify MFA guard exemptions:**
   
   Check `apps/backoffice/components/shared/auth/mfa-guard.tsx`:
   ```typescript
   const MFA_EXEMPT_PATHS = [
     "/setup-mfa",  // Must be included
     "/sign-in",
     "/sign-out",
     "/access-denied",
   ];
   ```

3. **Check route group structure:**
   
   The `/setup-mfa` page should be in `(mfa-setup)` route group, not `(authenticated)`:
   ```
   apps/backoffice/app/
   ├── (mfa-setup)/         # Has auth but NO MfaGuard
   │   └── setup-mfa/
   └── (authenticated)/     # Has MfaGuard
   ```

---

### "Incorrect code" Error During Sign-In

**Cause:** TOTP code race condition - code submitted before state updated.

**Solution:**

Ensure the code is passed directly to the handler:

```tsx
// ❌ BAD - State race condition
const handleCodeChange = (value: string) => {
  setCode(value);
  if (value.length === 6) {
    handle2FASubmit();  // Uses stale state
  }
};

// ✅ GOOD - Pass value directly
const handleCodeChange = (value: string) => {
  setCode(value);
  if (value.length === 6) {
    handle2FASubmit(undefined, value);  // Pass value directly
  }
};
```

---

### "Session verification required" During MFA Setup

**Cause:** Clerk requires a fresh session for sensitive operations.

**Solution:**

The user needs to sign out and sign back in:

```tsx
// Redirect to sign-in with return URL
window.location.href = "/sign-in?redirect_url=/setup-mfa";
```

---

### Can't Generate Backup Codes

**Cause:** Clerk requires password reverification for backup code generation.

**Solutions:**

1. **Current behavior:** MFA setup completes without backup codes; user can generate them later from settings.

2. **Generate later:**
   - Go to Security Settings
   - Click "Generate Backup Codes"
   - Complete password verification if prompted

---

## CORS Errors

### "Access-Control-Allow-Origin" Error

**Cause:** Request origin not in backend's allowlist.

**Solutions:**

1. **Check `packages/backend/src/core/http/create-app.ts`:**
   ```typescript
   const ALLOWED_ORIGINS = [
     "https://app.bumara.com",
     "https://backoffice.bumara.com",
     "http://localhost:3000",
     "http://localhost:3003",
     // Add your origin here
   ];
   ```

2. **Ensure using correct localhost:**
   - Use `http://localhost:3000` not `http://127.0.0.1:3000`
   - Or add both to the allowlist

3. **For Vercel previews:**
   ```typescript
   /^https:\/\/bumara-.*\.vercel\.app$/,
   ```

---

## Environment Issues

### "Invalid environment variables" Error

**Cause:** Missing or malformed environment variables.

**Example Error:**
```
❌ Invalid environment variables: [
  { expected: 'boolean', code: 'invalid_type', path: ['REQUIRE_MFA'], message: 'Invalid input: expected boolean, received string' }
]
```

**Solutions:**

1. **Check all required variables are set:**
   ```bash
   # List all CLERK_ variables
   env | grep CLERK_
   env | grep REQUIRE_MFA
   ```

2. **For boolean variables:** Use string values that transform correctly:
   ```bash
   # apps/backoffice/env.ts transforms "true"/"false" strings
   REQUIRE_MFA=true  # or "false"
   ```

3. **Verify no typos in variable names**

---

### CLERK_INTERNAL_ORG_ID Not Matching

**Symptoms:** Access denied even though user is in organization.

**Solution:**

1. Get org ID from Clerk Dashboard:
   - Organizations → Select backoffice org → Copy Organization ID

2. Compare with environment:
   ```bash
   echo $CLERK_INTERNAL_ORG_ID
   ```

3. Update if different:
   ```bash
   CLERK_INTERNAL_ORG_ID=org_correct_id_here
   ```

---

## Role Issues

### User Has Role But Still Denied

**Cause:** Role might not be recognized as valid backoffice role.

**Solutions:**

1. **Check valid roles in `packages/auth/helpers.ts`:**
   ```typescript
   export const BACKOFFICE_ROLES = [
     "org:admin",
     "org:manager",
     "org:member",
     "org:backoffice_admin",
     "org:backoffice_manager",
     "org:backoffice_member",
   ];
   ```

2. **Verify exact role string in Clerk:**
   - Dashboard → Organizations → Members → Check role

3. **Custom roles must follow format:** `org:role_name`

---

### Can't Approve - Need Manager Role

**Cause:** User has member role but approver middleware requires manager+.

**Solutions:**

1. **Upgrade user's role** in Clerk Dashboard

2. **Or check middleware requirements:**
   ```typescript
   requireBackofficeApprover  // Requires admin or manager
   ```

---

## Debugging Tips

### Enable Debug Logging

Add console logs to trace auth flow:

```typescript
// In proxy.ts
console.log("[Middleware] userId:", userId, "orgId:", orgId);

// In MfaGuard
console.log("[MfaGuard] pathname:", pathname, "REQUIRE_MFA:", env.REQUIRE_MFA);
```

### Check Network Tab

1. Open browser DevTools → Network
2. Look for 401/403 responses
3. Check response body for error details

### Verify JWT Claims

```typescript
// In API middleware
console.log("Auth claims:", {
  userId: c.get("userId"),
  orgId: c.get("orgId"),
  orgRole: c.get("orgRole"),
});
```

### Test with curl

```bash
# Get a valid token from browser DevTools (Application → Cookies → __session)
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8787/api/backoffice/test
```

---

## Getting Help

If you've tried the above solutions and still have issues:

1. **Check logs** in terminal and browser console
2. **Verify Clerk Dashboard** configuration matches environment
3. **Review recent changes** to auth middleware or guards
4. **Contact engineering** with:
   - Error message
   - User ID and email
   - Steps to reproduce
   - Relevant log output

---

## Quick Reference

| Issue | Likely Cause | Quick Fix |
|-------|--------------|-----------|
| Access denied | Wrong organization | Switch org in UI |
| Staff deactivated | is_active = false | Update DB record |
| Staff not found | No DB record | Create record or trigger webhook |
| Invalid domain | Email not allowed | Add domain to env var |
| MFA redirect loop | Missing exempt path | Add to exempt paths |
| Incorrect code | State race condition | Pass value directly |
| CORS error | Origin not allowed | Add to ALLOWED_ORIGINS |
| Role denied | Insufficient permissions | Upgrade role in Clerk |

