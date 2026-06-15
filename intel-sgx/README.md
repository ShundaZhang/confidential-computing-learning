# Intel SGX

Intel Software Guard Extensions（SGX）是一组 CPU 指令和平台机制，用于在普通进程内部创建受硬件保护的 enclave。SGX 的目标不是保护整台虚拟机，而是把敏感代码和数据切成更小的可信区域，使操作系统、hypervisor、系统管理员和同机进程即使拥有更高权限，也不能直接读取或修改 enclave 内存。

## 核心概念

- Enclave：进程地址空间中的受保护区域，包含可信代码、栈、堆和密钥材料。
- EPC（Enclave Page Cache）：存放 enclave 页面的受保护物理内存区域，位于 PRM（Processor Reserved Memory）内。
- MRENCLAVE：enclave 初始代码、数据和配置的度量值。
- MRSIGNER：签名者身份，用于表达“由哪个发布者签名”的信任。
- ECALL/OCALL：非可信世界进入 enclave、enclave 调用外部不可信服务的接口。
- Quote/Attestation：远程方验证 enclave 身份、平台 TCB 和安全补丁状态的证明。

## 工作原理

SGX 由 CPU 负责访问控制。EPC 页面只能由所属 enclave 在正确上下文中访问；其他软件即使运行在 ring 0、hypervisor 或 SMM，也不能直接读取明文。内存离开 CPU 封装时，由内存加密和完整性机制保护，防止 DRAM 上的被动窥探和部分篡改。

Enclave 生命周期通常包括：

1. 非可信应用创建 enclave，加载代码和初始数据。
2. CPU 对加载内容进行度量，形成 MRENCLAVE。
3. 应用通过 EENTER/ECALL 进入 enclave。
4. Enclave 内部处理敏感逻辑，必要时通过 OCALL 请求外部 I/O。
5. 远程服务通过 attestation 验证 enclave，再释放密钥或数据。
6. Enclave 可使用 sealing key 把本地状态加密持久化。

SGX 运行时的难点在于边界设计。系统调用、文件、网络、时间、线程调度都在 enclave 外部，必须视为不可信输入。常见工程方式包括手写 enclave 接口、使用 Intel SGX SDK，或通过 Gramine 等 LibOS 让未修改应用运行在 enclave 中。

## 远程证明与密钥

SGX 远程证明的核心是让 verifier 确认：

- 代码度量是否匹配预期 MRENCLAVE 或 MRSIGNER 策略。
- 平台是否是真实 Intel SGX 平台。
- TCB、微码、安全版本号是否满足策略。
- 报告中绑定的临时公钥是否由 enclave 控制。

早期 SGX 常见 EPID 证明；云和数据中心场景更多使用 DCAP，由平台生成 quote，verifier 根据 Intel PCK 证书链和 TCB collateral 离线验证。密钥释放应绑定 quote 中的 nonce、公钥、度量和安全版本，避免重放。

## 安全模型

SGX 通常信任：

- Intel CPU 封装、微码和 SGX 实现。
- Intel attestation 根、PCK/TCB collateral 或对应验证服务。
- Enclave 内部代码、可信运行时和依赖库。
- 正确配置的 BIOS/UEFI、平台固件和补丁链。

SGX 通常不信任：

- 宿主操作系统、hypervisor、系统管理员。
- Enclave 外部内存、文件系统、网络、时钟和系统调用返回值。
- 同机其他进程和其他租户。

## 安全边界与限制

- 侧信道不是默认解决项。缓存、页错误、分支预测、瞬态执行、计时和内存访问模式都可能泄露秘密。
- Enclave 接口是主要攻击面。OCALL/ECALL 参数、指针检查、序列化和异常路径需要严格审计。
- EPC 容量有限，EPC paging 会带来显著性能开销，也可能扩大侧信道。
- 持久化需要额外处理回滚。Sealing 只保护机密性和绑定关系，不自动提供全局单调状态。
- I/O 不在 enclave 内。网络和磁盘必须使用端到端加密、完整性校验和协议级认证。
- Enclave 内漏洞仍然是漏洞。SGX 不能防止可信代码把秘密主动泄露出去。

## 适用场景

SGX 适合密钥管理、加密服务、隐私查询、小型高价值逻辑、LibOS 包装的已有服务。若目标是迁移整台 VM，TDX/SEV-SNP 通常更合适；若目标是多方不信任计算且不能接受硬件信任根，应考虑 MPC/FHE/ZKP。

## 参考资料

- Intel SGX overview: https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html
- Intel SGX attestation: https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/attestation-services.html
- Gramine: https://gramineproject.io/

