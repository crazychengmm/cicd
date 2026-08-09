# 企业级发布系统 · 选型调研与架构设计

面向**万级应用规模**的企业级发布系统（CI/CD）设计资料，基于开源组件（Argo Workflows / Argo CD / Argo Rollouts / Karmada）构建。

## 文档索引

| 文档 | 内容 |
|------|------|
| [企业级发布系统选型与设计调研文档](./企业级发布系统选型与设计调研文档.md) | OAM 生态与 **KubeVela 深度分析**（架构/生态/CI-CD 集成/万级适配评估）、非 OAM 开源方案、企业级商业/信创方案调研；自研发布系统总体设计；万级规模分片架构；选型结论 |
| [企业级发布系统详细设计文档](./企业级发布系统详细设计文档.md) | 单分片部署拓扑、核心 CRD 设计、ArgoCD + Rollouts 联动的发布工作流模板、分片 DB 表结构、评审 PPT 大纲 |

## 核心架构一句话

> 全局薄入口（路由 / 审批 / 审计）+ 按 BU 分片的执行闭环（Argo Workflows 编排 + 分片 Argo CD 状态收敛 + 集群内 Argo Rollouts 发布策略）+ Kafka 异步化 + 一切大对象远离 etcd。

## 关键技术选型

- **编排引擎**：Argo Workflows（archive-to-DB + TTL GC）
- **状态收敛**：Argo CD 分片部署（控制器分片 + repo-server 缓存）
- **发布策略**：Argo Rollouts（金丝雀 / 蓝绿 + Analysis 门禁）
- **多集群**：Karmada / 集群内 Agent（Pull 模式）
- **分片原则**：按业务域切分，每片 2000–3000 应用，片内自闭环
