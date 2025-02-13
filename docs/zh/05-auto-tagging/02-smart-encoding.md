---
title: SmartEncoding
permalink: /auto-tagging/smart-encoding
---

DeepFlow 为每个服务自动注入 20+ 资源标签以及所有的 K8s 自定义 Label，在一个典型的生产环境中，需要为一条路径数据自动注入的标签可能多达 100+ 个。这些标签为后端存储带来了很大的压力，DeepFlow 的 SmartEncoding 机制创新的解决了此问题，使得性能开销显著降低。

![SmartEncoding](../about/imgs/smart-encoding.png)

SmartEncoding 依赖于对各类标签信息的采集和编码。Agent 通过信息同步获取到字符串格式的标签，汇总到 Server 上。Server 通过对所有的标签进行编码，为所有的数据统一注入 Int 类型的标签并存储到数据库中。在此之后，对观测数据的 SmartEncoding 过程包含三个阶段：
- 采集阶段：Agent 为观测数据自动注入 VPC（Integer）和 IP 标签
- 存储阶段：Server 根据 Agent 标记的 VPC 和 IP 标签，为观测数据自动注入 20+ 维度的 K8s 资源、云资源属性标签
- 查询阶段：Server 计算自定义标签和资源之间的关系，在查询阶段为观测数据自动注入所有 K8s 自定义 Label 标签

我们看到，AutoTagging 解决了数据孤岛的痛点，SmartEncoding 机制解决了资源开销的痛点。使用 DeepFlow 时，我们建议：
- 资源开通时，为云资源注入尽量丰富的标签
- 业务上线时，向 K8s 中注入尽量丰富的 Label
- 服务注册时，向服务注册中心注入尽量丰富的信息

通过这些方法，我们能够极大的避免在业务代码中手动注入标签的繁琐过程，并且能显著的降低后端存储的压力。

我们对 SmartEncoding 进行的 Benchmark 表明，通过对标签的编码可将数据写入的性能提升 10 倍。我们随机生成了一组长度为 16 个字符的标签，Cardinality 为 5000，基于这样的数据模型对 SmartEncoding、ClickHouse LowCard、无编码 三种方案进行了对比，测试结果如下：

| 类型                  | 标签字段类型    | CPU 用量 | 内存用量 | 磁盘用量 |
| --------------------  | --------------  | -------- | -------- | -------- |
| 基线（SmartEncoding） | Int             | 1        | 1        | 1        |
| 写时编码              | LowCard(string) | 10       | 1        | 1.5      |
| 无编码                | string          | 5        | 1.5      | 7.5      |

实际上对于 K8s 自定义 Label，我们无需将其随观测数据存储，由于 Label 和 K8s 资源的对应关系是明确的，我们可以在查询阶段将查询 SQL 语句翻译为对 K8s 资源的过滤和分组，因此而带来的存储开销将会更为客观。此外，由于编码后的数据体系大幅度降低，也能降低数据查询是的磁盘扫描量，提升查询性能。

详细的 Benchmark 数据集我们也将会公开在 GitHub 上。
