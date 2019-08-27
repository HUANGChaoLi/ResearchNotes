# 面向文档型数据库建模（MongoDB）

## 一. NoSQL与关系型数据库的区别
### Relative Databases
在关系型数据库中，我们通过将重复数据放入单表中来规范化模式以消除冗余，这给我们带来了以下的好处：
1. 不需要在更新发生的时候更新多个副本，这可以使得写操作更快
2. 通过使用单一副本而不是多副本，我们可以减少存储空间

然而，这带来了连接(join)操作，因为数据必须在多个表中检索，所以查询会花更多的时间才能完成。

### NoSQL Databases
1. 作为NoSQL系统的重要特性之一，非规范化可以认为是连接(join)的一种替代，我们可以对数据进行反规范化或复制，以便检索和存储数据。  
2. NoSQL系统的另一个关键特性是“无共享”(Share Nothing)水平扩展，即在许多服务器上对数据进行复制和分区，因为这个特性，NoSQL系统可以支持每秒大量的简单读/写操作。  
3. NoSQL基本上不提供对ACID的保障(注：MongoDB v4.0之后支持ACID)，但是支持BASE的保障。BASE指的是基本可用性(Basically)、软状态(Soft-state)、最终一致性(Eventually consistent)。  

> 注: 其中基本可用性指的是在大多数时间里大多数数据是可用的；软状态则指的是数据不总是在所有时间里都是一致的，但会达成最终一致的状态。  

## 二. 建模的主要问题以及建模的主要因素
与关系型数据库拥有完整的建模方法(ER模型)不同，NoSQL因为其灵活性目前尚未有标准的建模方法，但好消息是部分业界人员与研究人员对其进行了很详细的分析，目前可以找到部分建模指南，其关注的主要问题如下：
1. 在文档数据库中如何对关系建模？
2. 如何知道何时引用或者嵌入文档？
3. 何时进行反规范化操作？

为了回答这些问题，我们除了需要关注NoSQL与关系型数据库的区别之外，在MongoDB中还需要考虑以下限制与特性：
1. MongoDB最大的文档大小为16MB，最佳是几KB，需要确保单个文档不使用过多的RAM
2. MongoDB在v4.0之前都未支持跨文档事务
3. MongoDB提供了Sharding分片技术来在多个节点中分发数据，当节点包含不同数量的数据时，MongoDB会自动重新分配数据，以便在节点之间均匀分配负载

在具体到应用建模的时候，我们还需要考虑：
1. 查询特性（过滤器数量、要返回的属性数量等）
2. 查询响应的要求
3. 查询使用频率
4. 数据存取模式

## 三. 关系建模标准指南
### 关系基数
| ID | 基数 | 标记 | 例子 |
| -- | --- | --- | --- |
| 1）| One-to-One  |	1:1 |	Person ← → Id card    |
| 2）| One-to-Few  |	1:F |	Author  ← → Addresses |
| 3）| One-to-Many |	1:M |	Post  ← → Comments    |
| 4）| One-to-Squillions | 1:S | System  ← → Logs       |
| 5）| Many-to-Many      | M:M | Customer  ← → Products |
| 6）| Few-to-Few        | F:F | Employees  ← → Tasks   |
| 7）| Squillions-to-Squillions | S:S | Bank Transactions  ← → Logs |
> 注：Few 指的是几到几十；Many 指的是成百上千；Squillions 指的是海量

### 关系类别
| ID | 类别 | 标记 | 子类别 | 说明 |
| -- | --- | --- | ----- | ---- |
| 1）| Embedding（嵌入） | EMB | 1. 单向嵌入 2. 双向嵌入 | 从特性上讲，嵌入在从文档存储数据库中检索数据时提供了更好的读取性能。但是，在将数据写入数据库时，它可能会非常慢，不同于使用将数据水平写入较小文件的概念。 |
| 2）| Referencing（引用）| REF | 1. 子引用 2. 父引用 | 通常，引用可提供更好的写入性能。但是，读取数据可能需要更多次的往返服务器请求。 |
| 3）| Bucketing（分组）| BUK | 无 | 通过根据数量、时间等将海量数据切分成可管理的大小（如使用在1:S关系中，每n个生成一个文档），分组增强了数据检索能力。 |
> 注：
> 1. 规范化数据可能有助于节省一些空间，但随着当前技术的进步，空间不再是问题。
> 2. 了解数据访问模式，应用程序中使用的数据的性质，特定字段的更新速率以及实体之间的基数关系。这些是文档存储数据库的设计和建模结构的重要因素。

## 四. 关系类别使用标准指南
### Embedding（嵌入）
| ID | 规则 | 说明 |
| -- | --- | --- |
| G1 | 除非特殊情况，否则嵌入子文档 | 为了在保存和检索速度方面获得更好的系统性能，除特殊情况外请始终尝试嵌入子文档。使用嵌入时，无需执行单独的查询来检索嵌入的文档。 |
| G2 | 嵌入时使用数组概念 | 建议在建模“Few”关系时使用嵌入文档数组。 |
| G3 | 在父文档中定义数组上限 | 如果它包含上千个文档，则在关系的多个方面避免使用无限数组的ObjectID引用。 |
| G4 | 嵌入总是一起管理的记录 | 当一起查询、操作和更新记录时，它们应当被嵌入。 |
| G5 | 嵌入依赖文档 | 依赖是嵌入文档的关键指标之一。例如订单详细信息仅取决于订单本身，因此他们应该保存在一起。 |
| G6 | 嵌入一对一的关系 | 在建模一对一关系时，应该应用嵌入。 |
| G7 | 将具有相同波动性的数据进行分组 | 数据应根据其变化的速率来进行分组。例如，人们的生物数据和几个社交媒体帐户的状态。社交媒体状态的波动性高于生物数据，这些生物数据(例如，出生日期等)不会经常像电子邮件地址那样发生变化，甚至根本没有变化。 |
| G8 | 在N:M关系中，如果N关系的大小接近于M关系的大小时，优先使用双向嵌入 | 在N:M关系中，尝试通过预测N和M的最大数量来建立关系平衡。 |
| G9 | 在N:M关系中，如果N和M关系之间的大小差距巨大，优先使用单向嵌入 | 例如在N侧为3，在M侧为300000，则应考虑单向嵌入。 |

### Referencing（引用）
| ID | 规则 | 说明 |
| -- | --- | --- |
| G10 | 引用常变动的文档 | 文档的高波动性为引用文档而不是嵌入提供了良好的信号。例如，让我们考虑在社交媒体上发布的帖子，喜欢的标签的变化经常发生，因此，它从主文档中解除绑定，这样每次按下按钮时就不需要总是访问主文档。 |
| G11 | 引用独立的实体 | 如果单独访问子文档/对象，则应避免嵌入子文档/对象。嵌入时，单独检索单个实体会同时检索主实体。 |
| G12 | 使用引用数组来引用实体的“Many”关系 | 当关系是1:M关系或文档是独立文档时，使用引用数组是最好的选择。 |
| G13 | 对于大量文档，建议使用父引用 | 例如，如果实体的关系是Squillions(海量)的，父引用是首选。 |
| G14 | 如果子文档很多，则不要嵌入子文档 | 具有许多其他子实体的关键实体应该采用引用而不是嵌入，这将最小化高基数数组。 |
| G15 | 索引所有文档以获得更好的性能 | 如果文档的索引正确，则应用程序级别的连接操作无需担心。 |

### Bucketing（分组）
| ID | 规则 | 说明 |
| -- | --- | --- |
| G16 | 必要时结合嵌入和引用 | 嵌入和引用可以合并在一起并完美地工作。 例如，考虑亚马逊网站上的产品广告，包括产品信息，可能更改的价格以及评论和喜欢的列表。由于这个广告实际上结合了嵌入和引用，因此将这两种技术合并在一起可能是这种情况下的最佳实践。 |
| G17 | 将存储大量内容的文档进行分组 | 为了将文件分成适合的批次，如根据天，月，小时，数量等，应考虑分组。例如，作为分页的情况，关系中的squillions可以被分成每个显示500条记录。 |

### General（常规）
| ID | 规则 | 说明 |
| -- | --- | --- |
| G18 | 当读/写频率非常低时，对文档进行反规范化 | 仅在文档未定期更新时才对文档进行非规范化。因此，访问频率预测应指导是否对任何实体进行非规范化的决策。 |
| G19 | 对两个连接的文档进行非规范化以进行半组合检索 | 有时连接两个文档，但只检索一个文档，而第二个文档中的字段很少，非规范化可以在这里起到帮助作用。例如，在检索演示会话时，还需要显示演讲者姓名，但不是所有演讲者的详细信息，因此，第二个文档（演讲者）被非规范化以仅获取演示者的名称并将其附加到会话文档中。 |
| G20 | 使用标签类型的格式来进行数据传输 | 如果信息不敏感，建议将其打包在XML文档中的标签中。 |
| G21 | 如果安全性是更优先考虑的，则使用目录层次结构 | 将基于角色的授权应用于每个目录以进行访问保护。用户可以拥有访问一个目录或一组目录的权限，具体取决于用户角色。 |
| G22 | 使用文档集合类型的格式来可获得更好的读/写性能 | 这与G20（注：[2]中指的是G21，笔者认为是笔误）相同，但增加了更好的读/写性能。 |
| G23 | 使用不可见的元数据进行节点或服务器之间的数据传输 | 在许多情况下，API没有嵌入安全机制。因此，强烈建议在到达之前对传输和解码之前的敏感信息进行编码。这将提高数据运输中的数据安全性。 |

### 根据应用的特性来进行具体分析，根据优先级使用
#### 考虑读写操作
| 考虑情况 | 优先级 |
| ------ | ------ |
| 更高的可用性（读操作）| G6 > G1 > G17 > G15 > G2 > G7 > G11 > G9 > G3 > G19 > G5 > G4 > G22 > G8 > G12 > G10 > G13 > G14 > G18 > G23 > G16 > G20 > G21 |
| 更好的一致性（写&更新操作）| G1 > G6 > G4 > G5 > G7 > G18 > G10 > G11 > G14 > G12 > G8 > G15 > G3 > G9 > G13 > G16 > G2 > G21 > G19 > G23 > G22 > G20 > G17 |

#### 考虑关系基数
| 关系基数 | 优先级 |
| ------ | ------ |
| 1:1 | G6 > G1 > G5 > G15 > G11 > G10 > G22 > G20 > G231:F → G1 > G15 > G4 > G2 > G5 > G3 > G7 > G22 > G20 > G23 |
| 1:M | G1 > G15 > G2 > G3 > G11 > G4 > G12 > G22 > G20 > G23 |
| 1:S | G15 > G9 > G7 > G10 > G11 > G12 > G13 > G17 > G14 = G22 > G20 > G23 |
| F:F | G15 > G8 > G16 > G7 > G11 > G22 > G17 > G20 > G23 |
| M:M | G15 > G8 > G16 > G12 > G11 > G13 > G14 > G22 > G20 > G23 |
| S:S | G15 > G7 > G16 > G10 > G11 > G13 > G17 > G14 = G22 > G20 > G23|

## 五. 具体场景设计模式
这部分参考 **“Building with Patterns”** 系列[7]，具体看：[链接](https://www.mongodb.com/blog/post/building-with-patterns-a-summary)

## REFERENCES
[1] Imam AA, Basri S, Ahmad R, Abdulaziz N, González‐aparicio MT. **New cardinality notations and styles for modeling NoSQL document‐stores databases**. In: IEEE Region 10 Conference (TENCON), Penang, Malaysia, 2017. p. 6.

[2] Imam AA, Basri S, Ahmad R, Aziz N, Gonzlez‐aparicio MT, Watada J, Member S. **Data modeling guidelines for NoSQL document‐store databases**. In: IEEE Trans. Knowl. Data Eng. 2017. p. 1–14.

[3] Paolo Atzeni, Francesca Bugiotti, Luca Cabibbo, Riccardo Torlone. **Data Modeling in the NoSQL World**. To appear, 2017. hal-01611628: 20-23.

[4] Rupali Arora, Rinkle Rani Aggarwal. **Modeling and Querying Data in MongoDB**. International Journal of Scientific & Engineering Research, Volume 4, Issue 7, July-2013. ISSN 2229-5518.

[5] Fatma Abdelhedi, Amal Ait Brahim, Faten Atigui, Gilles Zur uh. **MDA-based approach for NoSQL Databases Modelling**. 19th International Conference on Big Data Analytics and Knowledge Discovery (DaWaK 2017), Aug 2017, Lyon, France. pp. 88-102. hal-01873732.

[6] Harley Vera Olivera, Maristela Holanda, Valeria Guimarâes, Fernanda Hondo, Wagner Boaventura. **Data Modeling for NoSQL Document-Oriented Databases**. SIMBig 2015: 129-135.

[7] Daniel Coupal, Ken W. Alger. **Building with Patterns**. MongoDB, 2019. Available: https://www.mongodb.com/blog/post/building-with-patterns-a-summary. [Accessed: 26-Aug-2019].

[8] Carol McDonald. **Data Modeling Guidelines for NoSQL JSON Document Databases**. MAPR, 2017.Available: https://mapr.com/blog/data-modeling-guidelines-nosql-json-document-databases/. [Accessed: 26-Aug-2019].
