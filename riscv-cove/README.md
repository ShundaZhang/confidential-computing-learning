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

## 预期软件栈

CoVE 生态可以按以下层次理解：

```text
TVM guest: guest firmware / guest kernel / workload
TSM:       TVM lifecycle, memory ownership, measurement, attestation
Host:      KVM/VMM, resource scheduling, shared memory, device emulation
Machine:   RISC-V privilege, interrupt, IOMMU/PMP/ePMP, secure boot
```

关键是 TSM（TEE Security Manager）。它的角色类似：

- Intel TDX 的 TDX Module。
- Arm CCA 的 RMM。
- AMD SEV-SNP 中 CPU/PSP/RMP 组合的一部分安全职能。

TSM 负责让 host 的 TVM 管理请求变成受检查的状态转换，而不是任由 host 修改 TVM 私有内存和度量。

## 内存所有权模型

CoVE 需要定义 host page 与 TVM private page 的转换语义。虽然具体实现会随规范和平台变化，但安全目标通常包括：

- Host 不能把 TVM 私有页映射到 host 地址空间读取。
- Host 不能把同一物理页同时分配给多个 TVM。
- TVM 必须能确认新加入的页已进入正确私有状态。
- 页回收时必须清理或重新初始化，避免数据残留。
- DMA 设备不能绕过 TVM 内存边界。

一个抽象生命周期：

```text
Host-owned page
  -> donate/delegate to TSM
  -> assign to TVM
  -> TVM accepts and uses privately
  -> optional share for I/O
  -> reclaim with zeroing/state reset
```

这和 CCA granule delegation、TDX private/shared GPA、SEV-SNP RMP validation 是同一类问题，只是 RISC-V 试图把接口标准化。

## Attestation 设计要点

CoVE 的 attestation 需要回答四个问题：

1. 平台是否从可信 Boot ROM/firmware 启动。
2. TSM 是否为预期版本和配置。
3. TVM 初始镜像、配置、启动参数是否匹配预期。
4. 当前报告是否绑定 verifier nonce 或 TVM 生成的临时公钥。

因为 RISC-V 是多厂商开放生态，证明链的互操作比单厂商平台更难。Verifier 需要理解设备证书、TSM 证书、SoC 安全属性、固件度量、规范版本和厂商扩展 claims。

## 与 Keystone 的区别

| 维度 | CoVE | Keystone |
| --- | --- | --- |
| 目标 | 标准化 confidential VM | 开源可定制 enclave 框架 |
| 粒度 | VM/TVM | 应用/runtime enclave |
| 可信管理层 | TSM | Security Monitor |
| 主要场景 | 云/服务器 RISC-V | 研究、边缘、可验证 TEE |
| 状态 | 标准和生态演进中 | 已有开源框架和论文实现 |
| 类比 | TDX/SEV-SNP/CCA | SGX/应用级 TEE |

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
- Host 仍可能控制调度、中断、I/O、时钟和资源压力。
- 设备访问需要 IOMMU/设备证明/受保护 I/O，否则 DMA 可能破坏边界。
- 规范未定稿时，不同原型的安全 claims 不能直接等价。
- 开放硬件提高可审计性，但也意味着平台安全质量差异可能更大。

## 适用场景

CoVE 适合关注开放硬件、可审计固件、学术研究、边缘平台和未来 RISC-V 云服务器的人群。若需要今天可用的云服务，应优先选择已有云厂商的 TDX/SEV-SNP/Confidential VM；若需要 RISC-V enclave 研究，可同时参考 Keystone。

## 参考资料

- RISC-V AP-TEE specifications: https://github.com/riscv-non-isa/riscv-ap-tee
- CoVE paper: https://arxiv.org/abs/2304.06167
- RISC-V International: https://riscv.org/
