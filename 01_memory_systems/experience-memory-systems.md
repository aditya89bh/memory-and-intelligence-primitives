# Experience Memory Systems

*Part of a series on memory systems for artificial agents. This article examines episodic memory — how raw agent experience gets structured into retrievable records, where that transformation fails, and what the write path must actually look like.*

---

## 1. Problem Framing

An agent that cannot learn from specific past experiences is not a capable agent — it is a capable function. The difference matters because functions are defined by their inputs and outputs. Agents are defined by what they accumulate. Experience is the raw material of that accumulation, and episodic memory is the system that transforms experience into something an agent can reason about later.

Most agent systems do not have this. They have logs. A log records what happened. An episodic memory system records what happened in a form that supports retrieval, consolidation, and behavioral update. The distinction is structural. A log is a byproduct of execution. An episode is a deliberate representation of experience, designed to be useful downstream.

The problem with treating logs as episodes is that logs are not designed to be retrieved. They are designed to be appended. Their structure reflects the convenience of the writing system, not the needs of the reading system. When an agent later needs to reason about a prior situation — what was the context, what was tried, what was the outcome, what should be tried differently — a log entry rarely contains what is needed in a form that is useful. The information may be present, buried in raw text, but it is not structured to support the kind of query that learning requires.

This article specifies what an experience memory system actually needs: how episodes are bounded, what their internal structure must contain, how the encoding transformation works, and what the write path must guarantee.

---

## 2. Current Approaches

Current implementations store experience in three ways.

The first is the raw transcript store. Every message, tool call, observation, and response is appended to a structured log with a timestamp. This is retrieved either by recency — returning the last N entries — or by keyword search. The structure is uniform regardless of content type. A successful task completion, a mid-task error recovery, and an irrelevant ambient observation all look identical in the store.

The second is the embedding store. Experiences are chunked, embedded, and stored as vectors. Retrieval is by semantic similarity. This loses temporal structure entirely — the sequence of events within an experience is not represented, and neither is the causal relationship between actions and outcomes. The experience is reduced to a bag of semantically-indexed fragments.

The third is the summary store. After a session, an LLM is prompted to summarize what happened, and the summary is stored. This is better than a log in that it is designed to be read, but it introduces a new problem: the summary is produced without knowing what future retrievals will need. Important structural information — specific action sequences, failure precursors, intermediate states — tends to be compressed away precisely because it is locally unimportant but globally significant.

All three approaches share a common failure. They treat experience storage as a write problem with a simple append policy, when it is actually a transformation problem with a non-trivial design space.

---

## 3. Where These Break

**Structure is lost at write time and cannot be recovered at read time.** If an episode is stored as a flat text block, the agent can retrieve it but cannot query its internal structure. It cannot ask: what actions were taken between state A and state B? What was the confidence level at the point of failure? What tools were invoked before the recovery succeeded? These questions require structured records. Unstructured storage makes them unanswerable without re-parsing the raw text — which is unreliable and expensive.

**Episode boundaries are implicit.** In a raw transcript store, every message is an entry. There is no concept of episode — no boundary that separates "the agent was doing X" from "the agent was doing Y." This means retrieval returns fragments of experiences, not experiences. An agent that tries to reason about a prior task will retrieve shards of it interleaved with shards of other tasks. Reconstruction is possible only if the agent can infer episode boundaries from the content — which it cannot do reliably from unstructured records.

**Salience is not encoded.** Not everything that happens during an agent's operation is equally worth remembering. A routing decision that led to a successful outcome is worth more than the routine steps in between. A failure that occurred mid-task is worth more than the surrounding context. If all experience is stored with equal weight, retrieval cannot distinguish high-signal from low-signal records. The agent surfaces memories based on semantic similarity to the query, not based on their informational value relative to prior behavior.

**Causality is not represented.** A transcript records what happened in order. It does not record why one thing led to another. If an action caused an observation, that causal link is not present in the record — only their temporal proximity is. An agent that needs to reason about why a prior approach failed has no structural basis for that reasoning in a flat log. The causal structure must be inferred from sequence, which is unreliable.

**Outcomes are decoupled from the records they evaluate.** In most implementations, the outcome of a task — whether it succeeded, how the user responded, what the evaluation metric returned — is recorded in a separate place from the episode it evaluates. The episode record does not contain its own outcome. This makes it impossible to retrieve "episodes where approach X failed" or "episodes where this class of task required recovery" without joining records that were never designed to be joined.

---

## 4. What an Experience Memory System Requires

An experience memory system requires three things: a definition of what constitutes an episode, a specification of what an episode record must contain, and a write policy that governs how raw experience is transformed into records.

**Episode boundaries.** An episode is a coherent unit of agent experience — a sequence of observations, reasoning steps, actions, and outcomes that belongs together because it was generated in the pursuit of a single goal or sub-goal. Defining the boundary is a design decision with significant downstream consequences.

Time-based boundaries — "an episode ends after N minutes" — are simple to implement and wrong for most tasks. Task-based boundaries — "an episode ends when the current goal is resolved" — are more principled but require the agent to represent and track goals explicitly. State-transition boundaries — "an episode ends when the agent's operational context changes significantly" — are more adaptive but harder to detect reliably.

The choice of boundary definition determines the granularity of episodic memory. Too coarse and episodes contain multiple distinct experiences that cannot be disentangled for retrieval. Too fine and episodes are fragments that cannot support consolidation into general knowledge. Neither extreme produces useful episodic structure.

**Episode record structure.** An episode record must contain more than content. A minimal specification includes:

- *Goal representation*: what the agent was trying to accomplish when the episode began
- *Initial context*: the state of the environment and agent at episode start, including any relevant prior history
- *Action-observation sequence*: the ordered record of what the agent did and what it observed in response, with timestamps and confidence levels where applicable
- *Decision points*: moments where the agent chose between alternatives, with the alternatives considered and the selection criterion used
- *Outcome*: the terminal state, whether the goal was achieved, and any evaluation signal received
- *Derived metadata*: episode type classification, tags for the task domain, links to related prior episodes

The action-observation sequence is the episode's spine. The goal representation and outcome are its endpoints. The decision points are where the episode's value for learning concentrates — they are the moments where different choices would have produced different outcomes, which is exactly the information that behavioral update requires.

**Write policy.** The write policy governs how raw execution traces are transformed into episode records. It has three components.

Salience filtering determines what is worth encoding at all. Not every observation, tool call, or intermediate reasoning step belongs in episodic memory. The filtering criterion is informational value: does this content contain information that would change what the agent does in a similar future situation? Routine steps in a well-understood task have low informational value. Unexpected observations, recoveries from failure, and novel action sequences have high informational value. A write policy that does not filter will produce an episodic store that grows rapidly and retrieves noisily.

Structuring transforms filtered content into the episode record format. This is where raw text becomes typed fields — where "the agent said X after observing Y" becomes a populated action-observation record with a causal link, a confidence level, and a timestamp. Structuring is the step most implementations skip. It is also the step that makes retrieval useful rather than approximate.

Outcome attachment links the evaluation signal back to the episode record at write time, not later. If the agent receives feedback — from a user, from a reward function, from an evaluation component — that signal must be written into the episode record before the record is closed. Deferred outcome attachment creates the decoupling problem described earlier. The record must be self-contained: it must carry its own evaluation.

---

## 5. Design Implications for Agent Systems

The write path is a system component, not a side effect of execution. It must be designed, not assumed. Most agent frameworks treat experience storage as a logging concern — something that runs in the background and captures output. An experience memory system treats it as a transformation concern — something that actively processes execution traces and produces structured episodic records. These are different architectural commitments with different resource implications.

Episode boundary detection needs an explicit mechanism. If the agent represents goals as typed objects, boundary detection can be triggered by goal resolution events. If the agent does not, boundary detection must infer structure from execution traces — which requires its own component with defined inputs and outputs. Leaving it implicit means every retrieval returns fragments whose boundaries must be inferred at read time, which is too late.

Decision points need to be flagged during execution, not reconstructed afterward. This means the agent's reasoning component must have a write pathway into the episodic store — not just the observation pipeline. When the agent evaluates alternatives and selects among them, that event should produce a decision record that enters the episode. Reasoning traces that are discarded after the decision is made are lost to episodic memory permanently.

The episodic store needs a schema, not a schema-on-read approach. Schema-on-read — "we will figure out the structure when we retrieve it" — sounds flexible and is expensive and error-prone. The episode record format should be defined at system design time. Fields may vary by episode type, but the core fields — goal, initial context, action-observation sequence, outcome — must be present and typed in every record. Retrieval quality is downstream of write quality. An episodic store with a well-enforced schema returns structured records that support precise queries. A store without one returns text blobs that require re-parsing on every retrieval.

The write path must account for partial episodes. Tasks fail mid-execution. Sessions end unexpectedly. The agent crashes. An experience memory system that can only store completed episodes will lose the most valuable class of experience: the episode that went wrong before resolution. Partial episode records — with a status field indicating incompleteness and whatever terminal state was captured — are more useful than no record at all. The write policy must handle the partial-episode case explicitly.

Encoding uncertainty is not optional for embodied or tool-using agents. When an agent's observations come from sensors or external tools rather than from a deterministic environment, the observations carry uncertainty. If that uncertainty is not encoded in the episode record, it is lost. Retrieval will later treat a low-confidence observation as ground truth, which is a source of incorrect behavioral generalization. The episode record must carry confidence metadata on observations, not just content.

---

## 6. Open Questions

What is the right granularity for action-observation records within an episode? Recording every token generated by the agent is too fine. Recording only tool calls misses intermediate reasoning. The granularity choice affects both storage cost and the precision of decision-point reconstruction. The general principle for choosing granularity is not established.

How should the write policy handle episodes that span multiple agent sessions? A long-running task may be interrupted and resumed across days or weeks. The episode record must accommodate this — either as a single record with gaps, or as a linked sequence of sub-episodes. The linking semantics and their effect on consolidation are unclear.

How should decision points be represented when the agent's reasoning is implicit? Language model inference does not expose an explicit list of alternatives considered. The decision point record must be inferred from the model's output, which may not contain enough signal to reconstruct the alternatives rejected. Whether this inference is reliable enough to be useful is an open question.

How should conflicting episode records be handled? Two episodes about similar situations may produce contradictory action-outcome pairs — approach X succeeded in one and failed in the other. The episodic store should represent both, but retrieval must surface the conflict rather than return one record arbitrarily. The conflict representation format and its effect on consolidation are design problems without established solutions.

What is the cost-accuracy tradeoff for structuring? Full structuring of every episode is expensive. Minimal structuring is fast but degrades retrieval precision. The right point on this curve likely depends on episode frequency and the downstream cost of retrieval errors. The general tradeoff has not been characterized empirically for agent systems.

How should the write path interact with privacy and retention constraints? In deployed systems, users may have the right to delete their interaction history. If episodic records contribute to consolidated semantic memories, deletion of the source episode may need to trigger revision of derived semantic content. The propagation semantics of deletion through an episodic-to-semantic pipeline are not yet specified in any standard framework.
