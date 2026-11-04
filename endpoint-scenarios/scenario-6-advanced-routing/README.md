# Scenario 6: Advanced Routing

> **Proposal.** Nothing here is implemented. The samples describe the shape an
> `Endpoint` could take to express canaries, blue/green, mirroring, A/B tests
> and path fan-out. They parse, but the CRD would reject the new fields today.

Today an Endpoint's backends get a share of every request, by weight. These
patterns need two things more: some requests go to a backend whatever the
weights (`match`), and some backends see a copy but never answer (`mirror`).
The patterns also tend to change the split per environment while they run, so
the split belongs on the binding as well as on the Endpoint.

| Sample | Pattern | Uses |
|---|---|---|
| [`01-canary.yaml`](endpoints/01-canary.yaml) | Release line takes a growing share | `weight`, binding `backendOverrides` |
| [`02-canary-with-testers.yaml`](endpoints/02-canary-with-testers.yaml) | Testers always reach the canary | `match.headers` |
| [`03-blue-green.yaml`](endpoints/03-blue-green.yaml) | Preview, then an atomic switch | `match.headers`, binding `backendOverrides` |
| [`04-mirroring.yaml`](endpoints/04-mirroring.yaml) | Shadow traffic, responses discarded | `mirror` |
| [`05-ab-testing.yaml`](endpoints/05-ab-testing.yaml) | Users held in one variant | `match.cookies`, `sessionPersistence` |
| [`06-path-fan-out.yaml`](endpoints/06-path-fan-out.yaml) | One API, several components by path | `match.path`, groups |

## The shape

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Endpoint
metadata:
  name: trips-api
spec:
  owner: {projectName: default}
  type: {kind: ClusterEndpointType, name: http}
  visibility: [external]
  autoDeploy: true
  parameters:
    pathPrefix: /trips               # the root every match path is relative to
  sessionPersistence:                # optional: keep a client on the backend it first reached
    type: Cookie                     # Cookie | Header
    name: trips-variant
    absoluteTimeout: 168h
  backendRefs:
    - name: stable                   # handle, unique in the Endpoint
      component:
        name: trips
        endpoint: http
        releaseLine: next            # optional: a release line of the component
      weight: 100                    # share within its group; default 1
      match:                         # optional: which requests this backend's group takes
        path: {type: PathPrefix, value: /next}            # PathPrefix | Exact
        headers: [{name: x-canary, value: "true"}]        # type: Exact (default) | RegularExpression
        cookies: [{name: exp, value: b}]                  # exact value
      mirror:                        # optional: copy requests here, discard the responses
        percent: 10                  # 1-100; only with no weight
```

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: EndpointReleaseBinding
metadata:
  name: trips-api-production
spec:
  owner: {projectName: default, endpointName: trips-api}
  environment: production
  endpointRelease: trips-api-7c9f5d
  backendOverrides:                  # this environment's weights and mirror shares
    - {name: stable, weight: 95}
    - {name: canary, weight: 5}
    - {name: shadow, mirror: {percent: 50}}
```

## How a request finds its backend

### Groups

Backends with **the same `match`** form a group. Backends with no `match` form
the **default group**. A group is one routing decision:

1. A request is checked against the groups with a `match`, most specific first.
   Within a match, conditions are ANDed: path, every header and every cookie.
2. The first group that matches takes the request, and its weights pick the
   backend.
3. A request no group claims goes to the default group, by weight. With no
   default group, it gets a 404.

The groups in sample 06:

| Group | Match | Backends (weight) |
|---|---|---|
| 1 | `/payments` | `payments` (90), `payments-v2` (10) |
| 2 | `/orders` + `x-beta: true` | `orders-beta` |
| 3 | `/orders` | `orders` |
| default | none | `catalog` |

A backend belongs to exactly one group. To have the canary in the weighted
split *and* under a tester header, list it twice under two names, as sample 02
does. Both entries point at the same release line.

### Precedence

Groups are ordered by specificity, as in Gateway API, so the order of
`backendRefs` never matters:

1. An `Exact` path beats a `PathPrefix`.
2. A longer `PathPrefix` beats a shorter one.
3. More header conditions win. Cookies count as headers.
4. A group with a match beats the default group.

Two groups that differ but are **equally specific** would be ambiguous for a
request that satisfies both, for example one matching `x-canary: true` and one
matching cookie `exp=b`. The Endpoint is rejected rather than having one
silently win.

### Weights

- A weight is a share **within its group**, not of the whole Endpoint. In
  sample 06, `payments-v2: 10` means 10% of `/payments` traffic.
- Unset means 1, as today.
- A group whose weights are all 0 takes no traffic. Its requests fall through
  to the next group that matches, or to the default group. This stages a
  header canary before it's deployed, the same way a weight-0 backend is staged
  today.

### Paths and rewrites

`match.path` is relative to `parameters.pathPrefix`, so a backend can't claim
anything outside the Endpoint's URL. For each backend, everything up to and
including its matched prefix is replaced by that backend's base path:

| Request | Group | Backend base path | Backend sees |
|---|---|---|---|
| `/shop/orders/42` | `/orders` | `/api/orders` | `/api/orders/42` |
| `/shop/payments/7` | `/payments` | `/` | `/7` |
| `/shop/products/3` | default | `/catalog` | `/catalog/products/3` |

Each group renders as its own route rule, so each can rewrite differently.
**The shared-base-path rule only applies within a group**, not across the
Endpoint, and fan-out works with components that were never designed to share
one.

Because every match hangs off `pathPrefix`, moving it moves the whole API.
That is one more reason for the root path to be immutable once the Endpoint
exists.

### Mirrors

A backend with `mirror` receives a copy of the given share of its group's
requests: the default group's if it has no `match`, otherwise its match
group's. Its responses are discarded and it never answers a client.

- A mirror has no weight.
- A group needs at least one weighted backend to mirror from.
- The copy is sent as is, writes included. Point a mirror at a backend with
  its own data, or mirror only reads (for example with a `match` on a read-only
  path).

### Session persistence

Weights pick a backend per request. For an A/B test, a client has to stay on
the variant it first reached, or the experiment's numbers mean nothing. With
`sessionPersistence`, the gateway sets a cookie (or reads a header) on a
client's first request and routes its later requests to the same backend, for
`absoluteTimeout`. It applies to every group of the Endpoint.

The alternative is to have the application assign the bucket and route on it
with `match.cookies`. That keeps assignment and analytics in one place (sample
05, `trips-api`).

## Per-environment overrides

Weights and mirror shares are the part that changes while a pattern runs:
5% today, 25% tomorrow, and different in staging than in production. Cutting
a new EndpointRelease for every step would promote a step meant for staging
through every environment.

`backendOverrides` on an `EndpointReleaseBinding` changes them for that
environment only:

- It may set `weight` and `mirror.percent` of a backend in the pinned release,
  by name.
- It may not add or remove backends, or change a `match`. Those change what
  the API routes, so they belong in a release and go through promotion.
- An override that names a backend missing from the pinned release is ignored
  and reported on the binding's conditions, so promoting a release that drops
  a backend doesn't fail.
- The Endpoint's weights are the defaults for an environment with no
  overrides, such as a new one.

Blue/green (sample 03) is the clearest case. The switch is
`blue: 0, green: 100` on the production binding, the rollback is the reverse,
and both take effect immediately and only there.

## The samples

### 01: Canary

Two lines of component `trips`. The Endpoint starts the canary at 0, staging
runs it at 50%, and production at 5%, all through binding overrides on one
release. The canary grows in production by editing one binding.

### 02: Canary with testers

Two groups:

- **The header group:** `x-canary: true` always reaches the canary, so testers
  can use it at 0% or 5%.
- **The default group:** everyone else, split 95/5.

### 03: Blue/green

Blue (`trips-python`) serves everyone. Green (`trips-go`) is fully deployed
but only answers `x-preview: green`, so it can be checked through the real URL
first. The production binding then flips the weights. Clients never change
the URL, and the preview header keeps working after the switch.

### 04: Mirroring

`trips-go` receives a copy of 25% of the requests `trips-python` answers. Its
responses are discarded, so it can be compared under real traffic, with no
risk to clients. Staging mirrors 100% through an override.

### 05: A/B testing

Two Endpoints showing the two ways to assign users:

- **`trips-api`:** the application sets cookie `exp`, and `exp=b` routes to
  line `exp-b`. Everyone else gets variant A.
- **`trips-api-weighted`:** the gateway assigns new users 80/20, and
  `sessionPersistence` keeps them there for 7 days.

### 06: Path fan-out

`shop-api` puts three components under `/shop`:

- `/orders` goes to `orders`. A more specific `/orders` + `x-beta: true` group
  sends beta users to its `beta` line.
- `/payments` is one group of two backends, a canary inside the fan-out.
- Everything else goes to `catalog`, the default group.

What a gateway type would render, as one `HTTPRoute` with rules in precedence
order:

```yaml
rules:
  - matches: [{path: {type: PathPrefix, value: /shop/payments}}]
    filters: [{type: URLRewrite, urlRewrite: {path: {type: ReplacePrefixMatch, replacePrefixMatch: <payments base path>}}}]
    backendRefs:
      - {name: payments, port: ..., weight: 90}
      - {name: payments-v2, port: ..., weight: 10}
  - matches:
      - path: {type: PathPrefix, value: /shop/orders}
        headers: [{name: x-beta, value: "true"}]
    filters: [{type: URLRewrite, urlRewrite: {path: {type: ReplacePrefixMatch, replacePrefixMatch: <orders base path>}}}]
    backendRefs: [{name: orders-beta, port: ...}]
  - matches: [{path: {type: PathPrefix, value: /shop/orders}}]
    filters: [{type: URLRewrite, urlRewrite: {path: {type: ReplacePrefixMatch, replacePrefixMatch: <orders base path>}}}]
    backendRefs: [{name: orders, port: ...}]
  - matches: [{path: {type: PathPrefix, value: /shop}}]
    filters: [{type: URLRewrite, urlRewrite: {path: {type: ReplacePrefixMatch, replacePrefixMatch: <catalog base path>}}}]
    backendRefs: [{name: catalog, port: ...}]
```

Gateway API orders rules by specificity itself, so the order above is for
reading. Istio orders `http` routes by list position, so a mesh type would
have to emit them in exactly this order.

## What an Endpoint type would see

Grouping, precedence, override merging and rewrite prefixes are the same for
every type, so the platform would compute them once. Templates wouldn't have
to rebuild them in CEL. Besides today's flat `backends`, the CEL context would
carry `routes`, already in precedence order with the binding's overrides
applied:

```yaml
routes:
  - match:
      path: {type: PathPrefix, value: /shop/payments} # pathPrefix already joined
      headers: []
      cookies: []
    backends:
      - {name: payments, serviceName: ..., port: ..., weight: 90, basePath: ...}
      - {name: payments-v2, serviceName: ..., port: ..., weight: 10, basePath: ...}
    mirrors: []
  - match:
      path: {type: PathPrefix, value: /shop/orders}
      headers: [{name: x-beta, value: "true", type: Exact}]
      cookies: []
    backends:
      - {name: orders-beta, serviceName: ..., port: ..., weight: 1, basePath: ...}
    mirrors: []
  # ...
  - match: null                                       # the default group, always last
    backends:
      - {name: catalog, ...}
    mirrors: []
```

Each type maps `routes` onto what its data plane can express, and declares
through its validations what it can't:

| Feature | Gateway API (`HTTPRoute`) | Istio (`VirtualService`) |
|---|---|---|
| Path, header match | core | `match.uri`, `match.headers` |
| Cookie match | a `RegularExpression` match on the `Cookie` header | `match.headers.cookie` regex |
| Weight within a group | `backendRefs[].weight` | `route[].weight` |
| Per-group rewrite | `URLRewrite` on the rule | `rewrite.uri` on the route |
| Mirror | `RequestMirror` filter, `percent` (extended, v1.2+) | `mirror`, `mirrorPercentage` |
| Session persistence | `sessionPersistence` on the rule (experimental) | `DestinationRule` consistent hash on a cookie |

A type that can't do one, such as a gateway without `RequestMirror`, rejects
an Endpoint that uses it, for example with `routes.all(r, r.mirrors.size() == 0)`,
so the Endpoint fails at admission rather than half-applied.

## Validation

The Endpoint would be rejected when:

- two backends share a `name`;
- a `match` is present but empty (it would duplicate the default group);
- two different groups are equally specific;
- a backend has both `weight` and `mirror`, or a mirror's group has no weighted
  backend;
- `mirror.percent` is outside 1-100;
- a `match.path` doesn't start with `/`.

A binding's `backendOverrides` would be rejected when it sets `mirror` on a
weighted backend, or `weight` on a mirror.

## Open questions

- **Per-group overrides.** `backendOverrides` addresses backends by name. A
  backend listed twice, as in sample 02, needs two overrides. That's simple,
  but it means a backend's identity is its entry, not its target.
- **Validating weights per type.** Istio expects a route's weights to sum to
  100, and Gateway API doesn't. Should the platform normalize within each group,
  or leave it to the type?
- **Who may override.** `backendOverrides` is operational: whoever runs a
  rollout in production. It may want its own permission, separate from editing
  the Endpoint.
- **Progressive delivery.** Moving a canary from 5% to 100% by hand is fine for
  a sample. Automating it, with steps, pauses and metric checks, would drive
  the same `backendOverrides`, so the shape doesn't stand in the way.
