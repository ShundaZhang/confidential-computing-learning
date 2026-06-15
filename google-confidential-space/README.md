# Google Confidential Space

Google Confidential Space 是 Google Cloud 面向多方数据协作的机密计算服务。它基于 Confidential VM、hardened OS image、容器 workload 和 attestation，把“数据拥有方只向授权工作负载释放秘密”作为核心模型。

## 核心概念

- Workload：处理受保护资源的容器镜像。
- Confidential Space image：基于 Container-Optimized OS 的加固镜像。
- Confidential VM：底层硬件隔离环境，可使用 AMD SEV、SEV-SNP 或 Intel TDX 等技术。
- Attestation service：返回包含 workload 身份和环境属性的 token。
- Data owner policy：数据拥有方根据 attestation claims 决定是否释放数据或密钥。
- Operator：部署和运行环境的一方，理论上不应看到被处理数据。

## 工作原理

Confidential Space 把多方协作拆成三个角色：

1. 数据拥有方保存敏感数据或密钥，并定义访问策略。
2. Workload 发布者提供容器镜像和预期度量。
3. Operator 在 Confidential Space 环境中部署 workload。

运行时，workload 在 Confidential VM 上启动。Attestation service 根据硬件 TEE、镜像、启动参数和 workload 属性生成 identity token。数据拥有方验证 token 中的 claims，确认“这是约定代码，运行在合格机密环境中”，再释放数据或密钥。Operator 即使能管理项目和部署流程，也不应获得明文数据。

## 安全模型

Confidential Space 通常信任：

- 底层 Confidential VM 硬件 TEE 和 guest attestation。
- Google Confidential Space image、attestation service 和 token claims。
- Workload 镜像、代码和数据拥有方策略。

通常不信任：

- 环境 operator、host 管理员和普通云运维面。
- 未经授权的 workload 镜像或启动参数。
- 外部网络、日志、普通存储和不在策略内的服务账号。

## 安全边界与限制

- Workload 代码本身是关键 TCB。数据拥有方必须审计镜像、digest、入口参数和输出路径。
- Attestation policy 应精确绑定镜像 digest、项目、服务账号、环境属性和 TEE 类型。
- 输出仍可能泄露输入。多方分析需要最小化结果、审计查询和必要时加入差分隐私。
- Operator 可拒绝服务或不运行 workload。
- 与所有 VM 级 TEE 一样，侧信道和 guest 内漏洞仍需独立治理。

## 适用场景

Confidential Space 适合跨组织数据 clean room、广告转化分析、金融风控、医疗研究、模型评估和“数据不出域但允许指定代码处理”的协作。它比裸 Confidential VM 更关注数据拥有方策略和 workload 身份；比 MPC/FHE 更易部署，但需要信任硬件和云平台证明。

## 参考资料

- Google Confidential Space overview: https://docs.cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview
- Google Confidential VM overview: https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview
- Confidential Space security overview: https://docs.cloud.google.com/confidential-computing/confidential-space/docs/security-overview

