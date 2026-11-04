# Scenario 2: Several Endpoint Types per Protocol

Here a protocol can match more than one Endpoint type. An HTTP endpoint can be
exposed as a plain route or as a managed API with API keys and rate limits.
The platform engineer marks one of them as the **default**. A developer who
says nothing gets the default, and one who wants the other type names it with
`via`.

| Alias | Protocols | Endpoint type | Default | Renders |
|---|---|---|---|---|
| `unmanaged` | `HTTP`, `GraphQL`, `Websocket` | `s2-unmanaged-http` | yes | One `HTTPRoute` per requested visibility |
| `managed` | `HTTP` | `s2-managed-http` | no | The same `HTTPRoute`s, plus a kgateway `TrafficPolicy` with API-key auth and rate limiting |

The component type, `s2-service`, renders the workload and its in-cluster
`Service`, but no routes of its own. Everything beyond the project goes through
an `Endpoint`.

## Files

```
platform/
├── s2-service-component-type.yaml          # ClusterComponentType s2-service
├── s2-unmanaged-http-endpoint-type.yaml    # ClusterEndpointType s2-unmanaged-http
└── s2-managed-http-endpoint-type.yaml      # ClusterEndpointType s2-managed-http
```

### `s2-service`

The same as scenario 1's `s1-service` except for `allowedEndpoints`. Each entry
now has a `name`, which is the alias a descriptor's `via` selects, and HTTP
matches two of them:

```yaml
allowedEndpoints:
  - name: unmanaged
    default: true
    protocols: [HTTP, GraphQL, Websocket]
    ref: {kind: ClusterEndpointType, name: s2-unmanaged-http}
  - name: managed
    protocols: [HTTP]
    ref: {kind: ClusterEndpointType, name: s2-managed-http}
```

At most one entry can be the default for a protocol. An entry without
`default: true` is still allowed, but only when it is chosen by name.

### `s2-unmanaged-http`

Scenario 1's `s1-http` under a new name. It renders a plain route with nothing
enforced, and takes the same options:

| Setting | Where | Default | Meaning |
|---|---|---|---|
| `pathPrefix` | `parameters` | `/` | Public path the endpoint is served under |
| `cors` | `parameters` | `false` | Allow browsers to call the endpoint cross-origin (external route) |
| `hostname` | `environmentConfigs` | generated | Serve the external route on your own domain in this environment |

### `s2-managed-http`

The same routes, plus a kgateway `TrafficPolicy` that covers all of them:

| Setting | Where | Default | Meaning |
|---|---|---|---|
| `pathPrefix`, `cors` | `parameters` | as above | As above |
| `requireApiKey` | `parameters` | `true` | Reject requests without a valid key in the `api-key` header |
| `requestsPerMinute` | `parameters` | `60` | Requests per minute the gateway admits, per gateway replica; excess gets 429 |
| `hostname` | `environmentConfigs` | generated | As above |
| `apiKeys` | `environmentConfigs` | none | Accepted keys by client name, rendered into a `Secret` |

With `requireApiKey` on and no keys set, every request is rejected. The keys
are stored in plain text on the binding, so treat this as a demo, not a
credential store. The type only accepts `HTTP` backends.

It needs the kgateway data plane gateway, and the data-plane cluster-agent must
be allowed to manage `gateway.kgateway.dev` `trafficpolicies` (the data-plane
chart grants this).

### URLs

| Type | Scheme | Path | Security |
|---|---|---|---|
| `s2-unmanaged-http` | `https` with an HTTPS listener, otherwise `http` | `pathPrefix` | `none` |
| `s2-managed-http` | likewise | `pathPrefix` | `apiKey` in the `api-key` header while `requireApiKey` is on |

Both types report these on each `EndpointReleaseBinding`'s `status.urls`, so a
client can see which credential an Endpoint expects.

## Set up

Apply the Endpoint types first, since the component type refers to them:

```bash
kubectl apply \
  -f platform/s2-unmanaged-http-endpoint-type.yaml \
  -f platform/s2-managed-http-endpoint-type.yaml
kubectl apply -f platform/s2-service-component-type.yaml
```

The component type then shows up as `deployment/s2-service`.

## How an exposure picks its type

The descriptor names a workload endpoint; its protocol and the expose entry's
`via` choose the `allowedEndpoints` entry:

| `via` | Picks |
|---|---|
| omitted | The entry marked `default` for the protocol |
| an alias | The entry with that name, if it matches the protocol |

A named alias never falls back to the default. A typo or a protocol mismatch
fails the build rather than quietly exposing the endpoint through a weaker
type. The expose entry's `parameters` are then validated against the chosen
type's schema.

### The default: no `via`

```yaml
endpoints:
  - name: greeter-api
    port: 9090
    protocol: HTTP
    expose:
      - visibility: [external]
        parameters: {pathPrefix: /greeter}
```

`HTTP` has one default entry, `unmanaged`, so this creates an
`s2-unmanaged-http` Endpoint: a plain route.

### Choosing the other type: `via: managed`

```yaml
endpoints:
  - name: greeter-api
    port: 9090
    protocol: HTTP
    expose:
      - visibility: [external]
        via: managed
        parameters:
          pathPrefix: /greeter
          requestsPerMinute: 30
```

This creates an `s2-managed-http` Endpoint, with API keys required and 30
requests per minute.

### Pinning the default: `via: unmanaged`

Naming the default explicitly gives the same Endpoint as omitting `via`, with
one difference: it no longer follows the default. If the platform engineer
later moves `default: true` to `managed`, the next build switches every
exposure without `via` to `s2-managed-http`, replacing the Endpoint under the
same name. An exposure that says `via: unmanaged` stays where it is.

Use no `via` to get whatever the platform recommends. Name the alias when the
endpoint depends on a type's behavior, such as staying open with no keys.

### Both on one endpoint

Each expose entry is resolved on its own, so one workload endpoint can be a
managed API to the outside and a plain route inside:

```yaml
expose:
  - name: external
    visibility: [external]
    via: managed
    parameters: {pathPrefix: /greeter, requestsPerMinute: 30}
  - name: internal
    visibility: [internal]
    parameters: {pathPrefix: /greeter}
```

### What fails

Each error names the endpoint and expose entry, and fails the build before
anything is written:

| Descriptor | Error |
|---|---|
| `via: premium` on an HTTP endpoint | `via "premium" is not an allowed Endpoint type for protocol HTTP (available: unmanaged, managed)` |
| `via: managed` on a GraphQL endpoint | `via "managed" is not an allowed Endpoint type for protocol GraphQL (available: unmanaged)` |
| Any expose entry on a gRPC endpoint | `no default Endpoint type for protocol gRPC; name one with via (available: none)` |
| `requestsPerMinute: fast` with `via: managed` | `parameters do not fit ClusterEndpointType s2-managed-http: ...` |

## Try it

Create a component of type `deployment/s2-service` that builds
[`service-go-greeter-api`](../../service-go-greeter-api). Its `workload.yaml`
uses the same two aliases as `s2-service`, so it exposes `greeter-api` through
both types:

```yaml
expose:
  - name: external
    visibility: [external]
    via: managed
    parameters:
      pathPrefix: /greeter-api
      requestsPerMinute: 30
      cors: true
      requireApiKey: false
  - name: internal
    visibility: [internal]
    via: unmanaged
    parameters:
      pathPrefix: /greeter-api
```

With a component named `greeter`, the build creates:

| Endpoint | Type | Visibility | Security |
|---|---|---|---|
| `greeter-greeter-api-external` | `s2-managed-http` | `external` | `none` (`requireApiKey: false`), 30 requests per minute |
| `greeter-greeter-api-internal` | `s2-unmanaged-http` | `internal` | `none` |

To see the default at work, delete `via: unmanaged` from the internal entry
and rebuild. The Endpoint is unchanged.

### Check

```bash
# The Endpoints and the type each one resolved to
kubectl get endpoints.openchoreo.dev -n default \
  -l openchoreo.dev/source-component=greeter \
  -o custom-columns='NAME:.metadata.name,TYPE:.spec.type.name,VISIBILITY:.spec.visibility'

# Their URLs and the security each declares
kubectl get endpointreleasebinding -n default \
  -o custom-columns='NAME:.metadata.name,EXTERNAL:.status.urls.external,INTERNAL:.status.urls.internal'
```

Call the managed route past its limit to see the 429s:

```bash
for i in $(seq 1 40); do
  curl -s -o /dev/null -w '%{http_code}\n' \
    "http://<host>:<port>/greeter-api/greeter/greet?name=OpenChoreo"
done | sort | uniq -c
```

### Turn on API keys

Set `requireApiKey: true` (or drop it, since it defaults to `true`) and
rebuild. Then give the environment some keys on the external Endpoint's
binding:

```bash
ERB=$(kubectl get endpointreleasebinding -n default -o jsonpath=\
'{.items[?(@.spec.owner.endpointName=="greeter-greeter-api-external")].metadata.name}')
kubectl patch endpointreleasebinding "$ERB" -n default --type merge \
  -p '{"spec":{"endpointTypeEnvironmentConfigs":{"apiKeys":{"demo-client":"s3cr3t"}}}}'

curl -H 'api-key: s3cr3t' \
  "http://<host>:<port>/greeter-api/greeter/greet?name=OpenChoreo"
```

A request without the header, or with a wrong key, is rejected. The internal
Endpoint is unmanaged, so it keeps answering without a key.

## Trade-offs

What this scenario adds over scenario 1:

- **Variants per protocol.** The platform offers several ways to expose HTTP
  and decides which one is the norm.
- **Safe by default, explicit otherwise.** Saying nothing gets the
  recommended type. Anything else is opted into by name, and a bad name never
  falls back.
- **Policy in one place.** Moving `default: true` changes every exposure
  without `via` on its next build.

What it costs:

- **Developers see a choice.** They need to know the aliases and what each
  type does, and the portal shows a picker for HTTP.
- **Parameters depend on the choice.** `requestsPerMinute` means something
  only to `managed`, so switching `via` can mean changing `parameters` too.
- **A moved default is a rollout.** Each component switches type on its next
  build, not all at once. Until then, the same descriptor gives different
  types across components.
- **Aliases are per component type.** Another component type can use
  `managed` for something else, so `via` is only meaningful together with the
  component type.
