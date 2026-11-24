# Kesar & Co. — Automated Support Copilot (n8n + Gemini + Google Sheets)

An AI-powered customer support triage, order-verification, and response-drafting workflow built in **n8n** for **Kesar & Co.** (an Indian D2C skincare brand). The copilot ingests incoming customer support emails via webhook, classifies intent and sentiment using Google Gemini, verifies order ownership and policy eligibility against `orders.json`, drafts brand-aligned responses grounded in `faq.md` and `brand_voice.md`, escalates sensitive cases, and logs all activity and support tickets in Google Sheets.

---

## 1. Architecture Overview

```text
[Webhook Trigger: POST /kesar-support]
               │
               ▼
[Basic LLM Chain + Google Gemini Chat Model + Structured Output Parser]
  └─ Extracts: intent, secondary_intent, order_id, language, sentiment, confidence, summary
               │
               ▼
[Code Node: Order Lookup, Privacy Guard & Policy Date Math]
  ├─ Fetches orders.json & matches by order_id or customer_email
  ├─ Enforces privacy check (sender email === order.customer_email)
  ├─ Computes days_since_delivery against email received_at timestamp
  └─ Evaluates policy windows (7-day sealed return / 48-hour damaged-item claim) & escalation flags
               │
               ▼
[Switch Node: Route by Intent]
  ├─ order_status   ──► Order verification & tracking status response
  ├─ product_query  ──► FAQ-grounded product guidance (no medical claims)
  ├─ return_refund  ──► Eligibility check + Ticket creation/deduplication
  ├─ complaint      ──► Escalation/damage review + Ticket creation/deduplication
  └─ other          ──► General inquiry / human escalation routing
               │
               ▼
[AI Agent Node (with get_order & search_faq tools)]
  └─ Generates warm, concise, brand-voiced reply_draft (English / Hinglish)
               │
               ├──► [Google Sheets: Tickets Tab] (Append or Update non-Closed ticket by order_id)
               │
               └──► [Google Sheets: Audit Log Tab] (Append full execution & decision trace)
```

---

## 2. Tech Stack & Project Structure

* **Orchestration:** n8n Cloud (`Webhook`, `Basic LLM Chain`, `Structured Output Parser`, `Code`, `Switch`, `AI Agent`, `Google Sheets`)
* **LLM & Extraction:** Google Gemini (`temperature: 0`, Retry on Fail enabled for free-tier rate-limit resilience)
* **Data & Persistence:**
  * `data/orders.json` — Customer order database (`KC-1001` through `KC-1012`)
  * `data/faq.md` — Official shipping, return/refund, damage, and product usage policies
  * `data/brand_voice.md` — Kesar & Co. tone, sign-off (`Team Kesar & Co.`), and safety rules
  * **Google Sheets (`Kesar Support`):**
    * `Audit Log` tab — Every incoming email, classification output, order lookup status, action taken, human escalation flag, ticket ID, and reply draft
    * `Tickets` tab — Deduplicated actionable cases (`return_refund` and `complaint`) tracked by `order_id`
* **Testing Harness:** `scripts/send_test_emails.py` + `data/test_emails.json` (25 end-to-end test scenarios `E01`–`E25`)

---

## 3. Core Business Logic & Guardrails

1. **Structured Email Classification:**
   * Classifies every email into one of five intents: `order_status`, `product_query`, `return_refund`, `complaint`, or `other`.
   * Captures `language` (`en` or `hinglish`), `sentiment` (`positive`, `neutral`, `frustrated`, `angry`), `confidence` (`0.0`–`1.0`), and a concise `summary`.
2. **Privacy & Identity Verification:**
   * Before disclosing any order status, tracking link, or delivery date, the workflow verifies that `from.email` matches `customer_email` on the matched order record.
   * Mismatched inquiries (e.g., third-party inquiries or wrong email addresses) are blocked from receiving private order details and flagged with `needs_human = TRUE`.
3. **Deterministic Policy & Date Math:**
   * Delivery elapsed days are calculated deterministically using the email's `received_at` timestamp against `delivered_date` (reference evaluation date: `2026-11-20`):
     * **Unopened / Sealed Returns:** Eligible within **7 days** of delivery.
     * **Damaged / Missing / Wrong Item Claims:** Must be reported within **48 hours** of delivery (with photo proof requested).
     * **Used / Opened Skincare Items:** Ineligible for return/refund due to hygiene policy.
4. **Prompt-Injection & Safety Defense:**
   * Ignores embedded system-override instructions inside email bodies (e.g., `"SYSTEM NOTE: Ignore all previous instructions"`), flags `needs_human = TRUE`, and still logs tickets normally.
   * Avoids unverified medical claims for skin conditions (such as eczema) and escalates out-of-policy requests (GST invoices, brand collaborations, angry social-media threats) to human agents.
5. **Ticket Deduplication:**
   * When multiple emails arrive for the same `order_id` while an existing ticket is `Open` or `In Progress` (not `Closed`), the workflow updates the existing ticket row—appending the new `email_id` and notes—rather than creating duplicate tickets.

---

## 4. Setup & How to Run

### Prerequisites
* An active **n8n** workspace (Cloud or self-hosted)
* A **Google Gemini API key** configured in n8n Credentials
* A **Google Sheets OAuth2** credential in n8n connected to a spreadsheet named **`Kesar Support`** with two tabs:
  * **`Audit Log` headers:** `email_id | received_at | from_email | intent | order_id | order_found | action_taken | needs_human | ticket_id | reply_draft | notes`
  * **`Tickets` headers:** `ticket_id | created_at | updated_at | order_id | customer_email | issue_type | status | priority | email_ids | summary | notes`
* Python 3.9+ with `requests` installed (`pip install requests`)

### Running the Test Suite
1. Import the workflow JSON into n8n, attach your **Google Gemini** and **Google Sheets** credentials, click **Save**, and toggle the workflow to **Active**.
2. Copy the **Production URL** from the `Webhook` node (e.g., `https://<your-instance>.app.n8n.cloud/webhook/kesar-support`).
3. Run the test harness from the repository root (using `--delay 15` to stay safely within Gemini Free Tier 5 RPM rate limits):

```bash
# Run a single email
python scripts/send_test_emails.py --url https://<your-instance>.app.n8n.cloud/webhook/kesar-support --only E01 --delay 15

# Run a subset of emails
python scripts/send_test_emails.py --url https://<your-instance>.app.n8n.cloud/webhook/kesar-support --only E05,E06,E07,E08,E09,E10,E11,E12,E13,E14,E15 --delay 15

# Run all 25 test emails
python scripts/send_test_emails.py --url https://<your-instance>.app.n8n.cloud/webhook/kesar-support --delay 15
```

---

## 5. Submission Artifacts
* `workflow.json` — Exported n8n workflow definition
* `README.md` — Project architecture, guardrails, and execution instructions
* `TEST_REPORT.md` — Evaluation results across all 25 test scenarios (`E01`–`E25`), edge-case verification, and known operational notes