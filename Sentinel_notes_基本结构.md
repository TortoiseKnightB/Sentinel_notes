# Sentinel_notes_基本结构（图片待画）

### 主要结构名称

- Slot：**插槽**，一个实现功能的模块，类似单链表的节点
- SlotChain：一系列插槽连接成的**插槽链**（Slot1 -> Slot2 -> Slot3 -> Slot4）
  - 官方有一些固定的 Slot，自己也可以实现自定义 Slot，实现自定义的功能
  - SlotChain 类似单链表结构，每个 Slot 都能调用下一个 Slot

------

- Entry：主要用来创建资源，类似双向链表的结点
- Resource：**资源**，被调用的对象，即一些列**业务逻辑**，这些业务逻辑被保护起来执行**限流、降级、统计**等
  - Resource 与 SlotChain 全局一一对应，与线程无关
  - 线程调用 Resource 的时候，执行 Resource 对应的 SlotChain。SlotChain 中每个 Slot 的功能都被执行一次
- Context：**上下文**，存在当前线程 ThreadLocal 中。每个线程都仅有一个 Context，Resource 的真正调用者
  - 线程调用 Resource，即—— 线程中的 Context 调用 Resource
  - Context 每一次调用 Resource（无论是否同一个 Resource）都会生成一个 Entry，多次调用会生成多个 Entry，多个 Entry 组成 **Entry 链**。Entry 链类似双向链表，每个 Entry 中保存了上一个 Entry 与下一个 Entry ，记录了 Context 调用Resource 的顺序
  - `Context.getCurEntry()` 指向当前调用的 Resource 所在的 Entry
  - 不同的 Context 可调用同一个 Resource
  - 一个 Context 可对同一个 Resource 进行多次调用，生成不同的 Entry

------

- Node：节点，用来保存资源实时统计数据，为 Sentinel 操作提供依据。Node 关系图如下

![2021-08-25_171300](C:\Users\Administrator\Desktop\1.png)

- DefaultNode：链路节点，与 Resource 对应，统计调用链路上某个 Resource 的数据
  - Context 对 Resource 进行调用时，会创建一个 DefaultNode 用来保存数据
  - 多个 Context 对同一个 Resource 进行调用时会创建多个 DefaultNode，分别保存其对应的 Context 的调用数据（Resource -> Map<context, DefaultNode>）
- ClusterNode：簇点，用于统计每个 Resource 全局的数据
  - 与 Resource 一一对应，不区分 Context
  - 关联同一 Resource 的不同 DefaultNode 都关联同一 ClusterNode
- EntranceNode：入口节点，特殊的链路节点，对应某个 Context 入口的所有调用数据
  - 与 Context 一一对应
- StatisticNode：最为基础的统计节点，包含秒级和分钟级两个滑动窗口结构



- 几种 Node 的维度（数目）：
  - ClusterNode 的维度是 resource
  - DefaultNode 的维度是 resource * context，存在每个 NodeSelectorSlot 的 map 里面
  - EntranceNode 的维度是 context，存在 ContextUtil 类的 contextNameNodeMap 里面
  - 来源节点（类型为 StatisticNode）的维度是 resource * origin，存在每个 ClusterNode 的 originCountMap 里面

