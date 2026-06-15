# RISC-V CoVE

RISC-V CoVE（Confidential VM Extension）是 RISC-V 面向机密虚拟机的体系结构和 ABI 工作方向，主要由 RISC-V AP-TEE Task Group 推进。它的目标是在开放 RISC-V 平台上提供类似 Intel TDX、AMD SEV-SNP、Arm CCA 的 confidential VM 能力。

## 当前状态

CoVE 相关规范仍在演进。RISC-V AP-TEE 仓库说明该规范用于支持 RISC-V application-processor 平台上的 Confidential VM Extension，且文档在未 ratify 前可能变化。因此，学习 CoVE 时应把它视为标准化中的参考架构，而不是已经像 SEV-SNP/TDX 那样大规模商用部署的技术。

## 核心概念

- TVM（Trusted Virtual Machine）：受保护的机密虚拟机。
- TSM（TEE Security Manager）：平台可信管理层，负责 TVM 生命周期、内存委派和 attestation。
- Host/VMM：普通 hypervisor，负责调度和资源管理，但不应访问 TVM 私有状态。
- Measurement：TVM 初始镜像、配置和运行时状态的度量。
- Attestation：TSM 基于平台根密钥为 TVM 状态生成证明。
- Memory ownership：平台需要明确区分 host 内存和 TVM 私有内存。

## 设计思路

CoVE 的基本思想是把“谁能访问哪些物理页”和“谁能证明当前 VM 状态”放到比 host hypervisor 更可信的层中。Host 仍创建 VM、分配资源、处理 I/O，但当某些页被转换为 TVM 私有页后，host 不能直接映射或读取它们。

RISC-V 平台可组合使用以下能力：

- H-extension 提供虚拟化基础。
- PMP/ePMP、IOMMU 或未来内存保护扩展限制物理访问。
- TSM 在机器模式或等价特权层管理安全状态转换。
- 安全启动链保护 TSM 和平台固件。
- Attestation 把 TVM 度量绑定到设备身份和 TCB 版本。

由于 RISC-V 是开放 ISA，CoVE 的一个关键价值是把机密 VM 所需的 ISA、非 ISA、固件、ABI 和 SoC 要求公开化，便于不同厂商实现可互操作的机密计算平台。

## 安全模型

CoVE 预期信任：

- RISC-V core、SoC 安全机制、Boot ROM 和安全启动链。
- TSM、平台固件、attestation key 保护机制。
- TVM guest firmware、guest OS 和 workload。

CoVE 预期不信任：

- Host OS、hypervisor、云管理员。
- 同机普通 VM 或其他不受信任 workload。
- 普通 I/O 路径、共享页和虚拟设备。

## 安全边界与限制

- 规范和生态仍在变化，实际安全性取决于具体芯片和固件实现。
- 仅有 ISA 规范不足以保证安全；还需要 SoC 内存控制、IOMMU、调试锁定、固件更新和证书体系。
- 与其他 VM 级 TEE 一样，CoVE 不自动消除侧信道、DoS、guest 漏洞和输出泄露。
- Attestation 互操作是落地关键：verifier 需要理解 TSM 版本、平台证书链和 TVM 度量语义。

## 适用场景

CoVE 适合关注开放硬件、可审计固件、学术研究、边缘平台和未来 RISC-V 云服务器的人群。若需要今天可用的云服务，应优先选择已有云厂商的 TDX/SEV-SNP/Confidential VM；若需要 RISC-V enclave 研究，可同时参考 Keystone。

## 参考资料

- RISC-V AP-TEE specifications: https://github.com/riscv-non-isa/riscv-ap-tee
- CoVE paper: https://arxiv.org/abs/2304.06167
- RISC-V International: https://riscv.org/

