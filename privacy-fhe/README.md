# 全同态加密（FHE）

同态加密允许在密文上执行计算，解密计算结果后得到与明文计算相同或近似的结果。全同态加密（Fully Homomorphic Encryption, FHE）进一步允许对任意可表示的电路进行计算，是“数据始终保持加密”的重要隐私计算技术。

## 核心概念

- Plaintext：原始数据。
- Ciphertext：加密后的数据，可被服务器计算。
- Evaluation：服务器在不知道明文和私钥的情况下执行加法、乘法或门电路。
- Secret key：通常只由数据拥有方持有，用于最终解密。
- Noise budget：密文中噪声会随计算增长，超过阈值后无法正确解密。
- Bootstrapping：刷新密文、降低噪声，使更深计算成为可能。

## 常见方案

- BFV/BGV：适合精确整数或模整数计算。
- CKKS：适合近似实数/复数计算，常用于机器学习和统计。
- TFHE/FHEW：适合布尔门或低延迟 bootstrapping。

不同方案的编码、误差、乘法深度、打包能力和性能差异很大。FHE 工程通常不是“把现有程序直接加密运行”，而是把算法重写成适合 FHE 的低深度电路。

## 工作原理

多数现代 FHE 基于 LWE/RLWE 等格困难问题。加密后，密文保留某种代数结构，使得：

- 密文加法对应明文加法。
- 密文乘法对应明文乘法，但会增加噪声和密文复杂度。
- Relinearization/Galois keys 支持乘法后降维、旋转和 SIMD packing。
- Bootstrapping 可同态执行“解密电路”，把噪声大的密文刷新为噪声小的密文。

CKKS 常通过近似编码把向量打包进一个密文，使用旋转、加法、乘法实现线性代数。代价是结果存在数值误差，需要管理 scale、level 和 rescale。

## 安全模型

FHE 通常信任：

- 数据拥有方私钥安全。
- 加密参数满足目标安全级别。
- 实现库正确处理随机数、参数、序列化和 side-channel。

FHE 通常不信任：

- 执行计算的服务器。
- 存储密文的云平台。
- 观察计算过程的外部攻击者。

## 安全边界与限制

- FHE 保护输入和中间值，但不自动保护输出。解密结果可能泄露敏感信息。
- 服务器可返回错误结果，除非结合可验证计算、ZKP、重复计算或审计。
- 计算电路、访问模式、密文大小、运行时间可能泄露元数据。
- 性能开销仍然显著，尤其是 bootstrapping 和深度神经网络。
- 参数选择很专业，错误参数会导致安全或正确性问题。
- CKKS 是近似计算，不适合要求逐 bit 精确的业务逻辑，除非设计可容忍误差。

## 与 TEE 的比较

FHE 不需要信任硬件或云管理员，理论边界更“密码学纯粹”。但它的可计算范围和性能受限，通常需要算法重写。TEE 可以运行通用程序但需要信任硬件和较大的软件 TCB。实际系统可能组合二者：TEE 管理密钥和预处理，FHE 处理最敏感数据，ZKP 验证结果。

## 适用场景

- 加密数据统计。
- 隐私保护机器学习推理。
- 医疗或基因数据查询。
- 低频高价值的外包计算。
- 需要云端永不见明文的计算服务。

## 参考资料

- Homomorphic Encryption Standardization: https://homomorphicencryption.org/
- OpenFHE: https://openfhe.org/
- Microsoft SEAL: https://www.microsoft.com/en-us/research/project/microsoft-seal/
- Google FHE transpiler: https://github.com/google/fully-homomorphic-encryption

