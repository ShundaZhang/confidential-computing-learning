# 机密计算与隐私计算学习资料

本仓库整理主流机密计算（Confidential Computing）和隐私计算（Privacy-Enhancing Computation）技术。重点关注数据在使用中（data in use）的保护：一类方案通过硬件可信执行环境（TEE）隔离运行时；另一类方案通过密码学或统计方法降低数据共享、联合建模和可验证计算中的隐私暴露。

资料更新时间：2026-06-15。机密计算依赖微码、固件、云平台能力和安全公告，落地前应复核厂商最新文档、实例规格和补丁状态。

## 目录

### 硬件与平台 TEE

| 技术 | 粒度 | 典型场景 | 资料 |
| --- | --- | --- | --- |
| Intel SGX | 应用/函数级 enclave | 密钥保护、小 TCB 服务、LibOS | [intel-sgx](./intel-sgx/README.md) |
| Intel TDX | 虚拟机级 TD | 云上 lift-and-shift 机密虚拟机 | [intel-tdx](./intel-tdx/README.md) |
| AMD SEV-SNP | 虚拟机级 CVM | 云上机密虚拟机、SNP attestation | [amd-sev-snp](./amd-sev-snp/README.md) |
| Arm CCA | 虚拟机级 Realm | Armv9-A 云和边缘机密 VM | [arm-cca](./arm-cca/README.md) |
| Arm TrustZone | SoC 安全世界 | 移动端、嵌入式、设备 Root of Trust | [arm-trustzone](./arm-trustzone/README.md) |
| RISC-V CoVE | 虚拟机级 TVM | RISC-V 平台机密 VM 标准化 | [riscv-cove](./riscv-cove/README.md) |
| RISC-V Keystone | 应用/运行时 enclave | 开源可定制 RISC-V TEE 研究 | [riscv-keystone](./riscv-keystone/README.md) |
| Apple PCC | 托管云 AI 节点 | Apple Intelligence 私有云推理 | [apple-pcc](./apple-pcc/README.md) |
| AWS Nitro Enclaves | 受限隔离 VM | EC2 内密钥处理、KMS 绑定解密 | [aws-nitro-enclaves](./aws-nitro-enclaves/README.md) |
| NVIDIA Confidential Computing | GPU/加速器级 | 机密 AI 推理和训练加速 | [nvidia-confidential-computing](./nvidia-confidential-computing/README.md) |
| IBM Secure Execution | 虚拟机级 | IBM Z/LinuxONE 受保护虚拟机 | [ibm-secure-execution](./ibm-secure-execution/README.md) |
| Azure Confidential VM | 云平台方案 | Azure 上 SEV-SNP/TDX 机密 VM | [azure-confidential-vm](./azure-confidential-vm/README.md) |
| Google Confidential Space | 云平台方案 | 多方数据协作、受证明容器工作负载 | [google-confidential-space](./google-confidential-space/README.md) |

### 隐私计算与可验证计算

| 技术 | 核心思想 | 资料 |
| --- | --- | --- |
| MPC | 多方在不暴露各自输入的情况下联合计算 | [privacy-mpc](./privacy-mpc/README.md) |
| FHE | 在密文上直接计算 | [privacy-fhe](./privacy-fhe/README.md) |
| ZKP | 证明陈述为真但不泄露见证 | [privacy-zkp](./privacy-zkp/README.md) |
| 差分隐私 | 用可量化噪声限制单个样本影响 | [differential-privacy](./differential-privacy/README.md) |
| 联邦学习 | 数据留在本地，交换模型更新 | [federated-learning](./federated-learning/README.md) |

### 应用专题

| 专题 | 核心问题 | 资料 |
| --- | --- | --- |
| TEE for ML / Confidential AI | 如何把 CPU/GPU TEE、远程证明、KMS、模型服务和输出治理组合起来保护 AI 推理/训练中的数据与模型权重 | [tee-for-ml](./tee-for-ml/README.md) |

## 架构图索引

| 技术 | 架构图 |
| --- | --- |
| Intel SGX | [SVG](./diagrams/intel-sgx.svg) |
| Intel TDX | [SVG](./diagrams/intel-tdx.svg) |
| AMD SEV-SNP | [SVG](./diagrams/amd-sev-snp.svg) |
| Arm CCA | [SVG](./diagrams/arm-cca.svg) |
| Arm TrustZone | [SVG](./diagrams/arm-trustzone.svg) |
| RISC-V CoVE | [SVG](./diagrams/riscv-cove.svg) |
| RISC-V Keystone | [SVG](./diagrams/riscv-keystone.svg) |
| Apple Private Cloud Compute | [SVG](./diagrams/apple-pcc.svg) |
| AWS Nitro Enclaves | [SVG](./diagrams/aws-nitro-enclaves.svg) |
| NVIDIA Confidential Computing | [SVG](./diagrams/nvidia-confidential-computing.svg) |
| IBM Secure Execution | [SVG](./diagrams/ibm-secure-execution.svg) |
| Azure Confidential VM | [SVG](./diagrams/azure-confidential-vm.svg) |
| Google Confidential Space | [SVG](./diagrams/google-confidential-space.svg) |
| 安全多方计算（MPC） | [SVG](./diagrams/privacy-mpc.svg) |
| 全同态加密（FHE） | [SVG](./diagrams/privacy-fhe.svg) |
| 零知识证明（ZKP） | [SVG](./diagrams/privacy-zkp.svg) |
| 差分隐私 | [SVG](./diagrams/differential-privacy.svg) |
| 联邦学习 | [SVG](./diagrams/federated-learning.svg) |
| TEE for ML / Confidential AI | [SVG](./diagrams/tee-for-ml.svg) |

## 总体对比

| 维度 | 应用级 TEE | VM 级 TEE | 云平台 TEE 服务 | 密码学隐私计算 |
| --- | --- | --- | --- | --- |
| 代表 | Intel SGX、Keystone | Intel TDX、AMD SEV-SNP、Arm CCA | AWS Nitro Enclaves、Azure CVM、Google Confidential Space、Apple PCC | MPC、FHE、ZKP |
| 保护对象 | 进程中的选定代码和数据 | 整个 VM 的内存和 CPU 状态 | 托管环境中的 VM、enclave 或容器 | 输入数据、计算过程或证明关系 |
| 主要信任根 | CPU/SoC、微码、TEE 运行时 | CPU/SoC、固件、机密 VM 固件 | 硬件 TEE + 云平台 attestation/key release | 密码学假设、协议参与方模型 |
| 对应用改造 | 通常较高，需拆 enclave 或 LibOS | 通常较低，可迁移整机镜像 | 中等，需适配平台密钥、网络、证明 | 较高，需重写计算或建模协议 |
| 主要优势 | TCB 小、密钥边界细 | 迁移成本低、适合通用云负载 | 运维集成好、KMS/IAM/审计完善 | 不依赖硬件供应商，安全定义清晰 |
| 主要短板 | enclave 接口复杂、侧信道面大 | TCB 大于应用级、I/O 边界复杂 | 信任和可用性绑定云平台 | 性能、工程复杂度和输出泄露控制 |

TEE for ML / Confidential AI 是一个横向应用分类，通常会把 VM 级 TEE、GPU TEE、云平台 attestation/KMS、容器编排和差分隐私等输出治理手段组合起来。它关注的不是“某一种 TEE 是否安全”，而是 AI 数据和模型权重从客户端、KMS、CPU、GPU、存储、日志到输出结果的端到端明文边界。

## 安全架构分类

从安全架构角度，可以把本仓库技术按“保护边界在哪里”分成几类：

| 类别 | 代表技术 | 安全边界 | 典型不可信对象 | 核心证明材料 |
| --- | --- | --- | --- | --- |
| 进程内 enclave | Intel SGX、RISC-V Keystone | 应用选定代码/数据 | OS、hypervisor、host 进程 | MRENCLAVE/measurement、quote |
| Confidential VM | Intel TDX、AMD SEV-SNP、Arm CCA、IBM Secure Execution | 整个 guest VM | Host OS、VMM、云管理员 | TD quote、SNP report、CCA token、protected image evidence |
| 受限子 VM/enclave | AWS Nitro Enclaves | 父实例内隔离 micro-VM | 父实例 root、代理进程 | Nitro attestation document、PCR |
| 加速器 TEE | NVIDIA Confidential Computing | GPU HBM/执行路径 + CPU CVM | Host、PCIe 路径、GPU 管理面 | CPU quote + GPU evidence |
| 托管隐私云系统 | Apple PCC、Google Confidential Space、Azure CVM | 云厂商产品化 TEE/证明/KMS | Operator、运维面、错误 workload | 透明日志、attestation token、vTPM/PCR、image digest |
| 机密 AI 架构 | TEE for ML / Confidential AI | CPU/GPU TEE 内的模型服务、权重、prompt、训练数据和 KV cache | 云 operator、模型客户管理员、GPU/host 管理路径 | CPU quote、GPU evidence、workload digest、KMS release policy |
| 密码学协议 | MPC、FHE、ZKP | 协议定义的输入/见证/密文 | 其他参与方、计算服务器、证明者 | 证明、密文、transcript、参数 |
| 统计隐私 | 差分隐私、联邦学习 + DP | 输出分布或模型更新 | 查询者、服务器、模型接收方 | epsilon/delta、privacy accountant、聚合证明 |

## 共同实现模式

多数机密计算系统最终都会落到类似流程：

```text
1. Build:   构建镜像/enclave/workload，形成 measurement 或 digest
2. Launch:  平台在 TEE 中启动 workload，并记录硬件/固件/镜像状态
3. Attest:  workload 生成带 nonce/公钥绑定的证明
4. Verify:  远程 verifier 检查平台、TCB、measurement、策略和新鲜性
5. Release: KMS/数据拥有方只向证明绑定的 workload 释放密钥或数据
6. Run:     workload 在边界内处理数据，通过共享页/I/O/输出通道交互
7. Audit:   记录证明、密钥释放、版本、输出和预算消耗
```

不同技术的差异主要体现在：

- 谁负责资源管理：OS、hypervisor、RMM/TDX Module/PSP/ultravisor、云平台。
- 私有内存如何标记：EPCM、PAMT/SEPT、RMP、GPT、PMP、Nitro 隔离。
- 证明绑定什么：代码 hash、签名者、VM firmware、kernel、容器 digest、GPU 固件、透明日志。
- 密钥由谁释放：远程 KMS、云 KMS、数据拥有方、客户端设备、HSM。
- 输出如何约束：TEE 本身通常不约束输出，需要 DP、审计、固定 workload 或业务策略。

## 关键术语对照

| 概念 | SGX | TDX | SEV-SNP | Arm CCA | Nitro Enclaves | Keystone |
| --- | --- | --- | --- | --- | --- | --- |
| 保护对象 | Enclave | Trust Domain | SNP VM | Realm VM | Enclave micro-VM | Enclave |
| 内存单位/元数据 | EPC/EPCM | PAMT/SEPT | RMP | Granule/GPT | Nitro isolation/PCR | PMP region |
| 管理组件 | CPU/SGX runtime/QE | TDX Module | AMD Secure Processor | RMM + EL3 firmware | Nitro Hypervisor | Security Monitor |
| Guest-host 接口 | ECALL/OCALL | TDCALL/SEAMCALL | GHCB | RSI/RMI | vsock | edge call |
| 初始度量 | MRENCLAVE | MRTD/RTMR | launch digest/measurement | Realm measurement | EIF PCR | enclave measurement |
| 证明 | SGX quote | TD quote | SNP report | CCA token | attestation document | monitor-signed report |

## 安全边界速查

| 技术 | 主要保护 | 不保护/弱保护 |
| --- | --- | --- |
| SGX | 小 TCB 代码和数据免受 OS/hypervisor 直接访问 | OCALL/ECALL 漏洞、侧信道、I/O、回滚 |
| TDX | 整 VM 私有内存和 vCPU 状态 | Guest 内漏洞、共享页、设备 I/O、DoS |
| SEV-SNP | VM 内存加密、寄存器状态、RMP 页归属 | Guest TCB、GHCB/共享页、平台固件补丁依赖 |
| Arm CCA | Realm 私有内存、granule ownership、Realm measurement | I/O、生态成熟度、guest 内漏洞、侧信道 |
| TrustZone | 设备 Secure world、密钥和安全外设 | 多租户 VM、TEE OS 漏洞、厂商实现差异 |
| CoVE | RISC-V confidential VM 标准化目标 | 规范/生态成熟度、具体 SoC 实现差异 |
| Keystone | 可定制 RISC-V enclave | 物理内存加密、PMP 数量、生产化供应链 |
| Apple PCC | Apple AI 请求的云侧受限处理 | 第三方自定义 workload、节点内明文计算、输出泄露 |
| Nitro Enclaves | 父实例内敏感服务隔离和 KMS 绑定 | 父实例 DoS/重放、vsock 协议漏洞、跨云可移植性 |
| NVIDIA CC | GPU HBM/执行路径和 CPU-GPU 数据流 | 模型输出泄露、多设备路径、驱动/runtime TCB |
| IBM Secure Execution | IBM Z protected Linux VM | 跨平台通用性、guest 内漏洞、运维功能限制 |
| Azure CVM | 云 VM + vTPM + Key Vault/attestation | Azure 策略配置、guest agent、数据离 VM 后边界 |
| Google Confidential Space | 受证明容器 workload 的多方数据协作 | Workload 输出泄露、policy 过宽、镜像供应链 |
| TEE for ML | AI 推理/训练中的 prompt、RAG 数据、模型权重、KV cache、梯度和 CPU/GPU 数据路径 | 输出泄露、prompt injection、模型提取、统计隐私攻击、GPU/多设备 I/O、KMS 策略错误 |
| MPC | 多方输入和中间值 | 输出推断、DoS、恶意输入、元数据 |
| FHE | 密文输入和中间计算 | 计算正确性、输出泄露、访问模式 |
| ZKP | 私有 witness 和计算正确性证明 | 公开输入、约束错误、业务真实性 |
| 差分隐私 | 输出对单个样本的可推断影响 | 原始数据访问、预算绕过、非个体秘密 |
| 联邦学习 | 原始数据本地化 | 梯度泄露、投毒、恶意服务器、最终模型泄露 |

## 常见安全边界

机密计算常见目标是把云管理员、宿主机内核、虚拟机监控器、同机租户、部分物理内存攻击排除在明文访问边界之外。但它通常不自动解决以下问题：

- 侧信道：缓存、分支预测、页错误、内存访问模式、计时、功耗等仍需要软件和平台缓解。
- I/O 信任：网络、磁盘、GPU、网卡和其他设备需要单独的加密、完整性和设备证明机制。
- 工作负载漏洞：TEE 保护错误代码“安全地运行”，但不会修复反序列化、越权、命令注入、模型泄露等应用问题。
- 输出泄露：合法计算结果可能泄露输入，尤其是统计查询、机器学习模型和低熵秘密。
- 可用性：宿主机或云平台通常仍能停止、降速、重启、拒绝调度或断开网络。
- 供应链与配置：安全属性依赖 CPU stepping、微码、固件、BIOS/UEFI、云实例规格、attestation policy 和补丁。

## 选型建议

- 需要尽量少改现有服务：优先评估 VM 级 TEE（TDX、SEV-SNP、Arm CCA）或云厂商 confidential VM。
- 需要最小化可信计算基：评估 SGX/Keystone/Nitro Enclaves，但要准备 enclave 边界设计、OCALL/系统调用审计和侧信道缓解。
- 需要保护 GPU 上的模型和数据：评估 NVIDIA Confidential Computing，并确认 CPU TEE、驱动、VBIOS、CUDA 和云实例组合在支持矩阵内。
- 需要保护 AI 推理/训练的端到端链路：阅读 [TEE for ML / Confidential AI](./tee-for-ml/README.md)，同时设计 CPU quote、GPU evidence、KMS policy、模型权重密钥、用户数据密钥和输出治理。
- 需要多方联合计算且不希望信任单一硬件/云：评估 MPC、FHE、ZKP，接受更高性能和工程成本。
- 需要发布统计结果或训练模型：差分隐私常作为 TEE、联邦学习、数据 clean room 的补充，而不是替代加密。

更细的选型问题：

- 你的主要敌手是谁：云管理员、同机租户、数据合作方、模型使用者、恶意客户端，还是外部攻击者？
- 秘密在哪里出现明文：CPU 内、GPU HBM、guest kernel、共享内存、磁盘、日志、网络、输出结果？
- 谁能写 workload：你自己、云平台、Apple/Google 固定服务、第三方容器作者？
- Verifier 能否稳定识别预期版本：measurement、image digest、签名者、TCB、透明日志是否可自动检查？
- 是否需要防输出推断：如果是，TEE 之外还要设计差分隐私、阈值、审计或固定查询。
- 是否需要跨云/跨硬件可移植：云原生 TEE 集成越深，可移植性通常越低。

## 参考入口

- Confidential Computing Consortium: https://confidentialcomputing.io/
- Intel SGX: https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html
- Intel TDX: https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html
- AMD SEV: https://www.amd.com/en/developer/sev.html
- Arm CCA: https://www.arm.com/architecture/security-features/arm-confidential-compute-architecture
- Arm TrustZone: https://www.arm.com/technologies/trustzone-for-cortex-a
- RISC-V AP-TEE/CoVE: https://github.com/riscv-non-isa/riscv-ap-tee
- Keystone: https://keystone-enclave.org/
- Apple Private Cloud Compute: https://security.apple.com/blog/private-cloud-compute/
- AWS Nitro Enclaves: https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html
- NVIDIA Trusted Computing: https://docs.nvidia.com/nvtrust/index.html
- Azure Confidential Computing: https://learn.microsoft.com/en-us/azure/confidential-computing/overview
- Google Confidential Computing: https://docs.cloud.google.com/confidential-computing/
- TEE for ML / Confidential AI 综述: https://aiwiki.ai/wiki/tee_for_ml
