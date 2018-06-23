# Online, Asynchronous Schema Change in F1

## Google F1 的 schema 变更算法

* F1 中的算法实现

租约

F1 中 schema 以特殊的 kv 对存储于 Spanner 中，同时每个 F1 服务器在运行过程中自身也维护一份拷贝。为了保证同一时刻最多只有 2 份 schema 生效，F1 约定了长度为数分钟的 schema 租约，所有 F1 服务器在租约到期后都要重新加载 schema 。如果节点无法重新完成续租，它将会自动终止服务并等待被集群管理设施重启。

中间状态

前面已经提过，F1 在 schema 变更的过程中，会把一次 schema 的变更拆解为多个逐步递进的中间状态。实际上我们并不需要针对每种 schema 变更单独设计中间状态，总共只需要两种就够了： delete-only 和 write-only 。

delete-only 指的是 schema 元素的存在性只对删除操作可见。

例如当某索引处于 delete-only 状态时，F1 服务器中执行对应表的删除一行数据操作时能“看到”该索引，所以会同时删除该行对应的索引，与之相对的，如果是插入一行数据则“看不到”该索引，所以 F1 不会尝试新增该行对应的索引。

具体的，如果 schema 元素是 table 或 column ，该 schema 元素只对 delete 语句生效；如果 schema 元素是 index ，则只对 delete 和 update 语句生效，其中 update 语句修改 index 的过程可以认为是先 delete 后再 insert ，在 delete-only 状态时只处理其中的 delete 而忽略掉 insert 。

总之，只要某 schema 元素处于 delete-only 状态，F1 保证该 schema 元素对应的 kv 对总是能够被正确地删除，并且不会为此 schema 元素创建任何新的 kv 对。

write-only 指的是 schema 元素对写操作可见，对读操作不可见。

例如当某索引处于 write-only 状态时，不论是 insert 、 delete ，或是 update ，F1 都保证正确的更新索引，只是对于查询来说该索引仍是不存在的。

简单的归纳下就是 write-only 状态的 schema 元素可写不可读。

示例推演

Google 论文的叙述过程是描述完两种中间状态后就开始了冗长的形式化推导，最后得以证明按照特定的步骤来做 schema 演化是能保证一致性的。这里我想先拿出一个例子把 schema 变更的过程推演一遍，这样形成整体印象后更有助于看懂证明：）我们以添加索引为例，对应的完整 schema 演化过程如下：

absent --> delete only --> write only --(reorg)--> public
其中 delete-only 和 write-only 是介绍过了的中间状态。 absent 指该索引完全不存在，也就是schema变更的初始状态。 public 自然对应变更完成后就状态，即索引可读可写，对所有操作可见。

reorg 指 “database reorganization”，不是一种 schema 状态，而是发生在 write-only 状态之后的一系列操作。这些操作是为了保证在索引变为 public 之前所有旧数据的索引都被正确地生成。

根据之前的讨论，F1 中同时最多只可能有两分 schema 生效，我们逐个步骤来分析。

先看 absent 到 delete-only 。很显然这个过程中不会出现与此索引相关的任何数据。

再看 delete-only 到 write-only 。根据 write-only 的定义，一旦某节点进入 write-only 状态后，任何数据变更都会同时更新索引。当然，不可能所有节点同时进入 write-only 状态，但我们至少能保证没有节点还停留在 absent 状态， delete-only 或 write-only 状态的节点都能保证索引被正确清除。于是我们知道：从 write-only 状态发布时刻开始，数据库中不会存在多余的索引。

接下来是 reorg ，我们来考察 reorg 开始时数据库的状态。首先因为 delete-only 的存在，我们知道此时数据库中不存在多余的索引。另外此时不可能有节点还停留在 delete-only 状态，我们又知道从这时起，所有数据的变更都能正确地更新索引。所以 reorg 要做的就是取到当前时刻的snapshot，为每条数据补写对应的索引即可。当然 reorg 开始之后数据可能发生变更，这种情况下底层Spanner提供的一致性能保证 reorg 的写入操作要么失败（说明新数据已提前写入），要么被新数据覆盖。

基于前面的讨论，到 reorg 完成时，我们的数据不少不多也不错乱，可以放心地改为 public 状态了。

证明过程简介

这里简单介绍下证明过程，以理解为主，详细情况可自行查阅论文。

我们定义数据库表示为存储引擎中所有 kv 对的集合。数据库表示 d 对于 schema S 是一致的，当且仅当

d 中不存在多余数据。
d 中的数据是完整的。
其中不存在多余数据要求：

d 中的列数据或索引必须是 S 中定义过的列或索引。
d 中所有索引都指向合法的行。
d 中不存在未知数据。
数据的完整性要求：

public 状态的行或索引是完整的。
public 状态的约束是满足的。
我们要求正确实现所有 delete, update, insert 或 query 操作，保证其对于任何schema S ，都不破坏数据库表示的一致性。

我们定义schema S1 至 S2 的变更过程是保持一致的，当且仅当

任何 S1 所定义的操作 OPs1 都保持数据库表示 d 对于 S2 的一致性。
任何 S2 所定义的操作 OPs2 都保持数据库表示 d 对于 S1 的一致性。
这里看起来第 2 点是没必要的，但确实是必须的，因为 F1 中在变更发生的过程中 S1 和 S2 是会同时生效的。我们考虑为 table 添加一列 optional 列 C 的变更（ optional 表示允许该列在数据库表示 d 中不存在，对应于 SQL 中未定义 NOT NULL 或有 DEFAULT 的情况）。首先， S2 定义的 insert 操作会写入 C 列的数据，其产生的数据库表示 d' 对 S2 是一致的，但对 S1 是不一致的（有多余数据）。现在如果发起由 S1 定义的 delete 操作， C 列对应的数据就不能被正确删除了。

显然根据定义，我们有如下推论：schema S1 至 S2 的变更过程是保持一致的，当且仅当 S2 至 S1 的变更过程也是保持一致的。

接下来我们可以得出如下结论：任何从 schema S1 至 S2 的变更，如果其添加或删除了一项 public schema 元素 E ，那么此变更不能保持一致性。

我们先考虑添加 E 的情况。不论 E 是 table, column 或 index ，由 S2 定义的 insert 都将插入 S1 未定义的数据，所以 S1 至 S2 的变更不能保持一致性。根据上面的“可逆”推论，删除的情况也得证。

接下来我们要证明：任何从 schema S1 至 S2 的变更，如果其添加了一项 delete-only schema 元素 E ，那么此变更过程保持一致。

因为 S1 和 S2 上定义的任何操作都不会为 E 创建数据，显然不会产生多余数据。又因为 E 不处于 public 状态，自然也不会破坏数据完整性。所以该变更保持一致性。

接下来我们要证明：任何从 schema S1 至 S2 的变更，如果其将一项 delete-only 状态的 schema optional 元素 E 置为 public ，那么此变更过程保持一致。

因为 S1 和 S2 上定义的 delete 操作都能正确地删除 E 所对应的 kv 对，不会产生多余数据。由于 S1 中 E 是 delete-only ， S1 所定义的 insert 不会为 E 写入对应的数据，但是因为 E 是 optional 的，数据的缺失最终不会影响一致性。所以该变更过程保持一致。

到这里，我们就有了针对添加 optional schema 元素的完整变更方案：

absent --> delete-only --> public
删除 schema 元素以及添加删除 required 元素的情况都是类似的推导过程，这里就不再赘述了，具体可参考论文。