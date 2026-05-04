# k3s 搭建与升级经验沉淀 (HomeLab 实践)

## 1. 基础搭建 (Day 0)

### 1.1 系统准备
- **禁用 Swap**: Kubernetes 要求禁用 Swap 以保证性能预测性。
  ```bash
  sudo swapoff -a
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  ```
- **SSH 免密**: 使用 Ansible 管理前，务必确保控制端 (Mac) 与所有节点之间的 SSH 免密登录已调通。

### 1.2 Master 安装关键点
- **禁用默认 Ingress**: 为了实现纯粹的 GitOps 管理，建议在安装时禁用内置的 Traefik，改用 Argo CD 统一部署 (如 `ingress-nginx`)。
  - 参数: `--disable traefik`
- **Kubeconfig 修复**: k3s 默认生成的 `k3s.yaml` 中 server 地址为 `127.0.0.1`。拉取到本地后需手动或通过脚本修改为 Master 的真实局域网 IP。

### 1.3 Worker 节点加入
- **版本一致性 (核心)**: Worker 节点的 k3s 版本必须与 Master **严格一致**。如果 Master 是旧版本，Worker 默认安装最新版会导致 API 报错（如 `v1beta1.EndpointSlice` 找不到）。
- **加入参数**:
  - `K3S_URL`: `https://<MASTER_IP>:6443`
  - `K3S_TOKEN`: 从 Master 的 `/var/lib/rancher/k3s/server/node-token` 获取。

## 2. 集群平滑升级

### 2.1 升级原则
1. **先 Master 后 Worker**: 永远先升级控制平面。
2. **在位升级 (In-place)**: k3s 支持直接重新运行安装脚本进行升级。

### 2.2 升级命令
通过指定 `INSTALL_K3S_VERSION` 环境变量实现：
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.28.11+k3s2 sh -
```

### 2.3 坑点记录
- **服务类型识别**: 在 Worker 节点升级时，如果不显式指定 `K3S_URL` 和 `K3S_TOKEN`，安装脚本可能会默认将其作为 `server` 安装，导致创建冲突的 `k3s.service`。
- **进程重启**: 有时安装脚本运行完后 `kubectl` 显示的版本未更新，这通常是因为旧的 `k3s-agent` 进程未退出。手动执行 `sudo systemctl restart k3s-agent` 即可解决。

### 2.4 多网卡 IP 识别问题 (Vagrant/VM 常见)
- **现象**: 节点加入后，`INTERNAL-IP` 显示为 `10.0.2.15` (NAT 网络) 而非局域网 IP。这会导致集群内 Pod 跨节点通信失败。
- **解决方法**: 在启动参数中显式指定 IP。
  - 参数: `--node-ip=<LAN_IP> --node-external-ip=<LAN_IP>`
- **强制刷新**: 如果 IP 已经注册错误，需先 `kubectl delete node <NAME>`，然后在 Worker 上删除 `/var/lib/rancher/k3s/agent/node-config.yaml` 并重启服务。

## 3. GitOps 兼容性
- **Server-Side Apply**: 在较旧的 K8s 版本上，Argo CD 安装大型 CRD（如 ApplicationSet）可能会因为 Annotation 过大报错。
  - 解决方法: 使用 `kubectl apply --server-side` 强制同步。
- **现代版本优势**: 升级到 `v1.28+` 后，对现代云原生工具的 API 支持更好，减少了因 API 废弃导致的同步失败。
