# NVIDIA Confidential Computing

NVIDIA Confidential Computing 把机密计算边界扩展到 GPU，主要面向 AI/ML 加速负载。传统 CPU TEE 可以保护 guest CPU 内存，但当模型和数据进入 GPU HBM、驱动和 PCIe 传输路径时，安全边界会被拉宽。NVIDIA Hopper H100 及后续平台提供 GPU 侧安全启动、内存保护、链路加密和 attestation 能力，用于保护敏感模型和数据在 GPU 上的使用中状态。

## 核心概念

- GPU CC mode：GPU 进入机密计算模式，限制调试和管理访问。
- Secure boot/firmware measurement：GPU 固件和启动状态可被度量。
- Attestation：验证 GPU 身份、固件版本、CC 模式和安全属性。
- Encrypted link：CPU TEE 与 GPU 之间的数据传输可加密保护。
- Protected HBM：GPU 本地显存中的模型权重、中间张量和数据受保护。
- nvtrust：NVIDIA 提供的证明、验证和部署工具集合。

## 工作原理

GPU 机密计算通常不是单独存在，而是与 CPU 侧 confidential VM 组合使用。一个典型部署会使用 AMD SEV-SNP 或 Intel TDX 保护 VM，随后把 NVIDIA GPU 置于 CC 模式，并让工作负载验证 CPU TEE 与 GPU attestation 都满足策略。

关键路径包括：

1. VM 启动并通过 CPU TEE attestation。
2. GPU 固件安全启动，进入 CC 模式。
3. 工作负载验证 GPU attestation evidence。
4. CPU TEE 与 GPU 建立受保护通道。
5. 模型和数据以加密形式穿过不可信 host 路径。
6. GPU 在受保护 HBM 中执行推理或训练。

对 AI 场景而言，保护对象不仅是用户输入，还包括模型权重、prompt、中间激活、embedding、梯度和推理结果。

## 安全模型

NVIDIA GPU CC 通常信任：

- 支持 CC 的 NVIDIA GPU、VBIOS、固件和硬件根。
- CPU 侧 TEE（TDX/SEV-SNP 等）和 guest OS。
- NVIDIA 驱动、CUDA 版本、nvtrust 工具和证明策略。

通常不信任：

- Host hypervisor、host OS、云管理员。
- 普通 PCIe 路径上的观察者。
- 同机其他租户或非授权管理工具。

## 安全边界与限制

- 需要检查支持矩阵。GPU 型号、VBIOS、驱动、CUDA、云实例和 CC 模式必须匹配。
- CPU TEE 与 GPU TEE 都要证明；只证明其中一个通常不足以保护端到端数据流。
- GPU 侧机密计算不自动防止模型通过输出泄露训练数据或 prompt。
- 侧信道、性能计数器、资源争用和多租户 MIG 的边界需要结合文档和云平台配置评估。
- 驱动和运行时仍是大 TCB 的一部分，供应链和版本管理很重要。

## 适用场景

NVIDIA Confidential Computing 适合机密 AI 推理、专有模型托管、医疗/金融数据 GPU 分析、多方模型评估和需要保护 GPU 中间状态的任务。若只处理 CPU 数据，可先评估 TDX/SEV-SNP；若要在不信任任何硬件供应商的前提下计算，应研究 FHE/MPC，但当前性能差异显著。

## 参考资料

- NVIDIA Trusted Computing Solutions: https://docs.nvidia.com/nvtrust/index.html
- NVIDIA H100 architecture whitepaper: https://resources.nvidia.com/en-us-tensor-core
- Google Confidential VM with NVIDIA CC: https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview

