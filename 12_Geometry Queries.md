# Geometry Queries
-----------------
## Introduction
-----------------
本章介绍如何将 `PhysX` 的碰撞功能与单个几何对象结合使用。几何查询主要有四种类型：

+ raycasts ("raycast queries") 针对几何对象测试射线。
+ sweeps ("sweep queries") 沿一条线移动一个几何对象，以查找与另一个几何对象相交的第一个点。
+ overlaps ("overlap queries") 确定两个几何对象是否相交。
+ penetration depth computations ("minimal translational distance queries", abbreviated here to "MTD") 测试两个重叠的几何对象，以找到它们可以沿最小距离分开的方向。

此外， `PhysX` 还提供了用于计算几何对象的 AABB 以及计算点与几何对象之间距离的帮助程序。

在以下所有函数中，几何对象由其`Shape`(PxGeometry structure)和`Pose`(PxTransform structure)定义。所有变换和向量都被解释为位于同一空间中，并且结果也会在该空间中返回。

## Raycasts
-------------
<img src=".\image\GeomQuery_1.PNG" alt="GeomQuery_1" style="zoom:100%;" />

`raycast query`沿线段追踪点，直到该点到达几何对象。 `PhysX` 支持所有几何体类型的`raycasts`。

以下代码演示如何使用射线投射查询：

```C++
PxRaycastHit hitInfo;
PxU32 maxHits = 1;
PxHitFlags hitFlags = PxHitFlag::ePOSITION|PxHitFlag::eNORMAL|PxHitFlag::eDISTANCE|PxHitFlag::eUV;
PxU32 hitCount = PxGeometryQuery::raycast(origin, unitDir,
                                          geom, pose,
                                          maxDist,
                                          hitFlags,
                                          maxHits, &hitInfo);
```

参数解释如下：

+ `origin` is the start point of the ray. 
+ `unitDir` is a unit vector defining the direction of the ray. 
+ `maxDist` is the maximum distance to search along the ray. It must be in the [0, inf) range. If the maximum distance is 0, a hit will only be returned if the ray starts inside a shape, as detailed below for each geometry. 
+ `geom` is the geometry to test against. 
+ `pose` is the pose of the geometry. 
+ `hitFlags` specifies the values that should be returned by the query, and options for processing the query. 
+ `maxHits` is the maximum number of hits to return. 
+ `hitInfo` specifies the PxRaycastHit structure(s) into which the raycast results will be stored. 
The `anyHit` parameter is deprecated. **It is equivalent to `PxHitFlag::eMESH_ANY`**, which should be used instead. 

返回的结果是找到的相交点数。对于每个交集，都会填充一个 `PxRaycastHit` 。此结构的字段如下所示：

```C++
PxRigidActor*   actor;
PxShape*        shape;
PxVec3          position;
PxVec3          normal;
PxF32           distance;
PxHitFlags      flags;
PxU32           faceIndex;
PxF32           u, v;
```

某些字段是可选的，标志字段(flags field)指示哪些成员已用结果值填充。如果在输入中设置了相应的标志，则查询将填充输出结构中的字段 - 例如，如果在输入 `hitFlags` 中设置了 `PxHitFlag::ePOSITION`，则查询将填充 `PxRaycastHit::position` 字段，并在 `PxRaycastHit::flags` 中设置 `PxHitFlag::ePOSITION` 标志。如果未为特定成员设置输入标志，则结果结构可能包含该成员的有效数据，也可能不包含该成员的有效数据。在输入中省略 `eNORMAL` 和 `ePOSITION` 标志有时会导致查询速度更快。

对于最初未与几何对象相交的射线投射，字段将按如下方式填充(可选字段与控制它们的标志一起列出)：

+ `actor` and `shape` are not filled (these fields are used only in scene-level raycasts, see Scene Queries). 
+ `position` (`PxHitFlag::ePOSITION`) is the position of the intersection. 
+ `normal` (`PxHitFlag::eNORMAL`) is the surface normal at the point of intersection. 
+ `distance` (`PxHitFlag::eDISTANCE`) is the distance along the ray at which the intersection was found. 
+ `flags` specifies which fields of the structure are valid. 
+ `faceIndex` is the index of the face which the ray hit. For triangle mesh and height field intersections, it is a triangle index. For convex mesh intersections it is a polygon index. For other shapes it is always set to 0xffffffff. 
+ `u` and `v` (PxHitFlag::eUV) are the barycentric coordinates of the intersection. These fields (and the flag) are supported only for meshes and heightfields. 

位置字段通过以下公式与重心坐标相关，其中 v0、v1 和 v2 是命中三角形的顶点：

*position = (1 - u - v)*v0 + u*v1 + v*v2;*

此映射在 `PxTriangle::pointFromUV()` 中实现。

有关如何使用面和顶点索引从`triangle meshes`、`convex meshes`和`HeightFields`中检索面和顶点数据的详细信息，请参阅Geometry。

如果射线在对象内启动，则上述行为的例外情况可能适用，在这种情况下， `PhysX` 可能无法计算某些字段的有意义的输出值。在这些情况下，该字段将保持未修改状态，并且不会设置相应的标志。具体细节因几何类型而异，如下所述。

射线投射交叉点的确切条件如下：

### Raycasts against Spheres, Capsules, Boxes and Convex Meshes
------------------------------------------------------------------
对于实体对象(Spheres、Capsules、Boxes、Convex Meshes)，最多返回 1 个结果。如果射线原点位于固体对象内部：

+ the reported hit distance is set to zero, and the `PxHitFlag::eDISTANCE` flag is set in the output. 
+ the hit normal is set to be the opposite of the ray's direction, and the `PxHitFlag::eNORMAL` flag is set in the output. 
+ the hit impact position is set to the ray's origin and the `PxHitFlag::ePOSITION` flag is set in the output. 

如果射线的起点或终点非常靠近物体的表面，则可以将其视为位于表面的任一侧。



### Raycasts against Planes
---------------------------
对于Raycasts，平面被视为包含其边界的无限单侧四边形(请注意，这与重叠(overlaps)不同)。最多返回一个结果，如果射线原点位于平面后面，则即使射线与平面相交，也不会报告命中。

如果射线的起点或终点非常靠近平面，则可将其视为位于平面的任一侧。

### Raycasts against Triangle Meshes
--------------------------------------
三角网格被视为thin triangle surfaces，而不是实体对象。它们可以配置为返回任意命中(arbitrary hit)、最近命中(closest hit)或多个命中(multiple hits)。

+ if `maxHits` is 1 and `PxHitFlag::eMESH_ANY` is not set, the query will return the closest intersection. 
+ if `maxHits` is 1 and `PxHitFlag::eMESH_ANY` is set, the query will return an arbitrary intersection. Use this when it is sufficient to know whether or not the ray hit the mesh, e.g. for line-of-sight queries or shadow rays. 
+ if `maxHits` is greater than 1, the query will return multiple intersections, up to `maxHits`. If more than `maxHits` intersection points exist, there is no guarantee that the results will include the closest. Use this for e.g. wall-piercing bullets that hit multiple triangles, or where special filtering is required. Note that `PxHitFlag::eMESH_MULTIPLE` must be used in this case. 

通常，"arbitrary hit"查询比"closest hit"查询快，"closest hit"查询比"multiple hits"查询快。

默认情况下，将剔除背面命中(三角形的外向法线(outward-facing normal )具有与射线方向的正点积)，因此对于任何三角形命中，报告的法线将具有射线方向的负点积。此行为可以通过网格实例的 `PxMeshGeometryFlag::eDOUBLE_SIDED` 标志和查询的 `PxHitFlag::eMESH_BOTH_SIDES` 标志进行修改：

+ if either `PxMeshGeometryFlag::eDOUBLE_SIDED` or `PxHitFlag::eMESH_BOTH_SIDES` is set, culling is disabled. 
+ if PxMeshGeometryFlag::eDOUBLE_SIDED is set, the reported normal is reversed for a back face hit. 

例如，透明玻璃窗可以建模为双面网格，以便射线会击中任何一侧，报告的法线朝向与射线方向相反。射线投射追踪可能穿透网格正面并从背面出现的子弹路径，可以使用 `eMESH_BOTH_SIDES` 来查找正面和背面的三角形，即使网格是单面的。

下图显示了在多个位置与网格相交的单个射线投射使用不同标志时发生的情况。

<img src=".\image\GeomQuery_2.PNG" alt="GeomQuery_2" style="zoom:100%;" />

要将 `PxHitFlag::eMESH_BOTH_SIDES` 用于选定的网格而不是所有网格，请在 `PxQueryFilterCallback` 中设置标志。

如果射线的起点或终点非常靠近三角形的表面，则可以将其视为位于三角形的任一侧。

### Raycasts against Heightfields
-----------------------------------
+ Heightfields are treated the same way as triangle meshes with normals oriented (in shape space) in +y direction when thickness is <=0 and in -y direction when thickness is >0. 
+ Double-sided heightfields are treated the same way as double sided triangle meshes. 

## Overlaps
------------
<img src=".\image\GeomQuery_3.PNG" alt="GeomQuery_3" style="zoom:100%;" />

重叠查询(Overlap queries)只需检查两个几何对象是否重叠即可。其中一个几何形状必须是`box`、`sphere`、`capsule`或`convex`，另一个可以是任何类型的几何形状。

下面的代码演示如何使用重叠查询：

```C++
bool isOverlapping = overlap(geom0, pose0, geom1, pose1);
```

重叠不支持命中标志(hit flags)，仅返回布尔值结果。

+ 平面被视为实心半空间(solid half-space)：也就是说，平面后面的一切都被视为体积的一部分。
+ 三角形网格被视为细三角形曲面(thin triangle surfaces)，而不是实体对象。
+ 高度场被视为按其厚度拉伸的三角形曲面。不与高度场曲面相交但位于拉伸空间内的重叠几何图形将报告命中。

如果网格和高度场需要的不仅仅是布尔值结果，请改用 `PxMeshQuery API`(请参阅 PxMeshQuery)。

## Penetration Depth
-------------------------
<img src=".\image\GeomQuery_4.PNG" alt="GeomQuery_4" style="zoom:100%;" />

当两个物体相交时， `PhysX` 可以计算出必须平移物体以分隔它们的最小距离和方向(这个数量有时被称为MTD，用于最小平移距离，因为它是最小长度的矢量，平移将分离形状)。一个几何对象必须是`box`、`sphere`、`capsule`或`convex mesh`，另一个可以是任何类型的。

以下代码说明了如何使用渗透深度查询(penetration depth query)：

```C++
bool isPenetrating = PxGeometryQuery::computePenetration(direction, depth,
                                                         geom0, pose0,
                                                         geom1, pose1);
```

参数解释如下：

+ `direction` is set to the direction in which the first object should be translated in order to depenetrate from the second. 
+ `distance` is set to the distance by which the first object should be translated in order to depenetrate from the second. 
+ `geom0` is the first geometry. 
+ `pose0` is the transform of the first geometry. 
+ `geom1` is the second geometry. 
+ `pose2` is the transform of the second geometry. 

如果对象正在穿透(penetrating)，则该函数返回 true，在这种情况下，它将设置direction和depth fields。通过`depenetration vector` $D = direction * depth$ 平移第一个对象会将两个对象分开。如果函数返回 true，则返回的深度将始终为正或为零。如果对象不重叠，则该函数返回 false，并且方向和距离字段的值未定义。

对于简单(凸)形状，返回的结果是准确的。

对于网格和高度场，使用迭代算法，并在 `PxExtensions` 中公开专用函数：

```C++
PxVec3 direction = PxComputeMeshPenetration(direction, depth,
                                            geom, geomPose,
                                            meshGeom, meshPose,
                                            maxIter, nb);

PxVec3 direction = PxComputeHeightFieldPenetration(direction, depth
                                                   geom, geomPose,
                                                   heightFieldGeom, heightFieldPose,
                                                   maxIter, nb);
```

此处， `maxIter` 是算法的最大迭代次数， `nb` 是可选的输出参数，将设置为执行的迭代次数。如果未检测到重叠，则将 `nb` 设置为零。该代码将尝试最多次的 `maxIter` 迭代，但如果找到`depenetration vector`，则可能会更早退出。通常 $maxIter = 4$ 会给出良好的结果。

这些函数仅计算近似的 `depenetration vector`，当几何对象与`mesh/heightfield`之间的重叠量很小时，这些函数效果最佳。特别是，当对象的中心位于三角形后面时，与三角形的交集将被忽略，如果这适用于所有相交的三角形，则不会检测到重叠，并且函数不会计算MTD向量。

## Sweeps
----------
<img src=".\image\GeomQuery_5.PNG" alt="GeomQuery_5" style="zoom:100%;" />

扫描查询(sweep query)在空间中跟踪一个几何对象，以查找第二个几何对象上的影响点，如果找到该影响点，则报告有关该影响点的信息。 `PhysX` 仅支持扫描查询，其中第一个几何对象(通过空间跟踪的对象)是`box`、`sphere`、`capsule`或`convex geometry`。第二个几何对象可以是任何类型。

下面的代码演示如何使用扫描查询：

```C++
PxSweepHit hitInfo;
PxHitFlags hitFlags = PxHitFlag::ePOSITION|PxHitFlag::eNORMAL|PxHitFlag::eDISTANCE;
PxReal inflation = 0.0f;
PxU32 hitCount = PxGeometryQuery::sweep(unitDir, maxDist,
                                        geomToSweep, poseToSweep,
                                        geomSweptAgainst, poseSweptAgainst,
                                        hitInfo,
                                        hitFlags,
                                        inflation);
```

参数解释如下：

+ `unitDir` is a unit vector defining the direction of the sweep. 
+ `maxDist` is the maximum distance to search along the sweep. It must be in the [0, inf) range, and is clamped by SDK code to at most PX_MAX_SWEEP_DISTANCE. A sweep of length 0 is equivalent to an overlap check. 
+ `geomToSweep` is the geometry to sweep. Supported geometries are: box, sphere, capsule or convex mesh. 
+ `poseToSweep` is the initial pose of the geometry to sweep. 
+ `geomSweptAgainst` is the geometry to sweep against (any geometry type can be used here). 
+ `poseSweptAgainst` is the pose of the geometry to sweep against. 
+ `hitInfo` is the returned result. A sweep will return at most one hit. 
+ `hitFlags` determines how the sweep is processed, and which data is returned if an impact is found. 
+ `inflation` inflates the first geometry with a shell extending outward from the object surface, making any corners rounded. It can be used to ensure a minimum margin of space is kept around the geometry when using sweeps to test whether movement is possible. 使用从对象表面向外延伸的壳体对第一个几何图形进行充气，使所有角都圆角。它可用于确保在使用扫描来测试是否可以移动时，在几何体周围保持最小的空间边距。

与 `raycasts` 一样，如果在输入 `hitFlags` 中设置了相应的标志，则字段将填充到输出结构中。 `PxSweepHit` 的字段如下所示：

```C++
PxRigidActor*   actor;
PxShape*        shape;
PxVec3          position;
PxVec3          normal;
PxF32           distance;
PxHitFlags      flags;
PxU32           faceIndex;
```

+ `actor` and `shape` are not filled (these fields are used only in scene-level sweeps, see Scene Queries). 
+ `position` (`PxHitFlag::ePOSITION`) is the position of the intersection. When there are multiple impact points, such as two boxes meeting face-to-face, `PhysX` will select one point arbitrarily. More detailed information for meshes or height fields may be obtained using the functions in PxMeshQuery. 
+ `normal` (PxHitFlag::eNORMAL) is the surface normal at the point of impact. It is a unit vector, pointing outwards from the hit object and backwards along the sweep direction (in the sense that the dot product between the sweep direction and the impact normal is negative). 
+ `distance` (PxHitFlag::eDISTANCE) is the distance along the ray at which the intersection was found. 
+ `flags` specifies which fields of the structure are valid. 
+ `faceIndex` is the index of the face hit by the sweep. This is a face from the hit object, not from the swept object. For triangle mesh and height field intersections, it is a triangle index. For convex mesh intersections it is a polygon index. For other shapes it is always set to 0xffffffff. For convex meshes the face index computation is rather expensive. The face index computation can be disabled by not providing the scene query hit flag PxHitFlag::eFACE_INDEX. If needed the face index can also be computed externally using the function PxFindFaceIndex which is part of the PhysX extensions library. 

与射线投射不同，u，v 坐标不支持扫描。

对于扫描到的几何对象：

+ 平面被视为实心半空间：也就是说，平面后面的一切都被视为要扫描的体积的一部分。
+ 与射线投射相同的背面剔除规则适用于扫描，但明显的区别在于不支持eMESH_MULTIPLE。

### Initial Overlaps
---------------------
与从物体内部开始的射线投射类似，扫描可以从两个几何图形最初相交开始。默认情况下， `PhysX` 将检测并报告重叠(`overlap`)。使用 `PxSweepHit::hadInitialOverlap()` 查看命中是否由初始重叠生成。

对于`triangle meshes`和`height fields`，在重叠检查之前执行背面剔除，因此，如果剔除三角形，则不会报告初始重叠。

根据 `PxHitFlag::eMTD` 的值， `PhysX` 也可能计算 `MTD` 。如果未设置 `PxHitFlag::eMTD`：

+ the `distance` is set to zero, and the `PxHitFlag::eDISTANCE` flag is set in the `PxSweepHit` result structure. 
+ the `normal` is set to be the opposite of the sweep direction, and the `PxHitFlag::eNORMAL` flag is set in the `PxSweepHit` result structure. 
+ the `position` is undefined, and the `PxHitFlag::ePOSITION` flag is not set in the `PxSweepHit` result structure. 
+ the `faceIndex` is a face from the second geometry object. For a heightfield or triangle mesh, it is the index of the first overlapping triangle found. For other geometry types, the index is set to 0xffffffff. 

如果设置了 `PxHitFlag::eMTD`，则命中结果定义如下：

+ the distance is set to the penetration depth, and the `PxHitFlag::eDISTANCE` flag is set in the `PxSweepHit` result structure. 
+ the normal is set to the depenetration direction, and the `PxHitFlag::eNORMAL` flag is set in the `PxSweepHit` result structure. 
+ the `position` is a point on the sweep geometry object (i.e. the first geometry argument) and the `PxHitFlag::ePOSITION` flag is set in the `PxSweepHit` result structure. 
+ the `faceIndex` is a face from the second geometry object: 
    + For triangle meshes and heightfields it is the last penetrated triangle found during the last iteration of the depenetration algorithm. 
    + For other geometry types, the index is set to 0xffffffff. 

在初始重叠的情况下，此标志将产生额外的处理开销。此外，以下限制适用：

+ `PxHitFlag::eMTD` 与 `PxHitFlag::ePRECISE_SWEEP` 和 `PxHitFlag::eASSUME_NO_INITIAL_OVERLAP` 不兼容(见下文)。将 `PxHitFlag::eMTD` 与这些标志中的任何一个结合使用都将导致发出警告，并且忽略与 `PxHitFlag::eMTD` 不兼容的标志。

测试初始重叠有时会使用专用代码路径，并导致性能下降。如果可以保证几何对象最初不重叠，则可以使用 `PxHitFlag::eASSUME_NO_INITIAL_OVERLAP`抑制重叠检查。使用此标志有一些限制(另请参阅Pitfalls)

+ Using `PxHitFlag::eASSUME_NO_INITIAL_OVERLAP` flag when the geometries initially overlap produces undefined behavior. 
+ `PxHitFlag::eASSUME_NO_INITIAL_OVERLAP` in combination with zero sweep distance produces a warning and undefined behavior. 

**使用`PxHitFlag::eMTD`的`sweep`使用两种背面剔除三角形。首先，根据`sweep`方向剔除三角形，以确定是否存在`overlap`。如果检测到`overlap`，则通过质心是否在三角形后面进一步剔除它们，如果未找到三角形，则方向将设置为与`sweep`方向相反，并且距离为0。**

**在大多数情况下，将第一个几何对象平移 -normal*distance 会将对象分开。但是，使用iterative  depenetration algorithm 来查找`triangle meshes`和`height fields`的MTD，并且在极端情况下，MTD结果可能无法从网格提供完全的depenetration。在这种情况下，应在应用translation后再次调用查询。**

**`PhysX 3.3`中的一个已知问题是，未设置 `eMTD` 标志时，对`convex mesh`进行`sweep`的`face index`未定义。**

### Precise Sweeps
------------------
`PxHitFlag::ePRECISE_SWEEP`支持更准确的`sweep`代码(默认情况下使用可能更快但不太准确的解决方案)。`ePRECISE_SWEEP`标志与膨胀参数(inflation parameter)或标志`PxHitFlag::eMTD`不兼容。

#### Sweeps against Height Fields
----------------------------------
+ Height fields are treated as thin triangle surfaces rather than solid objects.

+ Thickness magnitude has no effect on initial overlap detection or point of impact.

+ For single-sided height fields the normal of the hit will face in +y local space direction if thickness is < 0 and -y when thickness is > 0.

+ Height fields are treated as double sided if either one of `eDOUBLE_SIDED` or `eMESH_BOTH_SIDES` flags are used.
    + The returned hit normal will always face the sweep direction. 

+ `eMESH_ANY` flag has no effect.

+ `ePRECISE_SWEEP` flag has no effect.

#### Pitfalls
--------------
使用`sweep`时需要注意一些陷阱(Pitfalls)：

+ 由于数值精度问题，当对象具有非常大的尺寸差异时，可能会返回不正确的结果。

+ 由于算法差异，`sweep query`可能会检测到一组与`overlap query`不同的初始重叠形状。特别是，仅仅执行重叠检查以确定 `PxHitFlag::eIGNORE_INITIAL_OVERLAP` 标志的安全性是不够的。需要一致的overlap/sweep/penetration depth 信息的应用程序应使用扫描检查(sweep checks)和初始重叠测试(overlap testing)和`PxHitFlag::eMTD`标志。

## Additional PxGeometryQuery functions
----------------------------------------
<img src=".\image\GeomQuery_6.png" alt="GeomQuery_6" style="zoom:100%;" />

以下函数计算点与几何对象之间的距离。仅支持实体对象(`box`, `sphere`, `capsule`, `convex`)：

```C++
PxReal dist = PxGeometryQuery::pointDistance(point, geom, pose, closestPoint);
```

`closepoint` 是返回最近点的可选输出参数。

<img src=".\image\GeomQuery_7.png" alt="GeomQuery_7" style="zoom:100%;" />

以下函数计算包围几何对象的轴对齐边界框 (AABB)，给定其pose：

```C++
PxBounds3 bounds = PxGeometryQuery::getWorldBounds(geom, pose, inflation);
```

边界框按 `inflation` 值缩放，如果未明确指定，则默认为 1.01f。

## PxMeshQuery
---------------
`PhysX` 提供了额外的功能，用于获取`triangle mesh`和`height field`重叠的多个结果，以及针对三角形数组进行扫描。只有 `boxes` 、 `spheres` 和 `capsules` 可以使用这些函数针对网格或高度场进行测试。

### Mesh Overlaps
-----------------
以下代码说明了如何处理接触给定球面体积(given spherical volume)的`mesh triangles`：

```C++
PxU32 triangleIndexBuffer[bufferSize];
PxU32 startIndex = 0;
bool bufferOverflowOccured = false;
PxU32 nbTriangles = PxMeshQuery::findOverlapTriangleMesh(sphereGeom, spherePose,
                                                         meshGeom, meshPose,
                                                         triangleIndexBuffer, bufferSize,
                                                         startIndex, bufferOverflowOccured);

for(PxU32 i=0; i < nbTriangles; i++)
{
    PxTriangle tri;
    PxU32 vertexIndices[3];
    PxMeshQuery::getTriangle(meshGeom, meshPose, triangleIndexBuffer[i], tri, vertexIndices);

     ...  // process triangle info
}
```

`findOverlapTriangleMesh` 方法用于提取三角形的索引：

+ `sphereGeom` and `spherePose` specify the region to test for overlap. 
+ `meshGeom` and `meshPose` specify the mesh and its pose. 
+ `triangleIndexBuffer` and `triangleSize` specify the output buffer and its size. 
+ `startIndex` is used to restart the query if the buffer size is exceeded. In this case, to query for more triangles set this parameter to the number retrieved so far. 
+ `bufferOverflowOccured` is set if more triangles would be returned from the query than would fit in the buffer. 

`height fields`也存在类似的查询功能。

### Sweeps against Triangles
-----------------------------
有时，例如，在使用 `mesh overlap API` 时，能够对三角形组进行`sweep`非常方便。 `PhysX` 提供了一个专门用于此目的的功能，具有以下签名：

```C++
bool sweep(const PxVec3& unitDir,
           const PxReal distance,
           const PxGeometry& geom,
           const PxTransform& pose,
           PxU32 triangleCount,
           const PxTriangle* triangles,
           PxSweepHit& sweepHit,
           PxHitFlags hitFlags = PxHitFlag::eDEFAULT,
           const PxU32* cachedIndex = NULL,
           const PxReal inflation = 0.0f,
           bool doubleSided = false);
```

参数解释如下：

+ `unitDir`, `distance`, `geom` and `pose` function identically to the first four parameters of `PxGeometryQuery::sweep()`. `distance` is clamped to `PX_MAX_SWEEP_DISTANCE`. 
+ `triangleCount` is the number of triangles contained in the buffer against which to sweep. 
+ `triangles` is the buffer of triangles. 
+ `hitFlags` specifies the required information in the output. 
+ `cachedIndex`, if set, specifies the index of a triangle to test first. This can be a useful optimization when repeatedly sweeping against the same set of triangles. 
+ `inflation` functions identically to the `inflation` parameter of `PxGeometryQuery::sweep()`. 
+ `doubleSided` indicates whether the input triangles are double-sided or not. This is equivalent to the `PxMeshGeometryFlag::eDOUBLE_SIDED` flag - that is, it suppresses backface culling, and for any hit the returned normal faces opposite to the sweep direction (see Raycasts against Triangle Meshes). 

与其他扫描查询相比，此函数具有额外的限制：

+ the geometry type must be either a `sphere`, a `capsule` or a `box`. Convex geometry is not supported. 
+ the function returns a single hit. Multiple hits (and in particular `PxHitFlag::eMESH_MULTIPLE`) are not supported. 
+ The function always returns the closest hit. 
+ The only supported flags are `PxHitFlag::eDEFAULT`, `PxHitFlag::eASSUME_NO_INITIAL_OVERLAP`, `PxHitFlag::ePRECISE_SWEEP`, `PxHitFlag::eMESH_BOTH_SIDES` and `PxHitFlag::eMESH_ANY`. 

该函数按给定的顺序测试每个输入三角形。默认情况下，该函数将测试所有三角形并返回最接近的扫描命中(如果已找到命中)。如果使用`PxHitFlag::eMESH_ANY`，则一旦找到命中，该函数将立即返回(跳过剩余的未测试三角形)。此标志还可用于模拟 `PxHitFlag::eMESH_MULTIPLE`，方法是使用 `PxHitFlag::eMESH_ANY` 重复调用函数，使用以前返回的命中三角形作为起点(其索引介于 0 和 '`triangleCount`' 之间，可在 `sweepHit.faceIndex` 中使用)。