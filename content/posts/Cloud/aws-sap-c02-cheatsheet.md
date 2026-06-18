---
title: "AWS SAP-C02 认证速查笔记"
date: 2024-08-20T10:00:00+08:00
draft: true
categories: ["Cloud"]
---

> 这是一份认证备考速查笔记，按服务域和考试易错点组织；AWS 服务能力和考试范围会变化，正式备考应以 AWS 最新官方文档和考试指南为准。

## Identity
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>IAM</td><td>无法跨账户 assume role</td></tr>
<tr><td>IAM Service Role</td><td>将权限授予 AWS 服务，从而代为执行操作</td></tr>
<tr><td>STS（Security Token Service）</td><td>创建临时的、有限权限的凭证</td></tr>
<tr><td>SAML（Security Assertion Markup Language）</td><td>在身份提供商（IdP）和服务提供商（SP）之间交换认证和授权数据；通常用于实现单点登录（SSO）</td></tr>
<tr><td>Cognito</td><td>身份与访问管理：<br>· identity pool：向 AWS 授予临时访问权限<br>· user pool：用于用户注册和认证</td></tr>
</tbody>
</table>

### Identity-based Policy vs. Resource-based Policy
<figure><img src="/attachments/aws-sap-c02/diagram-9.png" alt="Identity-based 与 Resource-based 策略对比图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>

### IAM Permission Boundary
<p>权限边界：限制一个 IAM User / Role 能够获得的最大权限。</p>
<figure><img src="/attachments/aws-sap-c02/diagram-1.png" alt="IAM 权限边界示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>

### STS（AWS Security Token Service）
<p>创建临时安全凭证。</p>
<figure><img src="/attachments/aws-sap-c02/diagram-2.png" alt="STS 示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>
<figure><img src="/attachments/aws-sap-c02/diagram-3.png" alt="STS 示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>

### Trust Policy &amp; Permission Policy
<p>Trust Policy 指定哪个主体（Principal）可以执行哪些操作（Action）。</p>
      <pre><code>{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Alice"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}</code></pre>
<p>Permission Policy 定义拥有该角色的用户或服务，可以对哪些 AWS 资源执行哪些操作。</p>
      <pre><code>{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example_bucket"
    },
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example_bucket/*"
    }
  ]
}</code></pre>

## Computing
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>EC2</td><td>虚拟服务器实例</td></tr>
<tr><td>EC2 Graviton</td><td>更高的性价比与性能</td></tr>
<tr><td>Lambda@Edge</td><td>在 CloudFront 上运行 Lambda：<br>· 在 CloudFront 分发阶段运行，最长 15 分钟<br>· 在 VPC 内时，通过 NAT Gateway 访问公网<br>· 凭证不能存储在环境变量中</td></tr>
<tr><td>Lambda 并发</td><td>· Provisioned Concurrency：始终保持一定数量实例就绪，减少冷启动，成本较高<br>· Reserved Concurrency：限制函数可运行的最大实例数<br>· SnapStart：减少冷启动，不能与 Provisioned Concurrency 同时使用</td></tr>
<tr><td>Step Functions</td><td>编排 Lambda 与其他 AWS 服务：<br>· Standard workflow：exactly-once，最长 1 年，适合长时间、可审计的流程<br>· Express workflow：at-least-once，最长 5 分钟，适合高事件率负载</td></tr>
<tr><td>EKS（Elastic Kubernetes Service）</td><td>托管 Kubernetes，较复杂：<br>· EKS：在 AWS 托管的集群/基础设施上运行 Kubernetes<br>· EKS Anywhere：在本地（on-premises）基础设施上运行<br>· EKS Distro：在自有集群/基础设施上运行<br>· Topology Spread Constraints：确保 pods 在多个 AZ 间分布，提高弹性<br>· Cluster Autoscaler：自动伸缩节点</td></tr>
<tr><td>ECS（Elastic Container Service）</td><td>容器编排；run-task 时将 enableECSManagedTags 设为 true</td></tr>
<tr><td>Fargate</td><td>无服务器方式运行 ECS 容器</td></tr>
<tr><td>App2Container</td><td>将现有应用打包成容器并部署到 ECS 或 EKS，减少代码修改和手动配置（containerize and migrate existing app）</td></tr>
<tr><td>Compute Optimizer</td><td>分析利用率指标并生成优化建议；Enhanced infrastructure metrics 可将数据分析周期扩展到 3 个月（额外费用）</td></tr>
<tr><td>SES（Simple Email Service）</td><td>发送邮件，存储邮件模板</td></tr>
<tr><td>AppSync</td><td>托管 GraphQL API；实时数据查询与同步；支持 GraphQL 和 WebSocket</td></tr>
</tbody>
</table>
<figure><img src="/attachments/aws-sap-c02/diagram-4.png" alt="计算类服务示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>

### EC2
#### Launch Types
<ol>
<li>On-Demand</li>
<li>Spot</li>
<li>Reserved</li>
</ol>

#### Placement Groups
<figure><img src="/attachments/aws-sap-c02/diagram-5.png" alt="EC2 Placement Groups 示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>
<ol>
<li><strong>Cluster</strong>：同一 AZ、彼此靠近；适合低延迟应用，如 HPC、大数据</li>
<li><strong>Partition</strong>：分散到多个分区，避免单点硬件故障；适合大规模分布式、容错系统，如 Hadoop</li>
<li><strong>Spread</strong>：分散到不同的底层硬件</li>
</ol>

#### Metrics
<figure><img src="/attachments/aws-sap-c02/diagram-6.png" alt="EC2 Metrics 示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>

### Lambda
<figure><img src="/attachments/aws-sap-c02/diagram-7.png" alt="Lambda 示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>

## Data
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>S3</td><td>对象级存储（object-level），速度相对最慢：<br>· 事件通知：可用 Lambda 处理上传的对象<br>· Apache Parquet：用于 S3 inventory 报告，加速大数据任务<br>· S3 Replication Time Control：15 分钟内复制 99.99% 对象<br>· 跨区域：在新区域创建桶并配置跨区域复制</td></tr>
<tr><td>S3 Storage Lens</td><td>分析 S3 存储的使用情况和成本</td></tr>
<tr><td>S3 Select</td><td>适合查询特定数据，不适合大规模数据分析</td></tr>
<tr><td>S3 Transfer Acceleration</td><td>利用 CloudFront 全球边缘位置加速向 S3 上传文件</td></tr>
<tr><td>S3 Server Access Logging</td><td>只能记录请求，不能记录对象访问</td></tr>
<tr><td>S3 Object Lock</td><td>数据保护；通过 WORM（Write Once, Read Many）防止对象被删除或覆盖</td></tr>
<tr><td>EBS</td><td>块级存储（block-level），最大 16TB，持久化，单实例挂载，速度最快</td></tr>
<tr><td>Data Lifecycle Manager（DLM）</td><td>创建、保留和删除 EBS 卷快照与 AMI，实现更高效的备份管理</td></tr>
<tr><td>EFS</td><td>文件级存储（file-level），共享，无服务器，仅支持 Linux</td></tr>
<tr><td>FSx</td><td>共享、高性能、适合大数据分析：<br>· FSx for Lustre：Linux<br>· FSx for Windows：Windows<br>· FSx for OpenZFS：Linux / Windows<br>· 扩容：update-file-system</td></tr>
<tr><td>DynamoDB（DDB）</td><td>· WCU/RCU：写/读容量单位<br>· Provisioned Throughput：预留指定数量的 RCU/WCU<br>· On-Demand：按请求付费，无需容量规划，按实际流量自动伸缩<br>· 无法直接通知其他服务，需配合 EventBridge<br>· DAX（DynamoDB Accelerator）：仅用于读、不用于写，集群至少 3 个节点<br>· TTL：在指定时间戳后自动过期条目<br>· 转为全球表：A. 直接改为 global；B. 启用 DDB Stream 并在第二区域创建复制表</td></tr>
<tr><td>Aurora</td><td>全托管，仅支持 MySQL / PostgreSQL，性能优于 RDS：<br>· Aurora Serverless v1：应对突发流量，空闲期降低成本，适合不确定/间歇性负载<br>· Multi-master 集群：提升写性能、成本高、无法跨区域写<br>· Cluster Endpoint：集群唯一端点，路由到主实例，处理读写<br>· Reader Endpoint：连接 read replica，处理读操作</td></tr>
<tr><td>RDS</td><td>全托管数据库服务：<br>· RDS Proxy：管理应用与数据库间的连接池，降低连接管理开销<br>· Multi-AZ Deployment：提供高可用与数据冗余</td></tr>
<tr><td>Timestream</td><td>时序数据库</td></tr>
<tr><td>CloudFront</td><td>全球 CDN，根据用户位置自动选择最优边缘位置：<br>· 可有多个 origin，位于任意区域<br>· 可通过 origin group 实现自动区域切换</td></tr>
<tr><td>Kinesis Data Streams</td><td>数据流服务，实时处理和分析数据流（复杂事件），不能用作 store 或 buffer</td></tr>
<tr><td>Kinesis Data Firehose</td><td>捕获、转换并加载数据流到存储（简单事件）：全托管、自动扩展，输出到 S3、Redshift、OpenSearch、Splunk，可用作 buffer</td></tr>
<tr><td>MQ</td><td>适用于已在使用 ActiveMQ 或 RabbitMQ 并希望迁移到 AWS 的场景</td></tr>
<tr><td>SQS</td><td>全托管消息队列</td></tr>
<tr><td>OpenSearch</td><td>日志分析、实时搜索（源自 Elasticsearch）：<br>· general node：使用 EBS，大数据量时较贵<br>· instance-backed：EBS 或实例存储，容量有限<br>· UltraWarm：索引用 SSD、其余用 S3，最便宜，作为其他类型的补充</td></tr>
<tr><td>EMR</td><td>大数据分析（Apache Spark、Hive、Presto）</td></tr>
<tr><td>Athena</td><td>分析数据，使用 SQL 或 Apache Spark</td></tr>
<tr><td>QuickSight</td><td>商业智能（BI），仪表盘</td></tr>
<tr><td>Data Pipeline</td><td>自动化数据的移动与转换</td></tr>
<tr><td>DataSync</td><td>将文件数据迁移到 AWS，如 S3、EFS、FSx for Windows File Server</td></tr>
<tr><td>Database Migration Service（DMS）</td><td>将数据库迁移到 AWS（MySQL、PostgreSQL、Oracle、SQL Server 等）：支持迁移期间的数据复制，支持同构与异构迁移</td></tr>
<tr><td>Storage Gateway</td><td>本地访问云存储：<br>· File Gateway：S3、Glacier<br>· Tape Gateway：长期归档<br>· Volume Gateway：块存储</td></tr>
<tr><td>Glue</td><td>自动化 ETL 服务；Glue Data Catalog 管理元数据，与 Athena 集成，便于查询和分析 S3 中的数据</td></tr>
<tr><td>Transfer Family</td><td>文件传输；AWS Transfer for SFTP 为 S3 提供全托管 SFTP 服务</td></tr>
<tr><td>IAM database authentication</td><td>通过 IAM 用户或角色认证，不再需要数据库凭证</td></tr>
<tr><td>DocumentDB</td><td>MongoDB 兼容，高可用与自动扩展，适合迁移</td></tr>
<tr><td>Redshift</td><td>数据仓库，支持 SQL 查询与大规模数据分析；Redshift Spectrum 可直接查询 S3 数据，无需加载到集群</td></tr>
<tr><td>AWS Backup</td><td>集中备份管理、监控，可将备份拷贝到其他区域</td></tr>
<tr><td>ElastiCache</td><td>内存缓存，可使用 auth token</td></tr>
</tbody>
</table>

### 跨区域数据库（Cross-Region Database）
#### RDS
<ol>
<li><strong>Cross-Region Read Replicas</strong>（跨区域只读副本）：可将 read replica 提升（promote）为独立主实例，主区域故障时用于灾难恢复</li>
<li><strong>Cross-Region Backups and Restore</strong>（跨区域自动备份与恢复）：主区域故障时可从备份中恢复数据库</li>
</ol>
#### Aurora
<ol>
<li><strong>Global Database</strong>：RTO &lt; 1 分钟</li>
<li><strong>跨区域 DB 集群</strong>：支持跨区域读写</li>
</ol>
<figure><img src="/attachments/aws-sap-c02/diagram-8.png" alt="Aurora 跨区域数据库示意图"><figcaption>来自备考笔记的原始示意图。</figcaption></figure>
#### DynamoDB
<ol>
<li><strong>Global Tables</strong>（全球表）：某区域故障时其他区域副本仍可用</li>
<li><strong>Cross-Region Backup and Restore</strong>：主区域故障时可从备份中恢复表</li>
</ol>

### On-Demand
<ol>
<li>EC2 On-Demand Instances</li>
<li>RDS On-Demand Instances</li>
<li>DynamoDB On-Demand Capacity Mode</li>
</ol>

### Edge-optimized
<ol>
<li><strong>API Gateway Edge-Optimized Endpoints</strong>：将 API 请求路由到最近的 CloudFront 边缘位置</li>
<li><strong>S3 Transfer Acceleration</strong>：利用 CloudFront 边缘位置加速传输</li>
<li><strong>Lambda@Edge</strong>：在 CloudFront 边缘位置运行 Lambda 函数</li>
</ol>

### Saving Plan
<ol>
<li><strong>EC2 Instance Savings Plans</strong>：针对特定 EC2 实例类型，折扣更高，类似 Reserved Instances</li>
<li><strong>Compute Savings Plans</strong>：适用于所有 AWS 计算资源，包括 EC2、Lambda、Fargate</li>
</ol>

## Security
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>KMS（Key Management Service）</td><td>管理加密密钥：<br>· 跨账户访问：1. key policy；2. IAM policy<br>· 不能转换密钥，只能新建<br>· 使用 S3 bucket keys 减少对 KMS 的调用</td></tr>
<tr><td>Secrets Manager</td><td>管理密钥/凭证生命周期：可复制，不支持通过 AWS RAM 共享 secrets</td></tr>
<tr><td>Macie</td><td>保护敏感数据（sensitive data）</td></tr>
<tr><td>CloudTrail</td><td>监控 API 调用，跟踪 API 使用情况</td></tr>
<tr><td>Security Hub</td><td>集中查看安全状态：安全态势管理、合规性检查</td></tr>
<tr><td>Detective</td><td>分析数据，调查潜在安全问题</td></tr>
<tr><td>Audit Manager</td><td>审计 AWS 使用情况，简化风险评估</td></tr>
<tr><td>Shield</td><td>DDoS 防护</td></tr>
<tr><td>Inspector</td><td>识别安全漏洞，进行安全评估</td></tr>
<tr><td>GuardDuty</td><td>监控恶意活动、识别潜在威胁（机器学习、行为分析、威胁检测）</td></tr>
<tr><td>WAF（Web Application Firewall）</td><td>保护 Web 应用免受常见漏洞攻击：定义 Web ACL 允许或阻止流量，用于 API Gateway；规则动作：allow、block、count、CAPTCHA</td></tr>
<tr><td>Firewall Manager</td><td>集中管理跨多个账户的 AWS WAF 规则</td></tr>
<tr><td>Network Access Analyzer</td><td>识别和分析网络访问路径，评估网络安全性</td></tr>
<tr><td>RADIUS</td><td>网络协议，提供集中化身份验证，支持 MFA</td></tr>
</tbody>
</table>

### KMS vs. Secrets Manager
<ul>
<li><strong>KMS</strong>：管理加密密钥</li>
<li><strong>Secrets Manager</strong>：管理 SSH 密钥对等凭证</li>
</ul>

### 身份验证
<p><strong>SSO（Single Sign-On）</strong></p>
<ul>
<li>身份验证机制</li>
<li>使用一组凭据（如用户名和密码）登录多个系统</li>
<li>自动获得访问权限，不需要重复输入凭据</li>
</ul>
<p><strong>IdP（Identity Provider）</strong></p>
<ul>
<li>SSO 通常通过 IdP 实现</li>
<li>负责验证用户身份</li>
<li>通过 OIDC、OAuth、SAML 等协议实现</li>
</ul>

## Network
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>ENI（Elastic Network Interface）</td><td>挂载到 EC2 的虚拟网络接口，作为网络适配器，实现 EC2 与 VPC 内其他资源或公网的通信</td></tr>
<tr><td>ALB</td><td>HTTP/HTTPS 负载均衡：<br>· 限制访问：CloudFront 添加自定义 HTTP 头，ALB 只转发带该头的请求<br>· 配合 OIDC IdP 处理认证和授权</td></tr>
<tr><td>VPC（Virtual Private Cloud）</td><td>虚拟网络环境：<br>· CIDR block：地址范围内的 IP 用于 VPC 资源<br>· Subnets：VPC 的逻辑分区，可位于不同 AZ；Public Subnet 可直接访问公网，Private Subnet 不能<br>· Route Tables：将流量定向到目标</td></tr>
<tr><td>VPC Gateway Endpoint</td><td>VPC ↔ 仅 S3、DynamoDB；不能从本地访问</td></tr>
<tr><td>VPC Interface Endpoint</td><td>本地 / VPC ↔ 其他服务</td></tr>
<tr><td>Gateway Load Balancer</td><td>配合 AWS PrivateLink 实现 VPC 与 AWS 服务的安全通信；部署和扩展第三方虚拟设备</td></tr>
<tr><td>Transit VPC</td><td>连接多个 VPC</td></tr>
<tr><td>Transit Gateway</td><td>本地 ↔ 多个 VPC：通过单一网关集中管理和路由流量</td></tr>
<tr><td>PrivateLink</td><td>VPC ↔ AWS 服务，不经过公网；VPC Endpoint Service 可在 CIDR 重叠时使用</td></tr>
<tr><td>Direct Connect</td><td>本地 ↔ AWS（单区域），不经过公网：物理专线，传输不加密，部署时间长、费用高；Direct Connect Gateway 可跨区域连接多个 VPC，无需为每个区域单独配置</td></tr>
<tr><td>Site-to-Site VPN</td><td>本地 ↔ AWS，通过公网进行加密通信（IPSec 加密）</td></tr>
<tr><td>VPC Peering</td><td>VPC ↔ VPC：一对一，需双方配置路由，不经过公网，不支持传递路由</td></tr>
<tr><td>Client VPN</td><td>远程连接到 AWS 或本地</td></tr>
<tr><td>VIF（Virtual Interface）</td><td>Direct Connect 上的逻辑链路：<br>· private VIF：连接到 VPC<br>· public VIF：连接公共 AWS 服务（S3、DDB…）<br>· transit VIF：连接单区域内多个 VPC</td></tr>
<tr><td>Session Manager</td><td>登录到私有子网中的主机；无需 SSH 密钥对即可安全访问 EC2，无需重启实例</td></tr>
<tr><td>Internet Gateway</td><td>VPC ↔ 公网：配置入站和出站，允许公有 IP 访问公网</td></tr>
<tr><td>NAT（Network Address Translation）</td><td>仅出站，允许私有 IP 访问公网，不能跨多个 AZ：<br>· public NAT Gateway：private subnet → 公网（单向）<br>· private NAT Gateway：private subnet → 其他 VPC 或本地网络（单向）</td></tr>
<tr><td>NACL（Network Access Control List）</td><td>默认允许所有入站和出站；无状态（stateless），每个方向需独立配置；支持 Allow 与 Deny</td></tr>
<tr><td>Security Group</td><td>用于 VPC：默认阻止所有入站、允许所有出站；只能指定 allow 规则；有状态（stateful），入站允许后对应出站自动允许</td></tr>
<tr><td>Elastic IP</td><td>通常分配给 EC2、NLB、NAT Gateway，不分配给 ALB：<br>· 分配给 EC2：单实例应用，需直接访问实例<br>· 分配给 NLB：高可用、负载均衡、统一访问点</td></tr>
<tr><td>API Gateway</td><td>管理 REST、HTTP、WebSocket API：<br>· REST API 可直接调用 DDB<br>· 同步执行立即返回响应<br>· 可启用 CORS<br>· 配合 OIDC、OAuth、JWT 使用</td></tr>
<tr><td>Route 53</td><td>全球服务，DNS：域名注册、路由、健康检查：<br>· hosted zone：记录的容器（路由规则）<br>· alias record：类似 CNAME，指向负载均衡器<br>· 可将流量路由到不同区域的 API Gateway</td></tr>
<tr><td>Global Accelerator</td><td>提供两个全球静态公网 IP，自动将流量路由到最优端点</td></tr>
<tr><td>IoT</td><td>· Greengrass：离线、本地设备的边缘计算和数据处理<br>· IoT Core：连接海量设备、处理消息并路由；支持 MQTT；用 X.509 证书做设备认证</td></tr>
</tbody>
</table>

### Route 53
<p>DNS 解析：将域名与 AWS 资源（如 EC2、S3）或外部资源的 IP 地址映射起来，支持以下记录类型：</p>
<ul>
<li>A 记录：将域名映射到 IPv4 地址</li>
<li>AAAA 记录：将域名映射到 IPv6 地址</li>
<li>CNAME 记录：将一个域名映射到另一个域名</li>
<li>MX 记录：定义邮件服务器</li>
<li>TXT 记录：存储任意文本数据，常用于域名验证</li>
</ul>

### HTTP API vs. REST API
<ul>
<li><strong>HTTP API</strong>：简单的微服务架构，基本的集成与认证功能</li>
<li><strong>REST API</strong>：功能更全面，可直接集成 DDB</li>
</ul>

### NACL vs. Security Group
<ul>
<li><strong>NACL</strong>：无状态（stateless），每个方向（inbound &amp; outbound）独立配置；支持 Allow、Deny</li>
<li><strong>Security Group</strong>：有状态（stateful），入站被允许时返回的出站响应自动允许；只支持 Allow</li>
</ul>

### 连接方式（不经过公网）
<ol>
<li>Direct Connect（本地 ↔ VPC）</li>
<li>Site-to-Site VPN（本地 ↔ VPC）</li>
<li>PrivateLink（VPC ↔ 服务）</li>
<li>VPC Peering（VPC ↔ VPC）</li>
</ol>

## OS
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>Systems Manager / Patch Manager</td><td>管理资源并应用补丁（patch），适用于混合环境（Windows 和 Linux，EC2 和本地）</td></tr>
<tr><td>Systems Manager Parameter Store</td><td>以明文或加密形式存储值、密码</td></tr>
<tr><td>Systems Manager Run Command</td><td>远程管理配置</td></tr>
<tr><td>Systems Manager Agent</td><td>允许实例与 AWS Systems Manager 服务通信</td></tr>
<tr><td>OpsWorks</td><td>通过 Chef、Puppet 进行配置管理，主要用于配置管理和自动化部署</td></tr>
</tbody>
</table>

## Workflow
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>EventBridge</td><td>事件驱动；通过 Rule 触发 Lambda、SNS</td></tr>
<tr><td>RAM（Resource Access Manager）</td><td>与其他账户共享资源（如 VPC、RDS，不包含 Lambda）；不共享 secrets 或报告</td></tr>
<tr><td>X-Ray</td><td>调试应用</td></tr>
<tr><td>Config</td><td>评估、审计配置；跟踪和检查配置，包括 Security Group</td></tr>
<tr><td>Secrets Manager</td><td>管理、检索、轮换凭证</td></tr>
<tr><td>Certificate Manager</td><td>管理 SSL、TLS 证书</td></tr>
<tr><td>Outposts</td><td>在本地运行 AWS 基础设施（混合）：让 AWS 服务可在本地数据中心运行，低延迟</td></tr>
<tr><td>WorkSpaces</td><td>虚拟桌面（DaaS），提供完整的桌面环境；Connection Alias 与主目录关联，可触发故障转移</td></tr>
<tr><td>AppStream</td><td>虚拟桌面：桌面应用流式传输，仅访问特定应用</td></tr>
<tr><td>Batch</td><td>批处理</td></tr>
</tbody>
</table>

### WorkSpaces vs. AppStream
<ul>
<li><strong>WorkSpaces</strong>：完整的桌面环境</li>
<li><strong>AppStream 2.0</strong>：特定应用程序环境</li>
</ul>

## Organization
<p>本节与下方 Deployment 中的 AWS Organizations、SCP、Control Tower 等内容关联，按账户治理与多账户管理组织。</p>

## Deployment
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>AWS Organizations</td><td>account → OU ← SCP：<br>· Consolidated Billing Mode：所有账户统一计费<br>· All Features Mode：创建 OU 层级并在不同级别应用 SCP，控制跨账户的权限和操作</td></tr>
<tr><td>SCP（Service Control Policies）</td><td>设置 IAM 用户和角色能执行的权限上限：通过 deny 限制权限（而非 allow）；在组织中用 SCP 控制，而非 IAM</td></tr>
<tr><td>CUR（Cost and Usage Report）</td><td>从组织管理账户创建</td></tr>
<tr><td>Control Tower</td><td>多账户环境治理：<br>· Landing Zone：预配置的 AWS 环境，包含多个账户和 OU，已应用安全与治理措施<br>· Guardrails：防护措施；Detective Guardrail 用于治理与合规</td></tr>
<tr><td>Service Catalog</td><td>为 IT 服务提供一致的治理与管理：创建和管理已批准的 AWS 服务，确保部署符合标准，限制可访问的服务类别</td></tr>
<tr><td>Pinpoint</td><td>通过多渠道与客户沟通</td></tr>
<tr><td>CodeDeploy</td><td>自动化部署到 EC2、本地、Lambda 或 ECS；用 appspec.yml 定义部署流程</td></tr>
<tr><td>CodeArtifact</td><td>软件包管理</td></tr>
<tr><td>CloudFormation</td><td>· Template：定义资源及配置，一般在普通账户部署<br>· StackSets：跨多账户和区域的堆栈集合，集中管理，一般在管理账户部署<br>· DeletionPolicy：保护关键资源，删除堆栈时仍保留或安全备份重要资源（在模板的资源上添加以防止删除）</td></tr>
<tr><td>CloudWatch Logs</td><td>跟踪 VPC flow logs；CloudWatch Logs subscription filters 可配合 Lambda 或 Kinesis</td></tr>
<tr><td>CloudWatch Events</td><td>触发 Lambda、无服务器工作流</td></tr>
<tr><td>CloudWatch Synthetics</td><td>创建 canaries 监控 API</td></tr>
<tr><td>Service Quotas</td><td>资源的预定义限额</td></tr>
<tr><td>Amplify</td><td>构建全栈应用，适合静态网站的部署和管理</td></tr>
<tr><td>App Runner</td><td>构建、部署、运行容器，无需管理基础设施，不能跨区域</td></tr>
<tr><td>SAM（Serverless Application Model）</td><td>构建无服务器应用</td></tr>
<tr><td>Elastic Beanstalk</td><td>PaaS：部署和扩展应用，不能跨区域；自动处理部署、容量配置、负载均衡、自动扩展和健康监控；可交换 staging/production 完成部署</td></tr>
<tr><td>Lightsail</td><td>低成本、预配置的云资源</td></tr>
</tbody>
</table>

### SCP 与 IAM 策略的关系
<ul>
<li>SCP 定义账户可执行的最大权限范围，但不授予实际权限</li>
<li>IAM 策略用于具体授予用户或角色权限</li>
</ul>

## Infra、Migration、DR &amp; Charge
<table>
<thead><tr><th>AWS 服务</th><th>用途</th></tr></thead>
<tbody>
<tr><td>AWS Budgets</td><td>预算与预测：设置预算，超过阈值时发送警报，实时监控和控制成本</td></tr>
<tr><td>Cost and Usage Reports</td><td>成本与使用数据</td></tr>
<tr><td>AWS Auto Scaling</td><td>监控并自动调整容量，适用于 EC2、ECS、DynamoDB 表与索引、Aurora 副本</td></tr>
<tr><td>Trusted Advisor</td><td>提供帮助遵循 AWS 最佳实践的建议</td></tr>
<tr><td>Migration Evaluator</td><td>分析现有 IT 资产的成本和性能，提供运行成本与优化建议，评估总拥有成本（TCO）</td></tr>
<tr><td>Migration Hub</td><td>发现与迁移：数据探索，收集和分析数据，制定迁移规划</td></tr>
<tr><td>Database Migration Service（DMS）</td><td>· CDC（Change Data Capture）：通过检测事务日志实现实时数据同步<br>· Full Load Plus CDC：先做初始全量加载，再持续 CDC</td></tr>
<tr><td>Application Discovery Service</td><td>收集使用情况，与 Migration Hub、DMS Fleet Advisor 集成，发现本地数据中心的配置、性能、网络；Agentless Discovery 依赖 VMware 等虚拟化环境，数据不够详细</td></tr>
<tr><td>Application Discovery Agent</td><td>安装在本地，用于发现与迁移，收集详细的服务器指标</td></tr>
<tr><td>Application Migration Service（原 CloudEndure）</td><td>自动化从物理、虚拟或其他云迁移到 AWS</td></tr>
<tr><td>Elastic Disaster Recovery</td><td>AWS 上的灾难恢复（原 CloudEndure Disaster Recovery 已弃用）：灾难发生时将本地环境切换到 AWS</td></tr>
<tr><td>IAM Access Analyzer</td><td>分析资源访问策略，识别过度权限或错误配置，生成警报；支持 S3、Lambda、SQS、KMS、IAM</td></tr>
<tr><td>Cost Explorer</td><td>管理成本，设置预算和警报</td></tr>
<tr><td>Active Directory</td><td>微软的目录服务，用于 Windows 环境，管理网络中的用户和计算机资源</td></tr>
<tr><td>Directory Service</td><td>设置和运行目录服务：<br>· AWS Managed Microsoft AD：在 AWS 上运行 Windows Server Active Directory<br>· Simple AD：轻量级目录服务，适合不需要完整 AD 功能的小型负载<br>· AD Connector：目录代理，集成现有本地 AD 与 AWS</td></tr>
<tr><td>Amazon Connect</td><td>· Contact Control Panel（CCP）：接听和管理电话、查看客户信息、记录和标记通话<br>· Contact Lens：机器学习驱动，自动转录通话、分析情绪、生成报告</td></tr>
<tr><td>Cost Anomaly Detection</td><td>自动化成本异常检测与警报，监控成本的异常变化</td></tr>
<tr><td>Well-Architected Tool</td><td>改进架构</td></tr>
</tbody>
</table>

### VMware 迁移
<p>从 VMware 基础设施迁移到 Amazon EC2：</p>
<ol>
<li>用 VMware vSphere client 将应用导出为 OVF（Open Virtualization Format）格式镜像</li>
<li>在目标 AWS 区域创建 S3 桶来存储镜像</li>
<li>为 VM Import 创建并应用 IAM 角色</li>
<li>使用 AWS CLI 运行 EC2 import 命令</li>
</ol>

## Others
### IP 地址
<ul>
<li>A 类：8 位前缀，如 44.x.x.x</li>
<li>B 类：16 位前缀，如 44.66.x.x</li>
<li>C 类：24 位前缀，如 192.168.31.x</li>
<li>CIDR：可变前缀，使用子网掩码（VLSM）定位，如 192.0.2.0/24（24 位前缀）</li>
<li>Network prefixes：与本地网络关联的 IP 地址范围</li>
</ul>
### SSL
<ul>
<li>Amazon 签发的公共证书不能安装在 EC2 实例上</li>
<li>要实现端到端加密，必须使用第三方 SSL 证书</li>
</ul>
### Application Auto Scaling
<p>根据需求自动增加或减少资源容量，适用于 EC2、ECS、Lambda、DynamoDB、Aurora。</p>
### 易混词
<ul>
<li>trail：跟踪</li>
<li>trial：试炼</li>
</ul>

## DR（灾难恢复）
<table>
<thead><tr><th>策略</th><th>特征</th></tr></thead>
<tbody>
<tr><td>Active/Active</td><td>两端都对外提供流量，RTO 与 RPO 趋近于 0</td></tr>
<tr><td>Warm Standby</td><td>已就绪、随时可部署，RTO 与 RPO 为分钟级</td></tr>
<tr><td>Pilot Light</td><td>仅保留快照，RTO 与 RPO 约 10 分钟</td></tr>
<tr><td>Backup &amp; Restore</td><td>RTO 与 RPO 为小时级</td></tr>
</tbody>
</table>
