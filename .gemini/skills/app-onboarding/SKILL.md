---
name: app-onboarding
description: Application onboarding guide for the private GitOps architecture. Use this skill when the user wants to deploy a new application (Python, Go, Node.js, etc.) to the k3s cluster using Gitea and Argo CD.
---

# Application Onboarding Skill

This skill provides a standardized workflow for bringing new applications into the `qbeehole` GitOps ecosystem.

## 1. Containerization Pattern
Every application must be containerized. Follow these conventions:
- **Port Mapping**: Map internal app ports (e.g., 8081, 8080) to 80 via the K8s Service.
- **Environment Variables**: Use `ENV` in Dockerfile for defaults, and `values.yaml` for environment-specific overrides.
- **Lightweight Images**: Use `-slim` or `-alpine` variants.

## 2. GitOps Registry Procedure
1. **Initialize Chart**: Create `apps/<app-name>` in the `qbeehole` repo.
2. **Standard Template**: Use the `templates/all.yaml` pattern for Deployment, Service, and Ingress.
3. **App-of-Apps Registration**: Add the new application to `clusters/home-lab/templates/apps.yaml`.

## 3. Network & Access Pattern
- **Local Git**: Push source code to `http://git.want.trade:30080`.
- **Public URL**: Define a domain under `*.want.trade`.
- **Cloudflare Mapping**: Ensure the hostname is pointed to the tunnel in the Cloudflare Dashboard.

## 4. Troubleshooting Checklist
- **Image Pulls**: Ensure the worker node proxy is configured if using external registries.
- **Sync Delays**: Use the `argocd-ops` skill (`kubectl patch` command) to force immediate synchronization.
- **NFS Mounts**: If the app requires persistence, ensure `nfs-common` is installed via Ansible and the NAS path is exported.
