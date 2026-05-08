# System Primitives for Intelligence

*Part 4 of a series on cognitive architectures for agent systems.*
*Part 1: [Why Vector Databases Are Not Memory](./why-vector-databases-are-not-memory.md)*
*Part 2: [Cognitive Loops in Agents](./cognitive-loops-in-agents.md)*
*Part 3: [Memory Agents: The Architecture Nobody Has Fully Specified](./memory-agents-architecture.md)*

---

## 1. Problem Framing

The previous essays in this series identified specific architectural failures: memory systems that store without consolidating, loops that iterate without closing, agents that retrieve without learning. Each critique pointed at the same underlying problem from a different angle — current agent systems are built top-down. Capabilities are added as features. Components are integrated as modules. The result is a layered stack of approximations that behaves intelligently in narrow conditions and fails structurally outside them.

The alternative is to start from primitives.

A primitive, in this context, is not a simple thing. It is an irreducible functional unit — a capability that cannot be decomposed further without losing the property it provides. Primitives compose into mechanisms. Mechanisms compose into architectures. If the primitives are wrong or absent, no amount of architectural sophistication recovers what they were supposed to provide.

This essay specifies three primitives that are foundational to any system that can reasonably be called intelligent: memory, attention, and planning. Each is defined by its functional requirements, not by its implementation. The definitions are intended to be implementation-agnostic and precise enough to evaluate any given system against them.

The central claim is this: most current AI agent systems do not have clean implementations of any of these three primitives. They have approximations that work within specific constraints and degrade outside them. Building capable agent systems requires identifying those approximations, understanding where they break, and replacing them with implementations that satisfy the primitive's functional requirements.

---

## 2. What Makes Something a Primitive

Before specifying the three primitives, the term needs a precise definition.

A system primitive is a functional unit that satisfies three conditions. First, it is necessary: a system that lacks it cannot exhibit the class of behaviors that depend on it, regardless of how its other components are implemented. Second, it is irreducible: removing any part of its specification eliminates the property it provides. Third, it has defined interfaces: other system components interact with it through typed inputs and outputs, not through side effects or shared state mutations.

These conditions exclude many things that are called primitives in practice. A transformer attention mechanism is not a primitive under this definition — it is one implementation of a broader functional requirement. A key-value cache is not a primitive — it is a storage mechanism that may or may not satisfy the requirements of a memory primitive depending on how it is used. The primitive is the functional specification. The implementation is what satisfies it.

This distinction matters because conflating implementation with primitive produces systems that inherit the constraints of a specific implementation when they could have satisfied the functional requirement differently. Treating vector similarity search as the memory primitive, for example, means the system cannot consolidate, cannot decay, and cannot evolve — not because memory requires these limitations, but because this particular implementation imposes them.

---

## 3. Memory

Memory as a primitive has been treated extensively in Part 1 and Part 3. The functional specification, summarized here for completeness, is:

A memory primitive is a system component that (a) stores information in a form that can be queried by other components, (b) updates its contents based on new information and evaluation outcomes, (c) applies salience-based retention and decay so that stored content reflects current relevance rather than only historical accumulation, and (d) exposes typed interfaces for different classes of stored information — working, episodic, semantic, procedural.

The key property that distinguishes the memory primitive from a storage layer is condition (b): the contents must update. A system with static storage has persistence but not memory. The update mechanism — consolidation, salience adjustment, contradiction resolution — is what makes stored information functional rather than archival.

The interfaces matter as much as the internals. A memory primitive exposes at minimum: a write interface that accepts typed entries with metadata, a read interface that accepts typed queries and returns ranked results, an update interface that accepts evaluation outcomes and modifies stored entries accordingly, and a decay interface that can be triggered by the loop or by time to reduce retention scores on low-salience content.

Any component that does not expose these four interfaces is not implementing the memory primitive. It may be implementing something useful, but it is not memory in the functional sense required for intelligent behavior.

---

## 4. Attention

Attention is the primitive that determines what information an agent processes at any given moment. This is not the same as the transformer attention mechanism, which is one computational implementation of a broader functional requirement. The primitive is the requirement.

The functional specification: an attention primitive is a system component that (a) selects a bounded subset of available information for active processing, (b) adjusts selection criteria based on current task context and goals, (c) allocates differential weight to selected information based on relevance to the current processing objective, and (d) updates selection criteria based on outcomes — what was attended to, what action resulted, and whether the outcome was favorable.

Conditions (b) and (d) are the ones current systems consistently fail to satisfy. Transformer attention satisfies (a) and (c) within a single forward pass. It does not satisfy (b) in any durable sense — attention patterns are recomputed from scratch on each call with no persistent adjustment based on prior task context. It does not satisfy (d) at all — there is no pathway from action outcomes back to attention parameters outside of gradient-based training.

At the agent system level, attention manifests in decisions about what to retrieve from memory, what to include in the reasoning context, and what observations to prioritize from the environment. These are all attention operations, and they all require the same functional properties: context-sensitivity, bounded selection, differential weighting, and outcome-based update.

The absence of a durable attention primitive produces a specific failure mode: agents that retrieve and process the same classes of information regardless of task context, outcome history, or environment state. The retrieval strategy does not adapt. The context construction does not adjust. Every task is approached with the same informational framing, because there is no mechanism to do otherwise.

A well-specified attention primitive needs the following interfaces: a query interface that accepts a task context and returns a ranked selection of information sources to activate, a weight interface that returns differential relevance scores for currently active information, an update interface that accepts outcome signals and adjusts selection and weighting parameters, and a scope interface that enforces the bounded capacity constraint — preventing the system from attending to everything simultaneously and producing the prioritization pressure that makes attention useful.

The scope interface is not optional. Attention without a capacity constraint degrades into uniform processing, which is not attention. The constraint is what forces prioritization. Without it, the primitive loses its functional property.

---

## 5. Planning

Planning is the primitive that generates action sequences over time toward a goal. It is distinct from action selection, which is a single-step operation, and from policy execution, which follows a pre-specified behavioral pattern. Planning requires reasoning forward over possible futures and selecting actions based on their projected contribution to goal achievement.

The functional specification: a planning primitive is a system component that (a) maintains a representation of the current goal and the current state, (b) generates candidate action sequences that could reduce the distance between current state and goal state, (c) evaluates those sequences against a model of the environment and a model of the agent's own capabilities, and (d) updates its goal representation, its environment model, and its action generation strategy based on execution outcomes.

Current implementations approximate this in two ways. Prompt-based planning — instructing a language model to produce a plan as text — satisfies (b) superficially but not (a), (c), or (d). The goal and state are text strings in the prompt, not structured representations that can be queried or updated programmatically. Evaluation is implicit in the model's training, not explicit in the system architecture. And there is no update mechanism — the plan is generated once and executed until it succeeds, fails, or is manually revised.

Tool-use frameworks partially satisfy (b) and (c) for shallow planning horizons — tasks where the action sequence is short enough to fit in context and the environment model is simple enough to be captured in the system prompt. They fail for longer horizons because the plan representation degrades as context grows, and they fail when the environment model needs to update mid-execution because there is no pathway from observations back to the environment model.

A well-specified planning primitive requires: a goal representation that is structured, queryable, and updatable; a state representation that is updated by observations from the environment and action outcomes; a generation mechanism that produces action sequences given goal and state; an evaluation mechanism that scores candidate sequences against environment and capability models; and an update mechanism that modifies the environment model and action generation strategy based on execution outcomes.

The interaction between planning and memory is non-trivial. Effective planning requires access to episodic memory — prior attempts at similar tasks, their action sequences, and their outcomes. It requires access to semantic memory — generalized knowledge about environment dynamics and action consequences. And it requires the ability to write back to both: new episodes are generated by plan execution, and new semantic entries are generated when planning reveals generalizable patterns. A planning primitive that does not have read and write access to the memory primitive is operating without the historical evidence that makes planning non-trivial.

The interaction between planning and attention is equally critical. Planning operates over a potentially large space of candidate action sequences. Attention determines which parts of that space are actively considered. Without a functional attention primitive, planning either exhausts its computational budget on low-relevance sequences or defaults to shallow search over the most salient options, which may not be the optimal ones.

---

## 6. Composition and Emergence

The three primitives — memory, attention, planning — are not independent. Their interactions are where system-level behavior emerges.

Memory without attention is an archive. Content accumulates but is not selectively activated based on what the current task requires. Retrieval is either exhaustive or arbitrary.

Attention without memory is reactive. Selection is based on the current context only. There is no historical basis for adjusting what to prioritize or why.

Planning without memory is stateless. Each plan is generated from scratch with no benefit from prior attempts. The environment model does not improve. Failure modes repeat.

Memory without planning is recording. Observations are stored and can be retrieved but do not inform future action sequences in any structured way. The agent is a sophisticated log, not a purposive system.

The full composition — memory that feeds planning, attention that governs what memory is activated and what plan candidates are evaluated, planning that writes new episodes back to memory and adjusts attention scope based on goal state — is the minimal functional architecture for a system that can be called intelligent in any non-trivial sense. Each primitive is necessary. None is sufficient. Their interfaces with each other are as important as their internal specifications.

This is not an argument for biological plausibility or for a specific implementation. It is an argument about functional requirements. Any implementation that satisfies the primitive specifications and their interaction semantics will exhibit the target behaviors. Any implementation that approximates the primitives — treating vector search as memory, prompt instructions as attention, LLM-generated text as planning — will exhibit those behaviors only within the narrow constraints where the approximation holds.

---

## 7. Evaluating a System Against This Framework

The primitive specifications above are precise enough to evaluate any given agent system. The evaluation procedure is straightforward.

For each primitive, ask: does the system have a component that satisfies all conditions in the specification? For each condition, ask: does the system have an explicit mechanism that implements it, or is it approximated through another component's side effect?

A component that implements condition (a) but not condition (d) of the attention specification, for example, provides bounded selection but does not update based on outcomes. The system will exhibit context-sensitive attention within a single execution but will not improve its attention strategy across executions. That is a specific, predictable limitation that follows directly from the missing condition.

For each interface, ask: is it explicitly typed and exposed to other components, or does the primitive interact through shared mutable state or context injection? Implicit interfaces create coupling that degrades as the system scales and makes it impossible to replace or upgrade individual primitives without affecting the behavior of components that depend on them.

Systems that pass this evaluation — that have explicit implementations of all three primitives with all specified conditions and typed interfaces — do not currently exist in public open-source agent frameworks at the time of writing. Systems that partially satisfy the specifications exist and are useful. The evaluation framework identifies precisely where each system's partial implementation ends and what behavioral limitations follow from it.

---

## 8. Open Questions

**How should the three primitives share state without creating circular dependencies?** Memory informs attention, attention governs what planning evaluates, planning writes back to memory. The cycle is functional but creates consistency requirements. What is the correct consistency model for a system where primitives update each other?

**What is the minimal implementation of each primitive that preserves its functional properties?** Full implementations are expensive. There are likely threshold implementations — minimum sufficient conditions — below which the primitive's functional property is lost and above which further complexity yields diminishing returns.

**How should the attention primitive's capacity constraint be set?** A fixed capacity limit is a hyperparameter. A dynamic limit that adjusts based on task complexity and available compute is more principled but harder to specify. What determines the right capacity at runtime?

**How should planning update its environment model when observations contradict prior model entries?** The contradiction resolution policy for the planning primitive's environment model is analogous to the consolidation conflict resolution problem for memory. They may require the same mechanism. If so, the boundary between the planning and memory primitives needs clarification.

**How do the primitives compose across multiple agents?** In multi-agent systems, attention must account for the actions and states of other agents, memory must represent shared and private information separately, and planning must model other agents' planning processes. The single-agent primitive specifications are a necessary foundation but are not sufficient for multi-agent composition.

**What is the relationship between these primitives and model capability?** The specifications are silent on what model is used to implement generation and evaluation within each primitive. A weaker model implementing all three primitives correctly may outperform a stronger model with approximated or absent primitives. This is an empirical claim that current evaluation benchmarks are not designed to test.
