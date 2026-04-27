# Protocols Are Not Capabilities

---

## 1. Problem Framing

Agent systems are increasingly described through the protocols they support. MCP, REST, WebSockets, ROS topics, function calling, plugin interfaces. Once a system can speak a protocol, it is often treated as if it has acquired the capability that rides on top of that protocol.

This is a category error.

A protocol defines how components talk. A capability defines what the system can actually do. Confusing the two leads to brittle architectures, misleading product claims, and agents that appear powerful in demos but fail in deployment.

An agent with access to an MCP server does not thereby "have LinkedIn." It has a communication path to a tool that may or may not support LinkedIn operations, may or may not be reliable, and may or may not be safe to use under real operational constraints.

This distinction matters beyond marketing language. It changes how systems should be designed, evaluated, and trusted.

---

## 2. Current Approach

Most modern agent stacks are assembled in layers.

At the bottom sits some execution substrate: an API, a database, a browser session, a robotics controller, a filesystem, a message bus. Above that sits an interface layer that standardizes access: an SDK, a tool schema, an RPC contract, or a protocol such as MCP. The language model or planner sits above that and decides when to call which tool.

This layering is useful. Standardized interfaces reduce integration cost. A model can call many tools through one abstraction. Tool descriptions can be surfaced as natural-language affordances. Entire ecosystems can grow around the protocol boundary.

The mistake appears when the interface layer is treated as the source of capability rather than the transport for it.

Three examples:

1. **MCP** gives an agent a standard way to discover and call tools. It does not ensure the tool is legitimate, stable, low-latency, policy-compliant, or even semantically correct.
2. **Function calling** gives a model a structured output path. It does not guarantee that the backed function has the permissions, state, or external access required to complete the task.
3. **ROS topic access** gives an agent a way to publish and subscribe. It does not mean the robot exposes the right control surfaces for navigation, manipulation, or safety supervision.

In all three cases, the protocol is real. The implied capability is conditional.

---

## 3. Where This Breaks

### Capability inflation

Once an agent can call a tool through a standardized protocol, product language tends to collapse the stack. "The agent can post on LinkedIn." "The agent can operate the robot." "The model can use company memory." Often what is actually true is narrower: the agent can call a wrapper that attempts a posting endpoint, or publish a velocity command, or retrieve text fragments from a store.

This inflation is not harmless. It causes operators to over-trust the agent and under-specify failure handling.

### Hidden dependency risk

Protocols hide heterogeneity. Two MCP servers may expose identically named tools with radically different trust properties. One may wrap an official API. Another may rely on scraped session cookies and a fragile reverse-engineered endpoint. At the protocol boundary they look interchangeable. Operationally they are not.

### Evaluation mismatch

Systems are often evaluated at the protocol layer rather than the capability layer. Teams verify that a tool call succeeds syntactically, that JSON is returned, that the model selected the right function. They do not verify whether the external action succeeded in the world, whether the returned data is authoritative, or whether side effects were acceptable.

A robot illustrates this cleanly. Publishing to `/cmd_vel` proves only that a message was sent. It does not prove the path is safe, the localization is valid, the controller is calibrated, or the robot achieved the task.

### Missing policy boundaries

Capability is not merely access. It is access under constraints. A system that can technically send a message but lacks rate limits, approval gates, audit logs, or identity boundaries does not possess a deployable messaging capability. It possesses a raw primitive.

Protocol-level integration often arrives before policy-level design. This is why many agent systems can technically act before they can safely act.

### Wrong abstraction for debugging

When failures occur, teams often inspect the protocol trace: was the tool discovered, did the arguments validate, did the server respond. These are useful questions, but they are upstream of the real issue. The deeper questions are capability questions: was the underlying source authoritative, was the external state current, was the action semantically appropriate, did the environment match the assumptions embedded in the tool.

---

## 4. Deeper Abstraction

The clean separation is this:

- **Protocol**: how a request is expressed and transmitted.
- **Tool**: a callable interface exposed to the agent.
- **Capability**: the real-world operation the system can perform through that tool.
- **Policy**: the conditions under which that capability may be exercised.
- **Guarantee**: the reliability, observability, and safety properties attached to that capability.

These are distinct layers. Collapsing them hides design obligations.

A useful mental model is a five-stage stack:

```
Model/Planner
    ↓
Protocol Boundary (MCP / function call / RPC)
    ↓
Tool Adapter
    ↓
Operational Substrate (API / browser / controller / database)
    ↓
World Effect
```

Capability lives across the lower three layers, not at the protocol boundary alone.

For example, "post to LinkedIn" is not an MCP capability. It is a world effect whose success depends on:

- the tool adapter exposing a post action
- the backing service having valid auth
- the backing path actually supporting posting
- rate limits and policy constraints being handled
- the post surviving platform validation
- confirmation that the post exists after the write

The same logic applies to robotics. "Pick and place an object" is not a ROS capability because ROS messages exist. It is a composite capability requiring frame definitions, controller access, tool-center-point assumptions, calibration, collision boundaries, execution monitoring, and recovery behavior.

Protocols make capabilities accessible. They do not create them.

---

## 5. Design Implications for Agent Systems

### Evaluate capabilities, not integrations

A tool should not be considered "working" because it can be called. It should be considered working only if the underlying task succeeds under realistic constraints and failure conditions. Evaluation must extend from protocol correctness to world-level correctness.

### Encode guarantees with the tool

Tool metadata should describe more than parameters. It should encode operational properties:

- authority level of the data source
- expected latency
- whether the action is read-only or mutating
- approval requirements
- retry safety
- confidence of post-condition verification

Without this, the planner reasons over tools as if they are all equivalent.

### Separate raw primitives from deployable capabilities

"Send HTTP request" is a primitive. "Send customer-facing LinkedIn outreach" is a capability. Between them sit templates, personalization logic, compliance rules, pacing constraints, logging, and human approval in many domains. Agent architectures should model this middle layer explicitly.

### Make post-conditions first-class

A capability should define how success is verified. Did the profile search return authoritative results or guessed matches. Did the robot reach the target pose or merely accept the command. Did the CRM update persist and round-trip correctly. If the system cannot verify the world effect, it should report tentative completion, not success.

### Design around capability envelopes

Every tool has an operating envelope. A browser-backed LinkedIn tool may support low-volume supervised actions but not unattended industrial-scale outreach. A robot exposed only through velocity commands may support teleoperation but not task-level autonomy. Capabilities should be described with their envelope, not with their maximal theoretical interpretation.

### Treat protocols as replaceable

If the architecture is sound, the same capability may be carried through different protocols. MCP today, direct API tomorrow, internal RPC next quarter. This is only possible if the system models the capability separately from the transport.

---

## 6. Open Questions

1. How should agent frameworks represent capability guarantees in machine-readable form so planners can reason over trust, latency, and mutability?
2. What is the right abstraction for post-condition verification when the world state is only partially observable?
3. How should protocols surface provenance so agents can distinguish official APIs from reverse-engineered or browser-driven backends?
4. Can policy constraints be expressed as first-class capability metadata rather than ad hoc prompt instructions?
5. What evaluation benchmarks would measure capability success instead of tool-call success?
6. How should planners trade off a low-authority but available tool against a high-authority but slower or more expensive one?
7. What is the right way to compose primitives into higher-level capabilities without hiding the associated safety boundaries?
8. How should embodied systems express capability envelopes when available control surfaces are incomplete or partially trustworthy?
9. Can agent architectures automatically detect when protocol availability is being mistaken for real-world capability?
10. What design patterns best preserve transport independence while still allowing planners to exploit protocol-specific affordances?
