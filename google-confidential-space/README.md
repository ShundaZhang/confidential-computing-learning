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

## 角色与信任关系

Confidential Space 的重点是把“运行环境 operator”和“数据拥有方”拆开：

| 角色 | 拥有什么 | 不应看到什么 |
| --- | --- | --- |
| Data owner | 数据、密钥、释放策略 | 不一定控制运行项目 |
| Workload author | 容器镜像、代码、声明的处理逻辑 | 不一定拥有数据 |
| Operator | GCP 项目、VM、调度和网络 | 不应看到数据明文 |
| Verifier/KMS | attestation policy、密钥释放 | 只向合格 workload 放密钥 |

这个模型适合数据 clean room：参与方不把数据直接交给 operator，而是只交给经过证明的 workload。

## Attestation claims 与 workload identity

Confidential Space 的 attestation token 通常会表达：

- 底层是否是 Confidential VM。
- TEE 类型和平台属性。
- Confidential Space 镜像/环境属性。
- 容器镜像 digest。
- 启动命令、环境、project、service account 等上下文。
- Nonce 或 audience，防止 token 被挪用到其他协议。

数据拥有方的 policy 应尽量精确绑定镜像 digest，而不是 tag；绑定 service account 和 project，而不是只绑定组织；绑定 expected audience 和 nonce，而不是接受任意 token。

## 数据释放流程

一个典型多方数据处理流程：

```text
Operator starts Confidential Space VM
  -> container workload starts
  -> workload requests attestation token
  -> data owner verifies claims
  -> data owner releases encrypted dataset key
  -> workload decrypts data inside CVM
  -> workload computes approved result
  -> result is published to allowed sink
```

安全设计要同时控制输入和输出。输入密钥释放靠 attestation policy；输出则需要 workload 代码、结果最小化、审计、差分隐私或人工审批等机制约束。否则“可信 workload”仍可能把原始数据写到授权输出位置。

## 与普通 Confidential VM 的区别

| 维度 | Confidential VM | Confidential Space |
| --- | --- | --- |
| 抽象 | 一台机密 VM | 受证明的容器 workload + 数据协作策略 |
| 主要用户 | VM operator | 数据拥有方 + operator + workload author |
| 密钥释放 | 通常由 VM owner 控制 | 可由外部数据拥有方根据 claims 控制 |
| 适合 | 单组织 workload 保护 | 多方 clean room / 联合分析 |
| 风险重点 | guest TCB、I/O、云 KMS | workload digest、输出泄露、policy 精度 |

Confidential Space 可以看作把 confidential VM 的底层隔离包装成更适合跨组织协作的产品形态。

## 输出控制与 clean room 风险

Confidential Space 保护数据进入 workload 后不被 operator 直接读取，但合法 workload 的输出仍可能泄露敏感信息。常见控制包括：

- 只允许固定查询或固定模型评估。
- 对聚合结果设置 k-anonymity、阈值或差分隐私。
- 限制输出 sink，例如只能写到指定 bucket/table。
- 审计 workload 镜像源码和构建流程。
- 使用 signed container image 和 SLSA/provenance。
- 对每次密钥释放记录 token claims 和结果摘要。

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
- 容器 tag 可变，不能作为安全身份；应使用 immutable digest。
- 环境变量、命令行参数和 mounted secret 也可能影响 workload 行为，应纳入策略或禁止动态变化。
- Workload 网络出口可能成为数据外传路径，需要 egress policy 和应用层控制。
- 多方协作中还要解决法律授权、审计可解释性和结果再识别风险。

## 适用场景

Confidential Space 适合跨组织数据 clean room、广告转化分析、金融风控、医疗研究、模型评估和“数据不出域但允许指定代码处理”的协作。它比裸 Confidential VM 更关注数据拥有方策略和 workload 身份；比 MPC/FHE 更易部署，但需要信任硬件和云平台证明。

## 参考资料

- Google Confidential Space overview: https://docs.cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview
- Google Confidential VM overview: https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview
- Confidential Space security overview: https://docs.cloud.google.com/confidential-computing/confidential-space/docs/security-overview
