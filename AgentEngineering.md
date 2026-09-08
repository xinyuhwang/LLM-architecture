# Reliable AI Applications
deterministic software + probabilistic reasoning + external tools + state + recovery mechanisms
## Comparison Between Traditional Application With AI Application
* Traditional Application

  HTTP request -> validate input -> query database -> business logic -> return response
* AI application
```
User
  │
  ▼
LLM reasoning
  │
  ▼
Agent Loop  ◄────────────────┐
  │                          │
  ├──────────┬──────────┐    │
  ▼          ▼          ▼    │
Tool A     Tool B     Tool C │
  │          │          │    │
  └──────────┼──────────┘    │
             ▼               │
           State ────────────┘
             │
             ▼
     (loop again, or)
             │
             ▼
      Final response
```
## LLM APIs
A way to send text (a "prompt") to a model over HTTP and get text back.
## Tool calling
Tool calling lets an LLM API turn a user request into a structured, schema validated request to invoke a specific function which is decided and formatted by the model, but executed by the calling application, with the result fed back so the model can continue reasoning or respond.

Reliability features: retries, fallback, timeout, idempotency.
### Tool schema
A tool schema is the contract that tells the model what functions exist, what arguments they take, and what those arguments mean. Typically expressed as JSON Schema:
```
{
  "name": "check_shipping",
  "description": "Look up shipping status for an order",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": {"type": "string", "description": "The order ID, e.g. ORD-12345"}
    },
    "required": ["order_id"]
  }
}
```
The schema does two jobs at once: it constrains what the model is allowed to say (only these functions, only these argument shapes), and it documents when to use each tool via the description field, which the model reads as part of its reasoning.

Reliability features: Input validation.
## Agents
A semi-autonomous software system powered by a Large Language Model (LLM) that can making a sequence of decisions and taking actions based on what it learns along the way. It gets a goal, decides which tool to call, observes the result, decides the next step, and repeats until it thinks it's done.
### Agent loops
An agent loop is what you get when tool calling is applied repeatedly and the model itself decides when to stop: reason → call a tool → observe the result → reason again → call another tool → … → final response. Each iteration adds to the model's context, so its next decision is informed by everything that happened before.

The defining property of an agent loop, compared to a fixed pipeline, is that control flow is decided by the model at runtime.

Reliability features: Loop level timeout + max iterations, Retry that step, or fall back and let the model reason around the failure.
## Structured outputs
Use a schema as the guideline to format model output. E.g. a JSON object, a specific set of fields, an enum-constrained value. Mechanically this is enforced either by:
* Constrained decoding (the provider restricts which tokens are legal at each step, guaranteeing schema conformance), or
* Prompted formatting (the model is asked nicely to output JSON, with no hard guarantee)

Reliability features: Output validation.
## Reliability Layer
### Retries (for transient failures)
Failure mode: a tool call fails because of a transient condition. Retries apply at two layers: retrying the LLM API call itself, and retrying the tool call (database, third-party API). Retries are only safe when combined with idempotency.

* Transport level retries: Standard pattern is to retry with exponential backoff.
* Semantic/validation retries: Resending the request with an added error message.
### Fallback (for persistent or degraded failures)
Failure mode: retries exhausted, or the primary path is known to be degraded (a specific tool is down, a model provider is having an outage).

Fix: fall back to an alternative such as a secondary model provider, a cached/stale result, or a simpler non-LLM heuristic. Fallback is often paired with a circuit breaker: after enough consecutive failures, stop trying the primary path for a cooldown period.
### Timeout
Failure mode: a call (tool call, LLM call, or the agent loop as a whole) never returns, or takes too long to be useful.

Fix: bound every layer independently:

* Per-call timeout: a single tool call or model call gets cut off after N seconds.
* Per-loop timeout / max iterations: the entire agent loop gets cut off after N steps or M seconds total, regardless of whether any individual step timed out.
### Idempotency (safeguard for retries and fallback)
Idempotency keys is a unique identifier attached to the logical operation so that if the same operation is submitted twice, the downstream system recognizes it and returns the original result instead of repeating the side effect.
### Validation (for malformed or hallucinated calls)
Failure mode: the model emits a tool call or structured output that's syntactically or semantically wrong.

Fix: validate at two points:

* Input validation: before a tool actually runs, check the arguments against the schema and against domain rules before executing anything with a side effect.
* Output validation: after the model produces structured output, check the parsed result matches the expected schema and constraints.

Validation is the layer that catches errors before they become side effects, which is what makes it the natural partner to idempotency: idempotency limits the damage of a retried mistake, validation tries to stop the mistake from executing at all.
## Async in an agentic workflow
* Parallel tool execution.
* Parallel agent instances.
## Caching in an agentic workflow
* Prompt caching (provider level). In an agent loop, the same system prompt, tool definitions, and often a growing conversation history get resent on every single turn of the loop because LLM APIs are stateless. "Prompt caching" is the product level name, while "prefix caching" describes the specific technical mechanism most providers use to implement it. Some hosted inference providers keep a model's Key-Value (KV) cache active in GPU memory if consecutive prompts share the exact same starting text (prefixes). Prefix caching ensures you only pay full price for the newly added tokens on each loop iteration.
* Tool result caching (application level). Using traditional datastores to hash the exact input parameters of external tools. If an agent encounters a sub-task requiring it to call an external function multiple times with the same parameters, the orchestrator instantly returns the cached string instead of triggering a live, rate limited API call.
* Exact match response caching. If the same full prompt/context is likely to recur (common in eval runs, or repeated user queries), caching the entire model response and short circuiting the loop entirely can save a full round of calls.
## Orchestration
The management, sequencing, and coordination of data flows, prompt chains, tool calls, and model components to execute a cohesive, multi-step application workflow. It's the coordination layer of the agentic system.
### Workflows
A workflow is the defined structure of what steps happen and in what order.
* Static workflows (DAGs): the sequence is fixed by code ahead of time.
* Dynamic workflows (agent-driven): the LLM decides the next step at runtime.

Most real systems are a hybrid: a fixed outer skeleton (validate input → run agent loop → validate output → deliver result) with a dynamic agent loop nested inside one stage.
### State
State is whatever needs to persist across steps — conversation history, intermediate results, tool outputs, counters, flags. The orchestration layer owns this because individual steps shouldn't need to know how they got the input they're operating on; they just read from and write to a shared state object.

Design Questions:
* Scope: what's global to the whole workflow vs. local to one step or one loop iteration?
* Shape: is state a simple accumulating log, or a mutable structured object? Accumulating log is easier to debug and replay; mutable is more compact but loses history.
### Checkpoints
A checkpoint is a saved snapshot of state at a point in the workflow, so execution can be paused and resumed without starting over. This matters for two distinct reasons:
* Durability: if the process crashes in the middle of a workflow (server restart, deploy, timeout), you resume from the last checkpoint instead of re-running everything from the top. It's important for long running agent loops or workflows with expensive/side effecting steps you don't want to repeat.
* Human-in-the-loop: some workflows need to pause for approval, therefore a checkpoint is what lets the workflow suspend, wait indefinitely for outside input, and resume exactly where it left off.
### Parallel execution
Parallel execution is a pure latency/throughput optimization. Some steps don't depend on each other's output, so they can run concurrently instead of sequentially. Orchestration is responsible for:
* Identifying independence: which steps can safely run in parallel (no shared mutating state, no ordering dependency)
* Fan-out / fan-in: dispatching N parallel branches and then merging their results back into a single state before continuing
* Partial failure handling
### Error recovery
This is where all the reliability features from the previous discussion (retries, fallback, timeout, idempotency, validation) get applied at the orchestration level, rather than inside a single tool call. The orchestration layer has to decide, for any failure at any step:
* Retry the failed step, using the checkpointed state from just before it
* Fall back to an alternative path and continue the workflow in a degraded mode
* Compensate: undo or offset the effects of steps that already succeeded, if a later step in the same workflow fails
* Fail the whole workflow cleanly, surfacing a useful error rather than a partial, inconsistent state
## Production AI
### Observability
Observability is the ability to see what the system is doing internally, from its external outputs (logs, traces, metrics). Observability is a property of the system, which is the capability that makes analysis possible.
* Logging: full prompts, tool calls, tool results, and final outputs.
* Tracing: for multi-step workflows and agent loops, a single user request can spawn many LLM calls and tool calls. Tracing stitches these into one coherent timeline so you can see the whole chain of decisions.
* Metrics: latency per step, token usage, tool-call success/failure rates, loop iteration counts — aggregated over time to catch regressions or drift.
### Evaluation
Evaluation is the process of judging output quality against predefined standards using either fixed test cases or live traffic.
* Correctness: did the tool call use the right arguments? Did the final answer match the expected fact?
* Task completion: did the agent loop actually accomplish what the user needed, end to end? 
* Adherence: did the output follow the required schema, format, or policy constraints?
* Quality on fuzzy dimensions: is the tone appropriate, is the response helpful, is it appropriately concise, which is usually scored via rubric or LLM-as-judge.
### Security
Security is a property of the system (can it resist manipulation and limit damage from exploitation), and that property gets established through concrete features/controls, then checked through a separate measurement practice.

**Security threats**:
* Prompt Injection: Prompt injection is when untrusted content (a webpage, a document, a tool result, even a user message) contains text designed to hijack the model's instructions.

  Solutions:
  * Instruction/data separation: structurally marking untrusted content (e.g. with delimiters, XML tags, or separate message roles) so the model can distinguish "things to act on" from "things to treat as data to read, not obey." This doesn't guarantee immunity but reduces confusion.
  * Content sanitization/filtering: scanning retrieved or tool-returned content for known injection patterns before it reaches the model's context.
  * Privilege separation: split a system into components with different levels of trust/access, so that compromising one component doesn't give you the access of the others. In a agentic system, enforcement logic should lives in code that runs after the model's decision and outside the model's control, and not in text the model reads and is trusted to obey.
    * Possible locations could be:
      * Inside the tool function itself;
      * A gateway/middleware layer between model and tool (often better, since checks are centralized rather than duplicated per tool);
      * The underlying service's own permissions (this is defense in depth).
  * Output monitoring: flagging responses where the model's behavior suddenly diverges from the system prompt's intent (e.g. it starts trying to call a tool it was never asked about) as a signal of possible injection.
  * Least trust defaults for tool results: treating every tool result and every piece of fetched content as potentially adversarial, not as a trusted extension of the system prompt.
* Tool permission scoping: The core idea: bound the maximum damage a single tool call (whether from a hallucination, an injection, or a genuine bug) can do.

  Solutions:
  * Per-tool authorization: each tool has its own explicit allow-list of what it can do.
  * Rate limits: caps on how many times a tool can be called per session/user/time window.
  * Value/amount caps: hard limits on consequential parameters. E.g. a refund tool that structurally cannot process an amount above the order total, enforced in code, not just prompted as a rule.
  * Approval gates for high risk actions: irreversible or high stakes tool calls require human confirmation before executing.
  * Scoped credentials: the tool's underlying API key/service account has only the permissions it strictly needs (principle of least privilege).
  * Sandboxing for code execution tools: if a tool lets the model run code, that code runs in an isolated environment with no access to the host filesystem, network, or credentials beyond what's explicitly granted.
* Data exposure: Concerned with information leaking somewhere it shouldn't, which could across users, into logs, or to third parties.

  Solutions:
  * Context isolation between sessions/users: making sure one user's conversation history, retrieved documents, or tool results can never leak into another user's context.
  * PII redaction/masking: stripping or masking sensitive fields (SSNs, card numbers, health data) before they're sent to a model provider or written to logs.
  * Data minimization: only including the fields a tool or prompt actually needs.
  * Access control on retrieval: if the system uses RAG, making sure retrieval respects the requesting user's permissions, not the permissions of whoever originally indexed the document.
  * Logging hygiene: deciding deliberately what gets logged (full prompts and outputs are useful for observability, but may need redaction before storage or must be stored with the same access controls as the underlying sensitive data).
 
#### Broader/cross-cutting controls
* Input validation: rejecting malformed or out-of-range tool arguments before they execute, which also blocks a class of injection attempts that rely on malformed structured input.
* Audit logging (strict retention/tamper-resistance standards): an immutable record of every tool call, argument, and result, specifically so that if something does go wrong, you can reconstruct exactly what happened and who/what triggered it.
* Red-teaming / adversarial testing: proactively trying to break the above controls before an attacker does.
### Costs
Costs in LLM systems:
* Token costs: input + output tokens per call, multiplied by however many calls one user request triggers.
* Model selection
* Retries and loops: every reliability feature multiplies cost when triggered.
### Latency
The time between a request being made and a response (or some meaningful part of it) being received.

**Metrics**
* Time to first token (TTFT): How long before any output starts appearing. This is what dominates perceived responsiveness (a user staring at a blank screen for 3s feels very different from tokens starting to stream immediately).
* Total generation time: How long until the full response is done. It matters when the whole output is needed before anything downstream can use it (e.g. a structured JSON output that must be complete to parse).
* Per-call latency: Time for a single LLM call or single tool call. This is the basic unit where everything else is built from.
* End-to-end latency: Total time for the whole user facing request, including every LLM call, tool call, and orchestration overhead in an agent loop. This is what the user actually experiences, which is the one that compounds across loop iterations.

In practice:
* Streaming improves TTFT and perceived latency without changing total generation time at all. However, the user sees progress sooner, even though the model isn't actually finishing any faster. This is a UX lever, not a compute time reduction.
* In an agent loop, end-to-end latency is the sum of N sequential per-call latencies (reasoning → tool call → reasoning → tool call → ...) unless steps are parallelized. This is why a loop with 5 iterations can feel painfully slow even if each individual call is fast.
* Timeouts are set against per-call latency, not end-to-end. Therefore we bound each step individually and separately bound the whole loop, because a system that only checks total time can still let one hung step eat the entire budget before anything triggers.
### Human-in-the-loop
This is the deliberate design decision to interrupt automation at specific points, usually driven by risk:
* Approval gates: high stakes or irreversible actions (large refunds, account deletions, anything hard to undo) pause for human confirmation before executing. This is the checkpoint-and-resume mechanism from orchestration, applied specifically to risk management.
* Escalation paths: when the model's confidence is low, or it's outside its competence (edge cases, ambiguous requests, angry customers), the system routes to a human rather than forcing an automated answer.
* Feedback loops: human corrections and overrides feed back into evaluation data. Every human intervention is also a labeled example of where the system fell short. This is useful for both monitoring and future fine-tuning/prompt improvements.

The design tension here is that human-in-the-loop is a direct cost/latency/autonomy tradeoff against everything above it. More automation is cheaper and faster, but human oversight is what bounds the damage of a model or tool doing something wrong in a context that matters.

## How they fit together
Observability is the foundation. Evaluation reflect the effectiveness of changes. Security and human-in-the-loop are both risk controls. Cost and latency are the two resource dimensions that is used in trade off against reliability and quality. Nearly every reliability feature from the earlier discussion (more retries, more fallback attempts, bigger models) buys robustness at the direct expense of cost and latency, which is why production AI is ultimately about tuning that whole system of tradeoffs rather than maximizing any one axis.
