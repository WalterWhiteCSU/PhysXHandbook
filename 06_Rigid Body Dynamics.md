# Rigid Body Dynamics
---------------
在本章中，我们将介绍一些主题，一旦您熟悉了设置基本的刚体模拟世界，这些主题也很重要。

## Velocity
---------------
刚体的运动分为线性和角速度分量。在仿真过程中， `PhysX` 将根据重力、其他施加的力和扭矩以及各种约束（如碰撞或joint）来修改物体的速度。

可以使用以下方法读取物体的线性和角速度：

```C++
PxVec3 PxRigidBody::getLinearVelocity();
PxVec3 PxRigidBody::getAngularVelocity();
```

可以使用以下方法设置物体的线性和角速度：

```C++
void PxRigidBody::setLinearVelocity(const PxVec3& linVel, bool autowake);
void PxRigidBody::setAngularVelocity(const PxVec3& angVel, bool autowake);
```

## Mass Properties
---------------
`dynamic actor` 需要质量属性：质量、惯性矩和质量中心frame，质量中心frame指定`Actor`的质心位置及其主惯性轴。计算质量属性的最简单方法是使用 `PxRigidBodyExt::updateMassAndInertia（）` 帮助程序函数，该函数将根据`Actor`的`Shape`和均匀的密度值设置所有三个属性。此功能的变体允许组合每个`Shape`的密度和手动指定某些质量属性。有关更多详细信息，请参阅 `PxRigidBodyExt` 的参考。

北极样本中摇摇晃晃的雪人说明了不同质量属性的使用。雪人就像罗利聚乙烯玩具，通常只是一个空壳，底部装满了一些沉重的材料。低质心导致它们在倾斜后移回直立位置。它们有不同的风格，具体取决于质量属性的设置方式：

+ 第一种基本上是无质量的。在 `Actor` 的底部只有一个质量相对较高的小球体。由于产生的惯性矩很小，这导致相当快速的运动。雪人感觉很轻。

+ 第二个仅使用底部雪球的质量，导致更大的惯性。后来，质心被移动到`Actor`的底部。这种近似在物理上绝是不正确的，但由此产生的雪人感觉更饱满一些。

+ 第三个和第四个雪人使用`Shape`来计算质量。不同之处在于，人们首先计算惯性矩（从实际质心），然后将质心移动到底部。另一个计算我们传递给计算例程的低质心的惯性矩。请注意第二种情况的摆动速度要慢得多，尽管两者的质量相同。这是因为头部在惯性矩（与质心的距离平方）中占了更多。

+ 最后一个雪人的质量属性是手动设置的。该示例使用惯性矩的粗略值来创建特定的所需行为。对角线张量在 X 中具有低值，在 Y 和 Z 中具有高值，从而产生围绕 X 轴旋转的低阻力和围绕 Y 和 Z 的高阻力。因此，雪人只会围绕X轴来回摆动。

如果您有一个 3x3 惯性矩阵（例如，您的对象有现实中的惯性张量），请使用 `PxDiagonalize（）` 函数获取主轴和对角惯性张量来初始化 `PxRigidDynamic actor`。

当手动设置物体的质量/惯性张量时，PhysX 需要质量和每个主惯性轴的正值。但是，在这些值中提供 0 是合法的。当提供0质量或惯性值时，PhysX将其解释为表示围绕该主轴的无限质量或惯性。这可用于创建抵抗所有线性运动或抵抗所有或某些角度运动的物体。使用此方法可以实现的效果示例如下：

 + Bodies that behave as if they were kinematic. 
 + Bodies whose translation behaves kinematically but whose rotation is dynamic. 
 + Bodies whose translation is dynamic but whose rotation is kinematic. 
 + Bodies which can only rotate around a specific axis. 

下面详细介绍了可以取得的成就的一些例子。首先，让我们假设我们正在创建一个共同的结构 - 一个风车。下面提供了构建将成为风车一部分的body 的代码：

```C++
PxRigidDynamic* dyn = physics.createRigidDynamic(PxTransform(PxVec3(0.f, 2.5f, 0.f)));
PxRigidActorExt::createExclusiveShape(*dyn, PxBoxGeometry(2.f, 0.2f, 0.1f), material);
PxRigidActorExt::createExclusiveShape(*dyn, PxBoxGeometry(0.2f, 2.f, 0.1f), material);
dyn->setActorFlag(PxActorFlag::eDISABLE_GRAVITY, true);
dyn->setAngularVelocity(PxVec3(0.f, 0.f, 5.f));
dyn->setAngularDamping(0.f);
PxRigidStatic* st = mPhysics.createRigidStatic(PxTransform(PxVec3(0.f, 1.5f, -1.f)));
PxRigidActorExt::createExclusiveShape(*t, PxBoxGeometry(0.5f, 1.5f, 0.8f), material);
scene.addActor(dyn);
scene.addActor(st);
```

上面的代码为风车创建了一个静态箱形框架，并创建了一个十字形来表示风车的叶片。我们关闭风车叶片上的重力和角度阻尼，并赋予其初始角速度。因此，该风车叶片将无限期地以恒定的角速度旋转。但是，如果另一个物体与风车相撞，我们的风车将停止正常运行，因为风车叶片将被撞出原位。有几种选择可以使风车叶片在其他物体与之相互作用时保持在正确的位置。其中一种方法可能是使风车具有无限的质量和惯性。在这种情况下，与物体的任何相互作用都不会影响风车：

```C++
dyn->setMass(0.f);
dyn->setMassSpaceInertiaTensor(PxVec3(0.f));
```

此示例无限期地保留了风车以恒定角速度旋转的先前行为。然而，现在`body`的速度不能受到任何约束的影响，因为`body`有无限的质量和惯性。如果一个物体与风车叶片相撞，碰撞的行为就好像风车叶片是一个`kinematic`物体。

另一种选择是使风车具有无限的质量，并将其旋转限制在`body`的局部z轴周围。这将提供与在风车和静态风车框架之间应用旋转接头相同的效果：

```C++
dyn->setMass(0.f);
dyn->setMassSpaceInertiaTensor(PxVec3(0.f, 0.f, 10.f));
```

在这两个例子中，`body`的质量都设置为0，表明`body`具有无限的质量，因此其线速度不能被任何约束改变。但是，在此示例中，body 的惯性被配置为允许body 的角速度受围绕一个主轴或惯性的约束的影响。这提供了与引入旋转接头(a revolute joint)类似的效果。z轴周围的惯性值可以增加或减少，以使风车对运动的抵抗力更大/更小。

## Applying Forces and Torques
---------------------------

与物体相互作用的最物理友好的方式是向它施加力。在经典力学中，物体之间的大多数相互作用通常通过使用力来解决。由于定理：

$ f = m * a(force = mass * acceleration) $

力直接控制物体的加速度，但其速度和位置只是间接的。因此，如果您需要立即响应，通过强制控制可能会不方便。力的优点是，无论您对场景中的实体施加何种力，模拟都能够保持所有定义的约束（joint和接触）得到满足。例如，重力的工作原理是对物体施加力。

不幸的是，以系统的共振频率对铰接体(articulated bodies)施加大的力可能会导致速度不断增加，并最终导致求解器无法维持joint约束。这与现实世界的系统没有什么不同，在现实世界中，joint最终会断裂。

作用在body 上的力在每个模拟帧之前累积，施加到模拟中，然后重置为零，为下一帧做准备。下面列出了 `PxRigidBody` 和 `PxRigidBodyExt` 的相关方法。有关更多详细信息，请参阅 API 参考：

```C++
void PxRigidBody::addForce(const PxVec3& force, PxForceMode::Enum mode, bool autowake);
void PxRigidBody::addTorque(const PxVec3& torque, PxForceMode::Enum mode, bool autowake);

void PxRigidBodyExt::addForceAtPos(PxRigidBody& body, const PxVec3& force,
    const PxVec3& pos, PxForceMode::Enum mode, bool wakeup);
void PxRigidBodyExt::addForceAtLocalPos(PxRigidBody& body, const PxVec3& force,
    const PxVec3& pos, PxForceMode::Enum mode, bool wakeup);
void PxRigidBodyExt::addLocalForceAtPos(PxRigidBody& body, const PxVec3& force,
    const PxVec3& pos, PxForceMode::Enum mode, bool wakeup);
void PxRigidBodyExt::addLocalForceAtLocalPos(PxRigidBody& body, const PxVec3& force,
    const PxVec3& pos, PxForceMode::Enum mode, bool wakeup);
```

`PxForceMode` 成员默认为 `PxForceMode::eFORCE` 以应用简单力。还有其他可能性。例如，`PxForceMode::eIMPULSE` 将施加冲动力。 `PxForceMode::eVELOCITY_CHANGE` 也会做同样的事情，但也会忽略`body`的质量，有效地导致瞬间的速度变化。请参阅 `PxForceMode` 的 API 文档，了解其他可能性。

**注意： `PxRigidBodyExt` 中的方法仅支持力模式 `eFORCE` 和 `eIMPULSE` 。**

还有其他扩展函数，用于计算在下一个仿真帧中，如果要施加脉冲力或脉冲扭矩，则会出现线性和角速度变化：

```c++
void PxRigidBodyExt::computeVelocityDeltaFromImpulse(const PxRigidBody& body,
    const PxVec3& impulsiveForce, const PxVec3& impulsiveTorque, PxVec3& deltaLinearVelocity,
    PxVec3& deltaAngularVelocity);
```

此函数的一个用例可能是预测游戏对象的更新速度，以便在body 可能超过帧末尾的阈值速度时，可以在模拟帧之前启动资产加载。脉冲力和扭矩只是施加到`body`上的力和扭矩乘以模拟框架的时间步长。忽略约束力和接触力的影响，预计在下一个仿真帧中出现的线性和角速度的变化在 `delta` 线性 `Velocity` 和 `deltaAngularVelocity` 中返回。然后可以使用 `body.getLinearVelocity（） + deltaLinearVelocity` 计算预测的线速度，而预测的角速度可以使用 `body.getAngularVelocity（） + deltaAngularVelocity` 来计算。如果需要，可以使用 `body.setLinearVelocity（body.getLinearVelocity（） + deltaLinearVelocity）` 和 `body.setAngularVelocity（body.getAngularVelocity（） + deltaAngularVelocity）` 立即更新 `body` 的速度。

## Gravity
---------------
重力是模拟中常见的力， `PhysX` 使其特别容易应用。对于场景范围的重力效果或任何其他均匀的力场，请使用 `PxScene::setGravity（）` 设置 `PxScene` 类的重力矢量。

参数是重力引起的加速度。以米和秒为单位，这在地球上的星等约为9.8，并且应该指向下方。将在场景中每个物体的质心处施加的力是这个加速度矢量乘以`Actor`的质量。

某些特效可能要求一些动态`Actor`不受重力的影响。要指定此标志，请设置标志：

```C++
PxActor::setActorFlag(PxActorFlag::eDISABLE_GRAVITY, true);
```

**注意：在模拟过程中更改重力（或启用/禁用重力）时要小心。出于性能原因，更改不会自动唤醒睡眠中的`Actor`。因此，可能需要遍历所有`Actor`并手动调用`PxRigidDynamic::wakeUp（）`。**

`PxActorFlag::eDISABLE_GRAVITY` 的替代方法是对整个场景使用零重力矢量，然后将自己的重力施加到刚体上，每帧。这可用于创建径向重力场，如 `SampleCustomGravity` 中所示。

## Friction and Restitution
--------------------------
所有物理对象都至少有一种材质，该材质定义了用于解决与对象碰撞的摩擦和恢复属性。

要创建材质，请调用 `PxPhysics::createMaterial（）`：

```C++
PxMaterial* mMaterial;

mMaterial = mPhysics->createMaterial(0.5f, 0.5f, 0.1f); // static friction, dynamic friction,
                                                        // restitution
if(!mMaterial)
    fatalError("createMaterial failed!");
```

材质归 `PxPhysics` 对象所有，并且可以在多个场景中的对象之间共享。碰撞中涉及的两个物体的材料属性可以通过各种方式组合。有关更多详细信息，请参阅 `PxMaterial` 的参考文档。

碰撞几何体为三角形网格(triangle mesh )或高度场(heightfield)（请参见Shape）的 `PhysX` 对象可以为每个三角形提供材质。

摩擦使用库仑摩擦模型，该模型基于2个系数的概念：静态摩擦系数和动态摩擦系数（有时称为动摩擦）。摩擦抵抗两个接触的固体表面的相对横向运动。这两个系数定义了每个表面施加在另一个表面上的法向力与施加的摩擦力与抵抗横向运动的摩擦力之间的关系。静态摩擦力定义了在不横向相互移动的曲面之间施加的摩擦量。动态摩擦力定义了彼此相对移动的表面之间施加的摩擦量。

两个碰撞物体的恢复系数是一个分数值，表示撞击后和撞击前的速度比，沿撞击线取。据说恢复系数为1的弹性碰撞，而<恢复系数为1则称为无弹性的。

## Sleeping
-----------
当一个`Actor`在一段时间内不移动时，假设它将来也不会移动，直到一些外力作用于它，使其失去平衡。在此之前，不再模拟以节省资源。此状态称为睡眠(sleeping)。您可以使用以下方法查询执行组件的睡眠状态：

```C++
bool PxRigidDynamic::isSleeping() const;
```

但是，在`Actor`入睡或醒来时侦听 SDK 发送的事件通常更方便。要接收以下事件，必须为执行组件设置 `PxActorFlag::eSEND_SLEEP_NOTIFIES`：

```C++
void PxSimulationEventCallback::onWake(PxActor** actors, PxU32 count) = 0;
void PxSimulationEventCallback::onSleep(PxActor** actors, PxU32 count) = 0;
```

有关详细信息，请参阅 Callback Sequence 和 Sleep state change events 小节。

当`Actor`的动能在一段时间内低于给定阈值时，它就会进入睡眠状态。基本上，每个动态刚性 `Actor` 都有一个唤醒计数器，当 `Actor` 的动能低于指定的阈值时，该计数器会因仿真时间步长而递减。但是，如果在仿真步骤后能量高于阈值，则计数器将重置为最小默认值，并且整个过程将重新开始。一旦唤醒计数器达到零，它就不会进一步减少，并且 `Actor` 可以进入睡眠状态。请注意，零唤醒计数器并不意味着`Actor`必须处于睡眠状态，它仅表示它已准备好进入睡眠状态。还有其他因素可能会让`Actor`保持更长时间的清醒。

能量阈值以及 `Actor` 保持清醒的最短时间可以使用以下方法进行操作：

```C++
void PxRigidDynamic::setSleepThreshold(PxReal threshold);
PxReal PxRigidDynamic::getSleepThreshold() const;

void PxRigidDynamic::setWakeCounter(PxReal wakeCounterValue);
PxReal PxRigidDynamic::getWakeCounter() const;
```

**注意：对于`kinematic actors`，特殊的睡眠规则适用。除非设置了目标姿势，否则`kinematic actors`处于睡眠状态（在这种情况下，它将保持清醒状态，直到下一个模拟步骤结束，其中不再设置目标姿势）。因此，不允许对`kinematic actors`使用 `setWakeCounter（）`。**

如果`dynamic rigid actor`正在休眠，则保证以下状态：

+ The wake counter is zero(唤醒计数器为零). 
+ The linear and angular velocity is zero(线性和角速度为零). 
+ There is no force update pending(没有挂起的强制更新). 

当`Actor`入到场景中时，如果上述所有点都保持不变，它将被视为睡眠，否则它将被视为清醒。

通常，如果至少存在以下一项，则动态刚性actor(dynamic rigid actor)保证处于清醒状态：

+ The wake counter is positive(唤醒计数器为正). 
+ The linear or angular velocity is non-zero(线性或角速度为非零). 
+ A non-zero force or torque has been applied(施加了非零力或扭矩). 

因此，以下调用将自动唤醒执行组件：

+ `PxRigidDynamic::setWakeCounter()`, if the wake counter value is larger than zero(唤醒计数器大于0). 
+ `PxRigidBody::setLinearVelocity(), ::setAngularVelocity()`, if the velocity is non-zero(速度非0). 
+ `PxRigidBody::addForce(), ::addTorque()`, if the torque is non-zero(扭矩非0). 

此外，以下调用和事件会唤醒执行组件：

+ `PxRigidDynamic::setKinematicTarget()` in the case of a kinematic actor (因为这还会将唤醒计数器设置为正值). 
+ `PxRigidActor::setGlobalPose()`, if the autowake parameter is set to true (default).
+ Simulation gets disabled for a `PxRigidActor` by raising `PxActorFlag::eDISABLE_SIMULATION`. 
+ `PxScene::resetFiltering()`. 
+ `PxShape::setSimulationFilterData()`, if the subsequent re-filtering causes the type of the shape pair to transition between suppressed, trigger and contact(如果随后的重新过滤导致Shape对的类型在suppressed、trigger和contact之间转换). 
+ 与awake的`Actor`touch。
+ 一个`touching rigid actor` 从场景中移除（这是默认行为，但可以由用户指定，请参阅下面的注释）。
+ 与`static rigid actor`的Contact丢失。
+ 与`dynamic rigid actor`的Contact丢失，该`Actor`在下一个模拟步骤中醒来。
+ `Actor`被双向交互粒子(two-way interaction particle)击中。

**从场景中移除`Rigid Actor`或从 `Actor` 移除Shape时，可以指定是否唤醒在上一个模拟步骤中接触已移除对象的对象。有关详细信息，请参阅 `PxScene::removeActor（）` 和 `PxRigidActor::detachShape（）` 中的 API 注释。**

要显式唤醒睡眠对象或强制对象进入睡眠状态，请使用：

```C++
void PxRigidDynamic::wakeUp();
void PxRigidDynamic::putToSleep();
```

** *注意*：不允许将这些方法用于kinematic actors。kinematic actors的睡眠状态仅根据是否设置了目标位置来定义。 **

API 参考准确记录了哪些方法会导致执行组件被唤醒。

### Sleep state change events
----------------------------
如上所述， `PhysX` 提供了一个事件系统，用于报告在`PxScene::fetchResults（）`期间动态刚体的睡眠状态的变化：

```C++
void PxSimulationEventCallback::onWake(PxActor** actors, PxU32 count) = 0;
void PxSimulationEventCallback::onSleep(PxActor** actors, PxU32 count) = 0;
```

了解这些事件的正确用法及其局限性非常重要：

+ 自上一次 `fetchResults（）` 或 `flushSimulation（）` 以来添加的 `body` 将始终生成事件，即使没有发生睡眠状态转换也是如此。
+ 如果自之前的 `fetchResults（）` 或 `flushSimulation（）` 以来，`body`的睡眠状态发生了多次变化， `PhysX` 将仅报告最近的更改。

有时需要检测 awake 和 asleep 之间的转换，例如在跟踪awake body的数量时。假设一个asleep的body B 被应用程序唤醒，计数器递增，并且在下一个模拟步骤 B 保持awake。尽管 B 的睡眠状态在模拟期间没有改变，但自之前的 `fetchResults（）` 以来它已经发生了变化，因此将为它生成一个 `onWake（）` 事件。如果计数器再次递增以响应此事件，则其值将不正确。

若要使用睡眠状态事件来检测转换，必须保留感兴趣对象的睡眠状态记录，例如在哈希中。处理事件时，此记录可用于检查是否存在转换。

## Kinematic Actors
----------------------
有时，使用力或约束来控制`Actor`不够强大、精确或灵活。例如，moving platforms或character controllers通常需要操纵`Actor`的位置，或者让它完全遵循特定的路径。这种控制方案由`kinematic actors`提供。

`kinematic actor`使用`PxRigidDynamic::setKinematicTarget（）`函数进行控制。每个模拟步骤 `PhysX` 都会将`Actor`移动到其目标位置，而不管外力，重力，碰撞等。因此，对于每个`kinematic actor`，每次都必须不断调用`setKinematicTarget（）`，以使它们沿着所需的路径移动。`kinematic actor`的运动会影响与它碰撞或被joint约束的动态参与者。`Actor`看起来有无限的质量，并将把常规的`dynamic actors`推开。

要创建`kinematic actor`，只需创建一个常规的`dynamic actors`，然后设置其kinematic标志：

```C++
PxRigidBody::setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, true);
```

使用相同的函数将`kinematic actor`转换回常规`dynamic actor`。虽然您确实需要像为所有`dynamic actor`一样为`kinematic actor`提供质量，但当`Actor`处于运动学模式时，这个质量实际上不会用于任何事情。

警告：

+ 重要的是要了解`PxRigidDynamic::setKinematicTarget（）`和`PxRigidActor::setGlobalPose（）`之间的区别。虽然 `setGlobalPose（）` 也会将 `actor` 移动到所需的位置，但它不会使该 `actor` 与其他对象正确交互。特别是，在 `setGlobalPose（）` 中，`kinematic actor` 不会推开其他`dynamic actors`，而是会直接穿过它们。不过，`setGlobalPose（）` 函数仍然可以使用，如果只是想将`kinematic actor`传送到一个新的位置。
+ `kinematic Actor`可以推开动态对象，但没有什么能把它推回去。因此，kinematic可以很容易地将`dynamic Actor`挤压到`static Actor`或另一个`kinematic Actor`身上。因此，被挤压的动态物体可以深深地穿透它被推入的几何Shape。
+ `kinematic Actor`和`static Actor`之间没有交互或碰撞。但是，可以使用`PxSceneDesc::kineKineFilteringMode`和`PxSceneDesc::staticKineFilteringMode`请求这些情况的联系信息。

## Active Transforms
--------------------
**注意：active transforms当前已弃用。请参阅下一段关于"Active Actors "的替代内容。**

Active Transforms API 提供了一种有效的方法，可以将 PhysX 场景中的 Actor 变换更改反映为关联的外部对象（如渲染网格）。

当调用场景的 `fetchResults（）` 方法时，将生成一个 `PxActiveTransform` 结构数组，数组中的每个条目都包含一个指向移动的执行组件、其用户数据及其新转换的指针。因为只有移动过的`Actor`才会被包括在列表中，所以这种方法可能比单独分析场景中的每个`Actor`更有效。

下面的示例演示如何使用Active Transforms来更新呈现对象：

```C++
// update scene
scene.simulate(dt);
scene.fetchResults();

// retrieve array of actors that moved
PxU32 nbActiveTransforms;
PxActiveTransform* activeTransforms = scene.getActiveTransforms(nbActiveTransforms);

// update each render object with the new transform
for (PxU32 i=0; i < nbActiveTransforms; ++i)
{
    MyRenderObject* renderObject = static_cast<MyRenderObject*>(activeTransforms[i].userData);
    renderObject->setTransform(activeTransforms[i].actor2World);
}
```

**注意：必须在场景中设置 `PxSceneFlag::eENABLE_ACTIVETRANSFORMS`，才能生成Active Transforms数组。**

**注意：由于kinematic rigid bodies的目标变换由用户设置，因此可以通过设置标志`PxSceneFlag::eEXCLUDE_KINEMATICS_FROM_ACTIVE_ACTORS`将kinematics从列表中排除。**

## Active Actors
-----------------
Active Actors API 提供了一种有效的方法来将 `PhysX` 场景中的更改转换为关联的外部对象（如渲染网格）。

当场景的 `fetchResults（）` 方法被调用时，将生成一个活动 PxActor 数组。因为只有移动过的`Actor`才会被包括在列表中，所以这种方法可能比单独分析场景中的每个`Actor`更有效。

下面的示例演示如何使用Active Actors更新渲染对象：

```C++
// update scene
scene.simulate(dt);
scene.fetchResults();

// retrieve array of actors that moved
PxU32 nbActiveActors;
PxActor** activeActors = scene.getActiveActors(nbActiveActors);

// update each render object with the new transform
for (PxU32 i=0; i < nbActiveActors; ++i)
{
    MyRenderObject* renderObject = static_cast<MyRenderObject*>(activeActors[i]->userData);
    renderObject->setTransform(activeActors[i]->getGlobalPose());
}
```
**必须在场景中设置 PxSceneFlag：：eENABLE_ACTIVE_ACTORS，才能生成Active Actors数组。**

**由于 kinematic rigid bodies的目标变换由用户设置，因此可以通过设置标志`PxSceneFlag：：eEXCLUDE_KINEMATICS_FROM_ACTIVE_ACTORS`将kinematic从列表中排除。**

## Dominance
------------
Dominance是一种使动态物体能够相互支配的机制。Dominance有效地将dominant body注入无限质量的一对。这是约束求解器内局部质量修改的一种形式，因此可以覆盖一对中一个物体的质量。通过接触修改中的局部大规模修改可以达到类似的效果，但dominance的优势在于在SDK中自动处理，因此不会产生接触修改的额外内存和性能开销。

必须为每个`Actor`分配一个dominance group ID。这是一个范围 [0， 31] 中的 5 位值。因此，您最多只能使用 32 个dominance group。默认情况下，所有body 都放置在dominance group 0 中。可以在 `PxActor` 上使用以下方法将执行组件分配到dominance group：

```C++
virtual void setDominanceGroup(PxDominanceGroup dominanceGroup) = 0;
```

Dominance由以下结构中的 2 个实数定义：

```C++
struct PxDominanceGroupPair
{
    PxDominanceGroupPair(PxReal a, PxReal b)
        : dominance0(a), dominance1(b) {}
    PxReal dominance0;
    PxReal dominance1;
};
```
在 `PxScene` 上可以使用以下方法配置两个dominance group之间的支配关系：

```C++
virtual void setDominanceGroupPair(PxDominanceGroup group1, PxDominanceGroup group2,
    const PxDominanceGroupPair& dominance) = 0;
```

用户可以为给定的 `PxDominanceGroupPair` 定义3种不同的状态：
* 1：1。这表明两个机构具有同等的支配地位。这是默认行为。
* 1 : 0.这表明`body`B支配`body`A. 
* 0：1。这表明`body`A主导了`body`B。

除 0 和 1 以外的任何值在 `PxDominanceGroupPair` 中无效。将 0 分配给 `PxDominanceGroupPair` 的两端也是无效的。这些值可以被认为是应用于物体各自的逆质量和反惯性的尺度。因此，支配值为0将等同于无限质量体。

下面的示例将两个 actor（actorA 和 actor B）设置为不同的dominance group，并配置dominance group以使 actorA 支配 actorB：

```C++
PxRigidDynamic* actorA = mPhysics->createRigidDynamic(PxTransform(PxIdentity));
PxRigidDynamic* actorB = mPhysics->createRigidDynamic(PxTransform(PxIdentity));

actorA->setDominanceGroup(1);
actorB->setDominanceGroup(2);

mScene->setDominanceGroupPair(1, 2, PxDominanceGroupPair(0.f, 1.f));
```

Dominance不会影响joint。必须在 `PxJoint` 上使用以下方法对joints进行局部质量修改：

```C++
virtual void setInvMassScale0(PxReal invMassScale) = 0;
virtual void setInvMassScale1(PxReal invMassScale) = 0;
virtual void setInvInertiaScale0(PxReal invInertiaScale) = 0;
virtual void setInvInertiaScale1(PxReal invInertiaScale) = 0;
```

如前所述，dominance不允许0或1以外的值，并且任何dominance值都统一应用于逆质量和反惯性。通过contact modification的joint和触点允许定义单独的反质量和反惯性尺度，它们接受[0，PX_MAX_REAL]范围内的任何值，因此可用于实现比支配范围更广的效果。

如果滥用dominance，可能会产生一些非常奇特的结果。例如，给定的body  A、B 和 C 按以下方式配置：

+ Body A dominates body B 
+ Body B dominance body C 
+ Body C dominates body A 

在这种情况下，body A不能直接推动body C。但是，如果它把body B推入body C，它可以推动body C。

## Solver Iterations
--------------------
当刚体的运动受到contacts或joints的约束时，constraint solver就会发挥作用。solver通过迭代限制body运动一定次数的所有约束来满足body 的约束。迭代次数越多，结果就越准确。solver iteration计数默认为 4 次位置迭代和 1 次速度迭代。可以使用以下函数为每个机构单独设置这些计数：

```C++
void PxRigidDynamic::setSolverIterationCounts(PxU32 minPositionIters, PxU32 minVelocityIters);
```

通常，对于具有大量joint且joint误差容差较小的物体，只需要显着增加这些值。如果发现需要使用高于 30 的设置，则可能需要重新考虑模拟的配置。

Solver将contacts分组为friction patches;friction patches是一组contacts，它们共享相同的材料并具有相似的contact normals。但是，Solver允许每个contact manager（一对Shape）最多有 32 个friction patches。如果产生超过 32 个friction patches（这可能是由于非常复杂的碰撞几何Shape或非常大的contact offsets），Solver将忽略剩余的friction patches。发生这种情况时，checked/debug版本中将出现警告。

## Immediate Mode
------------------
除了使用 `PxScene` 进行模拟外， `PhysX` 还提供了一个名为"immediate mode"的低级模拟API。这提供了一个 API 来访问低contact generation和constraint solver。此方法目前仅支持 CPU 刚体，不支持articulations、clothing或particles。

immediate mode API 在 PxImmediateMode.h 中定义，并且有一个代码段演示了它在"SnippetImmediateMode"中的用法。

该 API 提供了contact generation的函数：

```C++
PX_C_EXPORT PX_PHYSX_CORE_API bool PxGenerateContacts(const PxGeometry* geom0, const PxGeometry* geom1, const PxTransform* pose0, const PxTransform* pose1, PxCache* contactCache, const PxU32 nbPairs, PxContactRecorder& contactRecorder,
        const PxReal contactDistance, const PxReal meshContactMargin, const PxReal toleranceLength, PxCacheAllocator& allocator);
```

此函数采用一组位于特定位置的 `PxGeometry` 对象对，并在这些对象之间执行碰撞检测。如果几何图形对发生碰撞，则会生成contacts，这些contacts将报告给 `contactRecorder` 。此外，信息可以缓存在 `contactCache` 中，以加速这些几何体对之间的未来查询。此缓存信息所需的任何内存都将使用"allocator"进行分配。

此外，immediate mode还为constraint solver提供了 API。这些函数包括用于创建求解器使用的实体的函数：

```C++
PX_C_EXPORT PX_PHYSX_CORE_API void PxConstructSolverBodies(const PxRigidBodyData* inRigidData, PxSolverBodyData* outSolverBodyData, const PxU32 nbBodies, const PxVec3& gravity, const PxReal dt);

PX_C_EXPORT PX_PHYSX_CORE_API void PxConstructStaticSolverBody(const PxTransform& globalPose,PxSolverBodyData& solverBodyData);
```

除了构建body之外， `PxConstraintSolverBodies` 还将提供的重力加速度集成到body速度中。

以下函数是可选的，用于批处理约束(batch constraints)：

```C++
PX_C_EXPORT PX_PHYSX_CORE_API PxU32 PxBatchConstraints(PxSolverConstraintDesc* solverConstraintDescs, const PxU32 nbConstraints, PxSolverBody* solverBodies, PxU32 nbBodies, PxConstraintBatchHeader* outBatchHeaders,
        PxSolverConstraintDesc* outOrderedConstraintDescs);
```

batch constraints对提供的constraints重新排序并生成批处理标题，solver可以使用这些批处理标题来加速约束求解(constraint solving)，方法是将独立约束(constraints)组合在一起并使用 SIMD 寄存器中的多个通道并行求解。此过程是完全可选的，如果不需要，可以绕过。请注意，这将更改constraints的处理顺序，这可能会更改求解器(solver)的结果。

提供了以下方法来创建接触约束(contact constraints)：

```C++
PX_C_EXPORT PX_PHYSX_CORE_API bool PxCreateContactConstraints(PxConstraintBatchHeader* batchHeader, const PxU32 nbHeaders, PxSolverContactDesc* contactDescs,
        PxConstraintAllocator& allocator, PxReal invDt, PxReal bounceThreshold, PxReal frictionOffsetThreshold, PxReal correlationDistance);
```

该方法可以与 `PxGenerateContacts` 产生的contacts或由特定于应用的contact generation approaches产生的contacts一起提供。

提供了以下方法来创建关节约束(joint constraints)：

```C++
PX_C_EXPORT PX_PHYSX_CORE_API bool PxCreateJointConstraints(PxConstraintBatchHeader* batchHeader, const PxU32 nbHeaders, PxSolverConstraintPrepDesc* jointDescs, PxConstraintAllocator& allocator, PxReal dt, PxReal invDt);

PX_C_EXPORT PX_PHYSX_CORE_API bool PxCreateJointConstraintsWithShaders(PxConstraintBatchHeader* batchHeader, const PxU32 nbBatchHeaders, PxConstraint** constraints, PxSolverConstraintPrepDesc* jointDescs, PxConstraintAllocator& allocator, PxReal dt, PxReal invDt);
```

这些方法为应用程序定义joint rows或应用程序使用 `PhysX PxConstraint` 对象（创建constraint rows）提供了一种机制。

以下方法解决了这些约束：

```C++
PX_C_EXPORT PX_PHYSX_CORE_API void PxSolveConstraints(PxConstraintBatchHeader* batchHeaders, const PxU32 nbBatchHeaders, PxSolverConstraintDesc* solverConstraintDescs, PxSolverBody* solverBodies,
        PxVec3* linearMotionVelocity, PxVec3* angularMotionVelocity, const PxU32 nbSolverBodies, const PxU32 nbPositionIterations, const PxU32 nbVelocityIterations);
```

此方法执行所有必需的位置和速度迭代，并更新对象的 delta 速度和 motion 速度，这些速度分别存储在 `PxSolverBody` 和 `linear/angularMotionVelocity` 中。

提供以下方法来积分物体的最终位置并更新物体的速度以反映constraint solver产生的运动。

在 `SnippetImmediateMode` 中提供了如何使用即时模式的示例。

## Enhanced Determinism
-------------------------
`PhysX` 提供有限的确定性仿真(Enhanced Determinism)。具体而言，如果使用相同的时间步进方案和在同一平台上运行的相同 `PhysX` 版本来模拟完全相同的场景（以相同的顺序插入相同的 Actor），则在运行之间模拟的结果将是相同的。模拟行为不受所用工作线程数的影响。

但是，如果以不同的顺序插入 `Actor` ，则模拟的结果可能会更改。此外，如果添加了其他 `Actor` 或从场景中删除了一些 `Actor` ，则模拟的整体行为可能会发生变化。这意味着，特定`Actor`集合的模拟可能会根据场景中是否存在其他`Actor`而变化，而不管这些`Actor`是否实际与`Actor`集合进行交互。这种行为属性通常是可以容忍的，但在某些情况下是不可接受的。

为了克服这个问题， `PhysX` 提供了一个标志：`PxSceneFlag::eENABLE_ENHANCED_DETERMINISM`，它提供了额外的确定性级别。具体而言，如果应用程序以确定性顺序插入参与者，并且此标志升起，则无论场景中的任何其他islands如何，islands的模拟都将是相同的。但是，此模式会牺牲一些性能来确保这种额外的确定性。

## Axis locking
----------------
可以使用 `PxRigidDynamicLockFlag` 在 `PhysX` 中限制沿特定世界空间轴或围绕特定世界空间轴的运动。例如，下面的代码片段演示了如何将 `PxRigidDynamic` 限制为二维仿真。在这种情况下，我们允许 `PxRigidDynamic` 仅围绕Z轴旋转，并且仅沿X轴和Y轴平移：

```C++
PxRigidDynamic* dyn = physics.createRigidDynamic(PxTransform(PxVec3(0.f, 2.5f, 0.f)));

...

//Lock the motion
dyn->setRigidDynamicLockFlags(PxRigidDynamicLockFlag::eLOCK_LINEAR_Z | PxRigidDynamicLockFlag::eLOCK_ANGULAR_X | PxRigidDynamicLockFlag::eLOCK_ANGULAR_Y);
```
限制围绕6个自由度的任意组合移动或旋转是合法的。