# BrightCare Diagnostics — Multi-Channel AI Agent

A multi-channel customer service agent built in **n8n**, for a fictional
medical diagnostics lab in Lagos, Nigeria. Built as part of the Big Brains
Learning internship (Phases 1–10).

One AI handles messages from **website chat, WhatsApp, and email**,
recognizes the same customer across all three, remembers the conversation,
answers from a fixed knowledge base, and hands off to a human when a
message needs one.

## What it does

- Receives messages from 3 channels and standardizes them into one shape
- Matches each message to a single customer profile (by email or phone),
  merging records as a customer uses more channels
- Classifies intent with an LLM (Groq) — pricing, appointment, complaint,
  technical support, general question, or needs a human
- Retrieves relevant facts from a knowledge base before answering, so the
  AI never invents a price, test, or policy
- Writes a reply shaped for the channel it's replying on (short for chat,
  a proper email with greeting/sign-off, casual for WhatsApp)
- Escalates to a human for complaints, urgent issues, or explicit requests,
  emails a full summary, and stops answering that customer automatically
  until the case is marked resolved
- Delivers replies for real: WhatsApp via UltraMsg, email via Gmail
  (threaded replies, not new messages), website chat via a live widget

## Architecture

```
Website chat ──┐
WhatsApp ───────┼──► Intake workflow (standardize) ──► CRM Upsert ──► AI Brain
Email ──────────┘                                                        │
                                              classify → retrieve KB → reply
                                                           │
                                     route by channel → escalate if flagged
                                                  │              │
                                          deliver reply   email a human
```

## Workflows in this repo

| File | Role |
|---|---|
| `MCA-Website Intake.json` | Receives website chat messages, normalizes them |
| `MCA- Whatsapp Intake.json` | Receives WhatsApp messages (UltraMsg format), normalizes them |
| `MCA-Email Intake.json` | Receives email (Gmail Trigger + a test webhook), normalizes it |
| `MCA-Upsert- CRM.json` | Finds/creates the customer, logs the message, calls the AI Brain |
| `MCA-AI Brain.json` | Classifies intent, retrieves knowledge, writes and delivers the reply |
| `Error wf.json` | Catches failed executions across the above workflows and reports them |

## Stack

- **n8n** (self-hosted, Docker)
- **Groq** (`openai/gpt-oss-120b`) for classification and response generation
- **Airtable** as the simulated CRM and knowledge base (Customers,
  Conversations, Knowledge Base tables)
- **UltraMsg** for WhatsApp send/receive
- **Gmail** (Trigger + Reply) for email send/receive
- A website chat widget (built on Manus), exposed publicly via ngrok during
  local development

## Data model (Airtable)

**Customers**: Customer ID, Name, Email, Phone, Company, Preferred Channel,
First/Last Interaction, Customer Status, Channels Used, Escalation Status

**Conversations**: Conversation ID, Customer ID, Channel, Message ID,
Message, Sender Type, Timestamp, Intent, Response, Analysis

**Knowledge Base**: Entry ID, Category, Title, Content, Keywords, Applies
To Intent — 19 entries covering tests, packages, policies, and FAQs

## Key design decisions

- **Matching is email-or-phone only.** A channel with no shared identifier
  to an existing record correctly creates a new customer rather than
  guessing. This is a stated limitation, not a bug.
- **Retrieval is keyword-based**, scored against the current message *and*
  recent conversation history (not just the current message — this was a
  fix made after testing showed a vague follow-up could otherwise retrieve
  the wrong facts). Appropriate at 19 knowledge base entries; would need a
  different approach at a much larger scale.
- **Escalation is sticky.** Once a customer is flagged, every subsequent
  message from them gets a holding reply instead of a fresh answer, until
  a human resets their status. Verified not to leak across customers.
- **AI calls are validated, not trusted blindly.** Both the classification
  and response steps check the model's JSON output against expected
  fields/enums, and fall back to a safe "needs a human" state if a call
  fails or returns something unusable — with the real failure reason
  recorded (e.g. a rate limit is labeled as that, not as "broken AI output").

## Testing

Tested in stages through 10 phases, then against 30 scripted scenarios
covering: new vs. returning customers, customers switching channels
mid-conversation, all 5 intent categories, questions the knowledge base
has no answer for, escalation from every channel, duplicate customers, and
incomplete contact information. Full write-up in
[`docs/architecture.md`](docs/architecture.md) and
[`docs/architecture.pdf`](docs/architecture.pdf).

## Known limitations

- n8n runs locally; the live demo depends on an ngrok tunnel staying open
- Escalation resolution is manual (no real human agent exists in this
  simulation)
- WhatsApp was tested live, but most two-way testing was done from one
  real number
- This is a fictional business with test data throughout — no real
  customer or production CRM data is used anywhere in this repo

## Fictional business

**BrightCare Diagnostics** — a medical diagnostics lab in Ikeja, Lagos.
8 tests, 3 packages, Mon–Sat 7am–6pm. All prices, policies, and customer
data in this repo are invented for testing.

---

Built by [Akogun Imoleayo](https://github.com/Imoleayo07) as part of the
**Big Brains Learning** AI Automation internship.
