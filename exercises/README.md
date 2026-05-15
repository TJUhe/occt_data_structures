# 练习题

这些练习不要求你完整编译 OCCT。它们更像源码阅读和小设计题，目标是把数据结构知识迁移到 CAD 内核场景中。

参考答案见 [SOLUTIONS.md](SOLUTIONS.md)。建议先自己写一版，再对照答案检查 key/value/index 语义是否清楚。

## 01. 类型翻译

把下面类型翻译成“key -> value”的中文含义：

```cpp
NCollection_DataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher>
```

继续回答：

- 如果 key 是原始 edge，value 可能表示什么？
- 如果 key 是结果 face，value 又可能表示什么？

## 02. Shape 相等性

阅读 `TopoDS_Shape.hxx` 中 `IsPartner`、`IsSame`、`IsEqual` 的注释和实现。

回答：

- 哪一个忽略 Location？
- 哪一个忽略 Orientation？
- `TopTools_ShapeMapHasher` 使用哪一个？
- 如果你要统计一条 wire 中方向不同的同一条 edge 出现次数，默认 hasher 是否足够？

## 03. 手写反向索引伪代码

用伪代码写一个简化版：

```cpp
MapShapesAndAncestors(root, childType, ancestorType, result)
```

要求：

- `result` 是 `child -> list<ancestor>`。
- 遇到新 child 时创建空列表。
- 把 ancestor 追加到 child 的列表。
- 再写一个 unique 版本，避免重复 ancestor。

## 04. IndexedMap 适用性判断

下面场景应该选 `Map`、`DataMap`、`IndexedMap` 还是 `IndexedDataMap`？

1. 判断一个 face 是否已经加入结果集合。
2. 给所有 edge 编号，然后用数组保存每条 edge 的长度。
3. 从 vertex 找所有相邻 edge。
4. 从 AIS 交互对象找到显示状态。
5. 保存所有输入 shape，保持插入顺序并允许重复。

## 05. HLR 编号压缩

仿照 `HLRBRep_Data` 的思路，设计一个“网格边可见性分析”的数据结构：

```text
edge shape -> edge index
edge index -> length, visible, adjacentFaces
face shape -> face index
```

写出你会使用的 OCCT 容器类型。

## 06. images/origins 双向关系

假设布尔运算后：

```text
原 face F1 -> 结果 face A, B
原 face F2 -> 结果 face B, C
```

请写出：

- `images` 映射。
- `origins` 映射。
- 为什么同时保存两张表比只保存一张更方便？

## 07. 句柄生命周期

阅读 `Standard_Handle.hxx` 中 `Assign` 和析构相关逻辑。

回答：

- handle 拷贝时引用计数如何变化？
- handle 析构时引用计数如何变化？
- 为什么说它是 intrusive smart pointer？
- 容器里保存 `occ::handle<T>` 时，删除容器节点会发生什么？

## 08. 源码阅读报告

选择一个文件：

```text
BOPAlgo_Builder_1.cxx
BOPAlgo_Builder_2.cxx
HLRBRep_Data.hxx
TopExp.cxx
AIS_InteractiveContext.hxx
```

写一页阅读报告，包含：

- 你看到的 3 个容器类型。
- 每个容器的 key/value 语义。
- 它们解决的算法问题。
- 你认为最容易出错的相等性或生命周期问题。

## 09. 完整小工具设计

设计一个 `AnalyzeShapeTopology(shape)` 工具，不需要写完整可编译代码，但要给出数据结构设计。

要求输出：

- face / edge / vertex 数量。
- 边界边列表。
- 非流形边列表。
- 每个 face 的相邻 face 数量。

请说明：

- 哪些地方用 `IndexedMap`。
- 哪些地方用 `IndexedDataMap`。
- 哪些地方适合用 `Array1`。
- 为什么默认 `TopTools_ShapeMapHasher` 的 `IsSame` 语义适合或不适合。

## 10. 图解复述

任选 `docs/images` 中两张图，用自己的话解释：

- 图中的 key 是什么？
- value 或 index 是什么？
- 数据流从哪里到哪里？
- 这个结构如果换成 STL，大致会对应什么容器？
