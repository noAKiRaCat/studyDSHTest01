# OpenShift 学习文档

> 配套文件：`学习大纲.md`（路线图）、`常用命令.md`（速查表）。
> 本文档按大纲的六个阶段展开，每章包含：目标 → 核心概念 → 关键命令 → 动手实验 → 自测问题。
> 建议边看边在 CRC 或测试集群里敲，光读不写等于没学。

---

## 第 0 章 先把关系理清楚

### 目标

能一句话说清「OpenShift 和 Kubernetes 差在哪」，并且在看到 OCP 专有资源时不慌。

### 核心概念

**OpenShift 是 Kubernetes 的一个发行版**，用同一套 API 机制，但在上面叠了一层企业向的能力：更严格的安全模型、内置构建流水线、自带路由、监控日志栈、以及 Operator 生态。所以 `kubectl` 那套知识全部有效，`oc` 只是多了些子命令和资源类型。

四组最容易混淆的对应关系：

| Kubernetes | OpenShift | 说明 |
| --- | --- | --- |
| Ingress | Route | OCP 自带 HAProxy Router，Route 是它的原生对象；Ingress 也能用，但默认走 Route |
| PodSecurityPolicy / PSA | SCC | SCC 更严格，默认禁止 root、禁止特权、限制 capabilities |
| Deployment | Deployment / DeploymentConfig | 新项目一律用 Deployment，DC 是历史包袱 |
| 自己搭 CI + Registry | BuildConfig + ImageStream | 集群内置 S2I 构建和镜像流 |

另外两个只有 OCP 才有的概念：

- **Project**：带配额和权限边界的 Namespace，`oc new-project` 建的就是它。
- **Operator / OLM**：把「怎么运维一个中间件」写成代码，通过 OperatorHub 一键安装。

### 动手实验

1. 在 CRC 里 `oc api-resources | grep route`，看看 Route 属于哪个 API 组。
2. 建一个 Deployment，再 `oc expose` 它，观察自动生成的 Route 和 Service。
3. 故意用一个 root 镜像跑 Pod，读一遍 SCC 报错，理解它拦的是什么。

### 自测问题

- 为什么 OCP 默认不让容器以 root 运行？这对镜像构建有什么要求？
- Route 相比 Ingress 多了哪些能力（提示：分流权重、TLS 终止方式）？

---

## 第 1 章 把环境跑起来

### 目标

在一台机器上得到可用的集群，并能判断它是不是健康的。

### 核心概念

三条路线：

1. **CRC（OpenShift Local）**：笔记本上跑单节点，资源占用约 10 GB 内存、4 核，最快上手。
2. **SNO（Single Node OpenShift）**：一台物理机模拟生产形态，能验证安装和升级流程。
3. **正式集群**：IPI（安装器自动provision 基础设施）或 UPI（自己准备机器）。

集群健康的三件套，任何异常先看这三个：

```bash
oc get nodes                     # 节点是否 Ready
oc get clusteroperators          # 各组件是否 Available / Progressing / Degraded
oc get clusterversion            # 版本、升级状态、可用更新
```

`clusteroperators` 里出现 `Degraded=True` 基本就锁定问题域了：是网络、存储还是认证。

### 关键命令

```bash
crc setup
crc start
crc console --credentials        # 拿 kubeadmin 密码
eval $(crc oc-env)               # 让 oc 找到 CRC 的配置
oc login -u kubeadmin -p <pwd> https://api.crc.testing:6443
```

### 动手实验

1. 启动 CRC，登录 console，确认三个健康指标全绿。
2. 用 `oc get co` 找到 `dns` 这个 operator，`oc describe co dns` 看它的 conditions。
3. 关掉 CRC，再用 `crc start` 恢复，记录启动耗时。

### 自测问题

- `oc get nodes` 显示 NotReady，你会按什么顺序排查？
- IPI 和 UPI 的分界线是什么？

---

## 第 2 章 核心对象与日常操作

### 目标

能独立把一个「有状态 + 对外访问」的应用部署起来。

### 核心概念

**工作负载**：Pod 是最小单位，但不直接用；用 Deployment 管无状态、StatefulSet 管有状态、DaemonSet 管每节点一个、Job/CronJob 管一次性与定时任务。

**配置外置**：ConfigMap 存普通配置，Secret 存敏感信息（base64，不是加密，别当保险箱）。

**服务发现**：Service 给 Pod 一个稳定入口（ClusterIP/NodePort/LoadBalancer），Route 把 Service 暴露到集群外。

**存储**：PV 是实际存储，PVC 是申请单，StorageClass 决定动态供给用哪家。访问模式里 RWO（单节点读写）最常见，RWX（多节点读写）需要 NFS/CephFS 这类后端。

**配额**：ResourceQuota 限制项目总量，LimitRange 限制单个容器的默认值与上限。两者一起用，避免一个 Pod 吃光整个项目。

### 关键命令

```bash
oc new-app --image=postgres:16 --name=pg
oc create configmap app-config --from-file=config.yaml
oc create secret generic db-secret --from-literal=password='s3cret'
oc set env deployment/pg --from=secret/db-secret
oc expose service/pg
oc get pvc
```

### 动手实验

部署 PostgreSQL：

1. 建 PVC（1 Gi，RWO）。
2. 用 Deployment 挂载 PVC，注入 Secret 作为密码。
3. 建 Service（ClusterIP），再 `oc expose` 出 Route（数据库一般不暴露，这里只为验证链路）。
4. 用 `oc rsh` 进 Pod，`psql` 建一张表，重启 Pod 后确认数据还在。

### 自测问题

- PVC 一直 Pending，可能是什么原因？
- ConfigMap 改了，Pod 里的文件会自己更新吗？为什么？

---

## 第 3 章 镜像与构建

### 目标

能把自己写的代码变成集群里的镜像，并理解两种构建路径的取舍。

### 核心概念

**ImageStream** 是镜像的抽象层，指向一个或多个实际镜像，支持 tag 自动更新。好处是「构建」和「部署」解耦：Deployment 引用 `myapp:latest`，构建推新镜像后可以自动触发滚动。

**BuildConfig** 四种策略：

- **Source（S2I）**：给定 Git 仓库和基础镜像，平台自动检测语言、装依赖、构建。省事，但定制能力弱。
- **Docker**：用仓库里的 Dockerfile 构建。最通用，推荐新项目使用。
- **Pipeline**：Jenkins 流水线（老方案，逐步被 Tekton 取代）。
- **Custom**：自己写构建镜像，完全掌控。

### 关键命令

```bash
oc new-app https://github.com/user/repo.git --name=myapp
oc new-build --strategy=docker --name=myapp --binary
oc start-build myapp --from-dir=. --follow
oc logs -f bc/myapp
oc get istag
oc tag myapp:1.0 myapp:latest
```

### 动手实验

1. 用 S2I 构建一个 Node.js 小项目。
2. 同一个项目加 Dockerfile，改成 Docker 策略重建，比较镜像大小和构建时间。
3. 给 BuildConfig 加 webhook，改代码推 Git，观察是否自动触发构建。

### 自测问题

- ImageStream 的 `latest` 标签被覆盖后，正在运行的 Pod 会自动换镜像吗？
- S2I 构建失败，第一眼看哪个日志？

---

## 第 4 章 安全与权限

### 目标

能设计一个「最小权限」的项目结构，并解释 SCC 报错。

### 核心概念

**认证（你是谁）**：HTPasswd（简单）、LDAP（企业常用）、OIDC（对接 IdP）。kubeadmin 只用于初始化，生产要删掉。

**授权（你能干什么）**：Role/ClusterRole 定义权限集合，RoleBinding/ClusterRoleBinding 把权限授予用户、组或 ServiceAccount。内置角色：`view`、`edit`、`admin`、`cluster-admin`。

**SCC（你能怎么跑）**：控制容器的安全上下文——是否 root、是否特权、能否挂载宿主机路径、capabilities 白名单。默认的 `restricted` 会随机分配 UID，所以镜像里不能写死用户，文件权限要组可写。

**NetworkPolicy**：默认全通，按需收紧。生产中通常先建「默认拒绝」，再逐条放行。

### 关键命令

```bash
oc adm policy add-role-to-user view alice -n demo
oc adm policy add-scc-to-user anyuid -z default -n demo    # 慎用
oc auth can-i create pods -n demo
oc auth can-i --list -n demo
oc get scc
oc create sa builder-sa -n demo
```

### 动手实验

1. 建两个项目 A、B，各跑一个 nginx。
2. 从 A 的 Pod `curl` B 的 Service，确认能通。
3. 给 B 加默认拒绝的 NetworkPolicy，再只放行来自 A 的流量，验证结果。
4. 用一个 root 镜像部署，读报错，再用 `anyuid` 验证能跑起来（实验后立刻撤销）。

### 自测问题

- 为什么生产环境不应该给 `anyuid`？
- ServiceAccount 的 token 现在挂在哪，和旧版本有什么区别？

---

## 第 5 章 可观测性与运维

### 目标

出问题时能在 10 分钟内定位到「是应用、是配置、还是集群」。

### 核心概念

**监控**：集群内置 Prometheus + Alertmanager + Thanos，用户工作负载监控需要单独开启。Grafana 通过 `openshift-monitoring` 项目的 Route 访问。

**日志**：OpenShift Logging 基于 Loki（新）或 Elasticsearch（老），按 namespace 收集。

**事件**：`oc get events` 是最被低估的排障工具，调度失败、镜像拉取失败、探针失败都在这里。

**节点运维**：`cordon` 停止调度，`drain` 腾空（注意 DaemonSet 和 emptyDir 数据），`uncordon` 恢复。`oc debug node/<node>` 可以起一个带宿主机文件系统的调试 Pod。

**升级**：`oc adm upgrade` 查看可用版本，升级前确认 etcd 备份、Operator 兼容性、节点资源余量。

### 排障五步法

```bash
oc get pods                                   # 1. 状态
oc describe pod <pod>                         # 2. 事件与原因
oc logs <pod> --previous                      # 3. 上一次崩溃的日志
oc rsh <pod> / oc debug deployment/<name>     # 4. 进容器
oc get route,svc,endpoints                    # 5. 网络链路
```

### 动手实验

1. 部署一个会 CrashLoopBackOff 的应用（比如缺环境变量），用五步法定位。
2. 故意把镜像 tag 写成不存在，观察 events 里的报错措辞。
3. `cordon` 一个节点，看 Pod 调度行为，再 `uncordon`。

### 自测问题

- Pod 处于 `ImagePullBackOff`，可能的原因有哪些？
- 集群升级卡在某个 ClusterOperator，怎么判断能不能继续？

---

## 第 6 章 Operator 与 CI/CD

### 目标

能用 Operator 装中间件，并把 GitOps 流程跑通。

### 核心概念

**Operator** = CRD（自定义资源）+ 控制器（把运维知识写成代码）。比如装一个 PostgreSQL Operator，你只需要声明「我要一个 3 副本、20 Gi 的集群」，剩下的主从、备份、故障切换由 Operator 负责。

**OLM（Operator Lifecycle Manager）**：管理 Operator 的安装与升级，核心对象是 `Subscription`（订阅哪个版本通道）、`ClusterServiceVersion`（Operator 的元数据）、`OperatorGroup`（作用范围）。

**Tekton（OpenShift Pipelines）**：云原生 CI，用 Task/Pipeline 描述步骤，Pod 里跑。

**Argo CD**：GitOps，集群状态以 Git 仓库为准，自动同步或手动同步。

### 关键命令

```bash
oc get csv -A
oc get subscriptions -A
oc get installplan -A
argocd app create myapp --repo <url> --path . --dest-server https://kubernetes.default.svc --dest-namespace demo
argocd app sync myapp
```

### 动手实验

1. 在 OperatorHub 装一个中间件 Operator（Strimzi Kafka 或 Crunchy Postgres）。
2. 用它创建一个小实例，观察 Operator 自动生成哪些资源。
3. 用 Argo CD 接管一个 Deployment，改 Git 里的副本数，观察集群自动对齐。

### 自测问题

- Operator 的 `Subscription` 卡在 `UpgradePending`，怎么处理？
- GitOps 和传统 CI/CD 的核心区别是什么？

---

## 附：学习检查清单

- [ ] 能解释 OpenShift 与 Kubernetes 的三处关键差异
- [ ] 能从零起一个 CRC 并部署应用
- [ ] 能写 RoleBinding 让某个 SA 只读某个项目
- [ ] 能解释并处理 SCC 报错
- [ ] 能用五步法在 10 分钟内定位一个 CrashLoopBackOff
- [ ] 能说清一次集群升级的步骤与回滚思路
- [ ] 能用 Operator 装一个中间件并用 GitOps 管理应用

## 附：推荐资料

- OpenShift 官方文档：`docs.redhat.com` → OpenShift Container Platform
- OpenShift 博客与学习路径：`developers.redhat.com/learn/openshift`
- Kubernetes 官方文档（基础概念对照）
- `oc` 命令参考：`docs.redhat.com` 里的 CLI Reference
