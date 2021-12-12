# Geometry

## introduction

这部分主要讨论`Physx`的几何类。`geometry`主要用来构建刚体的形状，刚体的形状主要用来作碰撞的触发和physx中的屏幕查询系统的量。Physx也提供独立的function用来做两个`geometry`的相交性检测，射线扫描和几何体扫描（sweep geometry）。

`geometry`是值类型，并且所有的`geometry`类都继承自共同的基类`PxGeometry`。每一个`geometry`类都定义了一个可以定位的体积或者表面。`transform`指明了`geometry`在当前帧的位置。`Physx`提供了对于`plane`和`capsule`几何类型的`helper function`来通过相同的可替代的表达构建他们的`transform`。

`Geometry`分为两类：

+ 包含所有数据的原始结构，包括`PxBoxGeometry`、`PxCapsuleGeometry`、`PxPlaneGeometry`
+ 一个包含指向更多数据指针的结构。包括网格和高度场`PxConvexMeshGeometry`、`PxTriangleMeshGeometry`、`PxHeightFieldGeometry`。指针指向的大数据为`PxConvexMesh`、`PxTriangleMesh`、`PxHeightField`。我们能在每一个被`PxGeometry Type`引用的大数据中使用不同的比例。大数据必须使用`cooking process`来创建。

当我们使用这些`geometry`作为`simulation geometry`来传入或者传出SDk时，`geometry`会拷贝进或者复制出`PxShape Class`。这样在不知道`geometry`的类型的情况下取回是非常危险的，`Physx`提供了一个`union-like wrapper class(PxGeometryHolder)`来通过`geometry`的值传递类型。每一个网格（或者高度场）有一个`reference count`来追踪指向mesh的`PxShape`的编号。

## Geometry Type

---

## Spheres

---



<img src=".\image\Geo_1.PNG" alt="Geo_1" style="zoom:100%;" />

`PxSphereGeometry `由一个属性（即其半径）指定，并以原点为中心。

## Capsule

---

<img src=".\image\Geo_2.PNG" alt="Geo_2" style="zoom:75%;" />

`PxCapsuleGeometry`的中心在原点。它通过声明半径和X轴向两边伸长的半程高来创建。

为了创建一个`geometry`为向上站立的`dynamic actor`，`shape`需要一个相对的`transform`。这个`transform`绕着Z轴旋转四分之一个圆。这样`capsule`就通过Y轴来延展`actor`。设置`shape`和`actor`和`sphere`是一样的。

```c++
PxRigidDynamic* aCapsuleActor = thePhysics->createRigidDynamic(PxTransform(position));
PxTransform relativePose(PxQuat(PxHalfPi, PxVec(0,0,1)));
PxShape* aCapsuleShape = PxRigidActorExt::createExclusiveShape(*aCapsuleActor,
    PxCapsuleGeometry(radius, halfHeight), aMaterial);
aCapsuleShape->setLocalPose(relativePose);
PxRigidBodyExt::updateMassAndInertia(*aCapsuleActor, capsuleDensity);
aScene->addActor(aCapsuleActor);

```

函数`PxTransformFromSegment`将定义`capsule`轴的线段转换成`transform`和`halfheight`。

## Boxs

---

<img src=".\image\Geo_3.PNG" alt="Geo_3" style="zoom:100%;" />

`pXBoxGeometry`有三个attribute

```c++
PxShape* aBoxShape = PxRigidActorExt::createExclusiveShape(*aBoxActor,
    PxBoxGeometry(a/2, b/2, c/2), aMaterial);
```

a、b 和 c 是生成的vox的边长。

## Planes

---

![Geo_4](.\image\Geo_4.PNG)

`plane`将空间分为上面和下面。下面的东西都会与`plane`产生碰撞

平面一般与YZ平面重合的平面且向朝上的`pointer`指向X正半轴。我们可以使用`PxTransformFormPlaneEquation()`将平面方程转换成等价的`transform`。

`PxPlaneEquationFromTransform()`提供相反的转换。

`PxPlaneGeometry`没有attribute，因为形状的位置完全定义了平面的碰撞体积。

`PxPlaneGeometry`的shape只能为静态actor创建。

## Convex Meshes

---

<img src=".\image\Geo_5.png" alt="Geo_5" style="zoom:150%;" />

对于一个多边形，如果给定两个其内部的点，这两点的连线一定在这个多边形的内部，那么我们说这个多边形是凸多边形。`PxConvexMesh`是由一系列的顶点和多边形面表示的凸多边形。`Physx`中凸多边形的顶点和面的数量限制在255内。

创建一个`PxConvexMesh`需要cooking。这里我们假设cooking library已经被初始化。接下来我们来看看如何创建`square pyramid`。

+ 首先定义凸包的顶点

  ```c++
  static const PxVec3 convexVerts[] = {PxVec3(0,1,0),PxVec3(1,0,0),PxVec3(-1,0,0),PxVec3(0,0,1),
      PxVec3(0,0,-1)};
  ```

  凸包数据的描述布局结构为

  ```c++
  PxConvexMeshDesc convexDesc;
  convexDesc.points.count     = 5;
  convexDesc.points.stride    = sizeof(PxVec3);
  convexDesc.points.data      = convexVerts;
  convexDesc.flags            = PxConvexFlag::eCOMPUTE_CONVEX;
  ```

+ 现在我们可以使用`cooking library`来构建`PxConvexMesh`

  ```c++
  PxDefaultMemoryOutputStream buf;
  PxConvexMeshCookingResult::Enum result;
  if(!cooking.cookConvexMesh(convexDesc, buf, &result))
      return NULL;
  PxDefaultMemoryInputData input(buf.getData(), buf.getSize());
  PxConvexMesh* convexMesh = physics->createConvexMesh(input);
  ```

+ 最后我们使用`PxConvexMeshGeometry`来创建形状。`PxConvexMeshGeometry`声明网格的实例

  ```c++
  PxShape* aConvexShape = PxRigidActorExt::createExclusiveShape(*aConvexActor,
      PxConvexMeshGeometry(convexMesh), aMaterial);
  ```



我们也可以不进行序列化直接cook`PxConvexMesh`然后将`PxConvexMesh`添加进`PxPhysics`。如果需要实时cooking时这很有用。当然，还是建议使用离线的cooking和streams。下面的代码展示了如何提升cooking的速度

```c++
PxConvexMeshDesc convexDesc;
convexDesc.points.count     = 5;
convexDesc.points.stride    = sizeof(PxVec3);
convexDesc.points.data      = convexVerts;
convexDesc.flags            = PxConvexFlag::eCOMPUTE_CONVEX | PxConvexFlag::eDISABLE_MESH_VALIDATION | PxConvexFlag::eFAST_INERTIA_COMPUTATION;

#ifdef _DEBUG
    // mesh should be validated before cooking without the mesh cleaning
    bool res = theCooking->validateConvexMesh(convexDesc);
    PX_ASSERT(res);
#endif

PxConvexMesh* aConvexMesh = theCooking->createConvexMesh(convexDesc,
    thePhysics->getPhysicsInsertionCallback());
```

我们需要在bebug和check build时确认网格的有效性。如果通过不确定的descriptor来创建网格可能导致不确定的行为。提供的`PxConvexFlag::eFAST_INERTIA_COMPUTATION`标志volume integration能够使用SIMD代码路径，这样能够使用更少的代价获得更快的计算速度。

使用者能够在`PxConvexMeshGeometry`中提供一个`per-instance PxMeshScale`。==The scale defaults to identity==。不支持负值。

`PxConvexMeshGeometry`也包含一些能够对convex object进行调整的flags。默认情况下系统会计算convex object周围的近似的边界(bounds)。我们可以使用`PxConvexMeshGeometryFlag::eTIGHT_BOUNDS`，能够得到更紧致的边界。当然，这样做会产生更多的计算，但是在非常多的object相交的场景下这样能得到更好的模拟结果。

`PxConvexMeshGeometry`包含一个叫`maxMargin`的变量。默认设为3.4e38f。如果 `maxMargin `小于 PCM 接触生成（PCM contact gen）计算的边距量，它将为缩小的形状选择最小边距，以使用 GJK 算法执行增量更新。在这种情况下，应用程序可能会注意到顶点碰撞周围的一些伪影。如果 `maxMargin `设置为较小的值，则可能会降低这些伪影的可见性。如果 `maxMargin `设置为零，PCM 将使用 GJK 算法的原始形状，这将导致此方法不会出现伪影。但是，在性能和准确性之间存在权衡。

## Convex Mesh cooking

-------

`Convex Mesh cooking`将网格数据转换为一种允许 SDK 执行高效的碰撞检测的形式。cooking的输入是通过输入 `PxConvexMeshDesc `结构体定义的。填充此结构体的方法不同，具体取决于是要仅从点云开始，还是从已经具有多面体的顶点和面开始生成`Convex Mesh`。

### If Only Vertex Points are Provided（只有点云）

当仅提供顶点时，设置 `PxConvexFlag::eCOMPUTE_CONVEX` 标志来计算网格：

```c++
PxConvexMeshDesc convexDesc;
convexDesc.points.count     = 20;
convexDesc.points.stride    = sizeof(PxVec3);
convexDesc.points.data      = convexVerts;
convexDesc.flags            = PxConvexFlag::eCOMPUTE_CONVEX;
convexDesc.maxVerts         = 10;

PxDefaultMemoryOutputStream buf;
if(!cooking.cookConvexMesh(convexDesc, buf))
    return NULL;
```

该算法尝试从源顶点创建`convex mesh`。字段 `convexDesc.vertexLimit` 指定生成的外壳中最大顶点数的限制。当源数据在几何上具有挑战性时，例如，如果它包含许多彼此靠近的顶点，则此例程有时会失败。如果cooking失败，则会向错误流报告错误，并且例程返回 false。\

如果使用` PxConvexFlag::eCHECK_ZERO_AREA_TRIANGLES`，则算法不包括面积小于 `PxCookingParams::areaTestEpsilon` 的三角形。如果算法找不到 4 个没有小三角形的初始顶点，则返回 `PxConvexMeshCookingResult::eZERO_AREA_TEST_FAILED`。这意味着提供的顶点位于非常小的区域内，cooker无法产生有效的hull。`toolkit helper`函数 `PxToolkit::createConvexMeshSafe` 说明了`Convex Mesh cooking`的最可靠策略：首先，它试图在没有inflation的情况下制造`Hull`。如果失败，它会尝试inflation，如果也失败了，则使用AABB或OBB。

建议在原点周围提供顶点，并将转换放在 `PxShape `中，否则添加` PxConvexFlag::eSHIFT_VERTICES`网格计算的flag。如果提供了大量的输入顶点，量化输入顶点可能会很有用，在这种情况下，请使用`PxConvexFlag::eQUANTIZE_INPUT`并设置所需的`PxConvexMeshDesc::quantizedCount`

Convex cooking 支持两种不同的算法：

###  Quickhull Algorithm

此算法不使用inflation。它将创建一个`Convex Hull`，其顶点是原始顶点的子集，并且顶点数保证不超过指定的最大值。

Quickhull 算法执行以下步骤：

+ 清理顶点 - 删除重复项等。
+ 查找包含输入集的顶点子集，不超过`vertexLimit`。
+ 如果达到`vertexLimit`，请围绕输入顶点展开有限的外壳，以确保我们封装所有输入顶点。
+ 计算顶点map table。（每个顶点至少需要 3 个相邻面。）
+ 检查多边形数据 - 验证所有顶点是否都在hull上或其内部。
+ 计算质量和惯性张量，假设密度为 1。
+ Saves data to stream.

构造hull时，添加的每个新顶点必须比 `PxCookingParams::planeTolerance` 距hull更远，如果不是，则丢弃该顶点。

### Inflation Based Incremental Algorithm

-----------

此算法始终使用 `PxConvexFlag::eINFLATE_CONVEX`标志，并通过 `PxCookingParams::skinWidth` 对hull planes进行膨胀。

Inflation Incremental Algorithm 执行以下步骤：

+ 清理顶点 - 删除重复项等。
+ 查找包含输入集的顶点子集，不超过`vertexLimit`
+ 从生产的封闭Hull中创建平面。
+ 通过定义的 `PxCookingParams::skinWidth` 来膨胀平面。
+ 通过膨胀的平面裁剪AABB并产生新的hull。
+ 计算顶点map table。（每个顶点至少需要 3 个相邻面。）
+ 检查多边形数据 - 验证所有顶点是否都在hull上或其内部。
+ 计算质量和惯性张量，假设密度为 1。
+ Saves data to stream.

请注意，Inflation Incremental Algorithm可以生成具有更多输入顶点的hull.该算法明显慢于Quickhull，并且产生的结果明显不那么稳定。建议使用Quickhull算法。

### Vertex Limit Algorithms

如果提供了顶点限制，则有两种算法可以处理顶点限制。

默认算法计算完整的外壳，以及输入顶点周围的 OBB。然后用hull平面对该OBB进行切片，直到达到顶点极限。默认算法要求将顶点限制至少设置为 8，并且通常生成的质量比平面平移(plane shifting)产生的结果要好得多。

启用平面移位(plane shifting ) （`PxConvexFlag::ePLANE_SHIFTING`） 后，当达到顶点限制时，hull计算将停止。然后，将hull平面平移以包含所有输入顶点，然后使用新的平面交点生成具有给定顶点限制的最终hull。平面偏移可能会在远离输入点云的点上产生尖锐的边缘，并且不能保证所有输入顶点都在生成的hull内。但是，它可以在顶点限制低至4的情况下使用，因此对于顶点计数非常低的小碎片等情况，它可能是更好的选择。

### Vertex Points, Indices and Polygons are Provided（顶点、顶点索引和多边形）

------

要创建给定一组输入顶点（凸顶点）和多边形（hullPolygons）的`PxConvexMesh`，请执行以下操作：

```c++
PxConvexMeshDesc convexDesc;
convexDesc.points.count             = 12;
convexDesc.points.stride            = sizeof(PxVec3);
convexDesc.points.data              = convexVerts;
convexDesc.polygons.count   = 20;
convexDesc.polygons.stride  = sizeof(PxHullPolygon);
convexDesc.polygons.data    = hullPolygons;
convexDesc.flags                    = 0;

PxDefaultMemoryOutputStream buf;
if(!cooking.cookConvexMesh(convexDesc, buf))
    return NULL;
```

提供点和面后，SDK 会验证网格并直接创建 `PxConvexmesh`。这是创建`Convex Mesh`的最快方法。请注意，SDK 要求每个顶点至少有 3 个相邻面。否则，不会创建 PCM 的加速结构，如果启用了 PCM，则会导致性能下降。

convex cooking过程中的内部步骤：

+ 计算顶点映射表（vertex map table），每个顶点至少需要 3 个相邻面。
+ 检查多边形数据 - 检查所有顶点是否都在hull上或hull内部。
+ 计算质量和惯性张量，假设密度为 1。
+ Save data to stream。

## Triangle Meshes

-------------

<img src=".\image\Geo_6.png" alt="Geo_6" style="zoom:100%;" />

与图形三角形网格一样，碰撞三角形网格由顶点和三角形索引的集合组成。创建三角形网格需要使用cooking库。此处假定cooking库已初始化

```c++
PxTriangleMeshDesc meshDesc;
meshDesc.points.count           = nbVerts;
meshDesc.points.stride          = sizeof(PxVec3);
meshDesc.points.data            = verts;

meshDesc.triangles.count        = triCount;
meshDesc.triangles.stride       = 3*sizeof(PxU32);
meshDesc.triangles.data         = indices32;

PxDefaultMemoryOutputStream writeBuffer;
PxTriangleMeshCookingResult::Enum result;
bool status = cooking.cookTriangleMesh(meshDesc, writeBuffer,result);
if(!status)
    return NULL;

PxDefaultMemoryInputData readBuffer(writeBuffer.getData(), writeBuffer.getSize());
return physics.createTriangleMesh(readBuffer);
```

此外，`PxTriangleMesh`可以cooked并直接插入`PxPhysics`中，而无需流序列化。如果需要实时cooking，这很有用。强烈建议使用离线cooking和流。如果需要，如何提高cooking速度的示例：

```c++
PxTolerancesScale scale;
PxCookingParams params(scale);
// disable mesh cleaning - perform mesh validation on development configurations
params.meshPreprocessParams |= PxMeshPreprocessingFlag::eDISABLE_CLEAN_MESH;
// disable edge precompute, edges are set for each triangle, slows contact generation
params.meshPreprocessParams |= PxMeshPreprocessingFlag::eDISABLE_ACTIVE_EDGES_PRECOMPUTE;
// lower hierarchy for internal mesh
params.meshCookingHint = PxMeshCookingHint::eCOOKING_PERFORMANCE;

theCooking->setParams(params);

PxTriangleMeshDesc meshDesc;
meshDesc.points.count           = nbVerts;
meshDesc.points.stride          = sizeof(PxVec3);
meshDesc.points.data            = verts;

meshDesc.triangles.count        = triCount;
meshDesc.triangles.stride       = 3*sizeof(PxU32);
meshDesc.triangles.data         = indices32;

#ifdef _DEBUG
    // mesh should be validated before cooked without the mesh cleaning
    bool res = theCooking->validateTriangleMesh(meshDesc);
    PX_ASSERT(res);
#endif

PxTriangleMesh* aTriangleMesh = theCooking->createTriangleMesh(meshDesc,
    thePhysics->getPhysicsInsertionCallback());
```

索引可以是 16 位或 32 位。此处使用的步幅假定顶点和索引分别是 PxVec3 和 32 位整数的数组，数据布局中没有间隙。

返回的结果枚举 `PxTriangleMeshCookingResult::eLARGE_TRIANGLE` 确实会在网格包含大三角形时警告用户，应进行细分以确保更好的仿真和 CCT 稳定性。

与`height fields`一样，三角网格支持每个三角形的材料索引。要将每个三角形材质用于网格，请在网格描述符中向cooking库提供每个三角形的索引。稍后，在创建 `PxShape `时，请提供一个将网格中的索引值映射到材质实例的表。

### 	Triangle Mesh cooking

----------

三角形网格cooking过程如下

+ 检查输入顶点的有效性。
+ 焊接顶点并检查三角形尺寸。
+ 为queries创建加速结构。
+ 计算边缘凸性信息和邻接。
+ Save data to stream. 

请注意，网格清理可能导致cooking产生的三角形集与原始输入集不同，网格清理可移除无效三角形（包含超出范围的顶点参照）、重复的三角形和零区域三角形。发生这种情况时，PhysX 可以选择输出一个网格重新映射表（mesh remapping table），该表将每个内部三角形链接到用户数据中的源三角形。

有多个参数可用于控制网格创建。

+ In *PxTriangleMeshDesc*:
  + `materialIndices` 定义每个三角形材质。当三角形网格与另一个对象碰撞时，碰撞点处需要材质。如果 `materialIndices `为 NULL，则使用 `PxShape` 实例的材料。

+ In *PxCookingParams*:
  + `scale` 定义公差刻度用于检查cooking好的的三角形是否不太大。此检查将有助于提高模拟稳定性。
  + `suppressTriangleMeshRemapTable `指定是否创建表面重映射表（face remap table）。否则，这将节省大量内存，但 SDK 将无法提供有关在碰撞、扫描或光线投射命中时击中哪个原始网格三角形的信息。
  + `buildTriangleAdjacies `指定是否创建了三角形邻接信息。可以使用 `getTriangle `检索给定三角形的相邻三角形。
  + `meshPreprocessParams `指定网格预处理参数。
    + `PxMeshPreprocessingFlag::eWELD_VERTICES` 可在三角形网格cooking过程中实现顶点焊接。
    + `PxMeshPreprocessingFlag::eDISABLE_CLEAN_MESH` 禁用网格清理过程。顶点重复不被搜索，巨大的三角形测试不做。顶点焊接未完成。能够加快cooking速度。
    + `PxMeshPreprocessingFlag::eDISABLE_ACTIVE_EDGES_PRECOMPUTE` 禁用顶点边缘预计算。使cooking更快，但减少接触生成（contact generation）。
  + `meshWeldTolerance `- 如果启用了网格焊接，则控制顶点焊接的距离。如果未启用网格焊接，则此值定义网格验证的验收距离（acceptance distance ）。
  + `midphaseDesc`指定所需的中相加速度结构描述符（midphase acceleration structure descriptor）。
    + `PxBVH33MidphaseDesc - PxMeshMidPhase::eBVH33` 是默认结构。它是在最近的`PhysX`版本中使用的版本，直到`PhysX 3.3`。它具有出色的性能，并且在所有平台上都受支持。
    + `PxBVH34MidphaseDesc - PxMeshMidPhase::eBVH34` 是 `PhysX 3.4` 中引入的重新访问的实现。在cooking性能和运行时性能方面，它的速度都明显更快，但目前仅在支持 SSE2 指令集的平台上可用。
+ *PxBVH33MidphaseDesc params*
  + `meshCookingHint `指定网格层次结构构造首选项。与碰撞性能相比，可实现更好的cooking性能，适用于cooking性能比创建最高质量的网格更重要的应用。
  + `meshSizePerformanceTradeOff `指定网格大小和运行时性能之间的权衡。
+ *PxBVH34MidphaseDesc params*:
  + numTrisPerLeaf 指定每片leaf的三角形数。每片leaf的三角形越少，产生的网格越大，运行时性能通常越好，cooking性能越差。

## Height Fields

--------

<img src=".\image\Geo_7.png" alt="Geo_7" style="zoom:100%;" />

`Height Fields`的局部空间轴为：

+ Row - X axis 
+ Column - Z axis 
+ Height - Y axis 

顾名思义，地形只能通过常规矩形采样网格上的高度值来描述：

```c++
PxHeightFieldSample* samples = (PxHeightFieldSample*)alloc(sizeof(PxHeightFieldSample)*
    (numRows*numCols));
```

每个样本都由一个 16 位整数高度值、两个材质（对于样本矩形中的两个三角形）和一个曲面细分标志组成。

标志和材质是指下方和采样点右侧的采样点，并指示沿哪个对角线将其拆分为三角形，以及这些三角形的材料。特殊的预定义材料 `PxHeightFieldMaterial::eHOLE` 指定高度字段中的孔。有关更多详细信息，请参阅 `PxHeightFieldSample `的参考文档。

<img src=".\image\Geo_8.PNG" alt="Geo_8" style="zoom:75%;" />

| Tesselation flags                                            | Result                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src=".\image\Geo_9.PNG" alt="Geo_9" style="zoom:75%;" /> | <img src=".\image\Geo_10.PNG" alt="Geo_10" style="zoom:75%;" /> |
| <img src=".\image\Geo_11.PNG" alt="Geo_11" style="zoom:75%;" /> | <img src=".\image\Geo_12.PNG" alt="Geo_12" style="zoom:75%;" /> |
| <img src=".\image\Geo_13.PNG" alt="Geo_13" style="zoom:75%;" /> | <img src=".\image\Geo_14.PNG" alt="Geo_14" style="zoom:75%;" /> |

要告诉系统每个方向上的采样高度数，请使用描述符来实例化 `PxHeightField `对象： 

```c++
PxHeightFieldDesc hfDesc;
hfDesc.format             = PxHeightFieldFormat::eS16_TM;
hfDesc.nbColumns          = numCols;
hfDesc.nbRows             = numRows;
hfDesc.samples.data       = samples;
hfDesc.samples.stride     = sizeof(PxHeightFieldSample);

PxHeightField* aHeightField = theCooking->createHeightField(hfDesc,
    thePhysics->getPhysicsInsertionCallback());
```

现在创建一个`PxHeightFieldGeometry`和一个形状：

```c++
PxHeightFieldGeometry hfGeom(aHeightField, PxMeshGeometryFlags(), heightScale, rowScale,
    colScale);
PxShape* aHeightFieldShape = PxRigidActorExt::createExclusiveShape(*aHeightFieldActor,
    hfGeom, aMaterialArray, nbMaterials);
```

行和列刻度告诉系统采样点在关联方向上的距离有多远。高度刻度将整数高度值缩放到浮点范围。

此处使用的 `createExclusiveShape()`的变体为高度字段指定了一个材质数组，该数组将由每个单元格的材料索引进行索引，以解决与该单元格的冲突。可以改用单材料变体，但高度字段材料索引必须全部为单个值或特殊值 `eHOLE`。

使用 `PxHeightFieldFlag::eNO_BOUNDARY_EDGES` 标志可以禁用地形边界处具有三角形边缘的接触生成，从而在排列了多个高度场形状以使其边缘接触时，可以更有效地生成接触。

### Heightfield cooking

--------------------------

`Heightfield`数据可以在离线状态下进行cooking，然后用于`createHeightField`。cooking预先计算并存储边缘信息。这样可以更快地创建`heightfield`，因为边缘已经预先计算过了。如果需要在运行时中创建`heightfields `，这将非常有用，因为它确实可以显著提高创建`HeightField`的速度。

Heightfield cooking 过程如下：

+ 将`heightfield`样本加载到内部存储器中。
+ 预先计算边缘碰撞信息。
+ Save data to stream. 

### Unified Heightfields

-------------------

`PhysX`为`heightfields`提供了两种接触生成方法。这些是：

+ 默认统一`heightfield `接触生成。
+ 传统`heightfield`接触生成。

默认的统一高度场接触生成方法（default unified heightfield contact generation approach）从高度场中提取三角形，并利用用于针对三角形网格生成接触的相同低级接触生成代码。如果可互换使用三角形网格或高度场，则此方法可确保等效的行为和性能。但是，通过这种方法，高度场表面没有厚度，因此如果不启用CCD，快速移动的物体可能会被穿过(tunnel)。经典高度场碰撞代码（在以前版本的 PhysX 中是默认的）的工作方式与三角形网格接触生成不同。除了生成与接触高度场表面的形状的接触外，它还生成与表面下方形状的接触。高度场的"厚度"用于控制在表面接触下方生成多远。这是通过沿垂直轴的"厚度"在粗检测中挤出高度场的AABB来工作的。将为边界与高度场拉伸边界相交的曲面以下的任何形状生成触点。

通过调用以下命令启用统一高度场联系生成(heightfield contact generation)：

```c++
PxRegisterHeightFields(PxPhysics& physics);
```

通过调用以下命令启用传统高度场联系人生成：

```c++
PxRegisterLegacyHeightFields(PxPhysics& physics);
```

这些调用必须在创建场景之前进行，否则将发出警告。高度场碰撞设置是全局设置，适用于所有场景。

如果调用 `PxCreatePhysics（...）`，这将自动调用 `PxRegisterHeightFields（...）` 来注册默认的统一高度场碰撞方法。如果调用 `PxCreateBasePhysics（...）`，则默认情况下不会注册任何高度场接触生成。如果使用高度场，应用程序必须调用相应的高度场注册函数。

## Deformable meshes

`PhysX`支持可变形网格(`Deformable meshes`)，即其顶点随时间移动的网格（而拓扑，即三角形索引，保持固定）。可变形网格仅支持 `PxMeshMidPhase::eBVH33` 数据结构。由于网格顶点将被更新，因此用户定义的顶点与 PhysX 的内部顶点之间的映射也必须保留。也就是说，`PhysX `不应在cooking过程中对顶点重新排序。因此，应禁用所有可以对顶点重新排序的cooking操作，并且用户有责任确保传递的顶点是正确的，并且禁用了操作。例如，应禁用网格清洁阶段:

```c++
cookingParams.midphaseDesc.setToDefault(PxMeshMidPhase::eBVH33);
    cookingParams.meshPreprocessParams      = PxMeshPreprocessingFlag::eDISABLE_CLEAN_MESH;
```

可以在同一场景中混合`eBVH33`和`eBVH34`网格，因此默认的cooking参数仍可用于不可变形/静态网格。

要修改顶点，请在模拟场景之前使用`PxTriangleMesh::getVerticesForModification()`和`PxTriangleMesh::refitBVH()`函数：

```c++
// get vertex array
PxVec3* verts = mesh->getVerticesForModification();

// update the vertices here
...

// tell PhysX to update the mesh structure
PxBounds3 newBounds = mesh->refitBVH();
```

然后使用`PxScene::resetFiltering()` 作为相应的网格执行器，告诉粗检测其边界已被修改：

```c++
scene->resetFiltering(*actor);
```

当网格变形并远离位于其上的物体时，所述网格会在网格上轻微反弹和抖动。对网格形状使用略微负的静止偏移有助于减少此影响：

```c++
PxShape* shape;
mesh->getShapes(&shape, 1);
shape->setRestOffset(-0.5f);   // something negative, value depends on your game's scale
```

这将使对象"沉入"到动态网格中。这样，接触就不会立即丢失，并且运动保持平稳。有关更多详细信息，请参阅 SDK 中的可变形网格片段。

## Mesh Scaling

---------------

共享的 `PxTriangleMesh `或 `PxConvexMesh `在被几何体实例化时可能会被拉伸或压缩。这允许对应用不同比例因子的同一网格进行多次实例化。缩放是使用 `PxMeshScale `类指定的，该类定义要沿 3 个正交轴应用的比例因子。因子大于 1.0 会导致拉伸，而因子小于 1.0 会导致压缩。轴的方向由四元数控制，并在形状的局部框架中指定。支持负网格缩放，负值沿每个相应轴产生相反方向的缩放。此外，当 scale.x*scale.y*scale.z < 0 时，`PhysX` 将翻转网格三角形的法线。

下面的代码创建一个形状，其 `PxTriangleMesh `沿 x 轴缩放 x 倍，沿 y 轴缩放 y，沿 z 轴缩放 z：

```c++
// created earlier
PxRigidActor* myActor;
PxTriangleMesh* myTriMesh;
PxMaterial* myMaterial;

// create a shape instancing a triangle mesh at the given scale
PxMeshScale scale(PxVec3(x,y,z), PxQuat(PxIdentity));
PxTriangleMeshGeometry geom(myTriMesh,scale);
PxShape* myTriMeshShape = PxRigidActorExt::createExclusiveShape(*myActor,geom,*myMaterial);
```

凸网格以类似的方式使用 PxMeshScale 类进行缩放。下面的代码创建一个形状，其 `PxConvexMesh `沿 (sqrt(1/2)、1.0、-sqrt(1/2))缩放 x 因子，沿 (0，1，0) 缩放 y 因子，沿 (sqrt(1/2)、1.0、sqrt(1/2))缩放z 因子 ：

```c++
PxMeshScale scale(PxVec3(x,y,z), PxQuat quat(PxPi*0.25f, PxVec3(0,1,0)));
PxConvexMeshGeometry geom(myTriMesh,scale);
PxShape* myConvexMeshShape = PxRigidActorExt::createExclusiveShape(*myActor,geom,*myMaterial);
```

还可以使用存储在 `PxHeightFieldGeometry `中的比例因子来缩放Height fields。在这种情况下，假定比例尺沿行、列和高度字段的高度方向的轴。缩放在 `SampleNorthPoleBuilder `中的 `SampleNorthPole.cpp` 中演示：

```c++
PxHeightFieldGeometry hfGeom(heightField, PxMeshGeometryFlags(), heightScale, hfScale, hfScale);
PxShape* hfShape = PxRigidActorExt::createExclusiveShape(*hfActor, hfGeom, getDefaultMaterial());
```

在此示例中，沿 x 轴和 z 轴的坐标按 `hfScale `缩放，而样本高度按 `heightScale `缩放。

## PxGeometryHolder

------------------

为`Shape`提供几何图形时，无论是在创建时还是使用 `PxShape::setGeometry()` 时，该几何图形都会复制到 SDK 的内部结构中。如果您知道形状几何形状的类型，则可以直接检索它：

```c++
PxBoxGeometry boxGeom;
bool status = shape->getBoxGeometry(geometry);
```

如果`Shape`的几何图形不是预期类型，则状态返回代码设置为 false。但是，在未首先知道其类型的情况下从`Shape`中检索几何对象通常很方便 - 例如，调用将 `PxGeometry `引用作为参数的函数。`PxGeometryHolder `是一个类似并集的类，它允许按值返回 `PxGeometry `对象，而不管类型如何。它的用法在`PhysXSample.cpp`中的`createRenderObjectFromShape()`函数中进行了说明：

```c++
PxGeometryHolder geom = shape->getGeometry();

switch(geom.getType())
{
case PxGeometryType::eSPHERE:
    shapeRenderActor = SAMPLE_NEW(RenderSphereActor)(renderer, geom.sphere().radius);
    break;
case PxGeometryType::eCAPSULE:
    shapeRenderActor = SAMPLE_NEW(RenderCapsuleActor)(renderer, geom.capsule().radius,
        geom.capsule().halfHeight);
    break;
...
}
```

函数` PxGeometryHolder::any()` 返回对 `PxGeometry `对象的引用。例如，要比较场景中的两个形状是否重叠：

```c++
bool testForOverlap(const PxShape& s0, const PxShape& s1)
{
    return PxGeometryQuery::overlap(s0.getGeometry().any(), PxShapeExt::getGlobalPose(s0),
                                    s1.getGeometry().any(), PxShapeExt::getGlobalPose(s1));
}
```

## Vertex and Face Data

----------

Convex meshes、triangle meshes和height fields都可以查询顶点和面数据。例如，在渲染凸形状的网格时，这特别有用。该方法为：

```c++
RenderBaseActor* PhysXSample::createRenderObjectFromShape(PxShape* shape,
    RenderMaterial* material)
```

在 `PhysXSample.cc` 包含一个 switch 语句，其中包含每个形状类型的大小写，说明了查询顶点和面所需的步骤。可以使用`PxMeshQuery::getTriangle`函数从triangle meshes或height fields中获取有关三角形的信息。您还可以检索给定三角形的相邻三角形索引（三角形的triangleNeighbour[i]共享edge vertex[i]-vertex[(i+1)%3]，其中三角形索引为"triangleIndex"，其中顶点在0到2的范围内）。要启用此功能，三角形网格在构建三角形相邻参数设置为 true 的情况下进行cooking。

## Convex Meshes

凸网格包含一个顶点数组、一个面数组和一个索引缓冲区，该缓冲区连接每个面的顶点索引。要解压缩Convex meshes，第一步是提取共享Convex meshes：

```C++
PxConvexMesh* convexMesh = geom.convexMesh().convexMesh;
```

然后获取对顶点和索引缓冲区的引用：

```c++
PxU32 nbVerts = convexMesh->getNbVertices();
const PxVec3* convexVerts = convexMesh->getVertices();
const PxU8* indexBuffer = convexMesh->getIndexBuffer();
```

现在循环访问面数组以对它们进行triangulate：

```C++
PxU32 offset = 0;
for(PxU32 i=0;i<nbPolygons;i++)
{
    PxHullPolygon face;
    bool status = convexMesh->getPolygonData(i, face);
    PX_ASSERT(status);

    const PxU8* faceIndices = indexBuffer + face.mIndexBase;
    for(PxU32 j=0;j<face.mNbVerts;j++)
    {
        vertices[offset+j] = convexVerts[faceIndices[j]];
        normals[offset+j] = PxVec3(face.mPlane[0], face.mPlane[1], face.mPlane[2]);
    }

    for(PxU32 j=2;j<face.mNbVerts;j++)
    {
        *triangles++ = PxU16(offset);
        *triangles++ = PxU16(offset+j);
        *triangles++ = PxU16(offset+j-1);
    }

    offset += face.mNbVerts;
}
```

观察多边形的顶点索引从` indexBuffer[face.mIndexBase] `开始，顶点计数由 `face.mNbVerts` 给出。

## Triangle Meshes

-----------

Triangle meshes包含顶点和索引三元组的数组，这些三元组通过索引到顶点缓冲区来定义三角形。这些数组可以直接从共享Triangle meshes访问：

```c++
PxTriangleMesh* tm = geom.triangleMesh().triangleMesh;
const PxU32 nbVerts = tm->getNbVertices();
const PxVec3* verts = tm->getVertices();
const PxU32 nbTris = tm->getNbTriangles();
const void* tris = tm->getTriangles();
```

索引可以使用 16 位或 32 位值存储，这些值在网格最初cooking时指定。要确定运行时的存储格式，请使用 API 调用：

```c++
const bool has16bitIndices = tm->has16BitTriangleIndices();
```

假设三角形索引以 16 位格式存储，请通过以下方式找到第 i 个三角形的第 j 个顶点：

```c++
const PxU16* triIndices = (const PxU16*)tris;
const PxU16 index = triIndices[3*i +j];
```

对应的顶点是：

```c++
const PxVec3& vertex = verts[index];
```

## Height Fields

Height Fields样本可以在运行时以矩形块的形式进行修改。在以下代码片段中，我们创建一个 Height Fields 并修改其示例：

```c++
// create a 5x5 HF with height 100 and materials 2,3
PxHeightFieldSample samples1[25];
for (PxU32 i = 0; i < 25; i ++)
{
    samples1[i].height = 100;
    samples1[i].materialIndex0 = 2;
    samples1[i].materialIndex1 = 3;
}

PxHeightFieldDesc heightFieldDesc;
heightFieldDesc.nbColumns = 5;
heightFieldDesc.nbRows = 5;
heightFieldDesc.thickness = -10;
heightFieldDesc.convexEdgeThreshold = 3;
heightFieldDesc.samples.data = samples1;
heightFieldDesc.samples.stride = sizeof(PxHeightFieldSample);

PxPhysics* physics = getPhysics();
PxHeightField* pHeightField = cooking->createHeightField(heightFieldDesc, physics->getPhysicsInsertionCallback());

// create modified HF samples, this 10-sample strip will be used as a modified row
// Source samples that are out of range of target heightfield will be clipped with no error.
PxHeightFieldSample samplesM[10];
for (PxU32 i = 0; i < 10; i ++)
{
    samplesM[i].height = 1000;
    samplesM[i].materialIndex0 = 1;
    samplesM[i].materialIndex1 = 127;
}

PxHeightFieldDesc desc10Rows;
desc10Rows.nbColumns = 1;
desc10Rows.nbRows = 10;
desc10Rows.samples.data = samplesM;
desc10Rows.samples.stride = sizeof(PxHeightFieldSample);

pHeightField->modifySamples(1, 0, desc10Rows); // modify row 1 with new sample data
```

`PhysX `不会保留从Height Fields到引用它的Height Fields Shape的映射。调用 `PxShape::set`引用高度字段的每个形状的几何图形的几何图形，以确保更新内部数据结构以反映新的几何图形：

```c++
PxShape *hfShape = userGetHfShape(); // the user is responsible for keeping track of
                                     // shapes associated with modified HF
hfShape->setGeometry(PxHeightFieldGeometry(pHeightField, ...));
```

另请注意，当对象位于旧几何或新几何体之上时，`PxShape::setGeometry() `不保证正确/连续的行为。方法 `PxHeightField::getTimestamp()` 返回Height Fields被修改的次数。

