# Go Greeter API

The [Go greeter service](../service-go-greeter), built as an **API Service** component. Its
workload descriptor (`workload.yaml`, `openchoreo.dev/v1alpha2`) exposes the one HTTP endpoint
through two different Endpoint types, picked by name with `via`:

| Endpoint | Visibility | `via` | Endpoint type | What it adds |
|---|---|---|---|---|
| `external` | external | `managed` | `managed-http` | rate limit (30 requests/minute), CORS, optional API keys |
| `internal` | internal | `unmanaged` | `unmanaged-http` | a plain route |

`via` names an entry of the component type's `allowedEndpoints`, not an Endpoint type, so a
platform engineer can point `managed` at another gateway without this file changing. Each
build applies the descriptor: it creates, updates and deletes these Endpoints, which the
portal shows read-only.

## Deploy in OpenChoreo

Follow these steps to deploy the application in OpenChoreo:

### 1. Create a Component
- Set up OpenChoreo following the instructions at https://openchoreo.dev
- Open the Backstage UI and navigate to **Create**
- Select **Component Type: API Service**, with this directory (`service-go-greeter-api`) as the app path
  and Docker context
- After creation, navigate to the **Workflows** tab to view builds

### 2. Build and Deploy
- Once the build completes successfully, go to the **Deploy** tab
- Click **Deploy** to deploy the service to your environment

### 3. Test the Service
Navigate to the **Test** section in the left menu and use the OpenAPI Console to test the service endpoints.

## API Endpoints

### Greet

Send a personalized greeting.

**Endpoint:** `GET /greeter/greet`

**Query Parameters:**
- `name` (optional) - The name to greet. Defaults to "Stranger" if not provided.

**Example Request:**
```
GET /greeter/greet?name=Alice
```

**Response:**
```
Hello, Alice!
```

**Example Request (without name):**
```
GET /greeter/greet
```

**Response:**
```
Hello, Stranger!
```

### Version

Report the version of the running build. The startup log, the log line of each greeting or version request, and exported traces carry the same version.

**Endpoint:** `GET /greeter/version`

**Response:**
```
v1.0
```

The `release-lines-next` branch builds `v1.1`. It backs the [OpenChoreo release lines sample](https://github.com/openchoreo/openchoreo/tree/main/samples/release-lines), which runs both versions of this service side by side.

## Project Structure

```
service-go-greeter/
├── main.go              # Main service implementation
├── go.mod               # Go module definition
├── Dockerfile           # Container build configuration
├── openapi.yaml         # OpenAPI specification
├── workload.yaml        # OpenChoreo workload descriptor
└── README.md            # This file
```

## Local Development

### Prerequisites

- Go 1.x or later

### Run Locally

```bash
# Navigate to the service directory
cd service-go-greeter

# Run the service
go run main.go
```

The service will start on http://localhost:9090

### Test Locally

```bash
# Test with a name
curl "http://localhost:9090/greeter/greet?name=Alice"

# Test without a name
curl "http://localhost:9090/greeter/greet"

# Get the version
curl "http://localhost:9090/greeter/version"
```