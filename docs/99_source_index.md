# 99. 源码索引和阅读路线

这一页按主题整理 OCCT 源码入口。建议打开文件后先搜类名和成员名，不要从第一行读到最后一行。

## 基础容器

```text
src/FoundationClasses/TKernel/NCollection/NCollection_BaseMap.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_BaseMap.cxx
src/FoundationClasses/TKernel/NCollection/NCollection_Map.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_DataMap.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_IndexedMap.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_IndexedDataMap.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_List.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_Sequence.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_Array1.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_DynamicArray.hxx
src/FoundationClasses/TKernel/NCollection/NCollection_Vector.hxx  # deprecated alias in 8.0
```

本教程图解：

```text
docs/images/01_occt_layers.svg
docs/images/02_basemap.svg
docs/images/03_shape_identity.svg
docs/images/04_topexp_dfs.svg
docs/images/05_ancestor_index.svg
docs/images/06_boolean_images_origins.svg
docs/images/07_hlr_index_arrays.svg
docs/images/08_handle_lifetime.svg
```

推荐搜索：

```text
myData1
myData2
BeginResize
Resizable
Add
FindIndex
FindKey
Bound
ChangeSeek
RemoveLast
Substitute
```

## 单元测试

```text
src/FoundationClasses/TKernel/GTests/NCollection_Map_Test.cxx
src/FoundationClasses/TKernel/GTests/NCollection_DataMap_Test.cxx
src/FoundationClasses/TKernel/GTests/NCollection_IndexedMap_Test.cxx
src/FoundationClasses/TKernel/GTests/NCollection_IndexedDataMap_Test.cxx
src/FoundationClasses/TKernel/GTests/Standard_Handle_Test.cxx
src/ModelingData/TKBRep/GTests/TopExp_Test.cxx
```

测试文件通常比正式算法更适合第一次阅读，因为它们展示 API 的最小用法。

## Shape identity

```text
src/ModelingData/TKBRep/TopoDS/TopoDS_Shape.hxx
src/ModelingData/TKBRep/TopoDS/TopoDS_Shape.cxx
src/ModelingData/TKBRep/TopTools/TopTools_ShapeMapHasher.hxx
```

推荐搜索：

```text
myTShape
myLocation
myOrient
IsPartner
IsSame
IsEqual
HashCode
Orientation
```

## 拓扑遍历

```text
src/ModelingData/TKBRep/TopExp/TopExp.hxx
src/ModelingData/TKBRep/TopExp/TopExp.cxx
src/ModelingData/TKBRep/TopExp/TopExp_Explorer.hxx
src/ModelingData/TKBRep/TopExp/TopExp_Explorer.cxx
```

推荐搜索：

```text
MapShapes
MapShapesAndAncestors
MapShapesAndUniqueAncestors
TopExp_Explorer
pushIterator
popIterator
More
Next
Current
Depth
```

## 旧别名迁移

```text
src/Deprecated/NCollectionAliases/TopTools_MapOfShape.hxx
src/Deprecated/NCollectionAliases/TopTools_IndexedMapOfShape.hxx
src/Deprecated/NCollectionAliases/TopTools_IndexedDataMapOfShapeListOfShape.hxx
src/Deprecated/NCollectionAliases/TopTools_ListOfShape.hxx
```

这些文件说明旧名仍为兼容存在，但推荐直接写 `NCollection_*` 模板类型。

## 布尔运算

```text
src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Builder.hxx
src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Builder_1.cxx
src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Builder_2.cxx
src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Builder_3.cxx
src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_PaveFiller.hxx
```

推荐搜索：

```text
myArguments
myMapFence
myImages
myOrigins
myShapesSD
FillImages
FillImagesContainer
Modified
Generated
NCollection_DataMap
NCollection_IndexedDataMap
```

## HLR 隐线算法

```text
src/ModelingAlgorithms/TKHLR/HLRBRep/HLRBRep_Data.hxx
src/ModelingAlgorithms/TKHLR/HLRBRep/HLRBRep_Data.cxx
src/ModelingAlgorithms/TKHLR/HLRBRep/HLRBRep_ShapeBounds.hxx
src/ModelingAlgorithms/TKHLR/HLRBRep/HLRBRep_Algo.hxx
```

推荐搜索：

```text
myEMap
myFMap
myEData
myFData
EDataArray
FDataArray
EdgeMap
FaceMap
ChangeValue
```

## 句柄和生命周期

```text
src/FoundationClasses/TKernel/Standard/Standard_Handle.hxx
src/FoundationClasses/TKernel/Standard/Standard_Transient.hxx
src/FoundationClasses/TKernel/Standard/Standard_Transient.cxx
src/FoundationClasses/TKernel/Standard/Standard_Type.hxx
```

推荐搜索：

```text
opencascade::handle
Handle(Class)
IncrementRefCounter
DecrementRefCounter
GetRefCount
DownCast
DynamicType
IsKind
```

## 可视化交互层

```text
src/Visualization/TKV3d/AIS/AIS_InteractiveContext.hxx
```

推荐搜索：

```text
myObjects
mySelection
myDetectedSeq
NCollection_DataMap
NCollection_Sequence
NCollection_List
```

这个文件适合观察 NCollection 容器如何从建模算法延伸到交互显示系统。
