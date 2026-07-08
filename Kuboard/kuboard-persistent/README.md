# Kuboard v3 持久化部署指南

## 📋 文档概述

本文档详细说明 `kuboard-v3.yaml` 中各个组件的作用、关系和数据流向，帮助您理解 Kuboard v3 在 Kubernetes 集群中的部署架构。

---

## 🏗️ 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                     Kuboard v3 系统架构                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────┐         ┌──────────────────┐           │
│  │ Kuboard v3 Pod   │◄────────│ ConfigMap        │           │
│  │ (Deployment)     │         │ (配置管理)       │           │
│  └────────┬─────────┘         └──────────────────┘           │
│           │                                                   │
│           ├──────────────────────────┬─────────────────────┐ │
│           │                          │                     │ │
│      ┌────▼────────────┐     ┌──────▼──────┐      ┌────────▼─┤
│      │ Kuboard etcd     │     │ PVC/PV      │      │ Agent    │
│      │ 集群 (3副本)     │     │ (数据持久化)│      │ 通信     │
│      │ - 配置元数据     │     │             │      │          │
│      │ - 权限信息       │     │ /data目录   │      │ UDP/TCP  │
│      │ - 应用策略       │     │ - 日志      │      │ 30081端口│
│      │ - 高可用状态     │     │ - 缓存      │      │          │
│      └─────────────────┘     │ - 备份      │      └──────────┤
│                               └─────────────┘                 │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┤
│  │ Kuboard Namespace: kuboard                               │
│  └──────────────────────────────────────────────────────────┤
│                                                               │
└─────────────────────────────────────────────────────────────┘

       │                                    ▲
       │ (管理多个集群)                     │ (Agent 上报)
       ▼                                    │
  ┌─────────────┐   ┌─────────────┐   ┌────────────┐
  │ 生产集群    │   │ 测试集群    │   │ 其他集群   │
  │ etcd        │   │ etcd        │   │ etcd       │
  └─────────────┘   └─────────────┘   └────────────┘
```

---

## 🔗 核心组件关系详解

### 1. **ConfigMap ↔ Kuboard v3 Pod**

#### ConfigMap 的角色
```yaml
kind: ConfigMap
metadata:
  name: kuboard-v3-config
  namespace: kuboard
```

**职责**：
- 集中管理 Kuboard v3 的所有配置参数
- 支持灵活切换认证方式（内置/GitLab/GitHub/LDAP）
- 通过 `envFrom` 注入到 Pod 环境变量

**关键配置项**：
| 配置项 | 说明 | 用途 |
|--------|------|------|
| `KUBOARD_ENDPOINT` | Kuboard 访问地址 | 提供给 Agent 的回调地址 |
| `KUBOARD_AGENT_KEY` | Agent 密钥（32位） | 保证 Agent ↔ Kuboard 通信安全 |
| `KUBOARD_AGENT_SERVER_TCP_PORT` | Agent TCP 端口 | 数据传输 |
| `KUBOARD_AGENT_SERVER_UDP_PORT` | Agent UDP 端口 | 心跳/控制 |
| `KUBOARD_SERVER_LOGRUS_LEVEL` | 日志级别 | 调试和监控 |

**工作流程**：
```
ConfigMap 更新
    │
    ▼
Pod 重启（挂载新配置）
    │
    ▼
Kuboard v3 启动时读取环境变量
    │
    ▼
应用新配置生效
```

**⚠️ 注意**：修改 `KUBOARD_AGENT_KEY` 后，需要重新导入所有 Agent，旧 Agent 无法连接。

---

### 2. **Kuboard v3 Pod ↔ Kuboard etcd 集群**

#### 为什么需要独立的 etcd 集群？

**Kuboard etcd 的核心职责**：
- 存储 **Kuboard 自身的配置、权限、策略数据**（非 K8s 资源）
- 支持 **多集群管理**（一个 Kuboard 管理多个 K8s 集群）
- 实现 **高可用和数据一致性**
- 与 K8s 原生 etcd **完全隔离**

**为什么不直接用 K8s etcd？**

| 维度 | K8s etcd | Kuboard etcd | 结论 |
|-----|---------|-------------|------|
| **数据类型** | K8s API 对象（Pod/Service/ConfigMap等） | Kuboard 配置、用户、权限 | 职责不同 |
| **写入频率** | 极高（K8s 不断调度） | 相对低频 | 性能隔离 |
| **故障独立性** | K8s etcd 故障 = 集群瘫痪 | 隔离故障 | 安全考虑 |
| **多集群支持** | 每个集群一个 | 所有集群共享一个 | 架构需求 |

**etcd 集群配置**：
```yaml
replicas: 3  # 容错模式：允许1个节点故障
storageClassName: openebs-rwx  # 分布式存储
storage: 5Gi  # 每个副本的存储空间
```

**集群节点间通信**：
```
kuboard-etcd-0 ←──(peer 2380)──→ kuboard-etcd-1
     ↑                                ↑
     └────────────┬─────────────────┘
                  │
              kuboard-etcd-2
```

**数据流向**：
```
Kuboard v3 Pod
    │
    ├─ 读权限信息 ─→ kuboard-etcd:2379
    │               (客户端端口)
    │
    ├─ 写应用配置 ─→ etcd leader
    │               (自动同步到其他副本)
    │
    └─ Watch 数据变化
        (实时监控配置更新)
```

---

### 3. **Kuboard v3 Pod ↔ PVC/PV（持久存储）**

#### PVC 的角色
```yaml
kind: PersistentVolumeClaim
metadata:
  name: kuboard-data-pvc
spec:
  storageClassName: openebs-hostpath
  storage: 10Gi
```

**/data 目录中存储的数据**：

| 数据类型 | 位置 | 说明 |
|---------|------|------|
| **应用日志** | `/data/logs/` | Kuboard 服务运行日志 |
| **缓存数据** | `/data/cache/` | 临时缓存、Session 数据 |
| **上传文件** | `/data/uploads/` | 证书、密钥、配置文件 |
| **备份数据** | `/data/backups/` | 定期备份的配置快照 |
| **临时文件** | `/data/tmp/` | 处理过程中的临时数据 |

**为什么需要独立的 PVC？**

```
etcd 集群存储 vs PVC 存储

etcd 的 /data
├─ 结构化数据（键值对）
├─ 必须高可用（3副本）
├─ StorageClass: openebs-rwx（多读多写）
└─ 5Gi（数据量小）

Kuboard PVC 的 /data
├─ 非结构化数据（日志、文件）
├─ 本地存储即可
├─ StorageClass: openebs-hostpath（单机存储）
└─ 10Gi（可包含大量日志）
```

**数据持久化流程**：
```
Kuboard 启动
    │
    ├─ 挂载 PVC 到 /data
    │
    ├─ 读取历史日志
    │
    └─ 初始化日志、缓存目录

运行过程中
    │
    ├─ 实时写入日志 → /data/logs/
    │
    ├─ 用户上传文件 → /data/uploads/
    │
    └─ 定期备份 → /data/backups/
```

**⚠️ Pod 重启时**：
- PVC 中的数据 **保留不丢失**
- etcd 中的数据 **由 3 个副本保障**
- 新 Pod 启动后恢复原状

---

### 4. **Kuboard v3 ↔ Agent 通信关系**

#### Agent 是什么？

**Agent（Kuboard Agent）** 是部署在 **被管理集群** 上的代理程序，充当 Kuboard 与 Kubernetes 集群之间的通信桥梁。

#### Agent 的核心功能

| 功能 | 说明 | 实现方式 |
|-----|------|--------|
| **集群接入** | 将分散的 K8s 集群纳入 Kuboard 统一管理 | Agent 向 Kuboard 注册 |
| **状态上报** | 实时收集并上报集群和工作负载状态 | 定期/事件驱动上报 |
| **命令执行** | 接收 Kuboard 的管理命令并在集群中执行 | 转发 kubectl API 调用 |
| **日志聚合** | 收集 Pod 日志并提供��� Kuboard UI 查询 | 通过 Agent 代理日志流 |
| **资源监控** | 监控集群资源使用情况和健康状态 | metrics 采集和上报 |
| **认证中转** | 处理用户认证和授权转发 | RBAC 策略执行 |
| **网络通道** | 为不同网络的集群建立连接隧道 | TCP/UDP 长连接 |

#### Agent 类型和部署场景

**Agent 有多种类型，根据部署位置和用途分类**：

| Agent 类型 | 部署位置 | 用途 | 网络要求 |
|----------|---------|------|--------|
| **Kubernetes Agent** | 被管理的 K8s 集群内（DaemonSet） | 标准 K8s 集群管理 | 能访问 Kuboard 服务 |
| **SSH Agent** | Linux 主机 | 管理非 K8s 的物理机/虚拟机 | SSH 连接 |
| **Cloud Agent** | 云平台（ECS/VM） | 管理云上的资源 | 云平台 API 访问 |
| **边界 Agent** | 集群边界网关 | 跨域网络的集群接入 | 代理所有通信 |

**常见部署场景**：

```
场景 1: 标准 K8s 集群
生产集群 A (Agent DaemonSet)
    ↓ (TCP/UDP 30081)
Kuboard v3
    ↑
测试集群 B (Agent DaemonSet)

场景 2: 混合环境
物理机群组 (SSH Agent)
    ↓
Kuboard v3
    ↑
公有云集群 (Cloud Agent)
    ↑
私有集群 (K8s Agent)

场景 3: 跨域网络
外网集群 (K8s Agent)
    ↓
边界 Agent (防火墙内)
    ↓
Kuboard v3 (内网)
```

#### Agent 通信架构

```
被管理集群-1          被管理集群-2          被管理集群-3
  (Agent Pod)         (Agent Pod)         (Agent Pod)
    或                  或                  或
  (Agent 主机)       (Agent 主机)       (Agent 主机)
    │                   │                   │
    │ 认证 (KUBOARD_AGENT_KEY)
    │ 端口 (KUBOARD_AGENT_SERVER_TCP_PORT/UDP_PORT)
    │ 目标 (KUBOARD_ENDPOINT)
    │                   │                   │
    └────────────────────┼────────────────────┘
                         │
                         ▼
                  Kuboard v3 Service
              (NodePort 30081/30082)
            ┌──────────────────────────┐
            │  监听 KUBOARD_AGENT_*_PORT
            │  验证 KUBOARD_AGENT_KEY
            │  处理 Agent 连接
            └──────────────────────────┘
                         │
                    ┌────┴────┐
                    │          │
                    ▼          ▼
            配置管理  命令下发  日志聚合
```

#### Agent 通信端口配置详解

| 端口 | 协议 | NodePort | 源使用 | 目的 |
|-----|------|---------|--------|------|
| 30080 | TCP | 30080 | 浏览器 | **Web UI** - Kuboard 管理界面 |
| 30081 | TCP | 30081 | Agent | **Agent TCP** - 数据传输、命令下发、日志流 |
| 30081 | UDP | 30082 | Agent | **Agent UDP** - 心跳、控制信息、快速通知 |

**Service 配置解析**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kuboard-v3
spec:
  type: NodePort  # 暴露到集群外，供 Agent 连接
  ports:
    - name: webui
      nodePort: 30080              # Web UI 访问端口
      port: 80
      targetPort: 80               # Pod 内 80 端口
    
    - name: agentservertcp
      nodePort: 30081              # Agent TCP 连接端口
      port: 30081
      targetPort: 30081            # Pod 内 30081 端口
    
    - name: agentserverudp
      nodePort: 30082              # Agent UDP 心跳端口
      port: 30081
      protocol: UDP
      targetPort: 30081            # Pod 内 30081 端口（UDP）
```

#### Agent 认证和连接流程

**Agent 连接时的认证机制**：

```
1. Agent 初始化连接
   │
   ├─ 配置 KUBOARD_ENDPOINT: http://kuboard.gongyongjun.com
   ├─ 配置 KUBOARD_AGENT_KEY: 32b7d6572c6255211b4eec9009e4a816
   ├─ 配置 KUBOARD_AGENT_SERVER_TCP_PORT: 30081
   └─ 配置 KUBOARD_AGENT_SERVER_UDP_PORT: 30081
   
         │
         ▼
2. Agent 发起连接请求
   │
   ├─ 解析 KUBOARD_ENDPOINT 获得 Kuboard 服务地址
   ├─ 使用 TCP (KUBOARD_AGENT_SERVER_TCP_PORT) 建立主连接
   ├─ 使用 UDP (KUBOARD_AGENT_SERVER_UDP_PORT) 建立心跳通道
   └─ 在认证帧中携带 KUBOARD_AGENT_KEY
   
         │
         ▼
3. Kuboard v3 验证
   │
   ├─ 检查 ConfigMap 中的 KUBOARD_AGENT_KEY
   ├─ 匹配 Agent 发送的密钥
   │
   ├─ 密钥正确 → 建立连接 ✅
   │   └─ 注册 Agent 身份
   │   └─ 分配集群 ID
   │   └─ 开始接收数据
   │
   └─ 密钥错误 → 拒绝连接 ❌
       └─ 返回认证失败错误
       └─ Agent 重试或停止
```

#### Agent 在 YAML 中的使用位置

**在 ConfigMap 中定义的 Agent 相关参数**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kuboard-v3-config
  namespace: kuboard
data:
  # ============ Agent 通信配置 ============
  
  # 【KUBOARD_ENDPOINT】
  # 用途：提供给 Agent 的 Kuboard 访问地址
  # Agent 根据此地址连接到 Kuboard v3 ���务
  # 格式：http://[kuboard-服务器地址]:[端口]
  # 位置：Agent 启动前需配置此值
  KUBOARD_ENDPOINT: 'http://kuboard.gongyongjun.com'
  
  # 【KUBOARD_AGENT_KEY】
  # 用途：Agent 与 Kuboard 通信的密钥，32位字符串
  # 安全性：必须修改为复杂的字符串，避免默认值
  # 影响范围：修改后所有已部署的 Agent 需要重新导入
  # 位置：Kuboard v3 Pod 启动时读取此密钥用于认证
  KUBOARD_AGENT_KEY: 32b7d6572c6255211b4eec9009e4a816
  
  # 【KUBOARD_AGENT_SERVER_TCP_PORT】
  # 用途：Agent 连接 Kuboard 的 TCP 端口（数据传输）
  # Agent 行为：使用此端口建立主连接通道
  # 端口范围：建议 30000-32767（K8s NodePort 范围）
  # 位置：Service 中定义 NodePort，Pod 监听 targetPort
  KUBOARD_AGENT_SERVER_TCP_PORT: '30081'
  
  # 【KUBOARD_AGENT_SERVER_UDP_PORT】
  # 用途：Agent 连接 Kuboard 的 UDP 端口（心跳/控制）
  # Agent 行为：定期发送心跳包确保连接活跃
  # 通常与 TCP 端口相同，但协议不同
  # 位置：Service 中定义 NodePort，Pod 监听 targetPort
  KUBOARD_AGENT_SERVER_UDP_PORT: '30081'
```

**在 Service 中的暴露**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kuboard-v3
  namespace: kuboard
spec:
  type: NodePort  # 必须是 NodePort 以供集群外 Agent 访问
  ports:
    # TCP 端口 - Agent 主连接
    - name: agentservertcp
      nodePort: 30081                    # 集群外访问的端口（KUBOARD_AGENT_SERVER_TCP_PORT）
      port: 30081                        # Service 内部端口
      protocol: TCP
      targetPort: 30081                  # Pod 内部监听的端口
    
    # UDP 端口 - Agent 心跳
    - name: agentserverudp
      nodePort: 30082                    # 集群外访问的端口
      port: 30081                        # Service 内部端口
      protocol: UDP                      # 关键：必须是 UDP
      targetPort: 30081                  # Pod 内部监听的端口
```

#### Agent 在被管理集群中的部署示例

**典型的 Agent 部署（在被管理集群中）**：

```yaml
# 被管理集群中的 Agent 配置示例
apiVersion: v1
kind: ConfigMap
metadata:
  name: kuboard-agent-config
  namespace: kuboard  # 通常部署在 kuboard 命名空间
data:
  # 指向 Kuboard v3 所在集群的地址
  KUBOARD_ENDPOINT: 'http://kuboard.gongyongjun.com:30081'  # 或使用 kuboard-v3 Service 的外部 IP
  
  # 必须与 Kuboard ConfigMap 中的密钥一致
  KUBOARD_AGENT_KEY: 32b7d6572c6255211b4eec9009e4a816
  
  # Agent 的集群标识
  KUBOARD_CLUSTER_NAME: "Production-Cluster-1"

---

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kuboard-agent
  namespace: kuboard
spec:
  selector:
    matchLabels:
      app: kuboard-agent
  template:
    metadata:
      labels:
        app: kuboard-agent
    spec:
      containers:
      - name: agent
        image: eipwork/kuboard-agent:v3
        env:
        - name: KUBOARD_ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: kuboard-agent-config
              key: KUBOARD_ENDPOINT
        - name: KUBOARD_AGENT_KEY
          valueFrom:
            configMapKeyRef:
              name: kuboard-agent-config
              key: KUBOARD_AGENT_KEY
        - name: KUBOARD_CLUSTER_NAME
          valueFrom:
            configMapKeyRef:
              name: kuboard-agent-config
              key: KUBOARD_CLUSTER_NAME
```

#### 故障排查：Agent 连接问题

| 故障现象 | 根本原因 | 检查方式 | 解决方案 |
|---------|--------|--------|--------|
| Agent 连接失败 | KUBOARD_AGENT_KEY 不匹配 | 对比两端的密钥值 | 确保密钥一致，重新部署 Agent |
| Agent 无法解析地址 | KUBOARD_ENDPOINT 配置错误 | 从 Agent Pod 中 ping KUBOARD_ENDPOINT | 修改 KUBOARD_ENDPOINT 指向正确的 Kuboard 地址 |
| 连接超时 | 网络隔离或防火墙阻止 | 检查 NodePort 30081/30082 是否开放 | 配置防火墙规则允许端口访问 |
| TCP 连接成功但 UDP 失败 | UDP 端口被阻止 | telnet/nc 测试 TCP，udp 工具测试 UDP | 放开 UDP 30082 端口或调整规则 |
| Kuboard UI 无法看到 Agent | Agent 认证成功但未注册 | 查看 Kuboard 日志中的 Agent 连接日志 | 检查 Agent 版本兼容性，查看详细日志 |

---

### 5. **etcd 集群内部的关系**

#### StatefulSet 配置
```yaml
kind: StatefulSet
metadata:
  name: kuboard-etcd
spec:
  replicas: 3
  serviceName: kuboard-etcd  # DNS 服务名称
  volumeClaimTemplates:      # 为每个副本创建独立 PVC
    - metadata:
        name: data
      spec:
        storage: 5Gi
```

#### 节点身份和通信
```
节点名称          DNS 地址                  监听地址
─────────────────────────────────────────────────────
kuboard-etcd-0 → kuboard-etcd-0.kuboard-etcd:2379
kuboard-etcd-1 → kuboard-etcd-1.kuboard-etcd:2379
kuboard-etcd-2 → kuboard-etcd-2.kuboard-etcd:2379

                        ↓

                  集群通信端口
        ─────────────────────────────
        客户端通信: 2379 (HTTP)
        节点间通信: 2380 (peer)
```

#### 初始化过程
```bash
# 启动时的关键参数
--initial-cluster-token kuboard-etcd-cluster-1  # 集群标识符
--initial-cluster-state new                     # 新集群模式
--data-dir /data/kuboard.etcd                   # 数据目录
```

**⚠️ 重启时修改**：
启动第二次及以后的重启，需要修改 `--initial-cluster-state` 为 `existing`：
```bash
--initial-cluster-state existing  # 现有集群模式
```

---

## 📊 数据流向总结

### 用户交互流程
```
1. 用户在浏览器访问 Kuboard UI
   (http://kuboard.gongyongjun.com:30080)
         │
         ▼
2. Kuboard Pod 处理请求
   - 认证授权
   - 读写配置
         │
         ├─ 读/写权限信息 ──→ etcd
         │                    (强一致性)
         │
         └─ 读/写日志、缓存 ──→ PVC
                               (本地存储)
         │
         ▼
3. 返回 UI 界面
```

### Agent 上报流程
```
1. Agent 检测集群状态
         │
         ▼
2. Agent 连接 Kuboard v3
   (使用 KUBOARD_AGENT_KEY 认证)
   TCP: kuboard-v3-service:30081
   UDP: kuboard-v3-service:30081 (心跳)
         │
         ▼
3. Kuboard 接收并处理
         │
         ├─ 存储集群元数据 ──→ etcd
         │
         ├─ 记录事件日志 ──→ PVC
         │
         └─ 更新 UI 显示
```

---

## 🔐 安全性考虑

| 安全需求 | 实现方式 | 配置位置 |
|---------|--------|--------|
| **Agent 认证** | 32位密钥 | ConfigMap: `KUBOARD_AGENT_KEY` |
| **数据一致性** | etcd 3节点集群 | StatefulSet: `replicas: 3` |
| **隔离性** | 独立命名空间 | Namespace: `kuboard` |
| **持久化** | PVC 和 etcd | volumeMounts |
| **网络暴露** | NodePort 服务 | Service: `type: NodePort` |

---

## 🚀 部署和维护

### 部署顺序
```
1. 创建 Namespace (kuboard)
2. 创建 ConfigMap (kuboard-v3-config)
3. 部署 etcd 集群 (StatefulSet - 等待就绪)
4. 创建 etcd Service
5. 创建 PVC
6. 部署 Kuboard v3 (Deployment)
7. 创建 Service (暴露 UI 和 Agent 端口)
```

### 故障排查

| 故障现象 | 可能原因 | 检查项 |
|---------|--------|--------|
| Kuboard UI 无法访问 | Pod 未就绪 | `kubectl get pods -n kuboard` |
| Agent 连接失败 | 密钥不匹配 | ConfigMap `KUBOARD_AGENT_KEY` 是否与 Agent 配置一致 |
| 数据丢失 | PVC 损坏 | `kubectl describe pvc kuboard-data-pvc -n kuboard` |
| etcd 集群不健康 | 节点故障 | `kubectl get pods -n kuboard -l app=kuboard-etcd` |

### 备份恢复
```bash
# 备份 etcd 数据
kubectl exec -n kuboard kuboard-etcd-0 -- \
  etcdctl --endpoints=localhost:2379 \
  snapshot save /data/kuboard-backup.db

# 恢复需要重建集群并恢复数据
# 具体步骤参考 etcd 官方文档
```

---

## 📝 配置示例

### 切换认证方式

#### 使用 GitLab 认证
```yaml
data:
  KUBOARD_LOGIN_TYPE: "gitlab"
  KUBOARD_ROOT_USER: "your-user-name-in-gitlab"
  GITLAB_BASE_URL: "http://gitlab.mycompany.com"
  GITLAB_APPLICATION_ID: "7c10882aa46810a0402d17c66103894ac5e43d6130b81c17f7f2d8ae182040b5"
  GITLAB_CLIENT_SECRET: "77c149bd3a4b6870bffa1a1afaf37cba28a1817f4cf518699065f5a8fe958889"
```

#### 使用 GitHub 认证
```yaml
data:
  KUBOARD_LOGIN_TYPE: "github"
  KUBOARD_ROOT_USER: "your-user-name-in-github"
  GITHUB_CLIENT_ID: "17577d45e4de7dad88e0"
  GITHUB_CLIENT_SECRET: "ff738553a8c7e9ad39569c8d02c1d85ec19115a7"
```

#### 使用 LDAP 认证
```yaml
data:
  KUBOARD_LOGIN_TYPE: "ldap"
  KUBOARD_ROOT_USER: "your-user-name-in-ldap"
  LDAP_HOST: "ldap-ip-address:389"
  LDAP_BIND_DN: "cn=admin,dc=example,dc=org"
  LDAP_BIND_PASSWORD: "admin"
  LDAP_BASE_DN: "dc=example,dc=org"
  LDAP_FILTER: "(objectClass=posixAccount)"
  LDAP_ID_ATTRIBUTE: "uid"
  LDAP_USER_NAME_ATTRIBUTE: "uid"
  LDAP_EMAIL_ATTRIBUTE: "mail"
  LDAP_DISPLAY_NAME_ATTRIBUTE: "cn"
  LDAP_GROUP_SEARCH_BASE_DN: "dc=example,dc=org"
  LDAP_GROUP_SEARCH_FILTER: "(objectClass=posixGroup)"
  LDAP_USER_MACHER_USER_ATTRIBUTE: "gidNumber"
  LDAP_USER_MACHER_GROUP_ATTRIBUTE: "gidNumber"
  LDAP_GROUP_NAME_ATTRIBUTE: "cn"
```

---

## 📚 文件结构说明

```
kuboard-v3.yaml 包含的 Kubernetes 对象
│
├─ Namespace: kuboard
│  └─ 所有其他资源的命名空间
│
├─ ConfigMap: kuboard-v3-config
│  └─ 集中管理所有环境变量和配置
│
├─ StatefulSet: kuboard-etcd
│  ├─ 3 个 etcd Pod
│  ├─ 每个 Pod 一个独立 PVC
│  └─ 集群自动编排和故障转移
│
├─ Service: kuboard-etcd
│  └─ etcd 集群的内部服务发现
│
├─ PersistentVolumeClaim: kuboard-data-pvc
│  └─ Kuboard v3 的数据存储
│
├─ Deployment: kuboard-v3
│  ├─ 1 个 Kuboard v3 Pod
│  ├─ 挂载 ConfigMap (环境变量)
│  └─ 挂载 PVC (持久化存储)
│
└─ Service: kuboard-v3
   ├─ NodePort 30080 → Web UI
   ├─ NodePort 30081 → Agent TCP
   └─ NodePort 30082 → Agent UDP
```

---

## 🔍 关键参数速查

| 参数 | 默认值 | 说明 | 修改影响 |
|-----|--------|------|---------|
| `KUBOARD_ENDPOINT` | http://kuboard.gongyongjun.com | Kuboard 访问地址 | Agent 连接地址 |
| `KUBOARD_AGENT_KEY` | 32b7d6572c6255211b4eec9009e4a816 | Agent 密钥 | 所有 Agent 需重新导入 |
| `KUBOARD_AGENT_SERVER_TCP_PORT` | 30081 | TCP 端口 | Agent 连接受影响 |
| `KUBOARD_AGENT_SERVER_UDP_PORT` | 30081 | UDP 端口 | Agent 心跳受影响 |
| `KUBOARD_SERVER_LOGRUS_LEVEL` | info | 日志级别 | 调试细度改变 |
| etcd replicas | 3 | etcd 副本数 | 故障转移能力 |
| etcd storage | 5Gi | etcd 存储大小 | 可存储的数据量 |
| PVC storage | 10Gi | 日志/缓存存储 | 可存储的日志量 |

---

## 📖 相关资源

- [Kuboard 官方文档](https://kuboard.cn/)
- [etcd 官方文档](https://etcd.io/)
- [Kubernetes StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)

---

## ⚡ 快速参考

### 查看 Kuboard 状态
```bash
# 查看所有资源
kubectl get all -n kuboard

# 查看 etcd 集群状态
kubectl get statefulset kuboard-etcd -n kuboard
kubectl logs -n kuboard kuboard-etcd-0

# 查看 Kuboard Pod 日志
kubectl logs -n kuboard deployment/kuboard-v3 -f

# 进入 Kuboard Pod 调试
kubectl exec -it -n kuboard deployment/kuboard-v3 -- /bin/sh

# 查看 Service
kubectl get svc -n kuboard
```

### 常见操作
```bash
# 修改 ConfigMap 并重启 Pod
kubectl edit configmap kuboard-v3-config -n kuboard
kubectl rollout restart deployment/kuboard-v3 -n kuboard

# 扩展 etcd 集群副本（谨慎操作）
kubectl scale statefulset kuboard-etcd --replicas=5 -n kuboard

# 查看 PVC 使用情况
kubectl describe pvc kuboard-data-pvc -n kuboard
```

---

**最后更新**：2026-07-08  
**维护者**：kubernetes 集群管理员
