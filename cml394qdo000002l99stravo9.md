---
title: "Cron vs Queue vs Event: Choosing the Right Trigger"
seoTitle: "Cron vs Queue vs Event: How to Choose the Right Trigger"
seoDescription: "A practical architectural guide to choosing between cron, queues, and events—explaining time-triggered vs event-driven execution, hybrid patterns"
datePublished: Sun Feb 01 2026 04:39:35 GMT+0000 (Coordinated Universal Time)
cuid: cml394qdo000002l99stravo9
slug: cron-vs-queue-vs-event-choosing-the-right-trigger
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769920068953/680cfe1b-de24-4ccf-a4a9-eaf0da270950.png
tags: devops, software-engineering, system-design, distributed-systems, cronjob, event-driven-architecture, job-queue, backendarchitecture

---

Most systems don’t fail because they picked the wrong database or framework. They fail because they picked the **wrong trigger**.

Something runs too early. Or too late. Or too often. Or only when a user happens to click a button. The code is correct, the infrastructure is healthy—and the behavior is still wrong.

This article is about learning to choose *how work starts*. Not how it’s written, not how it’s optimized, but **what causes it to run in the first place**. The frameworks and tools code examples used in this article are derived from my real production implementation.

Cron, queues, and events are not interchangeable. They encode different assumptions about time, causality, and responsibility. Understanding those assumptions is how you avoid architectural debt that only becomes visible years later.

---

## Three Triggers, Three Worldviews

At a high level, these mechanisms answer different questions:

* **Cron** answers: *“Is it time?”*
    
* **Queue** answers: *“Is there work waiting?”*
    
* **Event** answers: *“Did something happen?”*
    

They may all execute code, but they live in different mental models.

---

## Time-Triggered Execution (Cron)

Cron is **time-driven**. It does not care whether anything changed. It cares only that the clock says “now.”

In a Yii/HumHub setup, this often looks like:

```bash
* * * * * php yii cron/run
```

Inside the application:

```php
class CleanupJob extends \humhub\components\CronJob
{
    public function run()
    {
        Post::deleteAll(['<', 'created_at', strtotime('-30 days')]);
    }

    public function getSchedule()
    {
        return self::SCHEDULE_DAILY;
    }
}
```

Cron is ideal when:

* Work must happen *even if nothing else happens*
    
* You are reconciling or enforcing invariants
    
* You are comfortable with best-effort timing
    
* Skipped runs are acceptable or recoverable
    

Cron is **indifferent**. It runs whether the system is busy or idle, healthy or degraded.

That indifference is its power—and its danger.

---

## Work-Triggered Execution (Queues)

Queues are **state-driven**. They execute because work exists, not because time passed.

In Yii:

```php
Yii::$app->queue->push(new SendNotificationJob([
    'userId' => $userId,
]));
```

And later:

```bash
php yii queue/run
```

Queues are ideal when:

* Work volume is unpredictable
    
* You need retries, delays, or prioritization
    
* Execution should scale independently of scheduling
    
* Latency matters more than exact timing
    

Queues care deeply about **backlog**. If nothing is waiting, nothing runs. If a lot is waiting, they absorb pressure instead of collapsing.

Queues answer *how much* and *how fast*. They do not answer *when* something should be considered in the first place.

---

## Event-Driven Execution

Events are **causality-driven**. They run because *something specific happened*.

In application code:

```php
Event::trigger(User::class, User::EVENT_AFTER_INSERT, new Event([
    'sender' => $user,
]));
```

Or conceptually:

```php
onUserRegistered($user) {
    // react immediately
}
```

Events are ideal when:

* A specific state change matters
    
* The reaction must be immediate or contextual
    
* You want minimal latency
    
* You can tolerate missed events only rarely
    

Events encode **meaning**, not schedule. They answer *why* something should run.

But events are fragile when used for work that must happen regardless of user behavior.

---

## A Simple Decision Matrix

When deciding how to trigger work, ask these questions:

### 1\. Does this need to happen even if nobody does anything?

* **Yes** → Cron
    
* **No** → Queue or Event
    

### 2\. Does this need to happen immediately when something changes?

* **Yes** → Event
    
* **No** → Cron or Queue
    

### 3\. Is the amount of work unpredictable or bursty?

* **Yes** → Queue
    
* **No** → Cron or Event
    

### 4\. Can this safely run twice?

* **No** → Avoid cron unless idempotency or locking exists
    
* **Yes** → Cron becomes viable
    

### 5\. Is time the reason this work exists?

* **Yes** → Cron
    
* **No** → Queue or Event
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1769920525465/8e19a90b-eb81-4d77-9a5d-aa79cd42eb3d.png align="center")

Most bad designs come from answering these questions incorrectly—or not asking them at all.

---

## Hybrid Patterns (Where Real Systems Live)

Mature systems rarely choose just one trigger. They **compose** them.

### Pattern 1: Cron → Queue (Time Decides, Queue Executes)

This is the HumHub pattern that I’ve implemented.

```bash
* * * * * php yii cron/run
```

```php
public function run()
{
    Yii::$app->queue->push(new RecalculateStatsJob());
}
```

Cron decides *when*.  
The queue decides *how fast*.

This is ideal for periodic but heavy work.

### Pattern 2: Event → Queue (Meaning Decides, Queue Executes)

```php
Event::on(User::class, User::EVENT_AFTER_INSERT, function ($event) {
    Yii::$app->queue->push(new SendWelcomeEmailJob([
        'userId' => $event->sender->id,
    ]));
});
```

The event provides context.  
The queue provides resilience.

This keeps user-facing actions fast while preserving intent.

### Pattern 3: Event + Cron (Immediate + Reconciliation)

Events do the fast path. Cron does the safety net.

```php
// Event-driven
onOrderPaid($order) {
    markAsProcessed($order);
}

// Cron-driven reconciliation
class OrderReconciliationJob extends CronJob {
    public function run() {
        $this->fixInconsistentOrders();
    }
}
```

This pattern accepts that events can be missed and uses time-based checks to restore correctness.

This is not redundancy. It is **defensive architecture**.

---

## Why Misuse Creates Architectural Debt

The most expensive mistakes are subtle.

### Using Cron for Event-Driven Work

Example:

> “Send emails every minute and see who signed up.”

Problems:

* Unnecessary polling
    
* Delayed reactions
    
* Growing query cost
    
* Confusing intent
    

You encoded *meaning* (user signed up) as *time* (every minute). That mismatch becomes technical debt.

### Using Events for Time-Based Guarantees

Example:

> “Clean up expired sessions when users log in.”

What if nobody logs in?

Now cleanup depends on behavior unrelated to the task’s purpose. That’s accidental coupling.

### Using Queues as a Scheduler

Example:

> “Push a delayed job and hope it fires at the right time.”

Queues are not clocks. Delays drift. Retries compound. Restarts blur guarantees.

Time-based intent belongs to a scheduler, not a backlog.

---

## The Core Insight: Triggers Encode Assumptions

Every trigger bakes in assumptions:

* **Cron** assumes time is the reason work exists
    
* **Queues** assume work volume is the problem
    
* **Events** assume causality is the signal
    

If those assumptions are wrong, the system *still works*—just increasingly badly.

That’s why misuse is so dangerous. It doesn’t break immediately. It **ages poorly**.

---

## Designing Instead of Guessing

Instead of asking:

> “How should I run this code?”

Ask:

> “Why should this code run at all?”

* If the answer is *“because time passed”* → Cron
    
* If the answer is *“because work exists”* → Queue
    
* If the answer is *“because something happened”* → Event
    

Only after that do frameworks, tools, and syntax matter.

---

## The Takeaway

Cron, queues, and events are not competitors. They are **orthogonal tools**.

Good systems:

* Use cron sparingly and deliberately
    
* Let queues absorb pressure
    
* Let events carry meaning
    
* Combine them where guarantees matter
    

Bad systems pick one and force everything through it.

Choosing the right trigger is not an implementation detail.  
It’s an architectural commitment.

And once you see that clearly, entire classes of problems stop appearing—not because you fixed them, but because you never created them in the first place.

---

## ☰ Series Navigation

### Core Series

* **Introduction**
    
* **Part 1:** Cron: The Invisible Operating System
    
* **Part 2:** Anatomy of a Cron Job
    
* **Part 3:** Cron at Scale: Patterns and Anti-Patterns
    
* **Part 4:** Cron in Frameworks: From Theory to Convention
    
* **Part 5:** HumHub & Yii: Design Intent Behind the Cron Architecture
    
* **Part 6:** A Real Production Setup: What I Actually Built
    
* **Part 7:** Failure Modes, Tradeoffs, and Lessons Learned
    
* **Part 8:** The Evolution Path: From Cron to Orchestration
    

### Optional Extras

* ⏳ Cron Lies: When Scheduled Jobs Don’t Run
    
* 🔁 Idempotency: The Most Important Word in Cron
    
* → ⚖️ Cron vs Queue vs Event