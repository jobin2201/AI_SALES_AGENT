# 🤖 AI Sales Agent

> A lightweight, LLM-assisted sales automation system built with Python, Streamlit, Groq, Gmail API, Google Calendar API, and MongoDB. No LangChain. No agent frameworks. Just direct, composable Python.

---

## 📌 What Is This?

**AI Sales Agent** is a Streamlit-based web app that automates the early stages of a B2B sales pipeline. It helps you:

- **Score and qualify leads** using rule-based ICP (Ideal Customer Profile) and BANT (Budget, Authority, Need, Timeline) analysis
- **Generate and send personalized outreach emails** via Gmail or SendGrid
- **Auto-reply to inbound messages** using a 3-layer decision stack (LLM → rules → templates)
- **Simulate prospect replies** for testing (via Groq LLM acting as a customer persona)
- **Insert calendar availability** directly into email drafts using Google Calendar's free/busy API

The system is intentionally minimal: there is no LangChain, no vector store, no agent orchestration library. AI is used only where it genuinely adds value — reply generation — and everything else is deterministic Python.

---

## 🗂️ Application Flow

```
streamlit run mainapp.py
        │
        ├─ bootstrap: fixes Python package resolution
        ├─ load_dotenv()       → reads .env for API keys
        ├─ init_db()           → creates MongoDB indexes
        └─ renders Streamlit UI
                │
        User clicks / submits form
                │
   ┌────────────┴──────────────────────────────────┐
   │              ADD NEW LEAD                     │
   │                                               │
   │  → score_icp()  + assess_bant()               │
   │      Pure arithmetic on form fields           │
   │  → Store lead in MongoDB (leads_collection)   │
   │  → generate_email()  [string template]        │
   │  → send_email()  → Gmail Draft or SendGrid    │
   └───────────────────────────────────────────────┘
                │
   ┌────────────┴──────────────────────────────────┐
   │         RUN ONE AUTO-REPLY CYCLE              │
   │                                               │
   │  → Read sender inbox via Gmail API            │
   │  → Deduplicate against MongoDB                │
   │  → For each new message:                      │
   │      1. Try Groq LLM  (llama-3.1-8b-instant) │
   │      2. Fallback → rule-based keyword logic   │
   │      3. Fallback → pre-written template       │
   │  → Create Gmail draft in reply                │
   │  → Log to MongoDB (messages_collection)       │
   │  → Mark Gmail message as read                 │
   │                                               │
   │  Same pipeline runs for customer side         │
   │  (prospect simulation via Groq)               │
   └───────────────────────────────────────────────┘
                │
   ┌────────────┴──────────────────────────────────┐
   │           CALENDAR AVAILABILITY               │
   │                                               │
   │  → Query Google Calendar free/busy API        │
   │  → Compute open windows in user's timezone    │
   │  → Inject availability table into draft body  │
   └───────────────────────────────────────────────┘
```

---

## 🧠 The 3-Layer Decision Stack

When an inbound message arrives, the system tries three layers in order:

### Layer 1 — Groq LLM (`app/conversation_flow/llm.py`)

`generate_sender_reply_plan()` is called with:
- The customer's message
- Lead metadata: name, company, score, industry
- Calendar availability text

It makes a **raw HTTPS POST** to `https://api.groq.com/openai/v1/chat/completions` and expects a JSON response:

```json
{
  "subject": "...",
  "body": "...",
  "intent": "...",
  "should_close": true | false
}
```

Model: `llama-3.1-8b-instant` (configured via `.env`)

> No LangChain memory, no chain objects, no vector store. Just a direct HTTP call using `urllib.request`.

`generate_customer_reply_plan()` works the same way but tells the LLM to act as a prospect ("Daniel") replying naturally to a sales email — used for simulation/testing.

---

### Layer 2 — Rule-Based Logic (`app/services/reply_analyzer.py`)

If the LLM call fails or returns no body, keyword matching takes over:

| Signal | Keywords Checked |
|---|---|
| Positive intent | `yes`, `interested`, `sounds good`, `tell me more` |
| Negative intent | `not interested`, `remove`, `unsubscribe`, `stop` |
| Objection: price | `price`, `cost`, `expensive`, `budget` |
| Objection: timing | `busy`, `later`, `next quarter`, `not now` |
| Objection: competitor | `salesforce`, `hubspot`, `already using` |
| Scheduling intent | `schedule`, `meeting`, `slot`, `call`, `calendar` |
| Exit | `thank you for helping`, `unsubscribe`, `stop emailing` |

Based on detection, one of these actions is taken:

- `end_sequence` → send a closing/thank-you reply
- `send_objection_handler` → send pricing / timing / competitor reply
- `propose_meeting` → include calendar availability
- `send_proof` → send a case study template

---

### Layer 3 — Pre-Written Templates (`app/services/ai_service.py`)

If neither LLM nor a specific rule fires, `generate_email()` is called with one of these types:

| Template Type | Use Case |
|---|---|
| `first_touch` | Initial cold outreach |
| `price_objection` | Addressing cost concerns |
| `timing_objection` | Addressing "not now" responses |
| `case_study` | Social proof / success stories |
| `meeting_proposal` | Scheduling a discovery call |

Each template uses Python string substitution to inject `lead.name`, `lead.company`, and `lead.title`.

---

## 📊 Feature Breakdown: What Uses AI vs. What Doesn't

| Feature | Mechanism |
|---|---|
| Lead scoring (ICP + BANT) | Pure arithmetic on form fields — **no AI** |
| Email templates | Python string substitution — **no AI** |
| Reply classification | Keyword matching — **no AI** |
| Reply body generation | **Groq LLM** (`llama-3.1-8b-instant`) via raw HTTP |
| Customer/prospect simulation | **Groq LLM** via raw HTTP |
| Gmail drafts | Google Gmail API |
| Calendar availability | Google Calendar free/busy API |
| Persistence | MongoDB |
| UI | Streamlit (reruns on every interaction) |

---

## ⚙️ Lead Scoring — Pure Math, No AI

Scoring is entirely deterministic. `score_icp()` and `assess_bant()` in `ai_service.py` assign numeric weights to form fields (company size, industry match, budget range, decision-maker title, etc.) and sum them up.

There is **no ML model, no embedding, no classifier**. It is arithmetic.

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| UI | [Streamlit](https://streamlit.io) |
| LLM | [Groq](https://groq.com) — `llama-3.1-8b-instant` |
| Email | Gmail API / SendGrid |
| Calendar | Google Calendar API (free/busy) |
| Database | MongoDB |
| Language | Python 3.x |
| HTTP (LLM calls) | `urllib.request` (stdlib only) |

**No LangChain. No LlamaIndex. No AutoGen. No agent framework of any kind.**

---

## 🚀 Getting Started

### 1. Clone the repo

```bash
git clone https://github.com/your-username/ai-sales-agent.git
cd ai-sales-agent
```

### 2. Set up your `.env`

```env
GROQ_API_KEY=your_groq_key
GROQ_MODEL=llama-3.1-8b-instant
MONGODB_URI=mongodb://localhost:27017
GMAIL_CREDENTIALS_PATH=credentials.json
SENDGRID_API_KEY=your_sg_key   # optional
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Run the app

```bash
streamlit run mainapp.py
```

---

## 📁 Project Structure

```
ai_sales_agent/
└── salesagent/
    ├── mainapp.py
    ├── .env
    ├── requiremnts/
    │   ├── requirements-full.txt
    │   └── requirements-minimal.txt
    ├── app/
    │   ├── conversation_flow/
    │   │   ├── llm.py
    │   │   ├── flow.py
    │   │   ├── google_tools.py
    │   │   ├── calendar_tools.py
    │   │   └── config.py
    │   ├── services/
    │   │   ├── ai_service.py
    │   │   ├── reply_analyzer.py
    │   │   ├── email_service.py
    │   │   ├── conversation_memory.py
    │   │   └── two_way_draft_loop.py
    │   ├── data/
    │   │   └── database.py
    │   ├── sample_data/
    │   │   └── seed_data.py
    │   ├── server/
    │   │   ├── server.py
    │   │   └── schemas.py
    │   └── lead_gen3/
    │       └── dashboard.py
    ├── gmail_client_secret.json
    ├── gmail_token.json
    ├── gmail_sender_loop_token.json
    └── gmail_customer_token.json
```

---

## 📄 License

MIT — use freely, modify openly.
