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

## 工作原理

当设备判断本地模型不足以处理请求时，会选择 PCC。设备先验证候选 PCC 节点证书、软件度量和透明日志状态，然后把请求密钥只封装给符合策略的节点。负载均衡、日志、普通数据中心服务不持有解密请求所需密钥。

PCC 节点在处理过程中可以看到请求明文，因为大模型推理必须对输入进行计算。但架构通过以下方式限制风险：

- 节点不提供普通远程 shell 和交互调试。
- 代码签名防止运行时加载新工具或扩大权限。
- 数据卷密钥在重启时随机化且不持久化，实现加密擦除。
- 日志和指标是预定义、受审计、结构化的，避免任意日志泄露用户数据。
- 请求路由使用 target diffusion，降低小规模节点攻陷对特定用户的可利用性。

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

## 适用场景

PCC 适合 Apple 生态中的云端 AI 推理隐私保护。它的价值在于把设备安全模型扩展到云节点，而不是提供跨云、跨厂商的通用隐私计算接口。对于企业通用 workload，应评估 TDX/SEV-SNP/Nitro/Confidential VM；对于多方数据协作，应评估 Confidential Space、MPC 或数据 clean room。

## 参考资料

- Apple PCC technical overview: https://security.apple.com/blog/private-cloud-compute/
- Apple Platform Security: https://support.apple.com/guide/security/welcome/web

