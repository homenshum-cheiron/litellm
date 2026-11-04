# Retries, fallbacks and cooldowns

The router has three tools for failures, and they work at different scopes. A retry tries the same model group again, and selection can land on a different deployment. A fallback moves the request to a different model group. A cooldown takes one deployment out of selection for every request, for a few seconds.

![An attempt runs inside the retry loop of one model group, which runs inside the fallback walk across model groups; a failure may set a cooldown key that filters every later selection](../img/reliability-1.svg)

Every decision reads the failure's kind (rate limit, context window, content policy, auth, not found, timeout, ...), its status code and its `Retry-After`. The router reads these from the provider's exception and decides from them alone

The router is the only retry loop. It sets `max_retries` to 0 on the provider call, so neither the provider SDK nor the HTTP client retries on its own, and each retry can pick a different deployment

## Retries

The retry loop runs the first attempt. When it fails, the router works out how many retries this request gets, decides whether this failure is worth retrying, waits, and loops

![The retry loop: a guardrail intervention gives up at once, otherwise the router works out the retries, checks the failure, sleeps, selects again skipping failed deployments, and returns, loops or hands the failure to the fallback walk](../img/reliability-2.svg)

A failure is not worth retrying in these cases. A context window error when `context_window_fallbacks` is set, and a content policy error when `content_policy_fallbacks` is set, because a fallback will do better than another deployment of the same model. A status code that should not be retried, other than 401 and 403. A not found error. A rate limit when no healthy deployment is left and fallbacks exist. An auth or permission error in a group with a single deployment. And any failure when no healthy deployment is left

The sleep is zero while the group has more than one deployment and some are still healthy, since the next attempt can go somewhere else. Otherwise the router honors the provider's `Retry-After` up to one minute, and without one it backs off exponentially from 0.5 seconds up to 8 seconds, never less than the router's `retry_after`, plus up to 0.75 seconds of jitter.

### Retry policies

`retry_policy` sets the number of retries per failure kind, for example more retries for rate limits than for timeouts, and `model_group_retry_policy` does the same per model group

```python
Router(
    model_list=...,
    retry_policy={"RateLimitErrorRetries": 3, "TimeoutErrorRetries": 1, "ContentPolicyViolationErrorRetries": 0},
    model_group_retry_policy={"gpt-4o": {"RateLimitErrorRetries": 5}},
)
```

A policy's count for the failure's kind wins over `num_retries`, unless the request itself turned retries off with `num_retries=0`

## Fallbacks

When the retry loop gives up, the router chooses where to go next

![Where the router goes once retries are exhausted: raise on disable_fallbacks or a guardrail, else the next orders in the group, weighted failover, then the context window, content policy or generic fallbacks by failure kind](../img/reliability-3.svg)

Each list is a list of single-key dicts that map a model group to an ordered list of groups to try, for example `fallbacks=[{"gpt-4o": ["claude-sonnet", "gpt-4o-mini"]}]`. A target can also be a dict, `{"model": "gpt-4o-mini", "temperature": 0}`, whose other keys override the request's params for that hop. `default_fallbacks=["gpt-4o-mini"]` becomes the catch-all entry `{"*": ["gpt-4o-mini"]}`. The router looks up the group being served first and then the group the request asked for, so a fallback target's own fallbacks apply on the next hop. The lists come from the router's arguments or, when the router sets none, from the process settings in the dynamic snapshot. A request can also carry its own `fallbacks`, which win for that request

The walk takes each target in order. Before a target in another group it runs `fallback_access_check` (may this caller use that group) and `fallback_budget_check`. It skips targets this request already tried, then runs the whole retry loop again against the target, which can fall back further. A depth counter stops the walk at `max_fallbacks` (default 5, or the `ROUTER_MAX_FALLBACKS` environment variable). The access and budget checks use the proxy's key and team data. Each hop is reported as a `Fallback` routing event with its source, target, order, depth and level

![A fallback in sequence: model A fails with a 503 after three attempts, the fallback checks allow group B, and B answers with a 200](../img/reliability-4.svg)

With `fallbacks=[{"A": ["B"]}]`, `context_window_fallbacks=[{"A": ["A-long"]}]` and `num_retries=2`, a 503 from A is retried twice in A, then the request goes to B with two retries of its own. A context window error from A is not retried at all, since a context window fallback exists, and goes straight to A-long

### Order fallbacks

When a group's deployments use more than one `order`, a failure first walks down the orders inside the same group before it leaves the group. With `order: 1` PTU deployments and an `order: 2` pay-as-you-go deployment, a 429 on the PTU tier sends the request to pay-as-you-go in the same group, and only then to the group's configured fallbacks. Each order level gets its own retry loop. A context window or content policy error skips the order walk, since a lower priority deployment of the same model would fail the same way

### Weighted failover

With `enable_weighted_failover` and `simple-shuffle`, a failed deployment is excluded and the group is picked again among the deployments that are left and not cooling down, before any configured fallback runs. It spreads a failing deployment's traffic over its healthy siblings with their weights instead of sending it to another group. Context window and content policy errors skip it for the same reason as above

## Streams

A streamed attempt succeeds as soon as the provider opens the stream, but it can still fail while the caller reads it. The router wraps every stream it returns. When the stream breaks before the caller received any content, the router takes over with a retry in the same group or the next fallback, as if the attempt had failed up front. Once content has reached the caller, a retry would repeat output, so the stream fails, unless `enable_mid_stream_fallback_continuation` is set. Then a chat completions stream that broke after plain assistant text continues on a fallback deployment that supports assistant prefill: the router sends the text so far as a prefill, so the new deployment carries on instead of starting over, and the caller sees one response. Streams that already emitted tool calls, thinking blocks, audio or images, or constrained output such as JSON mode are never continued, and without an eligible deployment the original error is raised

![What happens when a stream breaks: before any content the router retries or falls back, after text with continuation on it continues on a fallback deployment with a prefill, otherwise it raises to the caller](../img/reliability-5.svg)

## Cooldowns

A cooldown is a key `deployment:{id}:cooldown` in the routing cache, with a TTL equal to the cooldown time. While the key exists, the cooldown filter drops the deployment from every selection, on every process that shares the Redis. When the TTL expires the deployment is simply eligible again. There is no probe request and no early reset on success

The failure handler decides whether a failure cools the deployment down. It runs two checks, first on the failure and then on the deployment's recent history

![How a failure on a deployment leads to a cooldown: the skip and counting checks, then either the allowed_fails counter or the last 60 seconds error-rate rules](../img/reliability-6.svg)

The cooldown time comes from the deployment's own `cooldown_time` if it sets one, then the provider's `Retry-After`, then the router's `cooldown_time`, which defaults to 5 seconds. Cooldown alerts and metrics fire from the `CooldownSet` routing event when a deployment is cooled down

### Allowed fails

`allowed_fails` lets a deployment fail a few times before it cools down. Each counted failure increments `deployment:{id}:allowed_fails`, which expires with the cooldown time, and the deployment cools down once the count goes over the limit. `allowed_fails_policy` sets the limit per failure kind, for example tolerating a few timeouts but cooling down on the first auth error

```python
Router(
    model_list=...,
    allowed_fails_policy={"TimeoutErrorAllowedFails": 5, "AuthenticationErrorAllowedFails": 0, "RateLimitErrorAllowedFails": 2},
)
```

A deployment can set its own `allowed_fails` or `allowed_fails_policy` in `model_info`, which wins over the router's. The router-wide `allowed_fails` only takes part when it was set explicitly or changed from its default, so a router that never mentions it uses the error-rate rule below

Two details are easy to miss. The success and failure counts for the error-rate rule are kept in memory only, per process, with a 60 second TTL, so on a fleet each pod judges a deployment by the traffic it saw itself. The `allowed_fails` counter and the cooldown key live in Redis when it is configured, so they apply to the whole fleet. And by default a lone deployment is protected from the error-rate rule, since cooling down the only deployment of a group would fail every request for that group

When every candidate is cooled down, selection raises `RouterRateLimitError` with the time until the soonest cooldown ends, and the fallback walk handles it like any rate limit. `enable_health_check_routing` with an allowed fails policy changes this, by ignoring cooldowns for the request when they would leave nothing
