# Joints

## Joint Basics
-----------------
关节(Joints)限制了两个`Actor`相对于彼此移动的方式。关节的典型用途是模拟门铰链或`character`的肩膀。关节在 `PhysX extensions library`中实现，涵盖了许多常见场景，但如果您有 `PhysX` 打包的关节无法满足的用例，则可以实现自己的用例。由于关节是作为扩展实现的，因此创建它们的模式与其他 `PhysX` 对象略有不同。

简单关节和限制(simple joints and limits)的创建在"SnippetJoint"代码段中进行了演示。

要创建关节，请调用关节的创建函数:

```C++
PxRevoluteJointCreate(PxPhysics& physics,
                      PxRigidActor* actor0, const PxTransform& localFrame0,
                      PxRigidActor* actor1, const PxTransform& localFrame1);
```

对于所有关节，这具有相同的模式:两个`Actor`，每个`Actor`都有一个约束帧(constraint frame)。

其中一个`Actor`必须是可移动的，可以是 `PxRigidDynamic` 或 `PxArticulationLink` 。另一个`Actor`可能是这些类型之一，或者是 `PxRigidStatic` 。在此处使用 NULL 指针来指示表示不可移动全局参考系(immovable global reference frame)的隐式`Actor`。

每个 `localFrame` 参数指定一个相对于`Actor`的全局位置(global pose)的约束帧(constraint frame)。每个关节定义全局位置与约束帧原点之间的关系，该关系将由 `PhysX` 约束求解器强制执行。在此示例中，旋转`joint`(revolute joint)将两个`frame`的原点限制为重合，并且它们的 x 轴重合，但允许两个`Actor`围绕此公共轴彼此自由旋转。

`PhysX` 支持六种不同的关节类型:

+ `fixed joint`将方向和原点刚性地锁定在一起
+ `distance joint`将原点保持在一定的距离范围内
+ `spherical joint(ball-and-socket)`将原点保持在一起，但允许方向自由变化。
+ `revolute joint(hinge)`将`frame`的原点和 x 轴保持在一起，并允许围绕此公共轴自由旋转。
+ `prismatic joint(slider)`保持方向相同，但允许每个`frame`的原点沿公共 x 轴自由滑动。
+ `D6 joint`是一种高度可配置的`joint`(highly configurable joint)，允许指定单独的自由度，可以自由移动或锁定在一起。它可用于实现各种机械和解剖关节，但配置起来不如其他关节类型直观。下面详细介绍了该关节。

所有关节都通过 `PxConstraint` 类作为 `SDK` 的插件实现。每个关节的许多属性都是使用 `PxConstraintFlag` 枚举配置的。

**与 `PhysX API` 的其余部分一样，极限和驱动目标(limits and drive targets)的所有关节角度都以弧度为单位指定。**

### Visualization
------------------
所有标准 `PhysX joint`都支持调试可视化。您可以可视化每个 `Actor` 的joint frames，以及关节可能具有的任何限制。

默认情况下，关节不可可视化。要可视化关节，请设置其可视化约束标志和相应的场景级可视化参数:

```C++
scene->setVisualizationParameter(PxVisualizationParameter::eJOINT_FRAMES, 1.0f);
scene->setVisualizationParameter(PxVisualizationParameter::eJOINT_LIMITS, 1.0f);

...

joint->setConstraintFlag(PxConstraintFlag::eVISUALIZATION)
```

### Force Reporting
-------------------
施加在关节上的力可以在模拟后通过调用`getForce()`来检索:

```C++
scene->fetchResults(...)
joint->getConstraint().getForce(force, torque);
```

力在`actor1`的`joint frame`的原点处解决。

请注意，只有在关节的`Actor`处于清醒(awake)状态时，此力才会更新。

### Breakage
--------------
所有标准的 `PhysX joints` 都可以制成易破坏的(breakable)。可以指定最大断裂力和扭矩，如果维持`joint`约束所需的力或扭矩超过此阈值，则`joint`将断裂。破坏关节会生成一个模拟事件(参见 `PxSimulationEventCallback::onJointBreak`)，并且该关节不再参与模拟，尽管它在被删除之前一直附着在它的`actor`上。

默认情况下，阈值力和扭矩设置为 `FLT_MAX` ，使`joint`有效牢不可破。要使`joint`易破坏，请指定力和扭矩阈值。

*joint->setBreakForce(100.0f, 100.0f);*

约束标志记录`joint`当前是否断开:

*bool broken = (joint->getConstraintFlags() & PxConstraintFlag::eBROKEN) != 0;*

断开`joint`会导致通过 `PxSimulationEventCallback::onConstraintBreak` 进行回调。在此回调中，指向关节及其类型的指针在 `PxConstraintInfo` 结构的 `externalReference` 和类型字段中指定。如果已实现自己的关节类型，请使用 `PxConstraintInfo::type` 字段来确定已断开约束的动态类型。否则，只需将 `externalReference` 转换为 `PxJoint`:

```C++
class MySimulationEventCallback
{
    void onConstraintBreak(PxConstraintInfo* constraints, PxU32 count)
    {
        for(PxU32 i=0; i<count; i++)
        {
            PxJoint* joint = reinterpret_cast<PxJoint*>(constraints[i].externalReference);
            ...
        }
    }
}
```

### Projection
-----------------
在压力条件(stressful conditions)下， `PhysX` 的动力学求解器(dynamics solver)可能无法准确执行`joint`指定的约束。 `PhysX` 提供运动学投影(kinematic projection)，即使求解器出现故障，也会尝试将违反的约束重新对齐。投影(projection)不是一个物理过程，不会保持动量或尊重碰撞几何(collision geometry)。如果可行，最好避免这样做，但在关节分离导致不可接受的伪影的情况下，这对于提高仿真质量非常有用。

默认情况下，投影处于禁用状态。要启用投影，请设置投影`joint`的线性公差和角度公差值(linear and angular tolerance values)，并设置约束投影标志(constraint projection flag):

```C++
joint->setProjectionLinearTolerance(0.1f);
joint->setConstraintFlag(PxConstraintFlag::ePROJECTION, true);
```

非常小的投影公差值(tolerance values)可能会导致`joint`周围抖动。

启用了投影的约束可以是由约束连接的刚体图(graph of rigid bodies)的一部分。如果此图是非循环的，则算法将在连接的刚体中选择一个根节点，遍历`graph`，然后将`body`投影到根。如果约束图(constraint graph)具有循环，则该算法会将`graph`拆分为多个非循环子图，删除创建循环的边缘(dropping edges that create cycles)，并为每个子图单独进行投影。请注意，在`graph`中将多个约束附加到固定锚点(fixed anchor)((world或static/kinematic rigid body)确实算作一个循环(例如，a chain of rigid bodies connected with constraints and both ends attached to world anchors)。如果指定了多个约束在同一`body`上发生冲突或冲突的投影方向，则将根据以下优先级选择投影方向(最高优先):

+ 世界依恋(world attachment)或具有投影约束(projecting constraint )的`rigid static actor`
+ 具有投影约束的`kinematic actor`
+ 所有占主导地位的`dynamic actor`(具有投影约束，并且所有这些都是向此动态的单向投影)
+ 占主导地位的`dynamic actor`(与上面相同，但至少还有一个双向投影约束)
+ 部分主导的`dynamic actor`(至少具有一个针对此动态的单向投影约束，以及至少一个针对其他`Actor`的单向投影约束)
+ 世界依恋或`rigid static actor`，没有任何投影约束
+ 没有任何投影约束的`kinematic actor`
+ 具有或不具有双向投影约束到其他`dynamic`的`dynamic actor`(其中，具有最高约束计数的`Actor`获胜)

### Limits
---------
一些 `PhysX joints` 不仅限制相对旋转或平移，还可以限制运动范围(range of that motion)。例如，在其初始配置中，`revolute joint`允许围绕其轴自由旋转，但通过指定和启用限制(Limits)，可以在旋转角度上放置下限和上限(lower and upper bounds)。

`Limits`是碰撞(collision)的一种形式，与`rigid body shapes`的碰撞一样，稳定的limit behavior需要 `contactDistance` 容差(tolerance)，指定在求解器尝试强制使用`joint configuration`之前，`joint configuration`可能离`Limits`有多远。请注意，强制功能(enforcement)始终在超出`limit`之前开始，因此， `contactDistance` 对`Limits`所起的作用类似于 `contactDistance` 的正值在碰撞检测(collision detection)中所起的作用。较大的 `contact` 使得即使在高相对速度下也不太可能违反`Limits`。但是，由于`Limits`在更多时间处于活动状态，因此模拟关节的成本更高。

`Limit configuration`特定于每种类型的关节。要设置限制，请配置限制几何图形并设置特定于`joint`的标志，以指示限制已启用:

```C++
revolute->setLimit(PxJointAngularLimitPair(-PxPi/4, PxPi/4, 0.1f)); // upper, lower, tolerance
revolute->setRevoluteJointFlag(PxRevoluteJointFlag::eLIMIT_ENABLED, true);
```

限制可以是强制(hard)的，也可以是软(soft)的限制。当超过强制限制时，如果限制配置为零恢复(zero restitution)，则相对运动将停止，如果恢复原状为非零(restitution is non-zero)，则相对运动将反弹(bounce)。当超过软`limit`时，求解器将使用由Limits的弹簧和阻尼参数(spring and damping parameters)指定的弹簧将`joint`拉回`limit`。默认情况下，限制是强制的，并且没有restitution，所以当关节达到`limit`时，运动将简单地停止。要指定`limit`的柔和度(softness)，请声明`limit`结构并直接设置弹簧和阻尼参数:

```C++
PxJointAngularLimitPair limitPair(-PxPi/4, PxPi/4, 0.1f));
limitPair.spring = 100.0f;
limitPair.damping = 20.0f;
revolute->setRevoluteJointLimit(limitPair);
revolute->setRevoluteJointFlag(PxRevoluteJointFlag::eLIMIT_ENABLED, true);
```

*Limits are not projected.*

使用弹簧限制时，强烈建议使用`eACCELERATION`。该标志将根据`limit`作用的物体的质量和惯性自动缩放弹簧的强度，并且可以大大减少良好，稳定行为所需的调谐量。

### Actuation
---------------
某些 `PhysX joint`可能由motor或由 `PhysX` 求解器隐式集成的弹簧驱动。虽然使用`actuated joints`进行仿真比简单地施加力更昂贵，但它可以提供更稳定的仿真控制。有关详细信息，请参阅 D6 Joint、Prismatic Joint和Revolute Joint。

**actuation产生的力不包括在求解器报告的力中，也不会有助于超过`joint`的断裂力阈值(breakage force threshold)。**

**更改`joint`的驱动参数，或者激活或停用驱动器，不会唤醒连接到`joint`的睡眠体。如果需要，请手动唤醒这些主体。**

使用弹簧驱动器(spring drives)(特别是 `D6 joint`上的驱动器)时，强烈建议使用`eACCELERATION`。该标志将根据`limit`作用的物体的质量和惯性自动缩放弹簧的强度，并且可以大大减少良好，稳定行为所需的调谐量。

### Mass Scaling
------------------
`PhysX joints`可以将尺度(scale)应用于两个连接体(connected bodies)的质量和惯性矩，以解析关节(resolving a joint)。例如，如果您在布娃娃中有两个质量为 1 和 10 的物体， `PhysX` 通常会通过改变较轻`body`的速度来解析关节，而不是较重的`body`。您可以对第一个物体应用mass scale 10，以使 `PhysX` 将两个物体的速度更改相等的量。为了确保线性和角速度具有相同的属性，还应根据物体的惯性调整惯性尺度(inertia scales)。应用质量尺度(mass scales)，使关节看到相似的有效质量和惯性，使求解器收敛得更快，这可以使单个关节看起来不那么橡胶或分离，并且关节体组看起来不那么抽搐。

许多优先考虑视觉行为而不是遵守物理定律的应用程序都可以从调整这些尺度值中受益。但是，如果您使用此功能，请记住，质量和惯性缩放(mass and inertia scaling)从根本上说是非物理的。一般来说，动量不会守恒，系统的能量可能会增加，为`joint`报告的力可能不正确，并且可能需要对破损阈值(breakage thresholds)和力极限(force limits)进行非物理调整。

## Fixed Joint
----------------
<img src=".\image\Joi_1.png" alt="Joi_1" style="zoom:100%;" />

`Fixed Joint`约束两个对象，以便其约束帧的位置和方向相同。

**所有关节都由动力学求解器(dynamics solver)强制执行，因此，尽管在理想条件下，对象将保持其空间关系，但可能会存在一些漂移。一种常见的替代方法是构建具有多个shapes的单个`Actor`，模拟成本更低且不会受到漂移的影响。然而，`Fixed Joint`是有用的，例如，当`joint`必须可断裂(breakable)或报告其约束力时。**

## Spherical Joint
--------------------
<img src=".\image\Joi_2.png" alt="Joi_2" style="zoom:100%;" />

`spherical joint`将 `Actor` 的约束帧的原点约束为重合。

`spherical joint`支持锥形`limit`，该`limit`约束两个约束帧的 X 轴之间的角度。Actor1 的 X 轴受`limit`锥的约束，该`limit`锥的轴是 actor0 约束帧的 x 轴。允许的`limit`值是围绕该帧的 y 轴和 z 轴的最大旋转。可以为 y 轴和 z 轴指定不同的值，在这种情况下，`limit`采用椭圆形角锥的形式:

```C++
joint->setLimitCone(PxJointLimitCone(PxPi/2, PxPi/6, 0.01f);
joint->setSphericalJointFlag(PxSphericalJointFlag::eLIMIT_ENABLED, true);
```

请注意，非常小或高度椭圆的`limit`锥可能会导致求解器抖动。

**`limit`表面的可视化可以大大有助于理解其形状。**

## Revolute Joint
------------------
<img src=".\image\Joi_3.png" alt="Joi_3" style="zoom:100%;" />

`revolute joint`可从两个对象上移除除单个旋转自由度之外的所有自由度。两个实体可沿其旋转的轴由joint frames的共同原点及其公共 x 轴指定。从理论上讲，沿旋转轴的所有原点都是等效的，但当点靠近物体最近的位置时，模拟稳定性在实践中是最好的。

关节支持具有上限和下限范围的旋转极限。当joint frames的 y 轴和 z 轴重合时，角度为零，并且从 y 轴向 z 轴移动时增加:

```C++
joint->setLimit(PxJointLimitPair(-PxPi/4, PxPi/4, 0.01f);
joint->setRevoluteJointFlag(PxRevoluteJointFlag::eLIMIT_ENABLED, true);
```

该`joint`还支持一个`motor`，该`motor`驱动两个`Actor`的相对角速度朝向用户指定的目标速度。`motor`施加的力的大小可以限制在指定的最大值:

```C++
joint->setDriveVelocity(10.0f);
joint->setRevoluteJointFlag(PxRevoluteJointFlag::eDRIVE_ENABLED, true);
```

默认情况下，当关节处的角速度超过目标速度时，`motor`充当制动器;自由旋转标志(freespin flag)禁用此制动行为。

`revolute joint`的`force limit`可以解释为力或脉冲，具体取决于`PxConstraintFlag::eDRIVE_LIMITS_ARE_FORCES`

## Prismatic Joint
-------------------
<img src=".\image\Joi_4.png" alt="Joi_4" style="zoom:100%;" />

`prismatic joint`可防止所有旋转运动，但允许 actor1 的约束帧的原点沿 actor0 约束帧的 x 轴自由移动。`prismatic joint`支持两个约束帧原点之间距离的上限和下限的单个极限:

```C++
joint->setLimit(PxJointLimitPair(-10.0f, 20.0f, 0.01f);
joint->setPrismaticJointFlag(PxPrismaticJointFlag::eLIMIT_ENABLED, true);
```

## Distance Joint
--------------------
<img src=".\image\Joi_5.png" alt="Joi_5" style="zoom:100%;" />

`distance joint`将约束帧的原点保持在一定的距离范围内。该范围可以同时具有上限和下限，这些上限和下限分别由标志启用:

```C++
joint->setMaxDistance(10.0f);
joint->setDistanceJointFlag(eMAX_DISTANCE_ENABLED, true);
```

此外，当关节达到其范围的极限时，超出该距离的运动可能被求解器完全阻止，或者用隐式弹簧推回其范围，为此可以指定弹簧和阻尼参数。

## D6 Joint
-----------
`D6 joint`是迄今为止最复杂的标准`PhysX joint`。在其默认状态下，它的行为类似于固定关节 - 也就是说，它严格地固定其两个`Actor`的约束帧。但是，可以解锁单个自由度，以允许围绕 x、y 和 z 轴旋转以及沿这些轴平移的任意组合。

### Locking and Unlocking Axes
-------------------------------
要解锁和锁定自由度，请使用关节的 `setMotion` 函数:

```C++
d6joint->setMotion(PxD6Axis::eX, PxD6Motion::eFREE);
```

解锁平移自由度允许 actor1 的约束帧的原点沿 actor0 的约束帧定义的轴子集移动。例如，仅解锁 X 轴即可创建相当于`prismatic joint`的部件。

旋转自由度分为`twist`(围绕 actor0 约束帧的 X 轴)和`swing`(围绕 Y 轴和 Z 轴)。通过解锁各种`twist`和`swing`组合来实现不同的效果。

+ 如果只解锁了一个角度自由度，则结果始终等效于`revolute joint`。建议如果只解锁一个角度自由度，则应该是`twist degree`，因为`joint`具有针对这种情况设计的各种配置选项和优化。
+ 如果两个 `swing` 自由度都已解锁，但`twist degree`仍锁定，则结果为`zero-twist joint`。actor1 的 x 轴可以自由swing，远离 actor0 的 x 轴，但会twist以最大程度地减少对齐两个帧所需的旋转。这创造了一种各向同性万向节(isotropic universal joint)，避免了通常的"engineering style"万向节(见下文)的问题，该万向节有时被用作一种twist约束。在π radians (180 degrees) swing处有一个令人讨厌的奇点，因此应使用swing limit来避免奇点。
+ 如果一个swing和一个twist自由度解锁，但剩余的swing保持锁定，则会产生`zero-swing joint`(通常也称为万向节universal joint)。例如，如果 SWING1(y 轴旋转)未锁定，则 actor1 的 x 轴将约束为与 actor0 的 z 轴保持正交。在角色应用中，该关节可用于模拟包含下臂自由`twist`的肘部swing关节(elbow swing joint)或包含小腿`twist`自由的膝关节swing关节(knee swing joint)。在车辆应用中，这些关节可以用作"方向盘"关节，其中子`Actor`是车轮，可以围绕其`twist`轴自由旋转，而父`Actor`中的自由swing轴充当转向轴。由于各向异性行为(anisotropic behavior)和角度为π/2的奇异性(singularities)(当心可怕的万向节锁)，因此必须注意这种组合，使`zero-twist joint`成为大多数用例中表现更好的替代方案。
+ 如果所有三个角度度都解锁，则结果等效于`spherical joint`。

从 `PhysX2` 中移除的三个关节已从 PhysX 3 中移除，可以按如下方式实现:

+ `cylindrical joint` (轴沿两个约束帧的公共 x 轴)由以下组合给出:

```C++
d6joint->setMotion(PxD6Axis::eX,     PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eTWIST, PxD6Motion::eFREE);
```

+ `point-on-plane joint`(平面轴沿 actor0 的约束帧的 x 轴)由以下组合给出:

```C++
d6joint->setMotion(PxD6Axis::eY,      PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eZ,      PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eTWIST,  PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eSWING1, PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eSWING2, PxD6Motion::eFREE);
```

+` point-on-line joint`(轴沿 actor0 的约束帧的 x 轴)由以下组合给出:

```C++
d6joint->setMotion(PxD6Axis::eX,      PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eTWIST,  PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eSWING1, PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eSWING2, PxD6Motion::eFREE);
```

**`Angular projection`仅适用于锁定两个或三个角度自由度的情况。**

### Limits
----------
与其指定轴是空闲的或锁定的，不如将其指定为受限轴。D6 支持三种不同的限制(Limits)，可以任意组合使用。

仅具有上限的单个线性`Limit`用于约束任何平移自由度。当约束帧投影到这些轴上时，限制约束帧的原点之间的距离。例如，组合:

```C++
d6joint->setMotion(PxD6Axis::eX, PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eY, PxD6Motion::eLIMITED);
d6joint->setMotion(PxD6Axis::eZ, PxD6Motion::eLIMITED);
d6joint->setLinearLimit(PxJointLinearLimit(1.0f, 0.1f));
```

将 actor1 的约束帧的 y 和 z 坐标约束为位于单位圆盘内。由于 x 轴是无约束的，因此效果是将 actor1 的约束帧的原点约束为位于半径为 1 的圆柱内，该圆柱体沿 actor0 的约束帧的 x 轴延伸。

twist自由度受具有上限和下限的pair limit的`Limit`，与`revolute joint`的限制相同。

如果两个swing自由度都受到限制，则生成一个限制锥体(limit cone)，该锥体与`spherical joint`的限制相同。与`spherical joint`一样，非常小或高度椭圆的限制锥可能会导致求解器抖动。

如果只限制了一个swing自由度，则使用与锥形限制的相应角度来限制旋转。如果另一个swing度被锁定，则限值的最大值为π。如果另一个swing度是自由的，则限制的最大值为 π/2。

### Drives
----------
D6 有一个线性驱动(Drive)模型和两种可能的角度驱动(Drive)模型。驱动器是比例导数(proportional derivative)驱动器(Drive)，其施加的力如下所示:

*force = spring * (targetPosition - position) + damping * (targetVelocity - velocity)*

驱动模型还可以配置为产生成比例的加速度而不是力，从而考虑关节所附着的`Actor`的质量。加速驱动通常比强制驱动更容易调整。

D6 的线性驱动模型具有以下参数:

+ `target position`, specified in actor0's constraint frame 
+ `target velocity`, specified in actor0's constraint frame 
+ `spring` 
+ `damping` 
+ `forceLimit` - the maximum force the drive can apply (note that this can be an impulse, depending on `PxConstraintFlag::eDRIVE_LIMITS_ARE_FORCES`) 
+ `acceleration drive flag` 

驱动器尝试遵循所需位置输入，并具有配置的刚度(stiffness)和阻尼(damping)属性。由于通过驱动弹簧作用的驱动体的惯性，将发生物理滞后;因此，在多个时间步长内会导致突然的步长变化。物理滞后可以通过stiffening the spring或supplying a velocity target来减少。

当输入位置固定且目标速度为零时，位置驱动器将以指定的弹跳/阻尼(springing/damping)特性绕该驱动器位置弹跳(spring):

```C++
// set all translational degrees free

d6joint->setMotion(PxD6Axis::eX, PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eY, PxD6Motion::eFREE);
d6joint->setMotion(PxD6Axis::eZ, PxD6Motion::eFREE);

// set all translation degrees driven:

PxD6Drive drive(10.0f, -20.0f, PX_MAX_F32, true);
d6joint->setDrive(PxD6JointDrive::eX, drive);
d6joint->setDrive(PxD6JointDrive::eY, drive);
d6joint->setDrive(PxD6JointDrive::eZ, drive);

// Drive the joint to the local(actor[0]) origin - since no angular
// dofs are free, the angular part of the transform is ignored

d6joint->setDrivePosition(PxTransform(1.0f));
d6joint->setDriveVelocity(PxVec3(PxZero));
```

角驱动与线性驱动在根本上的区别在于:角驱动没有一个无奇点的简单直观的表示。因此，`D6 joint`提供了两种角度驱动模型(angular drive models) - `twist and swing`和 `SLERP` (球面线性插值)。

这两个模型在估计当前方向和目标方向之间的四元数空间中路径的方式不同。在 `SLERP` 驱动器中，直接使用四元数。在`twist and swing`驱动中，它被分解成单独的twist和swing组件，每个组件分别插值。在许多情况下，twist和swing是直观的;但是，当被驱动到180度swing时，存在奇点。此外，驱动器不会遵循两个方向之间的最短弧度。另一方面，SLERP驱动器将遵循一对角度配置之间的最短弧，但可能会导致`joint`的twist和swing发生不直观的变化。

角驱动模型具有以下参数:

+ An `angular velocity` target specified relative to actor0's constraint frame 
+ An `orientation target` specified relative to actor0's constraint frame 
+ drive specifications for SLERP (slerpDrive), swing (swingDrive) and twist (twistDrive): 
+ `spring` - amount of torque needed to move the joint to its target orientation proportional to the angle from the target (not used for a velocity drive). 
+ `damping` - applied to the drive spring (used to smooth out oscillations about the drive target). 
+ `forceLimit` - the maximum torque the drive can apply (note that this can be an impulsive torque, depending on the value PxConstraintFlag::eDRIVE_LIMITS_ARE_FORCES) 
+ `acceleration drive flag`. If this flag is set the acceleration (rather than the force) applied by the drive is proportional to the angle from the target. 

当驱动目标输入(drive target inputs)与关节自由度(joint freedom)和极限约束(limit constraints)一致时，将获得最佳结果。

**如果锁定了任何角度自由度，则忽略 `SLERP` 驱动器参数。如果解锁了所有角度自由度，并为多个角度驱动器设置了参数，则将使用 `SLERP` 参数。**

### Configuring Joints for Best Behavior
----------------------------------------
`PhysX` 中关节的行为质量很大程度上取决于迭代求解器收敛的能力。只需增加控制求解器迭代计数的 `PxRigidDynamic` 的属性即可实现更好的收敛。但是，也可以对`joint`进行配置以产生更好的收敛。

+ 当轻对象(light object)被限制在两个重物之间时，求解器可能难以很好地收敛。在这种情况下，最好避免质量比高于10。

+ 当一个`body`明显比另一个`body`重时，使较轻的`body`成为关节中的第二个`Actor`。同样，当其中一个对象是`static`的或`kinematic`的(或者 actor 指针为 NULL)时，使动`dynamic body`成为第二个 actor。

关节的常见用途是在世界中移动物体。当求解器能够访问运动速度以及位置变化时，将获得最佳结果。

+ 如果你想要一个非常僵硬(stiff)的控制器，将对象移动到每帧的特定位置，请考虑将对象与`kinematic actor`连接，并使用 `setKinematicTarget` 函数来移动 `actor` 。

+ 如果您想要一个更具弹性(springy)的控制器，请使用带有驱动目标的`D6 joint`来设置所需的位置和方向，并控制弹簧参数以增加刚度(stiffness)和阻尼(damping)。通常，加速驱动比强制驱动更容易调整。

当使用质量缩放(mass scaling)或沿某些轴约束具有无限惯性的物体时，刚体自由度的降低加上浮点计算中的轻微不准确之处可能会产生任意僵硬的约束响应(arbitrarily stiff constraint responses)，试图纠正不明显的小误差。This can appear, for example, when attempting to perform 2D-simulation using infinite inertia to suppress velocity out of the plane of simulation.在这些情况下，请将标志 `PxConstraintFlag::eDISABLE_PREPROCESSING`，并将约束上的 `minResponseThreshold` 设置为较小的值，例如 1e-8。这将导致在遇到这种僵硬的约束行(stiff constraint rows)时被忽略，并且可以显著提高仿真质量。

## Custom Constraints
---------------------
也可以向 `PhysX` 添加新的关节类型。使用 `PhysXExtensions` 库中的现有`joint`作为参考，以及 `SnippetCustomJoint` 的源代码，该源代码显示了如何实现滑轮`joint`(Pulley Joint)。Serialization一章中讨论了序列化自定义对象，因此此处的讨论仅限于如何在模拟中实现所需的行为。这是一个高级主题，假设熟悉刚体仿真背后的数学原理。此处的演示假定关节约束两个`body`;`static body`的情况等价于无限质量的`dynamic body`，其变换是恒等式。

实现关节动态行为的函数是 `PhysX shaders`，其性质类似于 `PxFilterShader` (请参Collision Filtering)。特别是，这些函数可以并行和异步执行，并且不应访问除作为参数传入的状态之外的任何状态。

要创建自定义关节类，请定义以下内容:

+ 实现约束行为的函数。这些函数必须是无状态的，因为它们可以从多个线程同时调用。当调用每个函数时， `PhysX` 会传递一个常量块，该常量块可用于存储关节配置参数(偏移，轴，极限等)。
+ `PxConstraintShaderTable` 的静态实例，其中包含指向函数的指针
+ 一个实现 `PxConstraintConnector` 接口的类，它将自定义`joint`连接到 `PhysX`。

### Defining Constraint Behavior
--------------------------------
有两个函数定义了关节行为:求解器准备函数(solver preparation function)，它为 `PhysX` 的基于速度的约束求解器生成输入，以及投影函数(projection function)，它允许直接校正位置误差。

仿真过程中的处理顺序如下:

+ 在 `simulate()` 函数中，在开始模拟之前，场景会更新关节常数块的内部副本(以便可以在模拟过程中修改关节的副本而不会引起races)。
+ 碰撞检测运行，并可能唤醒`body`。如果关节连接两个物体，模拟将确保两个物体都处于清醒状态，或者两个物体都不是。
+ 对于连接到清醒体的每个关节，仿真调用求解器准备函数。
+ 求解器更新`body`速度和位置。
+ 如果设置了约束的 `ePROJECTION` 标志，则模拟将调用joint projection function。

#### The Solver Preparation Function
------------------------------------
`joint`的求解器准备函数(solver preparation function)具有以下特征:

```C++
PxU32 prepare(Px1DConstraint* constraints,
              PxVec3& bodyAWorldOffset,
              PxU32 maxConstraints,
              PxConstraintInvMassScale &invMassScale,
              const void* constantBlock,
              const PxTransform& bA2w,
              const PxTransform& bB2w);
```

参数如下:

+ `constraints` 约束行(constraint rows)的输出缓冲区。
+ `bodyAWorldOffset` 在世界空间中指定为与`body`A原点的偏移的点，约束力在其作用以强制关节。约束求解器忽略此值，因为信息已在约束数组中编码，但在报告力时，必须选择一个将力视为作用的点。对于 `PhysX` 关节，使用`body` B 上关节的连接点。
+ `maxConstraints` 缓冲区的大小，它限制可能生成的约束行数。
+ `invMassScale` 应应用于`body`以解析关节的逆质量标度(inverse mass scales)。在标准关节中，这些只是关节的质量刻度参数(参见Mass Scaling)。
+ `constantBlock` 仿真的关节常数块的副本。
+ `bA2w` 第一个`body`的transform。如果`Actor`是static的，或者在约束创建中提供了 NULL 指针，则它是identity transform。
+ `bB2w` 第二个`body`的transform。如果`Actor`是static的，或者在约束创建中提供了 NULL 指针，则它是identity transform。

求解器准备函数的作用是填充 `Px1D` 约束的缓冲区，为force reporting提供应用点，并提供质量刻度属性(mass scaling properties)。返回值是在输出缓冲区中生成的 Px1D 约束数。

请注意，尽管关节参数(relative pose等)通常是相对于`Actor`指定的，但解器准备函数适用于基础刚体的transforms。constraint infrastructure(请参见Data Management)可帮助关节在应用程序修改`Actor`的质心时保持一致性。

每个 `Px1D constraint` 约束两个`body`之间的一个自由度。结构如下所示:

```C++
struct Px1DConstraint
{
    PxVec3                linear0;
    PxReal                geometricError;
    PxVec3                angular0;
    PxReal                velocityTarget;

    PxVec3                linear1;
    PxReal                minImpulse;
    PxVec3                angular1;
    PxReal                maxImpulse;

    union
    {
        struct SpringModifiers
        {
            PxReal        stiffness;
            PxReal        damping;
        } spring;
        struct RestitutionModifiers
        {
            PxReal        restitution;
            PxReal        velocityThreshold;
        } bounce;
    } mods;

    PxReal               forInternalUse;
    PxU16                flags;
    PxU16                solveHint;
}
```

每个 `Px1D Constraint` 要么是`hard constraint`(例如，固定`joint`的一个轴)，要么是`soft constraint`(例如，弹簧)。关节可能具有硬约束行和软约束行的混合物 - 例如，布娃娃肩膀处的致动关节通常具有:

+ 3 hard 1D-constraints which prevent the shoulder from separating. 防止肩部分离。
+ 3 hard 1D-constraints constraining the angular degrees of freedom within some limits. 将角度自由度限制在某些限制内。
+ 3 soft constraints simulating resistance to angular motion from muscles. 模拟肌肉对角运动的阻力。

除非设置了 `Px1DConstraintFlag::eSPRING` 标志，否则该约束将被视为硬约束。

对于软约束和硬约束，每行的求解器速度都是数量:

```C++
v = body0vel.dot(lin0, ang0) - body1vel.dot(lin1, ang1)
```

##### Hard Constraints
-----------------------------
对于硬约束，求解器尝试生成:

+ 一组运动求解器速度 `vMotion` 用于对象，这些对象在积分时会尊重约束误差，由等式表示:

```C++
vMotion + (geometricError / timestep) = velocityTarget
```

+ 一组仿真后求解器(post-simulation solver)速度 `vNext` ，用于服从约束的对象:

```C++
vNext = velocityTarget
```

运动速度用于积分，然后被丢弃。仿真后速度是 `getLinearVelocity()` 和 `getAngularVelocity()` 返回的值。

硬约束有两种特殊选项，两者都最常用于实现limit:恢复原状(restitution)和速度偏差(velocity biasing)。它们由约束标志(constraint flags) `eRESTITUTION` 和 `eKEEPBIAS` 设置，是相互排斥的，并且 `restitution` 优先(从某种意义上说，如果设置了 `restitution` ，则忽略biasing)。

`Restitution` 模拟反弹(例如，超出限制)。如果仿真开始时的冲击求解器速度(impact solver velocity) `vCurrent` 超过恢复速度阈值(restitution velocity threshold)，则约束的目标速度将设置为:

```C++
restitution * -vCurrent
```
并且输入速度目标字段将被忽略。要使用 `Restitution` ，请设置 `Px1DConstraintFlag::eRESTITUTION`。

速度偏置生成仿真后速度，以满足与运动速度相同的约束:

```C++
vNext + (geometricError / timestep) = velocityTarget
```

例如，如果关节正在接近 limit 但尚未达到limit，则这可能很有用。如果目标速度为 0，并且几何误差是剩余到limit的距离，则求解器会将速度限制在积分后违反limit所需的速度以下。然后，关节应平稳收敛到limit。

##### Soft Constraints
-------------------------
或者，求解器可以尝试将速度约束解析为隐式弹簧。在这种情况下，运动速度 `vMotion` 和仿真后速度 `vNext` 是相同的。求解器求解等式:

```C++
F = stiffness * -geometricError + damping * (velocityTarget - v)
```

其中 F 是约束力。

弹簧是完全implicit的:即力或加速度是求解后位置和速度的函数。有一个特殊选项仅适用于软约束:加速弹簧(`PxConstraintFlag::eACCELERATION`)。使用此选项，求解器将根据两个物体的响应来缩放力的大小;它有效地隐式地求解了等式:

```C++
acceleration = stiffness * -geometricError + damping * (velocityTarget - v)
```

##### Force Limits and Reporting
-----------------------------------
所有约束都支持对每一row施加的最小或最大脉冲力的limit。有一个用于强制limit的特殊标志:`eHAS_DRIVE_FORCE_LIMIT`。如果设置了这个标志，除非为约束设置了`PxConstraintFlag::eLIMITS_ARE_FORCES`,那么强制limit将按时间步进行缩放。

`1D constraint`上的标志 `eOUTPUT_FORCE` 确定应用于该row的力是否应该包含在约束力输出中。报告的力也用于内部确定joint breakage。例如，如果创建一个带有角驱动(angular drive)的spherical joint，当线性部分上的应力超过阈值时，该spherical joint会break，请为线性相等行(linear equality rows)设置标志，但不要为角驱动行(angular drive rows)设置标志。

##### Solver Preprocessing
--------------------------
joint solver尝试预处理硬约束以改善收敛性。`solveHint` 值控制每行的预处理：

+ 如果约束是具有无界脉冲limits(unbounded impulse limits)的硬相等约束（即脉冲限制为 -PX_MAX_FLT 且PX_MAX_FLT），请将其设置为 `PxConstraintSolveHint::eEQUALITY`。
+ 如果其中一个 force limit 为零，而另一个 force limit 为无界，请将其设置为 `PxConstraintSolveHint::eINEQUALITY`。
+ 对于所有软约束，以及除上述情况外具有脉冲力限制的硬约束，请将其设置为 `PxConstraintSolveHint::eNONE`。

求解器不会检查提示值是否与 `Px1DConstraint` 中的值一致。使用不一致的值可能会导致未定义的行为。

#### The Projection Function
------------------------------
关节可能为模拟指定的另一个行为是`Projectio`。这是一种纯粹的位置校正，设计用于在基于速度的求解器发生故障时起作用。`projection function`具有以下签名：

```C++
typedef void (*PxConstraintProject)(const void* constantBlock,
                                    PxTransform& bodyAToWorld,
                                    PxTransform& bodyBToWorld,
                                    bool projectToA);
```

它接收常量块，两个body transforms。如果设置了 projectToA 标志，它应该更新 `bodyBToWorld` 转换，否则应该更新 `bodyBToWorld` 转换。有关如何定义投影函数的示例，请参阅扩展库中的实现。

### The Constraint Shader Table
-------------------------------
对行为函数(behavior functions)进行编码后，我们可以定义一个类型为 `PxConstraintShaderTable` 的结构，该结构保存指向约束函数(constraint functions)的指针。此结构将作为参数传递给 `PxPhysics::createConstraint`，并由关节的所有实例共享：

```C++
struct PxConstraintShaderTable
{
    PxConstraintSolverPrep          solverPrep;
    PxConstraintProject             project;
    PxConstraintVisualize           visualize;
};
```

约束可视化工具允许关节使用 `PxConstraintVisualizer` 接口生成可视化信息。该接口的功能有点偏向于标准接头;其使用示例可以在扩展库中找到。

### Data Management
---------------------
接下来，我们可以定义允许 `PhysX` 管理关节的类。此类应继承自 `PxConstraintConnector` 接口。

要创建关节，请调用 `PxPhysics::createConstraint`。此函数的参数是受约束的`Actor`、连接器对象(connector object)、着色器表(shader table)以及关节常量块的大小。返回值是指向 `PxConstraint` 对象的指针。

`PxConstraintConnector` 有许多数据管理回调：

```C++
virtual void*           prepareData();
virtual void            onConstraintRelease();
virtual void            onComShift(PxU32 actor);
virtual void            onOriginShift(const PxVec3& shift);
virtual void*           getExternalReference(PxU32& typeID);
```

这些函数通常是样板;可以在扩展库中找到关节的示例实现：

+ `prepareData()` 函数请求指向joint常量块的指针，并允许joint更新任何状态缓存等。当函数返回时，场景会创建此数据的内部副本，以便在没有争用条件的模拟过程中可以修改joint。在将joint插入场景后，在模拟步骤开始时调用该函数，并在后续模拟步骤中，如果 `PhysX` 被告知joint状态已更改，则调用该函数。要通知 `PhysX` joint状态已更改，请调用 `PxConstraint::markDirty()`。
+ `onConstraintRelease()` 与joint删除相关联。要删除joint，请在约束上调用 PxConstraint::release()。当可以安全地销毁关节时（因为当前执行的仿真线程没有内部引用），约束代码将调用 `PxConstraint::onConstraintRelease()`。此功能可以安全地运行析构函数并释放关节的记忆等。
+ 当应用程序在通过关节连接的一个`Actor`上调用 `setCMassLocalPose()` 时，将调用` onComShift()`。之所以提供此功能，是因为求解器准备和投影函数是使用底层刚体的框架定义的，但关节配置通常是根据`Actor`来定义的。
+ `onOriginShift()` 在应用程序移动场景的原点时调用。这是必要的，因为某些关节可能具有 NULL 执行组件，表示它们附加到世界框架。
+ `getExternalReference()` 由 `PhysX` 用于报告涉及约束的仿真事件，尤其是breakage。在事件回调中，返回的指针直接传递给应用程序，以及应用程序可以使用的 `typeID` ，以便将指针转换为适当的类型。对于每个自定义关节类型， `typeID` 应该是不同的，并且与 `PxJointConcreteType` 中的任何值都不同。如果接头还实现了 `PxBase` 接口，则对 `typeID` 使用 `PxBase` 中的具体类型值。

