# Module Summary: Deploy Sidecars to Azure App Service — Unit 4: Connect containers and share files

This unit focuses on inter-container communication over the shared network namespace and file exchange via the default shared `/home` volume. These mechanisms simplify tightly coupled AI components by removing the need for external service endpoints.

### 1. Connect through the shared network namespace

All containers in a sidecar-enabled app share one network namespace. A container reaches another container via `localhost:<port>`, not with a container name or service name. Each process must listen on a unique port.

Because the main API calls the model server locally (e.g., `http://localhost:11434`), you should use bounded HTTP clients with strict timeouts to prevent stalled local processes from holding API requests indefinitely.

**Code Example: Bounded HTTP Client for Local Requests**
```python
import asyncio
import httpx

MODEL_ENDPOINT = "http://localhost:11434/v1/chat/completions"

async def request_completion(messages: list[dict[str, str]]) -> dict:
    timeout = httpx.Timeout(10.0, connect=2.0)
    async with httpx.AsyncClient(timeout=timeout) as client:
        async with asyncio.timeout(30.0):
            response = await client.post(
                MODEL_ENDPOINT,
                json={"model": "support-model", "messages": messages},
            )
        response.raise_for_status()
        return response.json()
```

### 2. Design reliable local requests

Even though communication is local, treat the call as a fallible dependency. A practical request policy includes:
* A short connection timeout to detect unavailable sidecars.
* An application-level request deadline (latency target).
* Limited retries, restricted to operations that are safe to repeat (idempotent).
* Clear error mapping and correlation identifiers in logs.

### 3. Use the shared home volume

By default, App Service mounts the same `/home` volume into the main container and every sidecar. You can use directories like `/home/models` for file exchange without needing custom `volumeMounts` arrays.

To prevent race conditions (where a reader accesses a partially written file), writers should write to a temporary name and then rename/replace the file.

**Code Example: Safe File Exchange via `/home/models`**
```python
from pathlib import Path

SHARED_DIRECTORY = Path("/home/models")
SHARED_DIRECTORY.mkdir(parents=True, exist_ok=True)

temporary_manifest = SHARED_DIRECTORY / "manifest.json.tmp"
completed_manifest = SHARED_DIRECTORY / "manifest.json"
temporary_manifest.write_text('{"model": "support-model"}', encoding="utf-8")
temporary_manifest.replace(completed_manifest)
```

### 4. Avoid unnecessary custom mounts

The default shared `/home` volume is typically sufficient. Avoid adding explicit `volumeMounts` entries to site container definitions, as they can conflict with the runtime environment and prevent the app from starting.

### 5. Choose the correct storage boundary

The `/home` directory is excellent for files in the operational workflow:
* Temporary coordination files.
* Generated configurations.
* Local model metadata.

However, it is *not* a durable system of record. If data must survive App Service restarts scaling operations long-term, use Azure Blob Storage or Azure Files instead.

### 6. Validate communication and file access

Test integration using specific health operations rather than relying purely on external HTTP responses. Verify unique target ports, test local health routes, read/write artifacts across the shared path, and ensure temporary data recreates successfully after app restarts.