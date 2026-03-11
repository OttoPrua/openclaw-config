---
name: task-queue
description: Provider-aware task queue for managing LLM subscription concurrency limits across multiple agents. Use when spawning subagents to check provider slot availability, queue overflow tasks, and dispatch pending work. Covers two workflows — (1) SUBMIT: before spawning a subagent, check slots and either spawn directly or enqueue; (2) DISPATCH: drain pending queue when slots free up. Triggers on task queue operations, provider concurrency management, slot checking before spawn.
---

# Provider-aware Task Queue

Manages LLM provider concurrency to avoid hitting subscription slot limits when multiple agents share the same provider.

## Problem

When multiple agents share a single LLM provider subscription (e.g., 6 agents on one Anthropic Pro plan), simultaneous `sessions_spawn` calls can exceed the provider's concurrent session limit, causing failures. This skill adds a lightweight queue layer that checks slot availability before spawning and queues overflow tasks for later dispatch.

## Architecture

```
workspace/queue/
├── config.json        # Provider limits + agent-to-provider mapping
├── pending/           # Waiting tasks, by provider
│   ├── anthropic/
│   ├── openai-codex/
│   └── google/
├── active/            # Currently executing (tracking)
└── done/              # Completed results
```

## Setup

### 1. Create queue directories

```bash
mkdir -p workspace/queue/{pending/{anthropic,openai-codex,google},active,done}
```

### 2. Create config.json

```json
{
  "providers": {
    "anthropic": {
      "maxConcurrent": 5,
      "reserved": 1,
      "note": "reserved=1 keeps a slot for the main dispatcher agent"
    },
    "openai-codex": {
      "maxConcurrent": 2,
      "reserved": 0
    },
    "google": {
      "maxConcurrent": 2,
      "reserved": 0
    }
  },
  "priorities": ["urgent", "normal", "low"],
  "dispatchIntervalSec": 120,
  "taskTTLMinutes": 120,
  "agentToProvider": {
    "agent-a": "anthropic",
    "agent-b": "anthropic",
    "agent-c": "openai-codex",
    "agent-d": "google"
  },
  "dispatchCronId": "<your-cron-job-id>"
}
```

Adjust `maxConcurrent` to match your subscription limits. `reserved` keeps slots for main sessions (not available to queued tasks).

### 3. Set up dispatcher cron

Create a cron job (default: disabled, activated on demand):

```
cron add:
  name: "Task Queue Dispatcher"
  schedule: { kind: "every", everyMs: 120000 }
  payload: { kind: "agentTurn", message: "Execute Workflow 2: DISPATCH from skills/task-queue/SKILL.md" }
  sessionTarget: "isolated"
  enabled: false
```

Store the returned cron ID in `queue/config.json` as `dispatchCronId`.

## Workflow 1: SUBMIT (any dispatching agent)

Before calling `sessions_spawn`, follow these steps:

### Step 1: Determine provider

Read `queue/config.json`. Look up the target agent's provider via `agentToProvider`.

### Step 2: Count active slots

Call `sessions_list(activeMinutes=30)` and count sessions whose agent uses the same provider.

### Step 3: Decision

```
available = maxConcurrent - reserved - activeCount

if available > 0:
    → sessions_spawn directly (normal flow, zero delay)
else:
    → Write task file to queue (Step 4)
```

### Step 4: Enqueue (only when full)

Write task JSON to `queue/pending/<provider>/<timestamp>-<priority>-<label>.json`:

```json
{
  "id": "<uuid or timestamp>",
  "agentId": "<target agent>",
  "provider": "<provider>",
  "priority": "urgent|normal|low",
  "task": "<the task prompt>",
  "model": "<optional model override>",
  "mode": "run|session",
  "label": "<descriptive label>",
  "submittedBy": "<requesting agent>",
  "submittedAt": "<ISO timestamp>",
  "callbackSession": "<session key to notify on completion>"
}
```

### Step 5: Activate dispatcher

Enable the dispatcher cron:

```
cron(action=update, jobId=<dispatchCronId>, patch={enabled: true})
```

## Workflow 2: DISPATCH (cron job)

### Step 1: Scan pending tasks

List files in `queue/pending/*/`. If empty → skip to Step 5.

### Step 2: Check slots per provider

For each provider with pending tasks:
- Call `sessions_list(activeMinutes=30)` to count active sessions
- Calculate `available = maxConcurrent - reserved - activeCount`

### Step 3: Dispatch

For each provider with available slots:
1. Sort pending tasks: `urgent` first, then `normal`, then `low` (FIFO within same priority)
2. Call `sessions_spawn(agentId, task, mode, model, label)`
3. Move task file from `pending/` to `active/`

### Step 4: Handle completions

Check `active/` for completed sessions → move to `done/`, notify callback if set.

### Step 5: Self-disable check

If `pending/` AND `active/` are both empty:
- `cron(action=update, jobId=<dispatchCronId>, patch={enabled: false})`

## Design Philosophy

- **90% direct spawn**: Most tasks go through instantly when slots are available
- **10% queued**: Only overflow tasks enter the queue
- **Zero-cost idle**: Dispatcher cron stays disabled when queue is empty
- **Priority support**: urgent > normal > low, FIFO within same priority

## Priority Definitions

| Priority | Use Case |
|----------|----------|
| `urgent` | Direct user request, time-sensitive |
| `normal` | Routine delegated tasks |
| `low` | Background maintenance, non-urgent |

## Error Handling

- **Spawn failure**: Move task back to `pending/`, log error
- **Task timeout**: Tasks in `active/` exceeding `taskTTLMinutes` → move to `done/` with `timeout` status
- **Config read failure**: Fall back to conservative defaults (maxConcurrent=2)

## References

- Inspired by [block/agent-task-queue](https://github.com/block/agent-task-queue) (Named Queue + Semaphore pattern)
- OpenClaw built-in lane system handles message serialization; this skill adds provider-level concurrency on top
- OpenClaw Feature Request #13615 (`openclaw limits set`) may provide native support in the future

## Credits

- Research by the information gathering agent using web search across GitHub, ClawHub, and Chinese tech communities
- Thanks to Block (Square) engineering team for the agent-task-queue design pattern reference
