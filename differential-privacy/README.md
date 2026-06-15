# 差分隐私

差分隐私（Differential Privacy, DP）是一种数学隐私定义，用于限制单个个体数据对统计输出或模型参数的影响。它常用于发布统计数据、训练机器学习模型、联邦学习和数据 clean room 场景。与加密不同，DP 关注“输出还能推断出多少个体信息”。

## 核心概念

- 邻接数据集：只相差一个个体记录的两个数据集。
- 隐私预算 epsilon：越小隐私越强，噪声越大，精度通常越低。
- delta：允许极小概率的隐私失败，形成 `(epsilon, delta)-DP`。
- Sensitivity：单个记录变化对查询结果的最大影响。
- Noise mechanism：根据 sensitivity 和预算添加 Laplace、Gaussian 等噪声。
- Composition：多次查询会累计消耗隐私预算。

## 工作原理

一个机制满足差分隐私，意味着攻击者看到输出后，很难判断某个个体是否在数据集中。直观上，加入或删除某个人的数据，输出分布变化被限制在可量化范围内。

典型步骤：

1. 定义允许的查询或训练任务。
2. 分析 sensitivity，限制单个样本最大贡献。
3. 根据 epsilon/delta 添加噪声。
4. 记录每次查询或训练轮次消耗的隐私预算。
5. 当预算耗尽时停止发布或重新获取授权。

在机器学习中，DP-SGD 通常先裁剪每个样本梯度，再向聚合梯度加入噪声，从而限制单个样本对模型的影响。

## 中心化与本地差分隐私

- Central DP：可信数据处理方看到原始数据，在发布结果时加入噪声。精度较好，但需要信任处理方。
- Local DP：数据在离开用户设备前已随机化。服务端不见原始数据，但噪声更大，效用较低。
- Distributed/Federated DP：结合安全聚合和联邦学习，让服务器只看到带噪聚合更新。

## 安全边界

差分隐私能帮助：

- 抵御重识别和成员推断。
- 量化隐私损失，而不是只给模糊承诺。
- 在发布统计或模型时控制个体影响。

差分隐私不能自动解决：

- 原始数据在处理前的访问控制。
- 小 epsilon 之外的业务合规承诺。
- 查询设计错误导致的低效用或输出泄露。
- 非个体级秘密，例如公司整体机密。
- 恶意分析师绕过预算系统直接看原始数据。

## 常见误区

- “加了噪声就是 DP”：只有按 sensitivity、预算和机制证明添加的噪声才是 DP。
- “epsilon 越大越好”：epsilon 大意味着隐私弱，不能只看模型精度。
- “DP 让数据匿名”：DP 是输出机制属性，不是数据脱敏格式。
- “一次证明永久安全”：多次发布会组合消耗预算。

## 与其他技术的关系

TEE/MPC/FHE 保护计算过程中的访问；差分隐私保护结果发布后的推断风险。实际系统常组合使用：TEE 或 MPC 保护原始数据处理，DP 约束最终统计或模型输出。

## 适用场景

- 人口统计和公共数据发布。
- 产品遥测和指标分析。
- 联邦学习模型训练。
- 数据 clean room 查询结果发布。
- 机器学习成员推断风险缓解。

## 参考资料

- NIST differential privacy blog: https://www.nist.gov/blogs/cybersecurity-insights/getting-started-differential-privacy
- The Algorithmic Foundations of Differential Privacy: https://www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf
- OpenDP: https://opendp.org/
- TensorFlow Privacy: https://github.com/tensorflow/privacy

