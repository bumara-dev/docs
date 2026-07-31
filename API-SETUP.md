---
title: "API SETUP"
description: "This guide walks you through configuring the API application (packages/backend) with a Neon PostgreSQL database (via Drizzle ORM), exposing local..."
---

## Bumara API Setup (Hono + Neon + ngrok)

This guide walks you through configuring the API application (`packages/backend`) with a Neon PostgreSQL database (via Drizzle ORM), exposing local webhooks with ngrok, and running the Hono development server.


## Prerequisites

- Node.js 20+ and pnpm installed
- Neon account (PostgreSQL)
- Clerk account and application (for auth + webhooks)
- ngrok account and CLI
- Cloudflare Wrangler CLI


## 1) Install dependencies

Run from the repository root:

```bash
pnpm install
```


## 2) Create a Neon database

1. Sign in to Neon and create a new project.
2. Create (or use) the default branch and database.
3. Copy the connection string in PostgreSQL format. It will look like:
   - `postgresql://<user>:<password>@<host>/<database>?sslmode=require`

Notes:
- SSL is required on Neon; keep `sslmode=require`.
- This repo uses the Neon HTTP driver (`@neondatabase/serverless` + `drizzle-orm/neon-http`).


## 3) Configure API environment variables

Create `packages/backend/.env` with at least the following values:

```bash
# App & runtime
NODE_ENV=development
LOG_LEVEL=info

# Database (Neon)
DATABASE_URL=postgresql://<user>:<password>@<host>/<database>?sslmode=require

# Clerk (required for auth + webhooks)
CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxx
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxx
CLERK_WEBHOOK_SIGNING_SECRET=whsec_xxxxxxxxxxxxxxxxx

# Optional: 32+ chars if you plan to store encrypted credentials
# CREDENTIALS_ENCRYPTION_KEY=your_32_plus_char_random_string
```

Security reminders:
- Do not commit `.env` files. Keep secrets out of version control.
- Rotate keys immediately if they are ever exposed.


## 4) Generate and apply database migrations (Drizzle)

From the repository root, run the API package scripts with pnpm filter:

```bash
# Generate migrations from current schema
pnpm --filter @repo/database db:generate

# Apply migrations to the Neon database
pnpm --filter @repo/database db:migrate

# (Optional) Explore the DB with Drizzle Studio or neon website 
pnpm --filter @repo/database db:studio
```

These scripts are defined in `packages/backend/package.json` and read `DATABASE_URL` via `packages/backend/drizzle.config.ts`.


## 5) Launch the Hono development server (Wrangler)

The API runs as a Cloudflare Worker in development using Wrangler.

```bash
cd packages/backend
pnpm dev
```

Once started, the server will be available at:

- Base URL (default): `http://127.0.0.1:8787`
- API base path: `/api/v1`
- OpenAPI JSON: `/api/v1/doc`
- API Reference UI: `/api/v1/reference`

Webhook routes are mounted at the root (no `/api/v1` prefix). For example:

- Clerk webhook endpoint: `POST /webhooks/clerk`


## 6) Expose local webhooks using ngrok

1. Install and authenticate ngrok:
   ```bash
   # If not installed, see ngrok docs for your OS
   ngrok config add-authtoken <YOUR_NGROK_AUTHTOKEN>
   ```
2. Start a tunnel to your local API port (Wrangler default is 8787):
   ```bash
   ngrok http 8787
   ```
3. Copy the generated HTTPS forwarding URL, for example:
   - `https://abcd-1234.ngrok-free.app`
4. Your public webhook URL for Clerk becomes:
   - `https://abcd-1234.ngrok-free.app/webhooks/clerk`


## 7) Configure the Clerk webhook

1. In the Clerk Dashboard, go to Webhooks and add a new endpoint.
2. Set the endpoint URL to your ngrok URL plus `/webhooks/clerk` (see above).
3. Copy the “Signing secret” and set it as `CLERK_WEBHOOK_SIGNING_SECRET` in `packages/backend/.env`.
4. In Clerk, send a test event. You should get a 200 response if verification passes.

If headers are missing or the signature is invalid, the API will respond with a 400 (by design). Ensure `CLERK_WEBHOOK_SIGNING_SECRET` matches exactly.


## 8) Verify the API locally

With Wrangler running:

```bash
# Check OpenAPI spec
curl http://127.0.0.1:8787/api/v1/doc

# Open the interactive API reference in a browser
curl http://127.0.0.1:8787/api/v1/reference

```

Optional quick webhook check (expect 400 due to missing headers — this just confirms the route exists):

```bash
curl -i -X POST http://127.0.0.1:8787/webhooks/clerk
```


## 9) Troubleshooting

- Database connection issues:
  - Verify `DATABASE_URL` uses your Neon credentials and includes `sslmode=require`.
  - Ensure migrations were applied: `pnpm --filter @repo/backend db:migrate`.
- Wrangler dev errors:
  - Try `pnpm --filter @repo/backend dev -- --local` to run without Cloudflare login.
  - Confirm Node.js version is 20+.
- Webhook 400 errors:
  - 400 is expected if Clerk headers/signature are missing.
  - For end-to-end tests, use Clerk’s “Send test event” and confirm `CLERK_WEBHOOK_SECRET` is set.
- Frontend -> API communication:
  - If your Next.js app needs to call the API, set `NEXT_PUBLIC_API_URL` in the frontend env to your local API base, e.g. `http://127.0.0.1:8787`.

 
 ## 10) Create a Clerk JWT template for Hono
 
 Create a short‑lived JWT template in Clerk for API authentication.
 
 1. Open Clerk Dashboard → JWT Templates → Create template.
 2. Name: `hono`
 3. Token lifetime: `36000` seconds
 4. Claims (Custom claims JSON):
 
 ```json
 {
   "orgId": "{{org.id}}",
   "userId": "{{user.id}}",
   "orgRole": "{{org.role}}"
 }
 ```
 
 5. Save the template.
 
 Notes:
 - Keep the lifetime short (36000s) to reduce risk if a token is leaked.
 - Ensure your API validates the token and pulls these claims from the verified JWT payload.
 

## Notes on Compliance & Security

- Never log sensitive data (tax IDs, NAPSA numbers, financial info).
- Keep all API keys and secrets in environment variables.
- Add audit logs for compliance-related actions where applicable.
- Implement rate limits and retries when integrating external regulators.

 
 ## 11) Get a user JWT from the frontend and test in API docs
 
 Use the dev helper exposed by the frontend to mint a Clerk JWT using your `hono` template, then use it in the Hono API Reference UI.
 
 1. Run the frontend in development:
    ```bash
    cd apps/app
    pnpm dev
    # opens http://localhost:3000
    ```
 2. In the browser, sign in or create an account (Clerk).
 3. On the home page (`/`), open the browser DevTools Console and run:
    ```js
    await window.getApiToken({ template: 'hono' })
    ```
    - This returns a JWT in the console. Copy the token string.
 4. Open the Hono API Reference UI at:
    - `http://127.0.0.1:8787/api/v1/reference`
 5. Click “Authorize” (bearerAuth), paste the JWT (token only, no “Bearer ” prefix), and save.
 6. Call any authenticated endpoint. You should receive authorized responses (e.g., 200) instead of 401/403.
 
 
 ## 12) How the frontend connects to the Hono backend
 
 The frontend uses a typed API client generated from the Hono router and points to the API base URL defined in the frontend environment.
 
 - Set the frontend API base URL in `apps/app/.env.local`:
   ```bash
   NEXT_PUBLIC_API_URL=http://127.0.0.1:8787
   ```
- The client comes from `@repo/api-client` and is wrapped by `createAuthenticatedClient` to attach a Clerk JWT as a Bearer token. The helper memoizes the base URL from `NEXT_PUBLIC_API_URL` to keep every call consistent and appends the `Authorization` header when a token is available:

```ts
// apps/app/lib/api-client.ts
export function createAuthenticatedClient(
  getToken?: () => Promise<string | null>
) {
  if (!getToken) {
    return apiClient();
  }

  const authenticatedFetch: typeof fetch = async (input, init) => {
    const headers = new Headers(init?.headers);
    const token = await getToken();

    if (token) {
      headers.set('Authorization', `Bearer ${token}`);
    }

    return fetch(input, { ...init, headers });
  };

  return apiClient({ fetch: authenticatedFetch });
}
```
 
- Backend routes are mounted under `/api/v1` (see `packages/backend/src/core/config/base-path.ts`), and the generated client exposes them as typed methods, for example `client.regulators.$get()`.
 
 
 ## 13) Defining a React Query data hook
 
 Here’s a minimal pattern used in the repo to fetch authenticated data with React Query and Clerk:
 
 ```130:137:apps/app/lib/queries/regulators.ts
 export function useRegulators() {
   const { getToken } = useAuth();
 
   return useQuery({
     queryKey: ['regulators'],
     queryFn: () => fetchRegulators(getToken),
   });
 }
 ```
 
 Outline to create your own hook:
 - Use `useAuth()` to get `getToken()` from Clerk.
 - Create an API function that builds a client with `createAuthenticatedClient(getToken)` and calls the appropriate endpoint.
 - Wrap it in `useQuery({ queryKey, queryFn })` with a stable key.
 - Ensure `NEXT_PUBLIC_API_URL` points to your API and that you’ve configured the Clerk JWT template (section 10) if the route requires auth.
 
This completes the API setup for local development with Neon, ngrok, and Hono.

