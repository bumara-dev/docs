---
title: "Bumara ERP - Product Requirements Document"
description: "Product requirements for Bumara, a compliance-first business operating system for Zambian and wider African SMEs."
---

**Version:** 1.3  
**Status:** Draft  
**Last Updated:** October 2, 2025  
**Author:** Nsangu Phiri  
**Reviewers:** _Pending review_

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Sept 30, 2025 | Nsangu Phiri | Initial draft |

---

**Currency Note:** This document shows pricing in both Zambian Kwacha (ZMK/K) and US Dollars (USD/$) for reference. Exchange rate used: ~25:1 (approximate, for planning purposes only). Actual billing will be in customer's local currency. Zambia-based customers pay in ZMK via mobile money, bank transfer, or card. International customers pay in USD or local currency equivalent.

---

## 1. Executive Summary

### The Problem

Every month, SME owners across Zambia and wider Africa lose **8–15 hours** juggling compliance across ZRA, PACRA, NAPSA, and NHIMA—jumping between Excel spreadsheets, email threads, and WhatsApp groups just to piece together what they owe and when it's due.

**The ZRA Smart Invoice crisis:** Since October 1, 2024, all VAT-registered businesses must issue invoices through ZRA's Smart Invoice system or face penalties. Most SMEs are scrambling to integrate separate Smart Invoice software with their existing accounting tools—double data entry, manual uploads, disconnected systems.

**The measurable cost:** ZMK 2,500–15,000 annually in avoidable penalties, 15–25% of revenue trapped in uncollected invoices, and 3–7 days to compile financial statements when bank loans or business opportunities arise. Plus the hidden cost of maintaining two separate systems: your accounting tool AND Smart Invoice.

**The invisible cost:** Business owners stuck in compliance admin instead of growing revenue, strategic blind spots from fragmented data across siloed tools (inventory in one spreadsheet, invoices in another, payments tracked manually), and zero real-time visibility into the one number that matters most—**how much cash do I actually have?**

### The Solution

**Bumara** is a **compliance-first, mobile-ready business system** for African SMEs that unifies obligations tracking, submissions, invoicing (with **ZRA Smart Invoice built-in**), payments, and core accounting in one platform. It provides regulator-aware workflows (ZRA, PACRA, NAPSA, NHIMA, any more), proactive deadline alerts, automatic evidence trails, and real-time cashflow visibility—with modular plans so businesses start small and scale without re-implementation.

**No separate Smart Invoice software needed.** Every invoice issued in Bumara is automatically ZRA-compliant, transmitted in real-time via VSDC integration, and validated with Mark ID and QR code.

### Target Users

* **Primary:** **Chandamali** — Hands-on Business Owner growing a fast-moving services/tech outsourcing company (5-50 employees)
* **Secondary:** **Mutinta** — Bookkeeper/Assistant Accountant who posts invoices/receipts, reconciles payments, and prepares PAYE/NAPSA/NHIMA submissions
* **Tertiary:** **Lindiwe** — Compliance consultant who manages other businesses compliance needs and statutory requirements

### Key Differentiators

1. **Compliance-first data model:** Obligations, submissions, acknowledgements, and evidence are first-class objects (not add-ons) → audit-ready by default, fewer penalties, regulator-native workflows
2. **Mobile-first approvals:** Owners approve invoices, payments, and submissions from their phones in 2–3 taps → faster cash collection and on-time filings
3. **Regulator-native workflows:** Built specifically for ZRA tax codes, PACRA return formats, NAPSA contribution tracking, NHIMA employer reporting → no manual format conversion
4. **Modular adoption:** Start with Compliance + Invoicing (under 2 hours); add Accounting/Inventory/Payroll and CRM as needed → lower time-to-value, no big implementations

### Success Criteria (6 Months Post-MVP)

* **Customer Growth:** ≥75 paying organizations
* **User Satisfaction:** NPS ≥+35, CSAT ≥4.4/5
* **Platform Reliability:** 99.9% uptime
* **Compliance Impact:** On-time submissions ≥98%, compliance admin time reduced by 50%
* **Cash Impact:** DSO reduction ≥20%, invoice-to-payment cycle ≤30 days
* **Engagement:** ≥60% weekly active usage for Compliance module

---

## 2. Product Vision

### Vision Statement

> **To be the default business system for African SMEs—where compliance runs itself, cashflow is always visible, and business owners spend time growing revenue instead of chasing paperwork.**

### 3-Year Vision (by 2028)

* **Market leadership** in Zambia for SME compliance tooling; expanded to 3–4 SADC markets (Zimbabwe, Tanzania, Botswana, Namibia)
* **Regulator packs as a service:** Pre-configured obligation schedules, evidence checklists, and submission formats for each country—updated centrally when regulations change
* **AI-powered compliance:** Draft submissions automatically, flag anomalies before filing, predict cashflow gaps, and shorten month-end close by 50%
* **Partner ecosystem:** Network of accountants and compliance consultants using Bumara as their standard platform with white-label options

### Product Principles

1. **Compliance before convenience**  
   * **What it means:** Auditability cannot be traded for speed or simplicity
   * **In practice:** Immutable audit log for every transaction; evidence required before statutory submissions can be marked complete

2. **Mobile-first, desktop-strong**  
   * **What it means:** Core workflows must be thumb-friendly; owners approve on the go
   * **In practice:** Invoice approvals, payment confirmations, and deadline reminders optimized for &lt;3 taps on mobile

3. **Opinionated defaults, open edges**  
   * **What it means:** Useful out-of-the-box for 80% use cases; configurable for the other 20%
   * **In practice:** Template Chart of Accounts, pre-loaded regulator packs; allow account customization and custom obligation schedules

4. **Local fidelity beats generic breadth**  
   * **What it means:** Deep Zambia regulatory fit before expanding to multi-country breadth
   * **In practice:** ZRA/PACRA/NAPSA/NHIMA calendars, filing statuses, receipt formats, tax codes; expansion only when local compliance is proven

5. **Measured simplicity**  
   * **What it means:** Every click must earn its keep; complexity needs justification
   * **In practice:** P0 tasks completed in ≤3 taps; task completion time is a tracked KPI

---

## 3. Target Market

### Geographic Focus

**Phase 1 (2025):** Zambia (Lusaka, Copperbelt, Southern Province)  
**Phase 2 (2027–2028):** Zimbabwe, Malawi, Botswana, Namibia  
**Phase 3 (2028+):** Kenya, Tanzania, Ghana, Nigeria

### Industry Focus

**Primary (80% of initial focus):**

1. **Services & Outsourcing** — IT services, field services, domestic care, security, consulting
   * *Why:* Frequent invoicing, recurring statutory filings, minimal inventory complexity
2. **Retail/Distribution** — Small shops, distributors, wholesalers
   * *Why:* High transaction volume, AR collection pressure, inventory-lite needs
3. **Light Manufacturing** — Food processing, packaging, assembly
   * *Why:* Inventory tracking needs, compliance evidence critical, growing cashflow concerns

**Secondary (20%):** Professional services (accounting/legal), clinics/pharmacies, small construction, hospitality

### Company Size

**Sweet Spot:** **5–50 employees**

* **Too small (&lt;5 employees):** Owner-operator with low willingness-to-pay; spreadsheets "good enough"
* **Too large (>250 employees):** Require bespoke processes, heavier integrations, dedicated IT teams; better served by enterprise ERP
* **Just right (5–50):** Pain is acute, compliance burden is real, decisions are fast, budget exists but limited

### Budget Profile

**Current Spend:** K500–K5,000/month ($20–$200/month) on finance/operations tools (often fragmented: QuickBooks + Smart Invoice software + Excel)  
**Willingness to Pay:** K799–K23,999/month ($32–$960/month) tiered by team size, modules, and use case  
**Decision Maker:** Business Owner/MD or Finance Lead  
**Budget Cycle:** Monthly subscriptions; annual discounts (15%) available  
**Payment Methods:** Mobile money (MTN/Airtel), bank transfer, credit/debit card (Stripe)

**Note on Currency:** All pricing is shown in both Zambian Kwacha (ZMK) and US Dollars (USD) at approximate exchange rate of 25:1 for international reference. Actual billing in local currency based on customer location.

### Market Size (Zambia)

* **Total formal SMEs:** ~50,000–80,000
* **Digital-ready segment:** ~15,000–25,000 (have email, use smartphones, basic accounting software or serious spreadsheets)
* **Addressable (5-50 employees):** ~8,000–12,000
* **Accounting firms/consultants managing SMEs:** ~150–250 firms (potential Partner tier customers)

**Targets:**

* **Year 1:** 350 customers (~3-4% of addressable)
  * 250 direct SME customers (Solo/Team/Scale tiers)
  * 5-10 Partner firms managing ~100 additional SME orgs
* **Year 3:** 2,500+ customers (~20-30% of addressable)
  * 1,500 direct SME customers
  * 50-75 Partner firms managing ~1,000 SME orgs

**Partner Channel Multiplier:**

* Each Partner firm manages average 15 client orgs
* 10 Partner firms = 150 SME orgs reached
* Partner channel can 2-3× market penetration vs direct sales alone

---

## 4. Problem Statement

### Current State: How SMEs Manage Compliance Today

**Method 1: Excel/Google Sheets (~60% of SMEs)**

* Manual templates downloaded from regulator websites; version drift across team members
* No automatic reminders—rely on personal calendars or memory
* Formula errors cause incorrect calculations; no validation
* **Why they stick:** "Free," familiar, feels flexible

**Method 2: Paper Files + WhatsApp (~20%)**

* Physical filing cabinets for statutory documents; WhatsApp groups for deadline reminders
* Lost documents during audits; zero searchability
* Approvals via screenshot exchanges; no audit trail
* **Why they stick:** Habit, low tech literacy, "it's worked for 10 years"

**Method 3: Generic Accounting Software (~20%)**

* QuickBooks, Sage, Zoho Books—good at books, weak at regulator workflows
* Still need manual spreadsheet for NAPSA/NHIMA tracking
* Require expensive accountant add-ons for ZRA submissions
* **Why they stick:** Brand recognition, accountant recommendations

### The Pain Points (Ranked by Impact)

#### P0: ZRA Smart Invoice Compliance Chaos

**The Pain:**

* Since October 1, 2024, all VAT invoices MUST be issued through ZRA Smart Invoice or face penalties
* Most SMEs running separate Smart Invoice software alongside their accounting tools
* Double data entry: Create invoice in accounting tool → Re-enter in Smart Invoice → Pray they match
* Manual VSDC uploads when integration fails
* "Invoice not validated" errors killing cashflow (customers can't claim input VAT)

**Impact:**

* **2–4 hours/week** on duplicate invoice entry and reconciliation
* **ZMK 500–2,000/incident** in penalties for non-compliant invoices
* Lost sales when customers reject invoices without Mark ID/QR code
* Cashflow delays from invoice validation failures

**Frequency:** Every single invoice (daily for most businesses)

**Current Workaround:**

* Pay for Smart Invoice software separately ($30–$80/month)
* Manually export invoices from accounting tool → Import to Smart Invoice
* Hire someone just to manage Smart Invoice compliance

**Customer Quote:** *"I'm paying for TWO systems that should be ONE. My bookkeeper spends half her time just making sure the Smart Invoice numbers match our accounting books."*

---

#### P1: Invisible Compliance Until Penalties Hit

**The Pain:**

* Missed PAYE/NAPSA/NHIMA deadlines because reminder was not delivered
* Discovered non-compliance only when ZRA sends penalty notice or you want to obtain a tax clearance certificate
* Scrambling to compile submission at 11:59pm on deadline day and it too late to tender in a bid on ZPPA

**Impact:**

* **ZMK 2,500–15,000 annually** in avoidable penalties
* **4–6 hours** of emergency prep time per missed deadline
* Stress, sleepless nights, damage to business reputation and lost opportunities

**Frequency:** Monthly or quarterly (every regulator has different cycles)

**Current Workaround:** Calendar reminders that get snoozed; last-minute scrambles; paying penalties as "cost of doing business"

**Customer Quote:** *"The regulator reminds me with a penalty. That's when I know I missed something."*

---

#### P2: Disconnected Cash Lifecycle

**The Pain:**

* Invoices sent via email/WhatsApp PDF—no tracking of who opened, who paid
* Receipts scattered across email, SMS, bank app notifications
* No systematic follow-up on overdue invoices
* DSO creeps from 30 days to 60+ days

**Impact:**

* **15–25% of revenue trapped** in uncollected invoices at any given time
* **5–8 hours/week** for owners chasing payments manually
* Cashflow gaps leading to delayed supplier payments or loan reliance

**Frequency:** Continuous (every invoice issued)

**Current Workaround:** WhatsApp payment reminders; manual aging in spreadsheet; desperate phone calls

**Customer Quote:** *"I spend more time chasing money than making money."*

---

#### P3: Evidence Scattered Across Channels

**The Pain:**

* Invoices in Gmail, receipts in WhatsApp, bank statements in PDFs, contracts in Google Drive
* During audits or bank loan applications, takes **3–7 days** to compile complete documentation
* Version control nightmare—which is the final signed contract?

**Impact:**

* **Extra 2–3 days at month-end** reconciling and searching for documents
* Failed loan applications due to "incomplete documentation"
* Audit rework and potential penalties for insufficient evidence

**Frequency:** Monthly (month-end close) and quarterly (audits/loan apps)

**Current Workaround:** Shared drive folders with poor naming conventions; email search; prayer

**Customer Quote:** *"I know I have the receipt somewhere... just give me a few hours to find it."*

---

#### P4: Manual Reconciliations Are Error-Prone

**The Pain:**

* Payments arrive via multiple channels: bank transfers, MTN Mobile Money, Airtel Money, cash
* Each requires separate CSV export, manual matching to invoices
* Unmatched payments sit in "suspense" for weeks
* Posting delays cascade into inaccurate reports

**Impact:**

* **3–5 hours/week** reconciling payments
* Inaccurate cashflow visibility—never sure what's actually cleared
* Month-end adjustments delay financial close by 2–4 days

**Frequency:** Weekly or bi-weekly

**Current Workaround:** CSV copy/paste in Excel; manual detective work matching names

---

#### P5: High Onboarding Friction Kills Adoption

**The Pain:**

* New accounting software requires **8–20 hours** of setup: chart of accounts, opening balances, historical data
* Training bookkeeper takes another **4–8 hours**
* First value not seen until week 3 or 4—frustration builds, motivation drops

**Impact:**

* **High early churn** (30-40% abandon in first 30 days)
* Delayed time-to-value means delayed ROI realization
* Wasted sales effort if customer never goes live

**Frequency:** One-time (onboarding) but critical to retention

**Current Workaround:** Ad-hoc handholding by support team; slow rollout that never completes

---

### The Quantified Opportunity

**What SMEs Gain with Bumara:**

* ✅ **40–60% faster compliance task completion** (from 15 hrs/month to 6–8 hrs/month)
* ✅ **20–35% DSO reduction** via automated reminders and payment tracking (from 60 days to 40–50 days)
* ✅ **&lt;2 hours to first invoice** and regulator setup (vs 8–20 hours with traditional systems)
* ✅ **100% audit trail** on submissions—always audit-ready
* ✅ **3–7 days saved** during month-end close, audits, and loan applications

**ROI Example (Annual - Using Bumara Team Plan):**

| **Category** | **Savings (ZMK)** | **Savings (USD)** | **Calculation** |
|-------------|-------------------|-------------------|-----------------|
| Owner time saved | K27,000 | $1,080 | 9 hrs/month × 12 months × K250/hr |
| Bookkeeper time saved | K12,000 | $480 | 5 hrs/month × 12 months × K200/hr |
| **Separate Smart Invoice software eliminated** | **K9,720–K25,920** | **$389–$1,037** | **K810–K2,160/month × 12 months** |
| **Double data entry eliminated** | **K12,480** | **$499** | **2 hrs/week × 52 weeks × K120/hr** |
| Penalties avoided | K10,000 | $400 | Avg 2-3 missed deadlines × K2,500–5,000 |
| Faster collections | K15,000 | $600 | 20% DSO reduction on K750k revenue × 5% cost |
| **Total Annual Benefit** | **K86,200–K102,400** | **$3,448–$4,096** | |
| **Bumara Team Annual Cost** | K29,988 | $1,200 | K2,499/month × 12 months |
| **Net Savings** | **K56,212–K72,412** | **$2,248–$2,896** | |
| **ROI** | **2.9–3.4×** | **2.9–3.4×** | |

---

## 5. Solution Overview

### What is Bumara ERP?

**Elevator Pitch:**
Bumara is a compliance-first, mobile-ready business operating system for African SMEs that unifies obligations, documents, invoicing (**with ZRA Smart Invoice built-in**), payroll, CRM and core accounting—so owners see **cash due** and **filings due** on one screen and approve in 2–3 taps from their phone. **No separate Smart Invoice software. No double data entry. Just one system that does it all.**

**Positioning Statement:**
For SMEs across Zambia and Africa who waste time juggling compliance across disconnected tools (and now need separate Smart Invoice software), Bumara is a unified business platform that makes ZRA Smart Invoice, PACRA, NAPSA, and NHIMA obligations visible, trackable, and automated—unlike generic accounting software, payroll systems that requires manual regulator workflows, separate Smart Invoice integration, and costly double data entry.

### Core Value Propositions

**For Chandamali (Business Owner):**  
*"See everything that matters on one screen—cash due from customers and filings due to regulators. Every invoice is automatically ZRA Smart Invoice compliant (no separate software needed). Approve invoices and submissions from your phone in 3 taps. Spend 50% less time on admin."*

**For Mutinta (Bookkeeper):**  
*"Create invoices that are automatically transmitted to ZRA Smart Invoice—no double entry, no separate software, no validation errors. Post transactions with attached proof, reconcile payments quickly, submit to regulators on time with pre-filled forms and evidence checklists. Close the month in half the time."*

### How It Works

#### Owner Flow (Primary Persona)

1. **Sign up** → Create organization → Select regulators (ZRA, PACRA, NAPSA, NHIMA)
2. **Invite team** → Add bookkeeper; select subscription plan (Solo/Team/Scale)
3. **Issue first invoice** → Send to customer; enable payment reminders
4. **Review obligations** → See upcoming ZRA/NAPSA/NHIMA deadlines on dashboard
5. **Approve & monitor** → Approve submissions from mobile; track cash due vs filings due

**Time to first value: &lt;2 hours**

---

#### Bookkeeper Flow (Secondary Persona)

1. **Configure accounts** → Load Chart of Accounts template; enter opening balances
2. **Post daily transactions** → Create invoices with automatic ZRA Smart Invoice transmission, tax calculations, and evidence requirements
3. **Verify Smart Invoice compliance** → Check that all invoices show validated status (Mark ID assigned, QR code generated)
4. **Reconcile payments** → Match bank/mobile money receipts to invoices; update aging automatically
5. **Prepare submissions** → Compile PAYE/NAPSA/NHIMA pack with evidence checklist; submit for approval
6. **Record acknowledgements** → Upload ZRA/PACRA receipt; mark obligation complete

**Compliance confidence: 100% audit trail + 100% ZRA Smart Invoice compliance**

---

### Key Features Overview

#### Phase 1 (Oct-Dec 2025): Compliance, Invoicing, Accounting Core, Documents

**Compliance Module:**

* Pre-loaded regulator packs (ZRA, PACRA, NAPSA, NHIMA) with obligation schedules
* Automatic reminders (7 days, 3 days, 1 day before due date)
* Submission records with required evidence attachments
* Status tracking (Upcoming → Due Soon → Submitted → Acknowledged)

**Invoicing & Cash:**

* Create/send invoices with PDF generation; ZRA-compliant tax calculations
* **ZRA Smart Invoice integration:** Real-time VSDC transmission, Mark ID/QR code generation, validation tracking
* Customer directory with payment history
* Payment receipts and matching
* Aging report and overdue reminders

**Documents & Audit:**

* Evidence vault storage with tagging and linking to transactions/obligations
* Expiry tracking for licenses, contracts, insurance
* Immutable audit log for all system changes
* RBAC (Owner, Bookkeeper, Finance Manager roles)

**Platform:**

* Authentication (Clerk) with org multi-tenancy
* Subscription gating by plan tier
* Onboarding checklist with progress tracking
* Usage telemetry for feature adoption

---

#### Phase 2 (Jan-Apr 2026): Core accounting, Inventory-Lite, Payments, CRM Basics

**Accounting Core:**

* Chart of Accounts template (Zambian GAAP-aligned)
* General journal with double-entry validation
* Trial Balance, P&L, Balance Sheet
* Opening balances import (CSV with validation)

**Inventory-Lite:**

* SKU/equipment tracking with simple adjustments
* Low-stock alerts
* Basic inventory reports (stock levels, movement)

**Payments:**

* Payment links (card/mobile money integration)
* Automated payment reminders (dunning sequences)
* Enhanced aging report with customer risk scoring

**CRM Basics:**

* Customer relationship tracking (contacts, interactions, notes)
* Lead management and pipeline view
* Email integration (log customer communications)
* Deal tracking and opportunity management
* Task/follow-up reminders
* Customer segmentation and tags

---

#### Phase 3 (May-Dec 2026): AI Assists, Integrations, Payroll

**AI Features:**

* Draft ZRA/NAPSA submissions (pre-fill from transaction data)
* Anomaly detection (flag unusual patterns before filing)
* Smart reconciliation (auto-match payments to invoices)
* Cashflow predictions and insights

**Integrations:**

* Bank feed APIs (where available in Zambia)
* Accounting firm portal (white-label access)
* Third-party marketplace launch

**Payroll Module:**

* Employee records and contract management
* Salary calculation with PAYE/NAPSA/NHIMA deductions
* Payslip generation and distribution
* Statutory filing prep (PAYE schedule, NAPSA register, NHIMA report)
* Leave management and tracking
* Timesheet integration for hourly workers
* Payroll journal auto-posting to accounting

---

## 6. Product Scope

### What's IN Scope (v1.0)

#### 6.1 Compliance ✅

* ✅ Regulator packs (ZRA, PACRA, NAPSA, NHIMA) with pre-loaded obligations
* ✅ Obligation CRUD with cycles, due dates, and status workflow
* ✅ Submission records with evidence attachments and acknowledgement uploads
* ✅ Automated reminders (email + in-app notifications)
* ✅ Compliance dashboard showing upcoming, overdue, and completed obligations

#### 6.2 Invoicing & Cash ✅

* ✅ **ZRA Smart Invoice integration** (VSDC module, real-time transmission, Mark ID/QR code, validation tracking)
* ✅ Create/send invoices with PDF generation and ZRA Smart Invoice compliance
* ✅ Receipt recording with payment method tracking
* ✅ Customer directory with TPIN and payment history
* ✅ Due date reminders for unpaid invoices
* ✅ Simple aging view (0-30, 31-60, 61-90, 90+ days)

#### 6.3 Documents & Audit ✅

* ✅ Evidence vault (upload, tag, link to obligations/transactions)
* ✅ Document expiry tracking with alerts for key documents (licenses, insurance, contracts)
* ✅ Organization activity audit log (immutable, searchable)
* ✅ RBAC (Owner, Bookkeeper, Finance Manager with module-level permissions)

#### 6.4 Accounting Core ✅

* ✅ Chart of Accounts template (Zambian-standard, editable)
* ✅ General journal with double-entry validation
* ✅ Accounting periods with opening/closing
* ✅ Trial Balance, Profit & Loss, Balance Sheet reports
* ✅ Opening balances import (CSV with preview and validation)

#### 6.5 Platform ✅

* ✅ Authentication (Clerk OAuth/JWT) with organization multi-tenancy
* ✅ Subscription management with plan-based feature gating
* ✅ Onboarding checklist with completion tracking
* ✅ Usage telemetry (task completion times, feature adoption)

---

### What's OUT of Scope (v1.0)

❌ **Bank feed integrations** — API availability limited in Zambia; manual CSV import sufficient for MVP  
❌ **Full payroll module** — Complex statutory requirements; comprehensive payroll deferred to Phase 3  
❌ **CRM (Customer Relationship Management)** — Lead tracking, pipeline, full sales automation deferred to Phase 2; basic customer directory included in v1.0  
❌ **Advanced inventory** (multi-warehouse, serial tracking, batch management) — Inventory-lite in Phase 2 sufficient  
❌ **CRM automation** (lead scoring, email sequences, advanced workflows) — Phase 2 CRM basics, Phase 3 automation  
❌ **Advanced budgeting/forecasting** — Core reporting sufficient for initial value  
❌ **Third-party marketplace** — API ecosystem in Phase 3  
❌ **Multi-currency** — ZMK-first; multi-currency in regional expansion (Phase 2)  
❌ **Project management** — Task/project tracking deferred to Phase 3  
❌ **Time tracking** — Integrated with Payroll in Phase 3

**Rationale:** Focus Product on the two highest-ROI areas—compliance automation and cash collection—to prove core value proposition quickly. CRM, Payroll, and advanced inventory are valuable but not critical for initial go-to-market. These modules add 6-12 months to development timeline if included in initial Product development, delaying market entry and revenue. Better to launch fast, validate, iterate.

---

### Feature Prioritization Framework

**Priority Score = (User Impact × Strategic Value) / (Effort × Risk)**

**P0 (Must-Have for Initial Product development):**

* Compliance core (obligations, reminders, submissions)
* Invoicing (create, send, track)
* Evidence vault & audit log
* Subscription gating
* Journal & reports (CoA, TB, P&L, BS)

**P1 (Strong Nice-to-Have for MVP):**

* Payment receipt recording
* Aging report
* Customer directory
* Document expiry tracking

**P2 (Defer to Phase 2):**

* Inventory-lite
* Payment links (card/mobile money)
* Automated dunning

**P3 (Defer to Phase 3):**

* AI-assisted submissions
* Bank feeds
* Payroll basics
* API for integrations

---

## 7. Feature Requirements (Detailed)

### 7.1 Obligations & Submissions

**User Story:**  
As **Mutinta (Bookkeeper)**, I want preset obligations with reminders and evidence checklists so I never miss a filing deadline and always know what documents I need to attach.

**Business Value:**  

* Reduces penalties (direct cost savings)
* Core product differentiator (regulator-native workflows)
* Builds customer trust and long-term retention

**Must Have:**

* Obligation templates (pre-loaded for ZRA/PACRA/NAPSA/NHIMA)
* Automatic reminders (7, 3, 1 day before due date)
* Status workflow (Upcoming → Due Soon → Submitted → Acknowledged → Overdue)
* Submission records with evidence attachments
* Acknowledgement upload (regulator receipt/confirmation)
* Immutable audit trail

**Should Have:**

* Notes field for each obligation (bookkeeper can add context)
* Per-regulator evidence checklist (e.g., NAPSA requires payroll register + proof of payment)
* Export submission pack (PDF/ZIP for offline storage)

**Could Have:**

* Multi-step approval workflow (bookkeeper prepares → manager reviews → owner approves)
* Delegation (assign obligations to specific team members)

**Won't Have (v1):**

* Automated e-filing to regulator portals (requires API access we don't have)

**UI/UX Notes:**

* Kanban board OR list view sorted by due date (user toggle)
* Color-coded status (green = on track, yellow = due soon, red = overdue)
* "Due Soon" ribbon on dashboard to catch attention
* Mobile: swipe actions for quick status updates

**Technical Requirements:**

* Tables: `regulator`, `obligation`, `submission`, `evidence`, `acknowledgement`
* Scheduled jobs for reminder emails/notifications (runs daily at 6am local)
* File storage for evidence (S3-compatible or Cloudflare R2 with signed URLs)
* Audit log trigger on every obligation/submission update

**API Endpoints:**

* `GET /api/obligations` (list with filters)
* `POST /api/obligations` (create custom obligation)
* `PATCH /api/obligations/:id/status` (update status)
* `POST /api/submissions` (record submission with evidence)
* `GET /api/submissions/:id` (retrieve with evidence links)

**Acceptance Criteria:**

* [ ] Reminder fires on schedule (7, 3, 1 day before)
* [ ] Submission cannot be marked complete without required evidence attached
* [ ] Status changes logged in audit trail with timestamp + user
* [ ] Acknowledgement upload updates status to "Acknowledged"
* [ ] Overdue obligations appear in red on dashboard

**Error Handling:**

* Regulator portal down → surface message "Submission recorded; verify directly on [regulator] portal"
* File upload failure → resumable upload (chunked uploads for large files)
* Missing evidence → block submission with clear error: "NAPSA requires payroll register. Please attach."

---

---

### 7.2 ZRA Smart Invoice Integration

**User Story:**  
As **Mutinta (Bookkeeper)**, I want every invoice I create to automatically be ZRA Smart Invoice compliant—transmitted to ZRA in real-time, validated, and stamped with Mark ID and QR code—so I don't have to maintain two separate systems or risk penalties for non-compliant invoices.

**Business Value:**

* **Mandatory compliance** (legally required since Oct 1, 2024 for all VAT-registered businesses)
* **Eliminates duplicate systems** (no separate Smart Invoice software needed = $30-$80/month savings)
* **Eliminates double data entry** (2-4 hours/week saved)
* **Major competitive differentiator** (most ERPs don't have this built-in yet)
* **Reduces invoice rejection** (customers need Mark ID/QR to claim input VAT)

**Must Have:**

* VSDC integration (Virtual Sales Data Controller module)
* Real-time invoice transmission to ZRA Smart Invoice server
* Automatic Mark ID assignment from ZRA
* QR code generation and display on invoice PDF
* Invoice validation status tracking (Pending → Transmitted → Validated → Failed)
* Failed invoice alerts with error details
* Retry mechanism for transmission failures
* Smart Invoice receipt/acknowledgement storage
* Compliance dashboard showing validated vs pending invoices

**Should Have:**

* Bulk invoice transmission (for historical invoices)
* Offline mode with queued transmission (when internet down)
* Smart Invoice status visible on invoice list (green check = validated)
* Export Smart Invoice data for ZRA audits
* Device management (register/unregister VSDC devices)

**Could Have:**

* Predict invoice validation failures before transmission (pre-validation)
* Auto-correction suggestions (e.g., "Tax rate mismatch detected")
* Integration with ZRA's sandbox environment for testing

**Won't Have (v1):**

* Support for non-VAT invoices through Smart Invoice
* Integration with legacy EFD devices (deprecated by ZRA)

**UI/UX Notes:**

* Invoice creation screen shows "ZRA Smart Invoice" badge
* Real-time validation status indicator (green = validated, yellow = pending, red = failed)
* Failed invoices show actionable error message ("Invalid TPN. Please verify customer TPIN.")
* QR code prominently displayed on PDF (bottom right corner per ZRA spec)
* Mark ID displayed below QR code in format: MARK-YYYYMMDD-XXXXXX

**Technical Requirements:**

* VSDC module installation and configuration
* Tables: `smart_invoice_transmission`, `validation_status`, `mark_id`, `device_registration`
* Job queue for transmission (handle network failures gracefully)
* Webhook listener for ZRA validation responses
* Encryption for VSDC communication (TLS 1.2+)
* Rate limiting awareness (ZRA may throttle during peak hours)

**ZRA Smart Invoice Requirements:**

* Register organization with ZRA Smart Invoice portal
* Download and configure VSDC module
* Obtain VSDC credentials (taxpayer code, device ID, API key)
* Use ZRA-approved invoice format (line items, taxes, totals structure)
* Include required fields: TPIN (seller + buyer), invoice date, due date, item descriptions, quantities, unit prices, tax rates, totals
* Generate QR code with ZRA-specified data format
* Transmit within 24 hours of invoice creation (ZRA requirement)

**API Endpoints:**

* `POST /api/smart-invoice/transmit/:invoice_id` (send invoice to VSDC)
* `GET /api/smart-invoice/status/:invoice_id` (check validation status)
* `POST /api/smart-invoice/retry/:invoice_id` (retry failed transmission)
* `GET /api/smart-invoice/dashboard` (compliance overview)
* `POST /api/smart-invoice/device/register` (register VSDC device)

**Acceptance Criteria:**

* [ ] Invoice transmitted to ZRA within 30 seconds of creation
* [ ] Mark ID returned and stored within 60 seconds
* [ ] QR code generated and embedded in PDF
* [ ] Failed transmissions retry automatically (3 attempts, 5 min intervals)
* [ ] Validation status visible in real-time on dashboard
* [ ] Invoices without validation cannot be marked as "sent to customer" (block until validated)
* [ ] 99%+ transmission success rate (p99 &lt;90 seconds to validation)

**Error Handling:**

* **Network failure** → Queue transmission, retry when connection restored, notify user "Invoice queued for ZRA validation"
* **Invalid TPIN** → Block transmission, show error: "Customer TPIN [123456789] not found in ZRA system. Please verify."
* **Tax calculation mismatch** → Show error: "ZRA expects ZMK 160 VAT, invoice shows ZMK 150. Please correct."
* **VSDC not configured** → Onboarding prompt: "Set up ZRA Smart Invoice to issue compliant invoices. [Setup Guide]"
* **Mark ID not received** → Retry + escalation: "Invoice transmitted but validation pending. We'll notify you when Mark ID is assigned."
* **ZRA system downtime** → Status message: "ZRA Smart Invoice system is currently unavailable. Invoice saved; will transmit when system is back online."

**Compliance Notes:**

* **Mandatory as of October 1, 2024** for all VAT-registered taxpayers in Zambia
* Non-compliant invoices cannot be used to claim input VAT (per ZRA regulations)
* Penalties for non-compliance: ZMK 3,000 per offense + potential VAT disallowance
* Only ZRA-validated invoices are legally recognized for tax purposes
* Customers (buyers) can verify invoice authenticity by scanning QR code on ZRA portal

**Migration Path:**

* For businesses with existing invoices pre-Smart Invoice: Bulk upload historical invoices to Smart Invoice system (one-time migration)
* VSDC setup wizard guides user through ZRA portal registration, VSDC download, credential entry
* Test mode: Create sample invoice → Transmit to ZRA sandbox → Validate setup before going live

---

### 7.3 Invoices & Receipts (Standard)

**User Story:**  
As **Chandamali (Business Owner)**, I want to create and send professional invoices quickly, see who owes me money, and get automatic reminders sent to customers—so I reduce my DSO and spend less time chasing payments.

**Business Value:**

* Faster cash collection = improved cashflow (measurable DSO reduction)
* Time savings for owner (8 hrs/month → 3 hrs/month)
* Professional appearance increases payment likelihood

**Must Have:**

* Invoice creation (line items, quantities, unit prices, taxes)
* PDF generation (ZRA-compliant format with tax breakdown + Smart Invoice Mark ID/QR code)
* Send via email/WhatsApp link
* Payment receipt recording (match to invoice)
* Mark invoice as paid (auto or manual)
* Customer directory (name, contact, payment terms, TPIN for Smart Invoice)
* Due date reminders (7, 3, 1 day overdue)

**Should Have:**

* Shareable payment link (future: integrate card/mobile money)
* Quick notes on invoice ("Thank you for your business")
* Duplicate invoice (copy existing for repeat customers)

**Could Have:**

* Recurring invoices (monthly subscriptions)
* Partial payment tracking
* Customer payment history scorecard

**Won't Have (v1):**

* Online payment processing (defer to Phase 2)
* Quote/estimate workflow (invoices only)

**UI/UX Notes:**

* One-screen invoice creation (no multi-step wizard)
* Quick "Add Customer" inline (no need to leave invoice screen)
* Customer TPIN field required for VAT invoices (Smart Invoice compliance)
* Totals calculate automatically (subtotal + tax = total)
* PDF preview before sending (shows Mark ID and QR code after ZRA validation)
* Mobile: simple list view with "Paid" vs "Unpaid" toggle

**Technical Requirements:**

* Tables: `invoice`, `invoice_item`, `customer`, `payment`, `tax_rate`
* PDF generation library (react-pdf or similar)
* Email service integration (SendGrid/Postmark)
* Tax calculation engine (ZRA standard rates: 16% VAT)
* Integration with Smart Invoice transmission (Section 7.2)

**API Endpoints:**

* `GET /api/invoices` (list with filters)
* `POST /api/invoices` (create → triggers Smart Invoice transmission)
* `PATCH /api/invoices/:id` (update, mark paid)
* `POST /api/invoices/:id/send` (email PDF)
* `GET /api/invoices/:id/pdf` (download with Mark ID/QR code)

**Acceptance Criteria:**

* [ ] Invoice PDF renders with all fields (logo, customer info, line items, totals, taxes, Mark ID, QR code)
* [ ] Totals validate: sum(line items) + tax = invoice total
* [ ] Email reminder scheduled when invoice becomes overdue
* [ ] Payment receipt links to invoice and updates status to "Paid"
* [ ] Customer list shows total outstanding balance
* [ ] ZRA Smart Invoice validation completes before invoice can be sent to customer

**Error Handling:**

* Invalid email → clear error: "Please enter a valid email address"
* Tax rate missing → use default 16% VAT with warning
* Email send failure → retry queue (3 attempts, 5 min apart)
* Customer TPIN missing → block transmission: "Customer TPIN required for VAT invoices (ZRA Smart Invoice)"

---


**User Story:**  
As **Lindiwe (Finance Manager)**, I want validated journals and accurate financial reports (TB, P&L, BS) so I can close the month confidently and provide compliant statements to banks and regulators.

**Business Value:**

* Faster month-end close (8–12 days → 5 days)
* Audit-ready reports (reduces external accountant fees)
* Financial visibility for strategic decisions

**Must Have:**

* Chart of Accounts (template with Zambian standard accounts, editable)
* General journal (manual entry with double-entry validation)
* Accounting periods (monthly/quarterly close)
* Trial Balance (always balanced; real-time)
* Profit & Loss Statement (by period)
* Balance Sheet (as of date)
* Opening balances import (CSV with preview and rollback)

**Should Have:**

* Account search/filter
* Journal entry templates (recurring entries)
* Period lock (prevent changes to closed periods)
* Export reports (PDF/Excel)

**Could Have:**

* Custom report builder
* Budget vs actuals
* Multi-year comparisons

**Won't Have (v1):**

* Cost center tracking (defer to Phase 3)
* Fixed asset register (defer to Phase 3)
* Consolidated reporting (single entity only in v1)

**Technical Requirements:**

* Tables: `account`, `journal_entry`, `journal_line`, `period`
* Double-entry validation: sum(debits) = sum(credits) enforced at DB level
* Indexed queries for report performance (&lt;1.5s on 50k transactions)

**API Endpoints:**

* `GET /api/accounts` (Chart of Accounts)
* `POST /api/journal-entries` (create with lines)
* `GET /api/reports/trial-balance`
* `GET /api/reports/profit-loss?period=2025-09`
* `GET /api/reports/balance-sheet?as_of=2025-09-30`

**Acceptance Criteria:**

* [ ] Journal entry blocked if debits ≠ credits (error message: "Debits must equal credits")
* [ ] Trial Balance always sums to zero (debits = credits)
* [ ] Reports render in &lt;1.5s for 50,000 journal lines (p95)
* [ ] Opening balances import with preview shows balance validation before commit
* [ ] Period lock prevents editing journal entries in closed periods

**Error Handling:**

* Unbalanced entry → clear error with diff amount: "Debits exceed credits by ZMK 250"
* Account not found → suggest similar account names
* Period already closed → error: "Cannot edit. Period is closed."

---

### 7.4 Evidence Vault & Audit Log

**User Story:**  
As **Mutinta (Bookkeeper)**, I want every statutory submission and key transaction to have attached supporting evidence—so I'm always audit-ready and can retrieve documents instantly when needed.

**Business Value:**

* Eliminates 3–7 day scramble during audits/loan applications
* Compliance confidence (reduces audit adjustments and penalties)
* Time savings (searchable vault vs folder diving)

**Must Have:**

* Upload documents (PDF, images, spreadsheets)
* Tag and categorize (Invoice, Receipt, Contract, License, etc.)
* Link evidence to obligations and transactions
* Expiry tracking for key documents (licenses, insurance, contracts)
* Expiry alerts (30, 14, 7 days before expiry)
* Immutable audit log (user, action, timestamp, before/after values)
* RBAC (control who can view/edit sensitive documents)

**Should Have:**

* Full-text search in document names and tags
* Bulk upload (drag-drop multiple files)
* Version history (track document replacements)

**Could Have:**

* OCR for scanned receipts (extract date, amount, vendor)
* Auto-categorization based on filename/content

**Won't Have (v1):**

* E-signature workflow (defer to Phase 3)
* Advanced DMS features (check-in/check-out, approval workflows)

**Technical Requirements:**

* Object storage (S3-compatible) with signed URLs (expiring links for security)
* Tables: `document`, `document_tag`, `audit_event`
* Audit log trigger on all critical tables (obligations, submissions, invoices, payments, journal_entries)
* Scheduled job for expiry alerts (runs daily)

**API Endpoints:**

* `POST /api/documents/upload` (multipart form data)
* `GET /api/documents` (list with filters)
* `GET /api/documents/:id/download` (signed URL)
* `POST /api/documents/:id/link` (link to obligation/transaction)
* `GET /api/audit-log` (paginated, filterable)

**Acceptance Criteria:**

* [ ] Document upload succeeds for PDF, JPG, PNG, XLSX (max 10MB per file)
* [ ] Evidence retrievable with full history (who uploaded, when, linked to what)
* [ ] Expiry alert sent 30 days before document expires
* [ ] Audit log immutable (cannot delete or edit entries)
* [ ] RBAC enforced (Bookkeeper cannot access Owner-restricted documents)

**Error Handling:**

* File too large → error: "Max file size is 10MB. Please compress or split."
* Unsupported format → error: "Supported formats: PDF, JPG, PNG, XLSX"
* Storage quota exceeded → upgrade prompt: "Storage full. Upgrade plan or delete old files."

---

### 7.5 Notifications & Reminders

**User Story:**  
As **Chandamali (Owner)**, I want timely reminders for upcoming deadlines (compliance and invoices) delivered to my phone—so I never miss a filing or payment without actively checking the system.

**Must Have:**

* Email notifications (compliance due, invoice overdue, document expiry)
* In-app notifications (bell icon with unread count)
* Mobile push notifications (optional, user can enable/disable)
* Notification preferences (per user: email on/off, push on/off)

**Should Have:**

* WhatsApp notifications (via WhatsApp Business API—high open rates in Zambia)
* Notification digest (daily summary instead of per-event spam)

**Could Have:**

* SMS notifications (backup for users with limited data)
* Slack/Teams integration

**Technical Requirements:**

* Notification queue (Redis/BullMQ)
* Email service (SendGrid/Postmark with templates)
* Push notification service (Firebase Cloud Messaging)
* WhatsApp Business API integration (future)

**Acceptance Criteria:**

* [ ] Compliance reminder sent 7, 3, 1 day before due date
* [ ] Invoice overdue reminder sent 1, 7, 14 days after due date
* [ ] User can mute/unmute notifications in settings
* [ ] In-app notification marked as read when user clicks

---

### 7.7 Onboarding & Setup

**User Story:**  
As **Chandamali (Owner)**, I want to get started quickly—issuing my first invoice and seeing my first obligation reminder within 2 hours—so I achieve value immediately and stay motivated.

**Business Value:**

* Reduces 30-day churn (currently 30-40% with complex tools)
* Faster time-to-value = higher perceived ROI
* Word-of-mouth growth from delighted early users

**Must Have:**

* Guided onboarding checklist (visible progress bar)
* Quick wins prioritized (create invoice before importing historical data)
* Skip options (don't force full accounting setup if user just needs invoicing)
* Sample data pre-loaded (demo invoice, demo obligation)

**Onboarding Steps (Optimized Order):**

1. ✅ Create organization (name, industry, TPIN)
2. ✅ Select regulators (ZRA, PACRA, NAPSA, NHIMA—pre-check common ones)
3. ✅ **Set up ZRA Smart Invoice** (VSDC configuration wizard—5 min)
4. ✅ Create first invoice (quick win, validates Smart Invoice setup)
5. ✅ Invite bookkeeper (optional, can skip)
6. ✅ Import Chart of Accounts (template or custom CSV)
7. ✅ Set opening balances (optional, can defer)
8. ✅ Upload key documents (licenses, contracts—can defer)

**Should Have:**

* Progress tracking (7/7 steps complete → show celebration animation)
* Contextual help tooltips (no separate tutorial videos)
* Video walkthrough (2-min Loom overview—optional, not required)

**Could Have:**

* AI setup assistant (ask 3 questions, auto-configure)
* Pre-filled data from previous system (import from QuickBooks/Excel)

**Technical Requirements:**

* Onboarding state stored per user (`onboarding_progress` table)
* Telemetry: track time spent on each step
* Step completion tracked (skip counts as complete for metrics)

**Acceptance Criteria:**

* [ ] User can issue first invoice within 5 minutes of signup
* [ ] Onboarding completion rate ≥70% (users who complete 5/7 steps)
* [ ] Sample data helps users understand features without needing real data
* [ ] Skip button available on every optional step

---

### 7.8 Subscription & Plan Gating

**User Story:**  
As the **Bumara team**, we want to monetize effectively by gating advanced features behind paid plans—while ensuring the free trial provides enough value to convert users.

**Must Have:**

* Three plan tiers: **Starter** ($39/mo), **Growth** ($79/mo), **Scale** ($149/mo)
* Feature gating by plan (e.g., Starter: 2 users, 50 invoices/month)
* Usage limits enforced (soft cap with upgrade prompt)
* Stripe integration for payment (card, mobile money)
* Trial period (14 days, no credit card required)

**Plan Comparison:**

| **Feature** | **Solo** | **Team** | **Scale** |
|------------|------------|-----------|---------|
| Users | 1 | 5 | Unlimited |
| Invoices/month | 50 | 200 | Unlimited |
| Obligations tracked | 10 | Unlimited | Unlimited |
| Storage | 1 GB | 5 GB | 20 GB |
| Support | Email | Email + Chat | Priority Phone |
| Compliance module | ✅ | ✅ | ✅ |
| Invoicing | ✅ | ✅ | ✅ |
| Accounting Core | ❌ | ✅ | ✅ |
| Inventory-Lite | ❌ | ❌ | ✅ |
| API Access | ❌ | ❌ | ✅ |

**Should Have:**

* Annual billing discount (15% off—$399, $799, $1,499/year)
* Upgrade prompts when limits approached (e.g., "You've used 45/50 invoices this month")
* Downgrade option (with data retention guarantee)

**Technical Requirements:**

* Tables: `subscription`, `plan`, `usage_quota`
* Stripe webhooks for payment events
* Usage tracking middleware (increment counters on create)
* Feature flags by plan (check before rendering UI/API access)

**Acceptance Criteria:**

* [ ] User blocked when limit exceeded with upgrade CTA
* [ ] Stripe payment flow completes successfully
* [ ] Plan change reflects immediately (no manual intervention)
* [ ] Trial users can access all Growth features for 14 days

---

## 8. Non-Functional Requirements

### 8.1 Performance

**Page Load:**

* First Contentful Paint (FCP): &lt;2.0s on 4G connection
* Time to Interactive (TTI): &lt;3.5s
* Dashboard load (with 3 months data): &lt;2.5s

**API Response Times:**

* p50: &lt;150ms
* p95: &lt;400ms
* p99: &lt;800ms

**Report Generation:**

* Trial Balance (10k entries): &lt;800ms
* P&L/Balance Sheet (50k entries): p95 &lt;1.5s

**Database Queries:**

* Simple reads: &lt;50ms
* Complex joins (invoices + items + payments): &lt;200ms
* Monthly aggregations: &lt;500ms

---

### 8.2 Security

**Authentication:**

* Clerk OAuth/JWT with session management
* Optional MFA (TOTP or SMS)
* Password requirements: 12+ chars, mixed case, number, symbol

**Authorization:**

* Role-Based Access Control (RBAC)
* Organization-scoped data (strict tenant isolation)
* Per-module permissions (Owner can see all, Bookkeeper limited)

**Data Protection:**

* At-rest encryption (AES-256 for database, S3 buckets)
* In-transit encryption (TLS 1.2+)
* Signed URLs for file downloads (expire in 1 hour)

**API Security:**

* Rate limiting: 100 req/min per user (burst: 200 req/min)
* Input validation (reject malformed requests)
* SQL injection prevention (parameterized queries)
* CORS whitelist (no `*` wildcard)

**Audit & Compliance:**

* Immutable audit log for all critical actions
* Retention: 7 years (Zambian PACRA requirement)
* GDPR-ready data export/deletion (future for EU expansion)

---

### 8.3 Reliability

**Uptime:**

* Target: 98.9% (60 minutes downtime/month)
* Measured: monthly average excluding scheduled maintenance

**Backups:**

* Database: Daily snapshots (retained 30 days)
* Object storage: Versioning enabled (retain 3 versions)
* Transaction logs: Real-time replication

**Disaster Recovery:**

* Recovery Time Objective (RTO): &lt;4 hours
* Recovery Point Objective (RPO): &lt;24 hours
* Automated failover for database (read replicas)

**Monitoring:**

* Error tracking (Sentry or similar)
* Performance monitoring (APM with p50/p95/p99)
* Job queue health (compliance reminders, email sends)
* Storage usage alerts (notify at 80% capacity)

**Incident Response:**

* On-call rotation for P0 incidents (compliance reminder failures)
* Postmortem for all production outages >30 min

---

### 8.4 Usability

**Learning Curve:**

* Core tasks (create invoice, record submission): No training required
* Advanced tasks (journal entries, period close): &lt;30 minutes onboarding

**Accessibility:**

* WCAG 2.1 AA compliance targets
* Keyboard navigation for all critical flows
* Screen reader support (semantic HTML, ARIA labels)
* Colorblind-friendly (no red/green-only indicators)

**Internationalization:**

* Currency: Zambian Kwacha (ZMK) with proper symbol placement
* Date format: DD/MM/YYYY (Zambian standard)
* Number format: Thousands separator (comma or space)

---

### 8.5 Compatibility

**Browsers (Desktop):**

* Chrome/Edge (last 2 versions)
* Firefox (last 2 versions)
* Safari (last 2 versions)
* No IE11 support

**Mobile Browsers:**

* Chrome (Android)
* Safari (iOS)
* Progressive Web App (PWA) support for offline access

**Screen Sizes:**

* Desktop: Minimum 1280×720
* Tablet: 768×1024
* Mobile: 360×640 (smallest supported)

**Network Conditions:**

* Minimum: 3G connection (500 kbps)
* Optimized for: 4G (5 Mbps)
* Offline support (future): Cache invoices/obligations for view-only

---

### 8.6 Scalability

**v1.0 Targets (Year 1):**

* Organizations: 1,000 active
* Monthly Active Users (MAU): 10,000
* API throughput: 5 req/sec steady state (bursts to 50 req/sec)
* Database: 100k invoices, 500k journal entries

**Growth Plan (Year 2-3):**

* Horizontal scaling: Add app servers behind load balancer
* Database: Read replicas for report queries
* Job queue: Dedicated workers for reminders/emails
* CDN: Static assets + edge caching for API responses

**Monitoring for Scale:**

* Database connection pool usage (alert at 70%)
* API response time degradation (alert if p95 >600ms)
* Job queue backlog (alert if >1,000 pending)

---

## 9. Success Metrics

### 9.1 Product Metrics (6 Months Post-MVP)

**Adoption:**

* **350 active paying organizations** (1.5% of addressable 23k digital-ready Zambian SMEs)
* **1,200 active users** (avg 3.4 users per org)
* **DAU/MAU ≥25%** (daily active users / monthly active users)

**Engagement:**

* Median session duration: **6–9 minutes**
* Sessions per week per user: **≥3**
* Feature adoption: **≥60%** of orgs using Compliance module weekly

**Business:**

* Monthly Recurring Revenue (MRR): **K875,000–K1,034,000 ($35,000–$41,360)** by end of Year 1 (350 orgs with blended rate K2,500–K2,955/month or $100–$118/month)
  * Conservative mix: 50% Solo, 35% Team, 12% Scale, 3% Partner
  * Optimistic mix: 40% Solo, 45% Team, 13% Scale, 2% Partner
* Customer Acquisition Cost (CAC): **≤K3,500 (≤$140)**
* Lifetime Value (LTV): **≥K60,000 (≥$2,400)** - 24 months × K2,500/mo avg
* **LTV:CAC ≥17:1**
* Monthly churn: **&lt;2.5%**
* Annual Recurring Revenue (ARR): **K10.5M–K12.4M ($420k–$496k)** by end of Year 1

**Quality:**

* Customer Satisfaction (CSAT): **≥4.4/5**
* Net Promoter Score (NPS): **≥+35**
* Bug rate: **&lt;0.5 bugs per 1,000 users per month** (production severity P0/P1)

**Performance:**

* Average page load time: **&lt;2.5s**
* API p95 response time: **&lt;400ms**
* Uptime: **≥99.9%**

---

### 9.2 User Success Metrics

#### For Chandamali (Business Owner)

**Time Savings:**

* Admin time on compliance: **−50%** (from 15 hrs/month to &lt;8 hrs/month)
* Time to approve submissions: **&lt;2 minutes** (from 15 minutes in spreadsheet review)

**Cash Impact:**

* Days Sales Outstanding (DSO): **−20%** (from 60 days to 48 days)
* On-time compliance submissions: **≥98%** (from ~80%)
* Penalties avoided: **≥$400/year**

**Confidence Metrics:**

* Can answer "How much cash do I have?" in: **&lt;30 seconds** (vs 3–7 days)
* Real-time visibility into outstanding invoices: **100%**

---

#### For Mutinta (Bookkeeper)

**Efficiency:**

* Same-day transaction posting: **≥90%** (invoice/receipt posted within 24 hours)
* Evidence attached to submissions: **100%** (vs ~60% previously)
* Reconciliation time: **−40%** (from 5 hrs/week to 3 hrs/week)

**Quality:**

* Journal entry errors (unbalanced): **&lt;5%** (validation prevents most)
* Month-end close adjustments: **−30%** (fewer corrections needed)

**Compliance Confidence:**

* On-time submission rate: **≥98%**
* Audit-ready documentation: **100%** (all submissions have evidence)

---

### 9.3 Leading Indicators (Early Signals)

**Week 1:**

* Onboarding completion rate: **≥70%** (users who finish 5/7 setup steps)
* First invoice issued: **Within 24 hours** of signup

**Month 1:**

* Obligations configured per org: **≥2** (at least 2 regulators set up)
* First submission recorded: **≥1** per active org

**Month 3:**

* Organizations with reminders enabled: **≥60%**
* Organizations running monthly reports: **≥50%**
* Weekly Active Users (WAU): **≥40%** of MAU

---

## 10. Roadmap

### Phase 1: Initial Product development – Compliance + Invoicing + Accounting Core

**Timeline:** October–December 2025 (12 weeks)

#### Milestone 1.1: Foundations (Weeks 1–4)

* [ ] **Auth & Organizations**
  * Clerk authentication integration
  * Multi-tenant organization model
  * User roles (Owner, Bookkeeper, Finance Manager)
  * Invitation flow

* [ ] **Data Model & Schema**
  * Regulator packs (ZRA, PACRA, NAPSA, NHIMA)
  * Obligations, submissions, evidence tables
  * Invoices, customers, payments
  * Chart of Accounts, journal entries
  * Audit log structure

* [ ] **Subscription & Gating**
  * Stripe integration (payment, webhooks)
  * Plan tiers (Starter, Growth, Scale)
  * Usage quota enforcement
  * Trial period logic

**Success Criteria:**

* [ ] 10 design partners invited and authenticated
* [ ] Database schema deployed to staging
* [ ] Test payments successful in Stripe test mode

---

#### Milestone 1.2: Compliance & Documents (Weeks 5–8)

* [ ] **Obligations Module**
  * CRUD obligations with due dates, cycles
  * Status workflow (Upcoming → Due Soon → Overdue → Submitted → Acknowledged)
  * Compliance dashboard (upcoming, overdue, completed)

* [ ] **Reminders & Notifications**
  * Scheduled jobs for reminders (7, 3, 1 day before due)
  * Email notifications (SendGrid/Postmark)
  * In-app notifications (bell icon)

* [ ] **Submissions & Evidence**
  * Submission records with status
  * Evidence vault (S3 upload with signed URLs)
  * Link evidence to obligations/submissions
  * Acknowledgement uploads

* [ ] **Audit Log**
  * Immutable log for all critical actions
  * Searchable, filterable view
  * Export audit trail (CSV/PDF)

**Success Criteria:**

* [ ] 20 design partners actively tracking obligations
* [ ] ≥80% of obligations have reminders configured
* [ ] First submissions recorded with evidence attached
* [ ] Zero security vulnerabilities in penetration test

---

#### Milestone 1.3: Invoicing & Accounting Core (Weeks 9–12)

* [ ] **ZRA Smart Invoice Integration**
  * VSDC module setup and configuration wizard
  * Real-time invoice transmission to ZRA
  * Mark ID and QR code generation
  * Validation status tracking
  * Failed transmission retry mechanism
  * Compliance dashboard

* [ ] **Invoicing**
  * Create invoice (line items, taxes, totals)
  * PDF generation (ZRA-compliant with Mark ID/QR code)
  * Send via email
  * Payment receipt recording
  * Customer directory (with TPIN field)
  * Aging report (0-30, 31-60, 61-90, 90+)

* [ ] **Accounting Core**
  * Chart of Accounts (Zambian template, editable)
  * General journal (double-entry validation)
  * Accounting periods (open/close)
  * Trial Balance (real-time)
  * Profit & Loss Statement
  * Balance Sheet
  * Opening balances import (CSV)

* [ ] **Onboarding**
  * Guided setup checklist (8 steps including Smart Invoice)
  * Sample data pre-loaded
  * Skip options for optional steps
  * Progress tracking

**Success Criteria:**

* [ ] 30 design partners live and issuing ZRA Smart Invoice compliant invoices
* [ ] 100% of VAT invoices validated by ZRA (Mark ID assigned)
* [ ] ≥95% invoice transmission success rate within 60 seconds
* [ ] ≥70% onboarding completion rate
* [ ] Trial Balance always balanced (100% validation)
* [ ] Median time to first invoice: **&lt;5 minutes**
* [ ] VSDC setup completion: **&lt;10 minutes** with wizard

---

### Phase 2: Inventory-Lite + Payments + CRM Basics

**Timeline:** January–April 2026 (16 weeks)

* [ ] **Inventory-Lite**
  * SKU/equipment tracking
  * Stock adjustments (add/remove)
  * Low-stock alerts
  * Simple inventory report

* [ ] **Payment Integration**
  * Payment links (Stripe + mobile money)
  * Automated payment reminders (dunning sequences)
  * Receipt auto-matching to invoices

* [ ] **CRM Basics**
  * Customer relationship tracking (contacts, notes, interaction history)
  * Lead/deal pipeline management
  * Email integration (log communications)
  * Task and follow-up reminders
  * Customer segmentation and tagging
  * Sales opportunity tracking

* [ ] **Enhanced Cash Management**
  * Aging report with customer risk scoring
  * Cashflow forecast (30/60/90 days)
  * Payment term templates

**Success Criteria:**

* [ ] 150 active paying organizations
* [ ] ≥50% using Inventory-Lite
* [ ] ≥60% using CRM for customer tracking
* [ ] DSO reduction: **≥25%** for active users
* [ ] Payment link conversion rate: **≥40%**
* [ ] CRM adoption: **≥3 interactions logged per customer**

---

### Phase 3: AI Assists + Integrations + Payroll

**Timeline:** May–December 2026 (32 weeks)

* [ ] **AI Features**
  * Draft ZRA/NAPSA submissions (pre-fill from transaction data)
  * Anomaly detection (flag unusual patterns before filing)
  * Smart reconciliation (auto-match payments to invoices)
  * Cashflow prediction models
  * Report insights and recommendations

* [ ] **Integrations**
  * Bank feed APIs (where available in Zambia)
  * Accounting firm portal (white-label access)
  * Third-party marketplace launch
  * Zapier/Make integration for workflow automation

* [ ] **Payroll Module**
  * Employee records and contract management
  * Salary calculation engine (monthly, bi-weekly, hourly)
  * PAYE calculation (progressive tax rates per ZRA)
  * NAPSA contribution calculation (employer + employee)
  * NHIMA deduction tracking
  * Payslip generation (PDF with YTD totals)
  * Payslip distribution (email/SMS)
  * Statutory filing prep:
    * PAYE schedule for ZRA
    * NAPSA register
    * NHIMA employer report
  * Leave management (annual, sick, unpaid)
  * Timesheet integration for hourly/project workers
  * Payroll journal auto-posting to accounting
  * Payroll reports (by department, by employee, YTD)

**Success Criteria:**

* [ ] 500 active paying organizations
* [ ] ≥30% using AI-assisted submissions
* [ ] ≥40% using Payroll module (Scale/Partner plans)
* [ ] Bank feed connected for ≥20% of orgs
* [ ] Payroll module adoption:
  * Average 12 employees per payroll org
  * 98% payslip distribution success rate
  * Zero payroll calculation errors (tax/NAPSA)
* [ ] CRM pipeline velocity: **≥20% faster** deal closure

---

## 11. Risks & Assumptions

### 11.1 Assumptions

**A1: SMEs will adopt opinionated defaults**

* **Assumption:** If onboarding takes &lt;20 minutes, users will accept template Chart of Accounts and regulator packs rather than demanding full customization upfront
* **Validation:** Design partner feedback during onboarding

**A2: Regulator schedules are stable**

* **Assumption:** ZRA/PACRA/NAPSA/NHIMA due dates and requirements are predictable enough to encode as obligation templates
* **Validation:** Research regulator portals; interview 5 accountants

**A3: Mobile is primary context for owners**

* **Assumption:** Business owners primarily use smartphones for business tasks, not desktop computers
* **Validation:** User interviews (10 SME owners); analytics from design partners

**A4: Payment method diversity is acceptable**

* **Assumption:** SMEs are willing to pay via card OR mobile money OR bank transfer (we don't need to integrate all methods immediately)
* **Validation:** Survey 30 prospects on preferred payment method

**A5: Monthly pricing preferred over annual**

* **Assumption:** Cash-constrained SMEs prefer $50/month over $500/year upfront, even with discount
* **Validation:** A/B test pricing page; track conversion rates

---

### 11.2 Risks

#### R1: Regulator Changes Break Obligation Templates

**Probability:** Medium  
**Impact:** High (breaks core value proposition)

**Risk Description:**  
ZRA changes tax filing deadlines or form formats; our hard-coded obligation templates become outdated, causing users to miss deadlines despite using Bumara.

**Mitigation:**

* Config-driven obligation templates (JSON, not hard-coded)
* Admin dashboard for Bumara team to update templates quickly
* Monitor regulator websites for announcements
* Quarterly review with accountant partners

**Contingency:**

* Managed-service offering: Bumara team files on behalf of customer while updating templates
* Email alerts to users: "ZRA deadline changed; please verify on portal"

**Owner:** Product Manager  
**Review Date:** Monthly

---

#### R2: Low Willingness-to-Pay (Price Sensitivity)

**Probability:** Medium  
**Impact:** Medium (affects unit economics)

**Risk Description:**  
SMEs balk at $50/month subscription, expecting "free" tools like Excel or pirated software. Conversion rate &lt;10% kills growth.

**Mitigation:**

* Clear ROI calculator on website (show $3,000 annual savings)
* Generous 14-day trial (no credit card required)
* Flexible payment: monthly, annual (15% discount), mobile money
* Partner with accountants (bundle into their service fee)

**Contingency:**

* Freemium tier: Free invoicing + limited compliance (10 obligations/month)
* Pay-per-use pricing: $0.50 per submission
* Partner channel: Accountants pay bulk license, resell to clients

**Owner:** CEO / Sales Lead  
**Review Date:** After first 50 customers (track conversion rate)

---

#### R3: Poor Data Quality on Onboarding

**Probability:** High  
**Impact:** Medium (frustration, churn)

**Risk Description:**  
Users upload CSV opening balances with errors (unbalanced debits/credits, missing accounts); system rejects, user gives up.

**Mitigation:**

* CSV template with clear instructions and examples
* Import preview with validation before commit
* Balance check: show "Debits: ZMK 50k | Credits: ZMK 48k | Diff: ZMK 2k — Please fix"
* Optional: Skip opening balances, start fresh from current month

**Contingency:**

* Assisted onboarding for first 50 organizations (Zoom call to walk through import)
* "Fix My CSV" service: User emails CSV, we clean and send back

**Owner:** Implementation Lead  
**Review Date:** After 20 onboardings (track completion rate)

---

#### R4: Feature Creep Delays MVP

**Probability:** Medium  
**Impact:** High (delays revenue)

**Risk Description:**  
Team adds "nice-to-have" features (multi-currency, advanced budgeting) that push MVP launch from December 2025 to March 2026, missing market opportunity.

**Mitigation:**

* Strict P0/P1/P2 gates (anything beyond P0 deferred to Phase 2)
* Change control process: New features require CEO approval + impact assessment
* Weekly sprint reviews: Track scope vs timeline

**Contingency:**

* Cut scope mid-sprint if timeline slips (e.g., remove inventory-lite from MVP, push to Phase 2)
* Launch with 80% features to 30 design partners, iterate in production

**Owner:** Project Manager  
**Review Date:** Weekly sprint planning

---

#### R5: Regulator Portal Downtime During Critical Periods

**Probability:** Medium  
**Impact:** Medium (user frustration, but not Bumara's fault)

**Risk Description:**  
ZRA portal goes down on PAYE due date; users blame Bumara for not being able to submit.

**Mitigation:**

* Clear messaging: "Bumara tracks and reminds; submission happens on ZRA portal"
* Status indicator for regulator portals (if API available)
* Proactive alerts: "ZRA portal is down; submission may be delayed"

**Contingency:**

* Managed submission service: Bumara team submits on behalf of user (premium add-on)
* Document evidence that user attempted submission on time (screenshot, timestamp)

**Owner:** Customer Success Lead  
**Review Date:** During ZRA filing season (monthly for first 3 months)

---

#### R6: ZRA Smart Invoice VSDC Integration Complexity

**Probability:** Medium  
**Impact:** High (breaks core differentiator)

**Risk Description:**  
VSDC integration proves more complex than expected; transmission failures, validation delays, Mark ID assignment errors frustrate users and undermine the "all-in-one" value proposition.

**Mitigation:**

* Use ZRA's official VSDC API specification and sandbox for testing
* Implement robust retry mechanism (3 attempts, exponential backoff)
* Build offline queue for when ZRA system is down
* Monitor ZRA system status proactively (status page integration)
* Provide clear error messages with actionable fixes
* Fallback: Manual upload option if automated transmission fails

**Contingency:**

* Partner with ZRA-approved third-party Smart Invoice provider for white-label integration
* Managed service: Bumara team monitors failed transmissions and manually resolves
* Extended VSDC setup support: 1-on-1 Zoom calls for first 100 customers

**Owner:** CTO / Lead Developer  
**Review Date:** Weekly during development (Weeks 9-12), then monthly

---

### 11.3 Dependencies

#### External Dependencies

* **Clerk (Auth):** SaaS uptime, JWT reliability
* **Email Service (SendGrid/Postmark):** Delivery rates, spam filters
* **Object Storage (S3):** Upload/download performance, cost scaling
* **Stripe (Payments):** Payment success rates, mobile money integration
* **ZRA Smart Invoice System:** VSDC API availability, transmission success rates, Mark ID assignment speed, system uptime (especially during month-end rush)
* **Regulator Portals:** ZRA, PACRA, NAPSA, NHIMA uptime and API availability (limited)

**Mitigation:**

* Multi-vendor strategy where critical (e.g., backup email provider)
* Cache regulator portal data where possible
* Monitor third-party service status (StatusPage subscriptions)
* **ZRA-specific:** Build offline queue for Smart Invoice transmissions; retry mechanism; direct contact with ZRA IT team for escalations

---

#### Internal Dependencies

* **UI Component Library:** Finalized by Week 2 (blocks frontend dev)
* **Database Schema:** Locked by Week 3 (blocks backend dev)
* **CI/CD Pipeline:** Operational by Week 1 (enables rapid iteration)

**Mitigation:**

* Parallel workstreams where possible (UI mockups while schema designs)
* Weekly dependency review in sprint planning

---

#### Customer Dependencies

* **Customer/Product Data:** Sample customer lists, invoice formats, regulator IDs
* **Design Partner Availability:** 10 SMEs committed to testing MVP

**Mitigation:**

* Pre-recruit design partners with signed agreements (timeline, availability)
* Create synthetic test data as fallback

---

## 12. Appendix

### 12.1 Glossary

* **Obligation:** A regulator-defined filing or action with a specific due date (e.g., ZRA VAT Return due by 18th of following month)
* **Submission:** A recorded completion of an obligation, including evidence and acknowledgement from regulator
* **Acknowledgement:** Official receipt/confirmation from regulator that submission was received (PDF upload or reference number)
* **Evidence:** Supporting documents attached to obligations or transactions (invoices, receipts, contracts, licenses)
* **Chart of Accounts (CoA):** Structured list of general ledger accounts used for double-entry bookkeeping
* **Days Sales Outstanding (DSO):** Average number of days it takes to collect payment after invoice is issued
* **Trial Balance (TB):** Summary of all general ledger account balances; must balance (debits = credits)
* **Regulator Pack:** Pre-configured set of obligations, evidence checklists, and filing formats for a specific regulator (ZRA, PACRA, NAPSA, NHIMA)
* **ZRA Smart Invoice:** Mandatory electronic invoicing system implemented by Zambia Revenue Authority for all VAT-registered taxpayers (effective Oct 1, 2024)
* **VSDC (Virtual Sales Data Controller):** Software module that bridges ERP systems with ZRA's Smart Invoice server for real-time invoice transmission
* **Mark ID:** Unique identifier assigned by ZRA to validate an invoice as compliant with Smart Invoice system
* **TPIN (Taxpayer Identification Number):** Unique tax identification number issued by ZRA to all registered taxpayers in Zambia
* **EFD (Electronic Fiscal Device):** Legacy physical device for recording sales transactions; replaced by Smart Invoice system
* **CRM (Customer Relationship Management):** Software module for managing customer interactions, leads, deals, and sales pipeline (Phase 2)
* **Payroll:** Module for managing employee compensation including salary calculations, statutory deductions (PAYE, NAPSA, NHIMA), and payslip generation (Phase 3)
* **PAYE (Pay As You Earn):** Income tax deducted from employee salaries by employers and remitted to ZRA
* **Lead:** Potential customer or sales opportunity tracked in CRM (Phase 2)
* **Pipeline:** Sales process stages from lead to closed deal, visualized in CRM (Phase 2)

---

### 12.2 References

**Zambian Regulators:**

* ZRA (Zambia Revenue Authority): [www.zra.org.zm](https://www.zra.org.zm)
  * Smart Invoice Portal: [smartinvoice.zra.org.zm](https://smartinvoice.zra.org.zm)
  * Smart Invoice FAQs: [www.zra.org.zm/smart-invoice-learn-more](https://www.zra.org.zm/smart-invoice-learn-more)
  * VSDC API Specification (internal access)
* PACRA (Patents and Companies Registration Agency): [www.pacra.org.zm](https://www.pacra.org.zm)
* NAPSA (National Pension Scheme Authority): [www.napsa.co.zm](https://www.napsa.co.zm)
* NHIMA (National Health Insurance Management Authority): [www.nhima.co.zm](https://www.nhima.co.zm)

**Internal Documentation:**

* Database schema ERD (link to Figma/Miro)
* API specification (OpenAPI/Swagger doc)
* UI component library (Storybook)
* Pricing & packaging analysis (Google Doc)

**Competitive Research:**

* Odoo, Zoho Books, Sage, QuickBooks feature comparison
* User interview synthesis (10 SME owners, 5 bookkeepers)

---

### 12.3 Competitive Landscape

#### Odoo

**Strengths:** Breadth of modules, extensibility, open-source ecosystem  
**Weaknesses:** Heavy setup (20+ hours), requires technical partners, pricing per app + per user gets expensive (K2,000-K8,000+/month when fully configured), **NO native ZRA Smart Invoice integration** (requires custom development)  
**Pricing:** K540–K2,700/user/month ($22–$108/user/month) + implementation fees (K50,000–K200,000 / $2,000–$8,000)  
**Modules:** Separate CRM, Inventory, Payroll modules each add cost  
**Positioning:** Enterprise-focused; overkill for 5-50 employee SMEs  
**Our Advantage:** Faster onboarding (&lt;2 hrs), **ZRA Smart Invoice built-in**, Zambian compliance out-of-the-box, transparent all-inclusive pricing (CRM + Payroll included in plan), no implementation fees

---

#### Zoho Books / Zoho One

**Strengths:** Polished UX, strong value proposition, integrated suite  
**Weaknesses:** Limited Zambian compliance, **NO ZRA Smart Invoice support** (requires separate software K810-K2,160/month), generic regulator workflows  
**Pricing:** K540–K1,350/user/month ($22–$54/user/month) + separate Smart Invoice software  
**Modules:** CRM is separate product (Zoho CRM $320–$1,080/month), Payroll requires add-on  
**Positioning:** Global SMB focus; lacks local fidelity  
**Our Advantage:** Regulator-native (ZRA/PACRA/NAPSA/NHIMA packs), **ZRA Smart Invoice eliminates double data entry**, Africa-first vs global-generic, mobile-first approvals, **actually cheaper when you factor in Smart Invoice software + CRM + Payroll**

---

#### Sage / QuickBooks

**Strengths:** Accounting maturity, brand recognition, accountant network  
**Weaknesses:** Fragmented compliance (requires add-ons/consultants), **separate Smart Invoice software needed** (K810-K2,160/month), desktop-first UI  
**Pricing:** K540–K2,700/month ($22–$108/month) + Smart Invoice software + consultant fees  
**Modules:** No integrated CRM or payroll (requires third-party tools K1,000–K3,000/month)  
**Positioning:** Traditional accounting; not built for mobile-first SMEs  
**Our Advantage:** Mobile-native, compliance-first (not accounting-first), **all-in-one including ZRA Smart Invoice + CRM + Payroll** (no add-ons needed), better total cost of ownership

---

#### Standalone Smart Invoice Software (e.g., ZRA Portal, Third-Party Solutions)

**Strengths:** ZRA-compliant by definition, approved by ZRA  
**Weaknesses:** **Completely disconnected from accounting**, requires duplicate data entry, no cashflow visibility, no compliance tracking for NAPSA/NHIMA/PACRA  
**Pricing:** K810–K2,160/month ($32–$86/month) - software only, no accounting, no CRM, no payroll  
**Positioning:** Single-purpose compliance tool  
**Our Advantage:** **Unified platform**—Smart Invoice + Accounting + Compliance + Cash Management + CRM + Payroll in one system at K799-K7,999/month ($32-$320/month) depending on needs, eliminating double work and data fragmentation. Solo plan (K799/$32/month) is **same price** as Smart Invoice software alone while including full ERP.

---

#### Excel / Google Sheets (Do-It-Yourself)

**Strengths:** "Free," familiar, flexible  
**Weaknesses:** No reminders, no validation, error-prone, no audit trail, **CANNOT issue ZRA Smart Invoice compliant invoices** (penalties from Oct 1, 2024), no CRM, no payroll automation  
**Positioning:** Non-solution; inertia; now illegal for VAT-registered businesses  
**Our Advantage:** Automation (reminders, validations), audit-ready, time savings (8-15 hrs/month), ROI (penalties avoided), **legal compliance** with ZRA Smart Invoice mandate, integrated CRM and Payroll

---

**Pricing Summary:**

| **Competitor Setup** | **Monthly Cost (ZMK)** | **Monthly Cost (USD)** | **What's Missing** |
|---------------------|----------------------|----------------------|-------------------|
| Zoho Books + Smart Invoice | K2,835 | $113 | No NAPSA/NHIMA/PACRA tracking, manual compliance, no CRM |
| QuickBooks + Smart Invoice | K3,105 | $124 | No regulator workflows, desktop-first, no payroll |
| Odoo (configured) + Smart Invoice | K4,185 | $167 | Requires partners, heavy setup, expensive implementation |
| Standalone Smart Invoice only | K1,485 | $59 | NO accounting, NO compliance, NO cash management, NO CRM, NO payroll |
| **Bumara Team (all-in-one)** | **K2,499** | **$100** | **EVERYTHING included—Smart Invoice, Accounting, Compliance, Cash, CRM (Phase 2), Payroll (Phase 3)** |

**Value Proposition:** Bumara Team at K2,499/month ($100/month) costs **LESS** than Zoho + Smart Invoice (K2,835/$113) while including compliance tracking, regulator workflows, and mobile-first UX that competitors don't have. Scale plan (K7,999/$320/month) competes with fully-configured Odoo but deploys in days instead of months and includes payroll + CRM.

**Summary:** Bumara is the **ONLY ERP in Zambia** with native ZRA Smart Invoice integration built-in at competitive all-inclusive pricing. Every competitor requires either:

1. Separate Smart Invoice software (K810-K2,160/month extra + double data entry), OR
2. Custom development/integration (K50,000-K200,000 + weeks of work + technical debt), OR
3. Manual compliance (penalties + risk)

This is our **#1 competitive moat** for the Zambian market.

---

### 12.4 User Personas (Detailed)

#### Primary Persona: Chandamali - Business Owner

**Demographics:**

* Age: 35-45
* Role: Owner/Managing Director
* Company: IT outsourcing, 15 employees
* Location: Lusaka, Zambia

**Goals:**

* Grow revenue without drowning in admin
* Stay compliant (avoid penalties)
* See cashflow in real-time

**Pain Points:**

* Spends 10-15 hrs/month on compliance admin
* Never sure if invoices are paid until bank balance changes
* Scrambles at month-end to answer "How much cash do I have?"

**Tech Profile:**

* Smartphone-first (iPhone or Samsung Galaxy)
* Uses WhatsApp Business for customer comms
* Light Excel user (basic formulas)

**Buying Behavior:**

* Willing to pay K799-K2,499/month if ROI is clear (saves more than it costs)
* Decides fast (within 2 weeks of trial) when pain is acute
* Values testimonials from peers over marketing claims
* Price-sensitive but understands "cheap is expensive" when it causes compliance failures

**Success Criteria:**

* Admin time &lt;5 hours/month
* On-time submissions 100%
* Cashflow visible in &lt;30 seconds

---

#### Secondary Persona: Mutinta - Bookkeeper

**Demographics:**

* Age: 28-38
* Role: Bookkeeper / Assistant Accountant
* Experience: 3-7 years
* Location: Lusaka, Copperbelt

**Goals:**

* Post transactions accurately and quickly
* Never miss a regulator deadline
* Impress boss with clean month-end close

**Pain Points:**

* Manual reconciliations take 5+ hours/week
* Evidence scattered (invoices in email, receipts in WhatsApp)
* NAPSA/NHIMA forms require manual data entry

**Tech Profile:**

* Desktop + mobile (uses both)
* Comfortable with Excel, basic accounting software
* Frustrated by slow, clunky interfaces

**Success Criteria:**

* Same-day posting (invoice → journal entry)
* Evidence attached 100% of the time
* Month-end close in &lt;5 days

---

#### Tertiary Persona: Lindiwe - Finance Manager

**Demographics:**

* Age: 38-50
* Role: Finance Manager / Senior Accountant
* Credentials: ACCA, ZICA
* Experience: 10+ years

**Goals:**

* Accurate, timely financial statements
* Audit-ready records
* Strategic insights for business decisions

**Pain Points:**

* Month-end takes 8-12 days (should be 5)
* Reports require manual adjustments (unbalanced journals)
* Bank loan applications delayed due to incomplete records

**Tech Profile:**

* Desktop-primary
* Expert in accounting principles
* Skeptical of "simple" tools (wants rigor + validation)

**Success Criteria:**

* Trial Balance always balanced
* Report pack ready in &lt;15 minutes
* Zero audit adjustments

---

## 13. Next Steps

### Immediate Actions (This Week)

1. **[ ] Review PRD with core team** (CEO, CTO, Lead Developer, Designer)
   * Schedule 90-minute review meeting
   * Capture feedback in comments
   * Identify any missed requirements

2. **[ ] Lock P0 scope and timeline**
   * Confirm MVP feature list (Section 6.1)
   * Agree on 12-week timeline (Oct-Dec 2025)
   * Commit to "no scope creep" principle

3. **[ ] Break into GitHub issues**
   * Create epics for each major feature (Compliance, Invoicing, Accounting)
   * Break epics into stories with acceptance criteria
   * Label as P0, P1, P2
   * Assign story points (planning poker)

4. **[ ] Estimate and assign**
   * Development team estimates each story
   * Assign stories to developers
   * Identify critical path

5. **[ ] Prioritize Sprint 1 backlog** (Oct 7-20, 2025)
   * Top priority: Auth, Organizations, Database Schema
   * Secondary: Regulator packs data model
   * Stretch: Basic obligations CRUD

6. **[ ] Kickoff and track in GitHub Projects**
   * Create Sprint 1 board
   * Daily standups (async in Slack or 15-min video)
   * Weekly sprint reviews (demo progress)
   * Track velocity (story points completed per sprint)

---

### Key Milestones to Watch

* **Week 4 (Nov 1):** Foundations complete → 10 design partners authenticated
* **Week 8 (Nov 29):** Compliance module complete → First obligations tracked
* **Week 12 (Dec 27):** MVP complete → 30 design partners live, first invoices sent

---

## Approval & Sign-Off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| **Author** | Nsangu Phiri | __________ | Oct 4, 2025 |
| **Reviewer (Engineering)** | [Lead Dev] | __________ | ______ |
| **Reviewer (Design)** | [Designer] | __________ | ______ |
| **Reviewer (Customer Success)** | [CS Lead] | __________ | ______ |
| **Approved (CEO)** | [Your Name] | __________ | ______ |

---

**Document Status:** Ready for Review  
**Next Review Date:** After 50 customer milestone (estimated March 2026)

---

*This PRD is a living document. Updates will be versioned and tracked in the Document Control table at the top.*
