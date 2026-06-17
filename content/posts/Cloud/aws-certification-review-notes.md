---
title: "AWS 认证复习笔记：服务边界、架构取舍与易错点"
date: 2024-08-20T20:00:00+08:00
draft: false
categories: ["Cloud"]
tags: ["AWS", "Certification", "Cloud Architecture"]
---

这篇是 2024 年准备 AWS Certification 时留下的复习笔记整理版。它不是 AWS 服务手册，也不追求覆盖每个参数；更有价值的是把考试里反复出现的判断方式整理出来：服务边界是什么、架构取舍是什么、哪些地方容易被相似服务误导。

## 复习方法

AWS 认证题很少只问“这个服务叫什么”，更多是在一个具体场景里让你选“最合适”的组合。我的复习方式是把服务按能力域分组，然后给每组建立三类记忆：

- 服务边界：这个服务负责什么，不负责什么。
- 架构取舍：低成本、高可用、低延迟、少运维、跨账号、跨区域分别会把答案推向哪里。
- 易错点：名字相近、场景相近但答案不同的服务。

这种方式比单纯背服务列表更稳定。比如看到“跨账号统一管控”，第一反应不应该是 IAM，而应该继续判断它是在做权限边界、组织策略、账号基线、审计，还是资源共享。

## Identity 与 Security

IAM 仍然是底层权限模型的核心，但考试里经常会把 IAM 和 Organizations、STS、Cognito、KMS、Secrets Manager 混在一起考。

IAM policy 主要分 identity-based policy 和 resource-based policy。前者绑在用户、组、角色上，后者绑在资源上。Permission boundary 不是直接授权，而是限制 identity-based policy 最多能授到哪里。SCP 也不是给用户授权，而是在 AWS Organizations 层面限制账号能做什么。排查权限问题时要记住：最终权限是多层策略共同作用的结果，显式 deny 优先。

STS 解决临时凭证和角色切换。跨账号访问时，trust policy 决定谁能 assume role，permission policy 决定 assume 之后能做什么。SAML、SSO、外部 IDP 的题目，本质上也是在问身份源和 AWS 临时身份如何接起来。

KMS 和 Secrets Manager 也容易混。KMS 管密钥和加解密能力，Secrets Manager 管 secret 生命周期，比如存储、读取、轮转数据库密码。CloudTrail、Config、Security Hub、Detective、GuardDuty、Inspector、Macie、WAF、Shield 是另一组常见组合：CloudTrail 记录 API 调用，Config 做资源配置与合规状态，Security Hub 聚合安全发现，GuardDuty 做威胁检测，Inspector 偏漏洞扫描，Macie 偏 S3 敏感数据识别。

## Compute 与 Application

EC2 的题目通常关注实例族、placement group、Auto Scaling、AMI、EBS 和网络边界。Graviton 往往指向性价比，但前提是 workload 兼容 ARM。Compute Optimizer 用来给资源规格建议，不是迁移工具。

Lambda 题目常见三个点：并发、冷启动和边缘运行。Reserved Concurrency 是限制/预留某函数并发上限；Provisioned Concurrency 是为了减少冷启动，需要持续付费；Lambda@Edge 是和 CloudFront 边缘逻辑相关。SnapStart 则是 Java 冷启动优化方向。

Step Functions 要区分 Standard 和 Express。Standard 更适合长流程、可审计、精确一次语义的工作流；Express 更适合高吞吐、短时长、事件驱动的工作流。

容器部分重点是 ECS、Fargate、EKS 的边界。ECS 是 AWS 原生容器编排，Fargate 是免管服务器的运行方式，EKS 是托管 Kubernetes。EKS Anywhere 和 EKS Distro 这类名字容易让人误判，它们通常和混合环境、离线环境或自管集群有关。题目如果强调“最少运维”，Fargate、App Runner、Elastic Beanstalk、Lambda、SAM/Serverless 往往比自管 EC2 更像答案。

## Storage 与 Data

S3 是对象存储，围绕生命周期、版本、复制、加密、访问控制、静态网站、事件通知、Glacier 分层出题。EBS 是单 AZ 块存储，EFS 是多 AZ NFS 文件系统，FSx 是托管文件系统族，具体再分 Windows File Server、Lustre、NetApp ONTAP、OpenZFS 等场景。

DynamoDB 的核心是 partition key、sort key、WCU/RCU、on-demand capacity、DAX、Global Tables 和 Streams。DAX 解决读缓存，不是通用 Redis 替代；Global Tables 解决多区域低延迟和容灾；Streams 常用于变更捕获。

关系数据库里，RDS 是托管关系数据库，Aurora 是 AWS 自研兼容 MySQL/PostgreSQL 的高可用引擎。RDS Proxy 用来管理数据库连接池，尤其适合 Lambda 这类突发连接场景。Aurora 通过添加 reader instance 获得 Multi-AZ 读取能力和更好的可用性，不要把它简单等同于传统 RDS 的 standby。

数据流和消息队列里，Kinesis Data Streams 更偏实时流处理，Firehose 更偏把数据投递到 S3、Redshift、OpenSearch 等目的地。SQS 是队列，SNS 是发布订阅，Amazon MQ 是托管 ActiveMQ/RabbitMQ，适合迁移现有协议兼容需求。

## Networking

VPC 题目主要看连接方式。Gateway Endpoint 主要服务 S3 和 DynamoDB；Interface Endpoint 基于 PrivateLink，适合私有访问 AWS 服务或自建服务。PrivateLink 是私有服务暴露和消费的方式，不需要打通整个 VPC 网络。

VPC Peering 是两个 VPC 之间的私有连接，但不可传递路由。Transit Gateway 适合多 VPC、多账号、多连接的集中式网络。Direct Connect 是专线，Site-to-Site VPN 是公网 IPsec 隧道，Client VPN 是终端用户接入。混合网络题目要看诉求是带宽稳定、延迟稳定、快速上线、成本低，还是高可用双链路。

Security Group 是有状态、实例/ENI 级别；NACL 是无状态、子网级别。NAT Gateway 用于私有子网访问公网，Internet Gateway 用于 VPC 与公网互通。EC2 做 NAT instance 时要关闭 source/destination check，这是非常典型的易错点。

API Gateway 要区分 REST API、HTTP API、WebSocket API。REST API 功能更完整，HTTP API 更轻量便宜。边缘优化 API Gateway 和 CloudFront 缓存相关，区域型 API 更适合自己接 CloudFront 或区域内访问。Route 53 题目常围绕记录类型、健康检查、路由策略和私有 hosted zone。

## Organization、Deployment 与 Operations

Organizations、SCP、Control Tower、RAM、Service Catalog 是治理类题目的核心。Organizations 管账号结构，SCP 设组织级边界，Control Tower 建多账号 landing zone，RAM 做跨账号资源共享，Service Catalog 用来发布合规的自助服务模板。

部署类服务要看应用形态。CloudFormation 是基础设施模板，StackSets 用于跨账号跨区域部署。CodeDeploy 做部署编排，CodeArtifact 管包，CodePipeline 串流水线。Elastic Beanstalk 是应用平台，SAM 是 serverless 模板和本地开发工具，Amplify 更偏前端和全栈应用托管。

Operations 常见组合是 CloudWatch、EventBridge、Systems Manager、Config 和 X-Ray。CloudWatch 负责指标、日志、告警、synthetics；EventBridge 做事件路由；Systems Manager 做实例管理、Session Manager、Parameter Store、Run Command；X-Ray 做分布式追踪。Config 更偏资源配置历史和合规规则，不是实时监控系统。

## Migration、DR 与 Cost

迁移题目先判断迁移对象。DMS 迁数据库，Application Migration Service 迁服务器，Migration Hub 做迁移跟踪，Application Discovery Service 做现状发现，Migration Evaluator 做迁移评估。VMware 场景里，先做 discovery、评估依赖和容量，再选择迁移路径。

灾备题目绕不开四个层级：

- Backup & Restore：成本最低，RTO/RPO 最差。
- Pilot Light：核心组件保持最小运行，灾难时扩容。
- Warm Standby：完整环境低配运行，切换更快。
- Active/Active：多区域同时服务，成本最高，恢复能力最好。

成本类服务也要分清楚。AWS Budgets 用来预算和告警，Cost Explorer 用来分析和可视化费用，Cost and Usage Reports 是最细粒度账单数据，Trusted Advisor 提供优化建议，Cost Anomaly Detection 发现异常费用模式。Reserved Instances 和 Savings Plans 也不是一回事：Compute Savings Plans 灵活性更高，Reserved Instances 更偏特定实例承诺。

## 易错点

- Reserved Instances 和 Compute Savings Plans 都能省钱，但灵活性不同；跨实例族、区域、服务的题目要特别看约束。
- API Gateway CORS 要在 API Gateway 层正确配置 preflight，不只是后端返回 header。
- CodeDeploy 部署到 EC2 需要实例上有 CodeDeploy agent；自己做 AMI 时不要漏。
- EBS provisioned IOPS 不是“按需自动扩”；吞吐和 IOPS 要按卷类型和配置判断。
- Edge-optimized API Gateway 适合全球客户端，但缓存、CloudFront 和区域访问的边界要看清。
- EC2 作为 NAT instance 时需要关闭 source/destination check；NAT Gateway 则不需要自己管这个。
- Permission boundary、SCP、resource policy 都可能让看似有权限的 principal 被拒绝。
- VPC Peering 不支持传递路由；多 VPC hub-and-spoke 优先考虑 Transit Gateway。
- DAX 是 DynamoDB 读缓存，不能替代 DynamoDB Streams、ElastiCache 或 RDS Proxy。

## 总结

AWS 认证复习最重要的不是背下所有服务，而是建立服务边界和场景判断。看到题目时先问：它是在解决身份、网络、数据、部署、治理、迁移、灾备还是成本问题；然后再按“最少运维、最高可用、最低成本、最强合规、最小改造”去收敛答案。这样即使遇到没背过的细节，也更容易排除明显不符合场景的选项。
