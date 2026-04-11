---
title: "How Q&A Mode Works: Building a Knowledge-Powered Chat with RAG"
date: 2026-04-02T11:00:00+07:00
draft: false
tags: ["AI Research", "Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "Before any structured agenda runs, users chat freely. Here's how we built the default Q&A mode with RAG — document ingestion, vector search, scoped knowledge, and the transition to Agenda mode."
ShowToc: true
weight: 2
seriesTotal: 12
---

{{< series-nav >}}

*Day 2 of 12. In [Day 1](/posts/agenda-engine-deterministic-ai-conversations/) we covered the Agenda Engine — the state machine that makes AI conversations deterministic. Today: the other mode — Q&A, where users lead.*

---

## Two Modes, One Conversation

Every conversation starts in **Q&A mode**. The user asks whatever they want. The AI answers from a knowledge base. No structure, no agenda, no rules beyond "be helpful."

This is the mode most chatbots live in permanently. For us, it's just the starting point. The moment a user says something like "I want to sign up" or "I'd like to apply," the system transitions to [Agenda mode](/posts/agenda-engine-deterministic-ai-conversations/) — and the state machine takes over.

But Q&A mode handles the majority of conversations. Most users come with questions before they're ready to commit to a structured flow. The quality of your Q&A mode determines whether they stick around long enough to trigger an agenda.

## The RAG Pipeline

Q&A mode is powered by **Retrieval-Augmented Generation (RAG)** — the AI doesn't answer from its training data alone. It retrieves relevant documents, then generates an answer grounded in those documents.

![RAG Pipeline — from document upload to answer generation](/images/blog/rag-pipeline.svg)

The pipeline has two phases: **ingestion** (happens once per document) and **retrieval** (happens every message).

### Phase 1: Document Ingestion

When an admin uploads a knowledge source, it goes through an async pipeline:

```python
# Ingestion pipeline (simplified)
async def ingest_document(source: DocumentSource):
    # 1. Extract raw text based on source type
    if source.type == "pdf":
        text = await extract_pdf(source.url)
    elif source.type == "webpage":
        text = await crawl_webpage(source.url)
    elif source.type == "google_doc":
        text = await fetch_google_doc(source.id)
    # ... CSV, YouTube transcript, spreadsheet, etc.

    # 2. Split into chunks
    chunks = split_into_chunks(
        text,
        max_tokens=512,
        overlap=50,       # Overlap prevents losing context at boundaries
        strategy="paragraph"  # Respect paragraph boundaries when possible
    )

    # 3. Generate embeddings
    embeddings = await openai.embeddings.create(
        model="text-embedding-3-small",
        input=[chunk.text for chunk in chunks]
    )

    # 4. Store in vector database with metadata
    await vector_db.upsert(
        vectors=[
            {
                "id": chunk.id,
                "values": embedding.values,
                "metadata": {
                    "agent_id": source.agent_id,
                    "document_id": source.document_id,
                    "text": chunk.text,
                    "source_type": source.type,
                    "language": detect_language(chunk.text),
                    "scope": source.scope,  # "all" | specific agenda | specific channel
                }
            }
            for chunk, embedding in zip(chunks, embeddings.data)
        ]
    )
```

A few things worth noting:

**Async processing.** Ingestion runs on a background queue, not in the request path. A PDF with 100 pages takes time to chunk and embed — users shouldn't wait for it.

**Chunk overlap.** We use 50-token overlap between chunks. Without this, a question that spans two paragraphs might not match either chunk well enough.

**Metadata.** Every chunk carries its agent ID, document ID, source type, detected language, and scope. This metadata is critical for retrieval filtering.

### Supported Sources

The platform ingests from anything an enterprise team actually uses:

| Source | How It's Ingested |
|--------|------------------|
| PDFs | Text extraction, page-aware chunking |
| Webpages | Crawled, HTML stripped, auto-sync on schedule |
| Google Docs | API fetch, auto-sync when doc changes |
| Google Sheets | Row-by-row, each row becomes a retrievable chunk |
| YouTube | Transcript extraction, timestamped chunks |
| CSV / Excel | Row-based or column-based chunking |
| Plain text | Direct chunking |

**Auto-sync** is important for enterprise. When a support team updates a Google Doc with new pricing, the knowledge base should reflect it without manual re-upload. We poll live sources on a configurable schedule and re-embed changed content.

### Phase 2: Retrieval

When a user sends a message in Q&A mode, the retrieval pipeline runs:

```python
async def retrieve_context(
    query: str,
    agent_id: int,
    channel: str = None,
    agenda_id: int = None,
    language: str = "en",
) -> list[Document]:
    # 1. Rewrite the query for better retrieval
    search_query = await rewrite_query(query)

    # 2. Embed the query
    query_embedding = await openai.embeddings.create(
        model="text-embedding-3-small",
        input=search_query
    )

    # 3. Search vector DB with metadata filters
    results = await vector_db.query(
        vector=query_embedding.data[0].values,
        top_k=10,
        filter={
            "agent_id": agent_id,
            # Scope: only documents available for this context
            "scope": {"$in": ["all", channel, agenda_id]},
        }
    )

    # 4. Score and rank with language boost
    ranked = score_results(
        results,
        query_language=language,
        boost_same_language=1.3,  # Prefer same-language matches
    )

    return ranked[:5]  # Top 5 most relevant chunks
```

### Query Rewriting

Raw user messages often make poor search queries. "Yeah that sounds expensive" doesn't retrieve pricing docs well. The query rewriter turns conversational text into search-optimized queries:

```python
async def rewrite_query(user_message: str) -> str:
    """Turn conversational text into a search query."""
    response = await llm.chat(
        system_prompt="""Rewrite the user's message as a search query.
        Keep it concise. Focus on the information need.
        If the message is already a clear question, keep it as-is.""",
        user_prompt=user_message,
    )
    return response.content
```

"Yeah that sounds expensive" becomes "pricing plans cost" — a much better vector search query.

### Language-Specific Scoring

The platform serves clients across 40+ languages. A Vietnamese user asking a question in Vietnamese should get Vietnamese documents ranked higher, even if the English version is a slightly closer vector match.

We apply a 1.3x boost to documents matching the query language. This sounds simple, but it was the difference between "sometimes returns wrong-language results" and "consistently returns the right language."

## Knowledge Scoping

Not all knowledge should be available everywhere. A job application agenda shouldn't surface restaurant menu items. A WhatsApp channel for support shouldn't return internal HR documents.

Every document is scoped:

```python
class DocumentScope:
    ALL = "all"              # Available everywhere
    CHANNEL = "channel"      # Only on specific channels (web, WhatsApp, etc.)
    AGENDA = "agenda"        # Only during specific agendas
```

When an admin uploads a document, they choose its scope. During retrieval, the vector search filters by scope — only returning documents available in the current context.

This means the same AI agent can have completely different knowledge depending on whether the user is:
- Chatting on the website (full knowledge)
- Messaging on WhatsApp (support-only knowledge)
- In a screening agenda (role-specific knowledge)

## Gap Detection

What happens when the knowledge base can't answer a question? Most chatbots either hallucinate or say "I don't know." We added a third option: **alert the admin.**

```python
async def detect_missing_context(
    query: str,
    retrieved_docs: list[Document],
    response: str,
) -> bool:
    """Check if the AI had enough context to answer well."""
    result = await llm.parse(
        system_prompt="""Evaluate if the retrieved documents contain
        enough information to answer the user's question accurately.
        Return false if the AI would need to guess or generalize.""",
        user_prompt=f"Question: {query}\nContext: {retrieved_docs}\nAnswer: {response}",
        response_model=MissingContextCheck,  # Pydantic model
    )

    if result.is_missing:
        await notify_admin(
            agent_id=agent_id,
            question=query,
            reason=result.reason,
        )
    return result.is_missing
```

The admin gets a notification: "A user asked about refund policies, but we don't have documentation for this." Over time, these alerts tell admins exactly which documents to add — instead of guessing what users might ask about.

In production, this caught knowledge gaps we never anticipated. Users ask questions the team never thought of, and the gap detector surfaces those blind spots.

## The Transition to Agenda Mode

Q&A mode is the default, but it's not the destination. The real value is in structured agendas. The transition happens through **trigger detection**:

![How Q&A mode transitions to Agenda mode via trigger detection](/images/blog/qa-to-agenda-transition.svg)

While the user chats in Q&A mode, every message is checked against agenda triggers. The AI uses tool calling to detect matches:

```python
# Tool definition for agenda trigger detection
select_agenda_tool = Tool(
    name="select_agenda",
    description=f"""Detect if the user wants to start a structured workflow.
    Available triggers:
    {format_trigger_list(agent.agendas)}""",
    parameters={
        "agenda_id": {"type": "integer", "description": "Matched agenda ID"},
        "confidence": {"type": "number", "description": "Match confidence 0-1"},
    }
)
```

The trigger list might look like:

```
- "Apply for a role" → Candidate Screening agenda
- "I want to sign up" → Onboarding agenda
- "Book a demo" → Demo Booking agenda
```

When the LLM detects a match with high confidence, the system transitions:

1. **Confirm** — "Would you like to start the screening process?" (unless auto-start is enabled)
2. **Pre-populate** — Scan the last 20 Q&A messages for data the agenda needs (name, email, etc.)
3. **Start** — Begin executing agenda items in sequence

The pre-populate step is important. If the user already said "I'm Jane, I'm interested in the senior role" during Q&A, the Gather item shouldn't ask for their name again. The system scans recent history and auto-fills known outputs.

### Auto-Start

Some channels skip the confirmation step. On WhatsApp, where every conversation has a clear purpose, you might configure the agenda to auto-start — the user sends their first message and the screening begins immediately, no "Would you like to start?" prompt needed.

```python
class AgendaConfig:
    trigger_phrase: str
    auto_start: bool = False          # Skip confirmation
    auto_start_channels: list = []    # Only auto-start on specific channels
```

## Putting It All Together

Here's the full Q&A flow for a single user message:

```
User: "Do you offer refunds?"
  │
  ├── 1. Trigger detection → no agenda match
  │
  ├── 2. Query rewrite → "refund policy"
  │
  ├── 3. Vector search → retrieve top 5 chunks
  │     └── Filter: agent_id + channel scope
  │
  ├── 4. Language scoring → boost Vietnamese docs for Vietnamese query
  │
  ├── 5. LLM generation → answer with retrieved context
  │
  ├── 6. Gap detection → context sufficient? ✓
  │
  └── 7. Stream response → "Yes, we offer a 30-day refund policy..."

User: "Great, I want to sign up"
  │
  ├── 1. Trigger detection → matches "Onboarding" agenda ✓
  │
  ├── 2. Pre-populate → scan last 20 messages for known data
  │
  ├── 3. Confirm → "I'd be happy to help you sign up. Shall we begin?"
  │
  └── 4. Start agenda → transition to IN_AGENDA mode
```

Q&A mode is the front door. RAG makes it useful. Trigger detection makes it a gateway to structured conversations. Together, they handle the full lifecycle: from "I have a question" to "I'm ready to commit."

---

*Next in series: [Day 3 — Streaming AI Responses: From LLM to Browser in Chunks](/posts/streaming-ai-responses-llm-to-browser)*

*Previous: [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/)*
