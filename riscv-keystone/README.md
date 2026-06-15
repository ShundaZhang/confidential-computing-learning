# RISC-V Keystone

Keystone 是一个开源的 RISC-V TEE 框架，用于构建可定制的 enclave。它更偏研究和系统构建框架，而不是单一厂商固定产品。Keystone 的目标是用简单硬件原语（例如 RISC-V privilege levels 和 PMP）构造可审计、可裁剪、可移植的可信执行环境。

## 核心概念

- Enclave：受保护的应用执行环境。
- Security Monitor：运行在最高特权层的可信组件，负责 enclave 创建、销毁、内存隔离和 attestation。
- PMP（Physical Memory Protection）：RISC-V 硬件内存访问控制机制，用于限制 S-mode OS 对 enclave 内存的访问。
- Runtime：enclave 内部运行时，向应用提供内存、系统调用代理和基础服务。
- Edge call：enclave 与不可信 host 之间的调用/通信机制。
- Measurement：enclave 初始内容和配置的哈希，用于远程证明。

## 工作原理

Keystone 的典型部署把普通 OS 放在 S-mode，把 Security Monitor 放在 M-mode。普通 OS 负责加载 enclave 镜像和分配资源，但在 enclave 进入运行态后，Security Monitor 通过 PMP 等机制让 OS 无法直接访问 enclave 私有内存。

生命周期大致如下：

1. Host 应用请求创建 enclave。
2. Security Monitor 分配并锁定 enclave 内存区域。
3. Enclave runtime 和应用被加载并度量。
4. Security Monitor 配置 PMP，阻止 OS 访问 enclave 区域。
5. CPU 在 host 和 enclave 之间切换。
6. Enclave 通过 edge call 请求外部 I/O，并把外部返回视为不可信输入。

Keystone 的一个特点是可定制。研究者可以替换 runtime、调整内存模型、添加加密内存、验证 monitor 或扩展硬件隔离策略。这使它非常适合探索 TEE 设计，而不局限于 SGX/TDX/SEV 这些闭源商业实现。

## 远程证明

Keystone attestation 依赖平台根密钥和 Security Monitor 对 enclave measurement 的签名。远程 verifier 应验证：

- 设备或平台证书链是否可信。
- Security Monitor 版本和度量是否符合预期。
- Enclave measurement 是否匹配目标应用。
- Quote 中的 nonce 或临时公钥是否绑定当前会话。

工程上，密钥释放应在 attestation 通过之后发生，并且只释放给 quote 绑定的 enclave 公钥。

## 安全模型

Keystone 通常信任：

- RISC-V 硬件、PMP 实现、Boot ROM 和启动链。
- Security Monitor、enclave runtime 和 enclave 应用。
- 设备证明密钥和 verifier policy。

Keystone 通常不信任：

- S-mode OS、驱动、host 应用。
- 外部存储、网络、计时源和中断输入。
- 其他普通进程或 enclave 外部代码。

## 安全边界与限制

- PMP region 数量和粒度会限制 enclave 内存布局。
- 基础 Keystone 不自动提供强内存加密；离开芯片封装后的物理内存攻击需要平台扩展。
- I/O、文件系统和网络由不可信 host 代理，协议必须自行加密和认证。
- 侧信道、缓存争用、中断计时和页访问模式需要额外设计。
- 作为研究框架，生产使用需要严肃评估具体硬件、monitor、runtime 和供应链。

## 适用场景

Keystone 适合 RISC-V TEE 研究、教学、开源硬件安全实验、可验证 monitor、边缘设备 enclave 原型。若目标是标准化的 RISC-V confidential VM，应关注 CoVE；若目标是成熟云服务，应选择 TDX/SEV-SNP/Azure/Google 等现有方案。

## 参考资料

- Keystone project: https://keystone-enclave.org/
- Keystone documentation: https://docs.keystone-enclave.org/
- Keystone paper: https://arxiv.org/abs/1907.10119

