# Rigid Body Collision

## Introduction

----------

本节将介绍刚体碰撞的基础知识。

## Shapes

---------

`Shape`描述actor的空间范围（spatial extent ）和碰撞属性（collision properties ）。它们在` PhysX` 中用于三个目的：确定刚性对象的接触特征的相交性测试(intersection tests )、场景查询测试(scene query tests )（如光线投射）以及定义触发体积(defining trigger volumes )（当其他`Shape`与它们相交时生成通知）。

每个`Shape`都包含一个` PxGeometry` 对象和一个对 `PxMaterial` 的引用，这两个对象都必须在创建时指定。下面的代码创建一个具有球体几何图形和特定材质的`Shape`：

```c++
PxShape* shape = physics.createShape(PxSphereGeometry(1.0f), myMaterial, true);
myActor.attachShape(*shape);
shape->release();
```

方法 `PxRigidActorExt::createExclusiveShape()` 等效于上面的三行。

用于 `createShape()` 的参数 "true" 通知 SDK 该`Shape`不会与其他`Actor`共享。当有许多具有相同几何图形的 `Actor` 时，可以使用形状共享来降低模拟的内存成本，但共享形状具有非常严格的限制：在共享形状附加到 `Actor` 时，您无法更新共享形状的属性。

（可选）您可以通过指定类型为 `PxShapeFlags` 的形状标志来配置`Shape`。默认情况下，形状配置为:

+ 模拟`Shape`（在模拟期间启用接触生成( contact generation )）
+ 场景查询`Shape`（为场景查询启用）
+ 如果启用了调试渲染，则可视化

为`Shape`指定几何对象时，该几何对象将复制到该`Shape`中。对于可以为`Shape`指定几何图形有一些限制，具体取决于形状标志和父角色的类型。

+ 附加到动态`Actor`的模拟`Shape`不支持``TriangleMesh`、`HeightField`和`Plane geometries`，除非动态`Actor`配置为`kinematic`。
+ 触发器`Shape`不支持`TriangleMesh`和`HeightField`几何图形。

有关更多详细信息，请参阅以下部分。

如下所示将`Shape`与`Actor`分离：

```c++
myActor.detachShape(*shape);
```

## Simulation Shapes and Scene Query Shapes

------------

`Shape`可以独立配置为参与场景查询(scene queries)和接触测试(contact tests)中的一个或两个。默认情况下，形状将同时参与这两项操作。以下伪代码配置 `PxShape` 实例，使其不再参与`Shape`对交集测试：

```c++
void disableShapeInContactTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSIMULATION_SHAPE,false);
}
```

可以将 `PxShape` 实例配置为参与`Shape`对交集测试，如下所示：

```c++
void enableShapeInContactTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSIMULATION_SHAPE,true);
}
```

要从场景查询测试中禁用`PxShape` 实例，请执行以下操作：

```c++
void disableShapeInSceneQueryTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSCENE_QUERY_SHAPE,false);
}
```

最后，可以在场景查询测试中重新启用 `PxShape` 实例：

```c++
void enableShapeInSceneQueryTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSCENE_QUERY_SHAPE,true);
}
```

**注意：如果`Shape`的`Actor`的运动根本不需要由模拟控制，那么可以通过在``Actor`上额外禁用模拟来节省内存。例如形状仅用于场景查询，并在必要时手动移动**

## Kinematic Triangle Meshes (Planes, Heighfields)

-----------

可以创建一个运动学`PxRigidDynamic`，它可以具有三角形网格（`plane`，`heighfield`）形状。如果此形状具有模拟形状标志，则此`Actor`必须保持运动学。如果将标志更改为"非模拟"，你甚至可以切换运动标志。要设置运动三角形网格，请参阅以下代码：

```c++
PxRigidDynamic* meshActor = getPhysics().createRigidDynamic(PxTransform(1.0f));
PxShape* meshShape;
if(meshActor)
{
    meshActor->setRigidDynamicFlag(PxRigidDynamicFlag::eKINEMATIC, true);

    PxTriangleMeshGeometry triGeom;
    triGeom.triangleMesh = triangleMesh;
    meshShape = PxRigidActorExt::createExclusiveShape(*meshActor,triGeom,
        defaultMaterial);
    getScene().addActor(*meshActor);
}
```

要将运动三角形网格`Actor`切换为动态`Actor`：

```c++
PxRigidDynamic* meshActor = getPhysics().createRigidDynamic(PxTransform(1.0f));
PxShape* meshShape;
if(meshActor)
{
    meshActor->setRigidDynamicFlag(PxRigidDynamicFlag::eKINEMATIC, true);

    PxTriangleMeshGeometry triGeom;
    triGeom.triangleMesh = triangleMesh;
    meshShape = PxRigidActorExt::createExclusiveShape(*meshActor, triGeom,
        defaultMaterial);
    getScene().addActor(*meshActor);

    PxConvexMeshGeometry convexGeom = PxConvexMeshGeometry(convexBox);
    convexShape = PxRigidActorExt::createExclusiveShape(*meshActor, convexGeom,
        defaultMaterial);
```

## Broad-phase Algorithms

---------

`PhysX` 支持多种粗检测算法（broad-phase algorithms）：

+ *sweep-and-prune (SAP)* 
+ *multi box pruning (MBP)* 

`PxBroadPhaseTyp::eSAP` 是 `PhysX 3.2` 之前使用的默认算法。这是一个很好的通用选择，当许多物体处于睡眠状态时，它具有出色的性能。但是，当所有对象都在移动时，或者当在粗检测(broad-phase)中添加或删除大量对象时，性能可能会显著下降。此算法不需要定义世界边界(world bounds)即可工作。`PxBroadPhaseType::eMBP` 是 `PhysX 3.3 `中引入的一种新算法。它是一种替代的粗检测(broad-phase)算法，当所有对象都在移动或插入大量对象时，它不会遇到与`eSAP`相同的性能问题。但是，当许多对象处于休眠状态时，其通用性能可能不如 eSAP，并且它要求用户定义世界边界(world bounds)才能工作。所需的粗检测(broad-phase)算法由`PxSceneDesc`结构中的`PxBroadPhaseType`枚举控制。

## Regions of Interest

------------

感兴趣的区域(Regions of Interest)是围绕粗检测(broad-phase)控制的体积空间中的世界空间AABB包围盒。这些区域中包含的对象由粗检测(broad-phase)正确处理。落在这些区域之外的对象将丢失所有碰撞检测。理想情况下，这些区域应覆盖整个游戏空间，同时限制覆盖的空白空间的数量。区域可以重叠，但为了获得最大效率，建议尽可能减少区域之间的重叠量。请注意，AABB 仅接触的两个区域不被视为重叠。例如，`PxBroadPhaseExt::createRegionsFromWorldBounds` helper function通过将给定的世界 AABB 细分为常规 2D 网格来创建许多非重叠区域边界。区域可以由`PxBroadPhaseRegion` 结构定义，也可以由分配给它们的用户数据定义。它们可以在场景创建时或在运行时使用 `PxScene::addBroadPhaseRegion` 函数进行定义。SDK 返回分配给新创建区域的句柄，稍后可以使用 `PxScene::removeBroadPhaseRegion` 函数删除区域。新添加的区域可能会重叠已存在的对象。如果设置了 `PxScene::addBroadPhaseRegion` 调用中的 `populateRegion` 参数，SDK 可以自动将这些对象添加到新区域。但是，此操作并不便宜，并且可能会对性能产生很大影响，尤其是在同一帧中添加多个区域时。因此，建议尽可能禁用它。该区域将被创建为空，并且只会在创建区域后添加到场景中的对象填充，或者在更新时（即当它们移动时）使用先前存在的对象填充该区域。请注意，只有`PxBroadPhaseType::eMBP`需要定义区域。`PxBroadPhaseType::eSAP` 算法则不然。此信息在`PxBroadPhaseCaps`结构中捕获，该结构列出了每个粗检测(broad-phase)算法的信息和功能。此结构可以通过 `PxScene::getBroadPhaseCaps` 函数检索。有关当前区域的运行时信息可以使用 `PxScene::getNbBroadPhaseRegions` 和 `PxScene::getBroadPhaseRegions` 函数进行检索。区域的最大数量目前限制为 256 个。

## Broad-phase Callback

----------

可以在 `PxSceneDesc` 结构中定义与粗检测位相关(broad-phase-related )的事件的回调。当在指定的感兴趣区域（即"越界"）中发现对象时，将调用此`PxBroadPhaseCallback`对象。SDK 会禁用这些对象的碰撞检测。一旦对象重新进入有效区域，它就会自动重新启用。由用户决定如何处理越界对象。典型选项包括：

+ 删除对象
+ 让他们继续运动而不会发生碰撞，直到他们重新进入有效区域
+ 手动将他们传送回有效位置

## Collision Filtering

-------------
在几乎所有应用程序中，除了微不足道的应用程序之外，都需要免除某些对象对的相交性（interacting），或者以特定方式为交互对（interacting pair）配置 SDK 碰撞检测行为。在潜艇样本中，如上所述，当潜艇接触水雷或水雷链条时，我们需要得到通知，以便我们可以将它们炸毁。crab的AI还需要知道crab何时接触到高度场。在了解示例如何实现此目的之前，我们需要了解 SDK filtering系统的可能性。由于filtering可能交互对（interacting pair）发生在仿真引擎的最深处，并且需要应用于彼此靠近的所有对象对，因此它对性能特别敏感。实现它的最简单方法是始终为每个可能交互的对（interacting pair）调用回调函数，其中应用程序基于两个对象指针可以使用一些自定义逻辑（例如咨询其游戏数据库）确定该对是否应该进行交互。不幸的是，如果对于一个非常大的游戏世界来说，这会变得太慢，特别是如果碰撞检测处理发生在远程处理器（如GPU）或具有本地内存的其他类型的矢量处理器上，这些处理器必须暂停其并行计算，中断运行游戏代码的主处理器，并让它在继续之前执行回调。即使它要在CPU上执行，它也可能会在多个内核或超线程上同时执行，并且必须放置线程安全代码以确保对共享数据的并发访问是安全的。更好的做法是使用某种可以在远程处理器上执行的固定函数逻辑。这就是我们在 PhysX 2.x 中所做的 - 不幸的是，我们提供的基于组的简单过滤规则（simple group based filtering rules）不够灵活，无法涵盖所有应用程序。在3.0中，我们引入了一个着色器系统，它允许开发人员使用在矢量处理器上运行的代码实现任意规则系统(因此无法访问主内存中的任何最终游戏数据库(eventual game data base ))，这比2.x固定函数过滤更灵活，但同样高效，还有一个完全灵活的回调机制，其中过滤器着色器调用能够访问任何应用程序数据的CPU回调函数， 以性能为代价 -- 有关详细信息，请参阅 `PxSimulationFilterCallback`。最好的部分是，应用程序可以基于每对(per-pair)进行决策，以进行速度与灵活性的权衡。

让我们先看一下着色器系统：下面是由 `SampleSubmarine` 实现的`filter shader`：
```C++
PxFilterFlags SampleSubmarineFilterShader(
    PxFilterObjectAttributes attributes0, PxFilterData filterData0,
    PxFilterObjectAttributes attributes1, PxFilterData filterData1,
    PxPairFlags& pairFlags, const void* constantBlock, PxU32 constantBlockSize)
{
    // let triggers through
    if(PxFilterObjectIsTrigger(attributes0) || PxFilterObjectIsTrigger(attributes1))
    {
        pairFlags = PxPairFlag::eTRIGGER_DEFAULT;
        return PxFilterFlag::eDEFAULT;
    }
    // generate contacts for all that were not filtered above
    pairFlags = PxPairFlag::eCONTACT_DEFAULT;

    // trigger the contact callback for pairs (A,B) where
    // the filtermask of A contains the ID of B and vice versa.
    if((filterData0.word0 & filterData1.word1) && (filterData1.word0 & filterData0.word1))
        pairFlags |= PxPairFlag::eNOTIFY_TOUCH_FOUND;

    return PxFilterFlag::eDEFAULT;
}
```
`SampleSubmarineFilterShader`是一个简单的着色器函数，它是`PxFiltering.h`中声明的`PxSimulationFilterShader`原型的实现。`shader filter function`（如上称为 `SampleSubmarineFilterShader`）可能不引用函数的参数及其自己的局部堆栈变量以外的任何内存，因为该函数可以在远程处理器上编译和执行。`SampleSubmarineFilterShader()` 将用于所有彼此靠近的形状对(pairs of shapes) - 更准确地说：对于在世界空间中发现轴对齐边界框首次相交的所有形状对。除此之外的所有行为都由 `SampleSubmarineFilterShader()` 返回的内容决定。`SampleSubmarineFilterShader())` 的参数包括两个对象的 `PxFilterObjectAttributes` 和 `PxFilterData`，以及一个常量内存块。请注意，指向这两个对象的指针不会传递，因为这些指针引用计算机的主内存，并且正如我们所说，这可能不适用于着色器，因此指针不是很有用，因为取消引用它们可能会导致崩溃。`PxFilterObjectAttributes`和`PxFilterData`旨在包含可以从指针中快速收集的所有有用信息。`PxFilterObjectAttributes`是32位数据，用于编码对象的类型：例如`PxFilterObjectType::eRIGID_STATIC`，`PxFilterObjectType::eRIGID_DYNAMIC`，甚至`PxFilterObjectType::ePARTICLE_SYSTEM`。此外，它还可以让您了解对象是运动学的(kinematic,)还是触发器。`PhysX`中的每个`PxShape`和`PxParticleBase`对象都有一个类型为`PxFilterData`的成员变量。这是 128 位用户定义的数据，可用于存储与碰撞过滤(collision filtering)相关的应用程序特定信息。这是传递给每个对象的`SampleSubmarineFilterShader()` 的另一个变量。对于常量块。这是应用程序可以提供给着色器进行操作的每个场景的全局信息块。您将需要使用它来编码有关要过滤的内容和不过滤的内容的规则。最后，`SampleSubmarineFilterShader()` 还有一个 `PxPairFlags` 参数。这是一个输出，类似于返回值 `PxFilterFlags`，尽管用法略有不同。`PxFilterFlags` 告诉 SDK 它是否应该永久忽略该对 （eKILL），在重叠时忽略该对，但在过滤其中一个对象的相关数据更改时再次询问 （eSUPPRESS），或者在着色器无法决定时调用性能较低但更灵活的 CPU 回调（eCALLBACK）。`PxPairFlags` 指定了其他标志，这些标志代表模拟将来应针对此对执行的操作。例如，`eNOTIFY_TOUCH_FOUND`意味着当配对真正开始接触时通知用户，而不仅仅是潜在的。

让我们看看上面的着色器是做什么的：

```C++
// let triggers through
if(PxFilterObjectIsTrigger(attributes0) || PxFilterObjectIsTrigger(attributes1))
{
    pairFlags = PxPairFlag::eTRIGGER_DEFAULT;
    return PxFilterFlag::eDEFAULT;
}
```
这意味着，如果任一对象是触发器，则执行默认触发器行为（通知应用程序有关触摸的开始和结束），否则在它们之间执行"默认"碰撞检测。

```C++
// generate contacts for all that were not filtered above
pairFlags = PxPairFlag::eCONTACT_DEFAULT;

// trigger the contact callback for pairs (A,B) where
// the filtermask of A contains the ID of B and vice versa.
if((filterData0.word0 & filterData1.word1) && (filterData1.word0 & filterData0.word1))
    pairFlags |= PxPairFlag::eNOTIFY_TOUCH_FOUND;

return PxFilterFlag::eDEFAULT;
```

这表示对于所有其他对象，请执行"默认"碰撞处理。此外，还有一个基于 `filterDatas` 的规则，用于确定我们要求触摸通知(touch notifications)的特定对。要理解这意味着什么，我们需要知道样本赋予 `filterDatas` 的特殊含义。

示例的需求非常基础，因此我们将使用非常简单的方案来处理它。该示例首先使用自定义枚举为不同的对象类型提供命名代码：

```C++
struct FilterGroup
{
    enum Enum
    {
        eSUBMARINE     = (1 << 0),
        eMINE_HEAD     = (1 << 1),
        eMINE_LINK     = (1 << 2),
        eCRAB          = (1 << 3),
        eHEIGHTFIELD   = (1 << 4),
    };
};
```

该示例通过将每个`Shape`的 `PxFilterData：：word0` 分配给此筛选器组类型来标识每个`Shape`的类型。然后，它放置一个位掩码，指定每种类型的对象，当被 `word0` 类型的对象触摸到 `word1` 中时，这些对象应生成报告。每当创建`Shape`时，都可以在示例中执行此操作，但由于`Shape`创建封装在 `SampleBase` 中，因此在事后使用以下函数完成：
```C++
void setupFiltering(PxRigidActor* actor, PxU32 filterGroup, PxU32 filterMask)
{
    PxFilterData filterData;
    filterData.word0 = filterGroup; // word0 = own ID
    filterData.word1 = filterMask;  // word1 = ID mask to filter pairs that trigger a
                                    // contact callback;
    const PxU32 numShapes = actor->getNbShapes();
    PxShape** shapes = (PxShape**)SAMPLE_ALLOC(sizeof(PxShape*)*numShapes);
    actor->getShapes(shapes, numShapes);
    for(PxU32 i = 0; i < numShapes; i++)
    {
        PxShape* shape = shapes[i];
        shape->setSimulationFilterData(filterData);
    }
    SAMPLE_FREE(shapes);
}
```

这将设置属于传递的 `actor` 的每个`Shape`的 `PxFilterDatas`。以下是如何在 `SampleSubmarine` 中使用它的一些示例：
```C++
setupFiltering(mSubmarineActor, FilterGroup::eSUBMARINE, FilterGroup::eMINE_HEAD |
    FilterGroup::eMINE_LINK);
setupFiltering(link, FilterGroup::eMINE_LINK, FilterGroup::eSUBMARINE);
setupFiltering(mineHead, FilterGroup::eMINE_HEAD, FilterGroup::eSUBMARINE);

setupFiltering(heightField, FilterGroup::eHEIGHTFIELD, FilterGroup::eCRAB);
setupFiltering(mCrabBody, FilterGroup::eCRAB, FilterGroup::eHEIGHTFIELD);
```

这个方案可能过于简单，无法在实际游戏中使用，但它显示了filter shader的基本用法，并且它将确保为所有有用的对（interesting pairs）调用`SampleSubmarine::onContact()`。在扩展函数 `PxDefaultSimulationFilterShader` 中提供了另一种基于组的过滤机制和源。而且，同样，如果这个基于着色器的系统太不灵活，请考虑使用`PxSimulationFilterCallback`提供的回调方法。

## Aggregates
---------------
聚合是`actor`的集合。聚合不提供额外的模拟或查询功能，但允许您告诉 SDK 一组`actor`将聚类在一起，这反过来又允许 SDK 优化其空间数据操作。一个典型的用例是布娃娃，由多个不同的`actor`组成。如果没有聚集体，这会产生与布娃娃中的形状一样多的粗检测实体(broad-phase entries)。通常，将布娃娃在广义阶段表示为单个实体，并在必要时在第二次通过时执行内部重叠测试会更有效。另一个潜在的用例是具有大量附加`Shape`的单个Actor。

## Creating an Aggregate
----------
从 `PxPhysics` 对象创建聚合：

```C++
PxPhysics* physics; // The physics SDK object

PxU32 nbActors;     // Max number of actors expected in the aggregate
bool selfCollisions = true;

PxAggregate* aggregate = physics->createAggregate(nbActors, selfCollisions);
```

目前，`actor`的最大数量限制为128个，为了提高效率，应尽可能低。如果永远不需要聚合的`actor`之间的碰撞检测，请在创建时禁用它们。这比使用场景过滤机制更有效，因为它绕过了所有内部过滤逻辑。典型的用例是静态或运动学`actor`的聚合。请注意，最大`actor`个数和自碰撞属性（self-collision attribute）都是不可变的。

## Populating an Aggregate
-------
将`actor`添加到聚合中，如下所示：
```C++
PxActor& actor;    // Some actor, previously created
aggregate->addActor(actor);
```

请注意，如果 `Actor` 已属于某个场景，则调用将被忽略。将 `Actor` 添加到聚合，然后将聚合添加到场景中，或者将聚合添加到场景中，然后将 `Actor` 添加到聚合中。

要将聚合添加到场景中（在填充之前或之后）：

```C++
scene->addAggregate(*aggregate);
```

同样，要从场景中移除聚合，请执行以下操作：

```C++
scene->removeAggregate(*aggregate);
```

## Releasing an Aggregate
------------
要释放聚合:
```C++
PxAggregate* aggregate;    // The aggregate we previously created
aggregate->release();
```

释放 `PxAggregate` 不会释放聚合的`actors`。如果 `PxAggregate` 属于某个场景，则 `Actor` 会自动重新插入到该场景中。如果您打算同时删除 `PxAggregate` 及其`actors`，则最有效的方法是先释放`actor`，然后在 `PxAggregate` 为空时释放它。

## Amortizing Insertion
-------
在一帧中向场景中添加多个对象可能是一项代价高昂的操作。布娃娃就是这种情况，如前所述，这是 `PxAggregate` 的良好候选者。另一种情况是局部碎片，其自碰撞(self-collisions)经常被禁用。要将对象插入粗检测结构(broad-phase structure)的成本摊销到多个阶段结构中，请在 `PxAggregate` 中生成碎片，然后从聚合中删除每个 `Actor` ，然后通过这些帧将其重新插入到场景中。

## Trigger Shapes
----------
Trigger Shapes在场景模拟中不起作用（尽管它们可以配置为参与场景查询）。相反，它们的作用是报告与另一种`shape`重叠。不会为相交性测试生成触点，因此触点报告不适用于Trigger Shapes。此外，由于触发器在模拟中不起作用，SDK 将不允许同时引发`eSIMULATION_SHAPE`和 `eTRIGGER_SHAPE`标志;也就是说，如果引发一个标志，则引发另一个标志的尝试将被拒绝，并且错误将传递到错误流。

`SampleSubmarine`中使用了Trigger Shapes来确定潜艇是否已经到达宝藏。在下面的代码中，表示宝藏的 `PxActor` 将其单独形状配置为Trigger Shapes：

```C++
PxShape* treasureShape;
gTreasureActor->getShapes(&treasureShape, 1);
treasureShape->setFlag(PxShapeFlag::eSIMULATION_SHAPE, false);
treasureShape->setFlag(PxShapeFlag::eTRIGGER_SHAPE, true);
```

与Trigger Shapes的相交性测试通过 `PxSampleSubmarine` 类（`PxSimulationEventCallback` 的子类）中的` PxSimulationEventCallback::onTrigger` 的实现在 `SampleSubmarine` 中报告：

```C++
void SampleSubmarine::onTrigger(PxTriggerPair* pairs, PxU32 count)
{
    for(PxU32 i=0; i < count; i++)
    {
        // ignore pairs when shapes have been deleted
        if (pairs[i].flags & (PxTriggerPairFlag::eREMOVED_SHAPE_TRIGGER |
            PxTriggerPairFlag::eREMOVED_SHAPE_OTHER))
            continue;

        if ((&pairs[i].otherShape->getActor() == mSubmarineActor) &&
            (&pairs[i].triggerShape->getActor() == gTreasureActor))
        {
            gTreasureFound = true;
        }
    }
}
```

上面的代码循环访问涉及Trigger Shapes的所有重叠形状对。如果发现宝藏已被潜艇触摸，则旗帜`gTreasureFound`为真。

## Interactions
--------------
SDK 在内部为粗检测(broad-phase)报告的每个重叠对创建一个交互对象。这些对象不仅为成对碰撞的刚体创建，而且还为重叠的触发器对创建。一般来说，用户应该假设无论涉及对象的类型（刚体，触发器，布料等）以及所涉及的`PxFilterFlag`标志如何，都创建了此类对象。目前，每个`Actor`的此类交互对象限制为 65535 个。如果超过 65535 个交互涉及同一个`Actor`，则 SDK 会输出一条错误消息，并忽略额外的交互。
