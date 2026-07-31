---
title: "Bumara - Local Development Setup"
description: "Before you start, install these:"
---

## Prerequisites

Before you start, install these:

- [ ] **Node.js 20+** - [Download](https://nodejs.org/)

```bash
  node --version  # Should be v20 or higher
  npm install -g pnpm
  pnpm --version  # Should be 8.0+
  git --version

## 🔧 Git Setup & Branching Strategy

### Initial Setup (One-time)

```bash
# 1. Initialize Git repository (if not already done)
git init

# 2. Add your GitHub repository as remote
git remote add origin https://github.com/your-username/bumara.git

# 3. Create and switch to main branch
git checkout -b main
git push -u origin main

# 4. Create development branch
git checkout -b develop
git push -u origin develop

# Install all dependencies
pnpm install

# This installs packages for:
# - Root workspace
# - apps/app (Next.js)
# - packages/backend (Cloudflare Worker)
# - packages/* (shared packages)


# In apps/app
cp apps/app/.env.example apps/app/.env.local

# In packages/backend
cp packages/backend/.env.example packages/backend/.env

#3.2: Get Clerk API Keys

#Go to https://dashboard.clerk.com
#Sign in (or create account)
#Create new application: "Bumara Dev"
#Go to "API Keys" section
#Copy the keys

Add to apps/app/.env.local:
envNEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxx
CLERK_SECRET_KEY=sk_test_xxxxx

NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/dashboard

NEXT_PUBLIC_API_URL=http://localhost:3001
Add to packages/backend/.env:
envCLERK_PUBLISHABLE_KEY=pk_test_xxxxx
CLERK_SECRET_KEY=sk_test_xxxxx

#If using Neon, Supabase, or Railway:

#Create database
#Copy connection string
#Add to packages/backend/.env:

#env
#DATABASE_URL=[your-connection-string]

3.4: Other Environment Variables
Add to packages/backend/.env:
envNODE_ENV=development
PORT=3001
API_URL=http://localhost:3001

# Add any other vars your app needs
```