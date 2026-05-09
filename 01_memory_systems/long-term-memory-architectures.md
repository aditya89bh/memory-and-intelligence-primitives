# Long-Term Memory Architectures in Agent Systems

*Part of a series on memory systems for artificial agents. This article examines the architectural requirements for long-term memory — what it must contain, how its components interact, and where current designs fall short.*

---

## 1. Problem Framing

Most agent systems that claim to have long-term memory have a vector index that persists across sessions. Content goes in, embeddings are computed, and at query time the top-k nearest neighbors are returned. This is called memory because it stores things and retrieves them. It is not memory in any architecturally meaningful sense because it does not evolve, does not consolidate, and has no internal structure that reflects the different functional roles that stored information plays in agent behavior.

The problem is not retrieval quality. Approximate nearest-neighbor search has become fast and reliable. The problem is that long-term memory in a capable agent system is not a retrieval problem — it is a structural problem. Different classes of stored information have different retention dynamics, different access patterns, different update semantics, and different relationships to the agent's decision-making. Flattening these into a single index loses the distinctions that make memory useful for anything beyond simple fact lookup.

This article specifies what a long-term memory architecture actually requires: the types of memory that must be present, how they relate to each other, what processes connect them, and what the interfaces between them and the rest of the agent system must look like.

---

## 2. Current Approaches

Three patterns dominate current implementations.

The first is the vector store as memory store. Everything the agent encounters — user messages, tool outputs, retrieved documents, intermediate reasoning — is embedded and stored. Retrieval is a similarity query. The assumption is that semantic proximity is a sufficient proxy for relevance. This works for narrow retrieval tasks and fails for anything requiring temporal reasoning, causal tracing, or behavioral adaptation.

The second is the conversation log as memory. Interaction history is stored as structured records and retrieved by recency or keyword match. This captures what was said but not what was learned. It treats episodic history as the whole of memory rather than as one layer of a deeper system. Agents built on this pattern know what happened in prior sessions but cannot generalize across them, cannot identify patterns, and cannot update beliefs based on accumulated experience.

The third is the hybrid: a vector store for general retrieval plus a structured database for specific entities like user preferences or known facts. This is closer to correct in that it acknowledges multiple memory types exist. It fails to specify the processes that connect them — how episodic content becomes semantic knowledge, how contradictions get resolved, how relevance shifts over time. The types are present in name but not in function.

What all three share is the absence of architecture. Components are present. The system that connects them, updates them, and keeps them coherent is not.

---

## 3. Where These Break

The failure modes cluster around three problems.

**Type conflation.** When everything is stored in the same substrate with the same retrieval mechanism, the agent cannot distinguish between "something that happened once" and "something that is generally true." A specific failure event and a general operating principle look identical in a flat vector index. The agent retrieves both with equal confidence and cannot reason about which should generalize.

**No update pathway.** Long-term memory in a capable system must change in response to new experience. Not just grow — change. New observations should update existing beliefs. Repeated patterns should crystallize into general knowledge. Content that is no longer relevant should lose retrieval priority. None of these transformations happen in a static index. What the agent stored six months ago competes on equal footing with what it stored yesterday, regardless of what happened in between.

**No connection to behavior.** Retrieved content is injected into context. Whether it affects what the agent does depends entirely on in-context processing — there is no architectural guarantee that long-term memory influences planning, evaluation, or action selection in any structured way. Memory is a read-only reference layer, not a component of the agent's reasoning structure.

These are not retrieval failures. Better indexing or re-ranking cannot fix them. They are architectural absences.

---

## 4. What Long-Term Memory Requires

Long-term memory in an agent system requires three types of stored information, each with distinct structure and semantics, connected by processes that transform content between them.

**Episodic memory** stores records of specific past experiences. An episode has a temporal location, a context in which it occurred, a sequence of observations and actions, and an outcome. It is the most granular layer — close to raw experience, high in specificity, low in generalizability. An episode is not a single fact; it is a structured record of what happened, when, under what conditions, and with what result. This structure is what makes episodic memory useful for reasoning about particular prior situations. Similarity search alone cannot reconstruct it.

**Semantic memory** stores generalized knowledge — facts, beliefs, and principles that are not tied to a specific time or event. Semantic memories are derived from episodic content through consolidation: the process of identifying what repeated experiences reveal, extracting the pattern, and storing it in a form that is more general and more durable than any individual episode. A semantic memory is what remains when the details of specific episodes are compressed away. Its relationship to episodic memory must be traceable — it should be possible to identify which episodes contributed to any semantic belief and to update that belief when the underlying episodes are revised.

**Procedural memory** stores action schemas — representations of how to accomplish recurring tasks. Where semantic memory answers "what is true," procedural memory answers "what to do." It is derived from episodic sequences that were executed repeatedly with consistent outcomes. It is the most stable memory type and the most resistant to update, because behavioral competence is harder to revise than factual belief. Its retrieval is triggered differently too — not by semantic similarity to a query, but by recognition that a current situation matches a schema's activation conditions.

These three types are not interchangeable. Collapsing them into a single store loses the structural properties that make each one useful. Collapsing them into multiple stores without specifying how they interact produces components that are architecturally present but functionally isolated.

---

## 5. The Processes That Connect Them

The three memory types are not static stores. They are stages in a pipeline of transformation that runs continuously in the background of agent operation.

**Encoding** is the first stage: the transformation of raw experience into episodic records. This involves selecting what is worth storing — not everything an agent encounters should enter memory — and structuring it appropriately. The encoding decision is itself a design problem. What metadata attaches to an episode? What constitutes its boundaries? How is uncertainty represented? These choices constrain everything downstream.

**Consolidation** is the transition from episodic to semantic. It runs periodically or in response to triggers, reads from episodic memory, identifies patterns across episodes, and writes generalized knowledge to semantic memory. Consolidation is also where contradictions are resolved — when a new episode conflicts with an existing semantic belief, the system needs an explicit policy for which to trust and how to update. Without consolidation, episodic memory grows indefinitely and semantic memory never improves. The agent accumulates experience without accumulating knowledge.

**Decay** is the mechanism that adjusts retention based on salience. Entries that are rarely retrieved, rarely relevant to successful outcomes, or superseded by more recent knowledge should lose retrieval priority over time. Decay is not deletion. It is a continuous adjustment of how aggressively an entry competes for retrieval slots. Without it, the oldest content in the system is as accessible as the newest, which produces retrieval results that reflect the agent's history more than its current operational context.

**Promotion** is the inverse of decay — the mechanism by which frequently useful episodic content or recently confirmed semantic beliefs gain retrieval priority. Content that is repeatedly retrieved in successful task completions should become more accessible. This creates a feedback loop between memory and behavior: what the agent does well reinforces the memory content that supported it.

These four processes — encoding, consolidation, decay, promotion — are not features. They are the minimum set of mechanisms required for memory to evolve rather than merely accumulate.

---

## 6. Design Implications for Agent Systems

A long-term memory architecture must be specified at the level of types, processes, and interfaces, not at the level of storage technologies. The storage technology implements the architecture; it does not define it.

The interfaces matter as much as the internals. Episodic memory must expose a write interface that accepts structured episode records, not raw text. It must expose a read interface that supports temporal and contextual queries, not just semantic similarity. Semantic memory must expose an interface for consolidation processes to write to it and for planning components to read from it with typed queries. Procedural memory must expose a match interface that accepts current situation representations and returns applicable action schemas.

These interfaces are where the rest of the agent system connects to memory. If the interfaces are loose — if memory is accessed by injecting text into context — then memory is architecturally disconnected from reasoning and planning. It can inform but cannot constrain, contribute but cannot update. The connection between memory and behavior becomes implicit and fragile.

The consolidation process must be a first-class component, not a background script. It needs read access to episodic memory, write access to semantic memory, and a defined conflict resolution policy. Its trigger conditions — time-based, event-based, or load-based — must be specified. Its outputs must be traceable: any semantic memory entry should carry a reference to the episodic content that produced it.

Memory must also be connected to the agent's evaluation mechanism. When an action succeeds or fails, that outcome is signal for memory — it affects which episodic records are worth promoting, which semantic beliefs require revision, and which procedural schemas need updating. An agent that does not route evaluation outcomes back into memory cannot learn from experience at the architectural level.

---

## 7. Open Questions

What should determine episode boundaries? A natural episode boundary might be task completion, a significant state change, or a temporal gap between interactions. Different boundary definitions produce different episodic structures, with downstream effects on consolidation and retrieval. The right boundary definition is likely domain-dependent, but the general principle for choosing one is unclear.

How should consolidation handle partial patterns? A single episode is not a pattern. Ten identical episodes are a clear pattern. The threshold between them is not. Premature consolidation produces brittle semantic beliefs. Delayed consolidation leaves useful generalizations unexpressed. The trigger condition for consolidation is a design variable with significant behavioral consequences.

How should semantic memory handle belief revision without losing traceability? When new episodes contradict an existing semantic belief, the belief should update. But the prior belief may have influenced prior decisions, and those decisions may be part of the episode record. Updating the belief without recording that update creates an inconsistency in the historical record. The versioning semantics of semantic memory are an open design problem.

What is the right decay function? Exponential decay by time, decay by retrieval frequency, decay by association with failed outcomes — these produce different distributions of memory salience. The right function likely depends on the agent's task distribution. The general-purpose formulation is not established.

How should procedural memory interact with planning? Procedural schemas represent action patterns that have worked before. A planning component that ignores them is discarding useful prior experience. A planning component that over-relies on them cannot adapt to novel situations. The interface between procedural memory and planning — how schemas are surfaced, evaluated, and modified by planning outcomes — is an open architectural question that the remaining articles in this series will approach from the planning and evaluation side.
