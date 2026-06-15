# Intel TDX

Intel Trust Domain Extensions（TDX）是 Intel 面向云和虚拟化环境的 VM 级机密计算技术。它把一台虚拟机运行在 Trust Domain（TD）中，目标是在不大改应用的前提下，保护 VM 内存和 CPU 状态免受宿主机 VMM、hypervisor、host OS、云管理员和部分物理内存攻击的读取或篡改。

## 核心概念

- TD（Trust Domain）：受 TDX 保护的机密虚拟机。
- TDX Module：由 CPU 度量的软件模块，运行在 SEAM（Secure Arbitration Mode）中，管理 TD 生命周期。
- SEAMRR：为 TDX Module 保留的内存范围。
- Private/Shared GPA：TD 内存分为私有页和共享页，共享页用于与不可信 VMM 或设备交互。
- Secure EPT/PAMT：用于绑定、跟踪和保护 TD 私有页的页表与元数据。
- TDREPORT/Quote：TD 生成本地报告，再由 quote 机制转换为远程可验证证明。

## 工作原理

TDX 复用现有虚拟化模型，但把最敏感的 VM 状态交给 CPU 和 TDX Module 管理。普通 VMM 仍负责调度 vCPU、分配资源、处理 I/O 和设备模拟，但不能直接访问 TD 私有内存，也不能任意修改 TD CPU 状态。

TDX 的内存保护依赖多层机制：

- Intel TME-MK 为不同安全域提供内存加密密钥。
- TD 私有页用 TD 绑定的密钥加密。
- GPA shared bit 标识某个 guest physical address 是私有通信还是共享通信。
- Secure EPT 和 PAMT 防止 VMM 把物理页错误映射、重复映射或伪装成 TD 私有页。
- TD exit/entry 过程中，CPU 状态由 TDX Module 保存和恢复，减少对 VMM 的暴露。

启动时，TDVF（TD Virtual Firmware）和初始内存会被度量。运行中，RTMR 等可扩展寄存器可记录后续启动链、内核、initrd、策略等状态。远程方应把密钥释放绑定到这些度量，而不是只检查“是否是 TDX”。

## 远程证明

TD 内部先生成 TDREPORT，报告包含 TD 度量、配置、安全属性和调用方提供的数据。随后通过平台 quote 服务生成可远程验证的 quote。Verifier 需要检查：

- CPU 和 TDX Module 的身份、版本和 TCB 状态。
- TD 度量、RTMR、TD attributes 是否符合预期。
- Quote 中绑定的 nonce 或临时公钥是否匹配当前会话。
- 云平台和证书链是否满足自己的信任策略。

只有证明通过后，KMS、secret manager 或数据提供方才应释放工作负载密钥。

## 安全模型

TDX 通常信任：

- Intel CPU、微码、TDX Module、平台固件和 attestation 根。
- TDVF、guest kernel、guest userspace 和工作负载自身。
- Verifier 侧的 attestation policy。

TDX 通常不信任：

- VMM、host OS、host 管理员。
- 云平台普通运维面和同机租户。
- 共享内存、虚拟设备、外部网络与存储。

## 安全边界与限制

- TDX 主要保护 TD 私有内存和 CPU 状态，不保证所有设备 I/O 都可信。
- Host 仍控制调度和资源，因此可实施拒绝服务、降速、中断风暴或异常注入类攻击面。
- 侧信道仍需缓解，包括缓存、分支预测、内存访问模式、计时和共享资源争用。
- VM 级 TCB 大于应用级 enclave。Guest OS、驱动和服务漏洞仍在信任边界内。
- 共享页是必要通信机制，必须按不可信输入处理，避免把秘密放入共享缓冲区。
- 证明策略必须包含版本和补丁状态，否则“真实 TDX 平台”不等于“足够安全的平台”。

## 适用场景

TDX 适合把现有 Linux/Windows 服务迁移到机密 VM，尤其是数据库、AI 推理、企业应用和合规型云迁移。若需要更小 TCB，可把关键逻辑拆到 SGX/Nitro Enclave；若需要跨组织联合计算，可组合 TDX 与 Confidential Space、MPC 或差分隐私。

## 参考资料

- Intel TDX overview: https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html
- Intel Trust Authority: https://www.intel.com/content/www/us/en/security/trust-authority.html
- Google Confidential VM TDX overview: https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview

