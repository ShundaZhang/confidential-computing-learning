# Arm CCA

Arm Confidential Compute Architecture（CCA）是 Armv9-A 面向机密计算的体系结构。它在传统 Normal world 和 Secure world 之外引入 Realm world，使云和边缘平台能够运行受保护的 Realm VM。用户提到的 “ARM ACC” 通常应指 Arm CCA。

## 核心概念

- Realm：受 CCA 保护的执行环境，可承载机密 VM。
- RME（Realm Management Extension）：Arm CCA 的体系结构扩展，提供 Realm 内存隔离和状态转换。
- RMM（Realm Management Monitor）：管理 Realm 生命周期、Realm Execution Context 和证明的可信固件组件。
- GPT（Granule Protection Table）：按 granule 记录物理内存属于 Non-secure、Secure、Realm 或 Root 等安全状态。
- REC（Realm Execution Context）：Realm 中 vCPU/执行上下文的抽象。
- CCA attestation token：用于远程验证 Realm 度量和平台状态的证明。

## 工作原理

Arm CCA 把机密 VM 的安全边界从普通 hypervisor 中移出。Normal world 中的 host OS 和 hypervisor 仍负责资源管理、调度和普通 I/O，但 Realm 私有内存由 RME/GPT 和 RMM 保护。Host 不能直接访问 Realm 私有页，也不能把同一物理 granule 任意重映射到不可信地址空间。

典型启动流程：

1. 平台从 Root/EL3 固件建立信任链。
2. RMM 被加载和度量。
3. Host 请求创建 Realm，并把内存 granule 委派给 Realm。
4. RMM 验证状态转换，建立 Realm 初始度量。
5. Realm 运行 guest firmware、kernel 和 workload。
6. 远程 verifier 根据 CCA token 决定是否释放密钥。

CCA 和 TrustZone 的区别很重要。TrustZone 主要面向设备级 Secure world，常用于手机和嵌入式安全服务；CCA 面向多租户 confidential VM，目标是把云租户从 host hypervisor 中隔离出来。

## 安全模型

Arm CCA 通常信任：

- Arm CPU/SoC 的 RME 实现、EL3 固件和启动链。
- RMM、Realm guest firmware、guest OS 和 workload。
- Attestation service 与 verifier policy。

Arm CCA 通常不信任：

- Normal world host OS、KVM/hypervisor、云管理员。
- 同机普通 VM 或其他 Realm。
- 普通世界的设备模拟、共享内存和 I/O 路径。

## 安全边界与限制

- CCA 的核心边界是 Realm 私有内存和 Realm 执行上下文，不自动保护所有设备。
- I/O 仍是难点。设备直通需要 SMMU/IOMMU、设备证明和平台策略配合。
- Host 仍可影响调度、资源和可用性。
- 侧信道和共享微架构资源需要实现和软件共同缓解。
- 生态仍在成熟中。不同 SoC、固件、RMM 版本、Linux/KVM 支持状态会影响可用性。

## 适用场景

CCA 适合 Arm 服务器、边缘云、运营商基础设施和需要 VM 级隔离的多租户平台。它与 TDX、SEV-SNP 处于同一类技术，但会受到具体 Arm SoC、固件和云服务支持进度影响。

## 参考资料

- Arm CCA overview: https://www.arm.com/architecture/security-features/arm-confidential-compute-architecture
- Arm CCA documentation: https://developer.arm.com/documentation/den0125/latest/
- Arm CCA open-source software: https://www.trustedfirmware.org/projects/tf-rmm/

