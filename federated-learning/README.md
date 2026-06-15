# 联邦学习

联邦学习（Federated Learning, FL）是一种分布式机器学习范式：训练数据保留在客户端、设备或机构本地，中心服务器只协调训练并聚合模型更新。它常被归入隐私计算，但“数据不离开本地”不等于“天然安全”，还需要安全聚合、差分隐私和攻击防护。

## 核心流程

1. 服务器选择一批客户端或机构。
2. 服务器下发当前全局模型和训练配置。
3. 客户端在本地数据上训练若干步。
4. 客户端上传模型更新、梯度或统计量。
5. 服务器聚合更新，生成新全局模型。
6. 多轮迭代直到收敛。

常见算法是 FedAvg：对客户端本地模型更新按样本量或权重做平均。

## 系统架构

联邦学习至少包含以下组件：

```text
Coordinator/server
  - selects clients
  - sends model and training plan
  - aggregates updates
  - tracks rounds and metrics

Clients/silos/devices
  - hold local data
  - train local steps
  - upload updates or encrypted shares

Privacy/security layer
  - secure aggregation
  - differential privacy
  - client authentication
  - robust aggregation / anomaly detection
```

跨设备 FL 和跨机构 FL 差异很大。手机键盘场景可能有百万弱可信、易掉线设备；医疗跨院场景参与方少但每方数据价值高、合规要求强、恶意模型更新风险更突出。

## 横向与纵向联邦

- 横向联邦：各方特征空间相同，样本不同，例如多个医院都有相同字段但患者不同。
- 纵向联邦：各方样本实体有重叠，特征不同，例如银行和电商对同一用户拥有不同特征。
- 联邦迁移学习：样本和特征都不完全重叠，需要迁移或表示学习。

移动端大规模 FL 与跨机构 FL 的工程问题不同。前者关注设备掉线、非 IID、带宽、电量；后者关注合规、审计、恶意参与方和数据对齐。

## 隐私增强机制

联邦学习本身只减少原始数据集中化，不自动防止更新泄露。常见增强包括：

- Secure aggregation：服务器只能看到一批客户端更新之和，不能看到单个客户端更新。
- Differential privacy：裁剪和加噪，限制单个样本或客户端对模型的影响。
- MPC/HE：在聚合或纵向联邦中保护中间值。
- TEE：保护聚合服务器或训练协调器中的明文状态。
- Robust aggregation：缓解投毒、异常更新和拜占庭客户端。

## Secure Aggregation 细节

安全聚合的目标是让服务器只看到一组客户端更新之和，看不到单个客户端更新。典型思想：

1. 客户端之间协商 pairwise mask。
2. 每个客户端上传 masked update。
3. 掩码在求和时相互抵消。
4. 掉线客户端的掩码通过 secret sharing 恢复处理。

这能防服务器直接查看单个更新，但服务器仍能看到聚合结果。如果一轮只有很少客户端，聚合结果仍可能暴露个体信息。因此 secure aggregation 通常要配合最小参与阈值和差分隐私。

## 攻击面

联邦学习常见攻击：

- **Gradient inversion**：从梯度恢复训练样本。
- **Membership inference**：判断某样本是否参与训练。
- **Property inference**：推断客户端数据集属性。
- **Model poisoning**：恶意客户端上传污染更新，降低模型质量。
- **Backdoor attack**：让模型在触发器输入上输出攻击者指定结果。
- **Sybil attack**：攻击者伪造大量客户端影响聚合。
- **Malicious server attack**：服务器下发特制模型或梯度探针，诱导泄露。

因此“数据不出本地”只是起点，不是完整安全结论。

## 聚合与鲁棒性

普通 FedAvg 对恶意更新很脆弱。常见增强包括：

- Norm clipping：限制单个客户端更新幅度。
- Median/trimmed mean：降低极端值影响。
- Krum/Bulyan：选择与多数更新接近的方向。
- Update validation：在服务器或可信验证集上检测异常。
- Client reputation：结合身份和历史行为调整权重。

这些方法与隐私机制可能冲突。例如 secure aggregation 隐藏单个更新后，服务器更难做异常检测；DP 噪声也会影响鲁棒聚合判断。

## 安全模型

联邦学习系统需要明确：

- 服务器是否可信。
- 客户端是否可能恶意。
- 是否允许客户端看到全局模型。
- 更新是否加密，服务器能否看到单个更新。
- 输出模型是否可被任意查询。

不同答案会导向完全不同的设计。比如只防止“服务器收集原始数据”和防止“服务器从梯度反推样本”不是同一个安全目标。

## 安全边界与限制

- 梯度和模型更新可能泄露训练样本，尤其是小 batch、稀有特征和语言模型。
- 恶意服务器可下发特制模型，诱导客户端泄露更多信息。
- 恶意客户端可数据投毒、后门攻击或 Sybil 攻击。
- 非 IID 数据会影响收敛、公平性和模型偏差。
- Secure aggregation 保护单个更新，但聚合结果和最终模型仍可能泄露统计信息。
- 差分隐私会降低精度，需要预算会计和业务权衡。
- 客户端身份、采样策略和掉线模式会泄露元数据。
- 模型发布后仍可能通过查询接口发生反演或抽取攻击。
- 横向联邦中的实体对齐、纵向联邦中的中间梯度传递都可能成为隐私薄弱点。
- 合规上仍需定义数据控制者、处理者、审计和删除权，不因 FL 自动消失。

## 工程检查清单

- 每轮最小客户端数是否足够。
- 是否启用 secure aggregation，掉线协议是否安全。
- 是否有 client-level 或 example-level DP。
- 是否限制服务器下发任意训练计划。
- 是否验证客户端身份，防 Sybil。
- 是否有模型更新裁剪和异常检测。
- 是否审计最终模型的成员推断和数据记忆风险。
- 是否能处理客户端异构、非 IID 和通信失败。

## 与 TEE/MPC/DP 的关系

联邦学习是训练架构，不是单独的密码学安全定义。它经常组合：

- Secure aggregation/MPC 保护更新。
- DP 控制模型记忆。
- TEE 保护聚合器和密钥管理。
- ZKP 证明客户端更新满足范围或训练规则。

## 适用场景

- 移动端键盘、推荐和个性化模型。
- 医疗机构联合建模。
- 金融风控多机构协作。
- IoT/边缘设备模型训练。
- 数据不能集中但允许联合改进模型的场景。

## 参考资料

- Google Federated Learning: https://federated.withgoogle.com/
- TensorFlow Federated: https://www.tensorflow.org/federated
- Practical Secure Aggregation: https://research.google/pubs/practical-secure-aggregation-for-privacy-preserving-machine-learning/
- TensorFlow Privacy: https://github.com/tensorflow/privacy
