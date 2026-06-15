# AMD SEV-SNP

AMD Secure Encrypted Virtualization（SEV）是一组面向虚拟机的内存加密技术。SEV-SNP（Secure Nested Paging）是 SEV 系列的重要增强，在 SEV 的 VM 内存加密和 SEV-ES 的寄存器状态保护基础上，增加了对恶意 hypervisor 的内存重映射、重放和页状态攻击的硬件防护，并提供更强的 guest attestation。

## 技术演进

- SME：Secure Memory Encryption，提供系统内存加密能力。
- SEV：为不同 VM 使用不同内存加密密钥，使 hypervisor 难以读取 guest 内存明文。
- SEV-ES：加密 VM exit 时的 CPU register state，减少寄存器泄露和篡改。
- SEV-SNP：引入 RMP（Reverse Map Table）和页状态验证，增强内存完整性、所有权和 attestation。

## 核心机制

AMD SEV-SNP 的信任根是 CPU 与 AMD Secure Processor（也常称 PSP/ASP）。每个受保护 VM 拥有独立加密上下文，DRAM 中的 VM 内存以密文形式存在。Hypervisor 仍负责调度、二级页表、I/O 和设备模拟，但不能直接解密 guest 私有页。

SNP 的关键增强是 RMP。RMP 记录每个物理页当前归属、权限和状态，使硬件能够检查：

- 某个页是否被分配给指定 guest。
- Hypervisor 是否试图把同一页映射给多个 guest。
- Guest 是否已接受和验证某个页。
- 页状态转换是否符合 SNP 生命周期规则。

Guest 与 hypervisor 通常通过 GHCB（Guest-Hypervisor Communication Block）通信。GHCB 是显式共享的通信结构，guest 必须把其中内容视为不可信输入。

## 远程证明

SEV-SNP 支持 guest 在运行时向 AMD Secure Processor 请求 attestation report。报告通常包含：

- Guest policy、measurement、launch digest 和安全版本。
- VMPL、平台 TCB、固件版本等属性。
- 调用方提供的 report data，用于绑定 nonce 或临时公钥。
- VCEK/VLEK 证书链和 AMD KDS collateral。

Verifier 不应只检查报告签名，还应检查 TCB 版本、guest policy、镜像度量、启动参数和密钥释放策略。云环境中常见做法是把 SNP attestation 与 KMS 或 secret manager 结合。

## 安全模型

SEV-SNP 通常信任：

- AMD CPU、Secure Processor、微码和 SEV 固件。
- Guest firmware、guest kernel、驱动和用户态工作负载。
- Attestation verifier、证书链和密钥释放策略。

SEV-SNP 通常不信任：

- Host hypervisor、host OS、宿主机管理员。
- 同机其他 VM。
- 共享内存、虚拟设备、外部存储和网络。

## 安全边界与限制

- SNP 显著增强 VM 内存完整性，但不等于完整系统完整性。Guest OS 内漏洞仍可泄露秘密。
- 侧信道、计时、缓存和资源争用不是默认消失的风险。
- Hypervisor 仍可拒绝服务、影响调度、注入部分虚拟中断或操纵不可信 I/O。
- 设备直通、加速器和 DMA 需要 IOMMU、TDISP/Trusted I/O 或平台专门支持。
- 旧 SEV/SEV-ES 与 SEV-SNP 安全属性不同，不能混用威胁模型。
- 固件和平台补丁非常关键。证明策略应拒绝过旧 TCB。

## 工程落地

SEV-SNP 的优势是 VM 级透明度高，适合数据库、通用服务、机密 Kubernetes 节点和 AI 推理等已有工作负载。落地时需要关注：

- 云实例是否真的启用 SEV-SNP，而不是仅启用 SEV。
- Guest 镜像是否支持 SNP guest 驱动、attestation 和密钥注入。
- 是否把磁盘加密密钥、应用密钥与 attestation report 绑定。
- 是否避免在共享页、日志、崩溃转储和遥测中暴露敏感数据。

## 参考资料

- AMD SEV developer page: https://www.amd.com/en/developer/sev.html
- AMD SEV-SNP firmware ABI specification: https://www.amd.com/en/developer/sev.html
- Google Confidential VM overview: https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview
- Azure confidential VM overview: https://learn.microsoft.com/en-us/azure/confidential-computing/confidential-vm-overview

