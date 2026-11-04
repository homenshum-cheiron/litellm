# The life of a request

This page follows one `await router.acompletion(model="gpt-4o", messages=...)` from the facade, through the core, to the provider and back. The other endpoint methods (`aembedding`, `aresponses`, `aanthropic_messages`, the files and batches wrappers and the rest) take the same path with a different operation

## The shape

Fallbacks wrap retries, and retries wrap a single attempt. Each attempt selects a deployment and calls `litellm.<op>` with that deployment's params, so a retry can land on a different deployment

![Router.acompletion passes through fallbacks, retries and one attempt to litellm.acompletion and the provider HTTP call](../img/request-lifecycle-1.svg)

The router sets `max_retries` to 0 for the provider call, so neither the provider SDK nor the HTTP client retries on its own. The router's loop is the only retry loop, which is what lets each retry pick a different deployment

`litellm.acompletion` is the same function SDK users call. The router does not talk to providers itself. It chooses the arguments, and the provider transformation layer does the rest

The response carries the chosen deployment's id in `response._hidden_params["model_id"]`, and the proxy reports it in the `x-litellm-model-id` header together with the model group, the retries and the fallbacks the call used

## Step by step

`RustCore.acompletion` turns the call into one `RouterCoreHandle.start("acompletion", request)` and drives the steps it returns. The loop itself is `litellm-rust/crates/router-core/src/call.rs`, and the selection it runs per attempt is `select.rs`

![Numbered sequence of one acompletion call: the Router, the RustCore driver, the core's steps, the cache reads and writes, the attempt through PythonHost to the provider, and the response with routing headers](../img/request-lifecycle-2.svg)

Some details worth knowing

The core turns an attempt's error into a typed `AttemptError` with a failure kind (rate limit, context window, content policy, auth, timeout, ...), the status code and any `Retry-After`. Retry, fallback and cooldown rules read those facts

The core records each outcome itself, through cache steps, right after the attempt, so nothing depends on a logging callback firing. Routing events (`Selected`, `AttemptFailed`, `CooldownSet`, `Retry`, `Fallback`, `RetriesExhausted` and others) ride along on every step, and the driver hands them to the host's observers, stamps the request metadata, and raises the exception a caller expects

A streamed attempt is deferred. The core counts the request when the stream opens. The host's stream wrapper reports the success with the final usage when the stream ends, or the failure if it breaks, and a stream that breaks before the caller got content asks the core what may still take over (`call/stream.rs`)

Stateful strategies, such as latency and usage-based routing, learn from litellm's logging callbacks and keep their counters in Python. They run in `litellm/router_rust/steps/selection.py` when the core asks for a pick

## The sync path

`router.completion()` and the other sync methods run the same loop. `RustCore._call_sync` drives the same steps on the calling thread, and a sleep step blocks it. The proxy uses the async methods for every request
