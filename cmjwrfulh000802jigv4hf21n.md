---
title: "Lessons Learned and Future Improvements for my implemented Datahub"
seoTitle: "Datahub Lessons Learned and Evolution"
seoDescription: "Lessons learned from a real Datahub implementation, covering what worked, bottlenecks found, and how the architecture can evolve."
datePublished: Fri Jan 02 2026 10:58:01 GMT+0000 (Coordinated Universal Time)
cuid: cmjwrfulh000802jigv4hf21n
slug: lessons-learned-and-future-improvements-for-my-implemented-datahub
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1767350018887/9c29f59f-820f-49ed-9056-ceb47b28997c.png
tags: microservices, software-architecture, system-design, distributed-systems, case-study, event-driven-architecture, datahub

---

Series: [Designing a Microservice-Friendly Datahub](https://devpath-traveler.nguyenviettung.id.vn/series/designing-microservice-datahub)  
PART III — CASE STUDY: MY CSL DATAHUB IMPLEMENTATION  
Previous: [End-to-End Data Flow Scenarios in Datahub](https://devpath-traveler.nguyenviettung.id.vn/end-to-end-data-flow-scenarios-in-datahub)

Architectures don’t become “good” because they were well-designed on day one. They become good because they **survive contact with reality**, accumulate scar tissue, and adapt without collapsing.

This final article steps away from diagrams and patterns to reflect on what the CSL Datahub taught us in practice: what worked, what surprised us, where the cracks appeared first, and how the system could evolve if built again today.

This is where architecture grows up.

*Disclaimer (Context & NDA)  
The CSL Datahub implementation discussed throughout this case study was designed and built in 2021. While the architectural principles remain sound, some tooling choices could be updated today. To comply with NDA requirements, business-specific logic, schemas, and operational details are intentionally generalized.*

---

## What Worked Well (And Why It Worked)

Some design decisions paid off immediately—and kept paying dividends.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767350227402/e3262b18-0f26-4bf6-977d-4cb8e5df7365.png align="center")

### 1\. Clear Ownership of State

Making the CSL Web App and its MySQL database the **single source of truth** removed ambiguity early.

Benefits:

* No split-brain state
    
* No cross-service writes
    
* No schema politics
    

Every downstream system knew its role: *react, don’t mutate*.

This clarity prevented entire classes of bugs.

### 2\. Event Emission After Commit

Events were emitted **after** database transactions completed, never before.

```php
$db->commit();

$redis->xAdd(
    'csl:events',
    '*',
    [
        'type' => 'user.updated',
        'user_id' => $user->id,
        'occurred_at' => time()
    ]
);
```

This simple discipline avoided phantom events, partial updates, and confusing rollback scenarios. It also made replay and reasoning straightforward.

### 3\. Redis Streams as a Shock Absorber

Redis Streams quietly did exactly what they were meant to do:

* Absorb bursts
    
* Decouple time
    
* Protect the legacy app
    

They were rarely discussed—and that’s the highest compliment.

When systems are calm under pressure, it’s usually because something unglamorous is doing its job well.

### 4\. RabbitMQ for Decentralized Consumption

RabbitMQ scaled not just in traffic, but in **organizational complexity**.

Teams could:

* Add consumers independently
    
* Remove consumers without coordination
    
* Evolve their logic safely
    

Publish/subscribe worked as intended—no central registry of dependencies, no choreography meetings.

---

## What Was Harder Than Expected

Some challenges only appear once the system is alive.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767350520833/fe2562ff-510a-40ff-91cd-b6dfc71c0ae3.png align="center")

### 1\. Operational Visibility Took Real Effort

Event-driven systems fail *between* services, not inside them.

We learned quickly that:

* CPU metrics weren’t enough
    
* Service uptime lied
    
* Queue lag told the real story
    

It took deliberate work to monitor:

* Redis stream lag
    
* RabbitMQ queue depth
    
* Processor throughput
    
* DLQ growth
    

Visibility wasn’t optional—it was a feature we had to build.

### 2\. Idempotency Was Non-Negotiable

At-least-once delivery meant duplication was inevitable.

The teams that embraced idempotency early slept better:

```sql
INSERT INTO processed_events (event_id)
VALUES (:event_id)
ON DUPLICATE KEY UPDATE event_id = event_id;
```

The teams that didn’t… learned the hard way.

Idempotency isn’t clever. It’s defensive driving.

### 3\. The Processor Attracted Gravity

Despite best intentions, the Processor constantly tempted engineers with convenience.

“If we already have the event here, why not just…?”

That sentence is how **God services** are born.

It required discipline to:

* Keep logic shallow
    
* Push domain meaning to consumers
    
* Split handlers early when responsibilities diverged
    

Architectural boundaries don’t enforce themselves.

---

## Bottlenecks Discovered Over Time

No system escapes bottlenecks—only ignorance of them.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767350724982/aa6cc9a0-3a3f-42b2-a037-ddc4b681ae28.png align="center")

### Processor Throughput

As event volume grew, Processor lag became the first scaling signal.

The fix wasn’t “more CPU.” It was:

* Horizontal scaling
    
* Splitting handlers by responsibility
    
* Reducing synchronous API calls
    

The bottleneck revealed **where coupling still existed**.

### Message Storms

Some early event designs were too chatty.

Emitting events on every micro-change caused:

* Unnecessary fan-out
    
* Consumer overload
    
* Hard-to-reason flows
    

The solution was not throttling—it was **semantic restraint**:

* Fewer events
    
* More meaningful events
    
* Better aggregation
    

---

## Architecture Is a Living System

One of the most important lessons was psychological, not technical.

Architectures don’t “finish.” They **age**.

What worked at:

* 3 modules
    
* 2 teams
    
* 10k events/day
    

…needed adjustment at:

* 10 modules
    
* 6 teams
    
* 1M events/day
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767350853011/5039492c-5320-4e1e-ba0f-314d7762afea.png align="center")

Treating architecture as frozen design would have killed the system. Treating it as a living system kept it adaptable.

---

## Future Improvements (If We Were Building This Today)

With hindsight—and modern tooling—several evolutions would make sense.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767351277858/3824679c-72c9-4f07-aa4f-4dec2d70dbb9.png align="center")

### 1\. Kafka for High-Volume Event History

If:

* Event replay became core
    
* Stream processing emerged
    
* Retention requirements grew
    

Kafka would be a natural next step.

Not as a replacement for everything—but as a **data backbone** where history matters.

### 2\. Splitting the Processor by Responsibility

Rather than one Processor service:

* One for Redis → RabbitMQ
    
* One for RabbitMQ → CSL API
    
* One for external integrations
    

This would reduce blast radius and simplify scaling decisions.

### 3\. Formal Event Contracts

Early contracts were implicit. Later, they deserved structure.

A shared schema repo with versioning would:

* Reduce accidental breakage
    
* Improve onboarding
    
* Enable better validation
    

Contracts are how event-driven systems communicate *trust*.

### 4\. Deeper Observability by Default

If rebuilt today:

* Distributed tracing would be first-class
    
* Correlation IDs everywhere
    
* Pipeline-level dashboards from day one
    

Debugging async systems without visibility is archaeology.

---

## Reflection Is Part of Engineering

The biggest takeaway isn’t about Redis, RabbitMQ, or .NET.

It’s this:

> Architecture is not about being right.  
> It’s about being **able to change** without fear.

The CSL Datahub worked not because it was perfect, but because it respected:

* Boundaries
    
* Ownership
    
* Failure
    
* Time
    

Those principles outlast tools.

---

## Closing the Case Study (And the Series)

This article closes the CSL case study—but not the conversation.

If there’s one thing worth carrying forward, it’s this mental shift:  
**design systems as conversations, not call graphs**.

When systems speak in facts, tolerate delay, and respect ownership, they scale—not just technically, but humanly.

That’s what Datahub architecture is really about.

And that’s the kind of architecture worth building.

### **Optional Extra Articles**

If you’d like to go deeper, the following optional articles explore specific corners of the architecture—contracts, failure handling, observability, tooling trade-offs, and scaling pressure points that deserve their own focused discussion.

**🧾** [**Event Contracts as APIs**](https://devpath-traveler.nguyenviettung.id.vn/event-contracts-as-apis)

**♻️** [**Dead Letter Queues and Retry Strategies**](https://devpath-traveler.nguyenviettung.id.vn/dead-letter-queues-and-retry-strategies)

**🔍** [**Observability for Event-Driven Systems**](https://devpath-traveler.nguyenviettung.id.vn/observability-for-event-driven-systems)

**⚖️ Redis Streams vs Kafka: Choosing the Right Event Backbone**

**🚦 When the Processor Becomes a Bottleneck**