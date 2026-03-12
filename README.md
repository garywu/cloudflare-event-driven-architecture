# Event-Driven Architecture on Cloudflare Workers

```
                        ┌─────────────────────┐
                        │    Brand Agent       │
                        │  (Durable Object)    │
                        │                      │
                        │  state + strategy    │
                        │  decides WHAT + WHY  │
                        └──┬──────────────┬────┘
                           │              │
                    command │              │ command
                           ▼              ▼
              ┌────────────────┐  ┌────────────────┐
              │  RESEARCH_QUEUE│  │  PUBLISH_QUEUE  │
              └───────┬────────┘  └───────┬────────┘
                      │                   │
                      ▼                   ▼
              ┌──────────────┐    ┌──────────────┐
              │  GatherFeed  │    │  Pages-plus  │
              │  (research)  │    │  (publisher) │
              └──────┬───────┘    └──────┬───────┘
                     │                   │
               event │                   │ event
                     ▼                   ▼
              ┌────────────────────────────────┐
              │         EVENTS_QUEUE            │
              │  (fan-out consumer routes to    │
              │   all interested services)      │
              └────────────────────────────────┘
```

Stop calling `fetch()` between your Workers. Use Queues.

This is a reference architecture for building event-driven systems on Cloudflare's developer platform — Workers, Queues, Durable Objects, and the Agents SDK. It covers the patterns that work, the anti-patterns that don't, and the Cloudflare-specific constraints you need to design around.

---

## Table of Contents

- [The Problem with Imperative Architectures](#the-problem-with-imperative-architectures)
- [Two Patterns, Not One](#two-patterns-not-one)
- [Cloudflare Primitives](#cloudflare-primitives)
- [The Message Envelope](#the-message-envelope)
- [Queue Topology](#queue-topology)
- [Fan-Out Pattern](#fan-out-pattern)
- [Idempotency](#idempotency)
- [The Outbox Pattern](#the-outbox-pattern)
- [Consumer Middleware](#consumer-middleware)
- [Agents as Event Reactors](#agents-as-event-reactors)
- [Durable Workflows](#durable-workflows)
- [The Event Catalog](#the-event-catalog)
- [Small Patterns That Add Up](#small-patterns-that-add-up)
- [What Not to Do](#what-not-to-do)
- [What You Don't Need](#what-you-dont-need)
- [Cloudflare Constraints](#cloudflare-constraints)
- [Full Example: Content Pipeline](#full-example-content-pipeline)
- [References](#references)

---

## The Problem with Imperative Architectures

This is how most people wire up Cloudflare Workers:

```typescript
// Service A calls Service B, waits, then calls Service C
const research = await fetch("https://gatherfeed.workers.dev/api/v1/research", {
  method: "POST",
  body: JSON.stringify({ keyword: "best budgeting apps" }),
});
const data = await research.json();

const article = await fetch("https://content-engine.workers.dev/v1/generate", {
  method: "POST",
  body: JSON.stringify({ research: data }),
});
const content = await article.json();

await fetch("https://publisher.workers.dev/v1/publish", {
  method: "POST",
  body: JSON.stringify({ content }),
});
```

This is synchronous, imperative, and fragile. Every problem with microservices shows up:

- **Temporal coupling.** If GatherFeed is slow, the entire chain blocks. If it's down, everything fails.
- **Tight coupling.** Service A knows the URL, method, and payload shape of every downstream service.
- **No retry isolation.** A failure in step 3 means re-running steps 1 and 2.
- **No partial progress.** If the Worker hits the CPU time limit mid-chain, all work is lost.
- **Cost amplification.** One slow downstream service holds your Worker awake (and billable) while it waits.

The fix isn't better error handling. It's a different architecture.

---

## Two Patterns, Not One

People conflate "event-driven" and "message-driven." They're different, and you usually want both.

### Message-Driven (Commands)

A producer sends a message **to a specific consumer**. The producer knows who's receiving it.

```
"Hey GatherFeed, research these keywords."
```

This is a **command**. It's directed. It tells a service what to do. The producer is coupled to the consumer — it knows the destination queue exists and what the consumer expects.

### Event-Driven (Facts)

A producer emits a fact about **something that happened**. The producer doesn't know or care who's listening.

```
"Research completed for brand X. 50 keywords ready."
```

This is an **event**. It's broadcast. It describes what happened, not what should happen next. Consumers subscribe. The producer is decoupled — add or remove consumers without touching the producer.

### When to Use Which

| Pattern | Use When | Cloudflare Primitive |
|---------|----------|---------------------|
| Command (message) | You need a specific service to do a specific thing | Queue with dedicated consumer |
| Event (fact) | Multiple services might care about what happened | Shared events queue with fan-out consumer |

In practice, the flow is: **commands trigger work → work produces events → events trigger decisions → decisions produce commands**. That's the loop.

---

## Cloudflare Primitives

Cloudflare provides four building blocks for event-driven systems. Each has a specific role:

### Cloudflare Queues

The message backbone between Workers. Producer Workers write messages; consumer Workers process them.

| Property | Value |
|----------|-------|
| Delivery guarantee | At-least-once |
| Ordering | No guarantee |
| Max message size | 128 KB |
| Throughput | 5,000 messages/sec per queue |
| Consumer concurrency | Up to 250 auto-scaling consumers |
| Max retry | Configurable (default 3) |
| Dead letter queue | Supported |
| Message delay | 0–86,400 seconds (24 hours) |

The critical constraint: **one consumer Worker per queue**. This isn't a limitation if you design for it — it actually simplifies reasoning about message ownership.

### Cloudflare Agents SDK

Persistent, stateful agents built on Durable Objects. Each agent has:

- **SQLite database** (`this.sql`) — queryable, durable storage
- **Key-value state** (`this.state`) — syncs to connected WebSocket clients in real-time
- **Scheduling** (`this.schedule()`) — cron expressions or one-time delays
- **Internal queue** (`this.queue()`) — sequential background task processing
- **Workflow integration** (`this.runWorkflow()`) — trigger durable multi-step processes

Agents are the decision-makers. They react to events, maintain strategy, and emit commands.

### Cloudflare Workflows (AgentWorkflow)

Durable multi-step execution integrated with Agents:

- Steps are checkpointed — a crash resumes from the last completed step
- Each step retries independently with configurable backoff
- Can wait for external events for up to **one year** (`waitForApproval`)
- Steps can update agent state, broadcast to WebSocket clients, call agent methods via RPC

Workflows are for processes that **must complete**: research → generate → publish. Not for fire-and-forget.

### Event Subscriptions

Native Cloudflare service events (R2, KV, Workers AI, Workflows) published to Queues automatically:

```bash
npx wrangler queues subscription create my-queue --source r2 --events bucket.created
```

Events follow a standard structure:

```json
{
  "type": "cf.r2.bucket.created",
  "source": { "type": "r2" },
  "payload": { "name": "my-bucket", "location": "WNAM" },
  "metadata": { "accountId": "...", "eventTimestamp": "2026-03-11T10:00:00Z" }
}
```

This is true event-driven — the emitter doesn't know who's subscribed.

---

## The Message Envelope

Every message through any queue follows this shape:

```typescript
interface DomainMessage<T = unknown> {
  event_id: string;        // UUID v4 — deduplication key
  type: string;            // dot-notation: "research.requested", "content.published"
  source: string;          // emitting service: "scalable-media", "gatherfeed"
  timestamp: string;       // ISO 8601
  correlation_id?: string; // traces a chain of related messages
  payload: T;              // reference data — IDs, not objects
}
```

Rules:

1. **`event_id` is mandatory.** At-least-once delivery means duplicates happen. This is your deduplication key.
2. **`type` uses dot-notation.** Domain-first: `research.requested`, not `requested_research`. The domain is the noun, the action is the verb.
3. **Payloads carry references, not data.** `{ research_id: "abc" }`, not `{ full_research_object: {...} }`. The data lives in D1 or R2. Messages are signals, databases are state.
4. **`correlation_id` traces causality.** A brand cycle generates research commands, which produce completion events, which trigger generation commands. The correlation ID ties them together for debugging.

---

## Queue Topology

Each service owns its **inbound command queue**. There is one **shared events queue** with a fan-out consumer.

```
                     COMMAND QUEUES (directed, one consumer each)
                     ┌─────────────────────────────────────────┐
                     │                                         │
  ┌──────────────────┤  gatherfeed-commands  → GatherFeed      │
  │                  │  sm-commands          → Scalable Media   │
  │  Producers       │  publish-commands     → Pages-plus       │
  │  (any service)   │  social-commands      → Social-good      │
  │                  │                                         │
  │                  ├─────────────────────────────────────────┤
  │                  │                                         │
  │                  │  EVENT QUEUE (broadcast, fan-out)        │
  └──────────────────┤  brand-events  → Fan-out consumer       │
                     │                  → routes to all         │
                     │                    interested queues     │
                     └─────────────────────────────────────────┘
```

### Wrangler Configuration

Producer side (e.g., Scalable Media):

```jsonc
{
  "queues": {
    "producers": [
      { "queue": "gatherfeed-commands", "binding": "RESEARCH_QUEUE" },
      { "queue": "publish-commands", "binding": "PUBLISH_QUEUE" },
      { "queue": "brand-events", "binding": "EVENTS_QUEUE" }
    ],
    "consumers": [
      {
        "queue": "sm-commands",
        "max_batch_size": 10,
        "max_batch_timeout": 5,
        "dead_letter_queue": "sm-commands-dlq"
      }
    ]
  }
}
```

Consumer side (e.g., GatherFeed):

```jsonc
{
  "queues": {
    "producers": [
      { "queue": "brand-events", "binding": "EVENTS_QUEUE" }
    ],
    "consumers": [
      {
        "queue": "gatherfeed-commands",
        "max_batch_size": 5,
        "max_batch_timeout": 10,
        "dead_letter_queue": "gatherfeed-commands-dlq"
      }
    ]
  }
}
```

Dead letter queues are mandatory. A message that fails `max_retries` times goes to the DLQ instead of being silently dropped. Monitor the DLQ — it's your system telling you something is broken.

---

## Fan-Out Pattern

Cloudflare Queues supports one consumer per queue. To deliver the same event to multiple services, use a **fan-out consumer** — a Worker that reads from the shared events queue and re-publishes to destination queues:

```typescript
// fan-out-consumer/src/index.ts
export default {
  async queue(batch: MessageBatch<DomainMessage>, env: Env): Promise<void> {
    for (const msg of batch.messages) {
      const routes = getRoutes(msg.body.type);

      await Promise.all(
        routes.map((queue) => queue.send(msg.body))
      );

      msg.ack();
    }
  },
};

function getRoutes(eventType: string): Queue[] {
  // Route table — add new subscribers here, not in the producer
  const routes: Record<string, Queue[]> = {
    "research.completed": [env.SM_COMMANDS, env.ANALYTICS_QUEUE],
    "content.published":  [env.SM_COMMANDS, env.SOCIAL_COMMANDS],
    "content.performed":  [env.SM_COMMANDS],
  };
  return routes[eventType] ?? [];
}
```

The routing logic is JavaScript, not config. Add a subscriber by adding a line to the route table. Remove one by deleting the line. No producer changes needed.

This is the key advantage over static topic-based routing (Kafka, RabbitMQ): the routing is programmable. You can route based on payload fields, time of day, feature flags — anything.

---

## Idempotency

Cloudflare Queues delivers **at-least-once**. This means your consumer **will** see duplicate messages. Not might. Will.

Every queue consumer must be safe to run twice with the same input. There are three strategies:

### Strategy 1: Processed Events Table

The general-purpose approach. Check before acting:

```typescript
async queue(batch: MessageBatch<DomainMessage>, env: Env): Promise<void> {
  for (const msg of batch.messages) {
    const { event_id } = msg.body;

    // Already processed?
    const existing = await env.DB.prepare(
      "SELECT 1 FROM processed_events WHERE event_id = ?"
    ).bind(event_id).first();

    if (existing) {
      msg.ack();
      continue;
    }

    try {
      await handleMessage(msg.body, env);

      await env.DB.prepare(
        "INSERT INTO processed_events (event_id, type, processed_at) VALUES (?, ?, ?)"
      ).bind(event_id, msg.body.type, new Date().toISOString()).run();

      msg.ack();
    } catch (err) {
      msg.retry({ delaySeconds: 30 });
    }
  }
}
```

### Strategy 2: Natural Business Key

For writes where a natural key exists:

```sql
-- If we already have research for this keyword, skip
INSERT OR IGNORE INTO research (keyword, brand_slug, data, created_at)
VALUES (?, ?, ?, ?);
```

The `INSERT OR IGNORE` (or `ON CONFLICT DO NOTHING`) makes the write idempotent at the database level. No separate tracking table needed.

### Strategy 3: Upstream Deduplication

For external API calls, check if the result already exists before calling:

```typescript
async function researchKeyword(keyword: string, env: Env): Promise<void> {
  // Already have fresh research?
  const existing = await env.DB.prepare(
    "SELECT 1 FROM research WHERE keyword = ? AND created_at > datetime('now', '-7 days')"
  ).bind(keyword).first();

  if (existing) return; // Skip — don't burn an API call

  const result = await callPerplexityViaApiMom(keyword, env);
  await env.DB.prepare("INSERT INTO research ...").bind(...).run();
}
```

This saves money (no redundant API calls) and is naturally idempotent.

### Pruning

Processed event records don't need to live forever. Prune after 7 days:

```sql
DELETE FROM processed_events WHERE processed_at < datetime('now', '-7 days');
```

Run this on a cron schedule or inside a scheduled Agent task.

---

## The Outbox Pattern

There's a subtle failure mode: your D1 write succeeds, but the Queue publish fails (network issue, Worker CPU limit). Now your database says "research complete" but no event was emitted. Downstream services never find out.

The fix is the **outbox pattern**: write the event to an `outbox` table in the **same D1 transaction** as your business data. A separate process polls the outbox and publishes to Queues.

```typescript
// In your queue consumer — write data AND outbox event in one transaction
await env.DB.batch([
  env.DB.prepare(
    "INSERT INTO research (id, keyword, data) VALUES (?, ?, ?)"
  ).bind(id, keyword, JSON.stringify(data)),

  env.DB.prepare(
    "INSERT INTO outbox (event_id, type, payload, created_at) VALUES (?, ?, ?, ?)"
  ).bind(
    crypto.randomUUID(),
    "research.completed",
    JSON.stringify({ brand_slug, research_ids: [id] }),
    new Date().toISOString()
  ),
]);
```

A scheduled task (cron or Agent schedule) publishes outbox events:

```typescript
async function flushOutbox(env: Env): Promise<void> {
  const pending = await env.DB.prepare(
    "SELECT * FROM outbox WHERE published_at IS NULL ORDER BY created_at LIMIT 50"
  ).all();

  for (const row of pending.results) {
    await env.EVENTS_QUEUE.send({
      event_id: row.event_id,
      type: row.type,
      source: "gatherfeed",
      timestamp: row.created_at,
      payload: JSON.parse(row.payload),
    });

    await env.DB.prepare(
      "UPDATE outbox SET published_at = ? WHERE event_id = ?"
    ).bind(new Date().toISOString(), row.event_id).run();
  }
}
```

This guarantees that if the data was written, the event **will** eventually be published. The consumer's idempotency handling covers the case where the event publishes twice (outbox flushed, but the `published_at` update failed).

---

## Consumer Middleware

Borrowed from [Watermill](https://github.com/ThreeDotsLabs/watermill)'s middleware pattern. Every queue handler should be wrapped in composable middleware:

```typescript
type MessageHandler = (msg: DomainMessage, env: Env) => Promise<void>;
type Middleware = (next: MessageHandler) => MessageHandler;

// Deduplication middleware
function withDedup(db: D1Database): Middleware {
  return (next) => async (msg, env) => {
    const exists = await db.prepare(
      "SELECT 1 FROM processed_events WHERE event_id = ?"
    ).bind(msg.event_id).first();
    if (exists) return;

    await next(msg, env);

    await db.prepare(
      "INSERT INTO processed_events (event_id, type, processed_at) VALUES (?, ?, ?)"
    ).bind(msg.event_id, msg.body.type, new Date().toISOString()).run();
  };
}

// Logging middleware
function withLogging(logger: Logger): Middleware {
  return (next) => async (msg, env) => {
    logger.info({ event_id: msg.event_id, type: msg.type }, "processing");
    try {
      await next(msg, env);
      logger.info({ event_id: msg.event_id }, "completed");
    } catch (err) {
      logger.error({ event_id: msg.event_id, error: err.message }, "failed");
      throw err;
    }
  };
}

// Compose middleware
function pipe(...middlewares: Middleware[]): (handler: MessageHandler) => MessageHandler {
  return (handler) => middlewares.reduceRight((next, mw) => mw(next), handler);
}

// Usage
const processResearch = pipe(
  withDedup(env.DB),
  withLogging(logger),
)(async (msg, env) => {
  // Pure business logic — no boilerplate
  const { keywords, brand_slug } = msg.payload;
  for (const kw of keywords) {
    await researchKeyword(kw, brand_slug, env);
  }
});
```

Every handler gets deduplication, logging, and error tracking for free. Add metrics, rate limiting, or correlation tracking by adding a middleware. The business logic stays clean.

---

## Agents as Event Reactors

The [Cloudflare Agents SDK](https://developers.cloudflare.com/agents/) provides persistent, stateful Durable Objects that are perfect for the **decision-maker** role in an event-driven system.

A BrandAgent doesn't do the work. It **decides** what work needs doing, based on its accumulated state.

```typescript
import { Agent } from "agents";

interface BrandState {
  brand_slug: string;
  keywords_researched: number;
  articles_published: number;
  last_research_cycle: string | null;
  last_publish_cycle: string | null;
  pending_generation: number;
}

export class BrandAgent extends Agent<Env, BrandState> {
  initialState: BrandState = {
    brand_slug: "",
    keywords_researched: 0,
    articles_published: 0,
    last_research_cycle: null,
    last_publish_cycle: null,
    pending_generation: 0,
  };

  // Scheduled: wake up and assess what needs doing
  async discoveryCheck() {
    const daysSinceResearch = this.daysSince(this.state.last_research_cycle);

    if (daysSinceResearch > 7) {
      // Emit command — don't do the research here
      await this.env.RESEARCH_QUEUE.send({
        event_id: crypto.randomUUID(),
        type: "research.requested",
        source: "scalable-media",
        timestamp: new Date().toISOString(),
        correlation_id: `cycle-${this.state.brand_slug}-${Date.now()}`,
        payload: {
          brand_slug: this.state.brand_slug,
          keywords: await this.getTargetKeywords(),
          priority: "normal",
        },
      });

      this.setState({
        ...this.state,
        last_research_cycle: new Date().toISOString(),
      });
    }
  }

  // React to event: research is done
  async onResearchCompleted(event: DomainMessage<ResearchCompletedPayload>) {
    // Read research from GatherFeed's database
    const research = await this.fetchResearchByIds(event.payload.research_ids);

    // Apply strategy: filter, score, rank
    const candidates = this.evaluateKeywords(research);

    // Emit generation commands for the best candidates
    for (const candidate of candidates) {
      await this.env.CONTENT_QUEUE.send({
        event_id: crypto.randomUUID(),
        type: "content.generate",
        source: "scalable-media",
        timestamp: new Date().toISOString(),
        correlation_id: event.correlation_id,
        payload: {
          brand_slug: this.state.brand_slug,
          keyword: candidate.keyword,
          research_id: candidate.research_id,
          template: "pseo-article",
        },
      });
    }

    this.setState({
      ...this.state,
      keywords_researched: this.state.keywords_researched + research.length,
      pending_generation: candidates.length,
    });
  }

  // React to event: content was published
  async onContentPublished(event: DomainMessage<ContentPublishedPayload>) {
    this.setState({
      ...this.state,
      articles_published: this.state.articles_published + 1,
      pending_generation: Math.max(0, this.state.pending_generation - 1),
      last_publish_cycle: new Date().toISOString(),
    });

    // Schedule a performance check in 7 days
    await this.schedule(
      7 * 24 * 60 * 60, // seconds
      "checkPerformance",
      { urls: event.payload.urls }
    );
  }

  private evaluateKeywords(research: Research[]): Candidate[] {
    // Strategy logic: difficulty < 40, volume > 100, not already published
    return research
      .filter((r) => r.difficulty < 40 && r.volume > 100)
      .filter((r) => !this.isAlreadyPublished(r.keyword))
      .sort((a, b) => b.volume / b.difficulty - a.volume / a.difficulty)
      .slice(0, 10);
  }
}
```

The agent's core loop is **Decide → Evolve → Project** (borrowed from [Emmett](https://github.com/event-driven-io/emmett)'s event sourcing patterns):

- **Decide**: Given current state + incoming event → what commands to emit?
- **Evolve**: Update state based on what happened.
- **Project**: Build read models (published count, keyword performance, content calendar).

The agent never calls Perplexity. Never calls Gemini. Never publishes HTML. It decides, records, and commands. Everything else is someone else's job.

---

## Durable Workflows

For multi-step processes that **must complete**, use `AgentWorkflow`. Each step is checkpointed — a crash resumes from the last completed step, not from the beginning.

```typescript
import { AgentWorkflow } from "agents/workflows";
import type { AgentWorkflowEvent, AgentWorkflowStep } from "agents/workflows";

type GenerateParams = {
  brand_slug: string;
  keyword: string;
  research_id: string;
  template: string;
};

export class ContentWorkflow extends AgentWorkflow<BrandAgent, GenerateParams> {
  async run(event: AgentWorkflowEvent<GenerateParams>, step: AgentWorkflowStep) {
    const { brand_slug, keyword, research_id, template } = event.payload;

    // Step 1: Fetch research (durable — won't re-run on retry)
    const research = await step.do("fetch-research", {
      retries: { limit: 3, delay: "5 seconds", backoff: "exponential" },
      timeout: "30 seconds",
    }, async () => {
      return await fetchResearchFromGatherFeed(research_id, this.env);
    });

    this.reportProgress({ step: "research", status: "complete", percent: 0.2 });

    // Step 2: Generate outline
    const outline = await step.do("generate-outline", {
      retries: { limit: 3, delay: "10 seconds", backoff: "exponential" },
      timeout: "2 minutes",
    }, async () => {
      return await callGeminiViaApiMom("outline", { keyword, research, template }, this.env);
    });

    this.reportProgress({ step: "outline", status: "complete", percent: 0.4 });

    // Step 3: Draft article
    const draft = await step.do("draft-article", {
      retries: { limit: 3, delay: "10 seconds", backoff: "exponential" },
      timeout: "5 minutes",
    }, async () => {
      return await callGeminiViaApiMom("draft", { outline, research }, this.env);
    });

    this.reportProgress({ step: "draft", status: "complete", percent: 0.7 });

    // Step 4: Editorial pass
    const article = await step.do("editorial-pass", {
      retries: { limit: 2, delay: "10 seconds", backoff: "exponential" },
      timeout: "3 minutes",
    }, async () => {
      return await callGeminiViaApiMom("editorial", { draft, keyword }, this.env);
    });

    this.reportProgress({ step: "editorial", status: "complete", percent: 0.9 });

    // Step 5: Store and emit publish command
    await step.do("store-and-publish", {
      retries: { limit: 3, delay: "5 seconds", backoff: "exponential" },
    }, async () => {
      const contentId = await storeArticle(article, brand_slug, this.env);

      await this.env.PUBLISH_QUEUE.send({
        event_id: crypto.randomUUID(),
        type: "content.publish",
        source: "scalable-media",
        timestamp: new Date().toISOString(),
        payload: { brand_slug, content_id: contentId, keyword },
      });
    });

    await step.reportComplete({ keyword, brand_slug });
  }
}
```

Wrangler configuration for the workflow:

```jsonc
{
  "durable_objects": {
    "bindings": [
      { "name": "BRAND_AGENT", "class_name": "BrandAgent" }
    ]
  },
  "workflows": [
    {
      "name": "content-workflow",
      "binding": "CONTENT_WORKFLOW",
      "class_name": "ContentWorkflow"
    }
  ],
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["BrandAgent"] }
  ]
}
```

If step 3 (drafting) crashes because Gemini is down, the workflow retries step 3 — it doesn't re-fetch research or re-generate the outline. Steps 1 and 2 are already checkpointed.

---

## The Event Catalog

A shared, typed registry of every event in the system. Every service imports from it. This prevents message schema drift.

```typescript
// packages/events/src/index.ts

// ─── Base ────────────────────────────────────────────
export interface DomainMessage<T = unknown> {
  event_id: string;
  type: string;
  source: string;
  timestamp: string;
  correlation_id?: string;
  payload: T;
}

// ─── Research Domain ─────────────────────────────────
export interface ResearchRequestedPayload {
  brand_slug: string;
  keywords: string[];
  priority: "low" | "normal" | "high";
}

export interface ResearchCompletedPayload {
  brand_slug: string;
  research_ids: string[];
  keywords_researched: number;
}

// ─── Content Domain ──────────────────────────────────
export interface ContentGeneratePayload {
  brand_slug: string;
  keyword: string;
  research_id: string;
  template: string;
}

export interface ContentReadyPayload {
  brand_slug: string;
  content_id: string;
  keyword: string;
  word_count: number;
}

export interface ContentPublishPayload {
  brand_slug: string;
  content_id: string;
  keyword: string;
}

export interface ContentPublishedPayload {
  brand_slug: string;
  urls: string[];
  published_at: string;
}

// ─── Performance Domain ──────────────────────────────
export interface PerformanceReportPayload {
  brand_slug: string;
  url: string;
  impressions: number;
  clicks: number;
  position: number;
  period: string;
}

// ─── Type Map ────────────────────────────────────────
export type EventTypeMap = {
  "research.requested": ResearchRequestedPayload;
  "research.completed": ResearchCompletedPayload;
  "content.generate": ContentGeneratePayload;
  "content.ready": ContentReadyPayload;
  "content.publish": ContentPublishPayload;
  "content.published": ContentPublishedPayload;
  "performance.report": PerformanceReportPayload;
};

// ─── Helper ──────────────────────────────────────────
export function createMessage<K extends keyof EventTypeMap>(
  type: K,
  source: string,
  payload: EventTypeMap[K],
  correlationId?: string
): DomainMessage<EventTypeMap[K]> {
  return {
    event_id: crypto.randomUUID(),
    type,
    source,
    timestamp: new Date().toISOString(),
    correlation_id: correlationId,
    payload,
  };
}
```

Usage in any service:

```typescript
import { createMessage } from "@brand-engine/events";

await env.RESEARCH_QUEUE.send(
  createMessage("research.requested", "scalable-media", {
    brand_slug: "niche-fi",
    keywords: ["best budgeting apps", "compound interest calculator"],
    priority: "normal",
  }, correlationId)
);
```

Type-safe. Auto-completed. If the schema changes, every consumer gets a compile error.

---

## Small Patterns That Add Up

These are the building blocks. Each is small enough to drop into any Worker.

### Delayed Retry with Backpressure

When a downstream service is overloaded, don't hammer it. Delay the retry:

```typescript
async queue(batch: MessageBatch<DomainMessage>, env: Env): Promise<void> {
  for (const msg of batch.messages) {
    try {
      await handleMessage(msg.body, env);
      msg.ack();
    } catch (err) {
      if (err instanceof RateLimitError) {
        // Back off — re-deliver in 60 seconds
        msg.retry({ delaySeconds: 60 });
      } else if (err instanceof TransientError) {
        // Standard retry — re-deliver in 10 seconds
        msg.retry({ delaySeconds: 10 });
      } else {
        // Permanent failure — let it DLQ after max retries
        msg.retry();
      }
    }
  }
}
```

### Batch Sending

Queues support `sendBatch` — send up to 100 messages in one call. Use this when generating many commands at once:

```typescript
// BrandAgent discovers 50 keywords, sends them all at once
const messages = keywords.map((kw) => ({
  body: createMessage("research.requested", "scalable-media", {
    brand_slug: "niche-fi",
    keywords: [kw],
    priority: "normal",
  }, correlationId),
}));

await env.RESEARCH_QUEUE.sendBatch(messages);
```

### Event Deduplication in Agents

The Agent SDK's webhook guide recommends this pattern — track seen event IDs in the agent's SQLite:

```typescript
class BrandAgent extends Agent<Env, BrandState> {
  private async isProcessed(eventId: string): Promise<boolean> {
    const row = this.sql`
      SELECT 1 FROM processed_events WHERE event_id = ${eventId}
    `.toArray();
    return row.length > 0;
  }

  private async markProcessed(eventId: string, type: string): Promise<void> {
    this.sql`
      INSERT INTO processed_events (event_id, type, processed_at)
      VALUES (${eventId}, ${type}, ${new Date().toISOString()})
    `;
  }

  async handleEvent(event: DomainMessage) {
    if (await this.isProcessed(event.event_id)) return;
    // ... handle the event ...
    await this.markProcessed(event.event_id, event.type);
  }
}
```

### One Consumer, Multiple Queues

A single Worker can consume from multiple queues. Use `batch.queue` to route:

```typescript
export default {
  async queue(batch: MessageBatch<DomainMessage>, env: Env): Promise<void> {
    switch (batch.queue) {
      case "sm-commands":
        await handleCommands(batch, env);
        break;
      case "sm-events":
        await handleEvents(batch, env);
        break;
      case "sm-commands-dlq":
        await handleDeadLetters(batch, env);
        break;
    }
  },
};
```

### Correlation ID Propagation

Every command inherits the correlation ID from its triggering event. This lets you trace an entire cycle:

```typescript
// Event arrives: research.completed with correlation_id "cycle-niche-fi-1710..."
async onResearchCompleted(event: DomainMessage<ResearchCompletedPayload>) {
  // Commands inherit the correlation_id
  await env.CONTENT_QUEUE.send(
    createMessage("content.generate", "scalable-media", {
      brand_slug: event.payload.brand_slug,
      keyword: "best budgeting apps",
      research_id: "r_001",
      template: "pseo-article",
    }, event.correlation_id) // ← passed through
  );
}

// Later, in the DLQ handler, you can find every message in the chain:
// SELECT * FROM processed_events WHERE correlation_id = 'cycle-niche-fi-1710...'
```

### Scheduled Agent Cycle

An agent that wakes up on a cron, checks what needs doing, and emits commands:

```typescript
class BrandAgent extends Agent<Env, BrandState> {
  // Called by this.schedule("0 */6 * * *", "cycle")
  async cycle() {
    const state = this.state;

    // Check: do we need research?
    if (this.daysSince(state.last_research_cycle) > 7) {
      await this.requestResearch();
    }

    // Check: do we have unpublished content?
    const unpublished = this.sql`
      SELECT COUNT(*) as count FROM content WHERE published_at IS NULL
    `.toArray();

    if (unpublished[0].count > 0) {
      await this.publishPending();
    }

    // Check: do we need performance review?
    if (this.daysSince(state.last_performance_check) > 14) {
      await this.requestPerformanceData();
    }
  }
}
```

### Dead Letter Queue Monitor

A simple DLQ consumer that logs failures and alerts:

```typescript
// dlq-monitor/src/index.ts
export default {
  async queue(batch: MessageBatch<DomainMessage>, env: Env): Promise<void> {
    for (const msg of batch.messages) {
      // Log the failure with full context
      console.error(JSON.stringify({
        level: "error",
        message: "DLQ message received",
        event_id: msg.body.event_id,
        type: msg.body.type,
        source: msg.body.source,
        correlation_id: msg.body.correlation_id,
        timestamp: msg.body.timestamp,
        attempts: msg.attempts,
      }));

      // Store for investigation
      await env.DB.prepare(
        "INSERT INTO dlq_messages (event_id, type, source, payload, received_at) VALUES (?, ?, ?, ?, ?)"
      ).bind(
        msg.body.event_id,
        msg.body.type,
        msg.body.source,
        JSON.stringify(msg.body.payload),
        new Date().toISOString()
      ).run();

      msg.ack(); // Acknowledge so it doesn't loop
    }
  },
};
```

### Read-Only Cross-Service Data Access

Services can expose read-only HTTP APIs for data retrieval. This is the only permitted synchronous cross-service call:

```typescript
// GatherFeed exposes a read API — no auth needed for internal reads,
// or use service binding for zero-network-hop access
app.get("/api/v1/research/:id", async (c) => {
  const research = await c.env.DB.prepare(
    "SELECT * FROM research WHERE id = ?"
  ).bind(c.req.param("id")).first();

  if (!research) return c.json({ error: "not found" }, 404);
  return c.json(research);
});

// BrandAgent reads GatherFeed's data when deciding what to generate
// This is a query, not a command — it's fine to be synchronous
async fetchResearchByIds(ids: string[]): Promise<Research[]> {
  const results = await Promise.all(
    ids.map((id) =>
      fetch(`${this.env.GATHERFEED_URL}/api/v1/research/${id}`)
        .then((r) => r.json())
    )
  );
  return results;
}
```

### Schema Migration for Event Tables

Every service that consumes from queues needs these tables:

```sql
-- processed_events: idempotency tracking
CREATE TABLE IF NOT EXISTS processed_events (
  event_id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  processed_at TEXT NOT NULL
);

CREATE INDEX idx_processed_events_at ON processed_events(processed_at);

-- outbox: guaranteed event publication
CREATE TABLE IF NOT EXISTS outbox (
  event_id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  source TEXT NOT NULL,
  payload TEXT NOT NULL,
  correlation_id TEXT,
  created_at TEXT NOT NULL,
  published_at TEXT
);

CREATE INDEX idx_outbox_unpublished ON outbox(published_at) WHERE published_at IS NULL;

-- dlq_messages: dead letter queue investigation
CREATE TABLE IF NOT EXISTS dlq_messages (
  event_id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  source TEXT NOT NULL,
  payload TEXT NOT NULL,
  received_at TEXT NOT NULL,
  investigated_at TEXT
);
```

### Testing Queues Locally with Miniflare

You can test queue producers and consumers locally without deploying:

```typescript
import { Miniflare } from "miniflare";

const mf = new Miniflare({
  workers: [
    {
      name: "producer",
      modules: true,
      script: `
        export default {
          async fetch(request, env) {
            await env.QUEUE.send({ event_id: "test-1", type: "research.requested" });
            return new Response("sent");
          }
        }
      `,
      queueProducers: { QUEUE: "research-commands" },
    },
    {
      name: "consumer",
      modules: true,
      script: `
        export default {
          async queue(batch) {
            for (const msg of batch.messages) {
              console.log("received:", msg.body.type);
              msg.ack();
            }
          }
        }
      `,
      queueConsumers: { "research-commands": { maxBatchTimeout: 1 } },
    },
  ],
});

// Trigger the producer
const resp = await mf.dispatchFetch("http://localhost");
console.log(await resp.text()); // "sent"
// Consumer logs: "received: research.requested"
```

### Workflow Progress to WebSocket Clients

AgentWorkflow can broadcast progress to connected dashboard clients in real-time:

```typescript
class ContentWorkflow extends AgentWorkflow<BrandAgent, GenerateParams> {
  async run(event: AgentWorkflowEvent<GenerateParams>, step: AgentWorkflowStep) {
    // Non-durable: broadcasts to all WebSocket clients
    this.broadcastToClients({
      type: "workflow-started",
      keyword: event.payload.keyword,
    });

    const outline = await step.do("outline", { /* ... */ }, async () => {
      return await generateOutline(event.payload, this.env);
    });

    // Progress update — clients see this in real-time
    this.reportProgress({ step: "outline", status: "complete", percent: 0.3 });

    const draft = await step.do("draft", { /* ... */ }, async () => {
      return await generateDraft(outline, this.env);
    });

    // Durable state update — persists AND broadcasts
    await step.mergeAgentState({
      currentWorkflow: { keyword: event.payload.keyword, step: "editorial", percent: 0.7 },
    });

    // ... continue ...
  }
}

// Client-side React hook receives all updates automatically:
// const agent = useAgent({ agent: "brand-agent", name: "niche-fi",
//   onStateUpdate: (s) => setProgress(s.currentWorkflow)
// });
```

### Conditional Fan-Out

Route events based on payload content, not just event type:

```typescript
function getRoutes(event: DomainMessage): Queue[] {
  const routes: Queue[] = [];

  // Type-based routing
  if (event.type === "research.completed") {
    routes.push(env.SM_COMMANDS);
  }

  if (event.type === "content.published") {
    routes.push(env.SM_COMMANDS);

    // Only fan out to social if the brand has social enabled
    if (event.payload.brand_slug !== "internal-tools") {
      routes.push(env.SOCIAL_COMMANDS);
    }

    // Only fan out to analytics in production
    if (env.ENVIRONMENT === "production") {
      routes.push(env.ANALYTICS_QUEUE);
    }
  }

  // Priority-based routing
  if (event.payload.priority === "high") {
    routes.push(env.ALERTS_QUEUE);
  }

  return routes;
}
```

### Graceful Queue Consumer with Batch Acknowledgment

Process a batch, acknowledge the good messages, retry the bad ones:

```typescript
async queue(batch: MessageBatch<DomainMessage>, env: Env): Promise<void> {
  // Don't use batch.ackAll() or batch.retryAll()
  // Handle each message individually for granularity

  for (const msg of batch.messages) {
    try {
      await processMessage(msg.body, env);
      msg.ack(); // This one is done
    } catch (err) {
      if (isRetryable(err)) {
        msg.retry({ delaySeconds: computeBackoff(msg.attempts) });
      } else {
        // Log the permanent failure, ack so it goes to DLQ
        console.error(`Permanent failure for ${msg.body.event_id}:`, err);
        msg.ack(); // Will hit DLQ after max retries anyway
      }
    }
  }
}

function computeBackoff(attempts: number): number {
  // Exponential backoff: 5s, 10s, 20s, 40s, capped at 300s
  return Math.min(5 * Math.pow(2, attempts), 300);
}

function isRetryable(err: unknown): boolean {
  return err instanceof TransientError ||
    (err instanceof Response && err.status >= 500);
}
```

---

## What Not to Do

| Anti-Pattern | Why It Fails | Do This Instead |
|---|---|---|
| **Large payloads in messages** | 128KB limit. Queues are for signals, not data transfer. | Put data in D1/R2. Put a reference ID in the message. |
| **Assuming message order** | Queues don't guarantee order. `content.published` can arrive before `content.ready`. | Design every handler to work regardless of arrival order. Use state to reconcile. |
| **Sync disguised as async** | Sending a command then polling for the result is just HTTP with extra latency. | Let the completion event come to you. React, don't poll. |
| **Processing without dedup** | At-least-once will bite you. Duplicate research calls, duplicate articles, duplicate costs. | Check `event_id` before every action. Use `INSERT OR IGNORE` for writes. |
| **One giant queue** | Everything competes. A slow handler blocks fast ones. No isolation. | Separate command queues per service. Shared events queue with fan-out. |
| **Direct API calls between Workers** | Temporal coupling. CF error 1042 on same-account Workers. Cost amplification. | Queues for actions. Read-only HTTP only for data retrieval. |
| **Agent does the work** | The agent calls Perplexity, generates content, publishes HTML. Now it's a monolith in a Durable Object. | Agent decides and commands. Workflows and services do the work. |
| **Ignoring the DLQ** | Messages fail silently. You don't know something is broken until a customer complains. | Monitor DLQ. Alert on messages. Every DLQ message is a bug report. |
| **No correlation ID** | A brand cycle triggers 50 research commands, 30 generations, 20 publications. You can't trace the chain. | Every command inherits the `correlation_id` from the triggering event. |
| **Storing state in messages** | Messages are ephemeral. If you lose the message, you lose the state. | Store state in Agent SQLite or D1. Messages are notifications about state changes. |

---

## What You Don't Need

- **Kafka or RabbitMQ.** CF Queues handles 5,000 msg/sec per queue with auto-scaling consumers. Unless you're doing millions/sec, it's enough.
- **Full event sourcing.** Rebuilding state from event streams is powerful but complex. D1 is your source of truth. Events are signals, not the ledger.
- **A saga orchestrator library.** AgentWorkflow with durable steps IS the saga pattern. Each step is a compensatable transaction.
- **External pub/sub.** CF Queues + fan-out consumer covers fan-out. Event subscriptions cover native CF events.
- **Service mesh / service discovery.** Queue bindings in `wrangler.jsonc` ARE your service discovery. No runtime lookup needed.

---

## Cloudflare Constraints

Design around these. Don't fight them.

| Constraint | Impact | Design Response |
|---|---|---|
| One consumer per queue | No consumer groups like Kafka | One queue per consuming service. Fan-out consumer for broadcast. |
| At-least-once delivery | Duplicates will happen | Idempotency everywhere. `processed_events` table. Natural business keys. |
| No message ordering | Can't rely on sequence | Handlers must be order-independent. Use timestamps + state for reconciliation. |
| 128KB message limit | Can't send large payloads | Messages carry IDs. Data lives in D1/R2. |
| Worker CPU time limits | Long chains can timeout | Break chains into separate queue hops. Each hop is a fresh Worker invocation. |
| CF error 1042 | Same-account Workers can't `fetch()` each other via `workers.dev` | Don't use HTTP between Workers. Use Queues. Service bindings only for infrastructure. |
| No event replay | Can't rewind and replay like Kafka | Outbox pattern for guaranteed publication. DLQ for failed messages. Careful idempotency. |
| Queue `send()` can fail | Write succeeds, publish doesn't | Outbox pattern — write event to D1 in same transaction, publish from outbox. |

---

## Full Example: Content Pipeline

Here's the complete flow — a BrandAgent wakes up, triggers research, reacts to results, generates content, and publishes. Every arrow is a queue message:

```
1. BrandAgent scheduled wake-up fires (this.schedule, cron)
2. Agent checks state: "Last research was 8 days ago. Need keywords."
3. Agent sends →  RESEARCH_QUEUE:
     { type: "research.requested", payload: { brand_slug: "niche-fi", keywords: [...] } }

4. GatherFeed consumer picks up message
5. GatherFeed calls Perplexity via API Mom → keyword research
6. GatherFeed stores results in its D1
7. GatherFeed writes to outbox in same transaction
8. GatherFeed outbox flush → EVENTS_QUEUE:
     { type: "research.completed", payload: { brand_slug: "niche-fi", research_ids: [...] } }

9. Fan-out consumer reads event, routes to SM_COMMANDS queue

10. BrandAgent receives event, reads GatherFeed's DB via read API
11. Agent applies strategy: difficulty < 40, volume > 100, not published
12. Agent selects 10 keywords, starts ContentWorkflow for each

13. ContentWorkflow step.do("outline") → Gemini via API Mom
14. ContentWorkflow step.do("draft") → Gemini
15. ContentWorkflow step.do("editorial") → Gemini
16. ContentWorkflow step.do("store-and-publish") → D1 write + PUBLISH_QUEUE:
     { type: "content.publish", payload: { brand_slug: "niche-fi", content_id: "..." } }

17. Pages-plus consumer picks up message, reads content from SM's read API
18. Pages-plus writes to its D1, articles go live
19. Pages-plus writes to outbox → EVENTS_QUEUE:
     { type: "content.published", payload: { brand_slug: "niche-fi", urls: [...] } }

20. Fan-out consumer routes to SM_COMMANDS and SOCIAL_COMMANDS

21. BrandAgent records: articles live. Schedules performance check in 7 days.
22. Social-good receives event, creates social posts promoting the content.

23. Seven days later: BrandAgent wakes up, reads analytics, adjusts strategy.
24. Cycle continues.
```

No service waited for another. GatherFeed could take 5 minutes or 5 hours. If step 14 crashed, the workflow resumed at step 14, not step 1. Every service only knows about its own queue and the events queue. Add a new service by adding a line to the fan-out consumer's route table.

---

## References

### Cloudflare Documentation

- [Cloudflare Queues](https://developers.cloudflare.com/queues/) — message queues between Workers
- [How Queues Works](https://developers.cloudflare.com/queues/reference/how-queues-works/) — producers, consumers, batching, delivery guarantees
- [Queue Configuration](https://developers.cloudflare.com/queues/configuration/configure-queues/) — wrangler setup, DLQ, concurrency
- [Delivery Guarantees](https://developers.cloudflare.com/queues/reference/delivery-guarantees/) — at-least-once semantics
- [Batching and Retries](https://developers.cloudflare.com/queues/configuration/batching-retries/) — retry config, message delays
- [Consumer Concurrency](https://developers.cloudflare.com/queues/configuration/consumer-concurrency/) — auto-scaling consumers
- [Event Subscriptions](https://developers.cloudflare.com/queues/event-subscriptions/) — native CF service events (R2, KV, Workers AI, Workflows)
- [Cloudflare Agents SDK](https://developers.cloudflare.com/agents/) — persistent stateful agents on Durable Objects
- [Agents API Reference](https://developers.cloudflare.com/agents/api-reference/) — scheduling, queues, retries, state, workflows
- [Agent Workflows](https://developers.cloudflare.com/agents/concepts/workflows/) — durable multi-step execution
- [Build a Durable AI Agent](https://developers.cloudflare.com/workflows/get-started/durable-agents/) — tutorial: Agents SDK + Workflows
- [Workers Best Practices](https://developers.cloudflare.com/workers/best-practices/workers-best-practices/) — configuration, architecture, observability
- [Durable Objects](https://developers.cloudflare.com/durable-objects/) — stateful serverless with SQLite

### Libraries and Frameworks

- [@cloudflare/actors](https://github.com/cloudflare/actors) — Cloudflare's framework for easier Durable Objects, pub/sub, broadcast
- [Cloudflare Agents SDK](https://github.com/cloudflare/agents) — build and deploy AI agents on Workers
- [Durable Object Groups (DOG)](https://github.com/cloudflare/dog) — Cloudflare's library for DO replica clusters
- [Emmett](https://github.com/event-driven-io/emmett) — TypeScript event sourcing: decide/evolve/project pattern
- [Pongo](https://event-driven-io.github.io/Pongo/) — event sourcing on PostgreSQL (has Cloudflare Workers sample)
- [Watermill](https://github.com/ThreeDotsLabs/watermill) — Go event-driven library: middleware chains, pub/sub abstraction, CQRS
- [Dapr + Cloudflare Queues](https://github.com/diagrid-labs/dapr-cloudflare-queues) — event-driven reference with Dapr sidecar pattern
- [durable-objects-channel](https://github.com/jw-12138/durable-objects-channel) — pub/sub module built on Durable Objects
- [deco-cx/actors](https://github.com/deco-cx/actors) — Orleans-inspired virtual actors on Durable Objects and Deno

### Architecture Patterns

- [Event-Driven.io](https://event-driven.io/en/) — Oskar Dudycz's resources on event-driven architecture and event sourcing
- [CQRS Pattern](https://ibm-cloud-architecture.github.io/refarch-eda/patterns/cqrs/) — IBM's reference architecture for CQRS in event-driven systems
- [Awesome CQRS & Event Sourcing](https://github.com/leandrocp/awesome-cqrs-event-sourcing) — curated list of CQRS and event sourcing resources
- [Awesome Software Design](https://github.com/QDenka/awesome-software-design) — patterns, decisions, and design rules

---

## License

MIT
