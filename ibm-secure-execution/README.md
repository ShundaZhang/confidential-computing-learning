# IBM Secure Execution for Linux

IBM Secure Execution for Linux 是 IBM Z 和 LinuxONE 平台上的机密计算能力，用于运行受保护的 Linux 虚拟机。它的目标是在大型机虚拟化环境中保护 guest 内存和状态，使云管理员、host hypervisor 或其他高权限软件无法读取受保护 VM 的明文。

## 核心概念

- Protected VM：启用 Secure Execution 的受保护 Linux 虚拟机。
- Ultravisor：IBM Z 平台中位于 hypervisor 之下的可信保护层，用于隔离 protected guest。
- Secure image：经过加密和签名的 guest 镜像，只有符合平台安全条件时才能启动。
- Host hypervisor：可以是 KVM 或 z/VM，负责调度和资源管理，但不应访问 protected guest 明文。
- Attestation/verification：验证启动镜像、平台和保护状态，再释放密钥。

## 工作原理

Secure Execution 把虚拟机保护逻辑放入 IBM Z 硬件和固件支持的安全层。Host hypervisor 仍管理虚拟化资源，但当 guest 进入 protected 状态后，guest 内存受到硬件和 ultravisor 保护。Host 不能读取 guest 明文内存，也不能任意注入或修改受保护状态。

部署通常包括：

1. 生成受保护 guest 镜像，并把敏感启动材料加密。
2. 在 IBM Z/LinuxONE 上通过 KVM 或 z/VM 启动 protected VM。
3. 平台验证镜像和安全状态。
4. 受保护 guest 获取运行所需密钥。
5. Guest 在机密边界内处理数据。

IBM Z 的优势是成熟的企业级虚拟化、硬件安全模块和高可靠平台能力。Secure Execution 把这些能力带入 confidential VM 模型。

## 安全模型

Secure Execution 通常信任：

- IBM Z/LinuxONE 硬件、固件、ultravisor 和启动链。
- Protected VM 镜像、guest OS 和应用。
- 镜像签名者、密钥管理和 attestation policy。

通常不信任：

- Host hypervisor 管理员。
- Host OS、普通运维工具和同机 workload。
- 外部存储、网络和不可信 I/O。

## 安全边界与限制

- VM 内 OS 和应用仍属于 TCB；guest 内漏洞仍可泄露秘密。
- Hypervisor 仍可拒绝服务或影响调度。
- I/O 需要独立加密和完整性保护。
- 安全属性强依赖 IBM Z 平台、固件级别、Linux 发行版和云服务配置。
- 与 x86/Arm confidential VM 的证明格式和运维生态不同，跨平台迁移需要重新设计。

## 适用场景

Secure Execution 适合金融、政府、主机现代化、核心交易系统、合规要求高的 IBM Z/LinuxONE 工作负载。若企业已经运行在 IBM Z 生态，它可以作为机密计算优先选项；若目标是通用公有云迁移，可对比 Azure/GCP 上的 TDX/SEV-SNP。

## 参考资料

- IBM Secure Execution documentation: https://www.ibm.com/docs/en/linux-on-systems?topic=virtualization-secure-execution-linux
- IBM Hyper Protect Virtual Servers: https://www.ibm.com/products/hyper-protect-virtual-servers
- Linux on IBM Z: https://www.ibm.com/linuxone

