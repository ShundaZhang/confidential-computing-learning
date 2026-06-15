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

## 总体对比

| 维度 | 应用级 TEE | VM 级 TEE | 云平台 TEE 服务 | 密码学隐私计算 |
| --- | --- | --- | --- | --- |
| 代表 | Intel SGX、Keystone | Intel TDX、AMD SEV-SNP、Arm CCA | AWS Nitro Enclaves、Azure CVM、Google Confidential Space、Apple PCC | MPC、FHE、ZKP |
| 保护对象 | 进程中的选定代码和数据 | 整个 VM 的内存和 CPU 状态 | 托管环境中的 VM、enclave 或容器 | 输入数据、计算过程或证明关系 |
| 主要信任根 | CPU/SoC、微码、TEE 运行时 | CPU/SoC、固件、机密 VM 固件 | 硬件 TEE + 云平台 attestation/key release | 密码学假设、协议参与方模型 |
| 对应用改造 | 通常较高，需拆 enclave 或 LibOS | 通常较低，可迁移整机镜像 | 中等，需适配平台密钥、网络、证明 | 较高，需重写计算或建模协议 |
| 主要优势 | TCB 小、密钥边界细 | 迁移成本低、适合通用云负载 | 运维集成好、KMS/IAM/审计完善 | 不依赖硬件供应商，安全定义清晰 |
| 主要短板 | enclave 接口复杂、侧信道面大 | TCB 大于应用级、I/O 边界复杂 | 信任和可用性绑定云平台 | 性能、工程复杂度和输出泄露控制 |

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
- 需要多方联合计算且不希望信任单一硬件/云：评估 MPC、FHE、ZKP，接受更高性能和工程成本。
- 需要发布统计结果或训练模型：差分隐私常作为 TEE、联邦学习、数据 clean room 的补充，而不是替代加密。

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

