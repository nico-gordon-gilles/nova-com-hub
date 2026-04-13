# Nova Com Hub

> Central message router for a multi-agent AI system — with loop detection, human approval gates and delivery guarantees.

Nova Com Hub is the communication backbone between Nova's agents. When multiple AI agents need to talk to each other, the Com Hub ensures messages are routed correctly, loops are detected and broken, sensitive actions require human approval, and no message is delivered twice.

---

## The Problem It Solves

In a multi-agent system, agents can accidentally create loops:

```
Agent A sends to Agent B
Agent B sends to Agent C
Agent C sends back to Agent A
→ infinite loop, system hangs
```

Com Hub breaks this at the routing layer — before it becomes a problem.

---

## Architecture

```
Agent A
  ↓ POST /message { to, content, agent_id, token }
Router (router.js)
  ↓
  ├── Validate token (bcrypt hash check)
  ├── Loop Detector   → seen this message chain before? → reject
  ├── Approval Gate   → action requires human OK? → queue + notify Nico
  ├── Project Loops   → is this part of a defined workflow?
  └── Delivery (with semaphore per target)
        ↓
      Agent B / Bridge Connector / Claude Code
```

---

## Components

### Router
The central dispatcher. Every message passes through here:
- Token validation (bcrypt hash, stored in DB)
- Loop detection check
- Approval gate check
- Semaphore-protected delivery (one message at a time per target)

### Loop Detector
Tracks message chains and detects cycles:

```javascript
// Each message gets a chain ID
// Loop = same chain ID seen at the same agent twice
// → System alert → message dropped
writeSystemAlert("Loop detected", { chain_id, agent_id })
```

### Approval System
Some agent actions require Nico's explicit OK before delivery:

```
Agent wants to: delete files / send email / external API call
→ Com Hub queues the message
→ Push notification to Nico: "Agent X wants to do Y — approve?"
→ Nico: yes → deliver | no → drop
→ Timeout: auto-drop after configured interval
```

### Project Loops
Predefined multi-agent workflows. A project loop defines which agents participate and in what order:

```json
{
  "name": "research_pipeline",
  "steps": ["researcher", "summarizer", "nova"],
  "max_iterations": 3
}
```

### Delivery Semaphore
Per-target delivery lock — ensures only one message is in flight to each agent at a time. Prevents race conditions in sequential workflows.

---

## Token System

Each agent gets a unique token stored as bcrypt hash:

```javascript
// Register agent → get token
POST /agents/register { agent_id }
→ { token: "raw-token-shown-once" }

// Token stored as bcrypt hash in com_hub.db
// Agent uses raw token in every message
// Router verifies: bcrypt.compare(token, hash)
```

---

## Routing Targets

```
bridge-connector   ← Nova LLM pipeline
claude-code        ← Claude Code (via wake socket)
nova-core          ← Main panel (via core socket)
[any addon]        ← Via addon Unix socket
```

---

## Tech Stack

- Node.js
- SQLite (com_hub.db — agents, tokens, message log)
- bcryptjs (token hashing)
- Unix Sockets (inter-process communication)
- UUID (message and chain IDs)

---

## Part of Nova

Nova Com Hub is one component of **Nova** — an autonomous AI assistant system.

🌐 [neural-werk.de](https://neural-werk.de) · [Nova Platform](https://github.com/nico-gordon-gilles/nova-platform) · [Autonomi](https://github.com/nico-gordon-gilles/nova-autonomi)

