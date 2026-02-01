# CROWNED N BEAUTY AI - Implementation Guide

## Complete Setup Instructions for the Beauty Business Automation Platform

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Airtable Setup](#airtable-setup)
4. [Make.com Integration](#makecom-integration)
5. [Stripe Configuration](#stripe-configuration)
6. [Tally Forms Setup](#tally-forms-setup)
7. [Email Templates](#email-templates)
8. [Testing Checklist](#testing-checklist)
9. [Client Onboarding](#client-onboarding)
10. [Pricing & Costs](#pricing--costs)

---

## Overview

CROWNED N BEAUTY AI is a multi-tenant SaaS platform that automates beauty business operations including:

- **Booking Management** - Tally forms → Airtable → Stripe invoicing
- **Payment Processing** - Automated deposit collection and refunds
- **Appointment Reminders** - 24-hour automated notifications
- **No-Show Enforcement** - Policy-based deposit forfeiture and blacklisting
- **Chargeback Defense** - AI-powered dispute response automation
- **Client Risk Scoring** - Automatic tracking of no-show rates

### Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Tally.so   │────▶│  Make.com   │────▶│  Airtable   │
│   Forms     │     │  Scenarios  │     │  Database   │
└─────────────┘     └──────┬──────┘     └─────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌─────────┐ ┌─────────┐ ┌─────────┐
        │ Stripe  │ │  Gmail  │ │ Google  │
        │ Connect │ │  API    │ │Calendar │
        └─────────┘ └─────────┘ └─────────┘
```

---

## Prerequisites

Before starting, ensure you have accounts for:

- [ ] **Airtable** - Pro plan recommended ($20/mo)
- [ ] **Make.com** - Core plan minimum ($9/mo)
- [ ] **Stripe** - Business account with Connect enabled
- [ ] **Gmail** - Google Workspace account
- [ ] **Google Calendar** - For appointment scheduling
- [ ] **Tally.so** - Free or Pro plan
- [ ] **OpenAI** - API access for dispute defense

---

## Airtable Setup

### Step 1: Create the Base

1. Log into Airtable
2. Create a new base named: `Crowned N Beauty AI - Master`
3. Delete the default table

### Step 2: Create Tables

Create tables in this order (due to linked record dependencies):

#### 2.1 BUSINESSES Table

Reference: `airtable/schema/businesses.json`

**Fields to create:**
| Field Name | Type | Notes |
|------------|------|-------|
| Business ID | Auto Number | |
| Business Name | Single Line Text | Required |
| Owner Name | Single Line Text | Required |
| Owner Email | Email | Required |
| Owner Phone | Phone Number | |
| Business Address | Single Line Text | |
| Stripe Connect ID | Single Line Text | `acct_xxxxx` format |
| Subscription Tier | Single Select | Starter / Pro / Elite |
| MRR | Currency | $USD |
| Status | Single Select | Active / Paused / Churned |
| Onboarding Date | Date | |
| Business Type | Single Select | Hair Salon / Lash Studio / etc |
| Default Deposit % | Number | e.g., 50 |
| Cancellation Window | Number | Hours (e.g., 48) |
| No-Show Policy | Long Text | |
| Timezone | Single Select | PST / MST / CST / EST |
| Booking Form URL | URL | |
| Cancellation Form URL | URL | |
| Admin Dashboard URL | URL | |
| Logo URL | URL | |
| Brand Color | Single Line Text | Hex code |

**Linked Fields (create after other tables exist):**
- Clients → Links to CLIENTS
- Appointments → Links to APPOINTMENTS
- Policies → Links to POLICIES
- Disputes → Links to DISPUTES
- Email Templates → Links to EMAIL_TEMPLATES

**Computed Fields:**
- Total Clients (Count of Clients)
- Total Appointments (Count of Appointments)
- Total Revenue (Rollup: Sum of Appointments.Deposit Amount)
- Last Activity (Last Modified Time)

#### 2.2 CLIENTS Table

Reference: `airtable/schema/clients.json`

**Key Formulas:**

```
No-Show Rate %:
IF({Total Bookings} > 0, ROUND(({No-Shows} / {Total Bookings}) * 100, 1), 0)

Risk Score:
IF({No-Show Rate %} >= 30, '🔴 High Risk', IF({No-Show Rate %} >= 15, '🟡 Medium Risk', '🟢 Low Risk'))
```

#### 2.3 APPOINTMENTS Table

Reference: `airtable/schema/appointments.json`

**Key Formulas:**

```
Deposit Amount:
ROUND({Service Price} * ({Deposit %} / 100), 2)

Remaining Balance:
{Service Price} - {Deposit Amount}

Hours Before Cancellation:
IF({Cancellation Date}, DATETIME_DIFF({Appointment Date}, {Cancellation Date}, 'hours'), 0)

Refund Eligible?:
IF(
  AND({Appointment Status} = 'Cancelled', {Hours Before Cancellation} >= 48),
  '✅ Yes - Full Refund',
  IF(
    AND({Appointment Status} = 'Cancelled', {Hours Before Cancellation} >= 24, {Hours Before Cancellation} < 48),
    '⚠️ Partial - 50% Refund',
    IF({Appointment Status} = 'Cancelled', '❌ No - Deposit Forfeited', '')
  )
)

Refund Amount:
IF(
  {Refund Eligible?} = '✅ Yes - Full Refund',
  {Deposit Amount},
  IF({Refund Eligible?} = '⚠️ Partial - 50% Refund', ROUND({Deposit Amount} * 0.5, 2), 0)
)
```

#### 2.4 POLICIES Table

Reference: `airtable/schema/policies.json`

After creating the table, add sample policies from the JSON file.

#### 2.5 DISPUTES Table

Reference: `airtable/schema/disputes.json`

#### 2.6 EMAIL_TEMPLATES Table

Reference: `airtable/schema/email_templates.json`

### Step 3: Create Views

For each table, create the views specified in the schema JSON files.

### Step 4: Import Sample Data

Use the sample data files in `airtable/sample-data/` to populate test records:

1. Go to each table
2. Click "..." → "Import data"
3. Select JSON import
4. Upload corresponding sample-data file

---

## Make.com Integration

### Scenario 1: New Booking Request

Reference: `make-scenarios/01-new-booking-request.json`

**Setup Steps:**

1. Create new scenario in Make.com
2. Add **Webhooks > Custom Webhook** module
3. Copy webhook URL and save for Tally form
4. Add remaining modules per the blueprint:
   - Router (for multi-business routing)
   - Airtable - Search Records (get business)
   - Airtable - Search Records (find existing client)
   - Router (new vs returning client)
   - Airtable - Create Record (new client)
   - Tools - Set Variable (consolidate values)
   - Math - Evaluate (calculate deposit)
   - Airtable - Create Record (appointment)
   - Airtable - Search Records (get policy)
   - Stripe - Create Invoice
   - Airtable - Update Record (store Stripe IDs)
   - Airtable - Search Records (get email template)
   - Tools - Text Parser (replace variables)
   - Gmail - Send Email
   - Airtable - Update Record (mark sent)

**Connection Setup:**
- Airtable: Use personal access token or OAuth
- Stripe: Connect via OAuth (enable Connect for multi-business)
- Gmail: OAuth with your sending account

### Scenario 2: Payment Received

Reference: `make-scenarios/02-deposit-payment-received.json`

**Trigger:** Stripe Webhook → `invoice.payment_succeeded`

### Scenario 3: Cancellation Request

Reference: `make-scenarios/03-cancellation-request.json`

**Trigger:** Tally Webhook (cancellation form)

### Scenario 4: No-Show Enforcement

Reference: `make-scenarios/04-no-show-enforcement.json`

**Trigger:** Schedule (every 1 hour)

### Scenario 5: Dispute Defense

Reference: `make-scenarios/05-dispute-defense.json`

**Trigger:** Stripe Webhook → `charge.dispute.created`

**Required:** OpenAI API connection for GPT-4

### Scenario 6: 24-Hour Reminder

Reference: `make-scenarios/06-appointment-reminder.json`

**Trigger:** Schedule (every 6 hours)

---

## Stripe Configuration

### Enable Stripe Connect

1. Go to Stripe Dashboard → Settings → Connect
2. Enable Connect for your platform
3. Configure branding and terms

### For Each Client Business

1. Create Connected Account:
   ```
   Type: Express
   Country: US
   Business Type: Individual or Company
   ```

2. Complete onboarding flow
3. Save `acct_xxxxx` ID to BUSINESSES table
4. Configure webhook endpoints

### Webhook Setup

Add these webhook endpoints in Stripe:
- `invoice.payment_succeeded` → Make.com Scenario 2
- `charge.dispute.created` → Make.com Scenario 5

---

## Tally Forms Setup

### Booking Form

Reference: `tally-forms/booking-form.json`

1. Create new form in Tally
2. Add all fields per the configuration
3. Configure webhook integration with Make.com Scenario 1 URL
4. Customize branding per client
5. Set up hidden `business_id` field

### Cancellation Form

Reference: `tally-forms/cancellation-form.json`

1. Create new form in Tally
2. Add all fields per the configuration
3. Configure webhook integration with Make.com Scenario 3 URL
4. Add conditional logic for reschedule fields

---

## Email Templates

### Template Files

Located in `email-templates/`:

- `booking-confirmation.html` - Sent after booking form submission
- `payment-received.html` - Sent after deposit payment
- `appointment-reminder.html` - Sent 24 hours before appointment
- `cancellation-confirmation.html` - Sent after cancellation processed
- `no-show-notice.html` - Sent when marked as no-show

### Variable Replacement

All templates use `{{variable_name}}` syntax. Make.com Text Parser replaces these with actual values.

### Adding to Airtable

1. Open each HTML file
2. Copy the Email Body content (between `<body>` tags)
3. Create record in EMAIL_TEMPLATES table
4. Fill in metadata from the comment block at bottom of file

---

## Testing Checklist

### End-to-End Test Flow

- [ ] Submit test booking form
- [ ] Verify client record created in Airtable
- [ ] Verify appointment record created with correct calculations
- [ ] Check Stripe invoice created
- [ ] Verify confirmation email received with payment link
- [ ] Make test payment through Stripe
- [ ] Verify payment confirmation email received
- [ ] Verify Google Calendar event created
- [ ] Wait for 24hr reminder (or adjust filter to test)
- [ ] Submit cancellation with 48+ hours notice
- [ ] Verify full refund processed
- [ ] Submit cancellation with <24 hours notice
- [ ] Verify deposit forfeited
- [ ] Create test no-show scenario
- [ ] Verify no-show enforcement runs
- [ ] Test dispute flow (use Stripe test mode)

### Per-Client Testing

Before going live with a new client:

- [ ] Booking form submits correctly
- [ ] Business-specific policies applied
- [ ] Brand color and logo display correctly
- [ ] Emails send from correct address
- [ ] Stripe invoices show client's business name
- [ ] Calendar events go to client's calendar

---

## Client Onboarding

### Onboarding Checklist

**Pre-Setup:**
- [ ] Collect business information
- [ ] Set up Stripe Connect account
- [ ] Determine subscription tier and policies

**Airtable Setup:**
- [ ] Create business record
- [ ] Add custom policies
- [ ] Configure email templates

**Make.com Setup:**
- [ ] Add business route to Scenario 1 router
- [ ] Update Stripe Connect IDs
- [ ] Test end-to-end flow

**Forms Setup:**
- [ ] Clone booking form template
- [ ] Customize fields and branding
- [ ] Set hidden business_id
- [ ] Clone cancellation form
- [ ] Configure webhooks

**Go-Live:**
- [ ] Run full test booking
- [ ] Verify all emails received
- [ ] Test payment flow
- [ ] Schedule 1-week follow-up

---

## Pricing & Costs

### Your Costs Per Client

| Item | Monthly Cost |
|------|-------------|
| Make.com Operations | ~$5-15 |
| Airtable (shared) | ~$2 |
| OpenAI API | ~$5-15 |
| **Total** | ~$12-32 |

### Pricing Tiers

| Tier | Your Price | Your Cost | Profit |
|------|-----------|-----------|--------|
| Starter | $39/mo | ~$12 | $27 |
| Pro | $79/mo | ~$20 | $59 |
| Elite | $149/mo | ~$32 | $117 |

### Setup Fees

| Tier | Fee | Time |
|------|-----|------|
| Starter | $199 | 3-4 hours |
| Pro | $399 | 5-6 hours |
| Elite | $699 | 8-10 hours |

---

## Support & Maintenance

### Weekly Tasks
- Review Make.com scenario logs for errors
- Check for failed webhooks
- Monitor chargeback win rates

### Monthly Tasks
- Review client usage and MRR
- Update email templates as needed
- Optimize Make.com operations

### Quarterly Tasks
- Review pricing and margins
- Update policies and compliance
- Add new features based on feedback

---

## File Structure

```
crowned-automation/
├── airtable/
│   ├── schema/
│   │   ├── businesses.json
│   │   ├── clients.json
│   │   ├── appointments.json
│   │   ├── policies.json
│   │   ├── disputes.json
│   │   └── email_templates.json
│   └── sample-data/
│       ├── businesses.json
│       ├── clients.json
│       └── appointments.json
├── make-scenarios/
│   ├── 01-new-booking-request.json
│   ├── 02-deposit-payment-received.json
│   ├── 03-cancellation-request.json
│   ├── 04-no-show-enforcement.json
│   ├── 05-dispute-defense.json
│   └── 06-appointment-reminder.json
├── email-templates/
│   ├── booking-confirmation.html
│   ├── payment-received.html
│   ├── appointment-reminder.html
│   ├── cancellation-confirmation.html
│   └── no-show-notice.html
├── tally-forms/
│   ├── booking-form.json
│   └── cancellation-form.json
└── docs/
    └── IMPLEMENTATION_GUIDE.md
```

---

## Need Help?

- Review the detailed JSON blueprints in each folder
- Check Make.com logs for execution errors
- Verify Airtable formulas match exactly
- Test with small amounts in Stripe test mode first

**Ready to launch?** Follow this guide step-by-step and you'll have a fully automated beauty business platform! ✨
