# cadence-format
A minimal format for representing coordinated action. Two elements — actions and connections — describe any workflow, process, or coordination pattern. Human-readable, machine-parseable.
_____
A **cadence** is a container for coordinated action toward an outcome. It is made of **actions** and **connections**. Nothing else is required.

The Cadence Format describes the structure of coordinated action. It does not specify how cadences are created, executed, stored, or rendered. Those are concerns of the systems built on the format. Time, duration, and deadlines are system concerns — the format represents the structure of coordination, not the speed of it. The format represents completion and non-completion of actions but does not distinguish between types of non-completion. Systems that need failure semantics implement them above the format.

---

## Cadence

A cadence needs two things:

```json
{
  "cadence": {
    "name": "Process a request",
    "trigger": "Request received"
  },
  "actions": [],
  "connections": []
}
```

`name` — what this cadence is called.

`trigger` — what starts it. An event, a schedule, a condition, or a manual decision.

Optionally, a cadence can carry:

- `id` — so other systems can reference it. When cadences are exchanged between systems, `id` is required.
- `purpose` — what this cadence achieves or maintains, as distinct from its name
- `format_version` — the version of the Cadence Format this cadence conforms to

---

## Actions

An action is the basic unit of a cadence. An **actor** performs an action, and something changes.

```json
{ "id": "a1", "name": "Review the request", "actor": "Team A" }
```

An action has two parts:

`name` — what happens. A verb and an object:

- "Prepare the materials"
- "Review the request"
- "Approve the result"
- "Deliver the output"

`actor` — who or what performs it. A persona, a role, an agent, a team, a machine, etc. The actor represents the entity responsible for performing the action, as described by the cadence author.

An action can optionally carry a `description` for additional context. But name and actor are all it needs to exist.

An action can optionally carry an `expands_to` field — the ID of another cadence that describes this action in full detail:

```json
{ "id": "a4", "name": "Execute the project", "actor": "Team B", "expands_to": "cad_detailed_project" }
```

This is how nesting works at the format level. The referenced cadence is a full cadence with its own actions and connections. How the system retrieves it is a system concern.

An action is considered complete when the system representing it determines it has finished. The format does not prescribe how completion is determined.

---

## Connections

A connection links one action to another.

```json
{ "id": "c1", "from": "a1", "to": "a2" }
```

`from` — the source action.

`to` — the target action.

When the source action completes, the target action can begin. A connection defines a dependency: the target action must not begin before the source action completes. A connection with no other fields is unconditional — the target can begin once the source completes.

When an action has multiple incoming connections, all are dependencies — the action cannot begin until all source actions have completed. Systems may implement alternative semantics, but the default is all-required.

A connection can optionally carry a `condition`:

```json
{ "id": "c2", "from": "a1", "to": "a3", "condition": "Request denied" }
```

When a condition is present, the target action can only begin when the source action completes and the condition is met. The format records that a condition exists, not how it's checked. One condition per connection. Multiple connections from the same source action, each with a different condition, create branching.

A connection can optionally carry a `mechanism`:

```json
{ "id": "c1", "from": "a1", "to": "a2", "mechanism": "Summary forwarded with a recommendation" }
```

A mechanism describes how context transfers between actors. It is only relevant when the source and target actions have different actors — a **handoff**. When a mechanism is missing on a handoff, that is a **gap**.

An action with no outgoing connections is a terminal action. A cadence ends when a terminal action completes. When a cadence has multiple terminal actions across parallel paths, all must complete for the cadence to end.

---

## Nesting

A cadence can contain actions that themselves expand into full cadences. An action with an `expands_to` field references another cadence by ID. That cadence describes the action in full detail — more actions, more connections, finer granularity.

Same format at every level. Only the resolution changes. This is how cadences scale without changing structure. A cadence with `expands_to` references is not self-contained. Systems that export cadences for portability should bundle or inline referenced cadences.

---

## Gaps

A cadence does not have to be complete to be useful.

An action without an actor has a gap. A handoff without a mechanism has a gap. A branch with only one path defined has a gap. These are signals of incompleteness, not errors. The format permits empty fields. Incomplete cadences are valid.

---

## Emergent Patterns

Two elements produce every structural pattern without additional building blocks:

- **Sequence** — A connects to B
- **Branching** — A connects to B (condition X) and C (condition Y)
- **Parallel** — A connects to B and C, both unconditional
- **Convergence** — B and C both connect to D
- **Loops** — D connects back to A, with a condition
- **Handoffs** — any connection where the actor changes

These patterns emerge from combining actions and connections.

---

## Design Principles

**Two elements are sufficient.** If something can't be expressed as an action or a connection, it doesn't go in the format. Complexity enters through enrichment of existing elements, not through new ones.

**Enrichment doesn't change structure.** A cadence with two actions and one connection has the same structure as one with fifty actions and forty connections. The first is sparse. The second is rich. Neither is more structurally valid.

**Incomplete is valid.** The format permits empty fields and undefined paths. A cadence with gaps is still a cadence.

**The format is the product.** A cadence is portable, shareable, and forkable. It can be exported, imported, versioned, served via API, and rendered visually. The product is a way of creating, viewing, and operating on the format.

---

## Examples

Each example builds on the last, introducing one new concept.

**Sequence** — three actions, two connections.

```json
{
  "cadence": { "name": "Process a request", "trigger": "Request received" },
  "actions": [
    { "id": "a1", "name": "Review the request", "actor": "Team A" },
    { "id": "a2", "name": "Fulfill the request", "actor": "Team A" },
    { "id": "a3", "name": "Confirm completion", "actor": "Team A" }
  ],
  "connections": [
    { "id": "c1", "from": "a1", "to": "a2" },
    { "id": "c2", "from": "a2", "to": "a3" }
  ]
}
```

**Branching** — conditions on connections create a decision.

```json
{
  "cadence": { "name": "Process a request", "trigger": "Request received" },
  "actions": [
    { "id": "a1", "name": "Review the request", "actor": "Team A" },
    { "id": "a2", "name": "Fulfill the request", "actor": "Team A" },
    { "id": "a3", "name": "Return the request", "actor": "Team A" }
  ],
  "connections": [
    { "id": "c1", "from": "a1", "to": "a2", "condition": "Approved" },
    { "id": "c2", "from": "a1", "to": "a3", "condition": "Denied" }
  ]
}
```

**Handoff with mechanism** — the actor changes, and the mechanism captures how.

```json
{
  "cadence": { "name": "Process a request", "trigger": "Request received" },
  "actions": [
    { "id": "a1", "name": "Review the request", "actor": "Team A" },
    { "id": "a2", "name": "Approve the request", "actor": "Team B" },
    { "id": "a3", "name": "Fulfill the request", "actor": "Team A" }
  ],
  "connections": [
    { "id": "c1", "from": "a1", "to": "a2", "mechanism": "Summary forwarded with a recommendation" },
    { "id": "c2", "from": "a2", "to": "a3" }
  ]
}
```

Note that c2 is also a handoff — Team B to Team A — but has no mechanism. That's a gap.

**Loop** — a backward connection with a condition creates a cycle.

```json
{
  "cadence": { "name": "Process a request", "trigger": "Request received" },
  "actions": [
    { "id": "a1", "name": "Prepare the response", "actor": "Team A" },
    { "id": "a2", "name": "Review the response", "actor": "Team B" },
    { "id": "a3", "name": "Deliver the response", "actor": "Team A" }
  ],
  "connections": [
    { "id": "c1", "from": "a1", "to": "a2" },
    { "id": "c2", "from": "a2", "to": "a3", "condition": "Approved" },
    { "id": "c3", "from": "a2", "to": "a1", "condition": "Revisions needed" }
  ]
}
```

Each example uses the same two elements. Structural complexity increases through combination, not through new elements.

---

## Extensions

The core format is actions and connections. Extensions enrich cadences without altering the core structure. They are not required.

### Objects

Sometimes the same object moves through a cadence and changes state along the way. When multiple actions reference that object, or its state determines which connections are met, it can be tracked explicitly.

```json
{
  "id": "obj1",
  "name": "The request",
  "states": ["received", "in progress", "completed"],
  "transitions": [
    { "action": "a1", "to_state": "in progress" },
    { "action": "a3", "to_state": "completed" }
  ]
}
```

`name` — what the object is.

`states` — the ordered progression of states the object can move through. The first element is the initial state.

`transitions` — which actions change the object's state, and to what state.

Objects are only needed when an object's journey through the cadence must be tracked explicitly. Conditions and object states may correspond, but the format does not require or enforce a link between them.

---

## Terminology

Quick reference for terms defined above.

**Action** — the basic unit of a cadence. A named change performed by an actor. Completion is determined by the implementing system, not the format.

**Actor** — the entity responsible for performing an action, as described by the cadence author. A persona, role, agent, team, or machine.

**Cadence** — a container for coordinated action toward an outcome, made of actions and connections.

**Condition** — a requirement on a connection that gates the dependency. The format records that a condition exists, not how it's checked.

**Connection** — a dependency between two actions. The target action must not begin before the source action completes. Optionally gated by a condition.

**Convergence** — when an action has multiple incoming connections. By default, all source actions must complete before the target can begin.

**Gap** — a missing but expected piece of information in a cadence. An action without an actor, a handoff without a mechanism, a branch with undefined paths.

**Handoff** — a connection where the actor changes between the source and target actions, requiring context to transfer.

**Mechanism** — a description of how context transfers during a handoff.

**Object** — an extension. An optional named entity that moves through a cadence and changes state across multiple actions. Tracked through transitions that tie state changes to specific actions.

**Trigger** — what initiates a cadence. An event, schedule, condition, or manual decision.
