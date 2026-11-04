# Scenario 1: One Endpoint Type per Protocol

The simplest way to offer Endpoints: each protocol has exactly one Endpoint
type, so a developer never picks one. The workload endpoint's protocol decides
how it is exposed.

| Protocol | Endpoint type | Renders |
|---|---|---|
| `HTTP`, `GraphQL`, `Websocket` | `s1-http` | One `HTTPRoute` per requested visibility |
| `gRPC` | `s1-grpc` | One `GRPCRoute` per requested visibility |
| `TCP`, `UDP` | none | Cannot be exposed |

The component type, `s1-service`, renders the workload and its in-cluster
`Service`, but no routes of its own. Everything beyond the project goes through
an `Endpoint`.

## Files

```
platform/
├── s1-service-component-type.yaml   # ClusterComponentType s1-service
├── s1-http-endpoint-type.yaml       # ClusterEndpointType s1-http
└── s1-grpc-endpoint-type.yaml       # ClusterEndpointType s1-grpc
```

### `s1-service`

A copy of the default `service` component type with two changes:

- **No routes.** The `httproute-*` resources are gone; the type renders the
  `Deployment`, the `Service`, and the config and secret resources.
- **`allowedEndpoints`** lists one entry per protocol family, each marked
  `default: true` and carrying no `name`:

  ```yaml
  allowedEndpoints:
    - default: true
      protocols: [HTTP, GraphQL, Websocket]
      ref: {kind: ClusterEndpointType, name: s1-http}
    - default: true
      protocols: [gRPC]
      ref: {kind: ClusterEndpointType, name: s1-grpc}
  ```

  An unnamed entry cannot be selected with `via`, so the default is the only
  choice. A descriptor that sets `via` fails the build.

### `s1-http`

Routes `HTTP`, `GraphQL` and `Websocket` backends. For each visibility the
Endpoint requests, and the environment has a gateway for, it renders one
`HTTPRoute` that:

- serves a hostname unique to the Endpoint, environment and project;
- matches `parameters.pathPrefix` and rewrites it to the backends' base path;
- splits traffic across the backends by weight;
- adds a `CORS` filter on the external route when `parameters.cors` is on.

| Setting | Where | Default | Meaning |
|---|---|---|---|
| `pathPrefix` | `parameters` | `/` | Public path the endpoint is served under |
| `cors` | `parameters` | `false` | Allow browsers to call the endpoint cross-origin (external route) |
| `hostname` | `environmentConfigs` | generated | Serve the external route on your own domain in this environment |

The validations reject a non-HTTP backend, and backends that don't share one
base path, because a single rewrite filter serves them all.

### `s1-grpc`

Routes `gRPC` backends with one `GRPCRoute` per visibility, split by weight.
gRPC has no base path (the request path is `/<package>.<Service>/<Method>`), so
the Endpoint is isolated by hostname alone and there is no `pathPrefix`.
`environmentConfigs.hostname` works as in `s1-http`.

### URLs

Both types declare `urls` for `internal` and `external`, reported on each
`EndpointReleaseBinding`'s `status.urls` for the visibilities the Endpoint
requests:

| Type | Scheme | Host | Path | Security |
|---|---|---|---|---|
| `s1-http` | `https` with an HTTPS listener, otherwise `http` | `<generated>.<gateway host>`, or `hostname` (external) | `pathPrefix` | `none` |
| `s1-grpc` | `grpcs` or `grpc`, likewise | likewise | none | `none` |

The generated label is `oc_dns_label(endpointName, environment, project,
namespace)`.

## Set up

Apply the Endpoint types first, since the component type refers to them:

```bash
kubectl apply \
  -f platform/s1-http-endpoint-type.yaml \
  -f platform/s1-grpc-endpoint-type.yaml
kubectl apply -f platform/s1-service-component-type.yaml
```

The component type then shows up as `deployment/s1-service`.

## Try it

Create a component of type `deployment/s1-service` that builds
[`service-go-greeter`](../../service-go-greeter). Its `workload.yaml` exposes
the `greeter-api` endpoint twice, without `via`:

```yaml
endpoints:
  - name: greeter-api
    port: 9090
    protocol: HTTP
    expose:
      - name: external
        visibility: [external]
        parameters: {cors: true, pathPrefix: /greeter-service-source}
      - name: internal
        visibility: [internal]
        parameters: {cors: true, pathPrefix: /greeter-service-source}
```

`HTTP` resolves to `s1-http` through the default entry. When the build
publishes the workload, the API creates one Endpoint per `expose` entry, named
`<component>-<endpoint>-<expose name>`. With a component named `greeter`:

| Endpoint | Type | Visibility | Path |
|---|---|---|---|
| `greeter-greeter-api-external` | `s1-http` | `external` | `/greeter-service-source` |
| `greeter-greeter-api-internal` | `s1-http` | `internal` | `/greeter-service-source` |

Each Endpoint is labeled `openchoreo.dev/source-component: greeter` and is
owned by the component, so deleting the component deletes its Endpoints.

A gRPC endpoint needs nothing extra. The protocol picks `s1-grpc`, and its
`expose` entries take no parameters:

```yaml
endpoints:
  - name: grpc
    port: 50051
    protocol: gRPC
    expose:
      - visibility: [external]
```

### Check

```bash
# The Endpoints the build created
kubectl get endpoints.openchoreo.dev -n default \
  -l openchoreo.dev/source-component=greeter

# Their URLs in an environment
kubectl get endpointreleasebinding -n default \
  -o custom-columns='NAME:.metadata.name,EXTERNAL:.status.urls.external,INTERNAL:.status.urls.internal'

# Call the external one
curl "http://<host>:<port>/greeter-service-source/greeter/greet?name=OpenChoreo"
```

## Release lines

A release line builds and deploys beside the main line. Each line renders its
own `Service`, named `<component>` on the main line and
`releaseline.ComponentLineName(<component>, <line>)` on a release line, so the
lines' pods never share an address. The line's descriptor generates the line's
own Endpoints.

Add a `next` line to `greeter`, built from another branch:

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ReleaseLine
metadata:
  name: next
  namespace: default
spec:
  owner:
    projectName: default
    componentName: greeter
  workflow:
    parameters:
      repository:
        revision:
          branch: release-lines-next
```

When the line's build publishes its workload, the same `expose` entries
create a second set of Endpoints, named `<component>-<line>-<endpoint>-<expose
name>`:

| Endpoint | Routes to | Owned by |
|---|---|---|
| `greeter-greeter-api-external` | `greeter`, main line | Component `greeter` |
| `greeter-next-greeter-api-external` | `greeter`, line `next` | ReleaseLine `next` |

The line's Endpoints carry `openchoreo.dev/source-release-line: next`, and
their backend sets `component.releaseLine: next`. Both lines ask for the same
`pathPrefix`, but each Endpoint gets its own hostname, so the routes don't
collide. Deleting the line deletes its Endpoints; the main line's URL stays.

```bash
kubectl get endpoints.openchoreo.dev -n default \
  -l openchoreo.dev/source-component=greeter \
  -L openchoreo.dev/source-release-line
```

### Canary through one URL

The line's own Endpoints give `next` a separate URL for previews. To shift
real traffic, split one URL across both lines with a hand-made Endpoint:

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Endpoint
metadata:
  name: greeter-canary
  namespace: default
spec:
  owner:
    projectName: default
  type:
    kind: ClusterEndpointType
    name: s1-http
  visibility: [external]
  parameters:
    pathPrefix: /greeter
  backendRefs:
    - name: main
      component: {name: greeter, endpoint: greeter-api}
      weight: 90
    - name: next
      component: {name: greeter, endpoint: greeter-api, releaseLine: next}
      weight: 10
```

Keep the split off the descriptor-generated Endpoints. Each build re-applies
their `backendRefs` from the descriptor, so a backend added by hand is dropped
on the next build (only the weights of the backends it keeps survive).

Both lines must serve under the same base path, since `s1-http` uses one
rewrite filter for all backends.

## Trade-offs

What this scenario gets right:

- **No choice to make.** Developers never see Endpoint types. The protocol is
  enough, and the portal can expose an endpoint with one click.
- **One place per protocol.** Changing how HTTP is exposed means editing
  `s1-http`; every component picks it up on its next Endpoint render.
- **Clean split.** The component type owns the workload and its `Service`;
  Endpoint types own everything at the gateway.

What it does not handle:

- **No variants.** Every HTTP endpoint gets the same treatment. A managed API
  with API keys and rate limits next to a plain route would need a second HTTP
  type, and so a `via` choice.
- **TCP and UDP** cannot be exposed at all.
- **One base path per Endpoint.** Backends with different base paths cannot
  share an `s1-http` Endpoint.
- **The URL can move.** `pathPrefix` (on the Endpoint) and `hostname` (on each
  binding) are mutable, so the invoke URL is not fixed for the Endpoint's
  lifetime.
