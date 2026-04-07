<!-- 00_foundations/state_vs_memory_vs_context.md -->

# State vs Memory vs Context in Artificial Systems

## Problem

Agent architectures conflate state, memory, and context. This produces systems that fail unpredictably, leak information across sessions, and cannot scale beyond single-interaction exchanges.

The confusion manifests in implementation choices. Conversation history gets treated as memory. Vector database retrieval gets called semantic recall. Session state persists across user boundaries. Context windows become dumping grounds for everything an agent might need.

These are not semantic disagreements. They create architectural debt. When state is stored as memory, agents cannot clean up after themselves. When memory is built into context, systems hit token limits and lose critical information. When context construction has no defined boundaries, debugging becomes archeology.

This essay provides precise definitions and proposes clear interface boundaries for state, memory, and context in agent systems.

## Definitions

State is the current snapshot of system execution. It captures what is true right now: active variables, intermediate computation results, authentication credentials, sensor readings. State has a bounded lifetime, typically session-scoped or task-scoped. When the session ends, state should be discarded or explicitly promoted to memory.

Context is the information assembled for a specific decision point. It is constructed on-demand from state, memory, and external sources. Context does not persist. It exists to inform a single inference or action selection, then dissolves. The same query at different times may produce different contexts as state and memory change.

Memory is information that survives across sessions and influences future behavior. It is persistent, typically append-only or versioned. Memory accumulates over time and requires explicit management: consolidation, retrieval strategies, forgetting policies. Unlike state, memory is never complete; retrieval is selective based on relevance.

| | Persistence | Mutability | Access Pattern | Failure Mode | Security Concerns |
|---|---|---|---|---|---|
| State | Session/task-scoped | High-frequency writes | Read/write on every step | Stale state causes incorrect actions | Session leakage across users |
| Context | Transient (single inference) | Read-only (assembled, not stored) | Constructed per query | Missing information causes hallucination | Prompt injection via context manipulation |
| Memory | Cross-session, indefinite | Append-preferred, versioned updates | Write occasional, read selective | Wrong retrieval causes inconsistent behavior | Privacy violations from persistent user data |

## Existing Approaches

Most agent frameworks treat these three concepts as implementation details rather than architectural boundaries.

Stateless API Pattern: LLM providers expose stateless endpoints. The application layer manages conversation history and passes it as context. This works for isolated interactions but breaks for multi-turn tasks that require tracking intermediate state. Solutions bolt on external session stores without clear contracts for what belongs in state vs memory.

Conversation-as-Memory: Frameworks store entire conversation transcripts and retrieve relevant segments. This treats episodic interactions as equivalent to semantic memory. It conflates the record of what was said (state history) with what was learned (memory). Retrieval is keyword-based or embedding-similarity based, not structured by memory type or consolidation.

Vector Database Retrieval: Agents store experiences or facts as embeddings and retrieve via similarity search. This is positioned as "giving agents memory" but provides no mechanisms for temporal structure, consolidation, or forgetting. Every retrieval is independent; there is no notion of memory strength or interference.
