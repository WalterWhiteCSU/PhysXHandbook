# Articulations
-------------------

关节是单个演员，由一组链接（每个链接的行为都像一个刚体）组成，这些链接通过特殊的关节连接在一起。每个关节都有一个树状结构 - 所以不能有循环或断裂。它们的主要用途是模拟物理驱动的角色。与标准 PhysX 接头相比，它们支持更高的质量比、更精确的驱动模型、更好的动态稳定性和更稳健的接头分离恢复能力。但是，它们的模拟成本要高得多。

虽然关节不直接建立在关节上，但它们使用非常相似的配置机制。在本节中，我们假设熟悉PhysX关节。

## Creating an Articulation
---------------------------
要创建关节，请首先创建不带链接的关节执行组件：

```C++
PxArticulation* articulation = physics.createArticulation();
```

然后逐个添加链接，每次指定父链接（初始链接的父级为 NULL）和新链接的姿势：

```C++
PxArticulationLink* link = articulation->createLink(parent, linkPose);
PxRigidActorExt::createExclusiveShape(*link, linkGeometry, material);
PxRigidBodyExt::updateMassAndInertia(*link, 1.0f);
```

铰接连杆具有刚体功能的有限子集。它们可能不是运动学的，并且不支持阻尼、速度夹紧或接触力阈值。休眠状态和求解器迭代计数是整个连接的属性，而不是单个链接的属性。

每次创建超出第一个链接的链接时，都会在它与其父级之间创建一个 PxArticulationJoint。为每个关节指定关节框架，其方式与 Px 关节完全相同：每次创建超出第一个链接的链接时，都会在它与其父级之间创建一个 PxArticulationJoint。为每个关节指定关节框架，其方式与 Px 关节完全相同：

```C++
PxArticulationJoint* joint = link->getInboundJoint();
joint->setParentPose(parentAttachment);
joint->setChildPose(childAttachment);
```

最后，将铰接添加到场景中：

```C++
scene.addArticulation(articulation);
```

## Articulation Joints
------------------------
目前支持的唯一关节形式是解剖关节，其特性类似于为典型布娃娃配置的D6关节（参见D6关节）。具体而言，关节是球形关节，具有角驱动，子关节框架的x轴周围的扭曲极限以及围绕母关节框架x轴的椭圆摆动锥极限。这些属性的配置与 D6 或球形接头非常相似，但提供的选项略有不同。

摆动极限是硬椭圆锥极限，不支持垂直于极限表面的运动的弹簧或恢复。您可以按如下方式设置极限椭圆角：

```C++
joint->setSwingLimit(yAngle, zAngle);
```

用于 y 和 z 周围的极限角。与 PxJoint 锥形极限不同，限值提供了一个切向弹簧来限制轴沿极限曲面的移动。配置完成后，启用摆动限制：

```C++
joint->setSwingLimitEnabled(true);
```

扭曲极限允许配置上下角：

```C++
joint->setTwistLimit(lower, upper);
```

同样，您必须显式启用它：

```C++
joint->setTwistLimitEnabled(true);
```

与通常的联合限值一样，最好使用足够的限值接触距离值，以便求解器在超过极限阈值之前开始强制执行极限。

铰接接头不可断裂，并且无法检索施加在接头处的约束力。

## Driving an Articulation
----------------------------
铰接通过关节加速弹簧驱动。您可以设置方向目标、角速度目标以及弹簧和阻尼参数，以控制关节向目标的驱动强度。您还可以设置顺应性值，指示关节抵抗加速的强度。接近零的顺应性表示非常强的阻力，顺应性为1表示没有阻力。

铰接分两个阶段驱动。首先施加联合弹簧力（我们使用术语内力来表示这些），然后施加任何外力，例如重力和接触力。您可以在每个阶段的每个接头上提供不同的顺应性值。

请注意，对于接头加速弹簧，弹簧所需的强度仅使用由接头连接的两个主体的质量来估计。相比之下，铰接驱动弹簧占铰接中所有主体的质量，以及其他关节处致动的任何刚度。此估计是一个迭代过程，使用 PxArticulation 类的 externalDriveIterations 和 internalDriveIterations 属性进行控制。

无需为关节驱动设置目标四元数，而是可以将方向误差项直接设置为旋转矢量。该值设置为目标四元数的虚部，实部设置为 0。

*joint->setDriveType(PxArticulationJointDriveType::eERROR); joint->setTargetOrientation(PxQuat(error.x, error.y, error.z, 0));*

这允许弹簧以比2个四元数之间的差异产生的更大的位置误差驱动。通过计算目标四元数、链接帧和联合帧的误差，获得与目标四元数相同的行为，如下所示：

```C++
PxTransform cA2w = parentPose.transform(joint.parentPose);          // parent attachment frame
PxTransform cB2w = childPose.transform(joint.childPose);            // child attachment frame
transforms.cB2cA = transforms.cA2w.transformInv(transforms.cB2w);   // relative transform
if(transforms.cB2cA.q.w<0)                                          // shortest path path
    transforms.cB2cA.q = -transforms.cB2cA.q;

// rotation vector from relative transform to drive pose
PxVec3 error = log(j.targetPosition * cB2cA.q.getConjugate());
```

## Articulation Projection
----------------------------
当关节中的任何关节分离超过指定的阈值时，关节会自动投影在一起。投影是一个迭代过程，PxArticulation 函数 PxArticulation：：setSeparationTolerance（） 和 PxArticulation：：setMaxProjectionIterations（） 在投影发生时进行控制，并换取成本以获得稳健性。

## Articulations and Sleeping
--------------------------------
像刚性的动态物体一样，如果关节的能量在一段时间内低于某个阈值，关节也会进入睡眠状态。一般来说，睡眠部分中的所有要点也适用于关节。主要区别在于，只有当每个单独的关节链接都满足睡眠标准时，关节才能进入睡眠状态。
