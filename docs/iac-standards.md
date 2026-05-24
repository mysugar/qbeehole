# IaC (Infrastructure as Code) 实践规范

## 1. 核心原则
- **代码即真相 (Single Source of Truth)**: 所有的集群配置、系统参数、应用声明必须存在 Git 仓库中。严禁通过 `kubectl edit` 或手动 SSH 修改。
- **幂等性 (Idempotency)**: Ansible 脚本和 Argo CD 应用必须是幂等的。无论运行多少次，最终结果应保持一致。
- **环境无关性 (Environment Agnostic)**: 核心逻辑应与具体 IP/密钥解耦，通过 `group_vars` 或 `Secret` 进行注入。

## 2. 基础设施自动化 (Ansible)
- **编写规范**: 详见 [Ansible 自动化编写规范](ansible-standards.md)。
- **Day 0 覆盖范围**: 包括 OS 调优 (Swap)、容器运行时配置 (Proxy/Mirrors)、K8s 核心组件安装。
- **配置固化**:
  - 代理配置应写入 `/etc/systemd/system/k3s.service.env`。
  - 镜像加速端点应写入 `/etc/rancher/k3s/registries.yaml`。
- **一键引导**: 提供 `site.yml` 统一入口，支持新机器的快速扩容。

## 3. GitOps 生命周期 (Argo CD)
- **App of Apps 模式**: 使用一个 Root App 发现并管理所有子应用，保持集群结构的层级清晰。
- **变更即部署**: 任何对 `apps/` 或 `infrastructure/` 的 Git 推送应能自动触发集群更新。
- **强制调度**: 在混合网络环境下（如 Master 无法拉取镜像），利用 `nodeSelector` 或 `Taints/Tolerations` 确保关键组件运行在健康的节点上。

## 6. 应用环境变量管理规范
为了保证应用的可移植性与安全性，建议遵循以下环境变量注入模式：

### 6.1 分类注入模式 (Recommended)
- **普通变量 (Config)**: 直接在 `values.yaml` 中定义，并通过 Deployment 的 `env` 列表注入。适用于 `DB_HOST`, `DB_PORT`, `LOG_LEVEL` 等非敏感信息。
- **敏感变量 (Secrets)**:
  1. 在 Helm Chart 中创建 `templates/secret.yaml`。
  2. 使用 `stringData` 将 `values.yaml` 中的明文（或通过外部注入的变量）转换为 Secret。
  3. 在 Deployment 中通过 `valueFrom.secretKeyRef` 引用。适用于密码、API Token 等。

### 6.2 示例结构
**values.yaml**:
```yaml
db:
  host: "db-service"
  password: "secure-password" # 仅在私有库或通过 CI 注入时保留
```

**Deployment**:
```yaml
env:
  - name: DB_HOST
    value: {{ .Values.db.host }}
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Chart.Name }}-secret
        key: DB_PASSWORD
```

## 7. Kubernetes 秘钥管理技巧 (Secrets)
- **stringData vs data**: 
  - `data`: 要求提供 Base64 编码后的字符串。手动编码容易出错（如多出换行符）。
  - `stringData`: **推荐用法**。`kubectl` 会自动处理 Base64 编码过程。你可以直接写入明文字符串，集群会自动将其转换为 `data` 存储。
- **命令行快速更新**:
  使用 `kubectl patch` 配合 `stringData` 可以安全且快速地更新秘钥，无需先手动 `echo -n ... | base64`:
  ```bash
  kubectl patch secret <NAME> -n <NS> -p '{"stringData": {"key": "plain-text-secret"}}'
  ```

