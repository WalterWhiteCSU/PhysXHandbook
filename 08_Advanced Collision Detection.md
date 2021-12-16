# Advanced Collision Detection
---------------------------------
## Tuning Shape Collision Behavior
-----------------------------------
用于产生接触的`Shape`会影响它们通过contacts(Contact point)附着(attached)着的dynamic rigid bodies。约束求解器在contacts处产生脉冲力，以保持`Shape`静止或移动而不相互传递。`Shape`有两个重要参数，用于控制碰撞检测如何生成它们之间的contacts，而这些contacts又是碰撞或堆叠(stacking)时它们行为的核心： **`contactOffset`** 和 **`restOffset`** 。它们分别使用`PxShape::setContactOffset()`和`PxShape::setRestOffset()`进行设置。这两个值都不直接使用。碰撞检测始终对一对可能碰撞的`Shape`进行操作，并且始终考虑两个`Shape`的偏移总和。我们分别称这些为 **"contactDistance"** 和 **"restDistance"**。

碰撞检测能够在两个`Shape`之间产生contacts，当它们仍然相距很远时，当它们完全接触时，或者当它们相互穿透时。为了使讨论更简单，我们将相互渗透视为负距离。因此，两个`Shape`之间的距离可以是正数、零形或负数。 `SeparationDistance` 是碰撞检测开始生成接触的距离。它必须大于零，这意味着当两个`Shape`穿透时， `PhysX` 将始终生成`contact`(除非以某种方式完全禁用了两个`Shape`之间的碰撞检测，例如过滤)。默认情况下，在 `PxTolerancesScale` 中使用公制单位和默认缩放时， `contactOffset` 为 0.02，这意味着 `contactDistance` 的计算值将达到 4 厘米。因此，当两个`Shape`在4厘米内彼此接近时，将产生`contact`，直到它们再次移动到4厘米以外的距离。

然而，contacts的产生并不意味着会立即在这些位置施加大脉冲来分离`Shape`，甚至阻止在穿透方向上进一步运动。除非将仿真时间步长选择为很小，否则这将使仿真抖动变小，这对于实时性能是不可取的。相反，我们希望接触处的力随着穿透力的增加而平滑增加，直到它达到足够高的值以阻止任何进一步的穿透运动。达到最大力的距离是静止距离，因为在这个距离上，相互堆叠的两个`Shape`将达到静态平衡并静止。当`Shape`由于某种原因被推在一起，以至于它们的距离低于静止时，会施加更大的力将它们推开，直到它们再次处于静止距离。随着距离的变化而施加的力的变化不一定是线性的，但它是平滑和连续的，即使在大的时间步长下也能产生令人愉悦的模拟。

在为`Shape`选择 `contactOffset` 和 `restOffset` 时，需要考虑一些不同的事项。通常，相同的值可用于模拟中的所有`Shape`。首先确定 `restOffset` 是有意义的。目标通常是使`graphics shapes`看起来堆叠在一起，使它们完全接触，就像现实生活中的身体一样。如果`collision shapes`的大小与图形`Shape`的大小完全相同，则需要 `restOffset` 为零。如果碰撞`Shape`的 epsilon 大于图形`Shape`，则负 epsilon 的 `restOffset` 是正确的。这将使较大的碰撞`Shape`相互下沉，直到较小的图形`Shape`也接触。rest大于零的偏移是实际的，例如，如果在三角形几何体上滑动时存在问题，其中基于穿透的接触生成比分离contacts更难以产生平滑的contacts，从而导致更平滑的滑动。

确定 `restOffset` 后，应选 `contactOffset` 的值稍大一些。经验法则是使两者之间的差异尽可能小，从而在仿真使用的时间步长大时仍能有效避免抖动。较大的时间步长需要更大的差异。将其设置得太大的缺点是，当两个`Shape`接近时，将更快地生成`contact`，这会增加仿真必须担心的`contact`总数。这将降低性能。此外，仿真代码通常假设contacts靠近凸形的表面。如果接触偏移非常大，则此假设会崩溃，从而导致行为伪影。

## Contact Modification
--------------------------
在某些情况下，可能需要specialize contact behavior。例如，为了实现sticky contacts，使物体看起来彼此floating或swimming，或者使物体穿过墙壁上的明显孔。实现此类效果的一种简单方法是让用户在通过碰撞检测生成contacts后，在接触求解器之前更改`contact`的属性。由于这两个步骤都发生在 `scene simulate()` 函数中，因此必须使用回调。

对于用户已在filter shader中为其指定了标志 `PxPairFlag::eMODIFY_CONTACTS` 的所有碰撞'Shape'对，都会发生回调。

若要侦听这些修改回调，请从类 `PxContactModifyCallback` 派生：

```C++
class MyContactModification : public PxContactModifyCallback
{
    ...
    void onContactModify(PxContactModifyPair* const pairs, PxU32 count);
};
```

然后实现 `PxContactModifyCallback` 的 `onContactModify` 函数：

```C++
void MyContactModification::onContactModify(PxContactModifyPair *const pairs, PxU32 count)
{
    for(PxU32 i=0; i<count; i++)
    {
        ...
    }
}
```

每对'Shape'都附带一组contacts，这些contacts具有许多可以修改的属性，例如position、接触法线(contact normal)和separation。目前，`contact`的恢复(restitution)和摩擦性能(friction)不能改变。请参阅 `PxModifiableContact` 和 `PxContactSet` ，了解可以修改的属性。

除了修改 `contact` 属性外，还可以：

+ 为每个 `contact` 设置目标速度
+ 限制每个`contact`施加的最大脉冲
+ 分别调整每个主体的逆质量(inverse mass)和反惯性刻度(inverse inertia scales)

通过设置目标速度可以实现类似传送带的效果。通过使目标速度与接触法线的切向方向运行，可以获得最佳结果，但求解器也支持接触法线方向上的目标速度。

用户可以通过限制每个`contact`上施加的最大脉冲力来限制施加在每个`contact`上的脉冲力。这可用于产生"软"接触效应，例如，给人以由于压缩而导致的能量耗散的印象，或限制由于kinematic碰撞而施加在动态物体上的脉冲力。请注意，限制最大脉冲力可能会导致额外的穿透和物体相互传递。

调整质量和惯性尺度可用于调整一对物体之间的接触如何影响物体的线性和角速度。接触对中的每个物体都有一个单独的逆质量和反惯性刻度。这些刻度初始化为 1，可以作为回调的一部分进行调整。请注意，这些值在`contact`对内执行局部质量修改，并影响接触对内的所有`contact`。

均匀地缩放物体的逆质量和反惯性相同的值会导致`bodies`的行为像一个物体，根据所使用的值，它要么更重，要么更轻。提供<1的逆质量/反向惯性量表会导致身体看起来更重;提供>1的尺度会导致身体看起来更轻。例如，0.5的逆质量/惯性尺度导致身体看起来具有两倍的质量。提供4的逆质量/惯性尺度将导致身体看起来具有其原始质量的四分之一。提供0的逆质量/惯性量表会导致身体表现得好像它有无限的质量。

但是，也可以通过为物体的逆质量和反惯性尺度提供不同的值来非均匀地缩放物体的逆质量和反惯性。例如，可以通过仅调整惯性反比尺度来减少或增加接触引起的角速度变化量。这种修改的用例非常依赖于游戏，但可能涉及，例如，在街机风格的驾驶游戏中调整玩家的车辆和交通车辆之间的交互，其中玩家的汽车预计会被交通车辆撞倒，但如果汽车因碰撞而旋转出来，玩家会感到非常沮丧。这也可以通过使交通车辆比玩家的车辆轻得多来实现，但这可能会使交通车辆看起来"太轻"，从而损害玩家的沉浸感。

在执行局部质量修改时，PxSimulationEventCallback::onContact() 中报告的脉冲将相对于参与该接触的物体的局部缩放质量。因此，报告的这种脉冲力可能不再准确地反映由给定接触引起的动量变化。为了解决这个问题，我们在刚体延伸中提供了以下方法，以提取由接触引起的线性和角脉冲和速度变化，使用局部质量修改：

```C++
static void computeLinearAngularImpulse(const PxRigidBody& body, const PxTransform& globalPose,const PxVec3& point, const PxVec3& impulse, const PxReal invMassScale,const PxReal invInertiaScale, PxVec3& linearImpulse, PxVec3& angularImpulse);

static void computeVelocityDeltaFromImpulse(const PxRigidBody& body,const PxTransform& globalPose, const PxVec3& point, const PxVec3& impulse,const PxReal invMassScale, const PxReal invInertiaScale, PxVec3& deltaLinearVelocity,PxVec3& deltaAngularVelocity);
```

这些方法返回单独的线性和角脉冲力和速度变化值，以反映质量和惯性可能已不均匀缩放的事实。当使用局部质量修饰时，可能需要为对中的每个物体的每个contacts提取单独的线性和角脉冲力。请注意，提供这些辅助功能是为了向用户提供准确的脉冲值，绝不是强制性的。对于简单的用例，例如基于脉冲阈值的触发效应或损坏，即使使用了局部质量修改，`contact reports`报告的单一脉冲值也应该是完全可以接受的。但是，如果已经使用了局部质量修改，并且脉冲值用于更复杂的行为，例如布娃娃的平衡控制，那么很可能需要这些辅助功能来实现正确的行为。请注意，在铰接(articulations)的情况下，computeLinearAngularImpulse 将返回应用于相应 `articulations` 的正确脉冲力。但是，computeVelocityDeltaFromImpulse 不会返回articulation的正确速度变化，因为它不考虑铰接的任何其他链接的影响。

此外，使用局部批量修改(local mass modification)时，必须考虑以下事项：

+ 回调的强制阈值(Force thresholding)将基于`contact reports`中的标量脉冲值。这是使用物体的缩放质量/惯性计算的，因此使用质量缩放可能需要重新调整这些阈值。
+ 求解器中在按比例质量/惯量操作的脉冲值上发生最大脉冲钳位。因此，在使用质量缩放的情况下，从 computeLinearAngularImpulse(...) 计算的施加脉冲力的大小可能会超过maxImpulse。在使用均匀质量缩放的情况下，线性脉冲的大小不会超过massScale*maxImpulse，角脉冲的大小不会超过inertiaScale*maxImpulse。

由于回调来自 SDK 内部深处，因此对回调有一些特殊要求。特别是，回调应该是线程安全和可重入的。换句话说，SDK可以从任何线程调用onContactModify()，并且可以同时调用(即，要求同时处理 `contact` 修改对的集合)。

可以使用 `PxSceneDesc` 的 `contactModifyCallback` 成员或 `PxScene` 的 `setContactModifyCallback()` 方法来设置`contact`修改回调。

## Contact reporting
--------------------
下面是 `SampleSubmarine` 中的 `contact event function` 的示例：

```C++
void SampleSubmarine::onContact(const PxContactPairHeader& pairHeader,
    const PxContactPair* pairs, PxU32 nbPairs)
{
    for(PxU32 i=0; i < nbPairs; i++)
    {
        const PxContactPair& cp = pairs[i];

        if(cp.events & PxPairFlag::eNOTIFY_TOUCH_FOUND)
        {
            if((pairHeader.actors[0] == mSubmarineActor) ||
                (pairHeader.actors[1] == mSubmarineActor))
            {
                PxActor* otherActor = (mSubmarineActor == pairHeader.actors[0]) ?
                    pairHeader.actors[1] : pairHeader.actors[0];
                Seamine* mine =  reinterpret_cast<Seamine*>(otherActor->userData);
                // insert only once
                if(std::find(mMinesToExplode.begin(), mMinesToExplode.end(), mine) ==
                    mMinesToExplode.end())
                    mMinesToExplode.push_back(mine);

                break;
            }
        }
    }
}
```

`SampleSubmarine` 是 `PxSimulationEventCallback` 的子类。 `onContact` 接收已为其触发所请求contact events的对。上述函数仅对eNOTIFY_TOUCH_FOUND事件感兴趣，每当两个 `shapes` 开始接触时，这些事件就会升高。事实上，它只对潜艇的touch events感兴趣 - 这在第二个if语句中被检查。然后，它继续假设第二个 `actor` 是地雷(这在本例中有效，因为示例的配置使得在涉及潜艇 `actor` 时不会发送其他contact reports)。之后，它将地雷添加到一组应在下次更新期间爆炸的地雷中。

**默认情况下，不会报告kinematic rigid bodies与kinematic和static rigid bodies之间的碰撞。要启用这些报告，请在创建场景时使用 `PxSceneDesc::kineKineFilteringMode` 和 `PxSceneDesc::staticKineFilteringMode` 参数。**

通常，用户只对contact reports感兴趣，如果影响力(force of impact)大于某个阈值。这样可以减少需要处理的报告对的数量。要利用此选项，需要进行以下附加配置：

+ 使用 `PxPairFlag::eNOTIFY_THRESHOLD_FORCE_FOUND`、`PxPairFlag::eNOTIFY_THRESHOLD_FORCE_PERSISTS`、`PxPairFlag::eNOTIFY_THRESHOLD_FORCE_LOST` 而不是 `PxPairFlag::eNOTIFY_TOUCH_FOUND` 等。
+ 通过 `PxRigidDynamic::setContactReportThreshold()` 指定dynamic rigid body的阈值力。如果body与其他物体碰撞，并且接触力高于阈值，则将发送报告(如果根据对(pair)的 `PxPairFlag` 设置启用)。如果两个碰撞的dynamic bodies都指定了力阈值，则将使用较低的阈值。

**如果一个动态刚体与多个静态物体碰撞，那么所有这些`contact`的冲击力将被汇总并用于与力阈值进行比较。换句话说，即使对每个单个静态物体的冲击力低于阈值，如果这些力的总和超过阈值，仍将发送每对的接触报告。**

### Contact Reports and CCD
---------------------------
如果启用了多次通过的连续碰撞检测(CCD)，则快速移动的物体可能会在单个仿真步骤中多次反弹和弹出同一物体。默认情况下，在这种情况下，只有第一次影响(impact)才会报告为 `eNOTIFY_TOUCH_FOUND` 事件。为了同时获取其他影响的事件，必须为碰撞对提高 `PxPairFlag eNOTIFY_TOUCH_CCD`。这将触发非主要影响的 `eNOTIFY_TOUCH_CCD` 事件。出于性能原因，系统不能总是判断接触对(contact pair)是否在以前的CCD传递中失去了联系，因此也不能总是判断contact是新的还是持续的。 `eNOTIFY_TOUCH_CCD` 仅报告在CCD通过期间检测到两个碰撞物体接触的时间。

## Extracting Contact information
---------------------------------
`onContact` 模拟事件允许对给定 `PxContactPair` 的所有`contact`进行只读访问。在以前的版本中，这些对象可作为 `PxContactPoint` 对象的平展数组(flattened array)提供。但是，PhysX 3.3 为此数据引入了一种新的格式：压缩的接触流(the compressed contact stream)。现在，接触信息(contact information)被压缩为给定 `PxContactPair` 的适当格式，具体取决于某些属性，例如，取决于所涉及的`Shape`，`contact`的属性，材料以及`contact`是否可修改。

由于存在大量不同格式的组合，因此为用户提供了两种内置机制来访问接触数据(contact data)。第一种方法提供了一种从用户缓冲区中提取接触的机制，可以按如下方式使用：

```C++
void MySimulationCallback::onContact(const PxContactPairHeader& pairHeader,
    const PxContactPair* pairs, PxU32 nbPairs)
{
    const PxU32 bufferSize = 64;
    PxContactPairPoint contacts[bufferSize];
    for(PxU32 i=0; i < nbPairs; i++)
    {
        const PxContactPair& cp = pairs[i];

        PxU32 nbContacts = pairs[i].extractContacts(contacts, bufferSize);
        for(PxU32 j=0; j < nbContacts; j++)
        {
            PxVec3 point = contacts[j].position;
            PxVec3 impulse = contacts[j].impulse;
            PxU32 internalFaceIndex0 = contacts[j].internalFaceIndex0;
            PxU32 internalFaceIndex1 = contacts[j].internalFaceIndex1;
            //...
        }
    }
}
```

此方法需要将数据复制到临时缓冲区才能访问它。第二种方法允许用户循环访问接触信息，而无需提取自己的副本：

```C++
void MySimulationCallback::onContact(const PxContactPairHeader& pairHeader,
    const PxContactPair* pairs, PxU32 nbPairs)
{
    for(PxU32 i=0; i < nbPairs; i++)
    {
        const PxContactPair& cp = pairs[i];

                    PxContactStreamIterator iter(cp.contactPatches, cp.contactPoints, cp.getInternalFaceIndices(), cp.patchCount, cp.contactCount);

        const PxReal* impulses = cp.contactImpulses;

        PxU32 flippedContacts = (cp.flags & PxContactPairFlag::eINTERNAL_CONTACTS_ARE_FLIPPED);
        PxU32 hasImpulses = (cp.flags & PxContactPairFlag::eINTERNAL_HAS_IMPULSES);
        PxU32 nbContacts = 0;

        while(iter.hasNextPatch())
        {
            iter.nextPatch();
            while(iter.hasNextContact())
            {
                iter.nextContact();
                PxVec3 point = iter.getContactPoint();
                PxVec3 impulse = hasImpulses ? dst.normal * impulses[nbContacts] : PxVec3(0.f);

                PxU32 internalFaceIndex0 = flippedContacts ?
                    iter.getFaceIndex1() : iter.getFaceIndex0();
                PxU32 internalFaceIndex1 = flippedContacts ?
                    iter.getFaceIndex0() : iter.getFaceIndex1();
                //...
                nbContacts++;
            }
        }
    }
```

这种方法稍微复杂一些，因为它要求用户不仅要迭代所有数据，还要考虑诸如对(pair)是否被翻转(flipped)或是否报告了脉冲力(impulses)等条件。但是，这种就地循环访问数据(iterating over the data in-place)的方法可能更有效，因为它不需要复制数据。

### Extra Contact Data
-----------------------
由于接触报告(contact reports)中提供了指向接触对(contact pair)`Actor`组件的指针，因此可以直接在回调中读取`Actor`组件(actor properties)属性。但是，`Actor`的姿势和速度通常是指impact的时间。如果由于某些原因，碰撞响应后的速度令人感兴趣，则`Actor`无法提供该信息。同样，如果用户在模拟运行时更改了这些属性，则无法获得`Actor`速度或impact时的姿势(在这种情况下，将返回新设置的属性值)。最后，如果启用了具有多次通过(multiple passes)的CCD，则快速移动的物体可能会多次反弹和对同一物体反弹。每次此类impact的对象姿势和速度都无法从回调中的`Actor`组件指针中提取。对于这些方案，`PhysX SDK` 提供了一个附加的接触流(contact stream)，可以保存与接触对(contact pair)相关的各种额外信息。这些额外的信息是通过标记 `PxPairFlags` 的对每对请求的(有关详细信息，请参阅 `PxPairFlag::ePRE_SOLVER_VELOCITY`、`PxPairFlag::ePOST_SOLVER_VELOCITY`、`PxPairFlag::eCONTACT_EVENT_POSE` 的 API 文档)。如果需要，额外的数据流将作为`PxContactPairHeader`结构的成员提供。然后，可以使用预定义的迭代器 `PxContactPairExtraDataIterator` 或一些自定义解析代码来解析流(有关流格式的详细信息，请参阅 `PxContactPairExtraDataIterator` 的实现)。

```C++
void MySimulationCallback::onContact(const PxContactPairHeader& pairHeader,
    const PxContactPair* pairs, PxU32 nbPairs)
{
    PxContactPairExtraDataIterator iter(pairHeader.extraDataStream,
        pairHeader.extraDataStreamSize);
    while(iter.nextItemSet())
    {
        if (iter.postSolverVelocity)
        {
            PxVec3 linearVelocityActor0 = iter.postSolverVelocity->linearVelocity[0];
            PxVec3 linearVelocityActor1 = iter.postSolverVelocity->linearVelocity[1];
            ...
        }
    }
}
```

## Continuous Collision Detection
-----------------------------------
当连续碰撞检测(或 `CCD` )打开时，受影响的刚体不会高速穿过其他物体(这个问题也称为tunnelling)。要启用 `CCD` ，需要做三件事：

1: `CCD` 需要在场景级别打开：

```C++
PxPhysics* physx;
...
PxSceneDesc desc;
desc.flags |= PxSceneFlag::eENABLE_CCD;
...
```

2:需要在pair filter中启用成对 `CCD` ：

```C++
static PxFilterFlags filterShader(
    PxFilterObjectAttributes attributes0,
    PxFilterData filterData0,
    PxFilterObjectAttributes attributes1,
    PxFilterData filterData1,
    PxPairFlags& pairFlags,
    const void* constantBlock,
    PxU32 constantBlockSize)
{
    pairFlags = PxPairFlag::eSOLVE_CONTACT;
    pairFlags |= PxPairFlag::eDETECT_DISCRETE_CONTACT;
    pairFlags |= PxPairFlag::eDETECT_CCD_CONTACT;
    return PxFilterFlags();
}

...

desc.filterShader    = testCCDFilterShader;
physx->createScene(desc);
```

3:需要为每个需要 `CCD` 的 `PxRigidBody` 启用 `CCD` ：

```C++
PxRigidBody* body;
...
body->setRigidBodyFlag(PxRigidBodyFlag::eENABLE_CCD, true);
```

启用后， `CCD` 仅在相对速度高于其各自 `CCD` 速度阈值之和的`Shape`之间激活。这些速度阈值是根据`Shape`的属性自动计算的，并支持不均匀的比例。

### Contact Notification and Modification
----------------------------------------
CCD 支持离散碰撞检测(discrete collision detection)所支持的全套接触通知事件(contact notification events)。有关接触通知(contact notification)的详细信息，请参阅Callbacks。

CCD支持`contact`修改。若要侦听这些修改回调，请从类 `PxCCDContactModifyCallback` 派生：

```C++
class MyCCDContactModification : public PxCCDContactModifyCallback
{
    ...
    void onCCDContactModify(PxContactModifyPair* const pairs, PxU32 count);
};
```

然后实现 `PxContactModifyCallback` 的 `onContactModify` 函数：

```C++
void MyContactModification::onContactModify(PxContactModifyPair *const pairs, PxU32 count)
{
    for(PxU32 i=0; i<count; i++)
    {
        ...
    }
}
```

此 `onContactModify` 回调使用与离散碰撞检测接触修改回调相同的语义进行操作。有关更多详细信息，请参阅有关回调的文档。

与离散碰撞检测(discrete collision detection)一样，如果用户在过滤器着色器(filter shader)中指定了标记`PxPairFlag::eMODIFY_CONTACTS`的对， `CCD` 才会为给定对发出接触修改事件(contact modification events)。

### Triggers
-----------
目前，用`PxShapeFlag::eTRIGGER_SHAPE`标记的`Shape`将不包括在 `CCD` 中。但是，通过不将触发器`Shape`标记为 `PxShapeFlag::eTRIGGER_SHAPE`，而是filter shaders配置为为涉及触发器`Shape`的对返回以下状态，可以从 `CCD` 获取触发器事件：

*pairFlags = PxPairFlag::eTRIGGER_DEFAULT | PxPairFlag::eDETECT_CCD_CONTACT; return PxFilterFlag::eDEFAULT;*


应该注意的是，不将`Shape`标记为`PxShapeFlag::eTRIGGER_SHAPE`可能会导致触发器更加昂贵。因此，此变通办法应保留为仅在没有 `CCD` 的情况下错过重要触发事件的情况下使用。

### Tuning CCD
--------------
CCD通常应该在没有任何调整的情况下工作。但是，有 4 个属性可以调整：

1:`PxSceneDesc.ccdMaxPasses`：此变量控制我们执行的 `CCD` 传递次数。这默认为 1，这意味着尝试将所有对象更新为其第一次接触的 `TOI` 。在他们第一次接触的 `TOI` 之后的任何剩余时间都将被删除。增加此值允许 `CCD` 运行多个通道。这降低了时间下降的可能性，但会增加 `CCD` 的成本。

2:`PxRigidBody::setMinCCDAdvanceCoeffcient(PxReal advanceCoefficient)`：此方法允许您调整 `CCD` 在给定传递中推进对象的量。默认情况下，此值为 0.15，这意味着 `CCD` 将按 0.15 * ccdThreshold 前进对象，其中 `ccdThreshold` 是按`Shape`计算的值，该值充当在对象可能通过隧道传输之前可以消耗的最大时间量的下限。默认值 0.15 可提高运动的流动性，而不会有错过碰撞的风险。减小此值可能会对流动性产生负面影响，但会降低对象在帧结束时被剪切的可能性。增加此值可能会增加对象隧道的可能性。此值应仅在 [0，1] 范围内设置。

3:在 `PxSceneDesc.flags` 上启用标志 `PxSceneFlag::eDISABLE_CCD_RESWEEP`：启用此标志将禁用 `CCD` 重新扫视(resweeps)。这可能导致由于弹跳而错过碰撞，但有可能减少 `CCD` 的开销。通常，启用此前进模式仍可保证对象不会通过静态环境，但不再保证启用了 `CCD` 的动态对象不会相互传递。

4:`PxRigidBody::setRigidBodyFlag(PxRigidBodyFlag::eENABLE_CCD_FRICTION，true)`：启用此标志可以在 `CCD` 中应用摩擦力。默认情况下，此功能处于禁用状态。由于 `CCD` 仅使用线性运动运行，因此 `CCD` 内部的摩擦可能会导致视觉伪影。

### Performance Implications
----------------------------
对场景/场景中的所有实体启用CCD应该相对有效，但即使场景中的所有对象移动相对较慢，也会对性能产生一些影响。在优化CCD方面投入了大量精力，因此，当物体缓慢移动时，这种额外的开销只占整个仿真时间的很小一部分。随着物体速度的增加，CCD头顶将增加，特别是如果附近有很多高速物体。增加CCD通行证的数量可能会使CCD更加昂贵，尽管如果不需要额外的通行证，CCD将提前终止。

### Limitations
---------------
`CCD` 系统是一种尽最大努力的保守推进方案。它运行有限数量的 `CCD` 子步骤(默认为1)，并丢弃任何剩余时间。通常，时间只会在 `impact` 的那一刻落在高速物体上，因此不会引起注意。但是，如果您模拟的对象相对于模拟时间步长足够小/薄，并且如果物体从静止状态被重力加速1帧，则该物体可以隧道，则此伪影可能会变得明显，即薄如纸张的刚体。这样的物体将始终以高于其 `CCD` 速度阈值的速度移动，并可能导致该物体和与其位于同一岛屿上的任何物体(其边界与该物体的边界重叠的任何物体)的大量模拟时间被丢弃。这可能会导致明显的减速/卡顿效果，这是由于该岛中的对象与模拟的其余部分明显不同步。因此，建议尽可能避免使用薄纸/微小的物体。

还建议您过滤掉受约束在一起的身体之间的 `CCD` 相互作用，例如同一布娃娃中的四肢。允许同一布娃娃的四肢之间的 `CCD` 相互作用可能会增加 `CCD` 的成本，并可能导致不必要地缩短时间。 `CCD` 交互在铰接(articulation)中的链接(links)之间自动禁用。

## Raycast CCD
--------------
`PhysX SDK` 支持基于简单光线投射(simple raycasts)的替代 `CCD` 实现。这种"raycast CCD"算法在 `PhysX Extensions`中可用，并在代码片段("SnippetRaycastCCD")中进行了演示。与 `PhysX SDK` 中实现的内置 `CCD` 算法相反，这种更便宜、更简单的替代版本完全在 `SDK` 本身之外实现。

在传统的模拟/fetchResults调用之后，系统从`Shape`的中心位置执行raycasts，以仔细检查它们是否没有隧道传输(tunnel)。如果检测到对象隧道传输，则会将其移回光线沿线的先前位置(重叠位置)。然后下一帧， `SDK` 的接触生成接管并生成令人信服的动作。这里有一些微妙的细节没有描述，但简而言之，这就是它的工作原理。

由于它是基于光线投射的，因此解决方案并不完美。特别是，如果光线穿过边缘之间的裂缝或世界中的小孔(如门上的钥匙孔)，则小型动态对象仍然可以穿过静态世界。此外，动态与动态 `CCD` 是非常近似的。它仅适用于快速移动的动态对象与缓慢移动的动态对象的碰撞。其他已知的限制是，它目前仅适用于 `PxRigidDynamic` 对象(不适用于 `PxArticulationLink` )和具有一种`Shape`的简单 `actor` (不适用于"compounds")。

但是，实现应该能够防止重要对象离开游戏世界，前提是世界是无懈可击的。代码非常小，易于遵循或修改，其整体性能通常比内置 `CCD` 更好。因此，如果默认 `CCD` 变得过于昂贵，它可能是一个有价值的替代方案。

## Speculative CCD
-------------------
除了基于扫描的 `CCD` 之外， `PhysX` 还提供了一种更便宜但不太强大的方法，称为speculative CCD。这种方法与基于扫描的 `CCD` 不同，因为它完全作为离散仿真的一部分运行，方法是根据物体运动膨胀接触偏移(inflating contact offsets)，并依靠约束求解器来确保物体不会相互隧道(tunnel)。

这种方法通常效果很好，与基于扫描的`CCD`(sweep-based CCD)不同，对`kinematic actors`进行`Speculative CCD`是合法的。但是，在某些情况下，它可能无法确保对象不会相互传递。例如，如果约束求解器加速了`Actor`(由于碰撞或关节)，使得`Actor`在该时间步长内完全穿过物体，则推测`CCD`可能导致隧穿。

要启用此功能，请在需要 `CCD` 的刚体上提高 `PxRigidBodyFlag::eENABLE_SPECULATIVE_CCD`：

```C++
    PxRigidBody* body;
...
body->setRigidBodyFlag(PxRigidBodyFlag::eENABLE_SPECULATIVE_CCD, true);
```

与 `sweep-based CCD` 不同，这种形式的 `CCD` 不需要在场景或滤镜着色器(filter shader)中的对(pair)上引发设置(require settings)。

请注意，此方法最适合 `PCM` 冲突检测。如果使用基于 `SAT` 的传统碰撞检测方法，它可能无法正常工作。

此功能可以与`sweep-based CCD`结合使用，例如，如果快速移动的 `kinematic` 启用了`speculative CCD`，但dynamic rigid bodies使用`sweep-based CCD`。但是，如果将`speculative CCD`与`sweep-based CCD`一起用于`kinematic`，则必须确保使用`speculative contacts`的`kinematic actor`与启用 `CCD` 的`dynamic actors`之间的相互作用也不支持`sweep-based CCD`相互作用，否则`sweep-based CCD`可能会推翻`Speculative CCD`，从而导致不良行为。

## Persistent Contact Manifold (PCM)
------------------------------------
`PhysX SDK` 提供两种类型的冲突检测：

1:默认冲突检测

默认碰撞检测系统混合使用 SAT(分离轴定理) 和基于距离的碰撞检测来生成完全接触流形(full contact manifolds)。它在一个框架中生成所有潜在的`contact`，因此它更适合稳定的堆叠(stacking)。这种方法对于较小的接触偏移和静止偏移是稳定的，但在使用大偏移时可能无法生成正确的接`contact`，因为它通过平面偏移接近这些情况下的`contact`。

2:Persistent Contact Manifold (PCM) 

PCM是一个完全基于距离的碰撞检测系统。当两个`Shape`首次接触时，PCM会生成一个完整的`manifold of contacts`。它回收并更新流形中前一帧中的现有`contact`，然后，如果`Shape`相对于彼此的移动超过阈值量，或者如果`contact`从流形中掉落，则会在后续帧中生成新`contact`。如果由于帧中的大量相对运动而从流形上掉落了太多的`contact`，则将重新运行完整的流形生成。这种方法在性能和内存方面非常有效。但是，由于 PCM 生成的`contact`可能比默认冲突检测少，因此在模拟求解器迭代不足的高堆时，可能会降低堆叠稳定性。由于这种方法是基于距离的，因此它将为任意接触偏移/静止偏移生成正确的接`contact`。

要启用 PCM，请在 `PxSceneDesc::flags` 中设置标志：

```C++
PxSceneDesc sceneDesc;

sceneDesc.flags |= PxSceneFlag::eENABLE_PCM;
```