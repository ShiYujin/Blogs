# [WIP]Lightmap UV生成算法
Lightmap指的是存储光照信息的纹理贴图，是一种特殊的纹理，用于解决实时渲染中的全局光照（间接光、阴影等）问题。和其他纹理不一样的是，Lightmap不允许有有重复的UV，这是因为物体上的每一个点都有着不一样的光照信息。Lightmap可以对单个物体生成（如一般纹理一样），也可以对整个场景生成（纹理图会更大）。

本文将会介绍从一般UV坐标生成Lightmap UV的算法，并分析UE4的实现源码。
## 算法介绍


## UE4实现
UE4代码：Engine\Source\Developer\MeshUtilities\Private\MeshUtilities.cpp，L2560

步骤
- 准备数据，设置UV version
  - `FOverlappingCorners`: FindOverlappingCorners，找到共享/重合的点，并且标记下来。
  - `FLayoutUVRawMeshView`: 继承自`IMeshView`的类，对Mesh底层数据的封装，提供访问面片和顶点数据的接口。
- FindCharts
  - 根据`FOverlappingCorners`寻找相邻的面片，保存在并查集`FDisjointSet DisjointSet`里，标记为同一个`Chart`
    - 标准：三角形的三个顶点中，有两个顶点共享xyz和uv坐标，并且uv空间的朝向一致（注：这个标准比较严格，不但要几何上重合，还必须在UV空间重合）
    - 第i个面片的`DisjointSet[i]`指向与他若干阶相邻的index最大的一个三角形面片
    - 对`DisjointSet`按照`DisjointSet[i]`排序，保存在`SortedTris`里，这个`SortedTris`是接下来合并和组织`Chart`的关键，`SortedTris`是按照三角形所属的`Chart`编号排序的，这意味着同一个`Chart`的三角形都放在一起，便于`Chart.FirstTri`和`Chart.LastTri`记录当前`Chart`包含的三角形范围。
  - 构建Charts，每个Charts表示有邻接关系的面片的集合，`FMeshChart`结构体保存了该Charts内的下列信息，其中最后两个后面才会计算
    - 第一个面片的序号（这个序号是`SortedTris`的序号，即`SortedTris[i]`才是真正的面片序号）
    - 最后一个面片的序号（同上）
    - UV的bounding box
    - UV的总面积
    - UV的缩放比例
    - UV的偏置
  - [?为什么要单独合并这些Charts而不是直接生成为一个？]利用`TranslatedMatches`计算`Charts.Join`，并且合并Join Charts
    - `TranslatedMatches`保存的是这两个顶点xyz空间相邻但UV空间不相邻（但UV空间距离相等）的面片，用于构建`Chart`的邻接关系`Chart.Join`，`TranslatedMatches`用于指导`Chart`的合并。
- FindBestPacking
  - 计算packing的方案，这里主要是求Charts的Scale和Offset值。
  - 
- CommitPackedUVs
  - 