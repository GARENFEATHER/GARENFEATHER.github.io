# 渐进式锁：快速、可升级的读写锁

---

- 原文：[Progressive Locks : fast, upgradable read/write locks](http://wtarreau.blogspot.com/2018/02/progressive-locks-fast-upgradable.html)
- 翻译：[GarenRhine](http://ghost.garenfeather.cn/author/garenrhine/)
- 未授权，转载请注明出处


## 语境

创建弹性二叉树（ebtree）结构几年后，我开始从需要在多线程环境中使用它们的用户那里获得一些反馈，他们询问是否有线程安全（MT - safe）的版本可用。当时并没有什么对策，我只能建议他们在所有树操作相关代码周围放些锁。这种方法通常运行效果不错，但始终是次优的方案，因为这些树结构具备的部分良好特性在被频繁而沉重的锁包围时会受损。因此，我们需要思考一个改动幅度最小的方案来对2012年的版本进行改善集成。

## 常见用例

似乎弹性二叉树最常见的使用场合是这样的：

- 数据捕捉（与[弹性二叉树的LRU缓存示例](http://git.1wt.eu/web?p=ebtree.git;a=blob;f=examples/lru.c;h=e5ddf0065026f2756818e1b11ebd2a4b00c79700;hb=HEAD)相似，其导向的最终目标也是 HAProxy）
- 日志解析/分析（例如：统计某标准值的出现次数及其与其它标准值的关系）
- 数据转换（如 HAProxy 中的各种 map，多数用于基于IP的地理定位）

这些用例都包含一些重要特性：

- 树结构的读频率远高于写更新频率（更新通常少于 2-3%）
- 这些操作通常是作为某些更复杂操作中的一小部分，而且单个高级操作通常需要复数次的树结构访问，因此这些操作需要保持极高速。
- 涉及这些操作的工作负载通常较重，且旺旺受益于多线程。

乍看上去，一些[读/写锁](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock)在改目标下完美适用。但是由于读写锁在获取和释放锁时涉及2个原子操作，其速度通常较慢。而这意味着每次树查找都要执行4个原子操作，导致在特定场合中甚至可能会比树查找本身还要慢。当然只在写操作时使用锁速度会更快，但这不是我们这里关心的话题。从一些用户那里我确认到因为树结构的低并发性，他们通常倾向于在树操作周围使用更快的自旋锁，尽管这意味着无法进行任何并行化计算。

对其他有并行化需求和想要进行插入操作（缓存、日志处理等方面）的用户来说，读/写锁将可扩展性限制到了写操作耗时与其他不需要访问树结构操作的比率上。事实如此，写锁本身是互斥的，每执行一个写操作树结构都处于不可用状态，其他读线程不得不等待写操作完成。

## 弹性二叉树结构时间耗费观察

弹性二叉树平均查找插入时间复杂度为 O(log(N))，其中N是树中的条目数。实际上对小树来说这并不完全准确的，因为弹性二叉树是一个改良基数树，所以在查找或插入过程中访问过节点的数量实际上对应于沿着到被请求节点链的公共前缀的数量，其值最高可以与数据元素中的比特位数量相等。树下降操作通过与在每个位置找到的数据进行某些位值的比较来完成，并基于此决定向左或向右移动。因为在最坏的情况下也只需要操作4个指针，所以最终操作（返回找到的数据或者插入/删除节点）非常快。下面的系列团显示了如何将节点“3”添加到已经包含节点1、2、4和5的树中:

![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498484772405248/ebtree-1245.png)![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498375418511362/ebtree-1245252B3a.png)![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498607702999050/ebtree-1245252B3b.png)

在一个 i7-6700K 的核上以 4.4 GHz 频率运行，在一个包含 1 百万节点、深度超过 20 层，所有被访问节点都是 L1 高速缓存中热数据的树上进行一次查找耗时 40ns，而查找不存在的随机值造成大多数缓存未命中的耗时则高达 600ns。这限制最坏情况下（例如缓存被有意攻击）极限为每秒 167 万次查找。

这个比率很有意思，因为这表明在部分场景中，一个线程也许希望受益于常用键的快速查找（例如耗时 40ns 查找一个属于最近通信的包的源ip地址），而不被其他必须插入一些随机值的线程锁住整个树结构 600ns。

上述由 1% 写操作组成的工作负载在最差条件下，单核机器总计 2200 万每秒的查找有 87% 时间耗费在读上，13% 在写上。这意味着即便所有的读请求能在不限数量的内核上并行化，能实现的最大性能在每秒 1.6 亿次查找左右，或少于单核容量的 8 倍。

下图是一个图形表示，说明了单核与 8 核两台机器，在最差工作负载场景中时间是如何被花在快读与慢写上的，可以看到 8 核机器仅有单核机器的 4 倍速度，过程中它在持有写锁的情况下，48% 的总 CPU 资源耗费在等待上。每 1095ns 执行 100 次操作，即每秒 910 万次操作

![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498649495306242/lock-rw-1c8c.png?width=344&height=300)

注意，该问题实际上并不止存在于树结构上。它还会在一定程度上影响哈希表等其他结构。哈希表是一个由链表构成的数组，其单位元素为链表头。大部分时候元素不是有序的，所以会有值被添加到链表头（或表尾），并且可以在简单插入期间锁定链表头（或根据需要锁定整个哈希表）。然而考虑这里一个特定的场景：如果你想插入一个确保唯一的元素（正如缓存常做的那样），你需要原子操作查找并插入新值，预算同样的问题出现了，不同的是这次的最差查找时间为 O(N)：


![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498680075845632/hash-table.png?width=376&height=301)


## 设计解决方案

由于大部分时间用于在存储结构（树、表等等）上查找相应的位置，而且这种查找操作本身并不会修改结构，因而此时阻止对结构的读访问似乎是没有意义的。理想情况下，我们希望拥有仅在查找结束后将锁从读“升级”到写的能力。该锁升级操作必须有一定的限制，因为没有限制意味着所有的读锁都能被升级到写锁，而它们可能被阻塞在其它某个写操作正执行修改的位置上。

然而大部分时候我们能够知道某些查找是为写操作做准备的。因此我们可以设想一个不排斥读操作，且同一时刻在原子层面上保证当所有写操作结束，其本身能够升级到写锁的中间锁状态。

这正是渐进式锁（plock）的初始设计所做的。新的中间锁状态命名为“S”，来源于“seek”，仅在对存储结构进行查找而不做任何修改的情况下使用。就像 W 锁一样，单个 S 锁可以一次持有，W 锁和 R 锁互斥，而 S 锁和 R 锁兼容。

上文提到的最差情况小，我们考虑 600ns 插入一个键的时间中有 595ns 用于查找而 5ns 用于更新指针，因此这时整个树每 600ns 只有 5ns 的时间被锁，600ns 中的 595ns 可用于并行发生的读取操作。于是整体顺序变成如下情况：

- 一个速度较慢的写操作开始在持有 S 锁条件下对整棵树进行查找。
- 其他并行线程继续他们的读操作。在同一时间片中，每个线程约启动 15 个读操作，每个耗时约 40 ns。
- 当写操作找到他们想要插入新节点的位置时，其持有的 S 锁升级到 W 锁。
- 其他线程停止启动读操作，并完成已启动的读操作。
- 写操作等待所有的读操作完成。最差情况下等待一次完整读操作，即最多 40ns（平均 20ns ）。
- 最多 635ns 之后，写操作可以开始对树结构执行真正的修改。
- 最多 640ns 之后，写操作完成，锁被移除，系统再次开始接受读请求。

合计 8 个核都能保持忙碌状态处理 99% 比率的读请求与 1% 比率的写请求。由于 595 - 635ns 之间 S 锁都由其中 1 个核持有，7 个核每 40ns 能处理 105 个读操作。

读操作在获得互斥访问权前需要等待 0 - 40ns，即平均 20ns。因此每个写操作以及并行读操作拥有 620ns 的时间，这意味着每 620ns 执行100次操作，每秒 1.61 亿的操作吞吐量，或者说是比上述使用读/写锁快 77% 的速度：

![Alt text](https://media.discordapp.net/attachments/420095259125743637/462498705933598720/lock-rsw-8c.png?width=400&height=292)


## 初步实现

为了不重犯经典读/写锁存在的错误，单一原子操作能否被用于获取锁是重点问题。考虑到需要始终知道读操作的数量，锁需要大到能存储最大数量的读操作值，以及和其他锁一样的状态信息，这些需要存储到一个单词大小的内存当中用以保证原子级别访问的可能性。

通常锁的实现会使用 [compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap)（例如 x86 中的 **CMPXCHG** ）来在操作前检查是否存在竞争状态。数量较少时使用 compare-and-swap 非常好，但在处理一个由读操作数量决定的，变化非常之快的值时并不方便，因为失败的次数会极多，对部分用户来说甚至会有没成功过一次的情况出现。

在 x86 系统中有一个对读操作计数很便捷的 **XADD** 操作，但它容易溢出，而且对写操作计数并不方便。虽然当前条件下我们知道永远不会同时有很多写请求者，但是安全的设计还是要求能够让所有线程在这种情况下并行竞争。 

于是这里就有了将一个整型数分为 3 个部分的想法：

- 一部分作为读操作计数器
- 一部分作为 S 锁的请求数量
- 一部分作为 W 锁的请求数量

将一个 64 位整型数分为 3 等份，可以为每部分分配21位，从而支持多达 2097151 个线程而不会导致溢出。在 32 位系统上只能支持 1023 个线程而不会溢出，这一限制使得此方案对某些现有软件来说肯定会不适合，所以仍然需要改进。

如上述所言，W 锁需要与其他所有锁状态互斥，S 锁需要与 S 锁和 W 锁互斥，R 锁需要与 W 锁互斥。这意味着有可能将某些值无风险地溢出到其他值中：

- 如果 R 位溢出到 S 位中，由于 S 锁比 R 锁互斥条件更严格，所以读操作的限制能得到保障。
- 如果 S 位溢出到 W 位中，由于 W 锁比 S 锁互斥条件更严格，所以 S 锁使用者的限制也能得到保障。

但是我们需要保证 W 锁不溢出，或者给它分配足够的位来计数。

所以实际上，这个限制主要受代表 W 锁的位数支配。为了获得最优性能，同时避免强迫读操作等待，W 锁与 R 锁分配的位数相等似乎非常重要。原理上来说需要给 S 锁分配 1 个位。不过考虑到对 S 的并发访问有时会导致 W 的偶然出现，偶尔会有一些读操作无用的等待。因此我们决定使用 2 个位表示 S，即在不扰乱读操作的情况下并发支持最多 3 个查找请求，以给搜索操作充分的时间。

提供以下方案：

- 在 32 位平台上：
	* 14 位分配给 W
	* 2 位分配给 S
	* 14 位分配给 R

- 在 64 位平台上：
	* 30 位分配给 W
	* 2 位分配给 S
	* 30 位分配给 R

聪明的读者会注意到在 32 位平台上分配了 30 位而在 64 位平台上分配了 62 位，这给我们留下了 2 个额外的位用于其他任何目的，特别是弹性二叉树的配置位（统一性等等）：

![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498729531015169/plock-format.png?width=400&height=140)

初始版简单的编码显示，某些组合对应的情况于实践中不存在，而更高级的编码可以帮助存储一个额外状态。

确实，在弹性二叉树上取得的进展表明，某些 O（1）操作（如节点的移除）有朝一日会被实现为不需要任何锁的原子操作。但这与锁实现有什么关系呢？将一个应用或系统设计从有锁移植到无锁并不是一项简单的操作，有时在某些尝试上迈出一小步都需要间隔好几年的时间。能够使用一个只与自身兼容、与其他锁都互斥的特殊锁状态在转换应用时会非常有帮助。

因此这里引入一个新的 “A”（源于“原子”的含义） 状态，上文提及的所有基于位的状态编码被重定义，变得更复杂一些，但仍然快速而安全，如下文所述。


## 当前实现

目前 5 个已知的渐进式锁状态：

- U（未上锁）：无人持有锁
- R（读锁）：某些用户正在读取共享资源
- S（查找锁）：允许读但其他任何人不允许查找也不允许写入
- W（写锁）：读取中，拒绝其他访问
- A（原子锁）：一些用户正在执行可能与 R / S / W 所涵盖操作不兼容的安全原子操作

U，R 与 W 是经典读/写锁设计中的三种状态。S 状态是 R 与 W 的一个中间步骤，此时仍允许读操作加入，但拒绝其他 W 与 S 的请求同时保证到 W 状态的快速升级（以及从 W 到 S 的降级）。按照“单写多读”的模型，S 锁使得某些“无锁”操作的实现变为可能，在这种模型中一个谨慎的写操作能够执行修改而不阻止读操作对数据的访问。A 状态是为主要由有序写操作组成的原子操作而设计的，它能够并发的执行但不能与 R，S 或 W 状态混用。

在另一个 S 或 W 锁已经被持有的情况下不能获取 S 锁。一旦获得 S 锁，持有者自动被授予将其升级到 W 的权利，无须检查其它写入者。而且若有必要，它还可以多次原子的持有和释放 W 锁，只是必须等待所有读操作结束。

A锁支持并发写访问，当某些原子操作可以在也支持非原子操作的结构上执行时使用。它与其他锁都互斥，比 S/W 锁（S 是“可升级性“的承诺）要弱，开始前要等待其它读操作结束。

目前锁兼容矩阵如下图所示。请求兼容的锁立刻就能获得，不兼容的锁则使请求者等待不兼容锁操作的结束。

![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498753610383360/plock-matrix.png?width=400&height=233)

一个线程可以在多种状态间转换，如下述转换图所示：

![enter image description here](https://media.discordapp.net/attachments/420095259125743637/462498779967520770/plock-states.png?width=614&height=602)

一些其他的状态间转换也许会被尝试但并不保证成功。这里有一个持有读锁时尝试将其转换为另一个种被限制只能有一个实例的锁的案例：另一个线程也许会尝试执行同样的转换，而根据定义两者不能同时成功。此外，成功转换的一方需要等待读操作完成退出，因此转换失败的另一个线程也必须要放开他的读锁避免重新尝试转换。然而，plock 的 API 提供了对这些不常见功能的便捷途径。

从 W 到 A 的转换执行以下两个步骤：

- 请求锁
- 等之前不兼容的用户离开

如果没有对象持有 W 锁，则首先执行对读锁的声明。其背后原因在于确保持有写锁正在等待读操作结束的线程不需要在新的读操作进来时被迫等待过长时间。锁实现使用的数字位从最低到最高如下：

- R：读操作的数量（包括读，查找，写）
- S：查找请求数量
- W：写请求数量

每个常规锁（R，S，W）都算作一个读数。因为这些动作都会并发的执行读操作，而能支持的并发线程最大数量取决于分配给编码读数量的位数。

由于写操作与其他活动完全互斥，他需要与读相同数量的位数确保能承受最坏场景，所有线程在几乎同一时刻请求一个写锁，导致代表写的计数值与代表读的计数值相等。

查找请求保持在低位计数上，并且这个计数位刚好位于写入计数位之后，因此如果该值溢出，则暂时溢出到写入位上，表现为请求一个互斥写锁，这允许查找位值保持在一个低水平上，技术上而言该值为 1，但允许 2 来避免无用的常规冲突中会发生的一系列上锁/解锁过程。

综上所述，我们可以得知这些：

-  **R** 锁值由 **R** 位组成
-  **S** 锁值由 **S+R** 位组成
-  **W** 锁值由 **W+S+R** 位组成
-  **A** 锁值仅由 **W** 位组成

由于 W 锁由 W+S+R 位组成，以及 S 溢出到 W，每当获得一次 W 锁 W 值与 S 值都会加一，这导致了 W 值的增长比实际写请求要快。实际上由于 S 的两位低于 W，W:S 的比值增长为 ob101=5，W 代表 W:S 的 1/4，因此每个写操作消耗 5/4 的位置。不过考虑到 5 不能被 2 整除，我们保证对于任何可支持的写操作数量，我们总是在 W 部分保留至少一个非空位，并且考虑到 W 锁强于 S 锁，其他访问类型也能被保护。

在 W:S 中只设置了 W 位看起来似乎会与 A 锁混淆，然而 A 锁不占 R 位，并且与常规锁是互斥的，所以这种情况也并不模糊。

实际上，我们应该明白所有的状态在锁当中都是可观测的，包括下图所示的转换状态，“N”表示 _任何比0大_ 的数，“M”表示 _比1大（大多数）_ 。“X”表示 _包括0在内的任何数_ ：

![](https://media.discordapp.net/attachments/420095259125743637/462498802289475584/plock-map.png?width=400&height=198)

应具体要求，锁能在多个状态间升级和降级：

- **U⇔A** : pl_take_a() / pl_drop_a()
	* （加/减 W）
- **U⇔R** : pl_take_r() / pl_drop_r()
	* （加/减 R）
- **U⇔S** : pl_take_s() / pl_drop_s()
	* （加/减 S+R）
- **U⇔W** : pl_take_w() / pl_drop_w()
	* （加/减 W+S+R)
- **S⇔W** : pl_stow() / pl_wtos()
	* （加/减 W）
- **S⇒R** : pl_stor()
	* （减 S)
- **W⇒R** : pl_wtor()
	* （减 W+S）

这就是 **XADD** 指令突出表现的地方了：它通过仔细设置常量值，可以使用单个原子指令获取或升级任何锁。而与此同时当前值也已被获取，请求者可以检查是否存在冲突决定要不要回滚变化。解除，降级锁或取消一个失败的取锁尝试只需要原子级的减法。

为避免冲突，请求者需要定期通过读取锁值来观察其状态，一旦锁的状态看起来与请求兼容则重试。非常重要的一点是绝不能在不先读取值的情况下再次取锁，因为 CPU 缓存是这样工作的：对共享内存位置（锁在此处）的每次写入尝试都将向所有其他缓存广播请求，以刷新其挂起的更改并无效化缓存行。如果锁与锁保护下的修改在内存中的位置相近，重复尝试对锁的写请求会严重拉低其它持有锁的线程的处理速度，因为缓存行会在竞争线程之间弹来弹去。

相对来说，仅对锁进行读取，其他缓存也必须要洗牌挂起的变化，但不需要无效缓存行。这样也能正常工作，但运行速度不像在没有这种操作的场合那么快。

为了改善现状，渐进式锁实现了指数退避算法：每个失败的尝试都会使线程必须等待两倍于之前长度的时间之后才能重试。这种在锁和网络协议中使用的众所周知的方法具有三个直接的好处：

- 它限制了对当前持有锁线程的扰乱程度
- 避免了使用并发线程时的缓存风暴
- 通过改善请求尝试的随机性，为所有线程提供了更好的公平性

线程处于等待状态时，它会使用一些特定的 CPU 指令，如 x86 处理器上的 PAUSE 等，该指令将在几个周期内停止执行，并将CPU管道留给同一个 CPU 核心的另一个线程，因此进一步加速了并发多线程环境的处理速度。

## 性能测量

一个新的锁机制必须经过性能评估。名为 `lrubench` 的软件包提供使用哈希表实现的简单 LRU 缓存程序。应用程序查找随机值，并在缓存中不存在时插入该值。它允许设置并发线程数，缓存大小（可维持的键数量），键空间来控制缓存的命中率，缓存不命中的消耗（为模拟真实在单一操作上执行的循环次数，更昂贵的操作如 regex 调用），以及锁策略。

每种锁策略原则都相同：

- 在缓存中执行对一个值的查找
- 值存在则使用它
- 否则执行代价高昂的操作
- 如果它同时没有被另一个线程添加，则将这个代价高昂的操作的结果插入缓存中，否则用新条目替换这个旧条目
- 在插入的情况下，从缓存中删除可能多余的元素，使其保持所需的大小

支持 8 种锁策略：

- **pthread spinlock**：初次查找通过使用 `pthread_spinlock()` 函数提供保护。在插入值期间，查找、可能的移除以及插入动作本身都通过使用 `pthread_spinlock()` 函数来原子的执行。缓存上不存在并行操作。

- **pthread rwlock**：初次查找通过使用`pthread_rwlock_rdlock()` 函数提供保护。在插入值期间，查找、可能的移除以及插入动作本身都通过使用 `pthread_rwlock_rdlock()` 函数来原子的执行。初始查找可能会并行执行但不会与二次查找并行。

- **pthread W lock**：初次查找通过使用 W lock 提供保护。在插入值期间，查找、可能的移除以及插入动作本身都通过使用 W lock 来原子的执行。缓存上不存在并行操作。相当于 pthread spinlock 策略。

- **pthread S lock**：初次查找通过使用 S lock 提供保护。在插入值期间，查找、可能的移除以及插入动作本身都通过使用 S lock 来原子的执行。缓存上不存在并行操作。相当于上述 pthread W lock 策略但会更快一些，因为 S lock 不需要检查读操作。这是 plock 中最接近 pthread spinlock 的替代方案。

- **pthread R+W lock**：初次查找通过使用 R lock 提供保护。在插入值期间，查找、可能的移除以及插入动作本身都通过使用 S lock 来原子的执行。初始查找可能会并行执行但不会与二次查找并行。相当于上述 pthread rwlock 策略，但由于单一原子操作的缘故要快很多。

- **pthread R+S>W lock**：初次查找通过使用 R lock 提供保护。在插入值期间，查找使用 S lock 执行，可能的移除以及插入动作本身都通过将 S lock 升级到 W lock 来原子的执行。初始查找可能并行执行，也可能与二次查找并行执行。同一时间只能有一个二次查找在执行，初始查找仅在末尾的短暂的写入期间被阻塞。

- **pthread R+R>S>W lock**：初次查找通过使用 R lock 提供保护。在插入值期间，查找使用 R lock 执行，可能的移除以及插入动作本身都通过尝试将 R lock 升级到 S lock，或为避免冲突用 S lock 重复一次操作来原子的执行。然后 S lock 升级到 W lock，操作完成。初始查找可能并行执行，也可能与二次查找并行执行。除非写访问权限已被获取，导致二次查找被迫单独执行，否则二次查找也可能并行执行。

- **pthread R+R>W lock**：初次查找通过使用 R lock 提供保护。在插入值期间，查找使用 R lock 执行，可能的移除以及插入动作本身都通过尝试将 R lock 升级到 W lock，或为避免冲突用 W lock 重复一次操作来原子的执行。初始查找可能并行执行，也可能与二次查找并行执行。除非写访问权限已被获取，导致二次查找被迫单独执行，否则二次查找也可能并行执行。

（P.S 这里几种锁策略文字描述不清晰，译者补充如下一个表格辅助理解）

| |first lookup|insertion-lookup|insertion/removal|repeat|parallel|compare|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|pthread spinlock|pthread_spinlock()|pthread_spinlock()|pthread_spinlock()|-|No parallel|-|
|pthread rwlock|pthread_rwlock_rdlock()|pthread_rwlock_rdlock()|pthread_rwlock_rdlock()|-|Not the second ones|-|
|plock W lock|W lock|W lock|W lock|-|No parallel|pthread spinlock|
|plock S lock|S lock|S lock|S lock|-|No parallel|pthread spinlock(closet), W lock|
|plock R+W lock|R lock|W lock|W lock|W lock|Not the second ones|pthread rwlock, faster|
|plock R+S>W lock|R lock|S lock|S->W|-|May be, as well as the second ones|-|
|plock R+R>S>W lock|R lock|R lock|R->S|S lock, then S->W|May be, as well as the second ones|-|
|plock R+R>W lock|R lock|R lock|R->W|W lock|May be, as well as the second ones|-|

该程序在 2.5 GHz，开启了超线程设置的 12 核 Xeon E5-2680v3 上运行。

下图表示了使用这些不同锁策略下缓存查找操作每秒可达的次数，设置了 6 个不同的缓存命中率（50%, 80%, 90%, 95%, 98%, 99%），3 个不同的缓存缓存未命中成本（30, 100, 300 `snprintf()` 循环），以及一系列线程数。图的左侧每个 CPU 核（无超线程）只用一个线程，图右侧每个 CPU 核（超线程）使用两个线程。

低未命中损失（30 循环）

![1-1-1](https://media.discordapp.net/attachments/420095259125743637/462498824397783042/hit50-cost30.png)![1-1-2](https://media.discordapp.net/attachments/420095259125743637/462498861332824094/hit80-cost30.png)

![1-2-1](https://media.discordapp.net/attachments/420095259125743637/462498888994258964/hit90-cost30.png)![1-2-2](https://media.discordapp.net/attachments/420095259125743637/462498913384005632/hit95-cost30.png)

![1-3-1](https://media.discordapp.net/attachments/420095259125743637/462498941712203776/hit98-cost30.png)![1-3-2](https://media.discordapp.net/attachments/420095259125743637/462498964160380929/hit99-cost30.png)

中未命中损失（100 循环）

![2-1-1](https://media.discordapp.net/attachments/420095259125743637/462498990651604994/hit50-cost100.png)![2-1-2](https://media.discordapp.net/attachments/420095259125743637/462499009374978050/hit80-cost100.png)

![2-2-1](https://media.discordapp.net/attachments/420095259125743637/462499044300685322/hit90-cost100.png)![2-2-2](https://media.discordapp.net/attachments/420095259125743637/462499065981173761/hit95-cost100.png)

![2-3-1](https://media.discordapp.net/attachments/420095259125743637/462499089809014784/hit98-cost100.png)![2-3-2](https://media.discordapp.net/attachments/420095259125743637/462499109484625921/hit99-cost100.png)

高未命中损失（300 循环）

![3-1-1](https://media.discordapp.net/attachments/420095259125743637/462499131508785153/hit50-cost300.png)![3-1-2](https://media.discordapp.net/attachments/420095259125743637/462499150106460181/hit80-cost300.png)

![3-2-1](https://media.discordapp.net/attachments/420095259125743637/462499207258046471/hit90-cost300.png)![3-2-2](https://media.discordapp.net/attachments/420095259125743637/462499227935834112/hit95-cost300.png)

![3-3-1](https://media.discordapp.net/attachments/420095259125743637/462499249545019392/hit98-cost300.png)![3-3-2](https://media.discordapp.net/attachments/420095259125743637/462499271644807170/hit99-cost300.png)

毫不意外，不同锁策略之间的区别随着缓存不命中成本的上升、命中率下降而逐渐消失，因为相比争夺锁的时间，用于计算值的时间占比提高了。然而对于大量线程来说，高命中率的差异还是很惊人：24 个线程时，plock 能有读/写锁 5.5 倍，自旋锁的 8 倍快！即使是在低命中率高成本下，plock 也从不慢于 pthread lock，并会在 8-10 个线程后超过其速度。

查找时间越长，不同机制之间的差距就越大，因为在自旋锁和读/写锁的情况下，插入时必须获得互斥访问权，导致即使是偶尔插入的单个线程也会阻止所有其他线程使用缓存。这对 plock 来说都不是问题，除了在 R+R>W 模式下由于两个线程并发尝试插入条目导致升级失败的情况。这也是为什么当线程数量上升，R+R>W 的性能逐渐接近 R+W 的原因。

在这个工作负载中，R 锁的升级尝试似乎是没有意义的，因为两个插入之间存在竞争的情况非常少见。这种模式只有在当查找次数与缓存未命中率非常关键的时候，才能开始展现出一些优势。不过这种情况下失败线程必须重试升级操作，这样一来与序列化后围绕 S 锁使用之间就没有太大差别了。

在低命中率条件下，自旋锁成本比读/写锁要低（更少的操作量，平衡在 80% 与 10 个线程左右），plock 的 S 和 W 变体比自旋锁速度要稍快一些，随着线程的增多两者逐渐趋于相等。一种解释是指数退避在 plock 中的使用使其比相对的工作负载性能要好一些。但就算是在 80% 缓存命中率（即 20% 不命中）水平，所有严格锁的性能还是相似了，因此速度的提升受益于可升级锁这一点变得毫无疑问。

从图上也可以见到高命中率情况下自旋锁的的性能随着线程数量的增加逐步下降。 有看法认为这说明了自旋锁没有使用指数退避避免污染总线与缓存，但它的曲线与 plock-S 以及 plock-W 的曲线完美贴合，而 plock 是使用了指数退避的。事实上这里的原因是不一样的，简而言之，plock-S 和 plock-W 从激进的锁定 XADD 操作开始，而自旋锁使用了锁定的 CMPXCHG 操作。两种操作都无条件的发出一个总线写循环，*即便*是 *CMPXCHG*。这在 [英特尔软件开发者手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf) 中的CMPXCHG 指令描述里有所解释：“*为了简化处理器总线的接口，目标操作对象接受一个写循环而不考虑比较结果。如果比较失败，则写回目标操作对象，否则源对象被写入到目标（处理器在不产生锁定写的情况下决不产生锁定读。）*”。这些写操作对其他 CPU 的缓存有迫使无效化等负面影响，这解释了为什么性能会缓慢下降。

对 plock 来说，写之前会有一个读的尝试，这在最常见场景中对性能造成了一点小损失，但这一操作并不值得改变。归根结底这个初步读操作仅在获取 R 锁前执行，因为一般而言这个锁是在一个长期操作前获取的，两次读的消耗可以轻易的分期偿还，而且如果不这么做会增加 W 锁持有者等待最后一个读者离开的时间。

## 将渐进锁集成到程序中

plock 的 Git 树可以[到这里查阅与克隆](http://git.1wt.eu/web?p=plock.git;a=summary)。代码被设计成依赖性最小的结构。它只包含了两个 include 文件，一个将所有代码作为宏引入，另一个提供原子操作。宏的选择已经包括在代码中，因此调用可以自动用于 32 位和 64 位锁。32 位平台只能用 32 位锁但是 64 位平台两者皆可用。对多数用例来说，即便是 64 位平台似乎也不需要超过 16383 个线程，因此为了节省空间可能也会倾向于使用 32 位锁。

要使用 plock，程序只需要：

```c
#include  <plock.h>
```

基于此，为结构添加锁只需要一个整型数、一个长整型数甚至一个指针：

```c
struct cache {
	struct list head;  int plock;
};
```

锁值为 0 表示未上锁状态，这使得锁很容易初始化，因为通过 `calloc ( )` 分配的任何结构都相当于以初始化锁开始。

在不需要任何查找的情况下追加条目，只需在缓存访问操作周围使用 W 锁。

```c
struct cache_node *__cache_append(struct cache *,  const  char*);
void cache_insert(struct cache *cache,  const  char  *str)
{
	struct cache_node *node;
	pl_take_w(&cache->plock);
	node = __cache_append(cache, str);
	pl_drop_w(&cache->plock);
}
```

从缓存中读取则基本上需要在操作开始之前先获取 R 锁：

```c
struct cache_node *__cache_find(const  struct cache *,  const  char*);
int cache_find(struct cache *cache,  const  char  *str)  {
	const  struct cache_node *node;
	int ret; pl_take_r(&cache->plock);
	node = __cache_find(cache, str);
	ret = node ? node->value  :  -1;
	pl_drop_r(&cache->plock);
	return ret;
}
```

从缓存中删除条目需要在查找过程中持有 S 锁，并在最终写时将其升级到 W 锁：

```c
struct cache_node *__cache_find(const  struct cache *,  const  char*);
void __cache_free(struct cache *,  struct cache_node *);
void cache_delete(struct cache *cache,  const  char  *str)  {
	struct cache_node *node;
	pl_take_s(&cache->plock);
	node = __cache_find(cache, str);
	if  (!node)  {
		pl_drop_s(&cache->plock);
		return;
	}
	pl_stow(&cache->plock);
	__cache_free(cache, node);
	pl_drop_w(&cache->plock);
}
```

仅当条目不存在时插入值可以通过多种方式来完成（通常，如果找不到节点则取 S 锁进行查找并升级）。其中一种崩溃风险低、插入成功率高（如：日志索引）的方法是使用一个读锁并尝试升级到 W 锁，万一失败则变回 S 锁。通过这种方式许多线程能够并行插入，其操作的原子性也能够并行检查，因此整体的原子性得到保障。

```c
struct cache_node *__cache_find(const  struct cache *,  const  char*);
struct cache_node *__cache_append(struct cache *,  const  char*);
void cache_insert_unique(struct cache *cache,  const  char  *str)  {
	struct cache_node *node;
	pl_take_r(&cache->plock);
	node = __cache_find(cache, str);
	if  (node)  {
		pl_drop_r(&cache->plock);
		return;
	}
	if  (!pl_try_rtow)  {
		/* upgrade conflict, must drop R and try again */
		pl_drop_r(&cache->plock);
		pl_take_s(&cache->plock);
		node = __cache_find(cache, str);
		if  (node)  {
			pl_drop_s(&cache->plock);
			return;
		}
		pl_stow(&cache->plock);
	}
	/* here W is held */
	node = __cache_append(cache, str);
	pl_drop_w(&cache->plock);
}
```

跟 plock 一起提供的测试程序 `lrubench.c` 提供了一份关于如何使用相对自旋锁核标准读写锁而言不同锁策略的说明

## 其余相关

操作结束后还需要对锁进行一些清理工作，尤其是在以"pl_"为前缀、令人困惑的原子操作之后。它们当中的部分会被移到低一层的库中。这并不重要，因为正常来说用户是不应该直接使用那些函数的。

在 16 位系统上实现渐进式锁似乎没有意义（最多只能支持 127 个线程，或者如果为应用程序保留两个位的话则是 63 个线程），不过某些环境也许能通过复用现有结构中的孔充分利用这一设计。

为了线程间更好的公平性，我尝试了几种方法使用同样的机制实现 ticket，但并没有找到不需要执行多个原子操作就能保证 ticket 编号唯一性的方法。

似乎某种原子级别的“试验与添加（test-and-add）”操作在避免拿到锁后因为冲突回滚的情况下会非常有用。这种操作应该由“试验与设置（test-and-set）”和替代 XADD 的一种添加指令混合而成，这样添加的指令才能仅在特定位（一般是 S 和 W 位）被设置的情况下执行。然而目前这种指令不存在，考虑一些其他选择：

- HLE（硬件锁定），[Intel's TSX 扩展](https://software.intel.com/en-us/node/524022) 当中提供的新机制。其思想是定义一个代码区，在区域内锁被忽略，操作直接在 L1 缓存上尝试。如果成功则保存某些总线锁。如果失败则会使用真实锁再次进行尝试。至少在获取 R 锁时似乎可以用于 read+XADD，且避免此时不得不争夺总线锁。该指令需要实验。它的一个好处在于代码照此编写能够向后兼容旧处理器。
- RTM（限制事务内存）更复杂一些，它允许我们知道一系列操作能否在无锁情况下成功。万一失败操作会被再次重试。它与上述示例的 R->S 和 R->W 的机会型升级非常接近。这一指令也可以用于无锁查找操作后在树结构上插入或删除节点。

## 附件A: 锁的详细规则

这些锁赋予用户的权利和期望如下：

- unlocked（U）：
	* 不允许在受保护结构区做任何可见修改
	* 允许任何用户在自担风险的情况下读取保护区数据
	* 允许任何用户将锁改变到所有其他状态
- read-locked（R）：
	* 请求操作必须等待写入操作结束，然后才能被授予锁
	* 持有者可以读取保护区结构中任意位置的数据，不需要任何请求和指定
	* 其他用户可以读取保护区结构中任意位置的数据，不需要任何请求和指定
	* 其他用户也可以被授予 R 锁
	* 另一个用户也可能被授予 S 锁
	*  另一个用户可能获取 W 锁但首先要等待所有读操作结束，才能开始处理写操作
- seek-locked（S）：
	* 请求操作必须等待写入和查找操作完成，才能被授予锁
	* 持有者可以读取受保护结构区中任意位置的数据，不需要任何请求和指定
	* 其他用户可以读取受保护结构区中任意位置的数据，不需要任何请求和指定
	* 其他用户也可以被授予 R 锁
	* 持有者可以随时将锁立即升级到 W 锁，无需事先通知
	* 锁升级之后，持有者必须等待所有读操作完成，才能执行写操作
	* 持有者可以不升级并直接释放锁
	* 持有者也可以将 S 锁降级到 R 锁
- write-locked（W）
	* 请求者必须等待所有写入和查找操作结束才能被授予锁（从 S 锁升级上来的情况下这点已得到保障）
	* 一旦此锁被占有，其他成员不能尝试获取任何锁
	* 持有者可以对受保护结构区的数据进行任意修改
	* 其他用户完全没有权限访问受保护结构区
	* 持有者可以将锁降级到 S 锁
	* 持有者可以将锁降级到 R 锁
- atomic（A）
	* 请求者必须等待所有写入和查找操作结束才能被授予锁，但不排斥其他原子写入的用户
	* 一旦所有其他用户都离开，锁就会被授予
	* 持有者不保证是唯一的，其他用户可以并行持有A锁并同时执行修改
	* 持有者只能以与其自身使用兼容的方式，使用原子操作和内存屏障小心地对受保护结构区数据进行修改
	* 其他用户只能以与其自身使用兼容的方式，使用原子操作和内存屏障小心地对受保护结构区数据进行修改
	* 此类锁内包含的保护设置 100% 由应用决定。最常见被用于重启某些资源、用原子操作释放一系列指针等，所有兼容的函数都会使用此锁，而不是上面 3 种锁。

---

## 译者补充

端午节前就翻译完了，这算是第二篇正式翻译作品吧，成文字数是第一篇的差不多五倍Orz，磨磨蹭蹭到现在才发

错漏望指正

- 2018.06.30：替换原文需翻墙的 blogspot 的图片地址
- 2018.07.01：补充版权与原文信息

---

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNzg3OTkwODk0LC0xNTI0NDgzNDMsLTIxMD
Y5NzQxMywtMTM3MTg4MzA0MSwxMTU0MTQ1NjA4LDIwNzIxMTcy
MTcsMTI4MjM4MzA4LC0xNDcyODA5NTkwLC0zNzEzMzUzNDcsMj
AzOTYzMTUxNiwtMTc3MzUyNjAzMSwtMTA1OTE0OTE2MywxMjUx
MzY5MTc0LC0xODA5MDQ0NjcyLC02OTI3NzMyMTIsNTg3ODYwMT
E5LC0yMTM4NDU0NDUsLTE5NjU1MzUwODAsMTc3Njg4NzU1NCwx
NTc3MjA5OTE4XX0=
-->