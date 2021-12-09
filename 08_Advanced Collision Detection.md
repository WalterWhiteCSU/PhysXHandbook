# Advanced Collision Detection
---------------------------------
## Tuning Shape Collision Behavior
-----------------------------------
用于产生接触的`Shape`会影响它们通过contacts(Contact point)附着(attached)的dynamic rigid bodies。约束求解器在contacts处产生脉冲力，以保持`Shape`静止或移动而不相互传递。`Shape`有两个重要参数，用于控制碰撞检测如何生成它们之间的contacts，而这些contacts又是碰撞或堆叠(stacking)时它们行为的核心： `contactOffset` 和 `restOffset` 。它们分别使用`PxShape::setContactOffset（）`和`PxShape::setRestOffset（）`进行设置。这两个值都不直接使用。碰撞检测始终对一对可能碰撞的`Shape`进行操作，并且始终考虑两个`Shape`的偏移总和。我们分别称这些为"contactDistance"和"restDistance"。

碰撞检测能够在两个`Shape`之间产生contacts，当它们仍然相距很远时，当它们完全接触时，或者当它们相互穿透时。为了使讨论更简单，我们将相互渗透视为负距离。因此，两个`Shape`之间的距离可以是正数、零形或负数。 `SeparationDistance` 是碰撞检测开始生成接触的距离。它必须大于零，这意味着当两个`Shape`穿透时， `PhysX` 将始终生成触点（除非以某种方式完全禁用了两个`Shape`之间的碰撞检测，例如过滤）。默认情况下，在 `PxTolerancesScale` 中使用公制单位和默认缩放时， `contactOffset` 为 0.02，这意味着 `contactDistance` 的计算值将达到 4 厘米。因此，当两个`Shape`在4厘米内彼此接近时，将产生触点，直到它们再次移动到4厘米以外的距离。

然而，contacts的产生并不意味着会立即在这些位置施加大脉冲来分离`Shape`，甚至阻止在穿透方向上进一步运动。除非将仿真时间步长选择为很小，否则这将使仿真抖动变小，这对于实时性能是不可取的。相反，我们希望接触处的力随着穿透力的增加而平滑增加，直到它达到足够高的值以阻止任何进一步的穿透运动。达到最大力的距离是静止距离，因为在这个距离上，相互堆叠的两个`Shape`将达到静态平衡并静止。当`Shape`由于某种原因被推在一起，以至于它们的距离低于静止时，会施加更大的力将它们推开，直到它们再次处于静止距离。随着距离的变化而施加的力的变化不一定是线性的，但它是平滑和连续的，即使在大的时间步长下也能产生令人愉悦的模拟。

在为`Shape`选择 `contactOffset` 和 `restOffset` 时，需要考虑一些不同的事项。通常，相同的值可用于模拟中的所有`Shape`。首先确定 `restOffset` 是有意义的。目标通常是使图形`Shape`看起来堆叠在一起，使它们完全接触，就像现实生活中的身体一样。如果碰撞`Shape`的大小与图形`Shape`的大小完全相同，则需要 `restOffset` 为零。如果碰撞`Shape`的 epsilon 大于图形`Shape`，则负 epsilon 的 `restOffset` 是正确的。这将使较大的碰撞`Shape`相互下沉，直到较小的图形`Shape`也接触。rest大于零的偏移是实际的，例如，如果在三角形几何体上滑动时存在问题，其中基于穿透的接触生成比分离contacts更难以产生平滑的contacts，从而导致更平滑的滑动。

确定 `restOffset` 后，应选择触点关闭集的值稍大一些。经验法则是使两者之间的差异尽可能小，从而在仿真使用的时间步长大时仍能有效避免抖动。较大的时间步长需要更大的差异。将其设置得太大的缺点是，当两个`Shape`接近时，将更快地生成触点，这会增加仿真必须担心的触点总数。这将降低性能。此外，仿真代码通常假设contacts靠近凸形的表面。如果接触偏移非常大，则此假设会崩溃，从而导致行为伪影。

## Contact Modification
--------------------------
在某些情况下，可能需要specialize contact behavior。例如，为了实现sticky contacts，使物体看起来彼此漂浮或swimming，或者使物体穿过墙壁上的明显孔。实现此类效果的一种简单方法是让用户在通过碰撞检测生成contacts后，但在接触求解器之前更改触点的属性。由于这两个步骤都发生在 `scene simulate（）` 函数中，因此必须使用回调。

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

每对'Shape'都附带一组contacts，这些contacts具有许多可以修改的属性，例如位置、接触法线和separation。目前，触点的恢复和摩擦性能不能改变。请参阅 `PxModifiableContact` 和 `PxContactSet` ，了解可以修改的属性。

除了修改联系人属性外，还可以：

+ 为每个接触设置目标速度
+ 限制每个触点施加的最大脉冲
+ 分别调整每个主体的逆质量和反惯性刻度

通过设置目标速度可以实现类似传送带的效果。通过使目标速度与接触法线的切向方向运行，可以获得最佳结果，但求解器也支持接触法线方向上的目标速度。

用户可以通过限制每个触点上施加的最大脉冲来限制施加在每个触点上的脉冲。这可用于产生"软"接触效应，例如，给人以由于压缩而导致的能量耗散的印象，或限制由于运动学碰撞而施加在动态物体上的脉冲。请注意，限制最大脉冲可能会导致额外的穿透和物体相互传递。

调整质量和惯性尺度可用于调整一对物体之间的接触如何影响物体的线性和角速度。接触对中的每个物体都有一个单独的反质量和反惯性尺度。这些刻度初始化为 1，可以作为回调的一部分进行调整。请注意，这些值在触点对内执行局部质量修改，并影响接触对内的所有触点。

均匀地缩放物体的逆质量和反惯性相同的值会导致身体的行为像一个物体，根据所使用的值，它要么更重，要么更轻。提供<1的逆质量/反向惯性量表会导致身体看起来更重;提供>1的尺度会导致身体看起来更轻。例如，0.5的逆质量/惯性尺度导致身体看起来具有两倍的质量。提供4的逆质量/惯性尺度将导致身体看起来具有其原始质量的四分之一。提供0的逆质量/惯性量表会导致身体表现得好像它有无限的质量。

但是，也可以通过为物体的逆质量和反惯性尺度提供不同的值来非均匀地缩放物体的逆质量和反惯性。例如，可以通过仅调整惯性反比尺度来减少或增加接触引起的角速度变化量。这种修改的用例非常依赖于游戏，但可能涉及，例如，在街机风格的驾驶游戏中调整玩家的车辆和交通车辆之间的交互，其中玩家的汽车预计会被交通车辆撞倒，但如果汽车因碰撞而旋转出来，玩家会感到非常沮丧。这也可以通过使交通车辆比玩家的车辆轻得多来实现，但这可能会使交通车辆看起来"太轻"，从而损害玩家的沉浸感。

在执行局部质量修改时，PxSimulationEventCallback::onContact（） 中报告的脉冲将相对于参与该接触的物体的局部缩放质量。因此，报告的这种冲动可能不再准确地反映由给定接触引起的动量变化。为了解决这个问题，我们在刚体延伸中提供了以下方法，以提取由接触引起的线性和角脉冲和速度变化，使用局部质量修改：

```C++
static void computeLinearAngularImpulse(const PxRigidBody& body, const PxTransform& globalPose,
    const PxVec3& point, const PxVec3& impulse, const PxReal invMassScale,
    const PxReal invInertiaScale, PxVec3& linearImpulse, PxVec3& angularImpulse);

static void computeVelocityDeltaFromImpulse(const PxRigidBody& body,
    const PxTransform& globalPose, const PxVec3& point, const PxVec3& impulse,
    const PxReal invMassScale, const PxReal invInertiaScale, PxVec3& deltaLinearVelocity,
    PxVec3& deltaAngularVelocity);
```

这些方法返回单独的线性和角脉冲和速度变化值，以反映质量和惯性可能已不均匀缩放的事实。当使用局部质量修饰时，可能需要为对中的每个物体的每个contacts提取单独的线性和角脉冲。请注意，提供这些辅助功能是为了向用户提供准确的脉冲值，绝不是强制性的。对于简单的用例，例如基于脉冲阈值的触发效应或损坏，即使使用了局部质量修改，接触报告报告的单一脉冲值也应该是完全可以接受的。但是，如果已经使用了局部质量修改，并且脉冲值用于更复杂的行为，例如布娃娃的平衡控制，那么很可能需要这些辅助功能来实现正确的行为。请注意，在铰接的情况下，computeLinearAngularImpulse 将返回应用于相应铰接链路的正确脉冲。但是，computeVelocityDeltaFromImpulse 不会返回铰接链接的正确速度变化，因为它不考虑铰接的任何其他链接的影响。

此外，使用局部批量修改时，必须考虑以下事项：

+ 回调的强制阈值将基于联系人报告中的标量脉冲值。这是使用物体的缩放质量/惯性计算的，因此使用质量缩放可能需要重新调整这些阈值。
+ 求解器中在按比例质量/惯量操作的脉冲值上发生最大脉冲钳位。因此，在使用质量缩放的情况下，从 computeLinearAngularImpulse（...） 计算的施加脉冲的大小可能会超过最大脉冲。在使用均匀质量缩放的情况下，线性脉冲的大小不会超过质量尺度*最大脉冲，角脉冲的大小不会超过惯性尺度*最大脉冲。

由于回调来自 SDK 内部深处，因此对回调有一些特殊要求。特别是，回调应该是线程安全和可重入的。换句话说，SDK可以从任何线程调用onContactModify（），并且可以同时调用（即，要求同时处理联系人修改对的集合）。

可以使用 PxSceneDesc 的 contactModifyCallback 成员或 PxScene 的 setContactModifyCallback（） 方法来设置联系人修改回调。