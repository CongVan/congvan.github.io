---
title: "Q&A Item: Mid-Agenda Knowledge Without Losing State"
date: 2026-04-11T19:00:00+07:00
lastmod: 2026-04-11T19:00:00+07:00
draft: false
tags: ["AI Research", "Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "Users interrupt structured flows with off-topic questions all the time. The Q&A item pauses the agenda, answers from the knowledge base, and resumes exactly where it left off — no data loss, no re-asking."
description: "Users interrupt structured flows with off-topic questions all the time. The Q&A item pauses the agenda, answers from the knowledge base, and resumes exactly where it left off."
ShowToc: true
weight: 8
seriesTotal: 12
---

{{< series-nav >}}

*Day 8 of 12. [Day 6](/posts/gather-item-structured-data-collection/) was Gather. [Day 7](/posts/debate-item-ai-follow-up-questions/) was Debate. Today: the item that keeps agendas from falling apart when users ask off-topic questions in the middle.*

> **TL;DR**
> - **Problem**: users in a structured flow (ordering, applying, booking) constantly interrupt with questions the flow didn't anticipate. Without a fix, the AI either ignores them or gets derailed.
> - **Solution**: the Q&A item pauses the active agenda item, answers the question using [RAG](/posts/qa-mode-rag-knowledge-chat/) from the scoped knowledge base, and resumes the exact item — even the exact field within a Gather — when the user is done.
> - **Scoped knowledge**: each agenda can have its own knowledge subset. During a pizza order, only "Pizza X menu + delivery FAQ" is available. During a job application, only "HR policies + role description".
> - **Exit detection**: the AI decides when the user is done asking questions using a detect_end_of_qa tool call — no explicit "back to ordering" prompt needed.

---

## The Scenario That Forced This Item

Picture a pizza ordering flow on WhatsApp. A customer taps the restaurant's number and starts a Cart + Gather agenda: size, toppings, delivery address.

Without a Q&A item, here's what happens when the customer pauses mid-order:

```
🤖 Pizza X: Hey! What size pizza would you like — small, medium, or large?
👤 Jane: Medium
🤖 Pizza X: Great. What toppings?
👤 Jane: Pepperoni and mushrooms. Actually wait — are you guys open tonight?
           I'm thinking of ordering around 9pm.
🤖 Pizza X: Got it, pepperoni and mushrooms. What's your delivery address?
👤 Jane: you're not answering my question 😠
```

The Gather item is doing its job — collecting fields in order. The problem is there's no escape hatch for the off-topic question. The customer feels ignored, and they bail.

The fix is the Q&A item. Here's the same conversation with it wired in:

```
🤖 Pizza X: Hey! What size pizza would you like — small, medium, or large?
👤 Jane: Medium
🤖 Pizza X: Great. What toppings?
👤 Jane: Pepperoni and mushrooms. Actually wait — are you guys open tonight?
           I'm thinking of ordering around 9pm.
🤖 Pizza X: Yep! We're open until 11pm tonight. 9pm is no problem —
           last orders go out around 10:30pm. Anything else you want
           to ask before we continue?
👤 Jane: Do you deliver to District 2?
🤖 Pizza X: We do — District 2 is ~25 minutes out, so you'd get it by 9:25pm
           if you order at 9.
👤 Jane: Perfect, no more questions
🤖 Pizza X: Cool. So back to the order — pepperoni and mushroom medium. 
           What's your delivery address?
```

Same user, same order, totally different experience. The Gather item didn't lose its place. The pepperoni and mushroom toppings are still saved. When Jane eventually provides an address, the flow continues as if the detour never happened.

![Q&A item pauses and resumes the active agenda](/images/blog/qa-item-interrupt-flow.svg)

## How the Interruption Is Detected

The Q&A item isn't a separate item the user triggers — it's **automatic**. Every time a Gather or Debate item is waiting for a response, the agenda engine runs a quick classification on each incoming message: *is this an answer to the current question, or a new question entirely?*

```python
class AgendaItemBase:
    async def on_user_response(self, message: str):
        # Classify: is this message answering the current question or asking a new one?
        intent = await classify_intent(
            current_question=self.current_question,
            user_message=message,
        )

        if intent == 'answer':
            await self.handle_answer(message)

        elif intent == 'question':
            # Pause this item and hand off to Q&A
            qa = QAItem.fork_from(self)
            await qa.on_user_response(message)

            # When Q&A signals done, control flow returns here
            # The item's state (current field, attempts, outputs) is untouched
```

The classifier is a small LLM call with tool calling. It sees the last question the agenda asked, the user's latest message, and returns one of three outcomes:

```python
TOOL_SCHEMA = {
    'name': 'classify_user_intent',
    'parameters': {
        'type': 'object',
        'properties': {
            'intent': {
                'type': 'string',
                'enum': ['answer', 'question', 'exit'],
            },
            'reason': {'type': 'string'},
        },
    },
}
```

- `answer` → current item handles it normally
- `question` → fork to Q&A item
- `exit` → user wants to quit the flow (handled by [exit-agenda event](/posts/state-machine-event-driven-agenda-transitions/))

## The Q&A Item Itself

Once forked, the Q&A item does three things:

1. **Answer the current question** using the same RAG pipeline from [Day 2](/posts/qa-mode-rag-knowledge-chat/) — scoped to the active agenda's knowledge.
2. **Prompt for more questions** — a soft offer, not a requirement ("Anything else before we continue?")
3. **Detect exit** — when the user signals they're done, return control to the paused item.

```python
class QAItem(BaseAgendaItem):
    max_questions: int = 5  # don't let Q&A loop forever
    questions_asked: int = 0

    async def on_user_response(self, message: str):
        self.questions_asked += 1

        # Check if user wants to go back to the main flow
        if await self.detect_end_of_qa(message):
            self.should_end = True
            return

        # Otherwise answer from scoped knowledge base
        docs = await retrieve_context(
            query=message,
            agent_id=self.state.agent.id,
            scope_agenda_id=self.state.agenda.id,  # scoped!
        )

        answer = await llm.chat(
            system_prompt=self.persona.system_prompt(),
            user_prompt=message,
            context_docs=docs,
        )

        # Soft prompt to resume
        if self.questions_asked < self.max_questions:
            answer += "\n\nAnything else before we continue?"

        self.emit(MessageEvent(content=answer))
```

The `detect_end_of_qa` helper is another small tool-calling pass. It looks at the user's latest message and classifies whether they're asking a new question or signaling they're done:

```python
async def detect_end_of_qa(self, message: str) -> bool:
    tool = {
        'name': 'detect_end_of_qa',
        'parameters': {
            'type': 'object',
            'properties': {
                'user_is_done': {
                    'type': 'boolean',
                    'description': (
                        'True if the user is signaling they have no more '
                        'questions and want to continue the main flow. '
                        'Phrases like "no more questions", "that\'s all", '
                        '"okay let\'s continue", "perfect", or simple '
                        'acknowledgments after an answer.'
                    ),
                },
            },
        },
    }
    result = await llm.tool_call(tool, messages=[message])
    return result['user_is_done']
```

"Perfect, no more questions" → `user_is_done = True`. "Do you deliver to District 2?" → `user_is_done = False`, keep answering.

## Scoped Knowledge: Why It Matters

A generic chatbot has one knowledge base. A platform hosting 20+ enterprise clients has 20+ knowledge bases, and within each, the same agent may need totally different knowledge depending on what it's currently doing.

Example: a pizza chain deploys one AI agent. The agent is used for:
- **Ordering** (Cart + Gather agenda) — needs menu, pricing, delivery zones, hours
- **Job applications** (Gather + Debate agenda) — needs role descriptions, hiring process, benefits
- **Customer support** (Q&A mode, no agenda) — needs FAQ, refund policy, contact info

If Jane asks "are you hiring right now?" during an **order**, the Q&A item should politely deflect — the job knowledge isn't in scope. If she asks the same question during **Q&A mode** (no active agenda), the job info is available.

The scoping is enforced at retrieval time:

```python
docs = await retrieve_context(
    query="are you hiring",
    agent_id=pizza_agent.id,
    scope_agenda_id=ordering_agenda.id,  # knowledge scoped to this agenda
)
# docs list excludes everything tagged with job_application_agenda
```

Documents are uploaded with a scope field (`all`, `agenda:ordering`, `agenda:job_application`). The retrieval service filters by scope before running the vector search. If the answer isn't in the scoped knowledge, the item returns a graceful "I can help with your order — for other questions, let me know after we finish."

## Pausing and Resuming a Gather Mid-Field

The trickiest case: a Gather item in the middle of collecting a specific field. When the Q&A item returns control, the Gather has to re-ask the pending field cleanly, not from scratch.

Here's what the state looks like during a pause:

```python
# Snapshot of Gather item state when Q&A kicks in
gather_snapshot = {
    'item_id': 'gather-order-details',
    'outputs': {
        'size': FieldState.PROVIDED,        # medium
        'toppings': FieldState.PROVIDED,    # pepperoni, mushrooms
        'address': FieldState.PENDING,      # ← user was on this field
    },
    'current_field': 'address',
    'attempts': {
        'address': 0,  # hadn't tried yet
    },
}
```

This dict lives in Redis under `conversation:{id}:state:paused_item_snapshot` ([Day 4](/posts/conversation-state-in-redis/)). When the Q&A item signals done, the resume handler:

```python
async def resume_from_qa(conversation_id: str):
    snapshot = await redis.hGetAll(
        f'conversation:{conversation_id}:state:paused_item_snapshot'
    )
    item = reconstruct_gather_item(snapshot)

    # Generate a contextual re-ask, not a cold repeat
    reask = await generate_reask(item.current_field, context=snapshot)
    item.emit(MessageEvent(content=reask))
```

The "contextual re-ask" is the detail that sells it. Instead of "What's your delivery address?" (identical to the original), the LLM generates:

```
"Cool. So back to the order — pepperoni and mushroom medium. 
What's your delivery address?"
```

It acknowledges that a detour happened, summarizes what's already collected, and asks the still-pending question. To the user, the agenda feels like it has memory and awareness, not like a script that forgot it was talking.

## Real-World Examples

The Q&A item ends up being the most-triggered item in almost every agenda. Some real scenarios I've seen:

### Job application

```
🤖 Screener: Can you tell me about a time you handled conflict on a team?
👤 Alex: [long answer...] Actually what does this role pay?
🤖 Screener: Great story. For compensation — the base range is $140-180k plus 
            equity, and the posted JD has the full breakdown. Happy to answer
            anything else before I get to my next question?
👤 Alex: Is remote okay?
🤖 Screener: Yep, fully remote within US timezones. Good to continue?
👤 Alex: Yeah let's keep going
🤖 Screener: Back to where we were — after that conflict resolved, what would 
            you do differently now?
```

The screener retrieves from an HR knowledge base that the candidate can see. But notice — it doesn't answer salary by making up a number. It pulls from the knowledge base, which was uploaded by the hiring manager with the actual numbers.

### Appointment booking

```
🤖 Clinic: I have openings Tuesday at 10am, 2pm, or Thursday at 9am. Which works?
👤 Sam: How much will this visit cost?
🤖 Clinic: An initial consultation is $120 out-of-pocket. We accept most major 
          insurance — if you share your provider, I can check coverage.
👤 Sam: Blue Cross
🤖 Clinic: With Blue Cross PPO, initial consultations are typically covered 
          at 80% after the deductible. Exact cost depends on your plan.
          Ready to pick a time?
👤 Sam: Yes, Tuesday at 10am
🤖 Clinic: Tuesday at 10am — confirmed. I'll need your full name and phone number next.
```

Same pattern: mid-Gather question about pricing, answered from the clinic's fee schedule, then resumed at the field that was pending.

### Restaurant reservation

```
🤖 Host: Great, a table for 4 on Friday at 7pm. Can I get a name for the reservation?
👤 Mia: Can we bring our dog? He's trained.
🤖 Host: Our patio is dog-friendly on Fridays — we'd seat you outside. 
        For the reservation, what name should I put it under?
```

One-turn Q&A (she asked one question, got one answer), and the re-ask is embedded into the same message so the conversation feels continuous.

## Configuration

A Q&A item in an agenda config looks like this:

```yaml
type: gather
outputs:
  - name: size
    kind: asked
  - name: toppings
    kind: asked
  - name: delivery_address
    kind: asked
    validator: '\d+.*'  # must start with a number
  
  # Global Q&A behavior applied to every field
  on_user_question:
    enabled: true
    knowledge_scope: 'agenda:ordering'
    max_questions_per_interrupt: 5
    reask_style: 'contextual'  # not 'literal'
```

The `on_user_question` block is inherited by every field in the Gather. A Debate item has the same block. A Convey item doesn't need it (Convey doesn't wait for user input).

## What to Watch For

**Q&A loops**. Without `max_questions_per_interrupt`, a user can ask endless questions and never return to the flow. Cap it at 5 or so — after that the item forcibly resumes with "Let me know your answer to the current question, and we can pick up questions again after."

**Classifier false positives**. Early on, my classifier confused "medium, by the way what's the biggest pizza?" as an answer to "what size?" when the user was providing both. Fix: if the classifier finds both an answer fragment and a question fragment, handle the answer first, then fork to Q&A for the question.

**Stale pauses**. If a user stops responding mid-Q&A, the normal agenda timeout still fires. The Q&A item doesn't extend the TTL on the parent item's clock.

## What's Next

Q&A completes the four core item types — Convey, Gather, Debate, Q&A. The next posts cover how these items share **prompts** ([Day 9](/posts/multi-level-prompt-architecture/)), how they **branch** based on collected data ([Day 10](/posts/conditional-logic-branching-agenda-items/)), and how the **state machine** stitches it all together ([Day 11](/posts/state-machine-event-driven-agenda-transitions/)).

---

### Related in this series

- [Day 2 — How Q&A Mode Works: RAG](/posts/qa-mode-rag-knowledge-chat/) — the RAG pipeline the Q&A item reuses
- [Day 4 — Conversation State in Redis](/posts/conversation-state-in-redis/) — where the pause snapshot lives
- [Day 6 — The Gather Item](/posts/gather-item-structured-data-collection/) — what gets paused most often
- [Day 7 — The Debate Item](/posts/debate-item-ai-follow-up-questions/) — also pausable mid-round

### References

- [OpenAI function calling](https://platform.openai.com/docs/guides/function-calling) — the classifier and end-of-QA detection
- [Instructor (Python)](https://python.useinstructor.com/) — Pydantic-based structured outputs
