# Scenario 5: A Weight-Based Canary in the Mesh

A canary between two components, routed by the Istio mesh instead of a
gateway. Callers inside the data plane dial one host, and their sidecar sends
90% of requests to the stable component and 10% to the canary.

```
caller pod (in the mesh)
  │  GET http://greeter.<dp-namespace>.svc.cluster.local/greeter/greet
  ▼
Service greeter              ◄── rendered by the Endpoint: a name and a VIP, no pods
  │  VirtualService on that host
  ├── 90% ──► Service greeter-stable  ──► pods of component greeter-stable
  └── 10% ──► Service greeter-canary  ──► pods of component greeter-canary
```

## Files

```
platform/
└── s5-mesh-http-endpoint-type.yaml   # ClusterEndpointType s5-mesh-http
endpoint/
└── greeter-endpoint.yaml             # Endpoint greeter, fronting both components
```

## How the Endpoint relates to the components

Each component is an ordinary service with its own workload endpoint and its
own `Service`, rendered by its component type. The Endpoint doesn't replace
either. It adds a host in front of them.

| | `greeter-stable` | `greeter-canary` | Endpoint `greeter` |
|---|---|---|---|
| Workload endpoint | `greeter-api`, HTTP, port 9090 | `greeter-api`, HTTP, port 9090 | none |
| `Service` | `greeter-stable` (component type) | `greeter-canary` (component type) | `greeter`, port 80, no selector (Endpoint type) |
| Routing | none | none | `VirtualService` on `greeter.<dp-namespace>.svc.cluster.local` |
| Owned by | the component | the component | the project |

The Endpoint names each backend by component and workload endpoint:

```yaml
backendRefs:
  - name: stable
    component: {name: greeter-stable, endpoint: greeter-api}
    weight: 90
  - name: canary
    component: {name: greeter-canary, endpoint: greeter-api}
    weight: 10
```

For every environment, the Endpoint's binding resolves each backend:

1. It finds the component's `ReleaseBinding` in that environment.
2. It reads the workload endpoint's in-cluster address from
   `status.endpoints[].serviceURL`, which is the component's `Service`.
3. It reads the port from the deployed release.

The type turns each resolved backend into a `VirtualService` destination:

```yaml
http:
  - route:
      - destination:
          host: greeter-stable.<dp-namespace>.svc.cluster.local
          port: {number: 9090}
        weight: 90
      - destination:
          host: greeter-canary.<dp-namespace>.svc.cluster.local
          port: {number: 9090}
        weight: 10
```

The components don't know about the Endpoint, and neither needs
`allowedEndpoints` for it. A hand-made Endpoint can front any component in its
project.

## The Endpoint type

`s5-mesh-http` renders two objects in the project's data-plane namespace:

- **`Service <endpoint name>`**: a `ClusterIP` with port 80 (`appProtocol:
  http`) and no selector. It has no pods. It exists so callers have a name to
  resolve and the mesh has a host and a VIP to intercept.
- **`VirtualService`**: on that host, one route whose destinations are the
  backends' own Services, weighted.

It reports the in-mesh URL on `status.urls.namespace`:

```
http://greeter.<dp-namespace>.svc.cluster.local:80
```

The URL is deterministic: the Endpoint's name and the project's data-plane
namespace for the environment.

Its validations accept only `HTTP` backends and only `namespace` visibility.
Callers in the same project can always reach the backends. `namespace`
visibility makes the binding add a `NetworkPolicy` that lets callers in other
projects of the namespace reach the backends' pods too. The sidecar sends
traffic straight to those pods, not through a gateway.

| Setting | Where | Meaning |
|---|---|---|
| `waypoint` | `environmentConfigs` | Ambient mode only. Labels the host `istio.io/use-waypoint: <waypoint>` so the waypoint does the L7 routing. |

## Prerequisites

- Istio on the data plane.
- Callers in the mesh: a sidecar in sidecar mode, or a waypoint for the host in
  ambient mode. The split happens on the caller's side, so a caller outside the
  mesh hits the selectorless `Service` and gets no answer.
- Two components, `greeter-stable` and `greeter-canary`, each with an HTTP
  workload endpoint named `greeter-api`, deployed to the environment. Scenario
  3's components work with the names changed:
  [`service-go-variant-greeter`](../../service-go-variant-greeter) reports the
  variant in the `X-Variant` header, so you can count the split.

## Apply

```bash
kubectl apply -f platform/s5-mesh-http-endpoint-type.yaml
kubectl apply -f endpoint/greeter-endpoint.yaml
```

From a pod in the mesh, in the project's data-plane namespace or another
project's:

```bash
for i in $(seq 1 100); do
  curl -s -o /dev/null -D - http://greeter.<dp-namespace>.svc.cluster.local/greeter/greet \
    | grep -i '^x-variant'
done | sort | uniq -c
#   ~90 x-variant: stable
#   ~10 x-variant: canary
```

## Move the canary

Change the weights on the Endpoint. Keep them summing to 100, which is what
Istio expects:

```bash
kubectl patch endpoints.openchoreo.dev greeter -n default --type json -p '[
  {"op":"replace","path":"/spec/backendRefs/0/weight","value":50},
  {"op":"replace","path":"/spec/backendRefs/1/weight","value":50}]'
```

Each change makes a new EndpointRelease, which `autoDeploy` rolls out to the
first environment. Promote it to the others like any Endpoint release.

The binding behaves as in scenario 3:

- A canary at weight 0 can be listed before it is deployed. It is skipped, and
  the route has a single destination.
- A backend with weight that can't be resolved freezes the last rendered route
  instead of moving its share onto the other backend.

## Watch out for

- **Name collisions.** The Endpoint's name becomes a `Service` beside the
  components' Services, in the same namespace. An Endpoint named
  `greeter-stable` would collide with that component's `Service`. Nothing
  checks this today, so name the Endpoint after the API, not after one of its
  implementations.
- **Mesh only.** There is no gateway route. For traffic from outside, pair
  this with a gateway Endpoint, or have the type also render the
  `VirtualService` on an Istio ingress gateway.
- **Weights only.** Routing the canary by header (for example
  `x-canary: true`) needs a per-backend `match`, which the Endpoint doesn't
  have yet. A `VirtualService` would carry it directly as a `match` on an
  `http` route.
