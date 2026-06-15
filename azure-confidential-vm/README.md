# Azure Confidential VM

Azure Confidential VM 是 Microsoft Azure 的机密虚拟机服务，底层使用 AMD SEV-SNP 或 Intel TDX 等硬件 TEE。它面向“尽量少改代码”的云迁移：把整台 VM 置于硬件隔离边界内，同时结合 vTPM、Secure Boot、attestation 和机密 OS 磁盘加密。

## 核心能力

- VM 级硬件隔离：在 VM、hypervisor 和 host 管理代码之间建立硬件边界。
- Guest attestation：VM 可验证自己运行在 SEV-SNP 或 TDX 平台上。
- vTPM：为 VM 提供 TPM 2.0 风格的测量、密钥和安全状态。
- Confidential OS disk encryption：OS 磁盘密钥绑定到 VM 的证明和 vTPM。
- Secure key release：密钥释放与平台 attestation 成功结果绑定。
- Secure Boot：验证启动组件签名，降低启动链篡改风险。

## 工作原理

Azure Confidential VM 的启动链通常把平台硬件证明、固件度量、vTPM 状态和磁盘密钥释放连接起来。若平台缺少关键隔离设置，例如未启用 SEV-SNP，attestation 不应通过，VM 也不应获得解密启动磁盘所需密钥。

从用户视角，Confidential VM 仍是 IaaS VM。应用通常不需要改写为 enclave 结构，但为了获得完整保护，需要额外配置：

- 选择支持 confidential VM 的实例族和区域。
- 使用合格 OS 镜像。
- 开启或明确选择 confidential OS disk encryption。
- 在应用密钥释放前执行 guest attestation。
- 避免把敏感数据写入未加密临时盘、日志或外部遥测。

## 安全模型

Azure Confidential VM 通常信任：

- 底层 CPU TEE（AMD SEV-SNP 或 Intel TDX）。
- Azure attestation、vTPM 和密钥释放服务。
- Guest OS、驱动、应用和镜像供应链。

通常不信任：

- Host hypervisor 和 host OS。
- 云管理员或普通运维平面。
- 同机租户、外部网络、普通存储路径。

## 安全边界与限制

- Azure 服务集成改善易用性，但也引入云平台 attestation/KMS/IAM 配置依赖。
- VM 级 TCB 较大，guest OS 漏洞仍在边界内。
- 不同实例族、区域、OS 和功能支持不一致；部分功能如 live migration、backup、site recovery 可能受限。
- 若用户选择平台管理密钥，便利性更高，但信任模型不同于客户完全控制密钥。
- GPU confidential VM 需要单独确认 NCC/H100 等实例、NVIDIA CC 和驱动支持。

## 适用场景

Azure Confidential VM 适合企业云迁移、数据库、应用服务器、合规数据处理、机密容器节点和需要 Azure KMS/Attestation 集成的业务。如果应用可以拆成小型密钥服务，可考虑结合 enclave；如果是多方协作数据处理，可比较 Google Confidential Space 或 MPC。

## 参考资料

- Azure confidential computing overview: https://learn.microsoft.com/en-us/azure/confidential-computing/overview
- Azure confidential VM overview: https://learn.microsoft.com/en-us/azure/confidential-computing/confidential-vm-overview
- Azure attestation: https://learn.microsoft.com/en-us/azure/attestation/

