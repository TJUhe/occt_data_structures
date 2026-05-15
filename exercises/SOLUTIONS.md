# 练习题参考答案

这些答案不是唯一写法，尤其是设计题。重点是能说清楚 key、value、index、相等性和生命周期。

## 01. 类型翻译

类型：

```cpp
NCollection_DataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher>
```

含义：

```text
TopoDS_Shape -> 一组 TopoDS_Shape
```

它是一张“一对多”的哈希表。key 是一个 shape，value 是 shape 列表。`TopTools_ShapeMapHasher` 决定 key 的相等性：默认按 `IsSame` 判断，即比较 `TShape + Location`，忽略 `Orientation`。

如果 key 是原始 edge，value 可能表示：

```text
原始 edge -> 被切分、替换或生成出来的结果 edge 列表
```

例如布尔运算中：

```text
E -> [E1, E2, E3]
```

如果 key 是结果 face，value 可能表示：

```text
结果 face -> 它来自哪些原始 face
```

这就是 `origins` 这类反向映射。

## 02. Shape 相等性

`TopoDS_Shape` 的三层相等语义：

| 方法 | 比较 TShape | 比较 Location | 比较 Orientation | 含义 |
| --- | --- | --- | --- | --- |
| `IsPartner` | 是 | 否 | 否 | 同一个底层拓扑 |
| `IsSame` | 是 | 是 | 否 | 同一个空间位置的拓扑，忽略方向 |
| `IsEqual` | 是 | 是 | 是 | 完全相同 |

所以：

- `IsPartner` 忽略 Location，也忽略 Orientation。
- `IsSame` 忽略 Orientation。
- `IsEqual` 三者都不忽略。
- `TopTools_ShapeMapHasher` 使用 `IsSame` 作为相等判断。

如果要统计一条 wire 中方向不同的同一条 edge 出现次数，默认 hasher 不足够。因为 `TopTools_ShapeMapHasher` 忽略 orientation，会把同一条 edge 的 forward/reversed 版本当作同一个 key。

更合适的做法：

- 如果需要按方向区分，使用能体现 `IsEqual` 语义的自定义 key/hasher。
- 或者不用 map 去重，遍历时直接读取每次出现的 `Orientation()`。

## 03. 手写反向索引伪代码

简化版：

```cpp
void MapShapesAndAncestors(root, childType, ancestorType, result)
{
  for each ancestor in Explore(root, ancestorType)
  {
    for each child in Explore(ancestor, childType)
    {
      int index = result.FindIndex(child);
      if (index == 0)
      {
        NCollection_List<TopoDS_Shape> empty;
        index = result.Add(child, empty);
      }

      result(index).Append(ancestor);
    }
  }
}
```

unique 版本：

```cpp
void MapShapesAndUniqueAncestors(root, childType, ancestorType, result, useOrientation)
{
  for each ancestor in Explore(root, ancestorType)
  {
    for each child in Explore(ancestor, childType)
    {
      int index = result.FindIndex(child);
      if (index == 0)
      {
        NCollection_List<TopoDS_Shape> empty;
        index = result.Add(child, empty);
      }

      bool exists = false;
      for each oldAncestor in result(index)
      {
        if (useOrientation)
        {
          exists = oldAncestor.IsEqual(ancestor);
        }
        else
        {
          exists = oldAncestor.IsSame(ancestor);
        }

        if (exists)
        {
          break;
        }
      }

      if (!exists)
      {
        result(index).Append(ancestor);
      }
    }
  }
}
```

数据结构解释：

```text
child shape -> list of ancestor shapes
```

这就是图里的反向邻接表。

## 04. IndexedMap 适用性判断

1. 判断一个 face 是否已经加入结果集合。

推荐：`NCollection_Map<TopoDS_Shape, TopTools_ShapeMapHasher>`。

理由：只需要 membership test，不需要编号和值。

2. 给所有 edge 编号，然后用数组保存每条 edge 的长度。

推荐：

```cpp
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> edgeMap;
NCollection_Array1<double> edgeLengths;
```

理由：`IndexedMap` 负责 `edge -> id`，`Array1` 负责 `id -> length`。

3. 从 vertex 找所有相邻 edge。

推荐：

```cpp
NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher>
```

含义：

```text
vertex -> list<edge>
```

理由：这是反向索引，还需要稳定编号遍历所有 vertex。

4. 从 AIS 交互对象找到显示状态。

推荐：

```cpp
NCollection_DataMap<
    occ::handle<AIS_InteractiveObject>,
    occ::handle<AIS_GlobalStatus>>
```

理由：这是普通 key-value 查询，不一定需要稳定编号。

5. 保存所有输入 shape，保持插入顺序并允许重复。

推荐：`NCollection_List<TopoDS_Shape>`。

理由：允许重复且保持插入顺序，map 类容器会去重，不合适。

## 05. HLR 编号压缩

可以设计成：

```cpp
struct EdgeVisibilityData
{
  double length = 0.0;
  bool visible = false;
  int adjacentFaces = 0;
};

NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> edgeMap;
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> faceMap;

NCollection_Array1<EdgeVisibilityData> edgeData(1, edgeMap.Extent());
```

如果还需要从 edge 找相邻 face：

```cpp
NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> edgeToFaces;
```

完整流程：

```text
1. TopExp::MapShapes(shape, TopAbs_EDGE, edgeMap)
2. TopExp::MapShapes(shape, TopAbs_FACE, faceMap)
3. TopExp::MapShapesAndAncestors(shape, TopAbs_EDGE, TopAbs_FACE, edgeToFaces)
4. edge id -> edgeData(id)
```

这种设计的好处：

- `edgeMap` 解决对象到编号。
- `edgeData` 解决高频数组访问。
- `edgeToFaces` 解决拓扑邻接关系。

## 06. images/origins 双向关系

已知：

```text
F1 -> A, B
F2 -> B, C
```

`images` 映射：

```text
images[F1] = [A, B]
images[F2] = [B, C]
```

`origins` 映射：

```text
origins[A] = [F1]
origins[B] = [F1, F2]
origins[C] = [F2]
```

为什么两张表都需要：

- `images` 适合回答“这个原始 shape 变成了什么？”
- `origins` 适合回答“这个结果 shape 来自哪里？”

如果只保存 `images`，要回答 `origins[B]` 就必须扫描所有原始 shape 的结果列表，成本更高，也更容易漏掉关系。布尔运算、属性传播、选择高亮、错误追踪都经常需要双向查询。

## 07. 句柄生命周期

`opencascade::handle<T>` 是侵入式引用计数智能指针。引用计数存在被管理对象内部，也就是 `Standard_Transient::myRefCount_`。

handle 拷贝时：

```text
新 handle 指向同一对象
对象内部引用计数 +1
```

handle 析构或重新赋值离开原对象时：

```text
对象内部引用计数 -1
如果计数降到 0，调用 Delete()
```

为什么是 intrusive smart pointer：

```text
引用计数不在外部 control block 中，而在对象自身 Standard_Transient 内部。
```

容器里保存 `occ::handle<T>` 时：

- 插入或复制 handle 会增加引用计数。
- 删除容器节点会析构节点里的 handle，引用计数减少。
- 如果这是最后一个 handle，对象会被释放。

注意：不要长期保存从 handle 中取出的裸指针，除非你能保证有 handle 继续持有对象。

## 08. 源码阅读报告示例

这里以 `TopExp.cxx` 为例。

看到的容器类型：

```cpp
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher>
NCollection_Map<TopoDS_Shape, TopTools_ShapeMapHasher>
NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher>
```

语义：

```text
IndexedMap<Shape>:
  shape -> stable index
  用于收集所有 face/edge/vertex 并去重编号。

Map<Shape>:
  shape set
  用于临时去重，例如判断 vertex 是否已经处理。

IndexedDataMap<Shape, List<Shape>>:
  child shape -> list of ancestor shapes
  用于 MapShapesAndAncestors / MapShapesAndUniqueAncestors。
```

解决的算法问题：

- `TopExp_Explorer` 负责遍历拓扑树。
- `IndexedMap` 负责把遍历结果去重并编号。
- `IndexedDataMap` 负责建立反向邻接表。

最容易出错的问题：

- `TopTools_ShapeMapHasher` 使用 `IsSame`，默认忽略 orientation。
- 如果算法需要区分同一条 edge 的 forward/reversed 方向，必须额外检查 `Orientation()` 或使用更严格的相等语义。
- `MapShapesAndUniqueAncestors` 里的 `useOrientation` 参数要按业务需求选择。

如果选择 `BOPAlgo_Builder_1.cxx`，阅读报告可以围绕：

```text
myImages: original -> generated/modified list
myOrigins: result -> origin list
myShapesSD: shape -> same-domain representative
aMFence: local duplicate guard
```

如果选择 `AIS_InteractiveContext.hxx`，阅读报告可以围绕：

```text
myObjects: interactive object -> global status
mySelection: current selection object
myDetectedSeq: detected owner sequence/index order
```

## 09. 完整小工具设计

目标：`AnalyzeShapeTopology(shape)` 输出：

- face / edge / vertex 数量。
- 边界边列表。
- 非流形边列表。
- 每个 face 的相邻 face 数量。

推荐数据结构：

```cpp
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> faces;
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> edges;
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> vertices;

NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> edgeToFaces;

NCollection_Array1<int> faceNeighborCounts(1, faces.Extent());

NCollection_List<TopoDS_Shape> boundaryEdges;
NCollection_List<TopoDS_Shape> nonManifoldEdges;
```

流程：

```cpp
TopExp::MapShapes(shape, TopAbs_FACE, faces);
TopExp::MapShapes(shape, TopAbs_EDGE, edges);
TopExp::MapShapes(shape, TopAbs_VERTEX, vertices);

TopExp::MapShapesAndAncestors(
    shape,
    TopAbs_EDGE,
    TopAbs_FACE,
    edgeToFaces);
```

统计边界/非流形边：

```cpp
for (int i = 1; i <= edgeToFaces.Extent(); ++i)
{
  const TopoDS_Shape& edge = edgeToFaces.FindKey(i);
  const int nbFaces = edgeToFaces(i).Extent();

  if (nbFaces == 1)
  {
    boundaryEdges.Append(edge);
  }
  else if (nbFaces > 2)
  {
    nonManifoldEdges.Append(edge);
  }
}
```

统计每个 face 的相邻 face 数量：

一种简单做法是先准备：

```cpp
NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> faceToNeighborFaces;
```

然后遍历 `edgeToFaces`：

```cpp
for each edge -> facesOnEdge
{
  if facesOnEdge has exactly two faces: F1, F2
  {
    AddUnique(faceToNeighborFaces[F1], F2);
    AddUnique(faceToNeighborFaces[F2], F1);
  }
}
```

最后：

```cpp
for (int i = 1; i <= faces.Extent(); ++i)
{
  const TopoDS_Shape& face = faces.FindKey(i);
  const auto* neighbors = faceToNeighborFaces.Seek(face);
  faceNeighborCounts.SetValue(i, neighbors ? neighbors->Extent() : 0);
}
```

哪些地方用 `IndexedMap`：

- face / edge / vertex 收集、去重、编号。

哪些地方用 `IndexedDataMap`：

- `edge -> faces`。
- 可选的 `face -> neighbor faces`。

哪些地方适合用 `Array1`：

- `face index -> neighbor count`。
- `edge index -> edge quality/status`。

`TopTools_ShapeMapHasher` 是否适合：

- 对拓扑质量检查通常适合，因为共享边即使方向不同，也应该算同一条边。
- 如果要分析 wire 中边的方向出现次数，则不适合，因为它忽略 orientation。

## 10. 图解复述

示例一：`02_basemap.svg`

key：

```text
Map / DataMap / IndexedMap 中的 key，例如 TopoDS_Shape。
```

value 或 index：

```text
DataMap 节点里有 value。
IndexedMap 节点里有 index。
IndexedDataMap 节点里同时有 index 和 value。
```

数据流：

```text
key -> HashCode -> myData1 bucket -> node
index -> myData2[index - 1] -> node
```

STL 类比：

```text
NCollection_Map        ~= unordered_set
NCollection_DataMap    ~= unordered_map
NCollection_IndexedMap ~= unordered_set + vector<Node*>
```

示例二：`06_boolean_images_origins.svg`

key：

```text
images 的 key 是原始 shape。
origins 的 key 是结果 shape。
```

value：

```text
images 的 value 是结果 shape 列表。
origins 的 value 是来源 shape 列表。
```

数据流：

```text
原始 face F1 -> 结果 A, B
结果 B -> 来源 F1, F2
```

STL 类比：

```cpp
std::unordered_map<Shape, std::vector<Shape>> images;
std::unordered_map<Shape, std::vector<Shape>> origins;
```

但在 OCCT 中，`Shape` 的 hash/equality 必须使用符合拓扑语义的 hasher，通常是 `TopTools_ShapeMapHasher`。

示例三：`07_hlr_index_arrays.svg`

key：

```text
TopoDS_Edge 或 TopoDS_Face。
```

index/value：

```text
IndexedMap 给 edge/face 分配整数 id。
Array1 用这个 id 保存 HLRBRep_EdgeData / HLRBRep_FaceData。
```

数据流：

```text
TopoDS_Edge -> edge id -> myEData(edge id)
```

STL 类比：

```cpp
std::unordered_map<Shape, int> edgeToId;
std::vector<EdgeData> edgeData;
```

OCCT 使用 `NCollection_IndexedMap + NCollection_Array1`，是为了贴合自己的 allocator、1-based index 习惯和传统 API。
