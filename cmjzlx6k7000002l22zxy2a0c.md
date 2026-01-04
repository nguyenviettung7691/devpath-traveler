---
title: "Event Contracts as APIs"
seoTitle: "Event Contracts as APIs"
seoDescription: "Learn why event contracts should be treated as APIs, with practical versioning, validation, and idempotency patterns for event-driven systems."
datePublished: Sun Jan 04 2026 10:46:50 GMT+0000 (Coordinated Universal Time)
cuid: cmjzlx6k7000002l22zxy2a0c
slug: event-contracts-as-apis
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1767522686345/6dbda5f8-5dc7-497f-a949-0da66e97cb39.png
tags: microservices, software-architecture, system-design, distributed-systems, event-driven-architecture, backendarchitecture, event-contracts

---

Series: [Designing a Microservice-Friendly Datahub](https://devpath-traveler.nguyenviettung.id.vn/series/designing-microservice-datahub)

In event-driven systems, APIs don’t disappear—they **move**. Instead of living behind HTTP endpoints, they live inside messages. Every event you publish becomes a promise to unknown consumers, running unknown versions of code, at unknown times.

That promise is the **event contract**.

This article treats event contracts as first-class APIs and shows—using practical examples with **PHP**, **Redis Streams**, **.NET**, **RabbitMQ**, and **Node.js**—how to design, version, validate, and consume them safely. These frameworks and tools stack closely mirrors my actual implemented Datahub.

---

## What Is an Event Contract?

An event contract defines:

* **What happened** (semantic meaning)
    
* **How it’s represented** (schema)
    
* **What’s guaranteed** (fields, types, invariants)
    
* **How it evolves** (versioning & compatibility)
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767522861713/17fc201c-d083-4cf0-ac52-57584a0e4b46.png align="center")

If REST APIs answer *“What can you ask me to do?”*, event contracts answer *“What facts will I reliably tell you?”*

Once an event leaves the producer, the contract becomes public.

---

## A Minimal, Strong Event Envelope

Start with a boring, explicit envelope. Boring is good.

```php
{
  "event_type": "user.updated",
  "event_version": 1,
  "event_id": "9b6c8c2a-7c9d-4b7f-9a1e-1e1f9a5b3f6a",
  "occurred_at": "2025-01-02T10:15:30Z",
  "producer": "web-app",
  "data": {
    "user_id": 123,
    "display_name": "Alice Nguyen"
  }
}
```

Why this works:

* `event_type` + `event_version` → routing + compatibility
    
* `event_id` → idempotency
    
* `occurred_at` → temporal reasoning
    
* `producer` → ownership
    
* `data` → the only part consumers interpret
    

---

## Ownership: One Event, One Owner

Every event must have **exactly one owner**.

The owner:

* Defines the schema
    
* Controls versioning
    
* Decides meaning
    

Consumers:

* Interpret
    
* React
    
* Derive state
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767523016652/fefc010e-9db0-4a34-b093-9a4b45539c0e.png align="center")

They do **not** negotiate schema changes. If multiple teams “co-own” an event, no one owns it—and evolution stalls.

---

## Producing Events (PHP → Redis Streams)

Producers should emit events **after** state is committed. Keep emission simple.

```php
$redis->xAdd(
    'events',
    '*',
    [
        'event_type' => 'user.updated',
        'event_version' => 1,
        'event_id' => uuid_create(UUID_TYPE_RANDOM),
        'occurred_at' => gmdate('c'),
        'producer' => 'web-app',
        'data' => json_encode([
            'user_id' => $userId,
            'display_name' => $displayName
        ])
    ]
);
```

Notes:

* No routing logic
    
* No consumer awareness
    
* No retries here
    

The producer’s job is to **state facts**, not orchestrate outcomes.

---

## Versioning Rules (Non-Negotiable)

### Rule 1: Never Change Meaning In Place

Changing semantics without a version bump is a lie.

**Bad**

Change “display\_name” field to “full\_name” value without any warning.

```json
"display_name": "Alice"  // suddenly means "full_name"
```

**Good**

Bump version first, add new field for new meaning.

```json
{
  "event_version": 2,
  "data": { "full_name": "Alice Nguyen" }
}
```

### Rule 2: Prefer Additive Changes

Add fields; don’t remove or change existing ones.

```json
{
  "event_version": 1,
  "data": {
    "user_id": 123,
    "display_name": "Alice",
    "avatar_url": null
  }
}
```

Old consumers keep working. New consumers opt in.

### Rule 3: Breaking Changes Require New Versions

Breaking changes demand:

* New `event_version`
    
* Parallel support during migration
    
* Clear deprecation timelines
    

---

## Validating at the Edge (Producer / Translator)

Fail fast **before** publishing.

### .NET (schema gate before RabbitMQ publish)

```csharp
if (!SchemaRegistry.IsValid("user.updated", 1, payload))
{
    throw new InvalidOperationException("Invalid event contract");
}
```

Validation at the edge prevents malformed truth from spreading.

---

## Publishing to RabbitMQ (Translator / Bridge)

Use routing keys that describe *what happened*, not *who should react*.

```csharp
channel.ExchangeDeclare(
    exchange: "events",
    type: ExchangeType.Topic,
    durable: true
);

channel.BasicPublish(
    exchange: "events",
    routingKey: "user.updated",
    body: Serialize(payload)
);
```

Producers publish. Consumers opt in.

---

## Consuming Defensively (Node.js)

Consumers should:

* Accept only versions they support
    
* Ignore unknown fields
    
* Fail loudly on incompatible versions
    
* Be idempotent
    

```javascript
if (event.event_type !== "user.updated" || event.event_version !== 1) {
  throw new Error("Unsupported event contract");
}

// idempotency
await db.query(
  `INSERT INTO processed_events (event_id)
   VALUES (?) ON DUPLICATE KEY UPDATE event_id = event_id`,
  [event.event_id]
);
```

Silent acceptance of incompatible versions is how data corruption sneaks in.

---

## Idempotency Depends on the Contract

At-least-once delivery means duplicates happen. Contracts make retries safe.

```sql
INSERT INTO processed_events (event_id)
VALUES (:event_id)
ON DUPLICATE KEY UPDATE event_id = event_id;
```

No stable `event_id` → retries become dangerous.

---

## Contracts as Living Documentation

Treat schemas like code:

```plaintext
events/
  user.updated/
    v1.json
    v2.json
  order.completed/
    v1.json
```

Benefits:

* Shared vocabulary
    
* Onboarding material
    
* Reviewable changes
    
* Fewer meetings
    

Schemas replace tribal knowledge.

---

## Events vs REST APIs (Responsibility Check)

| Aspect | REST API | Event Contract |
| --- | --- | --- |
| Direction | Request/Response | Broadcast |
| Coupling | Caller knows callee | Producer ignores consumers |
| Failure | Immediate | Deferred |
| Versioning | Endpoint-based | Schema-based |
| Testing | Integration tests | Contract tests |

Different mechanics. Same responsibility.

---

## Common Anti-Patterns (Avoid These)

* “We’ll just add this field—no version needed”
    
* “Consumers can figure it out”
    
* “Reuse one event for multiple meanings”
    
* “We’ll document later”
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767523587834/90c5cee2-ef32-4fef-bb9e-8d641eedd2d9.png align="center")

All end the same way: silent drift, then painful rewrites.

---

## Closing Thought

In event-driven systems, **your schema is your handshake**.

Make it:

* Explicit
    
* Versioned
    
* Owned
    
* Boring
    

Events don’t just carry data.  
They carry **trust**.

And trust, once broken, is far harder to replay than any message stream.