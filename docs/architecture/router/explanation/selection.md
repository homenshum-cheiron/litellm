# How a deployment is picked

A pick turns a requested name into one deployment. It runs inside every attempt, so a retry picks again and can land somewhere else. `router.select()` runs the same pick without calling the provider.

## The filter chain

Resolution produces the candidates, and a chain of filters removes the ones that must not serve this request right now. Each filter can only remove candidates. The strategy at the end sees what is left.

![The filter chain from pre-routing and name resolution through every filter to the strategy, or to the no deployments available error](../img/selection-1.svg)

## The filters

**Health check filter.** With `enable_health_check_routing`, the background health checks' results decide too. A deployment the last check marked unhealthy is dropped, as long as that result is younger than `health_check_staleness_threshold`. `background_health_check_model_groups` limits this to some groups. If the filter would leave nothing, it keeps every candidate instead, so a broken health checker cannot take a whole group down

**Cooldown filter.** Drops every deployment with a live `deployment:{id}:cooldown` key in the routing cache. Cooldowns are how failures on one request protect the next ones, see [Retries, fallbacks and cooldowns](reliability.md). When health checks drive routing with an `allowed_fails_policy` and the cooldown filter would leave nothing, the router ignores cooldowns for that request

**Blocked filter.** Drops deployments with `model_info.blocked`, which an admin sets to pause a deployment without deleting it. If every deployment of a group is blocked, the error says so

**Callback filters.** Logging callbacks can narrow the candidates too. The budget limiter drops deployments whose provider is over its `provider_budget_config` budget, and deployment and session affinity keep a conversation on the deployment that served it before. They run as part of the chain like any other filter

**Routing plugin candidates.** When routing plugins ran during pre-routing, only deployments of the models they left stay

**Pre-call checks.** With `enable_pre_call_checks`, deployments whose context window the prompt exceeds, those outside the request's `allowed_model_region`, and those that cannot take a requested parameter are dropped. The prompt's token count and each deployment's limits come from the model cost map. When nothing is left, the error says which check removed the last deployment, for example a context window error, which the fallback walk can route to `context_window_fallbacks`

**Tag filter.** With `enable_tag_filtering`, a request's tags pick the deployments whose `litellm_params.tags` include them. Deployments tagged `default` serve untagged requests. `!tag` excludes deployments that carry a tag and `&tag` requires it

**Text-only filter.** A request that carries an image skips deployments that declare no vision support, as long as another candidate remains

**Order filter.** Keeps only the deployments with the lowest `litellm_params.order` among those left, or the order an order fallback targets. See below

**Excluded and retry-skipped ids.** A retry skips the deployments that already failed for this request, but only while something else remains. Weighted failover excludes ids outright, and an excluded id is never picked

## Priority with `order`

`order` is how in-group priority works. Deployments with `order: 1` take all traffic while any of them pass the filters. The ones with `order: 2` only see traffic when the fallback walk targets the group at that order, after every `order: 1` deployment failed for the request (see [Retries, fallbacks and cooldowns](reliability.md)). Mixing is easy to get wrong. While any candidate sets an order, a deployment without one is dropped by this filter, and the order fallbacks never target it either, so give every deployment of a group an order or none. A priority routing group serves each member's priority as its deployments' `order`, so the same filter walks the priorities (see [Strategies, routing groups and sub-routers](strategies.md))

![Priority with order: normal picks go to the two order 1 deployments, and the order 2 deployment only gets a request after order 1 failed](../img/selection-2.svg)

## Shortcuts

A name that resolves to one deployment by its id is served even while that deployment cools down, and the callback filters and the strategy are skipped, since there is nothing to choose. A custom strategy skips the whole chain, see [Strategies, routing groups and sub-routers](strategies.md)

## When nothing is left

The pick fails with "No deployments available for selected model". The error lists the deployments in cooldown and the seconds until the soonest cooldown ends, and says whether pre-call checks were on, since they are a common cause. The fallback walk treats it like a rate limit, so a group with every deployment cooling down falls back to its fallback groups. [Find out why a request went where it did](../how-to/debug-a-routing-decision.md) walks through finding which filter removed a deployment
