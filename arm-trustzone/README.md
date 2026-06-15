# Arm TrustZone

Arm TrustZone 是 Arm SoC 上广泛部署的安全隔离技术。它把系统划分为 Secure world 和 Non-secure world：普通操作系统通常运行在 Non-secure world，安全监控器、TEE OS 和可信应用运行在 Secure world。TrustZone 更像设备级安全基础设施，而不是云多租户机密 VM 技术。

## 核心概念

- Secure world：更高信任级别的执行世界，承载安全启动、密钥、TEE OS 和可信应用。
- Non-secure world：普通 rich OS，例如 Android、Linux 或 RTOS 应用环境。
- Secure Monitor：负责两个 world 之间切换的低层软件。
- NS bit：总线事务、页表、缓存和外设访问中的安全状态标记。
- TEE OS：如 OP-TEE，提供可信应用运行时、加密服务和安全存储。
- Trusted Application（TA）：运行在 TEE OS 内的安全应用。

## 工作原理

TrustZone 的隔离不是只发生在 CPU 指令层，而是贯穿 SoC。内存控制器、总线、防火墙、中断控制器和外设都可以根据安全状态决定访问权限。一个典型移动设备中，普通 Android kernel 无法直接读取 Secure world 的内存，也不能访问被标记为 secure 的密钥外设或安全显示路径。

常见调用路径：

1. Non-secure 应用调用 TEE client API。
2. Linux/Android 驱动发起 SMC（Secure Monitor Call）。
3. Secure Monitor 切换到 Secure world。
4. TEE OS 调度对应 TA。
5. TA 处理密钥、签名、生物识别或 DRM 逻辑。
6. 结果通过共享内存返回 Non-secure world。

TrustZone 的安全性高度依赖 SoC 集成。CPU 只提供基础状态和切换机制；真正的安全边界需要内存区域、外设、DMA、中断、调试口、启动链和固件共同正确配置。

## 安全模型

TrustZone 通常信任：

- SoC 硬件、Boot ROM、secure boot chain。
- Secure Monitor、TEE OS、可信应用和安全外设配置。
- 设备厂商签名、密钥管理和回滚保护机制。

TrustZone 通常不信任：

- Normal world OS、普通应用、root 权限攻击者。
- 普通世界驱动、文件系统、网络和 UI 输入。

## 安全边界与限制

- TrustZone 不是天然多租户云隔离技术。Secure world 通常是平台共享高权限环境，多个 TA 的隔离取决于 TEE OS。
- Secure world 漏洞影响很大，因为它常持有设备根密钥、支付密钥或生物识别数据。
- 共享内存和 TEE client API 是主要攻击面，需要严格参数验证。
- 侧信道、物理攻击、电压/时钟 glitch、调试口和供应链攻击需要额外防护。
- 不同厂商实现差异大，安全属性不能只根据“支持 TrustZone”判断。

## 适用场景

TrustZone 适合设备 root of trust、可信启动、密钥存储、移动支付、DRM、生物识别、IoT 设备身份和安全固件服务。如果目标是云租户隔离，应优先看 Arm CCA；如果目标是在 RISC-V 上研究可定制 TEE，可看 Keystone。

## 参考资料

- Arm TrustZone for Cortex-A: https://www.arm.com/technologies/trustzone-for-cortex-a
- OP-TEE: https://optee.readthedocs.io/
- Trusted Firmware-A: https://www.trustedfirmware.org/projects/tf-a/

