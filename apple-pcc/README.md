# Apple Private Cloud Compute

Apple Private Cloud Compute（PCC）是 Apple 为 Apple Intelligence 构建的私有云 AI 推理架构。它不是给第三方任意部署 workload 的通用 TEE，而是 Apple 自有设备、Apple silicon 服务器、受限云操作系统、透明日志和客户端验证机制组成的一套端到端隐私架构。

## 设计目标

Apple 公布的 PCC 目标包括：

- Stateless computation：用户数据只用于完成请求，不在响应后保留。
- Enforceable guarantees：隐私保证尽量由技术机制强制，而不是只依赖运维政策。
- No privileged runtime access：不提供可绕过隐私边界的远程 shell、调试或动态加载能力。
- Non-targetability：攻击者难以把特定用户请求导向被攻陷节点。
- Verifiable transparency：研究者可以验证生产软件镜像和 attestation 日志。

## 架构组成

- PCC node：基于 Apple silicon 的定制服务器节点，包含 Secure Enclave 和 Secure Boot。
- Hardened OS：裁剪自 iOS/macOS 基础能力的服务器操作系统，减少攻击面。
- Code Signing/Trust Cache：节点运行代码必须被签名并纳入 trust cache，运行时不能加载任意新代码。
- End-to-end encryption to node：设备只把请求加密给已验证 PCC 节点的公钥。
- OHTTP relay 和 blind signature：降低请求与用户身份、源 IP 的关联。
- Transparency log：生产镜像度量进入可验证透明日志，客户端只信任公开记录的镜像。

## 安全架构视角

PCC 的核心不是“云上跑一个 enclave”，而是把端侧设备、云侧 Apple silicon、签名系统、透明日志和请求路由绑成一条可验证链。它试图解决的问题是：当本地设备需要更大模型能力时，如何让云端处理尽量接近本地处理的隐私边界。

可以按以下层次理解：

```text
User device
  -> verifies PCC software identity and transparency log
  -> encrypts request to PCC node public key
  -> sends through privacy-preserving relay/routing

PCC node
  -> secure boot and signed software stack
  -> no privileged shell/debug path for production data
  -> stateless request processing
  -> constrained logging/telemetry
```

PCC 与通用 TEE 的最大差异：用户不能自定义 workload。Apple 把可运行软件、硬件、操作系统、模型服务和运维方式整体收紧，然后通过公开验证机制让设备决定是否把请求发给某个节点。

## 工作原理

当设备判断本地模型不足以处理请求时，会选择 PCC。设备先验证候选 PCC 节点证书、软件度量和透明日志状态，然后把请求密钥只封装给符合策略的节点。负载均衡、日志、普通数据中心服务不持有解密请求所需密钥。

PCC 节点在处理过程中可以看到请求明文，因为大模型推理必须对输入进行计算。但架构通过以下方式限制风险：

- 节点不提供普通远程 shell 和交互调试。
- 代码签名防止运行时加载新工具或扩大权限。
- 数据卷密钥在重启时随机化且不持久化，实现加密擦除。
- 日志和指标是预定义、受审计、结构化的，避免任意日志泄露用户数据。
- 请求路由使用 target diffusion，降低小规模节点攻陷对特定用户的可利用性。

## 请求生命周期

一个 PCC 请求可以拆成以下安全步骤：

1. **本地决策**：设备先尝试本地模型；只有需要更强能力时才进入 PCC。
2. **节点发现**：设备获得候选 PCC 节点和其可验证身份材料。
3. **软件验证**：设备检查节点运行的软件 measurement 是否出现在公开透明日志中。
4. **密钥绑定**：设备把请求加密给该节点的硬件/软件状态绑定公钥。
5. **隐私路由**：请求通过中继和路由机制降低源 IP、Apple ID 与请求内容的关联。
6. **节点处理**：PCC node 在受限 runtime 中执行模型推理。
7. **响应返回**：结果返回设备；节点不保留用户请求明文。
8. **状态销毁**：临时密钥、请求 buffer、ephemeral state 在请求结束后清理。

这里的安全重点是“数据只给已验证的软件栈”，而不是“云平台完全无法看到计算明文”。PCC node 内部仍然会处理请求明文，因此节点软件栈和运行时约束是 TCB。

## 透明性与不可定向攻击

PCC 强调两个普通 confidential VM 不一定具备的系统属性：

- **Verifiable transparency**：生产软件镜像进入透明日志，安全研究者和客户端可以检查可运行代码集合。
- **Non-targetability**：攻击者即使控制部分节点，也难以让特定用户的特定请求稳定落到被攻陷节点。

这两个属性面向的是云 AI 服务的现实威胁：单个硬件 TEE 证明只能说明“某个节点运行了某个镜像”，但不能说明服务是否偷偷给某类用户下发特制镜像、是否允许运维临时进入节点、是否记录了请求内容。PCC 通过系统级约束把这些行为变得更难隐藏。

## 与通用机密计算的关系

| 维度 | Apple PCC | TDX/SEV-SNP/CCA/Nitro |
| --- | --- | --- |
| workload | Apple 固定 AI 服务 | 用户可部署 VM/enclave/container |
| 证明对象 | Apple silicon 节点、PCC OS、透明日志镜像 | VM/enclave measurement 和平台 TCB |
| 密钥释放 | Apple 设备验证后加密请求 | KMS/数据方根据 attestation policy 放密钥 |
| 运行边界 | 无通用 shell/debug，服务专用 | 取决于云平台和用户镜像 |
| 生态 | Apple 封闭生态 | 云/硬件厂商生态 |
| 主要场景 | Apple Intelligence 私有云推理 | 企业 workload、数据协作、密钥处理 |

## 安全模型

PCC 通常信任：

- Apple silicon、Secure Enclave、Secure Boot 和硬件供应链控制。
- PCC hardened OS、签名服务、透明日志和客户端验证逻辑。
- Apple 发布的模型和服务代码符合公开隐私承诺。

PCC 尽量不信任：

- 数据中心普通运维面、负载均衡、日志系统。
- 单个被物理攻击或运行时攻陷的 PCC 节点。
- 网络路径和第三方中继。

## 安全边界与限制

- PCC 不是完全端到端加密计算；推理节点内部需要处理明文。
- 用户不能把任意代码部署到 PCC，也不能把它当作通用 confidential VM。
- 安全性依赖 Apple 的硬件、签名、透明日志和客户端策略。
- 公开透明提高可验证性，但二进制审计仍比源代码审计困难。
- 模型输出可能泄露输入中的敏感信息，应用层仍需最小化发送内容。
- PCC 主要保护请求处理过程，不解决用户主动输入敏感信息后模型输出传播的问题。
- 安全属性依赖客户端验证逻辑正确执行；若客户端策略被绕过，云端证明链失效。
- PCC 不面向跨组织数据 clean room 或第三方自定义计算。
- 模型权重、推理代码、调度策略和透明日志的审计粒度由 Apple 设计控制。

## 适用场景

PCC 适合 Apple 生态中的云端 AI 推理隐私保护。它的价值在于把设备安全模型扩展到云节点，而不是提供跨云、跨厂商的通用隐私计算接口。对于企业通用 workload，应评估 TDX/SEV-SNP/Nitro/Confidential VM；对于多方数据协作，应评估 Confidential Space、MPC 或数据 clean room。

## 参考资料

- Apple PCC technical overview: https://security.apple.com/blog/private-cloud-compute/
- Apple Platform Security: https://support.apple.com/guide/security/welcome/web
