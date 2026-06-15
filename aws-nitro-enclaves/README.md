# AWS Nitro Enclaves

AWS Nitro Enclaves 是 Amazon EC2 的 enclave 功能。它从一台父 EC2 实例中切分 vCPU 和内存，创建一个隔离、受约束的 enclave VM。Enclave 没有持久化存储、没有交互式 shell、没有外部网络，只能通过 vsock 与父实例通信。

## 核心概念

- Parent instance：启动和管理 enclave 的普通 EC2 实例。
- Enclave：从父实例切分出的隔离 VM，用于处理敏感数据。
- Nitro Hypervisor：提供父实例与 enclave 的 CPU/内存隔离。
- vsock：父实例与 enclave 的本地通信通道。
- Enclave Image File（EIF）：enclave 镜像格式，其度量进入 attestation document。
- Attestation document：Nitro 为 enclave 身份、镜像度量和公钥生成的签名证明。
- KMS condition：AWS KMS 可根据 attestation 只向特定 enclave 释放密钥。

## 工作原理

Nitro Enclave 的设计强调“强约束”。父实例负责拉取数据、转发网络请求、写入日志或访问 KMS；enclave 只运行敏感计算。父实例 root 用户不能 SSH 到 enclave，也不能读取 enclave 内存，但父实例仍控制输入输出和生命周期。

典型密钥处理流程：

1. 开发者构建 enclave 镜像，生成 EIF measurement。
2. 父实例启动 enclave，分配 vCPU 和内存。
3. Enclave 内服务生成临时密钥或请求。
4. Enclave 获取 Nitro attestation document。
5. AWS KMS 验证 attestation 条件，向 enclave 公钥加密返回数据密钥。
6. Enclave 解密数据并完成敏感处理。

这种模式常用于把 TLS 私钥、数据库解密密钥、PII 处理逻辑、签名服务放入更窄的边界。

## 安全模型

Nitro Enclaves 通常信任：

- AWS Nitro System、Nitro Hypervisor 和 attestation 根。
- Enclave 镜像、运行时和应用。
- KMS policy、IAM policy 和 attestation condition。

Nitro Enclaves 通常不信任：

- 父实例中的普通进程、root 用户和应用层攻击者。
- 外部网络、父实例代理、日志和存储。
- 同一父实例中 enclave 外的代码。

## 安全边界与限制

- Enclave 无直接网络和存储是安全优势，也是工程约束。所有 I/O 都要通过父实例代理。
- 父实例可拒绝服务、重放旧输入、篡改外部响应或隐瞒网络状态，因此协议必须认证、加密和防重放。
- Enclave 不能修复应用漏洞。若敏感服务本身可被远程利用，攻击者仍可能在 enclave 内执行代码。
- Attestation policy 是核心控制点。KMS 条件应绑定 PCR/measurement、镜像版本和业务上下文。
- Nitro Enclaves 是 AWS 专有平台能力，不是跨云可移植 TEE 标准。

## 适用场景

Nitro Enclaves 适合密钥使用、签名、证书私钥保护、PII tokenization、隐私数据转换和小型高价值服务。若需要迁移整台 VM，应评估 AWS 上的 confidential VM 能力或其他云 TDX/SEV-SNP；若需要 GPU 机密计算，应看 NVIDIA CC 与云实例组合。

## 参考资料

- AWS Nitro Enclaves overview: https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html
- AWS Nitro System security design: https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/the-aws-nitro-system.html
- Nitro Enclaves KMS integration: https://docs.aws.amazon.com/enclaves/latest/user/kms.html

