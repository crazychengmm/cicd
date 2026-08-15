# 企业级发布系统 · 详细设计文档（KubeVela 选项B）

| 项目 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 编写日期 | 2026-08-09 |
| 配套文档 | 《详细设计文档》（选项A）、《选型与设计调研文档》2.4 节、《PoC 验证与验收方案》 |
| 本文档范围 | KubeVela 路线（选项B）的单分片拓扑、CUE 定义体系、发布编排联动、与自建发布单集成、资源估算 |
| 状态 | 备选方案，是否采用以 PoC 对比结果为准 |

---

## 目录

1. [定位与选项A的关系](#1-定位与选项a的关系)
2. [单分片拓扑](#2-单分片拓扑)
3. [CUE 定义体系设计](#3-cue-定义体系设计)
4. [应用与环境模型](#4-应用与环境模型)
5. [发布编排设计](#5-发布编排设计)
6. [多集群设计](#6-多集群设计)
7. [与自建发布单/审批流的集成](#7-与自建发布单审批流的集成)
8. [资源估算](#8-资源估算)
9. [与选项A的组件映射与切换路径](#9-与选项a的组件映射与切换路径)
10. [已知局限与缓解](#10-已知局限与缓解)

---

## 1. 定位与选项A的关系

选项B 的核心思想：**分片内的"应用抽象 + 渲染 + 多集群分发"交给 KubeVela，自研部分聚焦全局层（发布单/审批/审计/门户）与流程编排**。

```
┌─────────────────────────────────────────────────────────────┐
│  两条路线共用（不变）                                          │
│  · 全局层：Portal/API、元数据 DB、审批流、审计、租户权限           │
│  · 分片框架：按 BU 切分、Kafka 异步、分片 DB                    │
│  · 发布流程引擎：Argo Workflows（发布单状态机、批次、审批节点）     │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  选项A（自研执行层）          │  选项B（KubeVela 执行层）        │
│  · 自研应用模型 + 渲染层       │  · ComponentDefinition 体系   │
│  · ArgoCD 收敛 K8s 资源       │  · CUE 渲染引擎（vela-core）   │
│  · Karmada/Agent 多集群       │  · cluster-gateway 多集群     │
│  · Rollouts 发布策略          │  · Rollouts addon 发布策略     │
└─────────────────────────────────────────────────────────────┘
```

**决策可后置**：全局层与分片框架不受影响，选项 A/B 只替换分片内执行组件，PoC 对比后再定稿（见《PoC 验证与验收方案》第 6 节）。

---

## 2. 单分片拓扑

```
                    ┌──────────────────────────────────────────┐
                    │         分片控制平面（管理集群）                │
  全局层 ─Kafka─▶   │  ┌────────────┐    ┌───────────────────┐ │
  (topic: bu-x)     │  │ shard-api  │◀──▶│ Argo Workflows 实例 │ │
                    │  │ ×3 无状态   │    │ (发布编排)           │ │
                    │  └────────────┘    └────────┬──────────┘ │
                    │                             │ 更新 Application│
                    │                    ┌────────▼──────────┐ │
                    │                    │ vela-core ×2       │ │
                    │                    │ (控制器分片, CUE渲染) │ │
                    │                    └────────┬──────────┘ │
                    │                    ┌────────▼──────────┐ │
                    │                    │ cluster-gateway ×2 │ │
                    │                    │ (多集群 API 接入)    │ │
                    │                    └────────┬──────────┘ │
                    │  ┌────────────┐   ┌─────────▼─────────┐ │
                    │  │ ArgoCD ×1   │   │ Git 仓库组           │ │
                    │  │(同步App CR) │◀──│ git.xxx.com/bu-x/* │ │
                    │  └────────────┘   └───────────────────┘ │
                    │  ┌────────────┐   ┌───────────────────┐ │
                    │  │ MySQL 分区   │   │ VelaUX（全局1套）    │ │
                    │  └────────────┘   │ 定义管理/多环境视图   │ │
                    │                    └───────────────────┘ │
                    └──────────────────┬───────────────────────┘
                                       │ Hub→成员集群需网络可达
                  ┌────────────────────┼────────────────────┐
                  ▼                    ▼                    ▼
          ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
          │ 业务集群 A      │    │ 业务集群 B      │    │ 业务集群 C      │
          │ (免 agent)     │    │ (免 agent)     │    │ (免 agent)     │
          │ argo-rollouts │    │ argo-rollouts │    │ argo-rollouts │
          └──────────────┘    └──────────────┘    └──────────────┘
```

与选项A 的关键差异：**业务集群内部署 agent**，由 cluster-gateway 代理访问（成员集群无侵入，但要求 Hub→成员集群 apiserver 网络可达）。

---

## 3. CUE 定义体系设计

### 3.1 企业抽象目录

由平台团队统一维护，Git 管理 + VelaUX 发布：

| 类型 | 名称 | 说明 |
|------|------|------|
| ComponentDefinition | `enterprise-webservice` | 无状态 Web 服务 |
| ComponentDefinition | `enterprise-worker` | 后台任务（无入口流量） |
| ComponentDefinition | `enterprise-job` | 定时任务（CronJob） |
| ComponentDefinition | `enterprise-middleware` | 中间件接入（引用云资源） |
| TraitDefinition | `scaler` | 副本数/HPA |
| TraitDefinition | `gateway` | 入口域名与路由 |
| TraitDefinition | `logging` | 日志采集配置 |
| TraitDefinition | `monitoring` | ServiceMonitor 注入 |
| TraitDefinition | `canary` | 封装 Rollouts 金丝雀 |
| PolicyDefinition | `env-override` | 环境差异覆盖 |

### 3.2 ComponentDefinition 示例

```yaml
apiVersion: core.oam.dev/v1beta1
kind: ComponentDefinition
metadata:
  name: enterprise-webservice
  namespace: vela-system
spec:
  workload:
    definition: { apiVersion: apps/v1, kind: Deployment }
  schematic:
    cue:
      template: |
        output: {
          apiVersion: "apps/v1"
          kind:       "Deployment"
          metadata: name: context.name
          spec: {
            replicas: parameter.replicas
            selector: matchLabels: app: context.name
            template: {
              metadata: labels: app: context.name
              spec: containers: [{
                name:  context.name
                image: parameter.image
                ports: [{ containerPort: parameter.port }]
                readinessProbe: {
                  httpGet: { path: parameter.healthPath, port: parameter.port }
                  initialDelaySeconds: 10
                }
                if parameter.cpu != _|_ {
                  resources: {
                    requests: cpu: parameter.cpu
                    limits:   cpu: parameter.cpu
                  }
                }
              }]
            }
          }
        }
        parameter: {
          replicas:   *2 | int
          image:      string
          port:       *8080 | int
          healthPath: *"/healthz" | string
          cpu?:       string
        }
```

### 3.3 TraitDefinition 示例（封装 Rollouts 金丝雀）

```yaml
apiVersion: core.oam.dev/v1beta1
kind: TraitDefinition
metadata:
  name: canary
  namespace: vela-system
spec:
  appliesToWorkloads: ["enterprise-webservice"]
  schematic:
    cue:
      template: |
        // 将 Deployment 替换为 Rollout，并注入金丝雀步骤
        outputs: rollout: {
          apiVersion: "argoproj.io/v1alpha1"
          kind:       "Rollout"
          metadata: name: context.name
          spec: strategy: canary: steps: parameter.steps
        }
        patch: {
          // 原 Deployment 由 Rollout 接管
          spec: replicas: parameter.replicas
        }
        parameter: {
          replicas: *2 | int
          steps: [...]: {
            setWeight?: int
            pause?: { duration?: string }
          }
        }
```

### 3.4 定义治理

- **版本化**：定义随 Git 版本管理，变更走 PR 评审（CUE 属于平台层"代码"）；
- **校验**：CI 中用 defkit / `vela def vet` 校验定义合法性后才可合入；
- **灰度发布**：新定义先在 staging 分片验证，再推生产分片；
- **兼容约束**：parameter 字段只增不删，删除需走废弃流程（避免存量 Application 渲染失败）。

---

## 4. 应用与环境模型

### 4.1 应用基线模板（Git 中每应用一目录）

```yaml
# git: bu-x/shop-service/base.yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: shop-service
  namespace: bu-x
spec:
  components:
    - name: shop-service
      type: enterprise-webservice
      properties:
        image: registry.example.com/shop-service:BASE_TAG   # CI 更新此处
        port: 8080
      traits:
        - type: scaler
          properties: { replicas: 2 }
        - type: monitoring
```

### 4.2 环境差异（override policy）

```yaml
# 生产环境覆盖：副本、资源、多集群分发
spec:
  policies:
    - name: prod-override
      type: env-override
      properties:
        components:
          - name: shop-service
            traits:
              - type: scaler
                properties: { replicas: 20 }
    - name: target-prod
      type: topology
      properties:
        clusters: [prod-cn-1, prod-cn-2]
  workflow:
    steps:
      - name: deploy-prod
        type: deploy
        properties:
          policies: [prod-override, target-prod]
```

环境差异**不复制基线**，全部走 override，避免多环境清单漂移（与选项A 的设计原则一致）。

---

## 5. 发布编排设计

### 5.1 职责划分（关键设计）

| 层 | 承载者 | 内容 |
|----|--------|------|
| 流程层 | **自建**（Argo Workflows + 发布单） | 发布单状态机、审批、变更窗口、批次计划、审计 |
| 集群内编排 | **KubeVela workflow** | deploy/suspend/notification 步骤、多集群分发顺序 |
| 发布策略 | **Rollouts addon** | 流量级金丝雀、蓝绿、Analysis 门禁 |

> 原则：**KubeVela workflow 不直接面向开发者做审批入口**，审批仍由全局层发布单驱动；KubeVela workflow 的 suspend 步骤由自建流程通过 API 恢复。

### 5.2 分批发布时序

```
发布单创建（全局层）
   │
   ▼ Argo Workflows
① pre-check（变更窗口/配额）
② approval（suspend，人工审批）
③ 更新 Git：base.yaml 镜像 tag → 目标版本
④ ArgoCD 同步 Application CR → vela-core 渲染
⑤ vela-core workflow：deploy 批次1（topology 限定 10% 集群/副本）
⑥ Rollouts Analysis 门禁 → 通过后暂停
⑦ 批次确认（全局层驱动 vela workflow resume）
⑧ 批次2（50%）→ 门禁 → 批次3（100%）
⑨ 发布完成，回写发布单 + 审计
   （任一步失败 → vela workflow 回滚 + Rollouts undo）
```

### 5.3 自建流程驱动 KubeVela 的接口

| 操作 | 接口 |
|------|------|
| 更新镜像 | 改 Git（GitOps 路径）或调 vela API 更新 Application |
| 查询发布状态 | 读 Application status / workflow status |
| 恢复 suspend 步骤 | `vela workflow resume <app>` 或等价 API |
| 回滚 | `vela workflow rollback` + Application revision 回退 |

---

## 6. 多集群设计

### 6.1 集群纳管

- 成员集群通过 `vela cluster join <kubeconfig> --name prod-cn-1` 注册；
- 凭据存于 Hub 集群 Secret，支持证书轮换；
- 集群元数据（地域、用途、容量）同步进全局元数据 DB。

### 6.2 网络要求（关键评估点）

- cluster-gateway 要求 **Hub→成员集群 apiserver 网络可达**；
- PoC 必须输出《网络可达性矩阵》：每对（Hub, 成员集群）的连通方式（专线/打通端口/代理）；
- 若存在不可达集群，评估降级：该部分集群退回选项A 的 Agent 模式，形成混合部署。

### 6.3 gateway 容量

- 按官方数据，单 gateway（2 核/1 GiB）可支撑大规模下发；PoC 按分片 2 实例起步；
- 监控 gateway 请求延迟与负载不均（官方 40 万应用测试曾因负载不均减速）。

---

## 7. 与自建发布单/审批流的集成

| 集成点 | 方式 |
|--------|------|
| 发布单创建 | 全局层 API → Kafka → 分片 Argo Workflows |
| 触发发布 | 工作流更新 Git 镜像 tag（GitOps）|
| 审批节点 | Argo Workflows suspend；审批系统回调恢复 |
| 批次推进 | 工作流调用 vela workflow resume |
| 状态回传 | 工作流轮询/监听 Application status → 写分片 DB |
| 审计 | 每次 Application 变更记录 revision，关联发布单号 |

**VelaUX 定位**：仅作为平台团队管理 CUE 定义与查看多环境状态的辅助界面，不作为开发者发布入口（开发者入口是自建 Portal）。

---

## 8. 资源估算（单分片，2500 应用）

| 组件 | 规格 | 副本 | 小计 |
|------|------|------|------|
| vela-core（控制器分片） | 1 核 / 2 GiB | 2 | 2 核 / 4 GiB |
| cluster-gateway | 1 核 / 1 GiB | 2 | 2 核 / 2 GiB |
| ArgoCD（同步 Application CR） | 1 核 / 2 GiB | 1 | 1 核 / 2 GiB |
| Argo Workflows 实例 | 0.5 核 / 1 GiB | 1 | 0.5 核 / 1 GiB |
| VelaUX（全局共享 1 套） | 2 核 / 4 GiB | 1 | 摊薄 |
| **单分片合计** | | | **~5.5 核 / 9 GiB** |

> 对照官方压测：0.5 核/1 GiB 即可管 3,000 小应用，本估算保留充足余量应对更新/GC 负载。

---

## 9. 与选项A的组件映射与切换路径

| 职责 | 选项A | 选项B |
|------|-------|-------|
| 应用抽象 | 自研应用模型 + 模板 | ComponentDefinition/TraitDefinition（CUE） |
| 渲染 | 自研渲染层（Helm/Kustomize） | vela-core CUE 引擎 |
| 状态收敛 | ArgoCD（管 K8s 资源） | ArgoCD（仅管 Application CR）+ vela-core |
| 多集群 | Karmada / Agent | cluster-gateway（免 agent） |
| 发布策略 | Argo Rollouts | Argo Rollouts（addon） |
| 流程编排 | Argo Workflows | Argo Workflows（不变） |

**切换/混合路径**：
1. 两路线共享 Git 仓库结构与发布单模型，切换时只迁移分片内组件；
2. 支持混合：渲染/多集群用 KubeVela，个别特殊集群用 Agent；
3. 若 PoC 后由 B 转 A，CUE 定义可导出为 Helm/Kustomize 模板（一次性成本）。

---

## 10. 已知局限与缓解

| 局限 | 缓解 |
|------|------|
| CUE 技术栈学习成本 | defkit Go SDK + 定义模板库 + 平台团队集中维护（开发者不写 CUE） |
| Hub→成员集群网络可达要求 | PoC B6 一票否决项提前验证；不可达集群走混合 Agent 模式 |
| 单一项目依赖 | 版本锁定 + 私有镜像仓库快照 + 关注季度 Roadmap；全局层不依赖 KubeVela 可整体替换 |
| KubeVela workflow 与自建流程双层编排的复杂度 | 明确职责边界（5.1），KubeVela workflow 只做集群内语义，审批/批次计划归自建 |
| 大对象进 etcd 风险 | 发布历史/日志同样不进 CRD status，遵守全局防坑清单 |

---

*文档结束 · 本文档与《详细设计文档》（选项A）配套，最终采用路线以 PoC 对比结果为准。*
