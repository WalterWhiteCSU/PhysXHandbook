# Scene Queries
-----------------
## Introduction
----------------
`PhysX` 在 `PxScene` 中提供了一些方法，用于对场景中的 `Actor` 和附加形状(attached shapes)执行碰撞查询(collision queries)。有三种类型的查询:光线投射(raycasts)、扫描(sweeps)和重叠(overlaps)，每种查询都可以返回单个结果或多个结果。从广义上讲，每个查询遍历包含场景对象的剔除结构，使用 `GeometryQuery` 函数执行精确测试(请参见Geometry Queries)，并累积结果。在精确测试之前或之后可能会发生过滤(Filtering)。

该场景使用两种不同的查询结构，一种用于 `PxRigidStaticacter` ，另一种用于 `PxRigidBodyactor` (PxRigidDynamic 和 PxArticulationLink)。这两种结构可以配置为使用不同的剔除实现，具体取决于所需的速度/空间特性(请参见 PxPruningStructureType)。

## Basic queries
----------------
### Raycasts
----------------
`PxScene::raycast()` 查询将用户定义的光线与整个场景相交。`raycast()`查询最简单的用例是查找沿给定光线的最接近的命中，如下所示:

```C++
PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance
PxRaycastBuffer hit;                 // [out] Raycast results

// Raycast against all static & dynamic objects (no filtering)
// The main result from this call is the closest hit, stored in the 'hit.block' structure
bool status = scene->raycast(origin, unitDir, maxDistance, hit);
if (status)
    applyDamage(hit.block.position, hit.block.normal);
```

在此代码段中， `PxRaycastBuffer` 对象用于接收来自光线投射查询的结果。如果命中了 `raycast()`，则对 `raycast()` 的调用将返回 true。`hit.hadBlock` 也设置为 true，如果有命中。光线投射的距离必须在 [0， inf) 范围内。

光线投射结果包括`position`、`normal`、`hit distance`、`shape`和`actor`，以及`triangle meshes`和`heightfields`的具有 UV 坐标的面索引。在使用查询结果之前，请先检查`PxHitFlag::ePOSITION`，`eNORMAL`，`eDISTANCE`，`eUV`标志，因为在某些情况下它们没有设置。

### Sweeps
------------
`PxScene::sweep()` 查询在几何上类似于 `raycast():PxGeometry` 形状从具有指定最大长度的方向单位 Dir 的指定初始pose扫描，以查找几何图形与场景对象的影响点。扫描的最大距离必须在 [0， inf) 范围内，并且将被夹紧(clamped)到文件 `PxScene.h` 中定义的 `PX_MAX_SWEEP_DISTANCE` 。

允许的形状是`box`，`sphere`，`capsule`和`convex`。

`PxSweepBuffer` 对象用于接收来自 `sweep()` 查询的结果:

```C++
PxSweepBuffer hit;              // [out] Sweep results
PxGeometry sweepShape = ...;    // [in] swept shape
PxTransform initialPose = ...;  // [in] initial shape pose (at distance=0)
PxVec3 sweepDirection = ...;    // [in] normalized sweep direction
bool status = scene->sweep(sweepShape, initialPose, sweepDirection, sweepDistance, hit);
```

扫描结果包括`position`、`normal`、`hit distance`、`shape`和`actor`，以及triangle meshes和heightfields的面索引。

### Overlaps
------------
`PxScene::overlap()` 查询在由指定形状包围的区域中搜索场景中任何重叠的对象。该区域被指定为变换后的`box`，`sphere`，`capsule`和`convex`几何。

`PxOverlapBuffer` 对象用于接收来自 `overlap()` 查询的结果:

```C++
PxOverlapBuffer hit;            // [out] Overlap results
PxGeometry overlapShape = ...;  // [in] shape to test for overlaps
PxTransform shapePose = ...;    // [in] initial shape pose (at distance=0)

PxOverlapBuffer hit;
bool status = scene->overlap(overlapShape, shapePose, hit);
```

重叠结果仅包括`actor/shape`和面索引，因为没有单一的交点。

## Touching and blocking hits
-----------------------------
对于具有多个结果的查询，我们会区分触摸(touching)和阻止(blocking)命中。命中是touching还是blocking由用户实现的过滤逻辑进行选择。直观地说，blocking hit会阻止在路径上 `raycast` 或 `sweep` ；touching hit会记录信息，但允许射线或扫描继续进行。因此，multiple-hit query将返回最接近的blocking hit(如果存在)，以及任何更接近的touching hits。如果没有blocking hit，则将返回所有touching hits。

有关详细信息，请参阅Filtering部分。

## Query modes
---------------
### Closest hit
--------------------
所有三种查询类型的默认操作模式均为"最接近的命中(closest hit)"。该查询查找所有blocking hits，选择距离最小的命中，并在 `PxHitBuffer::block` 成员中报告它。

+ 对于 `overlap()` 查询，选择任意blocking hit作为报告的blocking hit(对于所有 `overlap()` 命中，距离被视为零)。

### Any hit
-----------
所有三种查询类型都可以在"任意命中(Any hit)"模式下运行。这是对查询系统的性能提示，指示无需查找最接近的命中 - 遇到的任何命中都可以。此模式最常用于布尔阻塞/非阻塞查询(boolean blocking/non-blocking queries)。性能改进可能是 3 倍或更多，具体取决于方案。要激活此模式，请使用`PxQueryFlag::eANY_HIT`过滤器数据标志并将其设置为 `PxQueryFilterData` 对象，例如:

```C++
PxQueryFilterData fd;
fd.flags |= PxQueryFlag::eANY_HIT; // note the OR with the default value
bool status = scene->raycast(origin, unitDir, maxDistance, hit,
                             PxHitFlags(PxHitFlag::eDEFAULT), fdAny);
```

### Multiple hits
------------------
所有三种查询类型(raycast, overlap, sweep)还可以报告场景中对象的多次命中。

+ 要为raycast激活此模式，请使用 `PxRaycastBuffer` 构造函数，其中包含用户提供的用于touching hits的缓冲区。
+ 在此模式下，所有命中默认为"touching"类型，并记录在`PxRaycastBuffer::touchs`数组中。

例如:

```C++
PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance

const PxU32 bufferSize = 256;        // [in] size of 'hitBuffer'
PxRaycastHit hitBuffer[bufferSize];  // [out] User provided buffer for results
PxRaycastBuffer buf(hitBuffer, bufferSize); // [out] Blocking and touching hits stored here

// Raycast against all static & dynamic objects (no filtering)
// The main result from this call are all hits along the ray, stored in 'hitBuffer'
scene->raycast(origin, unitDir, maxDistance, buf);
for (PxU32 i = 0; i < buf.nbTouches; i++)
    animateLeaves(buf.touches[i]);
```

相同的机制用于overlap(使用 `PxOverlapBuffer` 与 `PxOverlapHit[]`)和sweep( `PxSweepBuffer` 与`PxSweepHit[]`)。

### Multiple hits with blocking hit
------------------------------------
在上面multiple hits的snippet中，我们只期望touching hits。如果遇到blocking hit和touching hits，则会在 `PxHitBuffer::block` 成员中报告，并且touch缓冲区将仅包含更接近的touching hits。这种组合在诸如子弹穿过窗户(在途中打破它们)或树叶(使它们沙沙作响)直到它们撞到阻挡物体(混凝土墙)的情况下很有用:

```C++
// same initialization code as in the snippet for multiple hits
bool hadBlockingHit = scene->raycast(origin, unitDir, maxDistance, buf);
if (hadBlockingHit)
    drawWallDecal(buf.block);
for (PxU32 i = 0; i < buf.nbTouches; i++)
{
    assert(buf.touches[i].distance <= buf.block.distance);
    animateLeaves(buf.touches[i]);
}
```

+ 默认情况下，当提供touch缓冲区时，需要假定是touch hits，并且筛选器回调(filter callback)应返回 `PxQueryHitType::eBLOCK` 以表示命中正在blocking。有关详细信息，请参阅 Filtering 。

+ 对于 `overlap()` 查询，即使遇到blocking hit并设置了 `PxQueryFlag::eNO_BLOCK` 标志，也会记录所有touching hits。

## Filtering
-----------
过滤(Filtering)控制如何从scene query结果中排除形状以及如何报告结果。所有三种查询类型都支持以下筛选参数:

+ `PxQueryFilterData` 结构，包含 `PxQueryFlags` 和 `PxFilterData`
+ 一个可选的 `PxQueryFilterCallback`

### PxQueryFlag::eSTATIC, PxQueryFlag::eDYNAMIC
------------------------------------------------
`PxQueryFlag::eSTATIC` 和 `PxQueryFlag::eDYNAMIC` 标志控制查询是否应包含来自static 和/或 dynamic查询结构(query structures)的形状。这是过滤掉所有static/dynamic shapes的最有效方法。例如，将力应用于区域中所有动力学的爆炸效果可以使用球形overlap query，并且只有`PxQueryFlag::eDYNAMIC`标志才能排除所有static，因为力不能应用于static对象。默认情况下，statics和dynamics都包含在查询结果中。

例如:

```C++
PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance
PxRaycastBuffer hit;                 // [out] Raycast results

// [in] Define filter for static objects only
PxQueryFilterData filterData(PxQueryFlag::eSTATIC);

// Raycast against static objects only
// The main result from this call is the boolean 'status'
bool status = scene->raycast(origin, unitDir, maxDistance, hit, PxHitFlag::eDEFAULT, filterData);
```

### PxQueryFlag::ePREFILTER, PxQueryFlag::ePOSTFILTER
-----------------------------------------------------
场景查询分三个阶段执行:粗检测(broad phase)、中检测(midphase)和细检测(narrow phase)。*不知道怎么翻译*

+ broad phase遍历全局场景空间分区结构，以找到midphase和narrow phase的候选项。
+ midphase遍历triangle mesh和heightfield内部剔除结构，以在broad phase报告的网格中找到三角形的较小子集。
+ narrow phase执行精确的相交测试(`raycast()`查询的光线测试(ray test)，以及`sweep()`和`overlap()`查询的精确扫描形状(exact sweep shape)测试或重叠测试(overlap tests))。

若要在查询中实现自定义筛选(custom filtering)，请使用所需的筛选逻辑设置 `PxQueryFlag::ePREFILTER` 和/或 `PxQueryFlag::ePOSTFILTER` 标志和子类 `PxQueryFilterCallback`。

+ 预滤波(Pre-filtering)发生在midphase和narrow phase之前，并允许在可能昂贵的精确碰撞测试之前有效地丢弃形状。对于triangle meshes, heightfields, convexes和大多数扫描，这些测试比仅涉及简单形状(如spheres, capsules and boxes)的raycast tests 和 overlap tests更昂贵。
+ 后过滤(Post-filtering)发生在narrow phase测试之后，因此可以使用测试结果(例如 `PxRaycastHit.position` )来确定是否应丢弃命中。这些结果可以通过命中输入参数访问后过滤回调(`PxQueryFilterCallback::postFilter`)。例如，使用`static_cast<PxRaycastHit&>(hit)`，访问特定于raycast query的数据，以及类似的overlaps(`PxOverlapHit`)和sweep(`PxSweepHit`)。

筛选回调的实现返回 `PxQueryHitType` 结果。

+ `eNONE` indicates that the hit should be discarded. 
+ `eBLOCK` indicates that the hit is blocking. 
+ `eTOUCH` indicates that the hit is touching. 

每当使用非零 `PxHitCallback::nbTouches` 和 `PxHitCallback::touchs` 参数调用 `raycast()`、`sweep()` 或 `overlap()` 查询时，将报告 `eTOUCH` 类型命中(touchDistance <= blockDistance)与最接近的 `eBLOCK` 类型命中数不进一步。例如，若要记录光线投射查询中的所有命中，请始终返回 `eTOUCH`。

**从筛选器回调(filter callback)返回 `eTOUCH` 要求命中缓冲区查询参数具有非零 `::touchs` 数组，否则 `PhysX` 将在检查的构建中生成错误并丢弃任何touching hits。**

**`eBLOCK` 不应从overlap()的用户筛选器(user filters)返回。这样做将导致未定义的行为，并将发出警告。如果设置了 `PxQueryFlag::eNO_BLOCK` 标志，则 `eBLOCK` 将自动转换为 `eTOUCH`，并禁止显示警告。**

### PxQueryFlag::eANY_HIT
-------------------------
使用此标志可强制查询将第一次遇到的命中(可能不是最接近的)报告为`blocking hit`。性能可能快三倍以上，具体取决于方案。对于与附近相交对象的long raycasts/sweeps，或与多个相交对象(multiple intersecting objects)overlap，可以预期最佳增益。

*Also see PxHitFlag::eMESH_ANY*

### PxQueryFlag::eNO_BLOCK
--------------------------
如果要override从filters返回到 `eTOUCH` 的 `eBLOCK` 值，或者在预期没有blocking hit的情况下(在本例中，此标志用作性能提示)，请使用此标志。然后，无论filter callback返回值如何，所有命中都将报告为touching。使用此标志时，提供给查询的 hit callback/buffer 对象需要具有非零 `PxHitBuffer::touchs` 缓冲区。只有在接触命中缓冲区溢出的情况下，才应期望显著的性能提升。

**此标志将overrides预筛选器函数(pre-filter functions)和后筛选器函数(post-filter functions)的返回值，因此以前作为blocking返回的命中将作为touching返回。**

### PxFilterData fixed function filtering
-----------------------------------------
`PxFilterData` 提供了一个快速的固定功能滤波器(fixed-function filter)， `PxFilterData` 是内置滤波方程(filtering equation)使用的4 * 32位位掩码。每个shape都有一个位掩码(bitmask)(通过 `PxShape::setQueryFilterData()`设置)，查询也有一个位掩码。

批处理查询(batched queries)和未批处理查询(unbatched queries)以不同的方式使用query data(有关批处理查询，请参见下文)。对于未批处理查询，将应用以下规则:

+ 如果查询的位掩码全部为零，则自定义筛选和交集测试(filtering and intersection testing)将照常进行。
+ 否则，如果查询的位掩码和`Shape`的位掩码的按位 AND 值为零，则跳过该`Shape`

或者换句话说:

```C++
PxU32 keep = (query.word0 & object.word0)
           | (query.word1 & object.word1)
           | (query.word2 & object.word2)
           | (query.word3 & object.word3);
```

此硬编码公式可以提供简单的筛选，同时避免筛选回调的函数调用开销。例如，若要模拟 `PhysX` 两个活动组的行为，请按如下方式定义组:

```C++
enum ActiveGroup
{
    GROUP1    = (1<<0),
    GROUP2    = (1<<1),
    GROUP3    = (1<<2),
    GROUP4    = (1<<3),
    ...
};
```

创建`Shape`时，可以将它们分配给组，例如 GROUP1:

```C++
PxShape* shape;                      // Previously created shape

PxFilterData filterData;
filterData.word0 = GROUP1;
shape->setQueryFilterData(filterData);
```

或多个组，例如 GROUP1 和 GROUP3:

```C++
PxShape* shape;                      // Previously created shape

PxFilterData filterData;
filterData.word0 = GROUP1|GROUP3;
shape->setQueryFilterData(filterData);
```

执行场景查询时，选择查询处于活动状态的组(例如 GROUP2 和 GROUP3)，如下所示:

```C++
PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance
PxRaycastBuffer hit;                 // [out] Raycast results

// [in] Define what parts of PxRaycastHit we're interested in
const PxHitFlags outputFlags = PxHitFlag::eDISTANCE | PxHitFlag::ePOSITION | PxHitFlag::eNORMAL;

// [in] Raycast against GROUP2 and GROUP3
PxQueryFilterData filterData = PxQueryFilterData();
filterData.data.word0 = GROUP2|GROUP3;

bool status = scene->raycast(origin, unitDir, maxDistance, hit, outputFlags, filterData);
```

## User defined hit callbacks for unbounded results
---------------------------------------------------
查询有时可能会返回非常多的结果(例如，具有非常大的对象或对象密度较高的区域中的查询)，并且保留足够大的内存缓冲区的开销可能过高。类 `PxRaycastCallback`、`PxSweepCallback` 和 `PxOverlapCallback` 为此类方案提供了基于回调的高效解决方案。例如，具有 `PxRaycastCallback` 回调的raycast查询将通过多个虚拟`PxHitCallback::p rocessTouches()`回调返回所有touching hits:

```C++
struct UserCallback : PxRaycastCallback
{
    UserData data;
    virtual PxAgain processTouches(const PxRaycastHit* buffer, PxU32 nbHits)
        // This callback can be issued multiple times and can be used
        // to process an unbounded number of touching hits.
        // Each reported touching hit in buffer is guaranteed to be closer than
        // the final block hit after the query has fully executed.
    {
        for (PxU32 i = 0; i < nbHits; i++)
            animateLeaves(buffer[i], data);
    }
    virtual void finalizeQuery()
    {
        drawWallDecal(this->block, data);
    }
};

PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance

UserCallback cb; cb.data = ...;
scene->raycast(origin, unitDir, maxDistance, cb); // see UserCallback::processTouches
```

在此代码片段中，raycast() 查询可能会多次调用 `processTouches` ，所有touching hits都已裁剪为全局最接近的blocking hit。

+ 请注意，如果所有 `eTOUCH` 结果都不适合提供的触摸缓冲区，并且还发现了blocking hit，则查询的开销可能高达两倍。
+ 另请参阅 `PxQueryFlag::eNO_BLOCK`

## Batched queries
------------------
`PhysX` 支持通过 `PxBatchQuery` 接口对场景查询进行批处理。使用此 API 可以简化多线程实现。

`PhysX` 版本 3.4 中已弃用批处理查询(Batched queries)功能。

+ `PxBatchQuery` 接口便于同时批处理和执行多个查询。 `PxBatchQuery` 缓冲raycast, overlap and sweep queries，直到调用 `PxBatchQuery::execute()`。
+ 使用 `PxScene::createBatchQuery`(const PxBatchQueryDesc& desc) 创建一个 `PxBatchQuery` 对象。
+ 硬编码的筛选公式(hardcoded filtering equation)不用于批处理查询。取而代之的是两个过滤器着色器，分别在之前(`PxBatchQueryPreFilterShader`)和之后(`PxBatchQueryPostFilterShader`)运行精确的每个形状碰撞测试(per-shape collision test)。请参阅 `PxBatchQueryDesc::preFilterShader` 和 `PxBatchQueryDesc::p ostFilterShader`。
+ `BatchQueryFilterData::filterShaderData` 将通过 `constantBlock` 参数进行复制并传递到过滤器着色器。
+ 结果将写入 `PxBatchQueryDesc` 中的用户定义缓冲区 `PxBatchQueryMemory` ，其顺序与查询在 `PxBatchQuery` 对象中排队的顺序相同。
+ 所使用的每种查询类型(`raycast` , `overlap` , `sweep`)的结果和命中缓冲区是单独指定的。
+ 可以在每次批处理查询执行调用之前更改这些缓冲区。SDK 将为具有 NULL 结果的批处理查询生成警告，或者针对相应查询类型(raycast, overlap or sweep)命中缓冲区。

## Volume Caching
-----------------
`PxVolumeCache` 提供了一种加速场景查询的机制。此类为指定体积中的对象实现缓存，并提供类似于 `PxScene` 的 API，用于执行 `raycasts`, `overlaps`, 和 `sweeps` 。当在同一个simulation frame内或在以后的frame中多次查询同一局部空间区域内的对象时， `PxVolumeCache` 可以提供性能提升。

`PhysX` 版本 3.4 中已弃用体积缓存功能。

`PxVolumeCache` 的一些预期用例是:

+ 一个粒子系统，具有从空间定位云(spatially localized cloud)中对每个粒子执行的许多raycasts。
+ 多个短程角色控制器(short range character controller)raycasts在角色周围的同一区域内。
+ 跨多个帧缓存查询(Caching query)的结果，可以使用前一帧上的较大体积填充缓存(可能沿预期的移动方向拉伸)，然后使用较小的体积进行查询。

缓存具有最大容量，在 `PxScene::createVolumeCache()` 中分别为动态和静态对象(dynamic and static objects)指定。

出于多线程访问的目的，缓存上的任何操作都算作对场景的读取调用。

### Filling the Cache
----------------------
要填充缓存(Filling the Cache)，请调用 `PxVolumeCache::fill()`。这将在场景中查询与几何图形定义的体积重叠(overlapping)的对象，并将结果转换并存储在内部缓冲区中，直至达到静态和动态对象(static and dynamic objects)的最大大小。 `cacheVolume` 仅支持 `PxBoxGeometry` 、 `PxSphereGeometry` 和 `PxCapsuleGeometry` 。调用将始终重新填充静态和动态内部缓存(static and dynamic internal caches)，即使新体积完全位于以前的缓存体积中也是如此。它返回类型为 `PxVolumeCache::FillStatus` 的结果。

如果场景查询子系统自上次填充以来已更新，则针对缓存的后续查询(`raycasts`, `overlaps`, `sweeps`, `forEach`)将使用相同的体积自动重新填充缓存。对于静态和动态，更新状态是独立跟踪(tracked independently)的，因此查询可能只重新填充动态的缓存，同时重用静态的有效缓存结果。如果任何填充或重新填充的尝试失败，则缓存无效，任何后续查询都将尝试填充缓存。

### Querying the Cache
-------------------------
`PxVolumeCache` 提供了一个用于光线投射、扫描和重叠的 API，类似于场景查询 API。签名的主要区别在于 `PxVolumeCache` 查询不支持单对象缓存。查询结果通过 `PxVolumeCache::Iterator::shapes()` 回调报告，查询可能会多次调用回调以提供多批结果。

+ 针对有效缓存的`Raycasts` 、 `overlaps` 和 `sweep` 将仅返回与缓存体积 `overlap` 的结果，但保证返回所有此类体积。
+ 针对无效缓存的`Raycasts` 、 `overlaps` 和 `sweep` 将回退到场景查询(scene queries)。在这种情况下，可能会返回不与缓存体积重叠的结果。

由于缓存会在场景已更改的任何查询上自动重新填充，因此这两个条件可确保对完全位于缓存体积内的缓存的查询将始终返回与查询场景完全相同的形状。如果查询不完全位于缓存体积内(并且缓存有效)，则只会返回与缓存体积重叠的那些形状。如果对从未调用过 `fill()` 的缓存发出查询，则会报告错误。

缓存还提供了一种低级 `forEach()` 机制，用于循环访问缓存的对象。如果在从未调用过 `fill()` 的缓存上执行 `forEach()`，则它将返回而不报告错误。如果缓存无效，`forEach()` 将直接从场景中检索与缓存体积重叠(overlap)的形状。此过程涉及临时缓冲区的分配，如果分配失败，`forEach()` 将发出错误消息并返回。

This code snippet shows how to use `PxVolumeCache`:

```C++
PxScene* scene;
PxVec3 poi = ...;                    // point of interest
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance
PxRaycastBuffer hit;                 // [out] Raycast results
const PxU32 maxStatics = 32, maxDynamics = 8;

// persistent cache, valid until invalidated by object movement,
// insertion or deletion
PxVolumeCache* cache = scene->createVolumeCache(maxStatics, maxDynamics);
cache->setMaxNbStaticShapes(64); cache->setMaxNbDynamicShapes(16);

// fill the cache using a box geometry centered around the point of interest
cache->fill(PxBoxGeometry(PxVec3(1.0f)), PxTransform(position));

...

// Perform multiple raycast queries using the cache
PxRaycastBuffer hit;
const bool status = cache->raycast(origin, unitDir, maxDistance, hit);

// low level iterator for stored actor/shape pairs
struct UserIterator : PxVolumeCache::Iterator
{
    UserData userData;
    virtual void shapes(PxU32 count, const PxActorShape* actorShapePairs)
    {
        for (PxU32 i = 0; i < count; i++)
           doSomething(actorShapePairs[i].actor, actorShapePairs[i].shape, userData);
    }
}   iter;

// invoke UserIterator::shapes() callback for all actor/shape pairs in the cache
cache->forEach(iter);
```

## Single Object Caching
---------------------------
加速场景查询的另一种特殊情况机制是使用 `PxQueryCache` 的单对象缓存(single-object caching)。

+ 此缓存可以为任何操作模式下的`raycast`和`sweep`查询提供额外的加速和内存节省。
+ 缓存对象定义应首先测试哪个形状。对于具有高时间一致性(high temporal coherence)的查询，这可以显著提高性能。捕获这种一致性的一个好策略是简单地用上一帧的 `eBLOCK` 结果(最后一个阻塞形状)填充给定查询的缓存对象。
+ 请注意，使用过去的touching hits(用 `eTOUCH` 标志记录)进行缓存可能是不正确的，因为它将被解释为阻止并覆盖任何过滤(blocking and override any filtering)。

例如，AI 可见性查询很有可能会为多个帧返回相同的视线阻塞形状(line-of-sight blocking shape)。将光线投射查询与正确填充的 `PxQueryCache` 对象结合使用将允许 `PhysX` 在遍历内部空间分区结构之前测试单个形状，并且在"cache hit"的情况下，可以完全绕过遍历。例如:

```C++
PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance
PxRaycastBuffer hit;                 // [out] Raycast results

// Per-raycast persistent cache, valid from one frame to the next
static PxQueryCache persistentCache;

// Define cache for current frame:
// - if there was a hit in the previous frame, use the cache.
// - otherwise do not (PhysX requires given cache has a valid shape pointer)
const PxQueryCache* cache = persistentCache.shape ? &persistentCache : NULL;

// Perform a raycast query using the cache
const bool status = scene->raycast(origin, unitDir, maxDistance, hit,
                                   PxHitFlags(PxHitFlag::eDEFAULT),
                                   PxQueryFilterData(), NULL, cache);
if(status)
{
    // We hit a shape. Cache it for next frame.
    persistentCache.shape = hit.block.shape;
    persistentCache.faceIndex = hit.block.faceIndex;
}
else
{
    // We did not hit anything. Reset the cache for next frame.
    persistentCache = PxQueryCache();
}
```

缓存在查找最接近的blocking hit或使用 `eANY_HIT` 标志的查询中也很有用。在这种情况下，首先测试以前最近的对象可以使 `PhysX` 尽早缩短查询距离，从而减少总narrow phase碰撞测试，并提前退出遍历。

**`PhysX` 不检测陈旧的指针，因此在删除形状时，应用程序负责缓存对象的有效性。**

**`Overlaps` 不支持单次命中阻止缓存。**

## PxPruningStructureType
-------------------------
`PhysX SDK` 提供了不同的修剪结构(pruning structures)，用于加速场景查询。本段描述了它们之间的差异。

### Generalities
----------------
场景查询系统使用两种不同的加速结构，即分层网格(hierarchical grid)和 AABB 树。

网格在 O(n) 时间内快速构建，查询在 O(1) 和 O(N) 时间之间执行，具体取决于对象在空间中的均匀分布的程度，当所有对象都聚集成在同一网格单元中时，最坏情况下的性能为 O(N)。

树在 O(nlog(n)) 时间内构建，但具有单个结果的查询通常在 O(log(n)) 时间内运行。返回多个结果的查询将遍历更多树，最坏的情况是在 O(n) 时间内返回场景中所有对象的查询。当对象位置更改时，如果相同的拓扑维护时间过长，并且在最坏情况下，查询性能可能会降低到 O(n) 时间，则树容易受到退化的影响。

必须根据要添加或删除的对象或由于位置或者几何形状的变化时而进行的对象AABB更新(object AABB updates)来不断修改加速结构。为了最大限度地降低成本，修改将尽可能长时间地推迟。因此，添加或删除对象或更新AABB发生在摊销的恒定时间内，修改成本推迟到更改"commit"。这发生在下一个后续查询(subsequent query)或下一个 `fetchResults()` 或下一个 `fetchQueries()` 时。若要强制立即提交，请调用 `PxScene::flushQueryUpdates()` 函数。

提交过程的确切细节取决于 `PxSceneDesc` 中指定的 `staticStructure` 和 `dynamicStructure` 的值。

为避免因插入到内部场景查询数据结构中而触发自动调整大小，请提前预留空间。请参阅 `PxSceneDesc::maxNbStaticShapes` 和 `PxSceneDesc::maxNbDynamicShapes`。 

### PxPruningStructureType::eNONE
---------------------------------
加速度结构类似于分层网格((hierarchical grid))。提交更改需要完全重建。如果您希望很少或从不更新此结构中的对象，这是一个不错的选择。

### PxPruningStructureType::eSTATIC_AABB_TREE
---------------------------------------------
加速度结构是一棵树。提交更改需要完全重建。通常不建议这样做，但如果场景中的静态 `Actor` 是在初始化时创建的，并且此后不进行修改，则它可能是 `staticStructure` 的不错选择。如果经常添加或删除static geometry，则默认 `eDYNAMIC_AABB_TREE` 设置通常是更好的选择，尽管它的内存占用量高于 `eSTATIC_AABB_TREE` 。

### PxPruningStructureType::eDYNAMIC_AABB_TREE
----------------------------------------------
在这种情况下，将同时使用树和网格，并且每个查询都搜索树和网格。

该树最初是由第一个提交构建的。构建树后，提交更改将按如下方式进行:树根据它所包含对象的更新和删除进行重新拟合(refitted)。添加的对象将插入到网格中。此类添加或移除网格中当前对象，或更改网格中对象的 AABB，都会导致重新生成该对象。

此外，在 `fetchResults()` 期间，在由 `PxScene` 的 `dynamicTreeRebuildRateHint` 属性控制的多个帧上，以增量方式构建一个新树。生成开始时，它将包括当前树和网格中的所有对象。当它完成时，在一些帧之后，新树将根据自构建开始以来的任何AABB更改或删除进行重新拟合，然后替换当前树。自生成开始以来添加的任何对象都将保留在网格中。

要强制立即完全重建，请调用`PxScene::forceDynamicTreeRebuild()`。这在以下情况下可能很有用:

+ 缓慢的重建速率通常是可取的，但偶尔大量添加的对象会在网格中产生高占用率，特别是如果添加是局部的，以便仅对少数网格单元施加压力。
+ 您正在远距离移动许多对象，因为改装可能会显著降低当前树的质量

## PxSceneQueryUpdateMode
--------------------------
`PxSceneQueryUpdateMode` 可以定义在 `PxScene::fetchResults` 中完成哪些与场景查询相关的工作。

默认情况下，`fetchResults` 将在模拟期间同步更改边界，并在pruners中更新场景查询边界，这项工作是强制性的。其他工作可以是可选的，基于 `PxSceneQueryUpdateMode` :

+ `eCOMMIT_ENABLED_BUILD_ENABLED` 允许在 `fetchResults` 期间执行新的 AABB 树构建步骤，此外，还会在应用任何更改的位置调用 `pruner` 提交。在提交期间， `PhysX` 会重新调整动态场景查询树，如果构建了新树并且构建完成，则该树将与当前 AABB 树交换。
+ `eCOMMIT_DISABLED_BUILD_ENABLED` 允许在 `fetchResults` 期间执行新的 AABB 树构建步骤。不调用 Pruner 提交，这意味着在 `fetchResults` 之后的第一个场景查询期间将进行 refit，或者可能由 `PxScene::flushSceneQueryUpdates()` 方法强制进行。
+ `eCOMMIT_DISABLED_BUILD_DISABLED` 不会执行进一步的场景查询工作。场景查询更新需要手动调用，请参阅 `PxScene::sceneQueriesUpdate`。建议在 fetch 之后立即调用` PxScene::sceneQueriesUpdateResults`，因为pruning structures未更新。

## PxPruningStructure
---------------------
提供对预计算的pruning structures的访问，该结构用于加速针对新添加的 `Actor` 的场景查询。

可以为`PxScene::addActors`提供pruning structure。然后， `Actor` 场景查询形状将直接合并到场景 AABB 树中，而无需 AABB 树重新计算:

```C++
// Create pruning structure from given actors.
PxPruningStructure* ps = PxPhysics::createPruningStructure(&actors[0], (PxU32)actors.size());
// Add actors into a scene together with the precomputed pruning structure.
PxScene::addActors(*ps);
ps->release();
```

`PxPruningStructure` 对象可以与其`Actor`一起序列化为集合。

有关 `PxPruningStructure` 的使用，请参阅代码片段 SnippetPrunerSerialization。

`PxPruningStructure` 的一个典型用例是一个大型世界场景，其中紧密定位的 `actor` 块被流式传输。

### Merge process
-------------------
场景查询加速结构的合并过程因 `PxPruningStructureType` 而异:`eSTATIC_AABB_TREE` - pruning structures直接合并到场景的AABB 树中。这可能会使树失衡，建议在某个时候重新计算静态树。 `eDYNAMIC_AABB_TREE` - pruning structures被合并到临时pruning structures中，直到场景的新优化的AABB树被计算出来。
