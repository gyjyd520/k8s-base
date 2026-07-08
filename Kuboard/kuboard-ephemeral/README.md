# 📦 Kuboard v3 临时部署指南

## 📋 快速概览

这是 **Kuboard v3 临时/测试部署** 的完整 Kubernetes 清单。适用于快速评估和测试场景，使用 `emptyDir` 存储（Pod 重启后数据丢失）。

如需持久化存储和高可用部署，参考 [`../kuboard-persistent/`](../kuboard-persistent/kuboard-v3.yaml)。

---

## 🎯 部署模式对比

| 特性 | 临时部署 | 持久化部署 |
|-----|--------|---------|
| **存储** | `emptyDir`（临时） | PVC + etcd 集群 |
| **数据持久性** | ❌ Pod 重启丢失 | ✅ 数据保留 |
| **高可用** | ❌ 单副本 | ✅ 3 副本 etcd |
| **适用场景** | 测试、演示、POC | 生产环境 |
| **配置复杂度** | 低 | 中等 |
| **部署时间** | ~2-3 分钟 | ~5-10 分钟 |
| **文件位置** | `kuboard-ephemeral/` | `kuboard-persistent/` |

---

## 📦 包含的 Kubernetes 资源

```
kuboard-v3.yaml
├── 1. Namespace: kuboard
├── 2. ServiceAccount: kuboard-bootstrap
├── 3. ClusterRoleBinding: kuboard-bootstrap-crb
├── 4. ConfigMap: kuboard-v3-config (★ 配置中心)
├── 5. Deployment: kuboard-v3 (★ 主服务)
├── 6. Service: kuboard-v3 (★ 网络暴露)
└── 7. PodDisruptionBudget: kuboard-v3-pdb
```

### 核心组件说明

#### 🔐 ServiceAccount & RBAC
- **kuboard-bootstrap**: 服务账户
- **ClusterRoleBinding**: 绑定 `cluster-admin` 角色，赋予 Kuboard 集群管理权限

#### ⚙️ ConfigMap（kuboard-v3-config）
集中管理所有配置参数：
```yaml
KUBOARD_ENDPOINT: "http://kuboard-v3:80"              # Agent 连接地址
KUBOARD_AGENT_SERVER_TCP_PORT: "10081"                # Agent TCP 端口
KUBOARD_AGENT_SERVER_UDP_PORT: "10081"                # Agent UDP 端口
KUBOARD_AGENT_KEY: "32b7d6572c6255211b4eec9009e4a816" # Agent 认证密钥
KUBOARD_SERVER_LOGRUS_LEVEL: "info"                   # 日志级别
```

#### 🚀 Deployment（kuboard-v3）
Kuboard 主服务容器：
- **镜像**: `eipwork/kuboard:v3.4.0`
- **副本**: 1
- **资源限制**: CPU 200m-1000m，内存 256Mi-1Gi

#### 🌐 Service（kuboard-v3）
暴露 Kuboard 的 3 个端口：
- **30080** (NodePort) → 80 (Web UI)
- **30081** (NodePort) → 10081 (Agent TCP)
- **30082** (NodePort) → 10081 (Agent UDP)

---

## 🚀 部署指南

### 前置条件

- ✅ Kubernetes 集群 1.19+
- ✅ 可访问的集群外网络（用于 Agent 连接）
- ✅ `kubectl` 命令行工具

### 快速部署

```bash
# 1. 应用清单
kubectl apply -f kuboard-v3.yaml

# 2. 验证部署
kubectl get all -n kuboard

# 3. 等待 Pod 就绪（通常 30-60 秒）
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=kuboard \
  -n kuboard --timeout=300s

# 4. 查看日志
kubectl logs -n kuboard deployment/kuboard-v3 -f
```

### 访问 Kuboard UI

#### 方式 1：使用 Port Forward（本地测试）
```bash
kubectl port-forward -n kuboard svc/kuboard-v3 8080:80

# 访问浏览器
# http://localhost:8080
```

#### 方式 2：使用 NodePort（集群内访问）
```bash
# 获取集群节点 IP
kubectl get nodes -o wide

# 访问浏览器（替换为实际节点 IP）
# http://<node-ip>:30080
```

#### 方式 3：使用 Ingress（推荐生产）
```yaml
# ingress-kuboard.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kuboard-ingress
  namespace: kuboard
spec:
  ingressClassName: nginx
  rules:
  - host: kuboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kuboard-v3
            port:
              number: 80
```

```bash
kubectl apply -f ingress-kuboard.yaml
# 访问 http://kuboard.example.com
```

---

## ⚙️ 配置说明

### 修改配置

所有配置集中在 `ConfigMap: kuboard-v3-config` 中：

```bash
# 编辑配置
kubectl edit configmap kuboard-v3-config -n kuboard

# 重启 Pod 应用新配置
kubectl rollout restart deployment/kuboard-v3 -n kuboard

# 验证更新
kubectl logs -n kuboard deployment/kuboard-v3 -f
```

### 常见配置修改

#### 1. 修改 Agent 认证密钥（⚠️ 重要）

```bash
# 生成新的 32 位密钥
openssl rand -hex 16

# 编辑 ConfigMap
kubectl edit configmap kuboard-v3-config -n kuboard

# 修改此行
KUBOARD_AGENT_KEY: "新生成的32位密钥"

# 重启
kubectl rollout restart deployment/kuboard-v3 -n kuboard

# ⚠️ 注意：所有已连接的 Agent 需要重新导入
```

#### 2. 修改 Kuboard 访问地址

```bash
# 例如，将地址改为域名
KUBOARD_ENDPOINT: "http://kuboard.example.com:30081"

# 或使用外部 IP
KUBOARD_ENDPOINT: "http://192.168.1.100:30081"
```

#### 3. 修改日志级别

```bash
# 可选值: debug, info, warn, error
KUBOARD_SERVER_LOGRUS_LEVEL: "debug"
```

---

## 📊 核心参数解析

### 环境变量详解

| 环境变量 | 默认值 | 说明 | 用途 |
|---------|--------|------|------|
| `KUBOARD_ENDPOINT` | `http://kuboard-v3:80` | Kuboard 访问地址 | Agent 连接地址，必须保证可达 |
| `KUBOARD_AGENT_SERVER_TCP_PORT` | `10081` | Agent TCP 连接端口 | Agent 与 Kuboard 进行数据传输 |
| `KUBOARD_AGENT_SERVER_UDP_PORT` | `10081` | Agent UDP 心跳端口 | Agent 心跳和快速通知 |
| `KUBOARD_AGENT_KEY` | `32b7d...816` | Agent 认证密钥 | 验证 Agent 身份，必须修改为强密码 |
| `KUBOARD_SERVER_LOGRUS_LEVEL` | `info` | 日志级别 | 调试和监控用 |

### 资源限制

```yaml
resources:
  requests:
    cpu: 200m      # 最少分配 200m CPU
    memory: 256Mi   # 最少分配 256MB 内存
  limits:
    cpu: 1000m     # 最多使用 1 核 CPU
    memory: 1Gi    # 最多使用 1GB 内存
```

**调整建议**：
- **开发/测试**: 保持默认值
- **生产环境**: 根据实际负载增加 limits（如 2 核、2GB）

### 端口配置

```yaml
ports:
- name: web
  containerPort: 80          # Pod 内监听端口
  protocol: TCP
- name: https
  containerPort: 443
  protocol: TCP
- name: agent-tcp
  containerPort: 10081       # Agent TCP 连接
  protocol: TCP
- name: agent-udp
  containerPort: 10081       # Agent UDP 心跳
  protocol: UDP
```

**Service NodePort 映射**：
| 服务 | 内部端口 | NodePort | 用途 |
|-----|---------|---------|------|
| Web UI | 80 | 30080 | 管理界面 |
| HTTPS | 443 | 30443 | 安全访问 |
| Agent TCP | 10081 | 30081 | 数据传输 |
| Agent UDP | 10081 | 30082 | 心跳通道 |

---

## 🔍 监控和调试

### 查看部署状态

```bash
# 查看所有资源
kubectl get all -n kuboard

# 查看 Pod 详细信息
kubectl describe pod -n kuboard -l app.kubernetes.io/name=kuboard

# 查看 Pod 事件
kubectl get events -n kuboard --sort-by='.lastTimestamp'
```

### 查看日志

```bash
# 实时查看日志
kubectl logs -n kuboard deployment/kuboard-v3 -f

# 查看最后 100 行
kubectl logs -n kuboard deployment/kuboard-v3 --tail=100

# 查看指定时间段的日志
kubectl logs -n kuboard deployment/kuboard-v3 --since=1h
```

### 进入容器调试

```bash
# 进入 Pod 容器
kubectl exec -it -n kuboard deployment/kuboard-v3 -- /bin/bash

# 或使用 sh
kubectl exec -it -n kuboard deployment/kuboard-v3 -- /bin/sh

# 查看容器内文件结构
kubectl exec -n kuboard deployment/kuboard-v3 -- ls -la /data/
```

### 健康检查

```bash
# 检查 Liveness Probe
kubectl get pod -n kuboard -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")]}'

# 手动测试 HTTP 端点
kubectl exec -n kuboard deployment/kuboard-v3 -- curl -s http://localhost:80/

# 查看探针执行结果
kubectl describe pod -n kuboard -l app.kubernetes.io/name=kuboard | grep -A 10 "Liveness"
```

---

## 🔐 安全性考虑

### RBAC 权限

当前部署使用 `cluster-admin` 角色，这在生产环境中**不推荐**：

```yaml
# ⚠️ 当前配置（过度权限）
roleRef:
  kind: ClusterRole
  name: cluster-admin
```

**改进方案**（生产环境）：
```yaml
# 创建受限的 ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kuboard-limited
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
```

### Agent 密钥安全

```bash
# ⚠️ 立即修改默认密钥
KUBOARD_AGENT_KEY: "32b7d6572c6255211b4eec9009e4a816"  # ❌ 默认值

# 生成强密钥
openssl rand -hex 16  # ✅ 使用此方式生成

# 存储在 Secret 中（推荐）
kubectl create secret generic kuboard-agent-key \
  --from-literal=key=$(openssl rand -hex 16) \
  -n kuboard
```

### 网络隔离

```bash
# 仅允许特定命名空间访问 Kuboard
kubectl label namespace kuboard restricted-access=true

# 创建 NetworkPolicy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuboard-network-policy
  namespace: kuboard
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: kuboard
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          restricted-access: "true"
EOF
```

---

## 📦 存储说明

### 临时存储（emptyDir）

```yaml
volumes:
- name: kuboard-data
  emptyDir:
    sizeLimit: 10Gi
```

**特点**：
- ✅ 部署快速简单
- ❌ Pod 重启后数据丢失
- ❌ Pod 驱逐时数据丢失
- ✅ 适合测试/演示

**存储内容**：
```
/data/
├── logs/           # 应用日志
├── cache/          # 临时缓存
├── uploads/        # 上传的文件
└── tmp/            # 临时文件
```

### 升级到持久化存储

参考 [`../kuboard-persistent/README.md`](../kuboard-persistent/README.md) 使用 PVC：

```yaml
volumes:
- name: kuboard-data
  persistentVolumeClaim:
    claimName: kuboard-data-pvc
```

---

## 🆘 故障排查

### 问题 1: Pod 无法启动

```bash
# 查看 Pod 事件
kubectl describe pod -n kuboard -l app.kubernetes.io/name=kuboard

# 查看日志
kubectl logs -n kuboard deployment/kuboard-v3

# 常见原因：
# - 镜像拉取失败 → 检查网络和镜像仓库
# - 资源不足 → 检查集群资源
# - 权限问题 → 检查 RBAC 配置
```

### 问题 2: 无法访问 Web UI

```bash
# 检查 Service
kubectl get svc -n kuboard

# 检查 NodePort 开放
kubectl get svc kuboard-v3 -n kuboard -o wide

# 测试 Pod 内连接
kubectl exec -it -n kuboard deployment/kuboard-v3 -- curl http://localhost:80

# 测试集群外连接
curl http://<node-ip>:30080
```

### 问题 3: Agent 无法连接

```bash
# 检查 Agent 密钥是否一致
kubectl get configmap kuboard-v3-config -n kuboard -o yaml | grep KUBOARD_AGENT_KEY

# 检查 KUBOARD_ENDPOINT 是否正确
kubectl get configmap kuboard-v3-config -n kuboard -o yaml | grep KUBOARD_ENDPOINT

# 检查端口是否开放
netstat -tuln | grep 30081
netstat -tuln | grep 30082

# 查看 Kuboard 日志中的连接信息
kubectl logs -n kuboard deployment/kuboard-v3 | grep -i agent
```

### 问题 4: 性能下降

```bash
# 检查 Pod 资源使用
kubectl top pod -n kuboard

# 检查 emptyDir 存储使用
kubectl exec -n kuboard deployment/kuboard-v3 -- du -sh /data

# 清理日志文件
kubectl exec -n kuboard deployment/kuboard-v3 -- rm -rf /data/logs/*
```

---

## 📚 常用命令快速参考

```bash
# 部署
kubectl apply -f kuboard-v3.yaml

# 查看状态
kubectl get all -n kuboard

# 查看日志
kubectl logs -n kuboard deployment/kuboard-v3 -f

# 编辑配置
kubectl edit configmap kuboard-v3-config -n kuboard

# 重启服务
kubectl rollout restart deployment/kuboard-v3 -n kuboard

# 监控状态
kubectl describe deployment kuboard-v3 -n kuboard

# 进入容器
kubectl exec -it -n kuboard deployment/kuboard-v3 -- /bin/bash

# 查看网络
kubectl get svc,ep -n kuboard

# 删除部署
kubectl delete -f kuboard-v3.yaml

# 查看 PDB
kubectl get pdb -n kuboard
```

---

## 🔄 升级和扩展

### 升级镜像版本

```bash
# 编辑 Deployment
kubectl set image deployment/kuboard-v3 \
  kuboard=eipwork/kuboard:v3.5.0 \
  -n kuboard

# 监视滚动更新
kubectl rollout status deployment/kuboard-v3 -n kuboard -w
```

### 扩展副本数（需要持久化存储）

```bash
# 只有使用 PVC 时才能安全地扩展多个副本
kubectl scale deployment kuboard-v3 --replicas=3 -n kuboard
```

### 添加环境变量

```bash
# 编辑 ConfigMap
kubectl patch configmap kuboard-v3-config -n kuboard \
  -p '{"data":{"NEW_VAR":"value"}}'

# 重启 Pod
kubectl rollout restart deployment/kuboard-v3 -n kuboard
```

---

## 🔗 相关资源

| 资源 | 说明 |
|-----|------|
| [Kuboard 官网](https://kuboard.cn/) | 官方文档和下载 |
| [Kuboard 持久化部署](../kuboard-persistent/README.md) | 生产环境部署指南 |
| [Kubernetes 官方文档](https://kubernetes.io/docs/) | K8s 官方参考 |
| [Docker Hub - Kuboard](https://hub.docker.com/r/eipwork/kuboard) | 镜像仓库 |

---

## 💡 最佳实践

### ✅ 推荐做法
- ✅ 定期备份配置（特别是 ConfigMap）
- ✅ 修改默认的 Agent 密钥
- ✅ 监控 Pod 日志和资源使用
- ✅ 使用 Ingress 提供域名访问
- ✅ 为生产环境考虑升级到持久化部署

### ❌ 避免事项
- ❌ 不要在生产环境使用 `cluster-admin` 权限
- ❌ 不要忽视数据备份（如需持久化）
- ❌ 不要直接编辑 Kubernetes 对象，使用 ConfigMap
- ❌ 不要在 Pod 内执行持久化操作（使用 emptyDir 时）
- ❌ 不要暴露默认的 Agent 密钥

---

## 📞 支持和问题反馈

如遇到问题：

1. **查看日志**：`kubectl logs -n kuboard deployment/kuboard-v3`
2. **检查事件**：`kubectl get events -n kuboard`
3. **查询文档**：[Kuboard 官方文档](https://kuboard.cn/)
4. **提交问题**：[GitHub Issues](https://github.com/gyjyd520/k8s-base/issues)

---

## 📝 变更日志

### v3.4.0 (2026-07-08)

- ✅ 修复 ServiceAccount 拼写错误（bootstrap）
- ✅ 添加 ConfigMap 配置管理
- ✅ 改进镜像版本指定
- ✅ 添加资源限制和安全上下文
- ✅ 优化健康检查参数
- ✅ 添加 PodDisruptionBudget
- ✅ 改进端口命名和标签体系

---

**最后更新**：2026-07-08  
**维护者**：[gyjyd520](https://github.com/gyjyd520)  
**许可证**：MIT

