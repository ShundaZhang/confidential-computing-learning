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

## 硬件与内存机制

SGX 的关键内存区域是 PRM（Processor Reserved Memory），其中包含 EPC（Enclave Page Cache）。EPC 以页为单位存放 enclave 代码和数据，CPU 通过 EPCM（EPC Metadata）记录每个 EPC 页的归属、权限、线性地址绑定和状态。

```text
Untrusted process virtual address space
  +---------------------------------------------------+
  | normal heap/stack/code                            |
  | enclave virtual range -> EPC pages in PRM         |
  +---------------------------------------------------+

CPU checks: enclave identity + page permission + EPCM metadata
Memory controller: encrypt/integrity-protect EPC traffic outside package
```

EPCM 的作用类似“页级安全账本”。即使 OS 控制页表，也不能把某个 EPC 页映射给错误 enclave 或普通进程访问。OS 可以调度、分页、触发异常，但访问检查由 CPU 做最终裁决。

SGX1 要求 enclave 初始内存基本在创建时确定；SGX2 引入动态内存管理能力，例如运行时增加、修改和删除 enclave 页。不过动态页仍需要 enclave 自己接受和维护状态，不能把 OS 提供的页当成可信。

## Enclave 生命周期细节

一个典型 SGX enclave 构建过程可以展开为：

1. **ECREATE**：创建 SECS（SGX Enclave Control Structure），定义 enclave 基本属性。
2. **EADD**：把代码/数据页加入 EPC。
3. **EEXTEND**：把页内容扩展进 measurement。
4. **EINIT**：校验签名结构 SIGSTRUCT，完成初始化并固定 MRENCLAVE。
5. **EENTER/ERESUME**：进入 enclave 执行。
6. **EEXIT/AEX**：显式退出或异步异常退出，回到不可信运行时。
7. **EREMOVE**：销毁 EPC 页，释放 enclave 资源。

MRENCLAVE 度量初始内容，MRSIGNER 度量签名者。生产策略常在两者间取舍：MRENCLAVE 精确锁定某个构建版本；MRSIGNER 允许同一发布者升级，但必须结合 ISVSVN 等安全版本号防回滚。

## ECALL/OCALL 与边界攻击面

SGX 的最大工程难点是 enclave 边界。ECALL 把外部输入带入 enclave；OCALL 让 enclave 请求外部服务。所有跨边界数据都应做深拷贝、长度检查、指针范围检查和 TOCTOU 防护。

常见风险包括：

- 外部传入指针指向 enclave 内部或重叠区域，导致 confused deputy。
- Enclave 检查长度后，OS 在使用前修改共享缓冲区。
- OCALL 返回结构体字段不一致，触发越界或逻辑绕过。
- Enclave 把秘密写入未清零的外部 buffer。
- 异常路径、panic、日志、崩溃转储泄露敏感数据。

LibOS（如 Gramine）可以降低移植成本，但会扩大 enclave 内 TCB。手写 SDK enclave 可以更小，但接口审计压力更大。

## 远程证明与密钥

SGX 远程证明的核心是让 verifier 确认：

- 代码度量是否匹配预期 MRENCLAVE 或 MRSIGNER 策略。
- 平台是否是真实 Intel SGX 平台。
- TCB、微码、安全版本号是否满足策略。
- 报告中绑定的临时公钥是否由 enclave 控制。

早期 SGX 常见 EPID 证明；云和数据中心场景更多使用 DCAP，由平台生成 quote，verifier 根据 Intel PCK 证书链和 TCB collateral 离线验证。密钥释放应绑定 quote 中的 nonce、公钥、度量和安全版本，避免重放。

证明链路可以这样理解：

```text
Target enclave
  -> local REPORT for Quoting Enclave
  -> Quoting Enclave signs/produces QUOTE
  -> remote verifier checks PCK cert chain, TCB info, QE identity
  -> verifier validates MRENCLAVE/MRSIGNER/ISVSVN/report_data
  -> verifier releases secret to enclave-bound public key
```

SGX report data 通常放入 nonce 或临时公钥 hash。否则攻击者可以把旧 quote 拿来重放，或者把密钥释放到 quote 之外的通道。

## Sealing 与持久化状态

Sealing key 允许 enclave 把状态加密写到不可信磁盘。常见策略：

- **MRENCLAVE sealing**：只有同一 enclave measurement 能解密，升级困难但边界精确。
- **MRSIGNER sealing**：同一签名者的后续版本可解密，适合升级，但要结合 ISVSVN 防止旧版本回滚。

Sealing 不自动提供 freshness。OS 可以回滚旧密文。需要单调计数器、远端状态、ledger 或业务层版本协议来防回滚。

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
- OS 可控制页调度、中断、线程和时间源，因此 page-fault side channel、controlled-channel attack 和计时攻击需要专门缓解。
- Enclave 初始镜像不能含明文长期秘密。通常先 attestation，再由远程 KMS 注入秘密。
- Quoting/Provisioning 相关 enclave、PCK 证书、TCB collateral 都是证明生态的一部分，验证器要检查版本和撤销状态。
- SGX 不保证可用性。恶意 OS 可以不调度 enclave、频繁 AEX 或拒绝 EPC。

## 与 VM 级 TEE 的对比

| 维度 | SGX | TDX/SEV-SNP/CCA |
| --- | --- | --- |
| 粒度 | 进程内 enclave | 整个 VM |
| TCB | 可做到很小 | 包含 guest OS 和驱动 |
| 应用改造 | 高，需要拆边界或 LibOS | 低，可迁移现有 VM |
| I/O | 全部出 enclave，经 OCALL/不可信 runtime | guest OS 可直接处理，但设备仍不可信 |
| 证明对象 | MRENCLAVE/MRSIGNER | VM firmware/kernel/config/runtime measurements |
| 主要痛点 | 接口审计、EPC、侧信道 | I/O、guest TCB、云平台 attestation |

## 适用场景

SGX 适合密钥管理、加密服务、隐私查询、小型高价值逻辑、LibOS 包装的已有服务。若目标是迁移整台 VM，TDX/SEV-SNP 通常更合适；若目标是多方不信任计算且不能接受硬件信任根，应考虑 MPC/FHE/ZKP。

## 参考资料

- Intel SGX overview: https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html
- Intel SGX attestation: https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/attestation-services.html
- Gramine: https://gramineproject.io/
