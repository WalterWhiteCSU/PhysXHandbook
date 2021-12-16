# Spatial Queries
--------------------
应用程序通常需要有效地查询空间中的卷，或者追踪光线或在空间中移动对象，以确定可能存在哪些内容。为此，PhysX 支持两个接口，一个用于场景中已有的对象，另一个用于针对任意 AABB 集进行查询。场景查询系统中讨论了场景查询。

# PxSpatialIndex
----------------
PxSpatialIndex 是一种 BVH 数据结构，允许在无需实例化 PxScene 的情况下执行空间查询。它支持插入、删除和更新定义边界框的任何对象，以及针对这些边界的光线投射、扫描和重叠查询。

空间索引在 3.4 中已被标记为已弃用，并将在将来的版本中移除。

SnippetSpatialIndex 显示了如何使用此类的示例。

PxSpatialIndex 没有内部锁定，从多个线程使用它时有一些特殊的注意事项。查询操作（在接口中标记为 const）不得与更新（非 const）操作并行发出，也不得彼此并行发出更新操作。并行发出查询操作时，请务必注意 PxSpatialIndex 会推迟对其内部数据结构的某些更新，直到发出查询。在单线程上下文中，这不会影响正确性或安全性，但是当同时从多个线程进行查询时，内部更新可能会导致数据危险。为了避免这些情况，请调用 flush（） 方法强制立即处理更新。在调用 flushUpdates（） 和任何后续更新操作之间，可以安全地并行发出查询。

针对 PxSpatialIndex 结构的查询将导致查询命中的每个 AABB 都会有一个回调，从而允许根据需要进行筛选或精确交集。PxGeometryQuery 类中的方法可用于执行这些交叉测试。结果通常按大致排序的顺序排列，当在针对 PxSpatialIndex 的光线投射或扫描查询中查找最接近的对象时，一个有用的优化是在回调中裁剪查询的长度。例如，在 SnippetSpatialIndex 中：

```C++
PxAgain onHit(PxSpatialIndexItem& item, PxReal distance, PxReal& shrunkDistance)
{
    PX_UNUSED(distance);

    Sphere& s = static_cast<Sphere&>(item);
    PxRaycastHit hitData;

    // the ray hit the sphere's AABB, now we do a ray-sphere intersection test to find out if
    // the ray hit the sphere

    PxU32 hit = PxGeometryQuery::raycast(position, direction,
                                         PxSphereGeometry(s.radius), PxTransform(s.position),
                                         1e6, PxHitFlag::eDEFAULT,
                                         1, &hitData);

    // if the raycast hits and is closer than what we had before, shrink the maximum length
    // of the raycast

    if(hit && hitData.distance < closest)
    {
        closest = hitData.distance;
        hitSphere = &s;
        shrunkDistance = hitData.distance;
    }

    // and continue the query

    return true;
}
```

**当形状在交叉或撞击的数值容差范围内时，PxGeometryQuery 中的方法可能会报告积极的结果。若要获得在使用 PxSpatialIndex 和不使用剔除层次结构时相同的结果，边界框必须稍微填充。PxGeometryQuery：：getWorldBounds 默认添加此填充。**

PxSpatialIndex 与使用 PxPruningStructureType：：eDYNAMIC_AABB_TREE 选项的场景查询系统具有相同的性能特征。如果 AABB 对应于移动对象，或者存在许多插入和删除，则树的质量可能会随着时间的推移而降低。为了防止这种情况，可以使用函数 rebuildFull（） 完全重建树。或者，第二棵树可以在后台通过许多小步骤以增量方式构建，使用函数 rebuildStep（） 执行与在 fetchResults（） 期间场景的动态修剪结构执行的相同的增量重建步骤。有关详细信息，请参阅 PxPruningStructureType。