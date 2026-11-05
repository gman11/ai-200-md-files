# Module Summary: Deploy Sidecars to Azure App Service — Unit 5: Diagnose sidecar failures

Because sidecars and the main application run together as one App Service application, a sidecar failure often appears as a generic main-app error. Troubleshooting requires breaking the problem down into distinct boundaries: deployment, image pull, process startup, resource pressure, and local communication.

### 1. Establish a symptom-first workflow

Instead of changing multiple settings at once, follow a systematic investigation sequence:
1. Confirm that the expected container definitions reached App Service.
2. Identify whether the container image was pulled successfully.
3. Confirm that the sidecar process starts and remains active.
4. Review CPU and memory metrics for shared resource pressure.
5. Test the `localhost` connection from the main app.
6. Validate the request and response at the sidecar application layer.

**List all site containers:**
Catch missing sidecars, unexpected image tags, or multiple main roles.
```bash
az webapp sitecontainers list \
  --name <app-name> \
  --resource-group <resource-group> \
  --output table
```

**Show complete definition for a single container:**
Compare its image, target port, identity settings, and startup commands against your release configuration.
```bash
az webapp sitecontainers show \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```

**Check runtime status:**
Verify if App Service reports the container as running.
```bash
az webapp sitecontainers status \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```

### 2. Diagnose image pull and startup failures

*   **Image Pull Failures:** Usually caused by incorrect image references, missing registry authorization, or network/firewall restrictions. Ensure managed identities have `AcrPull` or `Container Registry Repository Reader` roles.
*   **Startup Failures:** If the image pulls but crashes, check entry points, environment variables, base image compatibility, and ensure the target port matches where the process actually listens. 

**Retrieve logs for a specific site container:**
Use container-specific output to prevent unrelated main-app messages from obscuring the error.
```bash
az webapp sitecontainers log \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```

### 3. Diagnose resource pressure

Both main and sidecar containers share the underlying App Service plan resources on an instance. Loading heavy AI models into memory can cause restarts or slow responses. Compare CPU, memory, and HTTP queue metrics with sidecar logs. If measured demand exceeds the plan's capacity, you must either scale up the App Service plan, reduce concurrency, or host the model separately. 

### 4. Diagnose local connectivity

When the main app attempts to call the sidecar locally:
*   **Connection Refusal:** No process accepted the connection on the requested local port.
*   **Timeout:** Indicates slow startup, resource pressure, or a request exceeding processing budgets.
*   **HTTP Error Response:** The network connection succeeded, meaning the issue lies with request formatting, authorization, or model-server logic.

**Code Example: Handling local failures gracefully**
This Python fragment demonstrates how to distinguish local network failures in application code rather than returning internal crashes to a user.

```python
import httpx

async def check_model_health() -> str:
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            response = await client.get("http://localhost:11434/health")
            response.raise_for_status()
            return "ready"
    except httpx.ConnectError:
        return "connect-error"
    except httpx.TimeoutException:
        return "timeout"
    except httpx.HTTPStatusError as error:
        return f"http-{error.response.status_code}"
```

### 5. Diagnose shared-file failures

If the containers fail to exchange files, verify the following:
*   `WEBSITES_ENABLE_APP_SERVICE_STORAGE` must not be explicitly set to `false`.
*   Both applications must target the exact same path (e.g., `/home/models/manifest.json`).
*   Avoid adding explicit `volumeMounts` in site container definitions—rely on the default shared `/home` path.
*   *Note:* The `/home` volume supports file exchange but should not be used as a durable long-term database (use Azure Storage instead).

### 6. Use platform diagnostics

Correlate container-level logs with platform context. Use the App Service **Diagnose and solve problems** tool or Azure Monitor / Log Analytics to view complete incident records (including timestamps, exit codes, CPU/memory spikes, and client error categories).