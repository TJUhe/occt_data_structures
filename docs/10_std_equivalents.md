# 10. OCCT 数据结构与 std 容器对照

这一章专门回答一个实用问题：

```text
看到一个 OCCT 数据结构时，它在 std 世界里大概对应什么？
```

先强调一个边界：这里说的是“思维模型”和“可类比的 std 组合”，不是说 OCCT 源码底层真的用这些 std 容器实现。OCCT 的 `NCollection_*` 很多是自研容器，服务于自己的 allocator、异常、迭代器风格、1-based index 习惯和历史 API。

## 总览表

| OCCT 类型 | std 思维模型 | 是否通常由 std 直接实现 | 主要用途 |
| --- | --- | --- | --- |
| `NCollection_Map<Key, Hasher>` | `std::unordered_set<Key, Hasher, Equal>` | 否 | 去重、membership test |
| `NCollection_DataMap<Key, Value, Hasher>` | `std::unordered_map<Key, Value, Hasher, Equal>` | 否 | key-value 查询 |
| `NCollection_IndexedMap<Key, Hasher>` | `std::unordered_map<Key, int> + std::vector<Key>` | 否 | key 去重并分配稳定编号 |
| `NCollection_IndexedDataMap<Key, Value, Hasher>` | `std::unordered_map<Key, int> + std::vector<pair<Key, Value>>` | 否 | key 编号，同时挂 value |
| `NCollection_List<T>` | `std::list<T>` | 否 | 链表、一对多列表 |
| `NCollection_Sequence<T>` | 接近 `std::list<T>` / `std::deque<T>` 的序列接口 | 否 | 传统 OCCT 双向序列 |
| `NCollection_Array1<T>` | `std::vector<T>` 加自定义 lower/upper bound | 否 | 1-based 或自定义下标数组 |
| `NCollection_Array2<T>` | `std::vector<T>` 扁平化二维数组 | 否 | 自定义上下界二维数组 |
| `NCollection_DynamicArray<T>` | 分块式 `std::vector<T>` / `std::deque<T>` 思维 | 否 | 引用稳定的动态增长数组 |
| `NCollection_Vector<T>` | `NCollection_DynamicArray<T>` 的旧名 | 否 | OCCT 8.0 已 deprecated |
| `NCollection_Queue<T>` | `std::queue<T>` | 否 | FIFO 队列 |
| `NCollection_Stack<T>` | `std::stack<T>` | 否 | LIFO 栈 |
| `NCollection_UBTree<Obj, Bnd>` | 空间索引树，接近 R-tree / BVH 思维 | 否 | 包围盒查询、空间过滤 |
| `NCollection_CellFilter` | spatial hash / grid index | 否 | 空间划分、邻近候选过滤 |
| `opencascade::handle<T>` | `std::shared_ptr<T>` 的侵入式版本 | 否 | `Standard_Transient` 派生对象生命周期 |
| `TopoDS_Shape` | 值类型 handle，内部近似 `shared_ptr<TShape> + Location + Orientation` | 否 | 拓扑对象轻量引用 |
| `TopExp_Explorer` | DFS iterator，内部类似 `std::vector<Iterator>` 栈 | 否 | 拓扑遍历 |
| `TopTools_ShapeMapHasher` | `std::hash<TopoDS_Shape> + custom equal` | 部分使用 `std::hash` | shape map 的 hash/equality |
| `TopTools_*` 容器别名 | 对应 `NCollection_*<TopoDS_Shape, ...>` | 否 | 旧 API 兼容 |

## 哈希集合：NCollection_Map

OCCT：

```cpp
NCollection_Map<TopoDS_Shape, TopTools_ShapeMapHasher> seen;
seen.Add(face);
if (seen.Contains(face)) {}
```

std 思维模型：

```cpp
std::unordered_set<TopoDS_Shape, ShapeHash, ShapeEqual> seen;
seen.insert(face);
if (seen.contains(face)) {}
```

差别：

- `NCollection_Map::Add()` 返回 `bool`，表示是否新插入。
- OCCT 的迭代器常用 `More()/Next()/Value()`。
- OCCT 使用自己的 bucket/node 结构和 allocator。

适用场景：

```text
只关心对象是否已经出现，不需要保存额外 value，也不需要稳定编号。
```

## 哈希字典：NCollection_DataMap

OCCT：

```cpp
NCollection_DataMap<TopoDS_Shape, double, TopTools_ShapeMapHasher> areaOfFace;
areaOfFace.Bind(face, area);
const double area = areaOfFace.Find(face);
```

std 思维模型：

```cpp
std::unordered_map<TopoDS_Shape, double, ShapeHash, ShapeEqual> areaOfFace;
areaOfFace.emplace(face, area);
const double area = areaOfFace.at(face);
```

常见 API 对照：

| OCCT | std 近似 |
| --- | --- |
| `Bind(key, value)` | `emplace(key, value)` |
| `Bound(key, defaultValue)` | `try_emplace(key, defaultValue)` 后取 value |
| `Find(key)` | `at(key)` |
| `Seek(key)` | `find(key)`，不存在返回 `end()` |
| `ChangeSeek(key)` | `find(key)` 后修改 `it->second` |
| `IsBound(key)` | `contains(key)` |

`Bound` 是 OCCT 里非常常见的写法：

```cpp
myImages.Bound(original, NCollection_List<TopoDS_Shape>())
        ->Append(result);
```

std 近似：

```cpp
images.try_emplace(original, std::vector<TopoDS_Shape>{});
images[original].push_back(result);
```

## 编号集合：NCollection_IndexedMap

OCCT：

```cpp
NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher> edges;
int id = edges.Add(edge);
const TopoDS_Shape& e = edges.FindKey(id);
int oldId = edges.FindIndex(edge);
```

std 思维模型：

```cpp
std::unordered_map<TopoDS_Shape, int, ShapeHash, ShapeEqual> keyToId;
std::vector<TopoDS_Shape> idToKey;
```

伪实现：

```cpp
int Add(const TopoDS_Shape& key)
{
  auto it = keyToId.find(key);
  if (it != keyToId.end())
  {
    return it->second;
  }

  int id = static_cast<int>(idToKey.size()) + 1;
  keyToId.emplace(key, id);
  idToKey.push_back(key);
  return id;
}

const TopoDS_Shape& FindKey(int id)
{
  return idToKey[id - 1];
}
```

关键差别：

- OCCT 内部不是简单两份容器，而是 `myData1` 哈希桶和 `myData2` 索引数组都指向同一批节点。
- 对外 index 从 1 开始。
- `RemoveLast()` 很自然，任意删除会影响编号稳定性，需要谨慎。

适用场景：

```text
先给复杂对象编号，再用编号访问 Array1 或其他数组状态。
```

## 编号字典：NCollection_IndexedDataMap

OCCT：

```cpp
NCollection_IndexedDataMap<
    TopoDS_Shape,
    NCollection_List<TopoDS_Shape>,
    TopTools_ShapeMapHasher> edgeToFaces;
```

std 思维模型：

```cpp
std::unordered_map<TopoDS_Shape, int, ShapeHash, ShapeEqual> keyToId;
std::vector<std::pair<TopoDS_Shape, std::vector<TopoDS_Shape>>> idToKeyValue;
```

它同时回答三类问题：

```text
key -> id
id -> key
id -> value
```

这就是为什么它特别适合拓扑反向索引：

```text
edge -> index -> adjacent face list
vertex -> index -> incident edge list
```

std 近似虽然能写，但会比较啰嗦；OCCT 直接把这个组合封成一个容器。

## 链表：NCollection_List

OCCT：

```cpp
NCollection_List<TopoDS_Shape> images;
images.Append(result);
```

std 思维模型：

```cpp
std::list<TopoDS_Shape> images;
images.push_back(result);
```

为什么 OCCT 大量用 list：

- 一对多关系的结果数量不固定。
- 追加频繁。
- 传统 API 习惯使用 `Iterator::More()/Next()/Value()`。

但如果你自己写现代 C++ 应用层代码，且不需要稳定节点地址，`std::vector` 往往更缓存友好。OCCT 内核代码保留 `NCollection_List` 更多是历史和接口一致性。

## 序列：NCollection_Sequence

OCCT：

```cpp
NCollection_Sequence<int> detectedSeq;
detectedSeq.Append(ownerId);
```

std 思维模型：

```cpp
std::deque<int> detectedSeq;
// 或 std::list<int>，取决于你更关心随机访问还是插入删除
```

`NCollection_Sequence` 不是简单的 `std::vector`。它更像 OCCT 传统序列：

- 支持前后插入。
- 支持按 1-based index 访问。
- 支持 split、exchange 等序列操作。

粗略类比：

```text
如果你主要顺序遍历：像 std::list
如果你主要按 index 取值：像 std::deque 的接口需求
```

## 自定义下标数组：NCollection_Array1

OCCT：

```cpp
NCollection_Array1<double> lengths(1, edges.Extent());
lengths.SetValue(edgeId, length);
```

std 思维模型：

```cpp
std::vector<double> lengths(edges.size());
lengths[edgeId - 1] = length;
```

OCCT 的关键差别：

```text
Array1 有 Lower() 和 Upper()
合法下标不一定从 0 开始
```

所以：

```cpp
NCollection_Array1<int> a(5, 10);
```

合法下标是：

```text
5, 6, 7, 8, 9, 10
```

std 近似实现可以想成：

```cpp
template<class T>
class Array1Like
{
  int lower;
  std::vector<T> data;

  T& operator()(int index)
  {
    return data[index - lower];
  }
};
```

## 二维数组：NCollection_Array2

OCCT 思维：

```text
row lower..row upper
col lower..col upper
```

std 近似：

```cpp
std::vector<T> data(rowCount * colCount);
data[(row - rowLower) * colCount + (col - colLower)]
```

它适合矩阵、表格、二维参数域缓存。和 `std::vector<std::vector<T>>` 相比，扁平化数组通常更连续、更适合数值计算。

## 动态数组：NCollection_DynamicArray

OCCT 8.0 中 `NCollection_Vector<T>` 已 deprecated，推荐 `NCollection_DynamicArray<T>`。

std 思维模型：

```text
接近 std::deque<T> 的分块增长
也承担一部分 std::vector<T> 的动态数组角色
```

它的注释强调一个特性：元素引用在增长时保持有效。这一点和 `std::vector` 不同，因为 `std::vector` 扩容可能搬迁整块内存，让引用/指针失效。

所以更准确的类比是：

```text
NCollection_DynamicArray ~= 分块 vector，引用稳定性更像 deque
```

## 队列和栈：NCollection_Queue / NCollection_Stack

概念对照很直接：

```text
NCollection_Queue<T> ~= std::queue<T>
NCollection_Stack<T> ~= std::stack<T>
```

区别仍然是 OCCT 自己的 allocator、异常和迭代器习惯。

在拓扑遍历里，`TopExp_Explorer` 没有直接暴露 `NCollection_Stack`，但内部思想就是 DFS 栈：

```text
std::vector<TopoDS_Iterator> stack
```

## 空间索引：UBTree 和 CellFilter

这类结构没有一个完全对应的标准库容器，因为 C++ 标准库没有空间索引。

可以这样类比：

| OCCT | std / 常见算法思维 |
| --- | --- |
| `NCollection_UBTree<Obj, Bnd>` | R-tree / BVH / interval tree 思维 |
| `NCollection_CellFilter` | spatial hash grid |

用途：

```text
先用包围盒或网格快速筛掉不可能相交的对象，再做精确几何计算。
```

如果强行用 std 写，通常会是：

```cpp
std::vector<ObjectWithBox> objects;
// 每次查询线性扫描所有 box
```

但这会退化成 `O(n)` 候选过滤。空间索引的意义就是把候选数量先压下来。

## 句柄：opencascade::handle

OCCT：

```cpp
occ::handle<Geom_Curve> curve;
```

std 思维模型：

```cpp
std::shared_ptr<Geom_Curve>
```

但更准确地说：

```text
opencascade::handle<T> ~= intrusive shared_ptr<T>
```

差别：

| 项目 | `opencascade::handle<T>` | `std::shared_ptr<T>` |
| --- | --- | --- |
| 引用计数位置 | 对象内部 `Standard_Transient` | 外部 control block |
| 使用对象要求 | 必须继承 `Standard_Transient` | 任意可删除对象 |
| 删除方式 | `entity->Delete()` | deleter 删除 |
| 循环引用问题 | 仍然存在 | 也存在 |

不要把 `TopoDS_Shape` 直接理解成 `shared_ptr<TopoDS_Shape>`。它是值对象，内部持有 `handle<TopoDS_TShape>`，再加 `Location` 和 `Orientation`。

## Shape 哈希器：TopTools_ShapeMapHasher

std 思维模型：

```cpp
struct ShapeHash
{
  size_t operator()(const TopoDS_Shape& s) const
  {
    return std::hash<TopoDS_Shape>{}(s);
  }
};

struct ShapeEqual
{
  bool operator()(const TopoDS_Shape& a, const TopoDS_Shape& b) const
  {
    return a.IsSame(b);
  }
};

std::unordered_set<TopoDS_Shape, ShapeHash, ShapeEqual> shapes;
```

OCCT 把 hash 和 equal 放在同一个 hasher 类型里：

```cpp
TopTools_ShapeMapHasher
```

这点和 STL 的 `unordered_map` 模板参数略有不同：STL 通常把 hash 和 equality 分成两个模板参数。

## TopTools_* 旧别名

OCCT 8.0 中很多旧名已经 deprecated。可以这样翻译：

| 旧名 | 新写法 | std 思维模型 |
| --- | --- | --- |
| `TopTools_MapOfShape` | `NCollection_Map<TopoDS_Shape, TopTools_ShapeMapHasher>` | `unordered_set<Shape>` |
| `TopTools_IndexedMapOfShape` | `NCollection_IndexedMap<TopoDS_Shape, TopTools_ShapeMapHasher>` | `unordered_map<Shape,int> + vector<Shape>` |
| `TopTools_ListOfShape` | `NCollection_List<TopoDS_Shape>` | `list<Shape>` |
| `TopTools_IndexedDataMapOfShapeListOfShape` | `NCollection_IndexedDataMap<TopoDS_Shape, NCollection_List<TopoDS_Shape>, TopTools_ShapeMapHasher>` | `unordered_map<Shape,int> + vector<pair<Shape,list<Shape>>>` |

读旧教程时，先把 `TopTools_*` 翻译成 `NCollection_*`，再用 std 思维模型理解，会舒服很多。

## 选择表：从需求反推 OCCT 和 std

| 需求 | OCCT | std 思维模型 |
| --- | --- | --- |
| 只去重 | `NCollection_Map<T>` | `unordered_set<T>` |
| key 查 value | `NCollection_DataMap<K,V>` | `unordered_map<K,V>` |
| key 需要稳定 id | `NCollection_IndexedMap<K>` | `unordered_map<K,int> + vector<K>` |
| key 需要 id 和 value | `NCollection_IndexedDataMap<K,V>` | `unordered_map<K,int> + vector<pair<K,V>>` |
| 一对多 | `DataMap<K, List<V>>` | `unordered_map<K, vector/list<V>>` |
| 按编号保存状态 | `Array1<T>` | `vector<T>` 加 index offset |
| 双向顺序序列 | `Sequence<T>` | `list<T>` / `deque<T>` |
| 引用计数对象 | `occ::handle<T>` | intrusive `shared_ptr<T>` |
| 拓扑 DFS | `TopExp_Explorer` | stack-based DFS iterator |
| 空间候选过滤 | `UBTree` / `CellFilter` | R-tree/BVH/spatial hash |

## 本章小结

最实用的记忆方式：

```text
Map             -> unordered_set
DataMap         -> unordered_map
IndexedMap      -> unordered_map + vector
IndexedDataMap  -> unordered_map + vector<pair>
List            -> list
Array1          -> vector with custom lower bound
handle          -> intrusive shared_ptr
```

但写 OCCT 代码时不要直接替换成 std。真正要看的是：

```text
OCCT API 需要什么容器？
key 的相等性是什么？
是否需要稳定编号？
是否需要 OCCT allocator / handle / 传统迭代器？
```

这些问题答清楚，std 对照表才不会变成误导。
