---
title: "Series Plan: Building a Conversational AI Platform"
draft: true
---

# Blog Series: "Building a Conversational AI Platform from Scratch"

Architecture patterns from a production platform — 250K+ monthly interactions, 20+ enterprise clients.
1 post per day. 12 days. All code is original mock illustrations, no proprietary source code.

## Content Rules

- NO company names — use "a VC firm", "an enterprise client", "the platform"
- NO production code — all examples are original mock code illustrating the pattern
- NO internal file paths or repo names
- OK: personal metrics, tech stack names, architectural patterns, concept names (Agenda Engine)

## Scope

**Chat Flow:** Q&A mode + Agenda mode (Convey, Gather, Debate, Q&A items only)
**Channels:** Webchat, Widget, WhatsApp, Messenger
**Architecture:** Backend (Node.js) ↔ gRPC ↔ AI Service (Python) ↔ LLM

## Day-by-Day Schedule

---

### Day 1 — The Agenda Engine: Why Pure LLM Prompting Fails for Business Conversations ✅
- Type: Case Study | Tags: AI Research, Backend
- The problem: LLMs can't reliably follow multi-step workflows
- The solution: hybrid state machine — AI for language, code for structure
- Two modes: Q&A (user leads) vs Agenda (AI leads)
- Four core items: Convey, Gather, Debate, Q&A
- The Collect → Verify → Action pattern
- Two-service architecture: Backend ↔ gRPC ↔ AI Service
- Example: screening candidates for a VC firm
- 4 SVG diagrams included

---

### Day 2 — How Q&A Mode Works: Building a Knowledge-Powered Chat with RAG ✅
- Type: Tutorial | Tags: AI Research, Backend
- Q&A is the default mode before any agenda triggers
- RAG pipeline: document ingestion → chunking → vector embeddings → semantic search
- Knowledge sources: PDFs, webpages, spreadsheets — scoped per agent
- Retrieval with language-specific scoring across 40+ languages
- Gap detection: alerting admins when the knowledge base can't answer a question
- Transition to Agenda: how trigger phrases or auto-start switch modes

---

### Day 3 — Streaming AI Responses: From LLM to Browser in Chunks
- Type: How-to Guide | Tags: Backend, Frontend
- The two-hop streaming architecture: LLM → AI Service (gRPC) → Backend (HTTP) → Browser
- gRPC server streaming with protobuf EventResponse messages
- Backend forwards chunks via HTTP chunked transfer encoding
- Event types: ChunkEvent (token-by-token), EndMessage, StartAgenda, EndItem, Error
- Frontend: reading chunked JSON, progressive text rendering
- Multi-message grouping on messaging channels

---

### Day 4 — Real-Time Conversation State with Firebase Firestore
- Type: Tutorial | Tags: Backend, Frontend
- Why Firebase over WebSockets: persistent state, offline support, multi-client sync
- Real-time modules: conversations, agenda progress, collected outputs
- Admin dashboard: watching live conversations as they happen
- Separation of concerns: Firebase for state sync, streaming API for chat text
- Frontend: React hooks subscribing to Firestore snapshots

---

### Day 5 — The Convey Item: Delivering Scripted and AI-Generated Messages
- Type: Tutorial | Tags: AI Research, Backend
- Two modes: static (exact text) vs dynamic (AI generates from prompt + context)
- When to use which: legal disclaimers (static) vs personalized closings (dynamic)
- Placeholder substitution: outputs from earlier items inject into messages
- Lifecycle: on_start → emit message → on_end (simplest item type)
- Patterns: welcome messages, progress updates, confirmation summaries

---

### Day 6 — The Gather Item: Collecting Structured Data Through Conversation
- Type: Tutorial | Tags: AI Research, Backend
- Asked vs Inferred outputs: explicit questions vs AI extraction from context
- Field state machine: PENDING → PROVIDED / INVALID / STALE / PRE_FILLED
- Regex validation: email, phone, URL, date formats
- Confirmation: repeating data back for user approval
- Pre-populate: auto-fill from conversation history to avoid redundant questions
- Stale threshold: after 3 failed attempts, move on

---

### Day 7 — The Debate Item: AI-Generated Follow-Up Questions for Deep Conversations
- Type: Tutorial | Tags: AI Research
- How Debate differs from Gather: open-ended exploration vs specific field collection
- Starting question: static or dynamically generated from context
- Follow-up generation: AI reads response and probes deeper (configurable depth)
- Output extraction: after debate, AI summarizes into structured inferred outputs
- Use cases: candidate screening, consultative sales, needs assessment

---

### Day 8 — The Q&A Item: Knowledge Base Access Mid-Agenda
- Type: How-to Guide | Tags: AI Research, Backend
- Problem: users have questions during structured flows ("What's your return policy?")
- Solution: Q&A item pauses agenda, answers from knowledge base, then resumes
- Scoped knowledge: documents assigned to specific agendas or channels
- Transition detection: AI detects when user is done asking questions
- Same RAG pipeline as standalone Q&A mode

---

### Day 9 — Multi-Level Prompt Architecture: Agent → Agenda → Item
- Type: Tutorial | Tags: AI Research
- Level 1 — Agent: role, personality, communication style, knowledge context
- Level 2 — Agenda: purpose, trigger context, pre-population prompts
- Level 3 — Item: type-specific (Convey prompt, Gather extraction logic, Debate instructions)
- Dynamic placeholders: outputs from earlier items substitute into later prompts
- The compound effect: each level adds specificity

---

### Day 10 — Conditional Logic: Deciding Which Agenda Items Run
- Type: How-to Guide | Tags: AI Research, Backend
- Two modes: AI-evaluated ("only if user has 5+ years experience") vs JSON operators ($eq, $ne, $gt, $and, $or)
- When to use which: AI for subjective, JSON for deterministic
- Item skip flow: condition false → skip → evaluate next
- Branching example: senior path vs junior path based on gathered data
- Nested conditions for complex enterprise workflows

---

### Day 11 — State Machine: Event-Driven Agenda Transitions
- Type: How-to Guide | Tags: Backend
- Redis for fast state access during conversations
- PostgreSQL for durable agenda records
- 8 events: select, start, start-item, update, end-item, end, resume, exit
- Status tracking: completed, timed out, exited, paused by live chat
- Timeout handling via delayed queue jobs
- Data flow between items via placeholder substitution

---

### Day 12 — Four Channels, One Engine: Webchat, Widget, WhatsApp, and Messenger
- Type: Tutorial | Tags: Backend, DevOps
- Adapter pattern: all channels normalize to unified message format
- Webchat: full-page, pre-collection form, custom domains
- Widget: embedded on any website, floating icon
- WhatsApp: webhook ingestion, multi-message grouping, 48-hour window
- Messenger: Facebook Pages DM integration
- Channel-specific behaviors: pre-collection skips Gather fields, language switching
- Cross-channel continuity: same user continues across channels

---

## Series Architecture

```
Day 1: Agenda Engine (Why + What)
  ├── Day 2: Q&A Mode + RAG (default chat)
  ├── Day 3: Streaming (how responses reach the user)
  ├── Day 4: Firebase (live state sync)
  │
  ├── Day 5: Convey (deliver messages)
  ├── Day 6: Gather (collect data)
  ├── Day 7: Debate (AI follow-ups)
  ├── Day 8: Q&A Item (knowledge mid-agenda)
  │
  ├── Day 9: Prompt Architecture (how AI is instructed)
  ├── Day 10: Conditional Logic (branching)
  ├── Day 11: State Machine (events + persistence)
  │
  └── Day 12: Channels (webchat, widget, WhatsApp, Messenger)
```

## Internal Linking Strategy

Every post links back to Day 1 (Agenda Engine flagship).
Each item post (5-8) links to Day 9 (prompting) and Day 10 (conditions).
Day 12 (channels) links to Day 3 (streaming) and Day 4 (Firebase).
Day 11 (state machine) links to all item posts (5-8).

## Tags
- AI Research (Days 1, 2, 5, 6, 7, 8, 9, 10)
- Backend (Days 1, 2, 3, 4, 6, 8, 10, 11, 12)
- Frontend (Days 3, 4)
- DevOps (Day 12)
