---
name: tla-reasoning
description: Use simplified TLA+ mental models to reason about program state transitions, reducing logic bugs in workflows, agents, and distributed systems. Apply when designing workflows, debugging logic errors, reviewing architecture, reasoning about retries, designing agent loops, analyzing distributed coordination, validating edge cases, investigating race conditions, or validating state consistency.
---

# Skill: Reason About Program State Using TLA+ Concepts

## Purpose

Use simplified TLA+ mental models to reason about program state transitions.

The goal is **reducing logic bugs in workflows, agents, and distributed systems** by explicitly modeling:

- system state
- allowed transitions
- invariants
- edge cases
- failure paths

This skill is **not for writing formal TLA+ specifications**.  
It is a structured reasoning framework for analyzing program logic.

---

# Core Model

Every system can be described as:

State → Action → Next State

Where:

- **State** = all variables describing the system at a moment in time
- **Action** = a transition that changes state
- **Invariant** = something that must always remain true

Think in terms of **state machines**, not step-by-step code execution.

---

# When to Use This Skill

Apply this reasoning when:

- designing workflows
- debugging logic errors
- reviewing architecture
- reasoning about retries
- designing agent loops
- analyzing distributed coordination
- validating edge cases
- investigating race conditions
- validating state consistency

---

# Step-by-Step Reasoning Process

## 1. Identify State

List all variables describing system state.

Example:

```
state = {
  job_status
  retry_count
  queue
  active_workers
}
```

State should represent **facts**, not events.

Avoid things like:

```
last_event = "worker_started"
```

Prefer explicit state instead.

---

## 2. Define Valid Transitions

Describe **every way state can change**.

Example:

```
SubmitJob
WorkerStart
WorkerFinish
WorkerFail
RetryJob
Timeout
CancelJob
```

Every transition must specify:

```
preconditions
state updates
resulting state
```

If a transition is missing, the system may become stuck.

---

## 3. Describe Each Transition

Example:

```
RetryJob

Preconditions:
  job_status == failed
  retry_count < max_retries

State Change:
  retry_count += 1
  job_status = queued
```

Transitions should be:

- deterministic
- explicit
- state-driven

---

## 4. Identify Invariants

Invariants are **properties that must always hold**.

Examples:

```
retry_count <= max_retries

job_status ∈ {queued, running, failed, completed}

running_jobs <= worker_capacity

a job cannot be both running and completed
```

If any transition can violate an invariant, the design contains a bug.

---

## 5. Identify Impossible States

Look for combinations that should never occur.

Example:

```
job_status = running
worker_assigned = false
```

or

```
queue empty
workers idle
jobs pending
```

Impossible states often reveal hidden logic flaws.

---

## 6. Explore Edge Transitions

Check behavior for:

- retries
- timeouts
- cancellation
- duplicate events
- partial failures
- out-of-order events
- actor crashes

Ask:

```
What if this action happens twice?
What if it happens late?
What if it happens concurrently?
```

Systems must remain correct under these conditions.

---

## 7. Look for Missing Transitions

Many bugs occur because **some states cannot progress**.

Example:

```
job = running
worker = dead
```

If no transition handles this case, the system deadlocks.

---

# State Transition Table

Use a transition table to make system behavior explicit.

Template:

| Transition | Preconditions | State Updates | Next State |
|------------|--------------|--------------|------------|
| SubmitJob | queue accepts jobs | job_status = queued | queued |
| WorkerStart | job_status = queued AND worker available | job_status = running | running |
| WorkerFinish | job_status = running | job_status = completed | completed |
| WorkerFail | job_status = running | job_status = failed | failed |
| RetryJob | job_status = failed AND retry_count < max_retries | retry_count++, job_status = queued | queued |

Benefits:

- reveals missing transitions
- reveals conflicting transitions
- clarifies system behavior
- simplifies reasoning about concurrency

---

# Counterexample Thinking

Inspired by model checking tools like TLC.

Instead of asking:

```
Does the system work?
```

Ask:

```
Can I construct a sequence of events that breaks it?
```

Search for failure traces.

Example counterexample:

```
1. SubmitJob
2. WorkerStart
3. WorkerFail
4. RetryJob
5. WorkerStart
6. Timeout
7. WorkerFinish
```

Possible resulting state:

```
job_status = completed
timeout_handler already triggered retry
```

Now the job might be both:

```
completed
queued
```

This reveals a race condition.

---

# Counterexample Search Procedure

1. start from valid initial state
2. apply transitions in unusual orders
3. assume retries happen
4. assume events repeat
5. assume messages arrive late
6. assume actors crash

Try to reach:

- invariant violations
- deadlock states
- inconsistent state

If a sequence exists that breaks invariants, the design is unsafe.

---

# Concurrency Reasoning

Assume:

- actions can interleave
- actions can repeat
- actions can be delayed
- events can arrive out of order

Never rely on timing or ordering unless the system explicitly enforces it.

---

# Agent Loop Analysis

Agents can be modeled with state like:

```
state = {
  observation
  memory
  plan
  action_queue
  tool_results
}
```

Typical transitions:

```
Observe
Plan
Act
ToolReturn
Retry
Abort
```

Example invariants:

```
tool_calls_in_progress <= limit
memory_size <= cap
every action must eventually resolve
```

Agent systems frequently fail due to hidden state transitions.

---

# Common Bug Patterns This Skill Detects

## Missing Transition

State exists but cannot progress.

Example:

```
job = running
worker = dead
```

No recovery path.

---

## Illegal State Combination

Two state variables contradict each other.

Example:

```
status = completed
result = null
```

---

## Retry Explosion

Retry loop without termination.

Invariant required:

```
retry_count <= max_retries
```

---

## Race Condition

Two transitions finalize state.

Example:

```
WorkerFinish
Timeout
```

Both attempt to resolve the same job.

---

## Ghost State

State persists after it should be cleared.

Example:

```
active_worker reference remains after completion
```

---

# Output Format

When applying this skill, structure reasoning as:

```
State Variables

Transitions

State Transition Table

Invariants

Edge Cases

Counterexample Attempts

Potential Bugs

Recommended Fix
```

Focus on **state correctness**, not implementation details.

---

# Design Principles

Prefer:

- explicit state
- deterministic transitions
- bounded retries
- clearly defined invariants
- explicit failure handling

Avoid:

- hidden state
- implicit transitions
- unbounded loops
- reliance on timing

---

# Goal

Improve program correctness by reasoning in terms of:

```
states
transitions
invariants
counterexamples
```

This mindset exposes subtle bugs before implementation.
