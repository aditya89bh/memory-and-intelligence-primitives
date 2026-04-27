# Cognitive Loops in Agents

---

## 1. Problem Framing

The term "cognitive loop" appears frequently in agent system documentation. It implies a closed cycle: the agent perceives its environment, reasons about what it perceives, acts on that reasoning, and updates its internal state based on the outcome — then repeats. This cycle is supposed to be what separates an agent from a stateless function call.

Most current agent implementations do not have this. They have a sequence of operations that runs in a fixed order, terminates, and is re-invoked from outside. The loop is simulated at the orchestration layer, not implemented at the agent layer. The distinction matters because a genuine cognitive loop has specific structural requirements — state persistence across iterations, feedback pathways that alter internal parameters, and termination conditions that are themselves a product of the loop's execution. Linear pipelines have none of these by design.

The conflation is not arbitrary. Pipelines are easier to build, debug, and scale. Loop-like patterns such as ReAct and Reflexion have made the pipeline-as-loop framing credible by adding reflection steps that look like feedback. This essay examines what those patterns actually implement and where the structural gap lies.

---

## 2. What a Cognitive Loop Requires

A cognitive loop, defined precisely, is a control structure with four functional requirements.

**State that persists across iterations.** Each pass through the loop must be able to read and modify a shared state. That state must carry information forward — not just as injected context, but as a structured representation that the loop's components can query, update, and reason about independently.

**Feedback that modifies loop behavior.** The output of one iteration must have a causal effect on how subsequent iterations execute — not just what content they process. This means the loop's parameters, retrieval strategies, action selection policies, or termination conditions must be updatable based on execution outcomes.

**Internally-determined termination.** The loop must be capable of deciding when to stop based on its own state, not based on an external condition imposed before execution begins. Fixed iteration limits and hardcoded stopping rules are not loop termination; they are pipeline length.

**Observation-action coupling.** The action taken in one iteration must produce an observation that feeds directly into the next iteration's reasoning. The coupling must be structural, not incidental — the architecture must guarantee that action outcomes are captured and integrated, not just returned to a caller who may or may not pass them back.

These four requirements are the minimum bar. An agent architecture that does not satisfy all four is not implementing a cognitive loop.

---

## 3. What ReAct and Reflexion Actually Implement

ReAct structures agent execution as an interleaved sequence of reasoning traces and actions. At each step, the model generates a thought, selects an action, receives an observation, and continues. The pattern is defined at the prompt level — the model is instructed to produce thought-action-observation triples, and the orchestrator parses these outputs and executes the action calls.

This is a linear pipeline with a structured output format. The "loop" runs inside the context window. State is carried forward by appending prior thought-action-observation triples to the prompt. The model does not update any internal representation; it processes an ever-growing sequence and generates the next token in it. Each step is a fresh inference call with a longer input. There is no state object that persists, no component that learns from the sequence, and no mechanism for the loop to alter its own execution strategy.

Reflexion adds an explicit reflection step after task execution. A separate model call generates a verbal assessment of what went wrong, and this assessment is stored — typically as a text summary — and injected into future task attempts. This is closer to a feedback mechanism, but the feedback is verbal and injected into context rather than used to update any structural parameter. The reflection does not change how retrieval works, how actions are selected, or when the agent terminates. It changes what the model reads before generating the next output.

Both patterns are valuable engineering abstractions. ReAct makes multi-step reasoning tractable within a single model. Reflexion enables a crude form of cross-episode learning. Neither implements a cognitive loop as defined above. They simulate loop-like behavior through sequential context accumulation, which is a different mechanism with different failure modes.

---

## 4. Where the Implementation Breaks

**State is context, not structure.** In ReAct-style agents, state is the accumulated prompt. This has two consequences. First, state grows monotonically and is never compressed or reorganized. Second, all components that need state — the reasoning step, the action selector, the termination check — must re-parse the entire context on every iteration. There is no queryable state object, no typed representation, and no component-level isolation. Any error or irrelevant content in earlier context propagates forward and degrades all subsequent steps.

**Feedback affects content, not control.** Reflexion's verbal reflection changes what the model reads, but not how the system executes. The retrieval strategy does not update. The action selection policy does not update. The observation-processing logic does not update. If the agent's failure was structural — for example, it consistently retrieves irrelevant documents, or its action selection is poorly calibrated to a class of environments — verbal reflection cannot fix it because the feedback has no pathway into the components responsible.

**Termination is externally imposed.** In nearly all current implementations, the agent loop terminates when a maximum step count is reached or when the model outputs a designated stop token. Neither condition is computed from the agent's internal state. The agent cannot decide that it has gathered sufficient information, that its confidence in the current plan is above threshold, or that the task has become infeasible. Termination is a pipeline property, not an agent property.

**Observation-action coupling is fragile.** The pipeline assumes that action outputs will be returned to the agent as observations and appended to context. This assumption breaks under any non-trivial execution condition: asynchronous actions, partial failures, actions with delayed effects, or actions that produce structured outputs that the model cannot parse reliably. There is no architectural guarantee that observations are captured completely or integrated correctly. The coupling exists only as a convention in the orchestration code.

**No loop-level learning.** Each invocation of a ReAct or Reflexion agent starts from the same base model with the same retrieval index and the same action selection logic. The loop does not update any of these. Cross-episode experience accumulates only if an external process reads prior run logs and modifies the system components. The agent loop itself has no write access to its own operational parameters.

---

## 5. Design Implications for Agent Systems

Fixing the loop requires treating it as a first-class architectural component, not a prompt pattern.

**State must be externalized and typed.** The agent's working state should be a structured object — not a text buffer — that individual components read from and write to through defined interfaces. This separates state management from context management and makes it possible to query, compress, and update state independently of the model's context window.

**Feedback must target specific components.** Reflection outputs should be routed to specific system components, not injected as context. If retrieval quality is the failure mode, the feedback should update retrieval parameters. If action selection is the failure mode, the feedback should update the action selection policy or its associated scoring function. Generic context injection cannot achieve this routing.

**Termination must be computed from state.** The loop needs an internal evaluator that reads current state and determines whether execution should continue, pause, or terminate. This evaluator needs access to task objectives, confidence estimates, resource consumption, and prior iteration outcomes. Designing this evaluator is non-trivial, but delegating termination to a fixed step count is not a substitute.

**The observation pipeline must be guaranteed.** Observation capture must be an architectural guarantee, not an orchestration convention. This means the action execution layer must return structured observation objects, the integration layer must validate and store them, and the reasoning layer must have a defined interface to query observation history. Fragile convention-based coupling will fail at scale.

**Loop-level learning requires write access.** An agent that learns from experience needs the loop to have write access to at least some of its own operational parameters. This does not require gradient-based learning — it can be as simple as a retrieval index that is updated after each episode, or an action scoring table that is modified based on outcome feedback. The key constraint is that the write pathway must exist in the architecture and must be triggered by the loop itself.

---

## 6. Open Questions

**What is the right unit of state in an agent loop?** Key-value pairs, typed schemas, graph structures, and scratchpad text all have different tradeoffs for queryability, compression, and update semantics. The state representation choice constrains everything downstream.

**How should component-level feedback be routed without creating update instability?** If multiple components are updated based on a single episode's feedback, the interactions between updates can produce unstable or oscillating behavior. Defining safe update ordering and scope is an open design problem.

**What should an internal termination evaluator optimize for?** Task completion, resource efficiency, and confidence threshold satisfaction can conflict. An agent operating under time constraints may need to terminate before confident task completion. The objective function for termination is not obvious.

**How should partial observation failures be handled within the loop?** If an action produces an incomplete or unparseable observation, the loop needs a recovery policy. Retry, skip, flag, and replan are distinct strategies with different costs. The choice should be computable from the agent's current state, not hardcoded.

**At what granularity should loop-level learning operate?** Updating parameters after every step introduces noise. Updating only after full episode completion misses intra-episode signal. The right update granularity likely depends on task structure, but the general design principle is unclear.

**How should multiple concurrent loops be coordinated?** Multi-agent systems run loops in parallel, and their state objects may need to share or exchange information. The synchronization protocol, conflict resolution policy, and shared state access model are all open questions with significant behavioral consequences.
