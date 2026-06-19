# GoodScore — Gemini Research Explained Simply
> For an engineer who wants to understand what's going on, not just what words were used.

---

## Part 1 — The Problem GoodScore Is Solving

### The NBFC Name Confusion Problem (this is a big deal)

When you take a "Buy Now Pay Later" loan through Amazon Pay, you're not actually borrowing from Amazon. Amazon uses a company called **Axio** (previously Capital Float) or **IDFC First Bank** behind the scenes to give you that credit.

So when you open your credit report, you don't see "Amazon Pay Later." You see **"Axio"** or **"IDFC First"** — names you've never heard of.

Normal users panic. They think: *"Who is this? Did someone take a loan in my name? Is this fraud?"* They raise a dispute. The dispute is useless because the entry is actually correct. Chaos ensues.

GoodScore needs to solve this by maintaining a **mapping table** — fintech app name → actual NBFC/bank name — so the AI can tell users "that Axio entry is your Amazon Pay Later loan from March."

Here's the full mapping from the research:

| What you used | Who actually lent you money | What shows on credit report |
|---|---|---|
| Amazon Pay Later | Axio (ex-Capital Float) / IDFC First | Axio / IDFC |
| LazyPay | PayU Finance | PayU Finance India |
| ePayLater | Arthimpact Digital Loans | Arthimpact |
| ZestMoney | Aditya Birla / Tata Capital (varies) | Partner NBFC name |

This mapping database needs to be part of the AI's knowledge base. When a user asks "who is Arthimpact on my report?", the AI should instantly reply: "That's your ePayLater loan."

---

### The Regulatory Rules the AI Must Know (RBI Compliance)

The RBI (India's central bank) has strict rules. The AI system must be built within these walls. Here's what matters for engineering:

**Data rules:**
- All user data must be stored on **servers inside India** (no AWS US East, must use AWS Mumbai or similar)
- The app **cannot access** user's contacts, photos, or call logs — only with explicit one-time KYC consent
- Loan contracts need proper **digital signatures**, not just "I agree" OTP clicks

**Dispute rules:**
- If a user raises a dispute about wrong credit report data, the bureau has **30 days** to fix it
- If they don't fix it in 30 days → user gets **₹100/day compensation** automatically
- The AI should track this timer and auto-generate the compensation claim on Day 31

**Loan settlement rules:**
- If someone has defaulted badly, they can do a **One-Time Settlement (OTS)** — pay less than the full amount owed and close the loan
- RBI rules: OTS requires at least **25% upfront payment**
- After OTS, there's a **12-month cooling-off period** — you can't borrow from that same lender again for a year
- The AI must know these numbers and explain them when helping a user negotiate

---

## Part 2 — The 4 Roles of the AI Assistant (How They Actually Work)

### Role 1: Product Guide

Simple goal: Help users navigate the app without getting lost.

The target user has low financial literacy and low patience. If they can't find something in 10 seconds, they leave.

**What this role does technically:**
- Understands natural language navigation requests ("where do I pay my HDFC bill?")
- Uses **deep linking** — directly opens the right screen in the app from within the chat
- Instantly looks up the NBFC mapping table when a user asks about an unknown name on their dashboard
- Must be **very fast** — this is a simple use case, no heavy AI needed

**Engineering note:** This role should use a lightweight model (Llama 3 8B level) — it's basically a navigation assistant. Overkill to use GPT-4 here.

---

### Role 2: Financial Coach

Goal: Turn a scary, confusing credit report into something the user actually understands.

GoodScore already does this with AI-generated videos. The coach takes the user's specific data and creates a personalized explanation — not generic content.

Example: "Rahul, your score dropped from 690 to 642 last month. Here's why: your HDFC credit card utilization went from 40% to 78% in September, and you missed one EMI on your Bajaj Finserv loan on Sept 14th. Here's what to do next."

**What this role does technically:**
- Pulls the user's actual credit report data
- Generates a **personalized script** based on their specific numbers
- Converts that script into a **video** with a human avatar speaking it — in Hindi, Tamil, Kannada, Telugu, or English
- The video pipeline: Text → Text-to-Speech (TTS) audio → Lip-sync with avatar video → Final video delivered to user

**The video generation pipeline in plain terms:**
1. LLM writes a personalized script
2. A TTS model converts that script to audio in the user's language
3. A model called **Wav2Lip** (a GAN — Generative Adversarial Network) takes that audio + a base video of a human face and makes the lips move in sync
4. The final video is sent to the user

One known bug in Wav2Lip: **"silence leakage"** — the avatar's lips move even during silent pauses in audio. The fix is to use bounding-box normalization and masked attention so the model only reacts to actual audio, not silence.

**Engineering note:** Video generation is slow and expensive. This must run **asynchronously** — user gets a text response first ("your report analysis is ready, video generating..."), then video arrives via WebSocket push when it's done.

---

### Role 3: Action Assistant

Goal: Actually *do things* for the user, not just explain them.

This is the most complex role. It's not a chatbot — it's an agent that executes real workflows.

**Key actions it performs:**

**1. Flexible EMI restructuring**
- User says: "I can't pay my full ₹1 lakh credit card due"
- AI checks their bank balance (via connected account data)
- AI calculates what they can actually afford
- AI talks to the lender's API and requests restructuring — break ₹1 lakh into 5 × ₹20k EMIs
- If the loan is already NPA (Non-Performing Asset / defaulted), it drafts an OTS proposal

**2. Dispute filing for wrong bureau data**
- Identifies errors in the report (e.g., a settled loan still showing as active)
- Extracts the **15-digit Experian Reference Number** and **Unique Transaction ID** from the report
- Asks user to upload a No Objection Certificate (NOC) if needed
- Pushes the dispute directly to the bureau's API

**3. Bill payments, AutoPay setup, reminders**
- Sets reminders for due dates
- Initiates actual bill payments from within the app
- Sets up AutoPay on loans

**Engineering note:** This role needs a **powerful model** (GPT-4o / Claude Sonnet level) because it involves complex reasoning — multi-table database joins, financial math, real API calls. Can't use a small model here.

---

### Role 4: Proactive Assistant

Goal: Don't wait for the user to ask. Monitor their financial situation and alert them **before** something bad happens.

**Key proactive behaviors:**

**1. EMI shortfall prediction**
- Reads the user's SMS data (with consent — parsed on-device for privacy)
- Sees their bank balance, upcoming EMI dates
- If balance is ₹3,000 and an EMI of ₹8,000 is due in 3 days → immediately alerts + offers restructuring

**2. Dispute SLA monitoring**
- Tracks every open dispute + the date it was filed
- On Day 31, if unresolved → auto-generates the ₹100/day compensation claim
- Helps user escalate to the **RBI Integrated Ombudsman portal**

**3. Score improvement celebration + marketplace trigger**
- When score crosses 750 (creditworthy threshold) → proactively shows pre-approved loan offers from partner banks
- This is also a **revenue moment** for GoodScore — lenders pay origination fees for these leads

**4. Regular nudges**
- New credit report pulled → "Your score changed by X. Want to know why?"
- Bill due in 2 days → "Want me to set a reminder or pay it now?"
- Credit utilization above 75% → "This is hurting your score. Want help reducing it?"

---

## Part 3 — The AI Architecture (How to Build It Efficiently)

### The Core Insight: Don't Route Everything to GPT-4

If you send every user query to GPT-4, you'll burn money and have slow responses. The research proposes a **tiered routing system** — match query complexity to model complexity.

**Routing Tiers:**

| Tier | Example query | Role | Model to use | Why |
|---|---|---|---|---|
| Tier 1 — Simple/Navigation | "Where do I see my loans?" | Product Guide | Llama 3 8B / Mistral 7B (self-hosted) | No reasoning needed. Just navigation. Needs to be <100ms. |
| Tier 2 — Educational | "Why did my score drop?" | Financial Coach | Llama 3 70B or GPT-4o-mini + RAG | Needs to retrieve user data + explain it. Medium complexity. |
| Tier 3 — Complex Action | "Help me settle my HDFC default" | Action Assistant | GPT-4o / Claude 3.5 Sonnet + Text-to-SQL | Heavy reasoning, math, multi-step API calls. |

**How routing works technically:**
- User sends a message
- A **lightweight classifier model** (fine-tuned small LLM or embedding + KNN) reads the message
- It decides: is this Tier 1, 2, or 3?
- Dispatches to the right agent
- This classifier itself must be very fast (single-digit milliseconds)

---

### Semantic Caching — Don't Pay for the Same Answer Twice

Thousands of users will ask "how do I raise a dispute?" or "what is credit utilization?" — semantically identical questions with different wording.

Without caching: every question hits the LLM → costs money, adds latency.

**How semantic caching works:**
1. User sends a query
2. Query is converted to a **vector embedding** (a list of numbers representing its meaning)
3. System checks the cache: is there a stored embedding within 92% similarity to this one?
4. If yes → return cached answer instantly (under 10ms). No LLM call.
5. If no → call the LLM, store the answer + embedding in cache for next time

**Tool used:** GPTCache library. Embedding model: `BAAI/bge-m3`. Cache storage: vector DB with HNSW (Hierarchical Navigable Small World) indexing — fast approximate nearest neighbor search.

**Privacy split:**
- Generic questions (what is utilization?) → stored in a **global shared cache**
- Personal questions (what's my due date?) → stored in a **per-user session cache** that expires

The 92% similarity threshold is important — set it too low and you return wrong cached answers; set it too high and you get no cache hits.

---

### Text-to-SQL — Let Users Talk to Their Financial Data

Users will ask things like:
- "How much do I owe in total across all my loans?"
- "Which card am I spending the most on this month?"
- "Have my repayments improved compared to last quarter?"

This data is in a **relational database** (PostgreSQL). You can't use regular RAG (which works on text documents) — you need to query structured tables.

**Text-to-SQL pipeline:**

**Step 1 — Schema Linking**
Don't dump the entire DB schema into the prompt. Instead, use similarity search to find *which tables and columns* are relevant for this specific question.
- "Total Dues" → maps to `loan_accounts.outstanding_balance` column

**Step 2 — Planning Agent**
Before writing SQL, the LLM writes a plain-English plan: "I need to join the loan_accounts table with the user table, filter by user_id, sum the outstanding_balance column."

This planning step prevents the model from jumping into SQL and hallucinating wrong joins.

**Step 3 — SQL Generation + Execution**
LLM writes the SQL query. System runs it against a **read-only replica** of the DB (so the AI can never accidentally mutate data).

**Step 4 — Self-Correction Loop**
If the SQL has a syntax error → PostgreSQL returns an error message → feed that error back to the LLM → it fixes the query → try again. Loops until it works.

**Step 5 — Natural Language Response**
Query result (a table of numbers) → LLM converts it to human-readable answer → "Your total outstanding balance across 6 loans is ₹4,23,000. Your biggest liability is the HDFC home loan at ₹3.1 lakh."

---

## Part 4 — The System Architecture (How All the Pieces Fit Together)

Here's the overall backend structure in plain terms:

```
User App (Android/iOS)
        ↓
API Gateway Service
  - Auth token validation
  - Semantic router (Tier 1/2/3 classifier)
        ↓
┌─────────────────────────────────────┐
│         LLM Orchestration Layer     │
│  - Prompt templates per role        │
│  - Session memory management        │
│  - Text-to-SQL coordination         │
└─────────────────────────────────────┘
        ↓
┌─────────┬─────────┬─────────┐
│ Llama   │ GPT-4o  │ Claude  │
│ (T1/T2) │ (T3)    │ (T3)    │
└─────────┴─────────┴─────────┘
        ↓
┌───────────────────────────┐
│  Read-Only DB Replica     │
│  (loans, scores, bills)   │
└───────────────────────────┘

Async/Background Services:
┌───────────────────────────┐
│  Video Rendering Queue    │
│  (Celery workers)         │
│  TTS → Wav2Lip → Video    │
└───────────────────────────┘

┌───────────────────────────┐
│  Event Scheduler          │
│  - EMI due date checks    │
│  - Dispute SLA tracking   │
│  - Score change detection │
└───────────────────────────┘
```

**WebSocket for video delivery:** Since video takes seconds to render, the app keeps a WebSocket connection open. The user gets a text response immediately. When the video is ready, it's pushed over the WebSocket — no polling needed.

**All compute is in India:** AWS Mumbai region or local providers (Yotta Data Services). RBI compliance requirement — no user financial data leaves Indian territory.

**PII stripping before LLM calls:** Before any query hits an LLM, a middleware layer strips PAN numbers, loan account IDs, phone numbers etc. The LLM never sees raw personal identifiers — only anonymized tokens that get re-mapped in the response layer.

---

## Part 5 — Business Logic / Why This Makes Financial Sense

### Cost Savings (why the AI pays for itself)

**Support deflection:** GoodScore currently runs 24/7 human support teams. The majority of tickets are the same questions: "What is this NBFC on my report?", "How do I raise a dispute?", "Why did my score drop?" The AI handles all of these. Human agents are only needed for emotional escalations or complex edge cases.

**Tier 1 queries are practically free:** By running Llama 3 8B self-hosted (open source, no per-token API cost), the cost of navigational queries drops to near-zero. Only Tier 3 queries (complex) hit paid APIs.

### Revenue Generation (why the AI makes money)

**Subscription retention:** Users stay subscribed when they see real progress. The personalized video coach creates an emotional connection — "this app actually understands *my* situation." Lower churn = more recurring revenue.

**Lending marketplace:** This is GoodScore's biggest revenue driver. When a user's score crosses 750 (creditworthy), the Proactive Assistant shows them loan offers from partner banks. GoodScore earns a **origination fee** every time a user takes a loan through the app. The AI ensures this offer appears at exactly the right moment (post-recovery), maximizing conversion.

---

### Hardware Cost Strategy

| Workload | Hardware choice | Why |
|---|---|---|
| Real-time chat (Tier 1/2) | Self-hosted on NVIDIA L40S | Cost-effective for inference, good throughput |
| Complex queries (Tier 3) | OpenAI/Anthropic API (pay-per-use) | Too rare to justify dedicated GPU |
| Video rendering | Spot instances (NVIDIA L40S / PCIe) | Async, not real-time → use cheap preemptible instances |
| Exploring: AMD Instinct MI350X | Alternative GPU | Potentially cheaper cost-per-token than NVIDIA |

The key insight: **video rendering is not real-time**, so you can use cheap spot instances that can be interrupted — just queue the job and retry if the instance gets preempted.

---

## Quick Reference: Technologies Mentioned

| Technology | What it is | Used for |
|---|---|---|
| Llama 3 8B / 70B | Open-source LLMs by Meta | Self-hosted models for Tier 1/2 queries |
| GPT-4o / Claude 3.5 Sonnet | Frontier LLMs (API) | Tier 3 complex reasoning |
| BAAI/bge-m3 | Embedding model | Converting text to vectors for semantic search/caching |
| HNSW | Vector indexing algorithm | Fast nearest-neighbor search in vector DB |
| GPTCache | Semantic caching library | Avoid duplicate LLM calls |
| Wav2Lip | GAN for lip sync | Making avatar lips sync to audio |
| vLLM | LLM inference engine | High-throughput self-hosted model serving with PagedAttention |
| PostgreSQL | Relational database | Storing loan/score/bill data |
| Celery | Python task queue | Async video rendering jobs |
| WebSocket | Real-time protocol | Pushing video to app when ready |
| RAG | Retrieval-Augmented Generation | Letting LLM look up facts before answering |
| Text-to-SQL | NL → SQL conversion | Querying DB with natural language |
| KNN | K-Nearest Neighbors | Routing queries based on embedding similarity |

---

*Source: Gemini deep research report on GoodScore AI Financial Guide assessment*
