# CROWNED N BEAUTY AI - Quick Start Guide

## Get Started in 30 Minutes

### Step 1: Clone the Airtable Base (10 min)

1. Create a new Airtable base
2. Create these 6 tables in order:
   - BUSINESSES
   - CLIENTS
   - APPOINTMENTS
   - POLICIES
   - DISPUTES
   - EMAIL_TEMPLATES

3. Use the schema files in `airtable/schema/` for field configurations

### Step 2: Set Up Make.com Scenarios (15 min)

1. Start with **Scenario 1: New Booking Request**
2. Create webhook and save the URL
3. Connect your Airtable, Stripe, and Gmail accounts
4. Build remaining scenarios as needed

### Step 3: Create Tally Forms (5 min)

1. Create booking form using `tally-forms/booking-form.json` as reference
2. Add your Make.com webhook URL
3. Create cancellation form similarly

### Step 4: Test Everything

```
1. Submit test booking → Check Airtable + Email
2. Pay test invoice → Check confirmation + Calendar
3. Submit test cancellation → Check refund logic
```

---

## Essential Formulas

Copy these into Airtable:

### Deposit Amount
```
ROUND({Service Price} * ({Deposit %} / 100), 2)
```

### No-Show Rate %
```
IF({Total Bookings} > 0, ROUND(({No-Shows} / {Total Bookings}) * 100, 1), 0)
```

### Risk Score
```
IF({No-Show Rate %} >= 30, '🔴 High Risk', IF({No-Show Rate %} >= 15, '🟡 Medium Risk', '🟢 Low Risk'))
```

### Refund Eligible?
```
IF(AND({Appointment Status} = 'Cancelled', {Hours Before Cancellation} >= 48), '✅ Yes - Full Refund', IF(AND({Appointment Status} = 'Cancelled', {Hours Before Cancellation} >= 24, {Hours Before Cancellation} < 48), '⚠️ Partial - 50% Refund', IF({Appointment Status} = 'Cancelled', '❌ No - Deposit Forfeited', '')))
```

---

## File Reference

| Need | File Location |
|------|--------------|
| Table structures | `airtable/schema/` |
| Sample data | `airtable/sample-data/` |
| Automation blueprints | `make-scenarios/` |
| Email HTML | `email-templates/` |
| Form configurations | `tally-forms/` |
| Full documentation | `docs/IMPLEMENTATION_GUIDE.md` |

---

## Pricing Cheat Sheet

| Tier | Monthly | Setup Fee |
|------|---------|-----------|
| Starter | $39 | $199 |
| Pro | $79 | $399 |
| Elite | $149 | $699 |

**Your margins: 65-75%**

---

Ready for the full guide? See `docs/IMPLEMENTATION_GUIDE.md`
