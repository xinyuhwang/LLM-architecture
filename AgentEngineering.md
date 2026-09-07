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
## Structured outputs
Use a schema as the guideline to format model (JSON) output. E.g. Pydantic, JSON Schema.
## Tool calling
Tool calling lets an LLM API turn a user request into a structured, schema validated request to invoke a specific function which is decided and formatted by the model, but executed by the calling application, with the result fed back so the model can continue reasoning or respond.
## Agents
A semi-autonomous software system powered by a Large Language Model (LLM) that can making a sequence of decisions and taking actions based on what it learns along the way. It gets a goal, decides which tool to call, observes the result, decides the next step, and repeats until it thinks it's done.
## Orchestration
The management, sequencing, and coordination of data flows, prompt chains, tool calls, and model components to execute a cohesive, multi-step application workflow. It's the coordination layer of the agentic system.
## Async in an agentic workflow
* Parallel tool execution.
* Parallel agent instances.
## Retries in an agentic workflow
* Transport level retries: Standard pattern is to retry with exponential backoff.
* Semantic/validation retries: Resending the request with an added error message.
## Caching in an agentic workflow
* Prompt caching (provider level). In an agent loop, the same system prompt, tool definitions, and often a growing conversation history get resent on every single turn of the loop because LLM APIs are stateless. "Prompt caching" is the product level name, while "prefix caching" describes the specific technical mechanism most providers use to implement it. Some hosted inference providers keep a model's Key-Value (KV) cache active in GPU memory if consecutive prompts share the exact same starting text (prefixes). Prefix caching ensures you only pay full price for the newly added tokens on each loop iteration.
* Tool result caching (application level). Using traditional datastores to hash the exact input parameters of external tools. If an agent encounters a sub-task requiring it to call an external function multiple times with the same parameters, the orchestrator instantly returns the cached string instead of triggering a live, rate limited API call.
* Exact match response caching. If the same full prompt/context is likely to recur (common in eval runs, or repeated user queries), caching the entire model response and short circuiting the loop entirely can save a full round of calls.
