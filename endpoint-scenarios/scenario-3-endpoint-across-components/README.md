# Scenario 3: An Endpoint across Components

In scenarios 1 and 2, every Endpoint is generated from one component's
descriptor and belongs to that component. Here an Endpoint is written by hand
and is a product of its own: one URL in front of several components, with the
traffic split between them by weight.

The example is a blue/green migration. `greeter-blue` serves all traffic,
`greeter-green` is staged beside it, and the weights move from one to the other
while clients keep calling the same URL. When the migration is done,
`greeter-blue` is deleted and the URL stays.

```
                         Endpoint greeter (s3-http)
client ──► <host>/ ──►   ├── blue   weight 100 ──► Service greeter-blue
                         └── green  weight   0 ──► Service greeter-green
```

## Files

```
platform/
├── s3-service-component-type.yaml   # ClusterComponentType s3-service
└── s3-http-endpoint-type.yaml       # ClusterEndpointType s3-http
components/
├── greeter-blue.yaml                # Component greeter-blue
└── greeter-green.yaml               # Component greeter-green
endpoint/
└── greeter-endpoint.yaml            # Endpoint greeter, fronting both
```

### `s3-service`

The same as scenario 1's `s1-service`: it renders the workload and its
`Service`, but no routes, and allows `s3-http` for the HTTP family.

`allowedEndpoints` only governs the Endpoints generated from a component's
descriptor. A hand-made Endpoint can name any Endpoint type and route to any
component in its project, whatever that component's type allows. What it
accepts is decided by the Endpoint type's validations.

### `s3-http`

Scenario 1's `s1-http` under a new name: one `HTTPRoute` per requested
visibility, splitting traffic across the backends by weight. It doesn't care
which components the backends belong to.

| Setting | Where | Default | Meaning |
|---|---|---|---|
| `pathPrefix` | `parameters` | `/` | Public path the endpoint is served under |
| `cors` | `parameters` | `false` | Allow browsers to call the endpoint cross-origin (external route) |
| `hostname` | `environmentConfigs` | generated | Serve the external route on your own domain in this environment |

A single rewrite filter serves every backend, so **all the components behind
one Endpoint must serve under the same base path**. The two greeters run the
same code, so they do.

### The components

`greeter-blue` and `greeter-green` both build
[`service-go-variant-greeter`](../../service-go-variant-greeter). It answers
`GET /greeter/greet` with the value of `VARIANT`, in the body and in the
`X-Variant` header, so every response shows which component served it.

Its descriptor asks for no Endpoints. The components can be reached from
outside only through the hand-made Endpoint.

### The Endpoint

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Endpoint
metadata:
  name: greeter
  namespace: default
spec:
  owner:
    projectName: default
  type:
    kind: ClusterEndpointType
    name: s3-http
  visibility: [external]
  autoDeploy: true
  backendRefs:
    - name: blue
      component: {name: greeter-blue, endpoint: greeter-api}
      weight: 100
    - name: green
      component: {name: greeter-green, endpoint: greeter-api}
      weight: 0
```

The Endpoint carries no `openchoreo.dev/source-component` label, so the
Endpoint controller makes the **project** its owner instead of either
component. Deleting a component never deletes it.

## Set up

```bash
kubectl apply \
  -f platform/s3-http-endpoint-type.yaml \
  -f platform/s3-service-component-type.yaml
```

## Try it

### 1. Deploy blue and front it

```bash
kubectl apply -f components/greeter-blue.yaml
```

Once the build is done and the component is deployed to `development`, set its
variant on the ReleaseBinding:

```bash
kubectl patch releasebinding greeter-blue-development -n default --type merge \
  -p '{"spec":{"workloadOverrides":{"container":{"env":[{"key":"VARIANT","value":"blue"}]}}}}'
```

Then create the Endpoint:

```bash
kubectl apply -f endpoint/greeter-endpoint.yaml
```

`greeter-green` doesn't exist yet, but its backend has weight 0, so the binding
skips it and renders the route with blue alone. This is how you stage a new
backend before it is deployed in every environment.

```bash
ERB=$(kubectl get endpointreleasebinding -n default -o jsonpath=\
'{.items[?(@.spec.owner.endpointName=="greeter")].metadata.name}')

# blue resolved; green reported with a reason, not blocking at weight 0
kubectl get endpointreleasebinding "$ERB" -n default \
  -o jsonpath='{range .status.backends[*]}{.name}{"\t"}{.weight}{"\t"}{.resolved}{"\t"}{.reason}{"\n"}{end}'

# The URL clients use from now on
kubectl get endpointreleasebinding "$ERB" -n default -o jsonpath='{.status.urls.external}'
```

A helper to count which component answers:

```bash
URL="http://<host>:<port>/greeter/greet"
count() { for i in $(seq 1 100); do curl -s -o /dev/null -D - "$URL" | grep -i '^x-variant'; done | sort | uniq -c; }
count    # 100 x-variant: blue
```

### 2. Deploy green

```bash
kubectl apply -f components/greeter-green.yaml
# after its build and deployment:
kubectl patch releasebinding greeter-green-development -n default --type merge \
  -p '{"spec":{"workloadOverrides":{"container":{"env":[{"key":"VARIANT","value":"green"}]}}}}'
```

The binding now reports green as resolved. It still gets no traffic.

### 3. Shift the traffic

Each change to the weights makes a new EndpointRelease, which `autoDeploy`
rolls out to the first environment:

```bash
weights() {
  kubectl patch endpoints.openchoreo.dev greeter -n default --type json -p "[
    {\"op\":\"replace\",\"path\":\"/spec/backendRefs/0/weight\",\"value\":$1},
    {\"op\":\"replace\",\"path\":\"/spec/backendRefs/1/weight\",\"value\":$2}]"
}

weights 90 10;  count    # ~90 blue, ~10 green
weights 50 50;  count    # ~50 / ~50
weights 0 100;  count    # 100 green
```

The URL is the same throughout. Only the route behind it changes.

### 4. Retire blue

Drain blue before deleting it. Blue is at weight 0 after the last step, so
remove its backend and then the component:

```bash
kubectl patch endpoints.openchoreo.dev greeter -n default --type json \
  -p '[{"op":"remove","path":"/spec/backendRefs/0"}]'
kubectl delete component greeter-blue -n default
count    # 100 green, same URL
```

The Endpoint survives because the project owns it.

## What happens when a backend goes missing

The binding resolves each backend in every environment and reports the result
on `status.backends`. What happens to the route depends on the backend's
weight:

| Backend | Effect on the route |
|---|---|
| Unresolved, weight 0 | Skipped. The route is rendered with the other backends. |
| Unresolved, weight > 0 | The last rendered route is left as it was. `BackendsResolved` turns `False` and the binding stops syncing until the backend comes back. |
| No backend resolves | The same, with `No backend is available in environment ...`. |

A route rendered without a missing backend would silently move its share of
traffic onto the others. A route rendered with it would send that share to an
error. So the binding keeps the last good route and reports the problem.

So deleting a component that still has weight leaves that component's share
pointing at a Service that no longer exists until the weight goes to 0. Always
drain first, as in step 4.

Reasons you may see on a backend:

| Reason | Meaning |
|---|---|
| `NotDeployed` | The component is missing, not deployed in this environment, or undeployed |
| `EndpointNotAvailable` | The component has no such workload endpoint, or no in-cluster address yet |
| `ReleaseLineNotFound` | The backend names a release line that doesn't exist |
| `UnsupportedProtocol` | The workload endpoint's protocol can't be carried by a route |
| `NamespaceMismatch` | The component's Service isn't in the project's data-plane namespace |

## Trade-offs

What this scenario adds:

- **An Endpoint as its own product.** The URL belongs to the project, not to a
  component, so the components behind it can be replaced.
- **Migrations without client changes.** Moving traffic between
  implementations is a change to weights, and the URL never moves.
- **Safe staging.** A backend at weight 0 can be listed before it exists in
  every environment.

What it doesn't handle:

- **One base path.** The components behind an `s3-http` Endpoint must serve
  under the same base path. Two implementations with different base paths (for
  example the Go and Python reading-list services, at `/api/v1/reading-list`
  and `/reading-list`) can't share one, because a single rewrite filter serves
  all backends. Per-backend rewrites would lift this.
- **Weights only.** Every backend gets a share of every request. Routing by
  path or header (`/orders` to one component and `/payments` to another, or a
  canary header) needs a per-backend `match`, which the Endpoint doesn't have
  yet.
- **No generator manages it.** Nothing rebuilds a hand-made Endpoint from a
  descriptor, so it is changed by hand or through GitOps. That is also why the
  weights are safe here: a build never resets them.
