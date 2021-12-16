# Rigid Body Overview

## Introduction

----

本章将介绍使用 `NVIDIA PhysX ` 引擎`simulate`刚体动力学的基础知识。

## Rigid Body Object Model

-------------

`PhysX` 使用分层刚体对象模型，如下所示：

<img src=".\image\RigidBody_01.png" alt="RigidBody_01" style="zoom:150%;" />

| class                | Extends        | Functionality                                                |
| -------------------- | -------------- | ------------------------------------------------------------ |
| *PxBase*             | N/A            | 反射/查询对象类型。                                          |
| *PxActor*            | PxBase         | Actor名称、actor标志、作用域、客户端、聚合、查询世界边界。   |
| *PxRigidActor*       | PxActor        | Shapes 和 transforms                                         |
| *PxRigidBody*        | *PxRigidBody*c | 质量，惯性，body flags                                       |
| *PxRigidStatic*      | PxRigidActor   | 场景中静态主体的接口。这种身体具有隐含的无限质量/惯性。      |
| *PxRigidDynamic*     | PxRigidBody    | 场景中动态刚体的接口。引入对运动学目标（kinematic targets ）和对象休眠（object sleeping）的支持。 |
| *PxArticulationLink* | PxRigidBody    | `PxArticulation`中动态刚体链接的接口。介绍对查询关节和相邻链接的支持。 |
| *PxArticulation*     | PxBase         | 定义`PxArticulation` 的接口。实际上，包含引用多个`PxArticualtionLink`刚体。 |

下图显示了刚体管线中涉及的主要类型之间的关系：

<img src=".\image\RigidBody_02.PNG" alt="RigidBody_02" style="zoom:150%;" />

## The Simulation Loop

现在使用``PxScene::simulate()`方法及时推进世界前进。下面是示例的固定步进器类(fixed stepper class)的简化代码：

```c++
mAccumulator = 0.0f;
mStepSize = 1.0f / 60.0f;

virtual bool advance(PxReal dt)
{
    mAccumulator  += dt;
    if(mAccumulator < mStepSize)
        return false;

    mAccumulator -= mStepSize;

    mScene->simulate(mStepSize);
    return true;
}
```

每当应用完成事件处理并开始空闲时，就会从示例框架中调用此操作。它累积经过的实时时间，直到大于六十分之一秒，然后调用 `simulate()`，它将场景中的所有对象向前移动该间隔时间。这可能是在推进`simulate`时处理时间的众多不同方法中最简单的方法。

要允许`simulate`完成并返回结果，只需调用：

```c++
mScene->fetchResults(true);
```

`True` 表示`simulate`应阻塞，直到`simulate()`完成，以便在返回时保证结果可用。当 `fetchResults `完成时，您定义的任何`simulate`事件回调函数也将被调用。请参阅回调序列一章。在`simulate`过程中，可以从场景中读取和写入。示例利用这一点与物理场并行执行渲染工作。在 `fetchResults()` 返回之前，当前`simulate`步骤的结果不可用。因此，与`simulate`并行运行渲染会将演员渲染为调用 `simulate()`前的样子。在 `fetchResults()` 返回后，所有这些函数都将返回新的`simulate`后的状态。有关在`simulate`运行时读取和写入的更多详细信息，请参阅[*Threading*](Threading.html#threading)一章。为了使人眼将动画运动感知为平滑，每秒至少使用二十个离散帧，每帧对应于一个物理时间步长。要对更复杂的物理场景进行流畅、逼真的`simulate`，请至少每秒使用五十帧。

**注意： 如果您正在进行实时交互式(real-time interactive simulation)`simulate`，则可能会尝试采用不同大小的时间步长，这些步长对应于自上次`simulate`帧以来经过的实时量。如果这样做，请非常小心，与采用恒定大小的时间步长不同的是：`simulate`代码对非常小和很大的时间步长都很敏感，并且对时间步长之间的太大变化也很敏感。在这些情况下，它可能会产生抖动`simulate`（jittery simulation）。**

