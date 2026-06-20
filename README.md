# Unloqs — Monthly Prepayment Nudge Engine

An n8n workflow that sends Indian home-loan borrowers a personalised, monthly "you could prepay this much and save this much" nudge — automatically.

I built it for [Unloqs](https://unloqs.com), the home-loan prepayment app I'm working on, to replace manual outreach. It reads each active loan from Supabase, calculates the real impact of a prepayment (interest saved, months shaved off the tenure) in code, asks an LLM to turn those numbers into a short motivating message, logs it, and emails it.

## How it works

```
Monthly trigger
  → read current RBI repo rate (Supabase)
  → detect whether the rate moved since last run
  → fetch active loans (Supabase view)
  → compute prepayment impact per loan  ← all the math lives here
  → draft the message (OpenAI / gpt-4o-mini)
  → store the nudge (Supabase, for audit + in-app display)
  → email the borrower (SMTP)
```

<img width="1654" height="813" alt="image" src="https://github.com/user-attachments/assets/70cf35ca-98bc-4c01-8ca3-c2d38a2efb19" />

The one design decision worth calling out: **the LLM never does arithmetic.** Every rupee figure, the lakh formatting, the amortisation math — all of it happens in the Code nodes. The model only writes the wording. That keeps the numbers correct and the cost negligible (~₹0.0004 per nudge), and it's the difference between a toy and something you'd actually run in production.

## Example

For a ₹25,00,000 loan at 8.5% with a suggested ₹1,25,000 prepayment, the engine computes ~₹2.75 L saved and ~16 months off the tenure, then the model writes something like:

> **Subject:** A smart move this month
> You're ₹1,25,000 away from saving roughly ₹2.75 lakh in interest and finishing your loan about 16 months early. Small step, big difference — worth a look in Unloqs.

## Stack

n8n · Supabase (Postgres) · OpenAI API (swappable for Groq) · SMTP (swappable for WhatsApp via MSG91). All financial math is plain JavaScript using the standard amortisation formula.

## Running it yourself

Full instructions — Supabase schema, credentials, test data, troubleshooting — are in [SETUP.md](./SETUP.md). The short version:

1. Create a Supabase project and run the SQL in SETUP.md (tables + the `v_active_loan_users` view).
2. Import `unloqs-prepayment-nudge.n8n.json` into n8n.
3. Add three credentials: Supabase, OpenAI, and SMTP.
4. Seed one test loan with your own email and run it once.
5. Flip it to active for the monthly schedule.

Credentials are stripped on export, so nothing sensitive lives in the JSON.

## Cost

Effectively free to run: Supabase free tier, self-hosted n8n, and around ₹5/month of OpenAI usage at 100 nudges. Scales linearly from there.

## Notes

There's no clean public API for the RBI repo rate, so it's kept in a small Supabase config table that you update on MPC announcements (roughly 6–8 times a year). The workflow detects the change and the message adapts accordingly.
