# TEE for ML / Confidential AI

TEE for ML，也常被称为 Confidential AI 或 confidential inference，是把硬件可信执行环境用于机器学习训练、推理和模型托管的一类架构模式。它的目标不是发明一种新的 TEE，而是把 CPU TEE、GPU TEE、远程证明、密钥释放、模型服务运行时和输出治理组合起来，让敏感数据、模型权重和执行代码只在可证明的受保护边界内以明文出现。

本文主要参考 AI Wiki 的 [Trusted Execution Environments for machine learning](https://aiwiki.ai/wiki/tee_for_ml)（页面标注 2026-06-07 更新）并结合 NVIDIA、Apple、Google Confidential Space 和 Confidential Containers 的官方资料整理。AI Wiki 页面本身标注为 Needs citations，因此这里将其作为综述线索，而不是唯一事实来源。

## 架构图

![TEE for ML 架构图](../diagrams/tee-for-ml.svg)

## 为什么 ML 需要 TEE

传统云上 ML 服务有两个方向相反但同样棘手的秘密：

- 用户或企业侧秘密：prompt、RAG 文档、训练数据、微调数据、embedding、工具调用参数和输出结果。
- 模型提供方秘密：模型权重、system prompt、解码策略、模型路由逻辑、LoRA/adapter、商业特征工程和防滥用策略。

普通云部署中，宿主机内核、VMM、云管理员、调试工具、日志系统、GPU 管理路径或同机租户都有机会成为明文路径的一部分。密码学方案如 FHE/MPC 能减少硬件信任，但在大模型推理和训练上通常仍有显著性能与工程成本。TEE for ML 的定位是折中：在保留接近原生硬件性能的同时，把信任边界收缩到硬件、固件、被证明的 guest/workload、GPU 安全固件和密钥策略。

典型目标包括：

- 用户相信模型服务商和云平台看不到请求明文。
- 模型方相信客户或云平台管理员拿不到权重明文。
- 数据合作方只把数据交给经过证明的训练或分析代码。
- 审计方能知道某次推理、训练或评测是否运行在预期软硬件组合上。

## 核心 Primitive

TEE for ML 依赖三类底层 primitive，但在 ML 场景里它们的组合方式比普通密钥服务更复杂。

| Primitive | 在 TEE 中的含义 | ML 场景里的关键点 |
| --- | --- | --- |
| 受保护内存 | CPU/GPU 内存由硬件密钥加密或受访问控制保护，host 不能直接读写明文 | 不只保护用户输入，还要覆盖模型权重、KV cache、中间激活、梯度、optimizer state、checkpoint |
| 远程证明 | 硬件对启动状态、固件、镜像、配置和 nonce 生成可验证 evidence | Verifier/KMS 必须同时检查 CPU quote、GPU evidence、镜像 digest、TCB 版本、debug 状态和策略 |
| 安全 I/O | 数据离开 CPU 私有内存后仍通过加密链路、受保护 DMA、vsock、TEE-I/O 或应用层加密保护 | CPU 到 GPU、GPU 到 GPU、磁盘、网络、对象存储、日志、metrics 都可能重新打开明文路径 |

这三个 primitive 必须形成闭环。只保护 CPU 内存但 GPU 路径明文，或只证明 GPU 但 guest 镜像不可控，都不能构成端到端机密 AI。

## 威胁模型

TEE for ML 通常希望排除在明文边界之外的对象：

- 云平台 operator、宿主机 root、VMM、host kernel。
- 同机其他租户、普通管理代理和调试工具。
- 父 VM 或部署项目管理员，取决于 Nitro Enclaves、Confidential Space 等平台模型。
- GPU 设备路径上的未授权观察者、普通 PCIe/DMA 路径和不受信任驱动面。
- 模型客户侧基础设施管理员，适用于模型方保护权重的场景。

通常仍然信任：

- CPU/SoC/GPU 硬件根、微码、固件、TEE 管理组件。
- 厂商 attestation PKI、证书撤销和 TCB 状态发布。
- Guest OS、驱动、CUDA/ROCm/runtime、模型服务进程和容器镜像。
- Verifier、KMS、CI/CD 签名策略和供应链元数据。

通常不自动解决：

- Workload 自己的漏洞，例如反序列化、路径穿越、越权 API、prompt injection、日志泄露。
- 合法输出造成的统计泄露，例如训练数据抽取、membership inference、模型反演、模型提取。
- DoS、资源降速、粗粒度元数据、流量大小和时序侧信道。
- 供应链配置错误，例如镜像 tag 可变、debug 版本进生产、KMS policy 过宽。

## CPU TEE 在 ML 中的角色

CPU TEE 负责承载模型服务的控制面和密钥边界。不同粒度适合不同形态：

| 技术类型 | 代表 | 适合的 ML 用法 | 主要限制 |
| --- | --- | --- | --- |
| 应用级 enclave | Intel SGX、Keystone | 小型模型服务、密钥 unwrap、模型加载器、敏感后处理 | 内存容量和系统调用边界复杂；对大模型训练/推理不友好 |
| VM 级 TEE | Intel TDX、AMD SEV-SNP、Arm CCA、IBM Secure Execution | lift-and-shift 模型服务、容器运行时、GPU 驱动、Kubernetes PodVM | Guest TCB 较大；I/O、共享页、设备路径需要额外保护 |
| 受限子 VM/enclave | AWS Nitro Enclaves | KMS 代理、签名服务、模型密钥服务、敏感 sidecar | 无网络/无持久磁盘；通常不直接承载 GPU 大模型 |
| 托管 confidential workload | Google Confidential Space、Azure CVM、Apple PCC | 多方数据协作、固定 AI 推理服务、云厂商 KMS/IAM 集成 | 平台绑定更强；策略、透明日志和云控制面成为关键 |

对于大模型，主流趋势是 VM 级 TEE + GPU TEE。SGX 这类应用级 enclave 仍适合做“小 TCB 密钥闸门”，但很难独自承载完整 LLM 推理栈。

## GPU TEE 与加速器边界

ML 的特殊性在于明文最终会进入加速器。没有 GPU 侧保护时，CPU TEE 只能保护模型到达 GPU 之前的路径，GPU HBM、command buffer、kernel、DMA 和管理面仍可能暴露。

NVIDIA Hopper H100/H200 把 confidential computing 扩展到 GPU 侧，核心思路包括 GPU 安全启动、固件度量、CC mode、GPU attestation、受保护 HBM，以及 CPU confidential VM 到 GPU 之间的受保护数据路径。Blackwell 一代进一步强调 TEE-I/O 和多 GPU/NVLink/NVSwitch 场景下的链路保护。具体支持能力取决于 GPU 型号、云实例、VBIOS、驱动、CUDA、CC mode 和平台组合。

GPU TEE 在策略上至少要回答：

- GPU 是否是真实受支持设备，而不是被模拟或降级的设备。
- GPU 是否处于 CC mode，debug、profiling、性能计数器等能力是否按策略关闭或限制。
- GPU firmware/VBIOS/GSP/安全处理器版本是否在允许列表。
- CPU confidential VM 与 GPU evidence 是否属于同一个会话和同一台 workload。
- CPU-GPU、GPU-GPU、GPU-存储或 GPU-NIC 路径是否仍会出现未加密明文。

## 典型 Confidential Inference 流程

一个端到端 confidential inference 系统通常由四个角色组成：

| 角色 | 职责 |
| --- | --- |
| 用户/数据拥有方 | 加密 prompt、RAG 数据或任务输入；验证服务证明；接收输出 |
| 模型拥有方 | 加密发布模型权重；定义允许的 runtime 和版本策略 |
| Confidential runtime | 在 CVM/TEE 内启动模型服务、解密权重、处理请求、约束输出路径 |
| Verifier/KMS | 验证 CPU/GPU/workload evidence，按策略释放数据密钥或模型密钥 |

典型流程：

```text
1. 构建镜像：
   固化 guest image、container digest、模型加载器、驱动/runtime 版本和 release 策略。

2. 启动环境：
   云平台启动 TDX/SEV-SNP/CCA 等 confidential VM，并把 GPU 置于支持的 CC 模式。

3. 收集证明：
   guest 内 attestation agent 获取 CPU quote/report、vTPM PCR、workload digest、
   GPU attestation evidence，并绑定 verifier nonce 或临时会话公钥。

4. 验证策略：
   verifier 检查厂商证书链、TCB 版本、撤销状态、measurement、镜像 digest、
   GPU 固件、debug 状态、实例/区域/项目属性和 nonce 新鲜性。

5. 释放密钥：
   KMS 或模型拥有方只向合格 workload 释放模型权重密钥、数据密钥或 API 会话密钥。

6. 执行推理：
   模型权重、prompt、RAG 数据和 KV cache 只在 CPU/GPU 受保护边界内以明文出现。

7. 返回输出：
   输出经应用层加密、访问控制、审计、速率限制和必要的隐私策略后返回调用方。
```

这个流程的关键不是“有一个 TEE”，而是证明、密钥和数据流必须绑定到同一个被允许的软硬件组合。

## 典型系统与落地形态

| 系统/方案 | 主要形态 | TEE for ML 关注点 |
| --- | --- | --- |
| Apple Private Cloud Compute | Apple 自建云 AI 节点 | 设备只向可证明且在透明日志中的 PCC 软件发送请求；强调 stateless、无特权运行时访问、非定向化和可验证透明度 |
| Anthropic Confidential Inference | Confidential VM + NVIDIA CC GPU 的模型服务设计 | 通过 attested model loader 和受控 release pipeline 保护用户请求与模型权重，面向高价值模型推理 |
| Azure Confidential GPU/CVM | 云上 SEV-SNP/TDX + H100/H200 等组合 | 云 KMS、guest attestation、GPU CC 与企业模型服务集成 |
| Google Confidential Space | Confidential VM + hardened OS + 容器 workload | 数据拥有方根据 attestation claims 只向约定容器释放敏感数据，适合多方数据协作和 ML 分析 |
| AWS Nitro Enclaves | EC2 父实例中的隔离 microVM | 常用于 KMS 绑定解密、签名和密钥代理；可作为 AI 系统中的密钥隔离组件 |
| Fortanix / Anjuna / Edgeless / CoCo | 商业或开源编排层 | 把 attestation、KMS、容器镜像解密、mTLS、Kubernetes 和 TEE 运行时打包成可运营平台 |

这些系统的共同模式是“先证明，后放密钥”。差异在于证明材料由谁产生、策略由谁管理、代码是否开放可审计、以及用户是否能自行验证。

## 应用场景

### 机密推理

用户把 prompt 和上下文发送给运行在 TEE 中的模型服务。服务端 operator 不能直接读取请求、输出或中间状态。适合企业知识库问答、医疗/金融助手、法律文档分析、个人数据代理和需要防云侧明文访问的 API。

关键设计点：

- 客户端或网关应验证 attestation 后再建立会话密钥。
- RAG 文档、embedding、工具调用参数和输出都要纳入数据分类。
- 只保护 runtime 不够，还要避免日志、trace、metrics、错误栈和缓存落明文。

### 模型 IP 保护

模型方把权重以密文形式交给客户云或第三方平台。只有当硬件、guest、GPU、模型加载器和 release policy 都通过证明时，KMS 才释放权重密钥。这样可以降低客户管理员、云管理员或恶意运维直接复制权重的风险。

关键设计点：

- 权重解密和加载逻辑应尽量小，最好独立于复杂业务 API。
- 权重密钥应绑定 measurement、GPU evidence、版本和用途。
- checkpoint、LoRA、adapter、tokenizer、system prompt 也可能是 IP。

### 机密训练与联合训练

多个数据方把加密数据提供给约定训练 workload。TEE 证明训练代码、镜像和环境后释放数据密钥，训练过程在 confidential VM/GPU 内执行，最终只输出模型、梯度、指标或聚合结果。

关键设计点：

- 训练 recipe、数据访问范围、输出对象和 checkpoint 策略必须提前固定。
- 训练结果可能记忆原始样本，TEE 需要和差分隐私、评估审计、输出阈值结合。
- 多方数据协作还要考虑参与方之间的授权、撤回、审计和法律约束。

### 计算来源与治理

TEE attestation 能为“某个模型输出来自哪个镜像、哪个硬件 TCB、哪个模型版本和哪个 release policy”提供证据。它不能证明模型结论正确，但能把运行环境纳入审计链。

适合：

- AI agent 调用敏感工具前校验 runtime。
- 监管或客户审计模型服务版本。
- 模型评测平台证明评测脚本未被 operator 篡改。
- 数据 clean room 证明只运行约定查询。

## 安全边界与限制

TEE for ML 的边界应按数据生命周期逐段检查：

| 阶段 | 应保护内容 | 常见风险 |
| --- | --- | --- |
| 构建 | 镜像、模型加载器、驱动版本、签名、SBOM | 构建机被污染、tag 漂移、debug 镜像进入生产 |
| 启动 | TCB 版本、firmware、PCR/measurement、GPU CC mode | 测量不完整、策略只验 CPU 不验 GPU、nonce 未绑定 |
| 密钥释放 | 模型密钥、数据密钥、会话密钥 | KMS policy 过宽、证书撤销未查、密钥复用 |
| 数据加载 | prompt、RAG、训练数据、权重、tokenizer | 明文临时文件、对象存储缓存、日志泄露 |
| 推理/训练 | KV cache、activation、gradient、optimizer state | GPU 路径不受保护、多租户资源侧信道、runtime 漏洞 |
| 输出 | 文本、embedding、模型、指标、checkpoint | 成员推断、训练数据抽取、模型提取、授权输出变成外传通道 |

重要限制：

- TEE 不会自动阻止模型通过正常 API 泄露训练数据或商业秘密。
- TEE 不修复不安全的模型服务代码，也不能抵消 prompt injection 对工具调用的影响。
- 侧信道仍然存在，尤其是共享缓存、内存访问模式、页错误、GPU 资源争用和时序。
- 厂商 PKI、固件更新、TCB 状态和撤销机制成为供应链信任的一部分。
- Host 仍能拒绝服务、降低性能、观察粗粒度元数据或影响调度。
- 多 GPU、RDMA、NVLink/NVSwitch、GPUDirect Storage 会扩大 I/O 证明边界。

## 与 FHE、MPC、联邦学习、差分隐私的关系

| 技术 | 保护方式 | 优势 | 在 ML 中的短板 | 与 TEE 的组合 |
| --- | --- | --- | --- | --- |
| TEE for ML | 明文只在受证明硬件边界内出现 | 性能接近原生，能复用现有模型栈 | 信任硬件和厂商 PKI，侧信道和输出泄露仍需治理 | 作为实际推理/训练运行时 |
| FHE | 在密文上计算 | 不需要信任硬件运行方 | 大模型训练/推理成本高，算子和模型结构受限 | 用于小型敏感子计算或结果验证 |
| MPC | 多方秘密分享联合计算 | 不信任单一硬件或云 | 通信和交互成本高，恶意安全实现复杂 | TEE 可作为加速器或参与方隔离层 |
| 联邦学习 | 数据留在本地，上传模型更新 | 原始数据不集中 | 梯度泄露、投毒、恶意聚合方 | TEE 保护聚合器、训练 coordinator 或客户端证明 |
| 差分隐私 | 限制单样本对输出分布的影响 | 直接处理统计输出泄露 | 需要隐私预算和精度权衡 | TEE 保护原始训练过程，DP 控制最终模型/统计输出 |

生产系统常常是组合式的：TEE 保护运行时明文，DP 控制输出泄露，MPC/FHE 用在跨组织或强不信任场景，联邦学习减少数据集中化。

## 工程检查清单

- 明确敌手：云 operator、模型客户、模型服务商、数据合作方、同租户还是外部攻击者。
- 列出所有明文点：CPU 内存、GPU HBM、共享页、临时文件、日志、trace、metrics、checkpoint、输出 sink。
- 证明绑定完整：CPU quote、GPU evidence、vTPM PCR、container digest、模型加载器 hash、驱动/runtime 版本和 nonce。
- 密钥分层：模型权重密钥、用户数据密钥、会话密钥、日志/审计密钥分开管理。
- 策略精确：使用 immutable digest 和 allowlist，避免只按镜像 tag、实例类型或项目名放行。
- 默认 fail closed：证明失败、TCB 过期、证书撤销、GPU 非 CC mode、debug 开启时拒绝释放密钥。
- 输出治理：限制输出路径、做审计、设置速率限制，对统计结果考虑 DP 或阈值。
- 供应链闭环：CI 签名、SBOM、reproducible build、透明日志或 release manifest。
- 运维最小化：禁用 core dump、明文日志、调试 shell、profiling、非必要网络出口。
- 回滚治理：明确 measurement 版本生命周期，防止旧漏洞镜像重新获得密钥。

## 选型建议

- 只需要保护模型服务中的密钥或小段逻辑：优先评估 SGX、Nitro Enclaves 或小 TCB sidecar。
- 需要迁移完整推理服务：优先评估 TDX、SEV-SNP、Arm CCA 或云上 Confidential VM。
- 需要保护 GPU 中的模型权重和用户输入：必须评估 GPU CC、CPU-GPU 路径和 GPU evidence，而不是只看 CPU TEE。
- 多方数据协作或 clean room：优先看 Confidential Space、Confidential Containers、MPC/TEE 混合方案。
- 最担心模型输出泄露训练数据：TEE 只是底座，还要加入差分隐私、访问控制和输出审计。

## 参考资料

- AI Wiki: Trusted Execution Environments for machine learning: https://aiwiki.ai/wiki/tee_for_ml
- Confidential Computing Consortium: A Technical Analysis of Confidential Computing: https://confidentialcomputing.io/
- NVIDIA: Confidential Computing on H100 GPUs for Secure and Trustworthy AI: https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/
- Apple Security Research: Private Cloud Compute: https://security.apple.com/blog/private-cloud-compute/
- Google Cloud: Confidential Space overview: https://docs.cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview
- Confidential Containers: Overview and architecture: https://confidentialcontainers.org/docs/overview/
- Confidential Containers: Confidential AI use case: https://confidentialcontainers.org/docs/use-cases/confidential-ai/
