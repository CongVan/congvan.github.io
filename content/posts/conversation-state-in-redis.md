---
title: "Conversation State in Redis: Hashes, Streams, and a Single Source of Truth"
date: 2026-04-11T11:00:00+07:00
lastmod: 2026-04-11T11:00:00+07:00
draft: false
tags: ["Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "Chat text lives in Redis Streams. But what about the state — which agenda is running, which item is active, what data got collected? That lives in Redis too, in different data structures. Here's the full key layout, how events update state atomically, and why keeping everything in one store simplifies the AI flow."
ShowToc: true
weight: 4
seriesTotal: 12
---

{{< series-nav >}}

*Day 4 of 12. [Day 3](/posts/streaming-ai-responses-llm-to-redis/) parked chat text in Redis Streams. Today I put the **state** in Redis too — which agenda is running, which item is active, what data got collected. Same Redis instance, different data structures.*

---

## The State Problem

A conversation isn't just text. Every time a user sends a new message, the backend needs to answer a few questions before it can do anything:

- Which agenda is running right now? Or is the conversation still in Q&A mode?
- Which item is the AI currently executing?
- What data has been collected so far in this agenda?
- Is the conversation paused, timed out, or completed?

None of this lives in the chat transcript. Reconstructing it from messages alone is slow and error-prone. I need a single source of truth the backend can read in under 10ms, every single message, across every active conversation.

The obvious options are Postgres, Firebase, or Redis. I picked Redis. Here's why.

## Why Redis for State (Not Postgres, Not Firebase)

**Postgres** is the durable record — it's where completed agendas live forever. But it's too slow for per-message reads at load. Every message becomes 2-3 joins plus connection pooling overhead. Queries that take 50ms at 10 requests/sec turn into 500ms at 1000 requests/sec.

**Firebase Firestore** is great for frontend sync — any client can subscribe to a doc and get live updates. But it adds a second durable system I'd have to keep consistent with Postgres, and the per-read cost model doesn't fit a flow that reads on every single message.

**Redis** is already in the stack from [Day 3](/posts/streaming-ai-responses-llm-to-redis/). One store, multiple data structures, sub-millisecond reads on the hot path. Postgres stays in the picture — it's just not on the critical path anymore. When an agenda completes, the final state flushes to Postgres as one row.

## The Key Layout

Every conversation gets its own set of Redis keys, all prefixed with `conversation:{id}:`. Each key uses a different data structure picked for what that piece of state actually needs.

![Redis key layout per conversation](/images/blog/redis-key-layout.svg)

```
conversation:{id}:stream           → Stream       (chat chunks + events from Day 3)
conversation:{id}:state            → Hash         (mode, active agenda, current item)
conversation:{id}:agenda:outputs   → Hash         (collected data: name, email, score...)
conversation:{id}:agenda:history   → List         (completed item IDs with timestamps)
conversation:{id}:lock             → String + TTL (prevents concurrent processing)
```

A shared 24h TTL on every key keeps Redis memory bounded. Redis is not the durable store — it's the working set. If a conversation goes idle for 24 hours, everything expires automatically and the Postgres record is still there if the user comes back.

## The state Hash: Current Position Snapshot

The `state` key answers "what's happening in this conversation right now." It's a [Redis Hash](https://redis.io/docs/latest/develop/data-types/hashes/) with a handful of fields.

```typescript
// On agenda start
await redis.hSet(`conversation:${id}:state`, {
  mode: 'in_agenda',
  agendaId: 'founder-screening',
  currentItemId: 'gather-basics',
  status: 'running',
  startedAt: String(Date.now()),
})

// On every user message, the backend reads the full hash in one call
const state = await redis.hGetAll(`conversation:${id}:state`)
if (state.mode === 'in_agenda') {
  // pass current item context into the AI service request
}
```

Why a hash instead of a JSON blob in a plain string? Two reasons:

1. **Partial updates without read-modify-write.** `HSET` updates one field without touching the others. A plain JSON string would force a `GET`, parse, modify, `SET` cycle — three round trips instead of one.
2. **Per-field atomics.** `HINCRBY` atomically increments a counter field, which I use for things like message count without risking lost updates.

## The outputs Hash: What Got Collected

Collected data (the "outputs" from Gather and Debate items — details in [Day 6](/posts/gather-item-structured-data-collection/) and [Day 7](/posts/debate-item-ai-follow-up-questions/)) needs the same partial-update behavior, so it's also a hash. One field per output, JSON-encoded so types survive the round trip.

```typescript
// When the AI service emits the end-of-item event with collected data
for (const [field, value] of Object.entries(event.outputs)) {
  await redis.hSet(
    `conversation:${id}:agenda:outputs`,
    field,
    JSON.stringify(value),  // preserves numbers, booleans, nested objects
  )
}

// Read back in bulk for the next item or the completion webhook
const raw = await redis.hGetAll(`conversation:${id}:agenda:outputs`)
const outputs = Object.fromEntries(
  Object.entries(raw).map(([k, v]) => [k, JSON.parse(v)])
)
```

Each output is JSON-encoded so the hash can hold mixed types — strings, numbers, booleans, small objects — without losing them to Redis's string-only hash values.

## The history List: Audit Trail

A [Redis List](https://redis.io/docs/latest/develop/data-types/lists/) is the right shape for an append-only sequence of completed items. Each entry captures which item the user went through and when.

```typescript
// Append each completed item to the history list
await redis.rPush(
  `conversation:${id}:agenda:history`,
  JSON.stringify({ itemId: event.itemId, completedAt: Date.now() }),
)

// Read the whole history in order
const raw = await redis.lRange(`conversation:${id}:agenda:history`, 0, -1)
const history = raw.map(entry => JSON.parse(entry))
```

The history is used for two things: showing "which steps the user has been through" on the admin dashboard, and resuming an agenda if the user exits and comes back later.

## The lock Key: Preventing Concurrent Processing

On messaging channels, users sometimes fire 3 messages in 500ms. Without a lock, each message spawns its own AI call, all racing on the same state. I use a single-instance Redis lock: `SET NX` with an expiry.

```typescript
const acquired = await redis.set(
  `conversation:${id}:lock`,
  processId,
  { NX: true, EX: 30 },  // 30s TTL — auto-release if holder crashes
)

if (!acquired) {
  // Another worker is already processing this conversation.
  // Debounce and retry, or queue this message for later.
  return { deferred: true }
}

try {
  await processMessage(...)
} finally {
  await redis.del(`conversation:${id}:lock`)
}
```

This is the [Redlock pattern](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) — fine for a single Redis instance. A multi-instance Redis setup would need the full Redlock algorithm.

Multi-message grouping on WhatsApp and Messenger (where users legitimately send 3 messages in a row and the system groups them into one reply) is handled upstream in the channel adapter — more on that in [Day 12](/posts/four-channels-one-engine/).

## One Event, Multiple Atomic Writes

Every event from the AI service's gRPC stream (see [Day 3](/posts/streaming-ai-responses-llm-to-redis/)) triggers multiple Redis writes. They have to land atomically — if the chat stream sees the event but the state doesn't update, the next message will resume from the wrong place.

The fix is [Redis transactions](https://redis.io/docs/latest/develop/interact/transactions/): wrap every fan-out in MULTI/EXEC.

![One event fans out to multiple Redis writes, atomic via MULTI/EXEC](/images/blog/redis-event-to-state.svg)

```typescript
for await (const event of aiService.chat(payload)) {
  // Open a transaction for this event's fan-out
  const tx = redis.multi()

  // Every event goes to the chat stream (from Day 3)
  tx.xAdd(`conversation:${id}:stream`, '*', {
    event: JSON.stringify(event),
  })

  // State updates depend on event type
  switch (event.case) {
    case 'startAgenda':
      tx.hSet(`conversation:${id}:state`, {
        mode: 'in_agenda',
        agendaId: event.agendaId,
        currentItemId: event.firstItemId,
        status: 'running',
      })
      break

    case 'startAgendaItem':
      tx.hSet(
        `conversation:${id}:state`,
        'currentItemId',
        event.itemId,
      )
      break

    case 'endAgendaItem':
      // Save every collected output
      for (const [k, v] of Object.entries(event.outputs)) {
        tx.hSet(
          `conversation:${id}:agenda:outputs`,
          k,
          JSON.stringify(v),
        )
      }
      // Append to history
      tx.rPush(
        `conversation:${id}:agenda:history`,
        JSON.stringify({ itemId: event.itemId, at: Date.now() }),
      )
      break

    case 'endAgenda':
      tx.hSet(`conversation:${id}:state`, {
        mode: 'qa',
        status: 'completed',
      })
      break

    case 'error':
      tx.hSet(`conversation:${id}:state`, 'status', 'error')
      break
  }

  // Commit all writes atomically
  await tx.exec()
}

// On completion, flush to Postgres as the durable record
if (agendaCompleted) {
  await persistToPostgres(id)
}
```

The whole loop fans out every gRPC event to the right Redis keys in a single transaction per event. No second pass. No risk of partial state updates.

## The Full Message Cycle

Putting it all together, here's what happens on every user message:

![One message cycle: read state, call AI, write events](/images/blog/redis-read-write-cycle.svg)

1. **User message arrives** at the backend via HTTP or webhook
2. **Backend reads Redis state** — two `HGETALL` calls (`:state` and `:agenda:outputs`), about 1ms on LAN
3. **Backend calls the AI service** over gRPC, passing the current state as context
4. **AI service streams events** back (from [Day 3](/posts/streaming-ai-responses-llm-to-redis/))
5. **Backend fans out each event** into `XADD` + `HSET` writes, atomically per event
6. **Loop** until the stream emits `endAgenda` or `error`
7. **On completion**, flush the final state to Postgres as a durable row

Total Redis cost per message: ~5-15 round trips, all sub-millisecond. The LLM call itself is the dominant latency — state reads are noise.

## Flushing to Postgres

Redis is transient. Postgres is the system of record. The flush happens on three events:

- **`endAgenda`** — normal completion, write final state + outputs + scoring to `conversation_agendas` table
- **`exitAgenda`** — user exited early, write partial state with `status = exited`
- **`errorAgenda`** — unrecoverable error, write partial state with `status = error`

A separate cron job sweeps Redis every hour for agendas older than 1 hour with no activity and flushes them as `abandoned`. Without that, users who never return would leave agenda state in Redis until the 24h TTL expires.

All Postgres writes are idempotent — an upsert on `conversation_id + agenda_id`. If the flush fails mid-write, the cron picks it up next hour.

## Failure Modes I Had to Think About

**Redis crashes mid-conversation.** State is lost for any agenda that hasn't flushed to Postgres yet. Mitigation: [Redis AOF](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) (append-only file) with `fsync everysec`. The worst case is losing ~1 second of writes, which usually means one in-flight event. Acceptable tradeoff.

**Backend crashes mid-event.** Without MULTI/EXEC, some writes for one event could land and others not. With it, either all or none land. If "none" is the outcome, the state is one event behind the chat stream — but the next message will re-read state and the AI will recover.

**Postgres flush fails on agenda completion.** Redis still has the state. The cron job retries the flush on every sweep until it succeeds. Idempotent upserts make retries safe.

**A slow consumer holds the lock.** The 30s TTL auto-releases it. Worst case: a second worker starts processing the same conversation after 30s of the first worker being stuck. In practice, 30s is long enough that the first worker has definitely crashed (LLM calls take 1-5s).

## Closing

That's the full AI flow, end to end:

1. User message arrives
2. Backend reads state from Redis (2 reads)
3. Backend calls AI service via gRPC
4. AI service streams events back
5. Backend fans out each event atomically to Redis (chat stream + state hashes)
6. On completion, final state flushes to Postgres

No Firebase. No WebSockets. One Redis instance, four data structures, one transaction per event.

Now that the plumbing is covered (streaming in Day 3, state in Day 4), the next posts dig into the agenda items themselves — the actual building blocks that populate all this state. Next: [Day 5 — The Convey Item](/posts/convey-item-scripted-dynamic-messages/), the simplest item type.

---

### Related in this series

- [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — what the state is *representing*
- [Day 3 — Streaming AI Responses to Redis Streams](/posts/streaming-ai-responses-llm-to-redis/) — the other half of the Redis data flow

### References

- [Redis Hashes](https://redis.io/docs/latest/develop/data-types/hashes/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Redis Lists](https://redis.io/docs/latest/develop/data-types/lists/)
- [Redis MULTI/EXEC transactions](https://redis.io/docs/latest/develop/interact/transactions/)
- [Redis distributed locks (Redlock)](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis persistence (AOF)](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis key expiration](https://redis.io/docs/latest/commands/expire/)
