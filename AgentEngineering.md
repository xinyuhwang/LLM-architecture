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
