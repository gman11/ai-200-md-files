# Module Summary: Deploy Sidecars to Azure App Service — Unit 2: Choose the Sidecar Pattern

This unit focuses on evaluating when to use the sidecar pattern, understanding its execution model, analyzing resource coupling, and contrasting it with independently hosted microservices.

### 1. The Sidecar Execution Model

* **Composition:** An app can consist of **one main container** (`isMain: true`) and **up to nine sidecar containers** (`isMain: false`).

* **Ingress Routing:** App Service routes external HTTP/HTTPS traffic **exclusively to the main container**. Sidecars are isolated from direct external internet traffic but are fully accessible to each other and the main app over `localhost` via a shared network namespace.

* **Lifecycle Co-Management:** Sidecars scale, start, stop, and deploy as a single cohesive unit. When an app scales out or in, every instance receives the complete set of main and sidecar containers.

### 2. When to Choose the Sidecar Pattern (Ideal Fit)

* Every instance of the main app requires exactly one co-located supporting process (e.g., a local model inference server).

* The supporting component shares the exact same lifecycle, scaling requirements, and availability needs as the main application.

* Local communication over `localhost` eliminates network latency and removes the need for complex inter-service authentication.

* The combined resource usage fits comfortably within a single App Service Plan.

### 3. When *Not* to Use Sidecars (Separate Service Better)

* Multiple apps need to share a single centralized resource (e.g., a shared model endpoint or cache).

* The supporting component requires distinct hardware (such as GPUs) or completely independent scaling policies.

* The workload requires high isolation or separate security boundaries.

* The supporting component must remain continuously available while the main app scales to zero or restarts.

### 4. Configuration Model (`sitecontainers`)

* Sidecar-enabled apps abandon older single-container configurations or Docker Compose formats in favor of the **`sitecontainers`** model (`LinuxFxVersion=sitecontainers`).

* Each container is defined as an individual `Microsoft.Web/sites/sitecontainers` resource specifying its role (`isMain`), image, environment variables, startup commands, and port mappings.

* **Code Example (Role Fragments):** The following fragments show the central distinction between the containers. Exactly one container should have `isMain` set to `true`.

```json
{
  "name": "main-api",
  "properties": {
    "isMain": true
  }
}
```

```json
{
  "name": "model-server",
  "properties": {
    "isMain": false
  }
}
```

### 5. Capacity Planning & Resource Constraints

* All containers on an instance draw from the same App Service plan.

* If a sidecar (like a local model runner) consumes high memory or CPU, it directly impacts the responsiveness of the main app sharing that node. Proper capacity testing under peak concurrent load is critical before production rollout.