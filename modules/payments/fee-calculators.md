---
title: "Fee Calculators"
description: "Unified fee calculation system for compliance payments."
---

## Overview

The fee calculator module provides a registry-based system for calculating compliance fees across different regulators and fee types. It supports fixed fees, contribution-based fees, progressive tax, and pre-calculated fees with configurable handling fee rates.

## Key Features

- **Calculator Registry**: Auto-routing based on fee key
- **Multiple Fee Types**: Fixed, contribution-based, progressive tax, pre-calculated
- **Source-Specific Handling Fees**: Different rates for filings (0%) vs service requests (5%)
- **Detailed Breakdowns**: Line items for transparency
- **Currency Support**: All amounts in minor units (ngwee for ZMW)

## Fee Types

| Type | Calculator | Use Case | Examples |
|------|------------|----------|----------|
| `fixed` | `FixedFeeCalculator` | Table lookup from regulator_fees | PACRA annual returns, name clearance |
| `contribution_based` | `ContributionBasedCalculator` | Pre-calculated from payroll | NAPSA, NHIMA monthly contributions |
| `progressive_tax` | `ProgressiveTaxCalculator` | Pre-calculated tax bands | ZRA PAYE monthly |
| `pre_calculated` | `PreCalculatedCalculator` | Fee stored on source record | Company registration, turnover tax |

## Handling Fee Rates

Handling fees are charged as a percentage of the regulator fee, with different rates by source type:

| Source Type | Rate | Min | Max | Description |
|-------------|------|-----|-----|-------------|
| `filing` | 0% | - | - | No handling fee for filings |
| `service_request` | 5% | K50 | K5,000 | Service requests incur 5% handling fee |

```typescript
// Configuration in config.ts
export const FEES_CONFIG = {
  HANDLING_FEE_RATES: {
    filing: 0,           // 0% for filings
    service_request: 0.05, // 5% for service requests
  },
  MINIMUM_HANDLING_FEE: 5000,   // K50 in ngwee
  MAXIMUM_HANDLING_FEE: 500000, // K5,000 in ngwee
  DEFAULT_CURRENCY: "ZMW",
};
```

## Quick Start

### Basic Usage

```typescript
import { calculateFee, type BaseFeeContext } from "@repo/payments/calculators";

// Create calculator dependencies
const deps = { db: drizzleDb };

// Calculate fee for a filing (0% handling fee)
const filingResult = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_pacra",
  sourceType: "filing",
  sourceId: "fil_789",
  feeKey: "PACRA_ANNUAL_RETURN_COMPANY",
});

// Calculate fee for a service request (5% handling fee)
const serviceResult = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_pacra",
  sourceType: "service_request",
  sourceId: "sr_456",
  feeKey: "PACRA_NAME_CLEARANCE",
});
```

### Result Structure

```typescript
interface DetailedFeeBreakdown {
  regulatorFee: number;      // Fee in ngwee
  handlingFee: number;       // Handling fee in ngwee
  totalAmount: number;       // regulatorFee + handlingFee
  currency: string;          // "ZMW"
  feeKey: string;            // Fee key used
  feeSource: FeeSource;      // Where fee came from
  calculationType: FeeCalculationType;
  lineItems: FeeLineItem[];  // Detailed breakdown
  handlingFeeInfo: {
    rate: number;            // 0 or 0.05
    calculatedFrom: string;
    minimumApplied?: boolean;
    maximumApplied?: boolean;
  };
  notes?: string;
}
```

## Fee Key Mapping

The registry automatically routes to the correct calculator based on fee key:

```typescript
// PACRA Fixed Fees
PACRA_ANNUAL_RETURN_COMPANY       → fixed
PACRA_ANNUAL_RETURN_BUSINESS_NAME → fixed
PACRA_NAME_CLEARANCE              → fixed
PACRA_CHANGE_DIRECTORS            → fixed
PACRA_CERTIFICATE_GOOD_STANDING   → fixed

// PACRA Pre-calculated
PACRA_COMPANY_REGISTRATION        → pre_calculated

// NAPSA/NHIMA Contribution-based
NAPSA_MONTHLY_CONTRIBUTION        → contribution_based
NHIMA_MONTHLY_CONTRIBUTION        → contribution_based

// ZRA Progressive Tax
ZRA_PAYE_MONTHLY                  → progressive_tax

// ZRA Pre-calculated
ZRA_TURNOVER_TAX_MONTHLY          → pre_calculated
ZRA_WITHHOLDING_TAX               → pre_calculated
```

## Calculator Details

### Fixed Fee Calculator

Looks up fees from the `regulator_fees` table using `regulatorId` + `feeKey`.

```typescript
// Context
interface FixedFeeContext extends BaseFeeContext {
  entityConditions?: {
    entityType?: string;    // e.g., "company", "business_name"
    companyType?: string;   // e.g., "private", "public"
  };
}

// Example
const result = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_pacra",
  sourceType: "filing",
  sourceId: "fil_789",
  feeKey: "PACRA_ANNUAL_RETURN_COMPANY",
  entityConditions: {
    companyType: "private",
  },
});
```

### Contribution-Based Calculator

Accepts pre-calculated `totalContribution` from NAPSA/NHIMA return records.

```typescript
// Context
interface ContributionBasedContext extends BaseFeeContext {
  totalContribution: number;  // In ZMW (major units)
  lateFees?: number;          // Optional late fees
  penalties?: number;         // Optional penalties
}

// Example
const result = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_napsa",
  sourceType: "filing",
  sourceId: "napsa_return_123",
  feeKey: "NAPSA_MONTHLY_CONTRIBUTION",
  totalContribution: 5000.00,  // K5,000
});
```

### Progressive Tax Calculator

Wraps pre-calculated PAYE tax amounts.

```typescript
// Context
interface ProgressiveTaxContext extends BaseFeeContext {
  totalTaxDue: number;     // In ZMW (major units)
  filingPeriod: Date;      // Tax period
}

// Example
const result = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_zra",
  sourceType: "filing",
  sourceId: "paye_123",
  feeKey: "ZRA_PAYE_MONTHLY",
  totalTaxDue: 15000.00,           // K15,000
  filingPeriod: new Date("2025-01-01"),
});
```

### Pre-Calculated Calculator

Retrieves fee from context or source record lookup.

```typescript
// Context
interface PreCalculatedContext extends BaseFeeContext {
  preCalculatedAmount?: number;  // Optional - in ngwee
}

// Example with provided amount
const result = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_pacra",
  sourceType: "service_request",
  sourceId: "sr_456",
  feeKey: "PACRA_COMPANY_REGISTRATION",
  preCalculatedAmount: 75000,  // K750 in ngwee
});

// Example with lookup (company registration)
const result = await calculateFee(deps, {
  organizationId: "org_123",
  regulatorId: "reg_pacra",
  sourceType: "service_request",
  sourceId: "sr_456",
  feeKey: "PACRA_COMPANY_REGISTRATION",
  // preCalculatedAmount not provided - will lookup from companyRegistration record
});
```

## Line Items

Each calculation returns detailed line items for transparency:

```typescript
// Fixed fee example
{
  lineItems: [
    {
      code: "REGULATOR_FEE",
      label: "PACRA Annual Return Fee",
      amount: 50000,  // K500 in ngwee
      metadata: { feeKey: "PACRA_ANNUAL_RETURN_COMPANY", feeId: "fee_123" }
    },
    // Note: HANDLING_FEE omitted for filings (rate = 0%)
  ]
}

// Service request example
{
  lineItems: [
    {
      code: "REGULATOR_FEE",
      label: "PACRA Name Clearance Fee",
      amount: 35000,  // K350 in ngwee
      metadata: { feeKey: "PACRA_NAME_CLEARANCE", feeId: "fee_456" }
    },
    {
      code: "HANDLING_FEE",
      label: "Bumara Service Fee",
      amount: 5000,   // K50 (5% of K350, but minimum K50 applied)
      metadata: { rate: 0.05, minimumApplied: true }
    }
  ]
}

// Contribution-based example
{
  lineItems: [
    {
      code: "CONTRIBUTION",
      label: "Total Contribution",
      amount: 500000,  // K5,000 in ngwee
      metadata: { contributionInMajor: 5000, currency: "ZMW" }
    },
    {
      code: "LATE_FEE",
      label: "Late Filing Fee",
      amount: 25000,  // K250 in ngwee
      metadata: { lateFeeInMajor: 250 }
    },
    // Note: HANDLING_FEE omitted for filings (rate = 0%)
  ]
}
```

## Error Handling

```typescript
import { FeeCalculationError, type FeeCalculationErrorCode } from "@repo/payments/calculators";

try {
  const result = await calculateFee(deps, context);
} catch (error) {
  if (error instanceof FeeCalculationError) {
    switch (error.code) {
      case "CALCULATOR_NOT_FOUND":
        // Unknown fee key
        break;
      case "FEE_NOT_FOUND":
        // Fee not in regulator_fees table
        break;
      case "VALIDATION_ERROR":
        // Invalid context
        break;
      case "SOURCE_NOT_FOUND":
        // Could not lookup pre-calculated fee
        break;
    }
  }
}
```

## Package Structure

```
packages/payments/calculators/
├── index.ts                    # Public exports
├── types.ts                    # Type definitions
├── config.ts                   # Fee configuration, handling fee rates
├── registry.ts                 # Calculator registry
├── errors.ts                   # Error types
├── fixed-fee.ts                # Fixed fee calculator
├── contribution-based.ts       # NAPSA/NHIMA calculator
├── progressive-tax.ts          # ZRA PAYE calculator
└── pre-calculated.ts           # Pre-calculated fees
```

## Integration with Payments Service

The calculators integrate with the payments service layer:

```typescript
// In payments.service.ts
import { calculateFee, type BaseFeeContext } from "@repo/payments/calculators";

async function calculatePaymentForSource(ctx, deps, input) {
  // Build context based on source type and fee key
  const context = await buildCalculationContext(deps, input);

  // Use registry
  const result = await calculateFee(createCalculatorDeps(deps), context);

  // Return normalized breakdown
  return {
    regulatorFee: result.regulatorFee,
    handlingFee: result.handlingFee,
    totalAmount: result.totalAmount,
    currency: result.currency,
    feeKey: result.feeKey,
    feeSource: result.feeSource,
  };
}
```

## Adding New Fee Types

### 1. Add Fee Key Mapping

```typescript
// In config.ts
export const FEE_KEY_CALCULATOR_MAP: Record<string, FeeCalculationType> = {
  // ... existing mappings
  NEW_REGULATOR_FEE: "fixed",
};
```

### 2. Seed Regulator Fee

```sql
INSERT INTO regulator_fees (
  id, regulator_id, fee_key, name, amount, currency
) VALUES (
  gen_random_uuid(),
  'reg_new',
  'NEW_REGULATOR_FEE',
  'New Regulator Filing Fee',
  10000,  -- K100 in ngwee
  'ZMW'
);
```

### 3. Add Custom Calculator (if needed)

```typescript
// In new-calculator.ts
export class NewCalculator implements FeeCalculator<NewContext> {
  readonly calculatorId = "new_type";
  readonly calculationType = "new_type" as const;

  canHandle(context: BaseFeeContext): boolean { /* ... */ }
  validate(context: NewContext): string[] { /* ... */ }
  async calculate(context: NewContext): Promise<DetailedFeeBreakdown> { /* ... */ }
}

// Register in registry.ts
this.calculators.set("new_type", new NewCalculator(deps));
```

## Related Documentation

- [Payments Architecture](/modules/payments/architecture)
- [Payments API](/modules/payments/api)
- [Data Model](/modules/payments/data-model)
- [Compliance Workflow](/regulators/template-system)
