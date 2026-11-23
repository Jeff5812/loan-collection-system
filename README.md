# Micro-Lending Collections, Restructuring & Legal Router

An n8n workflow that automates delinquent-loan collections for a micro-lending business: it scores each account's risk with an AI agent, contacts borrowers on the right channel, offers restructuring, accepts payments, and escalates high-risk accounts for legal referral — with a daily summary to staff.

## What it does

1. **Daily Trigger** kicks off a scheduled run that fetches all delinquent accounts from Postgres.
2. **Loop Accounts** (batch loop) processes each account one at a time.
3. **Risk & Sentiment Agent** (LLM agent, Google Gemini) reads account/payment history and outputs a structured risk tier + sentiment read, parsed with a Structured Output Parser.
4. **Confidence Check** — if the model's confidence is too low, the account is flagged for human review instead of being auto-routed.
5. **Route by Risk Tier** (switch) sends the account down one of three paths:
   - **Low/medium risk** — a WhatsApp restructuring offer and/or an SMS reminder (Twilio) are sent, and the account is marked contacted in Postgres.
   - **High risk** — a legal referral package is compiled and emailed (Gmail) to the legal team.
   - **Low confidence** — flagged for a human to review manually.
6. **Payment flow** — a public webhook renders a hosted payment page for a given account, and a second webhook confirms payment, clears the delinquent status, logs the payment, and shows a thank-you page.
7. **Daily Trigger (reporting)** also fetches the day's stats and emails a summary report to staff.
8. Every account touch is logged to Postgres via **Log Execution**.

## Architecture

```
Schedule Trigger ─▶ Fetch Delinquent Accounts ─▶ Loop Accounts
                                                     │
                                        Risk & Sentiment Agent (Gemini)
                                                     │
                                            Confidence Check
                                          ┌──────────┴──────────┐
                                     (low conf)             (confident)
                                Flag for Human Review    Route by Risk Tier
                                                          ┌──────┴──────┐
                                                    Low/Med Risk    High Risk
                                              WhatsApp / SMS    Compile Legal Package
                                              Mark Contacted      Send Legal Referral
                                                     │
                                              Log Execution (Postgres)

Webhook: Show Payment Page ─▶ Fetch Account ─▶ Build Payment Page ─▶ Respond With Page
Webhook: Confirm Payment ─▶ Fetch Account ─▶ Clear Delinquent Status ─▶ Log Payment ─▶ Respond With Confirmation

Schedule Trigger (summary) ─▶ Fetch Daily Stats ─▶ Format Summary ─▶ Send Summary Report
```

**Nodes/integrations used:** n8n Schedule Trigger, Postgres, LangChain Agent node (Google Gemini chat model + Structured Output Parser), Code nodes for parsing/formatting, If/Switch for routing, WhatsApp, Twilio (SMS), Gmail, Webhook + Respond to Webhook for the payment flow.

## Running it locally

1. Install and start n8n (self-hosted): `npx n8n` or via Docker.
2. In n8n, go to **Workflows → Import from File** and select `workflow/loan-collections-router.json`.
3. Create the following credentials in n8n and attach them to the matching nodes:
   - Postgres
   - Gmail OAuth2
   - Twilio API
   - WhatsApp API
   - Google Gemini (PaLM) API
4. Provision a Postgres database with the tables this workflow expects (delinquent accounts, payment log, execution log) — see `PRODUCTION.md` for what's built vs. still to design on the data layer.
5. Set the workflow's error workflow (referenced in the workflow settings) or point it at your own.
6. Activate the workflow. The two webhook nodes (`Show Payment Page`, `Confirm Payment`) will expose public URLs once active — use n8n's own base URL, or place a reverse proxy in front for a custom domain.

### Placeholders to fill in after import

Personal and environment-specific values were replaced before publishing:

| Placeholder | Where | Replace with |
| --- | --- | --- |
| `your-n8n-host.example.com` | SMS Reminder, Restructuring Offer | Your n8n's public host (or tunnel) |
| `YOUR_TWILIO_NUMBER` | SMS Reminder | Your Twilio sender number |
| `YOUR_WHATSAPP_PHONE_NUMBER_ID` | Restructuring Offer | Your WhatsApp Business phone number ID |
| `legal-team@example.com` | Send Legal Referral | The legal team's inbox |
| `staff@example.com` | Send Summary Report | The staff inbox for the daily summary |

See `PRODUCTION.md` for an honest breakdown of what's production-hardened today vs. what's still designed-only or not addressed.
