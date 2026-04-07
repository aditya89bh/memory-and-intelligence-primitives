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

Database-Backed State: Agents persist state to databases (Redis, PostgreSQL) to survive process restarts. This is correct for state management but often mixes concerns. User preferences, learned behaviors, and active session data all end up in the same store with no lifecycle distinctions.

## Critique

### Where Approaches Fail

Current approaches fail to enforce boundaries. State leaks into memory when session data is never cleaned up. Memory leaks into context when entire knowledge bases are dumped into prompts. Context leaks into state when intermediate inference results are cached without expiration policies.

The root failure is treating data movement as the solution. If state management is a problem, add a database. If memory is needed, add a vector store. If context is insufficient, increase the window size. None of these address the underlying issue: undefined interfaces between components.

When state is lost, agents repeat work or lose track of progress. When memory retrieval fails, agents exhibit inconsistent behavior across sessions. When context is poorly constructed, agents hallucinate or ignore critical information. These are not edge cases; they are the dominant failure modes in production agent systems.

### Why Vector DB Retrieval Is Not Equivalent to Memory

Vector databases provide similarity-based retrieval over embeddings. This is a search primitive, not a memory system.

Human memory exhibits temporal dynamics. Recent memories are more accessible. Related memories prime each other. Consolidation moves information from episodic to semantic storage. Forgetting is selective, not random eviction based on storage limits.

Vector retrieval has none of this. Every query is independent. There is no memory strength, no priming, no consolidation pathway. Similarity search finds nearest neighbors in embedding space, which may correlate with semantic relatedness but does not implement recall.

Memory systems require write policies (what to store), consolidation policies (how to merge/strengthen), retrieval policies (what to surface when), and forgetting policies (what to discard and when). Vector databases provide storage and retrieval. The policies must be implemented in the layer above, and most agent frameworks do not implement them at all.

Calling vector retrieval "memory" is like calling a filesystem "a database" because both persist data. The abstraction level is wrong.

## Proposed Abstraction

Define three interfaces with clear contracts.

### StateStore

```python
class StateStore:
    def write(self, key: str, value: Any, ttl: Optional[int] = None) -> None:
        """Write state for current session/task.

        TTL in seconds. If None, expires at session end.
        """

    def read(self, key: str) -> Optional[Any]:
        """Read state. Returns None if expired or not found."""

    def clear_session(self) -> None:
        """Explicitly discard all session state. Called at session termination."""
```

Invariants:
- State expires. Either TTL-based or session-based.
- State is mutable. Overwriting is expected.
- State is isolated per session/user.

Never store in StateStore:
- User preferences that should persist across sessions
- Learned behaviors or patterns
- Historical interaction logs

### ContextBuilder

```python
class ContextBuilder:
    def build(
        self,
        query: str,
        state: Dict[str, Any],
        memory_refs: List["MemoryReference"],
    ) -> str:
        """Assemble context for a single inference.

        Deterministic given inputs. Returns formatted string for LLM prompt.
        """
```

Invariants:
- Context is constructed, not stored.
- Same inputs produce same context (deterministic).
- Context is read-only from the agent's perspective.

Never store in ContextBuilder:
- Nothing. This is a builder, not a store.

### MemoryStore

```python
class MemoryStore:
    def write(
        self,
        content: str,
        memory_type: "MemoryType",  # episodic, semantic, procedural
        metadata: Dict[str, Any],
        timestamp: "datetime",
    ) -> "MemoryID":
        """Append new memory. Returns ID for future reference."""

    def retrieve(
        self,
        query: str,
        memory_type: Optional["MemoryType"] = None,
        limit: int = 10,
        temporal_weight: float = 0.5,
    ) -> List["Memory"]:
        """Retrieve relevant memories.

        Combines similarity, recency, access frequency.
        """

    def consolidate(self, policy: "ConsolidationPolicy") -> None:
        """Background process: merge related memories, strengthen frequently accessed, mark for forgetting."""
```

Invariants:
- Memories persist across sessions.
- Writes are append-preferred (versioned updates, not overwrites).
- Retrieval is selective and policy-driven.
- Consolidation and forgetting are explicit operations.

Never store in MemoryStore:
- Current sensor readings or intermediate computation (use StateStore)
- Single-use context assembled for one query (use ContextBuilder)

## Examples

### Tool-Using Assistant (Email/Calendar/Browser)

State: OAuth tokens for Google/Microsoft APIs. Current email draft being composed. Active browser session cookies. Tool invocation results from the last step.

Context: User's current query. Relevant email thread snippets (retrieved from memory). Calendar availability for the next week (fetched via tool). Tool schemas for available actions.

Memory: User's communication preferences (formal vs casual tone). Frequently contacted people and their roles. Past task patterns (user always schedules meetings for Tuesday mornings). Learned tool usage strategies (always check calendar before proposing meeting times).

State is discarded when the task completes. Context is rebuilt for each user message. Memory persists and influences future interactions.

### Robotics Agent (Task Execution + Sensor Stream)

State: Current joint positions and velocities. Active trajectory plan. Immediate sensor readings (force/torque, camera frames). Collision avoidance flags.

Context: Task specification (pick object X, place at location Y). Object models relevant to current manipulation. Safety constraints for the workspace. Retrieved manipulation strategies from memory.

Memory: Learned grasp strategies for object categories. Workspace layout and obstacle positions (updated slowly). Failure cases and recovery procedures. Calibration parameters for sensors.

State is updated at control loop frequency (10-100 Hz). Context is assembled when planning a new action sequence. Memory is written after task completion and retrieved during planning.

## Open Research Questions

1. What is the optimal consolidation policy for episodic-to-semantic memory transfer in agents that operate over months or years?
2. How should memory retrieval balance similarity, recency, and access frequency when these metrics conflict?
3. Can we define formal invariants for state/memory boundaries that enable automatic verification or testing?
4. What memory types beyond episodic/semantic/procedural are necessary for embodied agents in dynamic environments?
5. How do we handle memory versioning when an agent's understanding of a concept changes over time?
6. What is the right abstraction for forgetting policies that preserve critical information while managing storage and retrieval costs?
7. Should context construction be deterministic or stochastic, and how does this choice affect agent reliability?
8. How do we implement memory priming (related memories activating each other) in retrieval systems without quadratic complexity?
9. What are the failure modes when state accidentally persists across sessions, and can we detect this automatically?
10. How should multi-agent systems share memory without leaking state or violating privacy boundaries?
