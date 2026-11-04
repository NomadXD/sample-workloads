# Scenario 4: Behavior per Environment

One Endpoint type, and the same descriptor everywhere. What the gateway
enforces changes with the environment: the endpoint is open in development,
rate limited in staging, and requires API keys in production. The developer
never chooses differently per environment, and the platform engineer decides
the policy once, in the type.

| Environment | Protection | Renders | URL security |
|---|---|---|---|
| `development` | `open` | `HTTPRoute`s | `none` |
| `staging` | `rateLimited` | `HTTPRoute`s + `TrafficPolicy` (300 requests per minute) | `none` |
| any other | `apiKey` | `HTTPRoute`s + `TrafficPolicy` (API keys, 60 requests per minute) + key `Secret` | `apiKey` in the `api-key` header |

Only the environments named in the type are relaxed. A new environment, such as
`production` or a second region `prod-eu`, gets the strictest protection until
the type says otherwise. The type fails closed.

## Files

```
platform/
├── s4-service-component-type.yaml   # ClusterComponentType s4-service
└── s4-http-endpoint-type.yaml       # ClusterEndpointType s4-http
```

### `s4-service`

The same as scenario 1's `s1-service`: it renders the workload and its
`Service`, but no routes. `s4-http` is the only type for `HTTP` and is the
default, so a descriptor's expose entries need no `via`.

### `s4-http`

Same routes in every environment. Only the policy on them changes. The
settings are split by who owns them:

| Setting | Where | Set by | Default | Meaning |
|---|---|---|---|---|
| `pathPrefix` | `parameters` | developer | `/` | Public path, the same in every environment |
| `cors` | `parameters` | developer | `false` | Allow browsers to call the endpoint cross-origin (external route) |
| `protection` | `environmentConfigs` | whoever deploys | follows the environment | `open`, `rateLimited` or `apiKey` |
| `requestsPerMinute` | `environmentConfigs` | whoever deploys | 300 in `staging`, 60 elsewhere | Per gateway replica; excess gets 429 |
| `apiKeys` | `environmentConfigs` | whoever deploys | none | Accepted keys by client name |
| `hostname` | `environmentConfigs` | whoever deploys | generated | Serve the external route on your own domain |

`environmentConfigs` lives on each environment's `EndpointReleaseBinding`. So
an override applies to one Endpoint in one environment, and the environment
default covers everything else.

Every template that depends on the protection computes it the same way:

```cel
environmentConfigs.?protection.orValue(
  metadata.environmentName == "development" ? "open"
  : metadata.environmentName == "staging" ? "rateLimited" : "apiKey")
```

The type uses this in four places:

- The `TrafficPolicy` is rendered only when protection isn't `open`.
- Its `apiKeyAuth` is set only for `apiKey`.
- The key `Secret` is rendered only for `apiKey`.
- `urls.*.security` reports `apiKey` or `none`, so a client can read from the
  binding whether this environment wants a key.

An open environment renders plain `HTTPRoute`s and works on any Gateway API
implementation. Protected environments need the kgateway data plane gateway,
and the data-plane cluster-agent must be allowed to manage
`gateway.kgateway.dev` `trafficpolicies`.

## Set up

```bash
kubectl apply -f platform/s4-http-endpoint-type.yaml
kubectl apply -f platform/s4-service-component-type.yaml
```

The examples use the getting-started environments: `development`, `staging`
and `production`. If yours have other names, change the names in the type.

## Try it

Create a component named `greeter`, of type `deployment/s4-service`, that
builds [`service-go-greeter`](../../service-go-greeter). Its `workload.yaml`
exposes `greeter-api` twice and sets only developer parameters:

```yaml
expose:
  - name: external
    visibility: [external]
    parameters: {cors: true, pathPrefix: /greeter-service-source}
  - name: internal
    visibility: [internal]
    parameters: {cors: true, pathPrefix: /greeter-service-source}
```

The build creates `greeter-greeter-api-external` and
`greeter-greeter-api-internal`. Both are deployed to `development`, the first
environment, by `autoDeploy`. Their bindings are named
`<endpoint>-<environment>`.

Some helpers:

```bash
EP=greeter-greeter-api-external

# The URL and the security an environment declares
url() { kubectl get endpointreleasebinding "$EP-$1" -n default -o jsonpath='{.status.urls.external}'; echo; }

# Status codes for 20 calls, with optional extra curl arguments
hit() { local u=$1; shift
  for i in $(seq 1 20); do curl -s -o /dev/null -w '%{http_code}\n' "$@" "$u"; done | sort | uniq -c; }
```

### Development: open

```bash
url development        # ... "security":{"scheme":"none"}
hit "http://<dev-host>:<port>/greeter-service-source/greeter/greet"
#   20 200
```

No `TrafficPolicy` is rendered here.

### Promote to staging: rate limited

Promotion is explicit beyond the first environment. Promote the component
first, since an Endpoint's binding waits for its backend to be deployed in the
same environment. Then promote both Endpoints:

```bash
promote() { local env=$1
  kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: ReleaseBinding
metadata: {name: greeter-$env, namespace: default}
spec:
  owner: {projectName: default, componentName: greeter}
  environment: $env
  releaseName: $(kubectl get component greeter -n default -o jsonpath='{.status.latestRelease.name}')
EOF
  for ep in greeter-greeter-api-external greeter-greeter-api-internal; do
    kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: EndpointReleaseBinding
metadata: {name: $ep-$env, namespace: default}
spec:
  owner: {projectName: default, endpointName: $ep}
  environment: $env
  endpointRelease: $(kubectl get endpoints.openchoreo.dev $ep -n default -o jsonpath='{.status.latestRelease.name}')
EOF
  done
}

promote staging
```

These are the same Endpoint releases as in development, so the routes are
identical. Only the `TrafficPolicy` is added, at 300 requests per minute. To
see 429s without sending 300 requests, lower the limit on this one binding:

```bash
kubectl patch endpointreleasebinding "$EP-staging" -n default --type merge \
  -p '{"spec":{"endpointTypeEnvironmentConfigs":{"requestsPerMinute":10}}}'
hit "http://<staging-host>:<port>/greeter-service-source/greeter/greet"
#   10 200
#   10 429
```

### Promote to production: API keys

```bash
promote production
url production         # ... "security":{"scheme":"apiKey","apiKey":{"in":"header","name":"api-key"}}
hit "http://<prod-host>:<port>/greeter-service-source/greeter/greet"
#   20 401
```

Every request is rejected until the environment has keys:

```bash
kubectl patch endpointreleasebinding "$EP-production" -n default --type merge \
  -p '{"spec":{"endpointTypeEnvironmentConfigs":{"apiKeys":{"demo-client":"s3cr3t"}}}}'
hit "http://<prod-host>:<port>/greeter-service-source/greeter/greet" -H 'api-key: s3cr3t'
#   20 200
```

### Override one Endpoint in one environment

The internal Endpoint is also protected in production, because protection
follows the environment, not the visibility. Callers inside the organization
are trusted, so open just that binding:

```bash
kubectl patch endpointreleasebinding greeter-greeter-api-internal-production -n default --type merge \
  -p '{"spec":{"endpointTypeEnvironmentConfigs":{"protection":"open"}}}'
```

The internal route in production loses its `TrafficPolicy`. The external
route keeps its keys.

## Trade-offs

What this scenario gets right:

- **One descriptor, every environment.** The developer's input, `pathPrefix`
  and `cors`, never changes on promotion. Promoting moves the same Endpoint
  release forward, and the environment adds the policy.
- **Policy owned by the platform.** Which environments are protected, and how,
  lives in one type. The operator can override a binding without touching the
  descriptor.
- **Fails closed.** Only the environments named in the type are relaxed. A new
  one is protected from its first deployment.
- **Visible to clients.** `status.urls` reports the security per environment,
  so a portal or client can show that production wants a key.

What it doesn't handle:

- **Environments matched by name.** An `EndpointType`'s CEL context doesn't
  include `Environment.spec.isProduction` or the environment's labels, so the
  type has to list names. Exposing them, as `environment.isProduction` and
  `environment.labels`, would let the type say "relax non-production
  environments" without knowing their names.
- **The same expression four times.** An `EndpointType` has no shared
  variables, so the protection rule is repeated in each template that needs
  it. A change has to be made in all four.
- **Overrides are per binding.** An override applies to one Endpoint in one
  environment. "Open every internal Endpoint in production" has no single
  switch. It's either a binding patch per Endpoint or a rule in the type, for
  example one keyed on visibility.
- **Keys in plain text.** `apiKeys` sits on the binding and is rendered into a
  `Secret`. This is a demo, not a credential store. A real type would reference
  an existing secret instead.
