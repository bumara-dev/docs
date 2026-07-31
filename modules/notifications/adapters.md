---
title: "Notifications Channel Adapters"
description: "Email and WhatsApp adapter implementations for the notification system."
---

## Table of Contents

1. [Overview](#overview)
2. [Adapter Interface](#adapter-interface)
3. [Email Adapter (Resend)](#email-adapter-resend)
4. [WhatsApp Adapter (Meta)](#whatsapp-adapter-meta)
5. [SMS Adapter (Future)](#sms-adapter-future)
6. [Error Handling](#error-handling)

---

## Overview

Channel adapters provide a consistent interface for sending notifications through different providers.

| Adapter | Provider | Use Case |
|---------|----------|----------|
| Email | Resend | Transactional emails |
| WhatsApp | Meta Cloud API | Template messages |
| SMS | Africa's Talking (future) | Urgent fallback |

---

## Adapter Interface

### Common Types

```typescript
// packages/notifications/adapters/types.ts

export interface SendResult {
  success: boolean;
  providerMessageId?: string;
  error?: string;
}

export interface AdapterMetadata {
  deliveryId: string;
  notificationId: string;
  tenantId: string;
}

export interface EmailPayload {
  to: string;
  subject: string;
  html: string;
  text: string;
  metadata: AdapterMetadata;
  replyTo?: string;
}

export interface WhatsAppPayload {
  toE164: string;
  templateName: string;
  language: string;
  variables: string[];
  metadata: AdapterMetadata;
}

export interface SmsPayload {
  toE164: string;
  text: string;
  metadata: AdapterMetadata;
}
```

---

## Email Adapter (Resend)

### Configuration

```typescript
// Environment variables
RESEND_TOKEN=re_xxxxx
RESEND_FROM=notifications@bumara.com
```

### Implementation

```typescript
// packages/notifications/adapters/email.adapter.ts

import { Resend } from 'resend';
import type { Env } from '../env';
import type { EmailPayload, SendResult } from './types';

let resendClient: Resend | null = null;

function getResendClient(env: Env): Resend {
  if (!resendClient) {
    resendClient = new Resend(env.RESEND_TOKEN);
  }
  return resendClient;
}

export async function sendEmail(
  env: Env,
  payload: EmailPayload
): Promise<string> {
  const resend = getResendClient(env);
  
  try {
    const response = await resend.emails.send({
      from: env.RESEND_FROM,
      to: payload.to,
      subject: payload.subject,
      html: payload.html,
      text: payload.text,
      reply_to: payload.replyTo,
      headers: {
        'X-Bumara-Delivery-Id': payload.metadata.deliveryId,
        'X-Bumara-Notification-Id': payload.metadata.notificationId,
        'X-Bumara-Tenant-Id': payload.metadata.tenantId,
      },
    });
    
    if (response.error) {
      throw new Error(response.error.message);
    }
    
    return response.data?.id ?? '';
  } catch (error) {
    console.error('Email send failed:', {
      to: payload.to,
      error: error instanceof Error ? error.message : 'Unknown error',
    });
    throw error;
  }
}

// Validate email format
export function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

### Email Templates

Use React Email for rich templates:

```typescript
// packages/email/templates/notification-base.tsx

import {
  Body,
  Container,
  Head,
  Html,
  Preview,
  Section,
  Text,
  Button,
  Tailwind,
} from '@react-email/components';

interface NotificationEmailProps {
  title: string;
  body: string;
  actionUrl?: string;
  actionText?: string;
}

export const NotificationEmail = ({
  title,
  body,
  actionUrl,
  actionText = 'View Details',
}: NotificationEmailProps) => (
  <Tailwind>
    <Html>
      <Head />
      <Preview>{title}</Preview>
      <Body className="bg-gray-50 font-sans">
        <Container className="mx-auto py-8 px-4">
          <Section className="bg-white rounded-lg shadow-sm p-8">
            <Text className="text-2xl font-semibold text-gray-900 mb-4">
              {title}
            </Text>
            <Text className="text-gray-600 mb-6">
              {body}
            </Text>
            {actionUrl && (
              <Button
                href={actionUrl}
                className="bg-blue-600 text-white px-6 py-3 rounded-md"
              >
                {actionText}
              </Button>
            )}
          </Section>
          <Section className="mt-8 text-center">
            <Text className="text-gray-400 text-sm">
              Bumara - Business Compliance Platform
            </Text>
          </Section>
        </Container>
      </Body>
    </Html>
  </Tailwind>
);
```

### Specific Templates

```typescript
// packages/email/templates/filing-due-soon.tsx

import { NotificationEmail } from './notification-base';

interface FilingDueSoonEmailProps {
  filingName: string;
  dueDate: string;
  daysUntilDue: number;
  actionUrl: string;
}

export const FilingDueSoonEmail = ({
  filingName,
  dueDate,
  daysUntilDue,
  actionUrl,
}: FilingDueSoonEmailProps) => (
  <NotificationEmail
    title={`Filing Due in ${daysUntilDue} Days`}
    body={`Your ${filingName} filing is due on ${dueDate}. Please ensure all required documents and information are submitted before the deadline.`}
    actionUrl={actionUrl}
    actionText="View Filing"
  />
);

// packages/email/templates/filing-overdue.tsx

export const FilingOverdueEmail = ({
  filingName,
  dueDate,
  daysPastDue,
  actionUrl,
}: {
  filingName: string;
  dueDate: string;
  daysPastDue: number;
  actionUrl: string;
}) => (
  <NotificationEmail
    title={`URGENT: Filing Overdue`}
    body={`Your ${filingName} filing was due on ${dueDate} (${daysPastDue} days ago). Immediate action is required to avoid penalties.`}
    actionUrl={actionUrl}
    actionText="Submit Now"
  />
);
```

---

## WhatsApp Adapter (Meta)

### Configuration

```typescript
// Environment variables
WHATSAPP_PHONE_NUMBER_ID=xxxxx
WHATSAPP_ACCESS_TOKEN=xxxxx
WHATSAPP_API_VERSION=v18.0
```

### Implementation

```typescript
// packages/notifications/adapters/whatsapp.adapter.ts

import type { Env } from '../env';
import type { WhatsAppPayload } from './types';

const WHATSAPP_API_BASE = 'https://graph.facebook.com';

interface WhatsAppResponse {
  messaging_product: string;
  contacts: Array<{ wa_id: string }>;
  messages: Array<{ id: string }>;
}

interface WhatsAppErrorResponse {
  error: {
    message: string;
    type: string;
    code: number;
    fbtrace_id: string;
  };
}

export async function sendWhatsAppTemplate(
  env: Env,
  payload: WhatsAppPayload
): Promise<string> {
  const apiVersion = env.WHATSAPP_API_VERSION ?? 'v18.0';
  const url = `${WHATSAPP_API_BASE}/${apiVersion}/${env.WHATSAPP_PHONE_NUMBER_ID}/messages`;
  
  // Build template components
  const components = buildTemplateComponents(payload.variables);
  
  const body = {
    messaging_product: 'whatsapp',
    recipient_type: 'individual',
    to: payload.toE164.replace('+', ''), // Remove + prefix for Meta API
    type: 'template',
    template: {
      name: payload.templateName,
      language: {
        code: payload.language,
      },
      components,
    },
  };
  
  try {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${env.WHATSAPP_ACCESS_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    });
    
    if (!response.ok) {
      const errorData = (await response.json()) as WhatsAppErrorResponse;
      throw new WhatsAppError(
        errorData.error.message,
        errorData.error.code,
        response.status
      );
    }
    
    const data = (await response.json()) as WhatsAppResponse;
    return data.messages[0]?.id ?? '';
  } catch (error) {
    console.error('WhatsApp send failed:', {
      to: payload.toE164,
      template: payload.templateName,
      error: error instanceof Error ? error.message : 'Unknown error',
    });
    throw error;
  }
}

function buildTemplateComponents(variables: string[]): Array<{
  type: string;
  parameters: Array<{ type: string; text: string }>;
}> {
  if (variables.length === 0) {
    return [];
  }
  
  return [
    {
      type: 'body',
      parameters: variables.map((text) => ({
        type: 'text',
        text,
      })),
    },
  ];
}

// Custom error class for WhatsApp
export class WhatsAppError extends Error {
  constructor(
    message: string,
    public code: number,
    public httpStatus: number
  ) {
    super(message);
    this.name = 'WhatsAppError';
  }
  
  isRateLimited(): boolean {
    return this.code === 130429 || this.httpStatus === 429;
  }
  
  isInvalidNumber(): boolean {
    return this.code === 131026;
  }
  
  isTemplateError(): boolean {
    return this.code >= 132000 && this.code < 133000;
  }
}

// Validate E.164 phone number
export function isValidE164(phone: string): boolean {
  return /^\+[1-9]\d{6,14}$/.test(phone);
}

// Normalize phone number to E.164
export function normalizeToE164(phone: string, defaultCountry: string = '260'): string {
  // Remove all non-digit characters
  let digits = phone.replace(/\D/g, '');
  
  // Handle Zambian numbers
  if (digits.startsWith('0') && digits.length === 10) {
    digits = defaultCountry + digits.substring(1);
  }
  
  // Add + prefix
  if (!digits.startsWith('+')) {
    return '+' + digits;
  }
  
  return digits;
}
```

### WhatsApp Template Mapping

```typescript
// packages/notifications/templates/whatsapp-templates.ts

import { NotificationEventType, NotificationSeverity } from '@repo/api-services/notifications';

interface WhatsAppTemplateConfig {
  templateName: string;
  variableMapping: (payload: any) => string[];
}

/**
 * WhatsApp Business template mapping.
 * 
 * Templates must be pre-approved in Meta Business Manager.
 * Naming convention: bumara_{category}_{purpose}
 */
export const WHATSAPP_TEMPLATES: Record<string, WhatsAppTemplateConfig> = {
  // Task templates
  [`${NotificationEventType.TASK_ASSIGNED}`]: {
    templateName: 'bumara_task_assigned',
    variableMapping: (payload) => [
      payload.taskTitle,
    ],
  },
  [`${NotificationEventType.TASK_ACTION_REQUIRED}`]: {
    templateName: 'bumara_task_action_required',
    variableMapping: (payload) => [
      payload.taskTitle,
      payload.requiredAction,
    ],
  },
  
  // Filing templates
  [`${NotificationEventType.FILING_DUE_SOON}`]: {
    templateName: 'bumara_filing_reminder',
    variableMapping: (payload) => [
      payload.filingName,
      payload.periodLabel,
      String(payload.daysUntilDue),
    ],
  },
  [`${NotificationEventType.FILING_OVERDUE}`]: {
    templateName: 'bumara_filing_overdue',
    variableMapping: (payload) => [
      payload.filingName,
      payload.periodLabel,
      String(payload.daysPastDue),
    ],
  },
  
  // Payment templates
  [`${NotificationEventType.PAYMENT_REQUIRED}`]: {
    templateName: 'bumara_payment_required',
    variableMapping: (payload) => [
      formatCurrency(payload.amount, payload.currency),  // amount in minor units
    ],
  },
  
  // Submission templates
  [`${NotificationEventType.SUBMISSION_STATUS_CHANGED}`]: {
    templateName: 'bumara_submission_update',
    variableMapping: (payload) => [
      payload.newStatus,
      payload.regulatorCode,
    ],
  },
};

function formatCurrency(cents: number, currency: string): string {
  const amount = cents / 100;
  return new Intl.NumberFormat('en-ZM', {
    style: 'currency',
    currency: currency || 'ZMW',
  }).format(amount);
}

/**
 * Get template configuration for an event.
 */
export function getWhatsAppTemplateConfig(
  eventType: NotificationEventType
): WhatsAppTemplateConfig | null {
  return WHATSAPP_TEMPLATES[eventType] ?? null;
}
```

### Meta Business Manager Templates

Create these templates in Meta Business Manager:

| Template Name | Category | Body |
|--------------|----------|------|
| `bumara_task_assigned` | UTILITY | You have been assigned a new task: &#123;&#123;1&#125;&#125;. Log in to Bumara to view details. |
| `bumara_task_action_required` | UTILITY | Action required on task: &#123;&#123;1&#125;&#125;. &#123;&#123;2&#125;&#125;. Please complete this as soon as possible. |
| `bumara_filing_reminder` | UTILITY | Reminder: Your &#123;&#123;1&#125;&#125; for &#123;&#123;2&#125;&#125; is due in &#123;&#123;3&#125;&#125; days. Submit before the deadline to avoid penalties. |
| `bumara_filing_overdue` | UTILITY | URGENT: Your &#123;&#123;1&#125;&#125; for &#123;&#123;2&#125;&#125; is &#123;&#123;3&#125;&#125; days overdue. Submit immediately to avoid penalties. |
| `bumara_payment_required` | UTILITY | Payment of &#123;&#123;1&#125;&#125; is required to proceed with your submission. Please complete payment on Bumara. |
| `bumara_submission_update` | UTILITY | Your submission status has been updated to: &#123;&#123;1&#125;&#125; for &#123;&#123;2&#125;&#125;. Log in to Bumara for details. |

---

## SMS Adapter (Future)

### Interface (Placeholder)

```typescript
// packages/notifications/adapters/sms.adapter.ts

import type { Env } from '../env';
import type { SmsPayload } from './types';

/**
 * SMS adapter (not implemented - deferred)
 * 
 * Intended provider: Africa's Talking
 * Use case: URGENT severity fallback when WhatsApp fails
 */
export async function sendSms(
  env: Env,
  payload: SmsPayload
): Promise<string> {
  throw new Error('SMS adapter not implemented');
}

// Placeholder for future implementation
// interface AfricasTalkingConfig {
//   username: string;
//   apiKey: string;
//   shortCode?: string;
// }
```

---

## Error Handling

### Error Categories

| Error Type | Retryable | Action |
|-----------|-----------|--------|
| Rate limit | Yes | Retry with backoff |
| Invalid recipient | No | Mark failed, skip |
| Template error | No | Log, alert, skip |
| Network timeout | Yes | Retry immediately |
| Auth error | No | Alert ops, pause |

### Adapter Error Detection

```typescript
// packages/notifications/adapters/errors.ts

export function isRetryableError(error: unknown): boolean {
  if (error instanceof WhatsAppError) {
    return error.isRateLimited();
  }
  
  if (error instanceof Error) {
    const message = error.message.toLowerCase();
    return (
      message.includes('rate limit') ||
      message.includes('timeout') ||
      message.includes('temporarily') ||
      message.includes('econnreset') ||
      message.includes('socket hang up')
    );
  }
  
  return false;
}

export function isPermanentError(error: unknown): boolean {
  if (error instanceof WhatsAppError) {
    return error.isInvalidNumber() || error.isTemplateError();
  }
  
  if (error instanceof Error) {
    const message = error.message.toLowerCase();
    return (
      message.includes('invalid') ||
      message.includes('not found') ||
      message.includes('blocked') ||
      message.includes('unsubscribed')
    );
  }
  
  return false;
}
```

---

## Testing Adapters

### Mock Adapters for Tests

```typescript
// packages/notifications/adapters/__mocks__/email.adapter.ts

import type { EmailPayload } from '../types';

const sentEmails: EmailPayload[] = [];

export async function sendEmail(
  env: any,
  payload: EmailPayload
): Promise<string> {
  sentEmails.push(payload);
  return `mock_email_${Date.now()}`;
}

export function getSentEmails(): EmailPayload[] {
  return [...sentEmails];
}

export function clearSentEmails(): void {
  sentEmails.length = 0;
}
```

### Integration Tests

```typescript
// packages/notifications/adapters/__tests__/email.adapter.test.ts

import { describe, it, expect, beforeEach } from 'vitest';
import { sendEmail, isValidEmail } from '../email.adapter';

describe('Email Adapter', () => {
  describe('isValidEmail', () => {
    it('validates correct email formats', () => {
      expect(isValidEmail('test@example.com')).toBe(true);
      expect(isValidEmail('user.name@company.co.zm')).toBe(true);
    });

    it('rejects invalid email formats', () => {
      expect(isValidEmail('invalid')).toBe(false);
      expect(isValidEmail('@example.com')).toBe(false);
      expect(isValidEmail('test@')).toBe(false);
    });
  });
});
```

