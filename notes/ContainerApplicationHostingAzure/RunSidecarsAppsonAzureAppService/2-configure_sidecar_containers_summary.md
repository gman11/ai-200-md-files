# Module Summary: Deploy Sidecars to Azure App Service — Unit 3: Configure main and sidecar containers

This unit covers the practical steps and configuration syntax for defining, creating, and authenticating main and sidecar containers using the `sitecontainers` model and Azure CLI commands.

---

### 1. Create a sidecar-enabled app

You can enable sidecar support when creating a new Linux web app by passing the `--sitecontainers-app` option. This configures the app to use site container resources instead of classic custom container settings.

```bash
az webapp create \
  --name <app-name> \
  --resource-group <resource-group> \
  --plan <app-service-plan> \
  --sitecontainers-app
```

If you have an existing Linux custom-container app, you can convert its configuration to the site container model:

```bash
az webapp sitecontainers convert \
  --mode sitecontainers \
  --name <app-name> \
  --resource-group <resource-group>
```

---

### 2. Define container roles and ports

Every container needs a unique name and image reference. Exactly one definition needs `isMain: true`. The main process must listen on port `80` or `8080`.

**Create the main API definition:**
```bash
az webapp sitecontainers create \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name main-api \
  --image <registry-name>.azurecr.io/support-api:<tag> \
  --target-port 8080 \
  --is-main
```

**Create the model server as a sidecar:**
Omitting `--is-main` sets the property to false.
```bash
az webapp sitecontainers create \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server \
  --image <registry-name>.azurecr.io/model-server:<tag> \
  --target-port 11434
```

---

### 3. Separate shared and container-specific configuration

App settings are available to all containers, while container-specific environment variables map to one site container definition. You can create the source App Settings first:

```bash
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <resource-group> \
  --settings \
    MODEL_ENDPOINT_VALUE=http://localhost:11434 \
    MODEL_NAME_VALUE=support-model
```

For complex deployments, use a JSON specification array:

```json
[
  {
    "name": "main-api",
    "properties": {
      "image": "<registry-name>.azurecr.io/support-api:<tag>",
      "targetPort": "8080",
      "isMain": true,
      "environmentVariables": [
        {
          "name": "MODEL_ENDPOINT",
          "value": "MODEL_ENDPOINT_VALUE"
        }
      ]
    }
  },
  {
    "name": "model-server",
    "properties": {
      "image": "<registry-name>.azurecr.io/model-server:<tag>",
      "targetPort": "11434",
      "isMain": false,
      "environmentVariables": [
        {
          "name": "MODEL_NAME",
          "value": "MODEL_NAME_VALUE"
        }
      ]
    }
  }
]
```

Apply the specification using this command:
```bash
az webapp sitecontainers create \
  --name <app-name> \
  --resource-group <resource-group> \
  --sitecontainers-spec-file ./sitecontainers-spec.json
```

---

### 4. Authenticate private image pulls

To pull images securely from Azure Container Registry without hardcoding passwords, use managed identities. The container definition can specify `authType: UserAssigned`.

```json
{
  "name": "model-server",
  "properties": {
    "image": "<registry-name>.azurecr.io/model-server:<tag>",
    "targetPort": "11434",
    "isMain": false,
    "authType": "UserAssigned",
    "userManagedIdentityClientId": "<identity-client-id>"
  }
}
```

*Note: The managed identity needs proper role assignments (like `AcrPull`) and the registry must accept Azure Resource Manager audience tokens.*

---

### 5. Verify the deployed definitions

Validate your configurations before routing production traffic.

**List all container definitions for the app:**
```bash
az webapp sitecontainers list \
  --name <app-name> \
  --resource-group <resource-group> \
  --output table
```

**Inspect a specific definition in detail:**
```bash
az webapp sitecontainers show \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```