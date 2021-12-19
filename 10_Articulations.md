# Articulations
-------------------
关节(`Articulation`)是单个`Actor`，由一组链接(links)(每个链接的行为都像一个刚体)组成，这些链接通过特殊的关节(`joint`)连接在一起。每个`Articulation`都有一个树状结构 - 所以不能有loop或break裂。它们的主要用途是模拟物理驱动的角色。与标准 `PhysX joints`相比，它们支持更高的质量比、更精确的驱动模型、更好的动态稳定性和更稳健的接头分离恢复能力。但是，它们的模拟成本要高得多。

虽然`Articulation`不直接建立在`joint`上，但它们使用非常相似的配置机制。在本节中，我们假设熟悉`PhysX joints`。

## Creating an Articulation
---------------------------
要创建`Articulation`，请首先创建不带链接的articulation actor：

```C++
PxArticulation* articulation = physics.createArticulation();
```

然后逐个添加链接，每次指定父链接(初始链接的父级为 NULL)和新链接的pose：

```C++
PxArticulationLink* link = articulation->createLink(parent, linkPose);
PxRigidActorExt::createExclusiveShape(*link, linkGeometry, material);
PxRigidBodyExt::updateMassAndInertia(*link, 1.0f);
```

Articulation links具有刚体功能(functionality of rigid bodies)的有限子集。它们可能不是kinematic的，并且不支持damping、velocity clamping或contact force thresholds。休眠状态和求解器迭代计数是整个`Articulation`的属性，而不是单个`link`的属性。

每次创建超出第一个link的其他link时，都会在它与其父级之间创建一个 `PxArticulationJoint` 。为每个joint指定joint frames，其方式与 `PxJoint` 完全相同：

```C++
PxArticulationJoint* joint = link->getInboundJoint();
joint->setParentPose(parentAttachment);
joint->setChildPose(childAttachment);
```

最后，将`Articulation`添加到场景中：

```C++
scene.addArticulation(articulation);
```

## Articulation Joints
------------------------
目前支持的唯一articulation joint形式是anatomical joint，其特性类似于为典型布娃娃配置的D6关节(参见D6关节)。具体而言，关节是spherical joint，具有角驱动、子joint frame的x轴周围的twist limit以及围绕母joint framex轴的椭圆摆动锥极限(elliptical swing cone limit)。这些属性的配置与D6joint 或 spherical joint非常相似，但提供的选项略有不同。

swing limit是hard elliptical cone limit，不支持垂直于极限表面的运动的spring或restitution。您可以按如下方式设置limit ellipse angle：

```C++
joint->setSwingLimit(yAngle, zAngle);
```

用于 y 和 z 周围的limit angles。与 `PxJoint` 锥形limit不同，limit提供了一个tangential spring来限制轴沿limit surface的移动。配置完成后，启用limit surface：

```C++
joint->setSwingLimitEnabled(true);
```

twist limit允许配置上下角：

```C++
joint->setTwistLimit(lower, upper);
```

同样，您必须显式启用它：

```C++
joint->setTwistLimitEnabled(true);
```

与通常的twist limit一样，最好使用足够的限值 `contactDistance` 值，以便求解器在超过极限阈值之前开始强制执行limit。

`Articulation`不可断裂，并且无法检索施加在接头处的约束力。

## Driving an Articulation
----------------------------
`Articulations`通过joint acceleration springs驱动。您可以设置orientation target、angular velocity target以及spring and damping parameters，以控制joint向目标的驱动强度。您还可以设置compliance values，指示joint抵抗加速的强度。接近零的compliance values表示非常强的阻力，compliance values为1表示没有阻力。

`Articulations`分两个阶段驱动。首先施加Articulations(我们使用术语internal forces来表示这些)，然后施加任何外力，例如重力和接触力。您可以在每个阶段的每个joint上提供不同的compliance values。

请注意，对于joint acceleration springs，弹簧所需的强度仅使用由joint连接的两个body的质量来估计。相比之下，`Articulation`的drive springs占`Articulation`中所有body的质量，以及其他joint处致动的任何stiffness 。此估计是一个迭代过程，使用 `PxArticulation` 类的 `externalDriveIterations` 和 `internalDriveIterations` 属性进行控制。

无需为`joint`驱动设置目标四元数，而是可以将方向误差项直接设置为旋转矢量。该值设置为目标四元数的虚部，实部设置为 0。

*joint->setDriveType(PxArticulationJointDriveType::eERROR); joint->setTargetOrientation(PxQuat(error.x, error.y, error.z, 0));*

这允许弹簧以比2个四元数之间的差异产生的更大的位置误差驱动。通过计算目标四元数(target quaternion)、链接帧(link frames)和关节帧(joint frames)的误差，获得与目标四元数相同的行为，如下所示：

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
当`Articulation`中的任何`joint`分离超过指定的阈值时，`Articulation`会自动投影在一起。投影是一个迭代过程， `PxArticulation` 函数 `PxArticulation::setSeparationTolerance()` 和 `PxArticulation::setMaxProjectionIterations() `在投影发生时进行控制，并换取成本以获得稳健性。

## Articulations and Sleeping
--------------------------------
像rigid dynamic objects一样，如果`Articulation`的能量在一段时间内低于某个阈值，关节也会进入睡眠状态。一般来说，Sleeping部分中的所有要点也适用于`Articulation`。主要区别在于，只有当每个单独的articulation link都满足睡眠标准时，`Articulation`才能进入睡眠状态。
