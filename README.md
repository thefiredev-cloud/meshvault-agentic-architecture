# MeshVault agentic architecture

A public reference for the mixture-of-agents and role-based orchestration patterns behind MeshVault.

> **Status:** architecture reference for a pre-release product. The MeshVault iOS secure agent messenger and macOS agentic browser are in design/build. This repository does not claim public availability.

## Why this exists

A useful agent system needs more than a large model. It needs several kinds of judgment, a single accountable operator, durable context, constrained tools, and explicit gates before real-world side effects.

MeshVault combines two orchestration layers:

1. **Mixture of Agents (MoA)** gives the acting agent diverse model advice before it decides what to do.
2. **Role orchestration** lets Chief delegate bounded work to five employee specialists and return one verified answer to the owner.

The layers complement each other. MoA improves reasoning. Role orchestration improves ownership, scope, and operational accountability.

## System view

```mermaid
flowchart TB
    Owner["Owner"]
    Channels["Owner interfaces\nmessaging gateway · planned iOS messenger · planned macOS browser"]
    Chief["Chief\nowner-facing coordinator"]

    subgraph Reasoning["Reasoning plane"]
        MoA["Hermes MoA"]
        RefA["Reference advisor A"]
        RefB["Reference advisor B"]
        Agg["Acting aggregator"]
        RefA --> MoA
        RefB --> MoA
        MoA --> Agg
    end

    subgraph Specialists["Role plane"]
        Alex["Alex\nsales & outreach"]
        Casey["Casey\noperations & research"]
        Jordan["Jordan\nclient success"]
        Morgan["Morgan\nmarketing & content"]
        Riley["Riley\nengineering"]
    end

    subgraph OwnedStack["Owned data and action plane"]
        Vault["Private knowledge vault\nplain Markdown"]
        Graphify["Graphify\nknowledge graph + recall"]
        Tools["Tool layer\nfiles · browser · APIs · code"]
        Models["Model router\nlocal + approved cloud"]
        Audit["Append-only audit trail"]
        Gate{"Approval gate"}
    end

    Owner --> Channels --> Chief
    Chief <--> Agg
    Chief <--> Alex
    Chief <--> Casey
    Chief <--> Jordan
    Chief <--> Morgan
    Chief <--> Riley

    Chief <--> Vault
    Vault <--> Graphify
    Chief <--> Models
    Chief --> Tools --> Gate
    Gate -->|approved| Audit
    Gate -->|not approved| Chief
```

## Hermes MoA loop

Hermes implements MoA as a virtual provider around the normal agent loop. Reference models are advisors: they receive a trimmed, text-only view of the current task state and cannot call tools. Their responses run in parallel, then the acting Hermes aggregator incorporates their private guidance and continues the normal agent and tool loop.

The aggregator is the sole acting model and retains Hermes’s normal tool loop. This separation matters: many models can advise, but only one agent owns the task and its side effects.

```mermaid
sequenceDiagram
    autonumber
    participant U as Owner
    participant A as Acting Hermes aggregator
    participant R1 as Reference advisor 1
    participant R2 as Reference advisor 2
    participant T as Tools
    participant G as Approval gate

    U->>A: Request
    par Parallel advisory fan-out
        A->>R1: Text-only task state
        R1-->>A: Analysis and risks
    and
        A->>R2: Text-only task state
        R2-->>A: Alternative analysis
    end
    A->>A: Incorporate private advisor guidance
    A->>T: Read, inspect, calculate, or build
    T-->>A: Tool result

    opt Preset cadence is per_iteration
        par Refresh advisory context
            A->>R1: Current iteration state
            R1-->>A: Updated advice
        and
            A->>R2: Current iteration state
            R2-->>A: Updated advice
        end
        A->>A: Incorporate refreshed guidance
    end

    alt Side effect requested
        A->>G: Stage action
        G-->>U: Request explicit approval
        U-->>G: Approve or reject
        G-->>A: Decision
    else Read-only work
        A-->>U: Verified result
    end
```

### Runtime properties

- Reference calls are independent and fan out concurrently, with a bounded worker pool.
- Advisors cannot use tools and do not act on the user's behalf.
- The aggregator is the acting model and retains Hermes's normal tool loop.
- If an advisor call fails, the failure is labeled in the aggregation context, successful advisor responses remain usable, and the turn continues with partial advice.
- Presets support `user_turn` cadence, which runs advisory fan-out once for each user turn, and `per_iteration` cadence, which runs advisory fan-out on each acting-agent loop iteration.
- Tool results shown to advisors are head-and-tail previews; the acting agent keeps the full transcript.
- Recursive MoA presets are rejected.
- Full MoA traces are opt-in, not automatic.
- Token usage and estimated cost are accounted for per advisor and aggregator model.

## Role orchestration

Chief is the sole owner-facing coordinator. The five employee agents are bounded workers, not independent public personas and not competing orchestrators.

```mermaid
flowchart LR
    Request["Owner request"] --> Classify{"Chief classifies scope"}

    Classify -->|sales or outreach| Alex["Alex"]
    Classify -->|operations, research, finance| Casey["Casey"]
    Classify -->|client success| Jordan["Jordan"]
    Classify -->|marketing or content| Morgan["Morgan"]
    Classify -->|engineering| Riley["Riley"]
    Classify -->|cross-functional| Fanout["Bounded parallel delegation"]

    Fanout --> Alex
    Fanout --> Casey
    Fanout --> Jordan
    Fanout --> Morgan
    Fanout --> Riley

    Alex --> Verify["Chief verifies evidence"]
    Casey --> Verify
    Jordan --> Verify
    Morgan --> Verify
    Riley --> Verify

    Verify --> Risk{"Side-effect level"}
    Risk -->|read or analyze| Return["One synthesized answer"]
    Risk -->|draft| Stage["Stage for owner review"]
    Risk -->|send · publish · pay · delete · account change| Approval["Explicit current-task approval"]
    Approval -->|approved| Execute["Execute + verify"]
    Approval -->|rejected| Return
    Execute --> Return
```

### Specialist contract

| Role | Primary scope | Default boundary |
|---|---|---|
| Chief | Coordination, verification, owner communication | One accountable final answer |
| Alex | Sales and outreach | Draft or stage outbound work unless approved |
| Casey | Operations, research, finance | Evidence-first; money movement remains gated |
| Jordan | Client success | Customer context with outbound actions gated |
| Morgan | Marketing and content | Draft freely; publishing remains gated |
| Riley | Engineering | Build and verify; deployments and destructive changes remain gated |

Employee responses are treated as untrusted input until Chief verifies them against live state or primary sources.

## Context and memory

```mermaid
flowchart LR
    Event["Conversation · file · tool result"] --> Capture["Capture durable signal"]
    Capture --> Vault["Private knowledge vault\nplain Markdown"]
    Vault --> Graph["Graphify index"]
    Query["New owner request"] --> Recall["Graph-first recall"]
    Graph --> Recall
    Recall --> Context["Bounded relevant context"]
    Context --> Agent["Chief / specialist / MoA advisor"]
    Agent --> Evidence["Live source verification"]
    Evidence --> Answer["Grounded answer or staged action"]
```

The vault is the durable source of knowledge. Graphify narrows retrieval before raw documents are opened. Live systems remain the source of truth for current account, service, deployment, and network state.

## Approval model

| Level | Example | Default behavior |
|---|---|---|
| L0 | Read, search, inspect, calculate | May run automatically |
| L1 | Draft, plan, summarize | May produce a staged artifact |
| L2 | Send, publish, deploy, change an account | Requires explicit current-task approval |
| L3 | Payment, destructive action, security/credential change | Requires deliberate approval and post-action verification |

The key rule is simple: delegation does not transfer authority. Every worker inherits the same action boundary as the coordinator.

## Trust boundaries

```mermaid
flowchart TB
    Public["Public internet"]
    PrivateMesh["Private mesh boundary"]
    Client["Owner devices"]
    Node["MeshVault node"]
    Data["Vault + graph + audit"]
    Model["Local models"]
    Cloud["Approved cloud models"]
    Actions["External services"]

    Public -. no direct trust .-> PrivateMesh
    Client <--> PrivateMesh <--> Node
    Node <--> Data
    Node <--> Model
    Node -->|minimum necessary context| Cloud
    Node -->|approval-gated calls| Actions
```

Design principles:

- Private mesh before public exposure.
- Local-first data and inference where practical.
- Minimum necessary context for cloud model calls.
- Explicit approval before external side effects.
- Append-only audit evidence for actions.
- One accountable coordinator even when many models or specialists contribute.

## What MoA is good for

MoA is useful when the cost of a shallow answer is higher than the cost of additional inference:

- architecture and security reviews
- ambiguous engineering decisions
- cross-functional plans
- adversarial critique before a high-impact action
- synthesis where models have meaningfully different strengths

It is a poor default for trivial lookups, deterministic calculations, or urgent low-risk tasks. More agents can add cost and correlated noise without adding insight.

## Current implementation notes

This public reference describes the intended and partially implemented system:

- Hermes MoA is a real runtime capability.
- Chief and five bounded employee roles are operational orchestration concepts used by MeshVault.
- MeshVault Vault, Graphify, tool routing, and approval gates are active implementation areas.
- The paired customer-facing iOS and macOS apps remain pre-release.
- Model names and providers are deployment choices, not architectural dependencies.

## Decision record

The architecture deliberately uses **one acting coordinator with advisory fan-out**, rather than several autonomous writers.

Why:

1. A single actor preserves a clear audit trail.
2. Advisory models can disagree without racing to mutate state.
3. Tool authority stays in one policy boundary.
4. Specialist work can be parallelized while final verification remains centralized.
5. Failure degrades to partial advice instead of conflicting side effects.

The trade-off is added latency and model cost. MeshVault therefore reserves MoA for decisions where diversity of judgment is worth paying for.

## Source basis

This reference is grounded in:

- Hermes MoA runtime behavior
- MeshVault’s Chief/employee role orchestration design
- the MeshVault owned-stack orchestration and approval model

No credentials, private paths, internal addresses, customer data, or proprietary implementation details are included.

## License

MIT. See [LICENSE](LICENSE).
