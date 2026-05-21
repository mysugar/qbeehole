# Onboarding Process: GoAssistant (Python FastAPI)

## 1. Containerization (Dockerfile)
We need to transform the macOS startup script into a Docker image.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies (if any)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Environment variables from run_backend_mac.sh
ENV KATAGO_SERVER_URL="http://katago-service:8082"
ENV NO_PROXY="localhost,127.0.0.1"

# Entrypoint
EXPOSE 8081
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8081"]
```

## 2. Infrastructure Definition (GitOps)
We will define the Kubernetes resources in our `qbeehole` repo.

### 2.1 Helm Chart (`apps/go-assistant`)
- **Port**: Mapping container 8081 to Service 80.
- **Ingress**: Mapping `assistant.want.trade` to the Service.
- **Config**: Using `ConfigMap` or `Secret` for `.env` variables.

## 3. Deployment Flow (Standardized)
1. **Push Code**: Push Python source + Dockerfile to a new Gitea repo `mysugar2011/go-assistant`.
2. **Build Image**: Trigger Gitea Actions to build and push to local Registry.
3. **Update Ops**: Update `apps/go-assistant/values.yaml` with the new image tag.
4. **Auto-Sync**: Argo CD detects the change and deploys to k3s.
5. **Public Access**: Access via `https://assistant.want.trade` via Cloudflare Tunnel.
