# 09. 综合案例：把 OCCT 当成数据结构实验室

前面几章分别讲了容器、shape 身份、拓扑遍历、反向索引和生命周期。本章把它们组合成几个更完整的任务。你可以把这些案例当成以后写 OCCT 工具时的模板。

## 案例一：统计模型拓扑并建立编号表

目标：

```text
输入一个 TopoDS_Shape
输出 face / edge / vertex 数量
并且能按编号访问每个子形状
```

数据结构：

```cpp
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> faces;
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> edges;
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> vertices;
```

流程：

```cpp
TopExp::MapShapes(shape, TopAbs_FACE, faces);
TopExp::MapShapes(shape, TopAbs_EDGE, edges);
TopExp::MapShapes(shape, TopAbs_VERTEX, vertices);
```

为什么不用 `NCollection_Map`？因为除了去重和统计数量，我们还想：

```cpp
const TopoDS_Shape& face17 = faces.FindKey(17);
int edgeId = edges.FindIndex(edge);
```

这就是 `IndexedMap` 的价值。

## 案例二：检查边界边和非流形边

目标：

```text
找出只属于一个 face 的 edge
找出属于超过两个 face 的 edge
```

数据结构：

```cpp
NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> edgeToFaces;
```

流程：

```cpp
TopExp::MapShapesAndAncestors(
    shape,
    TopAbs_EDGE,
    TopAbs_FACE,
    edgeToFaces);

for (int i = 1; i <= edgeToFaces.Extent(); ++i)
{
  const TopoDS_Shape& edge = edgeToFaces.FindKey(i);
  const NCollection_List<TopoDS_Shape>& faces = edgeToFaces(i);

  if (faces.Extent() == 1)
  {
    AddBoundary(edge);
  }
  else if (faces.Extent() > 2)
  {
    AddNonManifold(edge);
  }
}
```

这个案例的本质是图论：

```text
edge -> incident faces
```

列表长度就是拓扑质量信号。

## 案例三：给每个 face 绑定计算属性

目标：

```text
对所有 face 计算面积、颜色、状态或材料 id
后续能通过 face 快速查到属性
```

如果属性只在本轮算法中使用，且 face 已经有编号，可以用数组：

```cpp
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> faces;
TopExp::MapShapes(shape, TopAbs_FACE, faces);

NCollection_Array1<double> areas(1, faces.Extent());
for (int i = 1; i <= faces.Extent(); ++i)
{
  areas.SetValue(i, ComputeArea(faces.FindKey(i)));
}
```

如果外部经常拿一个 `TopoDS_Shape` 来查属性，则用 `DataMap`：

```cpp
NCollection_DataMap<TopoDS_Shape, double, TopTools_ShapeMapHasher> faceArea;

for (int i = 1; i <= faces.Extent(); ++i)
{
  const TopoDS_Shape& face = faces.FindKey(i);
  faceArea.Bind(face, ComputeArea(face));
}
```

选择原则：

```text
算法内部高频循环：IndexedMap + Array1
外部按对象查询：DataMap
```

## 案例四：维护修改历史

目标：

```text
一个操作把原 shape 切分或替换后
能回答 original -> results
也能回答 result -> originals
```

数据结构：

```cpp
NCollection_DataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> images;

NCollection_DataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> origins;
```

追加一个关系：

```cpp
void AddHistory(const TopoDS_Shape& original,
                const TopoDS_Shape& result)
{
  images.Bound(original, NCollection_List<TopoDS_Shape>())
        ->Append(result);

  origins.Bound(result, NCollection_List<TopoDS_Shape>())
         ->Append(original);
}
```

这就是 `BOPAlgo_Builder` 里 images/origins 思想的简化版。

## 案例五：从选择对象回到业务对象

目标：

```text
用户在视图里选中 AIS_InteractiveObject
程序找到它的业务状态
```

数据结构类似 `AIS_InteractiveContext`：

```cpp
NCollection_DataMap<
    occ::handle<AIS_InteractiveObject>,
    occ::handle<AIS_GlobalStatus>> objects;
```

如果你的应用有自己的业务对象，也可以写成：

```cpp
NCollection_DataMap<
    occ::handle<AIS_InteractiveObject>,
    MyPartId> viewObjectToPart;
```

这里 key 是 handle，不是 `TopoDS_Shape`。相等性由 handle 指向的对象身份决定。它解决的是 UI 对象到业务对象的映射，不是拓扑身份问题。

## 案例六：错误定位表

目标：

```text
算法发现某些 shape 有问题
每个 shape 可能有多个 warning
```

数据结构：

```cpp
NCollection_DataMap<
    TopoDS_Shape,
    NCollection_List<occ::handle<Message_Alert>>,
    TopTools_ShapeMapHasher> warnings;
```

追加警告：

```cpp
warnings.Bound(shape, NCollection_List<occ::handle<Message_Alert>>())
        ->Append(alert);
```

这和 `shape -> list` 的模式完全一致。区别只是 value 列表里放的是引用计数对象。

## 一张选择表

| 任务 | 推荐结构 | 原因 |
| --- | --- | --- |
| 去重 | `NCollection_Map` | 只关心是否出现 |
| 编号 | `NCollection_IndexedMap` | 同时支持 key 和 index |
| key 查属性 | `NCollection_DataMap` | 直接表达字典 |
| key 查多个对象 | `DataMap<Key, NCollection_List<Value>>` | 一对多 |
| key 查多个对象且要编号 | `IndexedDataMap<Key, List<Value>>` | 反向索引 |
| 高频数组计算 | `IndexedMap + Array1` | 对象世界转整数世界 |
| 对象生命周期共享 | `occ::handle<T>` | 侵入式引用计数 |

## 本章小结

OCCT 的数据结构套路可以浓缩为一句话：

```text
先用合适的 identity 找到对象，再用 map/index/list/array 表达关系和状态。
```

只要你能把一个需求翻译成 key、value、index、list、array，OCCT 的长类名就会变成很清楚的工程词汇。
