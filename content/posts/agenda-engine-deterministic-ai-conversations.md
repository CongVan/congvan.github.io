---
title: "Agenda Engine: Why LLM Prompting Fails in Production"
date: 2026-04-02T10:00:00+07:00
lastmod: 2026-04-11T16:00:00+07:00
draft: false
tags: ["AI Research", "Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "How I built a state machine that forces LLMs through structured business workflows — 250K+ monthly interactions, zero hallucinated API calls."
description: "How I built a state machine that forces LLMs through structured business workflows — 250K+ monthly interactions, zero hallucinated API calls."
ShowToc: true
weight: 1
seriesTotal: 12
---

{{< series-nav >}}

*Day 1 of 12. All code is original mock illustrating the patterns I designed in production.*

> **TL;DR**
> - **Problem**: LLMs skip steps, hallucinate API calls, and get derailed by creative users. "It usually works" is not acceptable at 250K+ monthly interactions.
> - **Solution**: A hybrid state machine — AI generates language, code enforces structure. Two modes (Q&A and Agenda), four core item types (Convey, Gather, Debate, Q&A).
> - **Pattern**: Every agenda runs Collect → Verify → Action. The code guarantees every step runs in order; the AI only decides *how* to phrase each step.

---

## The Problem: LLMs Can't Follow a Checklist

Ask GPT-4 to "collect the user's name, email, and phone number, then ask about their experience, then summarize and send to our API." It works great the first time. Maybe the second.

By the 500th conversation, it skips the phone number. By the 1000th, it hallucinates an API call. By the 5000th, a creative user convinces it to skip the entire flow.

I learned this building an AI-powered conversational platform for enterprise clients. When a VC firm uses the AI to screen startup founders, "it usually works" is not acceptable.

**LLMs are excellent at generating natural language. They are terrible at enforcing multi-step business processes.**

## The Solution: AI for Language, Code for Structure

I built what I call the **Agenda Engine** — a state machine that controls *what* the AI does, while letting the AI control *how* it says it.

The platform is split into two services:

```
┌──────────────┐  gRPC streaming  ┌──────────────┐      ┌─────────┐
│   Backend    │ ───────────────► │  AI Service   │ ───► │  LLM    │
│  (Node.js)   │ ◄─────────────── │  (Python)     │ ◄─── │ (GPT-4) │
│              │  EventResponse   │               │      └─────────┘
│  • HTTP API  │  stream          │  • Agent      │
│  • Database  │                  │  • Prompts    │
│  • Firebase  │                  │  • Agenda     │
│  • Redis     │                  │    Items      │
└──────────────┘                  └───────────────┘
```

The **backend** handles HTTP, database, caching, and real-time state. The **AI service** handles all LLM interaction, prompt construction, and agenda item execution. They communicate via gRPC server streaming — the backend sends a payload, the AI service streams back events.

The system operates in two conversation modes:

```python
class ConversationState(Enum):
    QA = "qa"                  # User leads — free-form questions
    PRE_AGENDA = "pre_agenda"  # Confirming agenda start
    IN_AGENDA = "in_agenda"    # AI leads — structured flow
    EXIT = "exit"              # User exiting early
```

**Q&A mode** is the default — users ask free-form questions, AI answers from a knowledge base. No structure enforced.

**Agenda mode** is where the engine takes over. The AI follows a predefined sequence of steps, and the code guarantees completion.

![Conversation Mode Transitions — Q&A to Confirm to In Agenda](/images/blog/agenda-conversation-modes.svg)

## The Four Core Building Blocks

Every agenda is built from composable item types. Each one is a class with lifecycle hooks:

```python
class BaseAgendaItem:
    """Every agenda item follows the same lifecycle."""

    async def on_start(self): ...
    async def on_user_response(self, message: str): ...
    async def on_tool_call(self, tool_call: ToolCall): ...
    async def on_assistant_response(self, response: str): ...
    async def on_end(self): ...

    @property
    def should_end(self) -> bool: ...

    @property
    def tools(self) -> list[Tool]: ...
```

Four types handle 80% of use cases:

| Item | What It Does | Example |
|------|-------------|---------|
| **Convey** | Delivers a scripted or AI-generated message | "Thanks for applying! I'll ask you a few questions." |
| **Gather** | Collects specific fields with validation | Name, email (regex), phone — with confirmation |
| **Debate** | Explores a topic with AI follow-up questions | "Tell me about your background" → 3-5 probing questions |
| **Q&A** | Pauses agenda for user questions | "Any questions?" → answers from knowledge base → resumes |

## A Real Example: Screening Candidates for a VC Firm

One of our enterprise clients — a venture capital firm — uses the platform to screen startup founders. Here's the agenda:

```
Agenda: "Founder Screening"
│
├── Convey: "Thanks for applying. I'll ask about your startup."
│
├── Gather:
│   ├── founder_name  (asked, required)
│   ├── email         (asked, regex validated)
│   └── funding_stage (inferred — AI extracts from conversation)
│
├── Debate: "Tell me about the problem you're solving"
│   ├── AI follow-up: "How big is this market?"
│   ├── AI follow-up: "Who are your competitors?"
│   └── AI follow-up: "What's your unfair advantage?"
│
├── Debate: "What's your go-to-market strategy?"
│   ├── AI follow-up: "How did you validate this?"
│   └── AI follow-up: "What's your CAC/LTV expectation?"
│
├── Q&A: "Do you have any questions about the program?"
│   └── (answers from knowledge base, then resumes agenda)
│
└── Convey (dynamic): AI generates personalized closing based on responses
```

![Founder Screening Agenda — 6 items using Convey, Gather, Debate, and Q&A](/images/blog/agenda-vc-screening.svg)

The code guarantees every founder goes through every step. The AI can't skip email collection. It can't hallucinate a funding stage. When a founder asks "What's your portfolio?" — the Q&A item handles it, then the agenda resumes exactly where it left off.

## How the Flow Works

### Step 1: The gRPC Request

When a user sends a message, the backend builds a payload and streams it to the AI service:

```protobuf
service ConversationService {
  rpc Chat(ChatPayload) returns (stream EventResponse) {}
}

message ChatPayload {
  UserInfo user = 1;
  AgentConfig agent = 2;
  repeated Agenda agendas = 3;
  ConversationState state = 4;
  repeated Message history = 5;
  string model = 6;
}
```

The backend doesn't touch LLM logic. It builds the payload and forwards the response stream to the browser:

```typescript
// Backend: gRPC stream → HTTP chunked response
const response = await aiService.chat(payload)

for await (const event of response) {
  await this.processEvent(event)  // Save to DB, update Firebase
  writer.write(event)              // Stream to browser
}
```

### Step 2: The Agent Processes the Request

Inside the AI service, an **Agent** orchestrates everything through a task hierarchy:

```python
# Task hierarchy determines conversation behavior
class QATask(BaseTask):          # Free-form Q&A from knowledge base
class ConfirmTask(BaseTask):     # "Would you like to start screening?"
class AgendaTask(BaseTask):      # Active agenda — items in sequence
    execution_queue: deque[BaseAgendaItem]
```

The agent runs a loop: receive message → build prompt → call LLM → stream response → process tool calls → check if item is done → advance:

```python
async def chat(self, user_message: str):
    # 1. Check for agenda triggers
    await self.detect_agenda_triggers()

    # 2. Build multi-level prompt
    system_prompt = self.build_prompt()

    # 3. Stream LLM response
    response = await llm.chat(
        system_prompt=system_prompt,
        history=self.history.messages,
        tools=self.current_task.tools,
        stream=True,
    )

    # 4. Process chunks as they arrive
    async for chunk in response:
        if chunk.is_tool_call:
            await self.current_task.on_tool_call(chunk)
        else:
            self.emit(ChunkEvent(content=chunk.text))

    # 5. Advance if current item is done
    if self.current_task.should_end:
        await self.current_task.on_end()
        self.advance_to_next_item()
```

### Step 3: Executing Items in Sequence

![Agenda Item Execution Flow — condition, pre-populate, execute, extract, save](/images/blog/agenda-item-execution.svg)

Each item type has its own logic. Here's how **Gather** collects structured data:

```python
class GatherItem(BaseAgendaItem):
    class FieldState(Enum):
        PENDING = "pending"
        PROVIDED = "provided"
        INVALID = "invalid"
        STALE = "stale"            # Asked 3+ times, give up
        PRE_FILLED = "pre_filled"  # Auto-filled from context

    async def on_start(self):
        # Auto-fill known fields from conversation history
        await self.pre_populate_from_context()

    async def on_tool_call(self, tool_call):
        for field in tool_call.extracted_fields:
            if self.validate(field):       # Regex, type checks
                self.mark_provided(field)
            elif self.attempt_count(field) >= 3:
                self.mark_stale(field)     # Stop asking, move on

    @property
    def should_end(self) -> bool:
        return all(
            field.state != FieldState.PENDING
            for field in self.required_fields
        )
```

And how **Debate** generates follow-up questions:

```python
class DebateItem(BaseAgendaItem):
    async def on_start(self):
        if self.use_dynamic_question:
            question = await self.generate_contextual_question()
        else:
            question = self.starting_question
        self.emit(MessageEvent(content=question))

    @property
    def should_end(self) -> bool:
        return self.follow_up_count >= self.max_follow_ups

    async def on_end(self):
        # Extract structured insights from the discussion
        await self.summarize_conversation()
```

### Step 4: Events Stream Back

The AI service emits structured events. The backend processes each one:

```protobuf
message EventResponse {
  oneof event {
    ChunkEvent chunk = 1;           // Token-by-token text
    EndMessageEvent end = 2;        // Message complete
    StartAgendaEvent start = 3;     // Agenda triggered
    EndAgendaEvent complete = 4;    // Agenda finished
    StartItemEvent item_start = 5;  // Item began
    EndItemEvent item_end = 6;      // Item done, outputs saved
    ErrorEvent error = 7;
  }
}
```

The backend handles each event: save messages to PostgreSQL, update state in Redis, sync to Firebase for the real-time admin dashboard, and forward chunks to the browser.

```
Browser ◄── HTTP chunks ◄── Backend ◄── gRPC stream ◄── AI Service ◄── LLM
  UI          (JSON)      (DB, Redis,   (EventResponse)  (Agent +     (streaming
 render                   Firebase)                       Items)       response)
```

## The Collect → Verify → Action Pattern

Every agenda follows three phases:

![Collect Verify Action — the three-phase pattern for every agenda](/images/blog/agenda-collect-verify-action.svg)

### Collect

**Gather** asks targeted questions. Each output has a type:
- **Asked**: Directly prompted — "What's your email?" Validated with regex.
- **Inferred**: AI extracts from conversation — "user seems to be pre-seed stage" without asking.

**Debate** explores open-ended topics. AI generates follow-up questions based on responses, then extracts structured summaries as inferred outputs.

### Verify

- Regex validation on asked outputs (email, phone, URL formats)
- Confirmation toggle — "Your email is jane@startup.com, correct?"
- Stale threshold — after 3 failed attempts, mark field as stale and move on
- Pre-populate from context — auto-fill from earlier messages so users don't repeat themselves

### Action

On completion, data flows to external systems:
- **Webhooks** — POST JSON to Zapier, internal APIs
- **CRM** — Create deals, update pipelines
- **Spreadsheets** — Append to Google Sheets, Airtable
- **Email** — Send confirmation to the user

## Why This Architecture Works at Scale

Splitting into two services was a key decision:

**The backend** handles what needs to be fast and reliable: HTTP, database writes, caching, real-time sync. It doesn't care about prompts or LLMs.

**The AI service** handles what needs to be flexible: prompt construction, tool calling, agenda item logic, LLM streaming. It doesn't care about HTTP or databases.

They communicate through a well-defined protobuf contract. The AI team iterates on prompts and model tuning without touching the backend. The platform team scales infrastructure without touching AI logic. This separation let us ship weekly with a team of 7 while serving 20+ enterprise clients.

The Agenda Engine itself solves three problems that pure prompting can't:

**Deterministic completion.** Code guarantees every item runs in order. The AI can't skip steps or forget outputs. Each item tracks its own state independently.

**Graceful interruptions.** Users ask off-topic questions mid-agenda. The Q&A item handles it. When they're done, execution resumes at the exact item where it paused.

**Data integrity.** Outputs are typed, validated, confirmed by the user, and stored in a relational database — not parsed from a chat transcript after the fact.

That's the foundation. In the next 11 posts, we'll build each piece.

---

*Next in series: [Day 2 — How Q&A Mode Works: Knowledge-Powered Chat with RAG](/posts/qa-mode-rag-knowledge-chat)*
