# Bumara Help Centre

Source for the Bumara help centre, published with [Mintlify](https://mintlify.com) at
**https://docs.bumara.com**.

Bumara helps Zambian businesses stay current with PACRA, ZRA, NAPSA and NHIMA while running the
day-to-day work that creates the filings — invoicing, payroll, inventory and documents.

## Local preview

```bash
npm i -g mint
mint dev
```

The preview runs at `http://localhost:3000`. Run `mint dev` from the repository root, where
`docs.json` lives.

To check for broken internal links before pushing:

```bash
mint broken-links
```

If the preview behaves oddly after an upgrade, run `mint update` to pull the latest CLI.

## Publishing

Pushing to `main` deploys automatically via the Mintlify GitHub app. Open a pull request for
anything you want reviewed first — Mintlify posts a preview link on the PR.

## Repository layout

| Path | Contents |
| --- | --- |
| `docs.json` | Site configuration: theme, colours, navigation, footer, SEO |
| `index.mdx` | Help centre landing page |
| `start/` | Getting started — account, business setup, orientation |
| `compliance/` | Obligations, tasks, evidence, submissions, fees |
| `regulators/` | PACRA, ZRA, NAPSA and NHIMA guides |
| `invoicing/` | Quotes, invoices, payments, Smart Invoice, reports |
| `payroll/` | Employees, pay runs, payslips, statutory returns |
| `inventory/` | Items, stock movements, point of sale, insights |
| `account/` | Organisation settings, team and roles, billing, security |
| `help/` | Glossary, statuses, troubleshooting, FAQ |
| `logo/`, `favicon.svg` | Brand assets |

## Writing conventions

- Every page needs `title` and `description` frontmatter — the description is what search engines
  and the site search show.
- Every new page must be added to `navigation` in `docs.json`. Pages that are not listed are not
  reachable and are excluded from indexing.
- Use British spelling ("organisation", "authorised"), matching the product interface.
- Link between pages with root-relative paths and no file extension: `/help/glossary`.
- Rates, thresholds and fees change. Where a page quotes one, keep the standing caveat that the
  regulator's current published position governs.
