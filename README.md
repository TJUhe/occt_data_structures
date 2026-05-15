# OCCTDataStructuresGuide

这是一个面向 C++/CAD 内核学习者的源码阅读教程。主题不是“再实现一遍数据结构”，而是观察 Open CASCADE Technology (OCCT) 这样的工业级几何内核怎样把数组、链表、哈希表、索引映射、引用计数和遍历栈组合成真实系统。

参考源码仓库：

- GitHub: <https://github.com/Open-Cascade-SAS/OCCT>
- 本教程分析版本：`master` 上 `adm/cmake/version.cmake` 显示的 OCCT `8.0.0` 代码。
- 重要路径变化：OCCT 8.0 源码按工具包分层，例如 `src/FoundationClasses/TKernel/...`、`src/ModelingData/TKBRep/...`。很多旧教程里的 `TopTools_IndexedMapOfShape.hxx` 现在位于 `src/Deprecated/NCollectionAliases`，推荐直接使用 `NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher>`。

## 你会学到什么

1. 为什么 OCCT 没有只依赖 STL，而是保留了 `NCollection_*` 容器族。
2. `NCollection_Map`、`NCollection_DataMap`、`NCollection_IndexedMap`、`NCollection_IndexedDataMap` 的真实内部结构。
3. `TopoDS_Shape` 为什么像“轻量句柄”，以及它的相等性如何影响哈希表。
4. `TopExp_Explorer` 怎样用栈遍历拓扑树，`TopExp::MapShapes*` 怎样建立拓扑索引。
5. 布尔运算 `BOPAlgo_Builder` 如何用“原形状 -> 结果形状列表”和“结果形状 -> 来源列表”管理修改历史。
6. HLR 隐线算法怎样把 Shape 映射成稳定编号，再用数组保存边/面计算数据。
7. `Handle`/`Standard_Transient` 的侵入式引用计数为什么是 OCCT 对象图的地基。
8. 如何把普通数据结构课程里的知识迁移到 CAD 内核源码阅读中。

## 建议学习路线

1. 先读 [01_occt_map.md](docs/01_occt_map.md)，理解 OCCT 容器族的定位。
2. 再读 [02_ncollection_hash.md](docs/02_ncollection_hash.md)，把哈希桶、负载因子、双数组结构看清楚。
3. 读 [03_shape_identity.md](docs/03_shape_identity.md)，掌握 `TopoDS_Shape` 的 identity 语义。
4. 读 [04_topology_traversal.md](docs/04_topology_traversal.md)，把拓扑结构当成树/图来理解。
5. 读 [05_ancestor_index.md](docs/05_ancestor_index.md)，学习反向索引和邻接表思想。
6. 读 [06_boolean_builder.md](docs/06_boolean_builder.md)，看大型算法怎样维护 images/origins。
7. 读 [07_hlr_arrays.md](docs/07_hlr_arrays.md)，观察“IndexedMap + Array1”的编号压缩模式。
8. 读 [08_handles_lifetime.md](docs/08_handles_lifetime.md)，理解句柄、RTTI 和生命周期。
9. 最后读 [09_case_studies.md](docs/09_case_studies.md)，把所有模式串成完整任务。
10. 读 [10_std_equivalents.md](docs/10_std_equivalents.md)，把 OCCT 容器翻译成 std 思维模型。

## 目录结构

```text
.
├── README.md
├── docs/
│   ├── 01_occt_map.md
│   ├── 02_ncollection_hash.md
│   ├── 03_shape_identity.md
│   ├── 04_topology_traversal.md
│   ├── 05_ancestor_index.md
│   ├── 06_boolean_builder.md
│   ├── 07_hlr_arrays.md
│   ├── 08_handles_lifetime.md
│   ├── 09_case_studies.md
│   ├── 10_std_equivalents.md
│   └── 99_source_index.md
└── exercises/
    ├── README.md
    └── SOLUTIONS.md
```

## 核心源码速查

| 主题 | OCCT 源码路径 |
| --- | --- |
| 哈希表公共基类 | `src/FoundationClasses/TKernel/NCollection/NCollection_BaseMap.hxx` |
| 普通集合 | `src/FoundationClasses/TKernel/NCollection/NCollection_Map.hxx` |
| key-value 映射 | `src/FoundationClasses/TKernel/NCollection/NCollection_DataMap.hxx` |
| key 到稳定编号 | `src/FoundationClasses/TKernel/NCollection/NCollection_IndexedMap.hxx` |
| key 到稳定编号和值 | `src/FoundationClasses/TKernel/NCollection/NCollection_IndexedDataMap.hxx` |
| 动态数组 | `src/FoundationClasses/TKernel/NCollection/NCollection_DynamicArray.hxx` |
| Shape 哈希器 | `src/ModelingData/TKBRep/TopTools/TopTools_ShapeMapHasher.hxx` |
| Shape 轻量对象 | `src/ModelingData/TKBRep/TopoDS/TopoDS_Shape.hxx` |
| 拓扑遍历和映射 | `src/ModelingData/TKBRep/TopExp/TopExp.hxx`, `TopExp.cxx`, `TopExp_Explorer.*` |
| 布尔运算构造器 | `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Builder.hxx`, `BOPAlgo_Builder_*.cxx` |
| 隐线算法数据 | `src/ModelingAlgorithms/TKHLR/HLRBRep/HLRBRep_Data.hxx`, `HLRBRep_Data.cxx` |
| 侵入式句柄 | `src/FoundationClasses/TKernel/Standard/Standard_Handle.hxx`, `Standard_Transient.hxx` |

## 图解索引

| 章节 | 图解 |
| --- | --- |
| 01 | [OCCT 数据结构分层](docs/images/01_occt_layers.svg) |
| 02 | [NCollection BaseMap 双表结构](docs/images/02_basemap.svg) |
| 03 | [TopoDS_Shape 身份层级](docs/images/03_shape_identity.svg) |
| 04 | [TopExp Explorer DFS](docs/images/04_topexp_dfs.svg) |
| 05 | [子形状到祖先列表](docs/images/05_ancestor_index.svg) |
| 06 | [BOPAlgo images/origins](docs/images/06_boolean_images_origins.svg) |
| 07 | [HLRBRep 编号与数组](docs/images/07_hlr_index_arrays.svg) |
| 08 | [Handle 引用计数](docs/images/08_handle_lifetime.svg) |

## OCCT 与 std 对照速查

| OCCT | std 思维模型 |
| --- | --- |
| `NCollection_Map` | `std::unordered_set` |
| `NCollection_DataMap` | `std::unordered_map` |
| `NCollection_IndexedMap` | `std::unordered_map<Key, int> + std::vector<Key>` |
| `NCollection_IndexedDataMap` | `std::unordered_map<Key, int> + std::vector<pair<Key, Value>>` |
| `NCollection_List` | `std::list` |
| `NCollection_Array1` | `std::vector` 加自定义下标偏移 |
| `opencascade::handle<T>` | 侵入式 `std::shared_ptr<T>` 思维 |

完整解释见 [docs/10_std_equivalents.md](docs/10_std_equivalents.md)。

## 阅读方法

把 OCCT 当成一份“数据结构应用题集”来读。每遇到一个容器，先问四个问题：

1. key 是什么？它的相等性是否符合业务语义？
2. value 是什么？它是数据本身、编号、列表，还是反向索引？
3. 顺序重要吗？如果重要，是插入顺序、拓扑顺序还是稳定编号？
4. 这个结构服务于哪条算法路径？遍历、去重、分组、反查、缓存，还是生命周期管理？

这个问题清单比记住类名更重要。OCCT 的类名很长，但底层思路并不神秘。
