# 锡林郭勒盟教育系统 DevOps 平台设计文档

**日期**: 2026-05-16 | **状态**: 方案阶段

---

## 一、项目背景

为锡林郭勒盟教育系统的两个平台（课题管理平台、阅卷系统）搭建统一的 DevOps 体系，覆盖 CI/CD、镜像管理、GitOps 部署、多环境管理、可观测性。两个平台技术栈相似（Java + Spring Cloud + yudao-cloud），可以复用同一套 DevOps 工具链。

### 目标平台概况

| 平台 | 技术栈 | 数据库 | 微服务数 | 部署环境 |
|------|--------|--------|----------|----------|
| 课题管理平台 | Spring Cloud + yudao-cloud + Flowable | MySQL 8.0 | 5 个（project/review/fund/result/auth） | 私有云 |
| 阅卷系统 | Spring Cloud Alibaba + yudao-cloud | PostgreSQL | 6 个（考务/报名/阅卷/成绩/支撑/运维） | 公有云/私有云/政务云 |

---

## 二、DevOps 中控服务器

### 服务器配置
- **系统**: Ubuntu 22.04.5 LTS
- **CPU**: 16 核
- **内存**: 78 GB
- **磁盘**: 3.5 TB
- **角色**: DevOps 中控节点（不承载业务），通过 ArgoCD Hub-Spoke 模式远程管理各目标环境集群

### 最终技术栈选型

| 类别 | 选型 | 部署位置 |
|------|------|----------|
| 容器编排 | K3s | 中控 + 各目标环境 |
| 代码仓库 + CI | GitLab CE | 中控 K3s |
| CI 构建 | GitLab Runner + Kaniko | 中控 K3s Pod |
| 镜像仓库 | Harbor | 中控 K3s |
| GitOps CD | ArgoCD (Hub-Spoke) | 中控 K3s |
| 多集群管理 | Rancher | 中控 K3s |
| 指标监控 | Prometheus + Grafana | 中控 + 每集群 |
| 日志聚合 | Loki + Promtail | 中控 + 每集群 DaemonSet |
| 告警 | AlertManager | 中控 |
| 配置管理 | deploy-config Git 仓库 + Kustomize | GitLab |
| IaC | Terraform | 按需 |

### 组件资源分配

| 组件 | 内存最小 | 存储建议 |
|------|----------|----------|
| K3s 底座 | 0.5 GB | — |
| GitLab CE | 8 GB | 500 GB |
| Harbor | 2 GB | 200 GB |
| ArgoCD | 1 GB | 20 GB |
| Rancher | 2 GB | 50 GB |
| Prometheus + Grafana (中控) | 3 GB | 100 GB |
| Loki (中控) | 2 GB | 200 GB |
| MinIO (Loki 后端) | 1 GB | 500 GB |
| **合计** | **~19.5 GB** | **~1.5 TB** |

---

## 三、GitOps 配置仓库结构

```
deploy-config/
├── 课题管理平台/
│   ├── base/                  # 公共 K8s 配置
│   ├── dev/                   # 开发环境 overlay
│   ├── staging/               # 预发环境 overlay
│   └── prod/                  # 生产环境 overlay
├── 阅卷系统/
│   ├── base/
│   ├── dev/
│   ├── prod-public/           # 公有云 overlay
│   └── prod-gov/              # 政务云 overlay（达梦 DB/东方通/麒麟）
└── shared/                    # 公共中间件
    ├── mysql/
    ├── redis/
    ├── nacos/
    ├── rabbitmq/
    └── minio/
```

### 多环境差异化策略

- **Kustomize Overlay**: 每个环境只覆盖差异部分（DB 连接、存储类、节点亲和性），不复制整份配置
- **Spring Cloud ConfigMap + profile**: `application-gov.yml` 作为 ConfigMap 挂载，同一镜像通过环境变量选 profile

---

## 四、CI/CD 流水线

```
代码推送(GitLab) → GitLab CI 触发 → Kaniko 构建镜像 → Harbor 镜像仓库 → 更新 deploy-config → ArgoCD 同步 → K8s Apply
```

### GitLab CI 模板

```yaml
stages:
  - test
  - build
  - scan

variables:
  HARBOR_URL: harbor.your-domain.com
  KANIKO_IMAGE: gcr.io/kaniko-project/executor:debug

test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn clean test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

build:
  stage: build
  image:
    name: ${KANIKO_IMAGE}
    entrypoint: [""]
  script:
    - /kaniko/executor
        --context ${CI_PROJECT_DIR}
        --dockerfile ${CI_PROJECT_DIR}/Dockerfile
        --destination ${HARBOR_URL}/${CI_PROJECT_NAME}:${CI_COMMIT_SHORT_SHA}
        --cache=true

scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image ${HARBOR_URL}/${CI_PROJECT_NAME}:${CI_COMMIT_SHORT_SHA}
```

### 分支策略

| 分支 | 触发条件 | 目标环境 |
|------|----------|----------|
| `dev` | 自动 | 开发环境 |
| `main` | 手动审批后 | 生产环境 |

---

## 五、可观测性设计

### 指标监控
- 每个目标集群装 `kube-prometheus-stack`（轻量版），采集 Pod 资源、JVM 指标（Micrometer）、数据库指标
- 中控 Prometheus Federation 聚合所有集群关键指标
- Grafana 统一面板，多数据源切换

### 日志聚合
- Spring Boot 应用日志输出 stdout → K3s 自动收集
- Promtail DaemonSet 每节点采集，注入集群名/命名空间/Pod 名标签
- Loki 存储到 MinIO（500GB），保留 30 天

### 告警规则

| 级别 | 条件 | 通知 |
|------|------|------|
| 紧急 | Pod CrashLoopBackOff / 节点 NotReady / DB 连接池耗尽 | 飞书 + 邮件 |
| 警告 | CPU > 80% 5min / JVM GC 频繁 / 磁盘 > 85% | 飞书 |
| 通知 | 部署成功/失败/回滚 | 飞书 |

---

## 六、分阶段推进计划

| 阶段 | 内容 | 产出 |
|------|------|------|
| **Phase 1** | K3s 安装 + GitLab + Harbor + ArgoCD | 代码→镜像→部署最小链路 |
| **Phase 2** | 接入课题管理平台 CI/CD | 跑通第一个项目全流程 |
| **Phase 3** | 接入阅卷系统 + Rancher 多集群管理 | 验证多环境部署 |
| **Phase 4** | 监控日志全栈（Loki + Prometheus + 告警）| 补齐可观测性 |

---

## 七、Phase 1 详细安装步骤

> **前置条件**: Ubuntu 22.04 服务器，已完成基础配置（hostname、防火墙、时区）

### 7.1 系统初始化

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 设置 hostname
sudo hostnamectl set-hostname devops-controller

# 关闭 swap（K3s 要求）
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# 配置内核参数
cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k3s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# 安装必要工具
sudo apt install -y curl wget vim htop net-tools ca-certificates gnupg lsb-release
```

### 7.2 安装 K3s

```bash
# 安装 K3s（不使用 Traefik，后续我们用 Ingress-Nginx 或直接用 NodePort）
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -

# 验证安装
sudo kubectl get nodes
sudo k3s kubectl get pods -A

# 配置 kubectl（从 root 用户）
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# 测试
kubectl get nodes
```

### 7.3 安装 Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 验证
helm version
```

### 7.4 安装 GitLab CE

```bash
# 创建命名空间
kubectl create namespace gitlab

# 添加 GitLab Helm 仓库
helm repo add gitlab https://charts.gitlab.io/
helm repo update

# 创建 values.yaml
cat <<EOF > gitlab-values.yaml
global:
  hosts:
    domain: your-domain.com       # 替换为你的域名或 IP
    externalIP: "10.0.0.100"      # 替换为服务器 IP
  ingress:
    configureCertmanager: false
    class: "none"
  # 关闭不需要的组件以节省资源
  edition: ce
certmanager:
  install: false
nginx-ingress:
  enabled: false
prometheus:
  install: false
gitlab-runner:
  install: true
  runners:
    executor: kubernetes
    builds:
      cpuLimit: 2000m
      memoryLimit: 4Gi
    services:
      cpuLimit: 500m
      memoryLimit: 1Gi
registry:
  enabled: false           # 使用外部 Harbor，不启用内置 Registry
gitlab:
  webservice:
    minReplicas: 1
    maxReplicas: 1
    ingress:
      enabled: false
  gitaly:
    resources:
      requests:
        memory: 1Gi
        cpu: 500m
  sidekiq:
    resources:
      requests:
        memory: 1Gi
        cpu: 200m
postgresql:
  resources:
    requests:
      memory: 512Mi
      cpu: 200m
redis:
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
EOF

# 安装（使用 NodePort 方式暴露）
helm install gitlab gitlab/gitlab \
  -n gitlab \
  -f gitlab-values.yaml \
  --set global.hosts.https=false \
  --set gitlab.webservice.service.type=NodePort \
  --set gitlab.webservice.service.nodePort=30080 \
  --timeout 600s

# 获取初始 root 密码
kubectl get secret gitlab-gitlab-initial-root-password -n gitlab \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

**注意**: GitLab CE 初次启动需要 10-15 分钟，可通过 `kubectl get pods -n gitlab -w` 监控进度。

### 7.5 安装 Harbor

```bash
# 创建命名空间
kubectl create namespace harbor

# 添加 Harbor Helm 仓库
helm repo add harbor https://helm.goharbor.io
helm repo update

# 创建 values.yaml
cat <<EOF > harbor-values.yaml
expose:
  type: nodePort
  tls:
    enabled: false
  nodePort:
    name: harbor
    ports:
      http:
        port: 80
        nodePort: 30081
externalURL: http://your-server-ip:30081

# 持久化存储使用本地路径
persistence:
  enabled: true
  persistentVolumeClaim:
    registry:
      size: 200Gi
    chartmuseum:
      size: 20Gi
    jobservice:
      size: 10Gi
    database:
      size: 20Gi
    redis:
      size: 10Gi
    trivy:
      size: 10Gi

# 关闭不需要的组件
notary:
  enabled: false
trivy:
  enabled: true         # 镜像安全扫描

# 初始密码
harborAdminPassword: "Harbor12345"

# 资源限制
core:
  resources:
    requests:
      memory: 256Mi
      cpu: 200m
registry:
  resources:
    requests:
      memory: 256Mi
      cpu: 200m
EOF

# 安装
helm install harbor harbor/harbor -n harbor -f harbor-values.yaml --timeout 600s

# 获取状态
kubectl get pods -n harbor
```

**安装后配置**:
- 浏览器访问 `http://your-server-ip:30081`
- 用户名: `admin`，密码: `Harbor12345`
- 创建项目: `课题管理平台`、`阅卷系统`
- 创建机器人账号供 CI 推送镜像

### 7.6 安装 ArgoCD

```bash
# 创建命名空间
kubectl create namespace argocd

# 安装 ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 等待 Pod 就绪
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# 修改 service 为 NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8080, "nodePort": 30082, "name": "https"}]}}'

# 获取初始 admin 密码
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo

# 安装 ArgoCD CLI
sudo curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd

# 登录（替换为你的服务器 IP）
argocd login your-server-ip:30082 --username admin --password <上面获取的密码> --insecure

# 更新密码
argocd account update-password
```

### 7.7 配置 ArgoCD 连接 GitLab

```bash
# 在 GitLab 中创建 deploy-config 仓库后，添加为 ArgoCD 仓库
argocd repo add http://your-server-ip:30080/root/deploy-config.git \
  --username root \
  --password <gitlab-password>
```

### 7.8 验证整体连通性

```bash
# 确认所有组件 Pod 运行正常
kubectl get pods -A

# 端口映射清单
echo "
GitLab:    http://$(hostname -I | awk '{print $1}'):30080
Harbor:    http://$(hostname -I | awk '{print $1}'):30081
ArgoCD:    https://$(hostname -I | awk '{print $1}'):30082
"
```

### 7.9 后续步骤（Phase 2-4 简要）

**Phase 2** — 接入课题管理平台：
- 在各项目仓库添加 `.gitlab-ci.yml` 和 `Dockerfile`
- 在 `deploy-config` 仓库创建对应的 K8s 配置
- 在 ArgoCD 中创建 Application 指向 deploy-config

**Phase 3** — 阅卷系统 + Rancher：
```bash
# Rancher 安装（Phase 3 执行）
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm install rancher rancher-latest/rancher \
  -n cattle-system --create-namespace \
  --set hostname=rancher.your-domain.com \
  --set replicas=1
```

**Phase 4** — 监控日志全栈：
```bash
# kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Loki + Promtail
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n monitoring \
  --set promtail.enabled=true \
  --set grafana.enabled=false  # 使用已有的 Grafana
```

---

## 八、多实例架构：DevOps 中控 ↔ 远程目标集群

### 8.1 整体拓扑

以 10 个实例为例的推荐分配：

```
DevOps 中控服务器 (1台, 已有)
  └─ K3s + GitLab + Harbor + ArgoCD + Rancher + 监控

部署目标实例 (9台)
  ├─ 课题管理平台 dev (1台, 单节点 K3s)
  ├─ 课题管理平台 prod (3台, 1 server + 2 agent K3s 集群)
  ├─ 阅卷系统 dev (1台, 单节点 K3s)
  ├─ 阅卷系统 prod-public (2台, 阿里云 ACK 托管, 无需自建 K3s)
  └─ 阅卷系统 prod-gov (2台, 单节点 K3s, 政务云)
```

**原则**：
- **dev 环境**：单节点 K3s，省资源，挂了不心疼
- **prod 环境**：至少 2-3 节点形成 K3s 集群，保证高可用
- **公有云环境**：直接用云厂商托管 K8s（ACK），省运维
- **政务云**：单节点 K3s（政务云一般有自己运维体系，按需扩展）

### 8.2 远程实例初始化：安装 K3s

**每台目标实例都要执行**（以一台 dev 实例为例）：

```bash
# ===== 系统初始化（所有实例通用）=====
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname target-dev-ketizu   # 改成实际主机名
sudo swapoff -a && sudo sed -i '/swap/d' /etc/fstab

# 内核模块
cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k3s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# ===== 场景一：单节点 K3s（dev 环境 / 政务云单机）=====
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -

# 验证
sudo kubectl get nodes
```

**场景二：多节点 K3s 集群（prod 环境 3 台）**：

```bash
# === Server 节点（第1台）===
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -

# 获取 server token（agent 加入时需要）
sudo cat /var/lib/rancher/k3s/server/node-token

# 获取 server IP
hostname -I | awk '{print $1}'

# === Agent 节点（第2、3台）===
# 替换 SERVER_IP 和 TOKEN 为上面获取的值
curl -sfL https://get.k3s.io | \
  K3S_URL="https://SERVER_IP:6443" \
  K3S_TOKEN="TOKEN" \
  INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -

# 在 server 节点验证集群
sudo kubectl get nodes
# 应该看到 3 个节点，状态 Ready
```

### 8.3 中控服务器获取各集群的 kubeconfig

每个 K3s 集群的 kubeconfig 在 `/etc/rancher/k3s/k3s.yaml`。需要把 server 地址改为可路由的 IP（默认是 127.0.0.1）。

**在每个集群的 server 节点上执行**：

```bash
# 获取集群的 kubeconfig 并替换 127.0.0.1 为实际 IP
SERVER_IP=$(hostname -I | awk '{print $1}')
sudo sed "s/127.0.0.1/${SERVER_IP}/g" /etc/rancher/k3s/k3s.yaml > /tmp/kubeconfig-$(hostname).yaml

# 把文件传到 DevOps 中控服务器
scp /tmp/kubeconfig-$(hostname).yaml your-user@devops-server-ip:/tmp/
```

### 8.4 在 ArgoCD 中注册远程集群

**在 DevOps 中控服务器上执行**：

```bash
# 先登录 ArgoCD
argocd login localhost:30082 --username admin --password <your-password> --insecure

# ===== 方法一：argocd cluster add（推荐，自动配置）=====
# 先在中控服务器上放置 kubeconfig
mkdir -p ~/.kube/clusters

# 每个远程集群的 kubeconfig 放到不同文件
# 例如：~/.kube/clusters/ketizu-dev.yaml
#       ~/.kube/clusters/ketizu-prod.yaml
#       ~/.kube/clusters/yuejuan-dev.yaml
#       ~/.kube/clusters/yuejuan-prod.yaml

# 注册集群
export KUBECONFIG=~/.kube/clusters/ketizu-dev.yaml
argocd cluster add $(kubectl config current-context) --name ketizu-dev

export KUBECONFIG=~/.kube/clusters/ketizu-prod.yaml
argocd cluster add $(kubectl config current-context) --name ketizu-prod

export KUBECONFIG=~/.kube/clusters/yuejuan-dev.yaml
argocd cluster add $(kubectl config current-context) --name yuejuan-dev

# ===== 方法二：手动注册 =====
# 如果 argocd cluster add 不成功，手动方式：
argocd cluster add \
  --kubeconfig ~/.kube/clusters/ketizu-dev.yaml \
  --name ketizu-dev \
  --in-cluster=false

# 查看所有注册的集群
argocd cluster list
# 输出示例：
# SERVER                          NAME            STATUS
# https://kubernetes.default.svc  in-cluster      Successful  (中控自身)
# https://10.0.1.10:6443          ketizu-dev      Successful
# https://10.0.1.20:6443          ketizu-prod     Successful
# https://10.0.2.10:6443          yuejuan-dev     Successful
```

---

## 九、微服务部署配置实操

### 9.1 deploy-config 仓库完整结构

```
deploy-config/
├── ketizu-platform/                # 课题管理平台
│   ├── base/                       # 公共 K8s 资源（所有环境共享）
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── configmap.yaml          # 通用 ConfigMap
│   │   └── secret.yaml             # 通用 Secret（sealed-secrets）
│   ├── dev/
│   │   ├── kustomization.yaml      # 引用 base + 覆盖
│   │   ├── ingress.yaml
│   │   └── patch-db.yaml           # 覆盖数据库连接（dev MySQL）
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── ...
│   └── prod/
│       ├── kustomization.yaml
│       ├── ingress.yaml
│       └── patch-resources.yaml    # 覆盖资源配置（更高 limits）
│
├── yuejuan-system/                 # 阅卷系统
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── ...
│   ├── dev/
│   ├── prod-public/
│   │   └── patch-ack.yaml          # ACK 特定配置（存储类、LB）
│   └── prod-gov/
│       └── patch-gov.yaml          # 政务云特定（达梦 DB、东方通）
│
└── shared/                         # 公共中间件（MySQL/Redis/Nacos 等）
    ├── mysql/
    ├── redis/
    ├── nacos/
    └── rabbitmq/
```

### 9.2 kustomization.yaml 示例

**base/kustomization.yaml**（课题管理平台公共基础）：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ketizu-platform

resources:
  - namespace.yaml
  - configmap.yaml

images:
  - name: harbor.devops.local/ketizu-platform/project-service
    newTag: latest     # 会被 overlay 覆盖
  - name: harbor.devops.local/ketizu-platform/review-service
    newTag: latest
  - name: harbor.devops.local/ketizu-platform/fund-service
    newTag: latest
  - name: harbor.devops.local/ketizu-platform/result-service
    newTag: latest
```

**prod/kustomization.yaml**（生产环境覆盖）：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../base

namespace: ketizu-platform

# 覆盖生产环境镜像 tag
images:
  - name: harbor.devops.local/ketizu-platform/project-service
    newTag: "abc1234"              # CI 自动更新这个 tag
  - name: harbor.devops.local/ketizu-platform/review-service
    newTag: "abc1234"

# 覆盖资源配置
patchesStrategicMerge:
  - patch-resources.yaml
```

### 9.3 微服务 Deployment 示例

**project-service/deployment.yaml**：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-service
  namespace: ketizu-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: project-service
  template:
    metadata:
      labels:
        app: project-service
    spec:
      containers:
        - name: project-service
          image: harbor.devops.local/ketizu-platform/project-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"          # 通过环境变量选 profile
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: ketizu-config
                  key: db_host
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: project-service
  namespace: ketizu-platform
spec:
  selector:
    app: project-service
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

### 9.4 ArgoCD Application 配置

在 `deploy-config` 仓库创建 ArgoCD Application 定义：

**argocd-apps/ketizu-dev.yaml**：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ketizu-platform-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://gitlab.devops.local/root/deploy-config.git
    targetRevision: main
    path: ketizu-platform/dev          # 指向 dev overlay 目录
  destination:
    server: https://10.0.1.10:6443     # ketizu-dev 集群的 API server
    namespace: ketizu-platform
  syncPolicy:
    automated:
      prune: true                       # 自动删除不再定义的资源
      selfHeal: true                    # 自动修复漂移
    syncOptions:
      - CreateNamespace=true
```

**argocd-apps/ketizu-prod.yaml**：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ketizu-platform-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://gitlab.devops.local/root/deploy-config.git
    targetRevision: main
    path: ketizu-platform/prod
  destination:
    server: https://10.0.1.20:6443     # ketizu-prod 集群
    namespace: ketizu-platform
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 9.5 CI 自动更新 deploy-config（关键步骤）

GitLab CI 构建完镜像后，自动更新 deploy-config 仓库中的镜像 tag，触发 ArgoCD 同步。

```yaml
# .gitlab-ci.yml 中的 update-deploy-config stage
update-deploy-config:
  stage: deploy
  image: alpine/git:latest
  before_script:
    - git clone http://root:${GITLAB_TOKEN}@gitlab.devops.local/root/deploy-config.git
    - cd deploy-config
    - git config user.email "ci@devops.local"
    - git config user.name "GitLab CI"
  script:
    - cd deploy-config
    # 更新 dev 环境的镜像 tag
    - cd ketizu-platform/dev
    - kustomize edit set image \
        harbor.devops.local/ketizu-platform/${CI_PROJECT_NAME}:${CI_COMMIT_SHORT_SHA}
    - cd ../..
    # 提交并推送
    - git add .
    - git commit -m "ci: update ${CI_PROJECT_NAME} image to ${CI_COMMIT_SHORT_SHA}"
    - git push origin main
  only:
    - dev

# 生产环境需要手动触发或审批
update-deploy-config-prod:
  stage: deploy
  extends: update-deploy-config
  script:
    - cd deploy-config/ketizu-platform/prod
    - kustomize edit set image \
        harbor.devops.local/ketizu-platform/${CI_PROJECT_NAME}:${CI_COMMIT_SHORT_SHA}
    - cd ../..
    - git add .
    - git commit -m "ci(prod): update ${CI_PROJECT_NAME} to ${CI_COMMIT_SHORT_SHA}"
    - git push origin main
  only:
    - main
  when: manual              # 生产需要手动触发
```

---

## 十、前端项目部署配置

Vue 3 / uni-app 前端项目部署流程与后端微服务不同：构建产出一组静态文件，需要用 Nginx 托管。

### 10.1 前端 CI/CD 流程

```
代码推送 → GitLab CI → npm build → 生成 dist/ → Docker build (Nginx 基础镜像)
  → 推送 Harbor → 更新 deploy-config → ArgoCD 同步 → K3s 部署 Nginx Pod
```

### 10.2 前端 Dockerfile

```dockerfile
# 多阶段构建
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build        # 生成 dist/

# 运行阶段：Nginx 托管静态文件
FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 10.3 前端 nginx.conf

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Vue Router history 模式
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # API 反向代理到后端网关
    location /api/ {
        proxy_pass http://gateway-service:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 10.4 前端 Deployment 示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-frontend
  namespace: ketizu-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: admin-frontend
  template:
    metadata:
      labels:
        app: admin-frontend
    spec:
      containers:
        - name: nginx
          image: harbor.devops.local/ketizu-platform/admin-frontend:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: admin-frontend
  namespace: ketizu-platform
spec:
  selector:
    app: admin-frontend
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

### 10.5 移动端 H5（uni-app）特殊处理

阅卷系统的移动端（学生查分、教师阅卷）如果是 uni-app H5：

```javascript
// vite.config.js / vue.config.js
export default {
  base: '/h5/',              // 部署在子路径
  build: {
    outDir: 'dist-h5'
  }
}
```

```nginx
# nginx 配置增加子路径
location /h5/ {
    alias /usr/share/nginx/html/h5/;
    try_files $uri $uri/ /h5/index.html;
}
```

---

## 十一、完整端到端部署流程（以课题管理平台 project-service 为例）

### 流程总览

```
1. 开发者推送代码到 GitLab (dev 分支)
       │
2. GitLab CI 自动触发
       │
3. Maven 编译 + 测试
       │
4. Kaniko 构建 Docker 镜像
       │   → 推送到 Harbor: harbor.devops.local/ketizu-platform/project-service:abc1234
       │
5. GitLab CI 自动更新 deploy-config 仓库
       │   → 修改 ketizu-platform/dev/kustomization.yaml 中 newTag: "abc1234"
       │   → git push
       │
6. ArgoCD 检测到 deploy-config 变更
       │   → 对比当前集群状态 vs 期望状态
       │   → 自动同步到 ketizu-dev 集群
       │
7. K3s 拉取新镜像并滚动更新 Pod
       │
8. 健康检查通过，流量切换到新 Pod
       │
9. ArgoCD 面板显示 Synced + Healthy
```

### 验证命令

```bash
# 1. 查看代码是否推送
git log --oneline -5

# 2. 在 GitLab 面板查看 CI Pipeline 状态

# 3. 确认镜像已推送到 Harbor
curl -u admin:Harbor12345 \
  http://devops-server:30081/api/v2.0/projects/ketizu-platform/repositories/project-service/artifacts

# 4. 在 ArgoCD 面板查看同步状态
argocd app get ketizu-platform-dev

# 5. 在目标集群确认 Pod 已更新
export KUBECONFIG=~/.kube/clusters/ketizu-dev.yaml
kubectl get pods -n ketizu-platform
kubectl describe pod project-service-xxxxx -n ketizu-platform | grep Image

# 6. 查看应用日志
kubectl logs -f deployment/project-service -n ketizu-platform
```

---

## 十二、架构图

架构 SVG 文件位置: `项目管理/落地实施/devops-architecture.svg`

---

## 十三、关键技术决策记录

| 决策 | 选项 A | 选项 B | 选择 | 原因 |
|------|--------|--------|------|------|
| 容器编排 | 完整 K8s | K3s | K3s | 单机轻量，完全兼容 K8s API |
| CI/CD 方案 | GitLab 全家桶 | Gitea + Drone | GitLab CE | 集成度高，政府项目审计追溯需要 |
| CI 构建 | Docker | Kaniko | Kaniko | K3s 原生 containerd，无需额外 Docker daemon |
| GitOps 模式 | Hub-Spoke | 每集群独立 | Hub-Spoke | 集中管理，随项目增多优势放大 |
| 镜像构建方式 | Docker | Kaniko | Kaniko | K3s 使用 containerd，Kaniko 无需 Docker daemon，安全合规 |
| 日志方案 | ELK | Loki | Loki | 轻量、原生集成 Grafana，无需全文检索 |
