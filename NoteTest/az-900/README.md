# AZ-900 学习文档（按 Microsoft Learn 路径编排）

> 对应认证：Microsoft Azure Fundamentals（AZ-900）
> 结构完全按 Microsoft Learn 的三条学习路径组织，每条路径下的模块顺序与官方一致。
> 官方页面：https://learn.microsoft.com/en-us/training/paths/az-900-describe-cloud-concepts/

## 考试概览

| 项目 | 内容 |
| --- | --- |
| 时长 | 约 45 分钟 |
| 题量 | 40～50 题（单选、多选、拖拽、是非、热区） |
| 及格分 | 700 / 1000 |
| 费用 | 99 美元（部分地区有优惠） |
| 有效期 | 基础级认证不过期 |
| 形式 | 线上监考或考试中心 |

## 分值分布（官方 skills measured）

| 领域 | 占比 |
| --- | --- |
| 云概念（Cloud concepts） | 25～30% |
| Azure 架构与服务（Azure architecture and services） | 35～40% |
| Azure 管理与治理（Azure management and governance） | 30～35% |

## 三条学习路径

### 路径一：Describe cloud concepts（云概念）

| 模块 | 主题 | 笔记 |
| --- | --- | --- |
| 1 | 云计算的定义与模型 | [01-云概念](01-云概念.md#模块-1-云计算基础) |
| 2 | 云的优势（高可用、可伸缩、弹性、可靠性、可预测性、安全性、可管理性） | [01-云概念](01-云概念.md#模块-2-云的优势) |
| 3 | 云服务类型（IaaS / PaaS / SaaS）与责任共担 | [01-云概念](01-云概念.md#模块-3-云服务类型) |
| 4 | 云部署模型（公有云、私有云、混合云、多云） | [01-云概念](01-云概念.md#模块-4-云部署模型) |

### 路径二：Describe Azure architecture and services（架构与服务）

| 模块 | 主题 | 笔记 |
| --- | --- | --- |
| 1 | 核心架构组件（区域、可用性区域、资源组、订阅、管理组） | [02-架构与服务](02-架构与服务.md#模块-1-核心架构组件) |
| 2 | 计算与网络服务（VM、App Service、容器、AKS、Functions、VNet、NSG、VPN、ExpressRoute） | [02-架构与服务](02-架构与服务.md#模块-2-计算与网络) |
| 3 | 存储服务（Blob、Files、Disk、Queue、Table、冗余选项、迁移工具） | [02-架构与服务](02-架构与服务.md#模块-3-存储服务) |
| 4 | 身份、访问与安全（Entra ID、MFA、SSO、RBAC、Zero Trust、Defender、Key Vault） | [02-架构与服务](02-架构与服务.md#模块-4-身份访问与安全) |

### 路径三：Describe Azure management and governance（管理与治理）

| 模块 | 主题 | 笔记 |
| --- | --- | --- |
| 1 | 成本管理（定价因素、TCO 计算器、成本管理工具、标签） | [03-管理与治理](03-管理与治理.md#模块-1-成本管理) |
| 2 | 治理与合规（Azure Policy、蓝图、资源锁、信任中心） | [03-管理与治理](03-管理与治理.md#模块-2-治理与合规) |
| 3 | 部署与管理工具（Portal、CLI、PowerShell、ARM/Bicep、Cloud Shell、Arc、部署模式） | [03-管理与治理](03-管理与治理.md#模块-3-管理与部署工具) |
| 4 | 监控（Azure Monitor、Log Analytics、Application Insights、Service Health、Advisor） | [03-管理与治理](03-管理与治理.md#模块-4-监控工具) |

## 配套文件

- [04-术语速查表](04-术语速查表.md)：考前最后一天刷这个
- [05-模拟题](05-模拟题.md)：40 题，含答案解析

## 建议节奏（7 天冲刺）

| 天 | 内容 |
| --- | --- |
| 1 | 路径一全过一遍 + 术语表 A～C |
| 2 | 路径二模块 1、2（架构 + 计算网络） |
| 3 | 路径二模块 3、4（存储 + 安全） |
| 4 | 路径三全部 + 术语表收尾 |
| 5 | 模拟题第一遍，错题回查 |
| 6 | 官方 practice assessment（Microsoft Learn 上免费） |
| 7 | 术语表二刷 + 错题三刷 |

## 考试技巧

- 题干出现 "minimize cost" → 优先选 PaaS、Serverless、按用量计费。
- 出现 "minimize administrative effort" → 托管服务、自动化、策略。
- 出现 "compliance / regulatory" → Azure Policy、信任中心、合规性 offerings。
- 拖拽题大多是「服务 → 类别」匹配，先把明显错误的划掉。
- 是非题常考「SaaS 由客户管理操作系统」这类责任共担的细节——记住：越往 SaaS，客户管得越少。
