# Vehicles
-----------
## Introduction
-------------------
`PhysX` 对载具(Vehicles)的支持在3.x中得到了显着的重新设计。为了取代 `NxWheelShape` 类 2.8.x，已经开发出核心 `PhysX SDK` 和载具仿真代码的更优化集成。更具体地说，载具组件现在以类似于PhysXExtensions的方式位于核心SDK之外。此更改允许在一次pass中更新载具，并推动一个更直观的载具数据建模方法。载具支持已从2.8.x的悬架/车轮/轮胎建模扩展到更完整的模型，该模型结合了模块化载具组件，包括发动机(engine)，离合器(clutch)，齿轮(gears)，自动变速箱(autobox)，差速器(differential)，车轮(wheels)，轮胎(tires)，悬架(suspensions)和底盘(chassis)。快速浏览PxVehicleComponents.h中的数据结构将提供 `PhysX` 载具支持的行为。

## Algorithm
------------
`PhysX Vehicle SDK`将载具建模为弹簧质量(sprung masses)的集合，其中每个弹簧质量(sprung masses)代表一条带有相关车轮和轮胎数据的悬架线(suspension line)。这些弹簧质量的集合具有作为rigid body actor的互补表示(complementary representation)，其质量(mass)、质量中心(center of mass)和惯性矩(moment of inertia)与弹簧质量的质量和坐标完全匹配。如下图所示。

<img src=".\image\Vehicle_1.png" alt="Vehicle_1" style="zoom:100%;" />

图 1a:载具表示为rigid body actor，具有底盘和车轮的shape。请注意，车轮静止偏移(wheel rest offsets)是相对于质心(center of mass)指定的。

<img src=".\image\Vehicle_2.png" alt="Vehicle_2" style="zoom:100%;" />

图 1b:载具表示为质量为 M1 和 M2 的弹簧质量的集合。

弹簧质量和刚体载具表关示之间的系可以用刚体质量中心方程在数学上形式化:

$M = M_1 + M_2$

$X_{cm} = (M_1 x X_1 + M_2 x X_2)/(M_1 + M_2)$

其中 M1 和 M2 是弹簧质量;X1和X2是`Actor`空间中的弹簧质量坐标;M是刚体质量;而 Xcm 是刚体质量偏移的中心。

`PhysX Vehicle SDK` 更新函数(update function)的目的是使用弹簧质量模型计算悬架和轮胎力，然后以修改的速度和角速度的形式将这些力的聚合应用于 `PhysX SDK` 刚体表示。然后，`PhysX SDK` 管理刚体 `Actor` 与其他场景对象的交互和全局pose更新(global pose update )。

每辆车的更新从每个悬架线的raycast开始。raycast从轮胎顶部的正上方以最大压缩开始，并沿着悬架行进的方向向下投射到最大下垂时轮胎底部正下方的位置。如下图所示。

<img src=".\image\Vehicle_3.png" alt="Vehicle_3" style="zoom:100%;" />

图 2:悬挂limit和悬挂raycast。

悬挂力(The suspension force )由每个细长(elongated)或压缩弹簧计算得到，需要将其添加到要施加到刚体上的聚集力(aggregate force)中。此外，suspension force用于计算轮胎承受的载荷。该载荷用于确定将在接触平面中产生的轮胎力，然后添加到要施加到刚体的aggregate force中。轮胎力的计算实际上取决于许多因素，包括转向角(steer angle)、外倾角(camber angle)、摩擦力(friction)、车轮转速(wheel rotation speed)和刚体动量(rigid body momentum)。然后将所有轮胎和悬架力的聚合力应用于与载具关联的rigid body actor，以便在下一次`PhysX SDK`更新中可以相应地修改变换。

除了作为弹簧质量的集合外， `PhysX` 载具还支持各种驱动模型(drive models)。驱动模型的中心是一个扭力离合器(torsion clutch)，它通过离合器两侧转速差异产生的力将车轮和发动机耦合在一起。离合器的一侧是发动机，它直接由油门踏板提供动力。发动机被建模为一个刚体，其运动是纯粹的旋转，并且仅限于单一的旋转自由度。离合器的另一侧是传动系统，差速器和车轮。离合器另一侧的有效转速可以直接从传动比和通过差速器耦合到离合器的车轮的转速来计算。该模型自然允许发动机扭矩传播到车轮，车轮扭矩传播回发动机，就像在标准汽车中一样。

描述 `PhysX vehicle` 每个组件的数据可以在Tuning Guide部分中找到。

## First Code
---------------
### Vehicle SDK Initialization
------------------------------
在使用载具 SDK 之前，必须首先对其进行初始化，以便从各种公差等级(various tolerance scales)中设置一些阈值。这与调用以下函数一样简单:

```C++
PX_C_EXPORT bool PX_CALL_CONV PxInitVehicleSDK
    (PxPhysics& physics, PxSerializationRegistry* serializationRegistry = NULL);
```

此函数应在设置所需的 `PxPhysics` 和 `PxFoundation` 实例后调用。如果需要载具序列化，则需要指定 `PxSerializationRegistry` 实例。可以使用 `PxSerialization::createSerializationRegistry()` 创建 `PxSerializationRegistry()` 实例，请参阅Serialization。

还必须配置载具模拟的基向量(basis vectors)，以便可以明确地计算纵向和横向轮胎打滑(longitudinal and lateral tire slips):

```C++
void PxVehicleSetBasisVectors(const PxVec3& up, const PxVec3& forward);
```

可以在首次执行 `PxVehicleUpdates` 之前的任何时间调用此函数。

与载具关联的rigid body actors可以通过速度修改立即更新，也可以使用在下一个 `PhysX SDK` 模拟调用中应用的加速度进行更新。以下函数可用于选择所需的更新模式:

```C++
void PxVehicleSetUpdateMode(PxVehicleUpdateMode::Enum vehicleUpdateMode);
```

不出所料，载具 SDK 还有一个需要调用的shutdown:

```C++
PX_C_EXPORT void PX_CALL_CONV PxCloseVehicleSDK
    (PxSerializationRegistry* serializationRegistry = NULL);
```

这需要在 `PxPhysics` 实例和 `PxFoundation` 实例发布之前调用;也就是说，关闭顺序与初始化顺序相反。此外，如果需要序列化，则需要将 `PxInitVehicleSDK` 指定的 `PxSerializationRegistry` 传递给 `PxCloseVehicleSDK` 。如果使用载具序列化(vehicle serialization)，则必须在关闭 `PhysXExtensions` 之前调用此序列化。

作为这些函数用法的说明，SnippetVehicle4W 具有以下初始化代码:

```C++
PxInitVehicleSDK(*gPhysics);
PxVehicleSetBasisVectors(PxVec3(0,1,0), PxVec3(0,0,1));
PxVehicleSetUpdateMode(PxVehicleUpdateMode::eVELOCITY_CHANGE);
```

SnippetVehicle4W中的shutdown代码如下:

```C++
PxCloseVehicleSDK();
```

### Introduction To Vehicle Creation
--------------------------------------
以下伪代码说明了设置 `PxVehicleDrive4W` 实例的基本过程:

```C++
const PxU32 numWheels = 4;

PxVehicleWheelsSimData* wheelsSimData = PxVehicleWheelsSimData::allocate(numWheels);
setupWheelsSimulationData(wheelsSimData);

PxVehicleDriveSimData4W driveSimData;
setupDriveSimData(driveSimData);

PxRigidDynamic* vehActor = myPhysics.createRigidDynamic(startPose);
setupVehicleActor(vehActor);
myScene.addActor(*vehActor);

PxVehicleDrive4W* vehDrive4W = PxVehicleDrive4W::allocate(numWheels);
vehDrive4W->setup(physics, veh4WActor, *wheelsSimData, driveSimData, numWheels - 4);
wheelsSimData->free();
```

上面的代码首先实例化了一个 `PxVehicleWheelsSimData` 实例，该实例的内部缓冲区足够大，可以存储四个轮子的配置数据。此配置数据包括悬架强度(suspension strength )和阻尼速率(damping rate)、车轮质量(wheel mass)、轮胎刚度(tire stiffness)和悬架行驶方向(suspension travel direction)等字段。下一步是创建一个 `PxVehicleDriveSimData4W` 实例。此结构存储驱动模型(drive model)的配置，并包括发动机峰值扭矩(engine peak torque)、离合器强度(clutch strength)、传动比(gearing ratios)和阿克曼转向校正(steering correction)等数据字段。在此之后， `PxRigidDynamicActor` 被实例化并配置车轮和底盘的几何`Shape`以及动态属性，如质量，转动惯量和质心。最后一步是实例化 `PxVehicleDrive4W` 实例，并将其与`Actor`和载具配置数据相关联。

功能设置 `WheelsSimulationData` ， `setupDriveSimData` 和 `setatVehicleActor` 实际上非常复杂，并将在将来的章节 setupWheelsSimulationData ， setupDriveSimData 和 setupVehicleActor 中讨论。

### Introduction To Vehicle Update
-----------------------------------
`PhysX Vehicles SDK` 利用批处理场景查询(batched scene queries)来查询每个轮胎下的几何图形。有关 PhysX 批处理场景查询的更详细讨论，请参阅批Batched queries部分。

以下伪代码初始化一个批处理场景查询，其缓冲区对于具有四个车轮的单个载具来说足够大:

```C++
PxRaycastQueryResult sqResults[4];
PxRaycastHit sqHitBuffer[4];
PxBatchQueryDesc sqDesc(4, 0, 0);
sqDesc.queryMemory.userRaycastResultBuffer = sqResults;
sqDesc.queryMemory.userRaycastTouchBuffer = sqHitBuffer;
sqDesc.queryMemory.raycastTouchBufferSize = 4;
sqDesc.preFilterShader = myFilterShader;
PxBatchQuery* batchQuery = scene->createBatchQuery(sqDesc);
```

`PxBatchQuery` 实例通常作为初始化阶段的一部分进行实例化，然后重用每个帧。可以为每个载具实例化一个 `PxBatchQuery` 实例，或者使用足够大的缓冲区来实例化单个 `PxBatchQuery` 实例，以容纳批量载具数组的所有车轮。唯一的限制是在载具仿真帧开始时配置的所有批处理载具数组(batched vehicle arrays)和关联缓冲区(associated buffers)必须一直持续到载具模拟帧结束。

`PhysX vehicles`利用场景查询过滤器着色器(scene query filter shaders)来消除与发出raycast的载具以及任何不被视为可驾驶表面的几何体的相交点。有关如何设置"myFilterShader"的更多详细信息，请参阅下面的"Filtering"部分。

对于仅包含单个四轮载具的batch，可以使用以下伪代码执行suspension raycasts:

```C++
PxVehicleWheels* vehicles[1] = {myVehicle};
PxVehicleSuspensionRaycasts(batchQuery, 1, vehicles, 4, sqResults);
```

`PxVehicleSuspensionRaycasts` 方法为批量载具数组中的所有载具执行suspension raycasts。 `sqResults` 数组中的每个元素都对应于单个suspension的raycast报告。指向 `sqResults` 中连续块的指针由每辆车依次存储，因为该函数在载具数组中进行迭代。这些存储块由每辆车存储，以便它们可以轻松查询 `PxVehicleUpdates` 中的suspension raycasts结果。因此， `sqResults` 数组必须至少持续到 `PxVehicleUpdates` 结束，并且长度必须至少与载具数组中的车轮总数一样大。

载具将使用以下函数调用进行更新:

```C++
PxVehicleUpdates(timestep, gravity, frictionPairs, 1, vehicles, NULL);
```

`PxVehicleUpdates` 方法可更新每辆车的内部动态(internal dynamics)，呈现载具`Actor`的wheel shape，并将速度或加速度更改应用于`Actor`，具体取决于使用 `PxVehicleSetUpdateMode` 选择的更新模式。更多详细信息可以在Wheel Pose部分和Vehicle Update部分中找到。参数 `frictionPairs` 基本上是一个查找表，它将独特的摩擦值与轮胎类型和`PxMaterial`的组合相关联。这里的想法是允许针对每种表面类型调整轮胎响应。这将在Tire Friction on Drivable Surfaces部分进行更深入的讨论。

## Snippets
-----------
当前实现了四个代码段来说明 `PhysX Vehicles SDK` 的操作。这些是:

```C++
1. SnippetVehicle4W
2. SnippetVehicleTank
    3. SnippetNoDrive
3. SnippetVehicleScale
4. SnippetVehicleMultiThreading
    5. SnippetVehicleWheelContactMod
```

每个代码片段都在整个指南中使用。

### SnippetVehicle4W
-------------------
`SnippetVehicle4W` 演示了如何实例化和更新 `PxVehicleDrive4W` 类型的载具。它在plane上创建载具，然后控制载具，以便执行许多精心设计的动作，例如加速，倒车，制动，手刹和转弯。

### SnippetVehicleTank
-----------------------
`SnippetVehicleTank` 演示了如何实例化和更新 `PxVehicleDriveTank` 类型的载具。它在plane上创建一个坦克，然后控制坦克，以便它执行许多精心设计的机动，如加速，倒车，软转弯和硬转弯。

### SnippetVehicleNoDrive
-------------------------
`SnippetVehicleNoDrive` 演示了如何实例化和更新 `PxVehicleNoDrive` 类型的载具。它在plane上创建载具，然后控制载具，以便它执行许多精心设计的操作，例如加速，倒车，软转弯和硬转弯。

### SnippetVehicleScale
-----------------------
`SnippetVehicleScale` 演示了当米不是所选的长度刻度时如何配置 `PhysX vehicle` 。该代码段设置一个载具，其中采用米作为长度刻度，然后修改载具参数，这样新的载具仍然是同一载具，但以厘米作为所选的长度刻度。

### SnippetVehicleMultiThreading
------------------------------------
`SnippetVehicleMultiThreading` 演示了如何实现多线程载具。它在平面上创建多个载具，然后跨多个线程并行模拟它们。

## Advanced Concepts
--------------------
### Vehicle Creation
--------------------
本节讨论载具仿真数据的配置，并介绍如何在 PhysX SDK 中设置表示载具的`Actor`。载具创建简介部分确定了载具配置的三个不同阶段:车轮仿真数据的配置、驱动仿真数据的配置和`Actor`配置。这些阶段中的每一个都依次讨论。

#### setupWheelsSimulationData
--------------------------------
以下代码取自 `SnippetVehicle4W` ，实例化了 `PxVehicleWheelsSimData`:

```C++
void setupWheelsSimulationData(const PxF32 wheelMass, const PxF32 wheelMOI,
    const PxF32 wheelRadius, const PxF32 wheelWidth, const PxU32 numWheels,
    const PxVec3* wheelCenterActorOffsets, const PxVec3& chassisCMOffset,
    const PxF32 chassisMass, PxVehicleWheelsSimData* wheelsSimData)
{
    //Set up the wheels.
    PxVehicleWheelData wheels[PX_MAX_NB_WHEELS];
    {
        //Set up the wheel data structures with mass, moi, radius, width.
        for(PxU32 i = 0; i < numWheels; i++)
        {
            wheels[i].mMass = wheelMass;
            wheels[i].mMOI = wheelMOI;
            wheels[i].mRadius = wheelRadius;
            wheels[i].mWidth = wheelWidth;
        }

        //Enable the handbrake for the rear wheels only.
        wheels[PxVehicleDrive4WWheelOrder::eREAR_LEFT].mMaxHandBrakeTorque=4000.0f;
        wheels[PxVehicleDrive4WWheelOrder::eREAR_RIGHT].mMaxHandBrakeTorque=4000.0f;
        //Enable steering for the front wheels only.
        wheels[PxVehicleDrive4WWheelOrder::eFRONT_LEFT].mMaxSteer=PxPi*0.3333f;
        wheels[PxVehicleDrive4WWheelOrder::eFRONT_RIGHT].mMaxSteer=PxPi*0.3333f;
    }

    //Set up the tires.
    PxVehicleTireData tires[PX_MAX_NB_WHEELS];
    {
        //Set up the tires.
        for(PxU32 i = 0; i < numWheels; i++)
        {
            tires[i].mType = TIRE_TYPE_NORMAL;
        }
    }

    //Set up the suspensions
    PxVehicleSuspensionData suspensions[PX_MAX_NB_WHEELS];
    {
        //Compute the mass supported by each suspension spring.
        PxF32 suspSprungMasses[PX_MAX_NB_WHEELS];
        PxVehicleComputeSprungMasses
            (numWheels, wheelCenterActorOffsets,
             chassisCMOffset, chassisMass, 1, suspSprungMasses);

        //Set the suspension data.
        for(PxU32 i = 0; i < numWheels; i++)
        {
            suspensions[i].mMaxCompression = 0.3f;
            suspensions[i].mMaxDroop = 0.1f;
            suspensions[i].mSpringStrength = 35000.0f;
            suspensions[i].mSpringDamperRate = 4500.0f;
            suspensions[i].mSprungMass = suspSprungMasses[i];
        }

        //Set the camber angles.
        const PxF32 camberAngleAtRest=0.0;
        const PxF32 camberAngleAtMaxDroop=0.01f;
        const PxF32 camberAngleAtMaxCompression=-0.01f;
        for(PxU32 i = 0; i < numWheels; i+=2)
        {
            suspensions[i + 0].mCamberAtRest =  camberAngleAtRest;
            suspensions[i + 1].mCamberAtRest =  -camberAngleAtRest;
            suspensions[i + 0].mCamberAtMaxDroop = camberAngleAtMaxDroop;
            suspensions[i + 1].mCamberAtMaxDroop = -camberAngleAtMaxDroop;
            suspensions[i + 0].mCamberAtMaxCompression = camberAngleAtMaxCompression;
            suspensions[i + 1].mCamberAtMaxCompression = -camberAngleAtMaxCompression;
        }
    }

    //Set up the wheel geometry.
    PxVec3 suspTravelDirections[PX_MAX_NB_WHEELS];
    PxVec3 wheelCentreCMOffsets[PX_MAX_NB_WHEELS];
    PxVec3 suspForceAppCMOffsets[PX_MAX_NB_WHEELS];
    PxVec3 tireForceAppCMOffsets[PX_MAX_NB_WHEELS];
    {
        //Set the geometry data.
        for(PxU32 i = 0; i < numWheels; i++)
        {
            //Vertical suspension travel.
            suspTravelDirections[i] = PxVec3(0,-1,0);

            //Wheel center offset is offset from rigid body center of mass.
            wheelCentreCMOffsets[i] =
                wheelCenterActorOffsets[i] - chassisCMOffset;

            //Suspension force application point 0.3 metres below
            //rigid body center of mass.
            suspForceAppCMOffsets[i] =
                PxVec3(wheelCentreCMOffsets[i].x,-0.3f,wheelCentreCMOffsets[i].z);

            //Tire force application point 0.3 metres below
            //rigid body center of mass.
            tireForceAppCMOffsets[i] =
                PxVec3(wheelCentreCMOffsets[i].x,-0.3f,wheelCentreCMOffsets[i].z);
        }
    }

    //Set up the filter data of the raycast that will be issued by each suspension.
    PxFilterData qryFilterData;
    setupNonDrivableSurface(qryFilterData);

    //Set the wheel, tire and suspension data.
    //Set the geometry data.
    //Set the query filter data
    for(PxU32 i = 0; i < numWheels; i++)
    {
        wheelsSimData->setWheelData(i, wheels[i]);
        wheelsSimData->setTireData(i, tires[i]);
        wheelsSimData->setSuspensionData(i, suspensions[i]);
        wheelsSimData->setSuspTravelDirection(i, suspTravelDirections[i]);
        wheelsSimData->setWheelCentreOffset(i, wheelCentreCMOffsets[i]);
        wheelsSimData->setSuspForceAppPointOffset(i, suspForceAppCMOffsets[i]);
        wheelsSimData->setTireForceAppPointOffset(i, tireForceAppCMOffsets[i]);
        wheelsSimData->setSceneQueryFilterData(i, qryFilterData);
        wheelsSimData->setWheelShapeMapping(i, i);
    }
}
```

`PxVehicleComputeSprungMasses` 函数计算每个悬架的弹簧质量，以便它们共同匹配刚体质量中心(rigid body center of mass)。这是在`Actor`的帧中执行的。在`Actor`的帧中执行 `PxVehicleComputeSprungMass` 是有意义的，因为刚体质量中心总是在`Actor`的帧中指定。另一方面，载具悬架系统在质心帧中指定。因此，函数 `setWheelCentreOffset` 、 `setSuspForceAppPointOffset` 和 `setTireForceAppPointOffset` 都描述了刚体质心的偏移量。这种方法的直接性可以使刚体质心的变化比预期的要复杂一些。为了解决这个问题，引入了 `PxVehicleUpdateCMassLocalPose` 函数，尽管在上面的代码中没有使用。此功能重新计算和设置所有悬架偏移，重新计算弹簧质量，并以保留每个弹簧的固有频率和阻尼比的方式设置它们。

上述许多参数和功能的详细信息可以在Tuning Guide部分中找到。功能设置 `NonDrivableSurface` 为每个悬架raycast设置场景查询过滤器(scene query filter)数据，将在Filtering部分中更详细地讨论。此外， `TIRE_TYPE_NORMAL` 与轮胎摩擦之间的联系应在 Tire Friction on Drivable Surfaces部分中明确。最后，在Wheel Pose部分中应阐明方法 `WheelShapeMappping` 的使用。

#### setupDriveSimData
-------------------------
以下代码取自 SnippetVehicle4W ，实例化了 `PxVehicleDriveSimData4W`:

```C++
PxVehicleDriveSimData4W driveSimData;
{
    //Diff
    PxVehicleDifferential4WData diff;
    diff.mType=PxVehicleDifferential4WData::eDIFF_TYPE_LS_4WD;
    driveSimData.setDiffData(diff);

    //Engine
    PxVehicleEngineData engine;
    engine.mPeakTorque=500.0f;
    engine.mMaxOmega=600.0f;//approx 6000 rpm
    driveSimData.setEngineData(engine);

    //Gears
    PxVehicleGearsData gears;
    gears.mSwitchTime=0.5f;
    driveSimData.setGearsData(gears);

    //Clutch
    PxVehicleClutchData clutch;
    clutch.mStrength=10.0f;
    driveSimData.setClutchData(clutch);

    //Ackermann steer accuracy
    PxVehicleAckermannGeometryData ackermann;
    ackermann.mAccuracy=1.0f;
    ackermann.mAxleSeparation=
        wheelsSimData->getWheelCentreOffset(PxVehicleDrive4WWheelOrder::eFRONT_LEFT).z-
        wheelsSimData->getWheelCentreOffset(PxVehicleDrive4WWheelOrder::eREAR_LEFT).z;
    ackermann.mFrontWidth=
        wheelsSimData->getWheelCentreOffset(PxVehicleDrive4WWheelOrder::eFRONT_RIGHT).x-
        wheelsSimData->getWheelCentreOffset(PxVehicleDrive4WWheelOrder::eFRONT_LEFT).x;
    ackermann.mRearWidth=
        wheelsSimData->getWheelCentreOffset(PxVehicleDrive4WWheelOrder::eREAR_RIGHT).x-
        wheelsSimData->getWheelCentreOffset(PxVehicleDrive4WWheelOrder::eREAR_LEFT).x;
    driveSimData.setAckermannGeometryData(ackermann);
}
```

上述许多参数和功能的详细信息可以在Tuning Guide部分中找到。

配置 `PxVehicleDriveSimDataNW` 和 `PxVehicleDriveSimDataTank` 实例遵循非常相似的过程，尽管组件略有不同。例如，可以在 `SnippetVehicleTank` 中找到更多详细信息。

#### setupVehicleActor
---------------------
以下代码(所有vehicle snippets通用)设置了一个具有几何图形、filter和dynamics data的`rigid dynamic Actor`:

```C++
PxRigidDynamic* createVehicleActor
(const PxVehicleChassisData& chassisData,
 PxMaterial** wheelMaterials, PxConvexMesh** wheelConvexMeshes, const PxU32 numWheels, const PxFilterData& wheelSimFilterData,
 PxMaterial** chassisMaterials, PxConvexMesh** chassisConvexMeshes, const PxU32 numChassisMeshes, const PxFilterData& chassisSimFilterData,
 PxPhysics& physics)
{
        //We need a rigid body actor for the vehicle.
        //Don't forget to add the actor to the scene after setting up the associated vehicle.
        PxRigidDynamic* vehActor = physics.createRigidDynamic(PxTransform(PxIdentity));

        //Wheel and chassis query filter data.
        //Optional: cars don't drive on other cars.
        PxFilterData wheelQryFilterData;
        setupNonDrivableSurface(wheelQryFilterData);
        PxFilterData chassisQryFilterData;
        setupNonDrivableSurface(chassisQryFilterData);

        //Add all the wheel shapes to the actor.
        for(PxU32 i = 0; i < numWheels; i++)
        {
                PxConvexMeshGeometry geom(wheelConvexMeshes[i]);
                PxShape* wheelShape=PxRigidActorExt::createExclusiveShape(*vehActor, geom, *wheelMaterials[i]);
                wheelShape->setQueryFilterData(wheelQryFilterData);
                wheelShape->setSimulationFilterData(wheelSimFilterData);
                wheelShape->setLocalPose(PxTransform(PxIdentity));
        }

        //Add the chassis shapes to the actor.
        for(PxU32 i = 0; i < numChassisMeshes; i++)
        {
                PxShape* chassisShape=PxRigidActorExt::createExclusiveShape(*vehActor, PxConvexMeshGeometry(chassisConvexMeshes[i]), *chassisMaterials[i]);
                chassisShape->setQueryFilterData(chassisQryFilterData);
                chassisShape->setSimulationFilterData(chassisSimFilterData);
                chassisShape->setLocalPose(PxTransform(PxIdentity));
        }

        vehActor->setMass(chassisData.mMass);
        vehActor->setMassSpaceInertiaTensor(chassisData.mMOI);
        vehActor->setCMassLocalPose(PxTransform(chassisData.mCMOffset,PxQuat(PxIdentity)));

        return vehActor;
}
```

`wheelSimFilterData` 、 `chassisSimFilterData` 、 `wheelQryFilterData` 和 `chassisQryFilterData` 的重要性将在Filtering部分讨论。此外，上述代码中wheel shape的orderin与 `PxVehicleWheelsSimData::setWheelShapeMapping` 函数之间的联系在 Wheel Pose 部分中得到了阐述。

### Filtering
---------------
本节将描述vehicle query和vehicle simulation filtering背后的概念。

载具scene query和simulation filtering的主要目标是确保载具受到悬架弹簧力(suspension spring forces)的支撑，而不会受到wheel shape相交的干扰。过滤的要求如下:

```C++
1. wheel shapes must not hit drivable surfaces
2. suspension raycasts can hit drivable surfaces
3. suspension raycasts must not hit the shapes of the vehicle issuing the raycasts
```

通过仿真过滤(simulation filtering)，可以确保wheel shape不会撞到可驾驶表面。这在 Collision Filtering部分有更详细的讨论。载具代码段使用以下simulation filter shader:

```C++
PxFilterFlags VehicleFilterShader
(PxFilterObjectAttributes attributes0, PxFilterData filterData0,
 PxFilterObjectAttributes attributes1, PxFilterData filterData1,
 PxPairFlags& pairFlags, const void* constantBlock, PxU32 constantBlockSize)
{
        PX_UNUSED(attributes0);
        PX_UNUSED(attributes1);
        PX_UNUSED(constantBlock);
        PX_UNUSED(constantBlockSize);

        if( (0 == (filterData0.word0 & filterData1.word1)) && (0 == (filterData1.word0 & filterData0.word1)) )
                return PxFilterFlag::eSUPPRESS;

        pairFlags = PxPairFlag::eCONTACT_DEFAULT;
        pairFlags |= PxPairFlags(PxU16(filterData0.word2 | filterData1.word2));

        return PxFilterFlags();
}
```

这些代码段还将仿真筛选器数据(simulation filter data)应用于wheel shape，如下所示:

```C++
PxFilterData wheelSimFilterData;
wheelSimFilterData.word0 = COLLISION_FLAG_WHEEL;
wheelSimFilterData.word1 = COLLISION_FLAG_WHEEL_AGAINST;

    ...

wheelShape->setSimulationFilterData(wheelSimFilterData);
```

最后，将以下仿真滤波器数据(simulation filter data)应用于可驾驶表面:

```C++
PxFilterData simFilterData;
simFilterData.word0 = COLLISION_FLAG_GROUND;
simFilterData.word1 = COLLISION_FLAG_GROUND_AGAINST;

...

shapes[0]->setSimulationFilterData(simFilterData);
```

碰撞标志( `COLLISION_FLAG_WHEEL` 、 `COLLISION_FLAG_GROUND_AGAINST` 等)和filter shader的组合可确保wheel shape不会与可驾驶表面发生冲突。

可以采用非常相似的过程来配置补充场景查询过滤器(complementary scene query filters)。这是在载具代码段中使用以下代码完成的:

```C++
void setupDrivableSurface(PxFilterData& filterData)
{
    filterData.word3 = (PxU32)DRIVABLE_SURFACE;
}

void setupNonDrivableSurface(PxFilterData& filterData)
{
    filterData.word3 = UNDRIVABLE_SURFACE;
}

PxQueryHitType::Enum WheelRaycastPreFilter
(PxFilterData filterData0, PxFilterData filterData1,
 const void* constantBlock, PxU32 constantBlockSize,
 PxHitFlags& queryFlags)
{
    //filterData0 is the vehicle suspension raycast.
    //filterData1 is the shape potentially hit by the raycast.
    PX_UNUSED(constantBlockSize);
    PX_UNUSED(constantBlock);
    PX_UNUSED(filterData0);
    PX_UNUSED(queryFlags);
    return ((0 == (filterData1.word3 & DRIVABLE_SURFACE)) ?
        PxQueryHitType::eNONE : PxQueryHitType::eBLOCK);
}
```

每个车轮都获得使用设置配置 `NonDrivableSurface` 的filter数据，并通过以下方式传递给载具:

```C++
wheelsSimData->setSceneQueryFilterData(i, qryFilterData);
```

`WheelRaycastPreFilter` 中的参数 `filterData0` 对应于使用 `PxVehiceWheelsSimData::setSceneQueryFilterData` 传递给载具的参数 `qryFilterData` 。另一方面，参数 `filterData1` 对应于可能被raycast击中的`Shape`的查询筛选器数据(query filter data)。在vehicle snippets中，drivable ground plane的`Shape`具有场景查询过滤器数据配置了方法 `setupDrivableSurface` 。这满足了`suspension raycasts`可以击中可驾驶表面的要求。另一方面，载具`Shape`配置了`setupNonDrivableSurface`。这满足了suspension raycasts 不得撞击发出raycasts信号的载具的限制，但也防止载具驾驶可能添加到场景中的任何其他载具。通过采用更复杂的filter shader，可以很容易地避免这种额外的限制，该着色器可能利用`Shape`筛选器数据(shape filter data)和应用于查询本身的筛选器数据中编码的唯一 ID。但是，必须注意配置filter，以确保suspension raycasts仅与其他载具的`Shape`相互作用。

**至关重要的是，如果允许过滤器数据对(filter data pair)进行raycast命中，则 `WheelRaycastPreFilter` 返回 `PxQueryHitType::eBLOCK`。使用 `PxQueryHitType::eBLOCK` 可以保证每个raycast都不会返回任何命中，或者只是返回最接近raycast起点的命中。这很重要，因为 `PxVehicleSuspensionRaycasts` 和 `PxVehicleUpdates` 期望每个轮子与传递给批处理查询的 `PxRaycastQueryResult` 和 `PxRaycastHit` 数组中的每个元素之间有一对一的对应关系。**

### Tire Friction on Drivable Surfaces
----------------------------------------
在本节中，将讨论设置轮胎类型(setting up tire types)，可驾驶表面类型(drivable surface types)以及轮胎组合上的轮胎摩擦(tire friction on combinations of tire)和表面类型(surface type)。

要为轮胎类型和表面类型的每种组合实现唯一的摩擦值，首先需要将轮胎类型分配给轮胎。在setupWheelsSimulationData部分中，为每个轮胎分配了轮胎类型:

```C++
//Set up the tires.
PxVehicleTireData tires[PX_MAX_NB_WHEELS];
{
    //Set up the tires.
    for(PxU32 i = 0; i < numWheels; i++)
    {
        tires[i].mType = TIRE_TYPE_NORMAL;
    }
}
```

为每个曲面指定一个类型稍微复杂一些。基本思想是，每次suspension raycast命中都会返回raycast所击中`Shape`的 `PxMaterial` 。通过了解PxMaterial数组，可以将命中表面的类型与 `PxMaterial` 数组元素的索引相关联，该索引与raycast击中的材料相匹配。此查找和摩擦值表由类 `PxVehicleDrivableSurfaceToTireFrictionPairs` 管理。为了使该功能更通用， `PxMaterial` 数组的每个元素实际上都与 `PxVehicleDrivableSurfaceType` 实例相关联。这允许多个 `PxMaterial` 实例共享相同的曲面类型。

在vehicle snippets中，以下代码在 `PxMaterial` 和表面类型之间进行关联，然后将轮胎和表面类型的每个组合与摩擦值相关联:

```C++
PxVehicleDrivableSurfaceToTireFrictionPairs* createFrictionPairs
    (const PxMaterial* defaultMaterial)
{
    PxVehicleDrivableSurfaceType surfaceTypes[1];
    surfaceTypes[0].mType = SURFACE_TYPE_TARMAC;

    PxMaterial* surfaceMaterials[1];
    surfaceMaterials[0] = defaultMaterial;

    PxVehicleDrivableSurfaceToTireFrictionPairs* surfaceTirePairs =
        PxVehicleDrivableSurfaceToTireFrictionPairs::allocate(MAX_NUM_TIRE_TYPES,
        MAX_NUM_SURFACE_TYPES);

    surfaceTirePairs->setup(MAX_NUM_TIRE_TYPES, MAX_NUM_SURFACE_TYPES,
        surfaceMaterials, surfaceTypes);

    for(PxU32 i = 0; i < MAX_NUM_SURFACE_TYPES; i++)
    {
        for(PxU32 j = 0; j < MAX_NUM_TIRE_TYPES; j++)
        {
            surfaceTirePairs->setTypePairFriction(i,j,gTireFrictionMultipliers[i][j]);
        }
    }
    return surfaceTirePairs;
}
```

**没有必要提供所有材料的详尽数组。如果 `PxVehicleDrivableSurfaceToTireFrictionPairs` 不知道命中材料，则假定表面类型的值为零。**

`PhysX vehicles SDK`中使用的摩擦值没有上限。虽然遵守物理定律的最大摩擦力值为1.0，但`PhysX vehicles SDK`故意不强制执行此规则。其中一个原因是，载具模型远非真实载具的完整描述，这意味着需要对摩擦值采取一些自由度才能产生所需的行为。在给定一组特定的载具参数的情况下，更完整的模型肯定会提供更高的精度，但目前还不清楚它是否会提供更大范围的可编辑和可控行为，或者是否具有游戏所需的性能特征。摩擦力不被钳制在1.0的另一个原因是游戏通常以60Hz模拟物理更新。这是以牺牲数值精度为代价的，特别是当存在许多需要KHz更新频率的瞬态轮胎效应时。数值精度的一个来源是悬架的振荡幅度，其依次取决于载具在每次更新之间在重力作用下的距离。在KHz更新频率下，这种仿真伪像小是可以接受的，但在60Hz时却不是。最后一个原因是，根本不需要对`vehicles SDK`施加严格的摩擦规则。这可以产生有趣的行为，当受到刚体和轮胎动力学定律的约束时，这些行为是不可能的。然而，话虽如此，在60Hz下模拟的实现模型应该具有足够的完整性，只需要在1.0以上进行微小的调整。如果需要非常大的摩擦值，例如大于2.0，则可能是更新顺序有问题，或者可能使用了非常非物理的载具数据。

`PxVehicleDrivableSurfaceToTireFrictionPairs` 实例作为每次调用 `PxVehicleUpdates` `的函数参数传递。PxVehicleDrivableSurfaceToTireFriction` 的每个实例只需在 `PxVehicleUpdates` 期间持续存在。在调用 `PxVehicleUpdates` 之间编辑轮胎类型，材料和摩擦值是完全合法的。在 `PxVehicleUpdates` 仍在执行时编辑这些值中的任何一个都将导致未定义的行为。

### Vehicle Controls
------------------------
在本节中，应讨论用于驾驶载具的控制值设置。

设置载具控制值(set vehicle control values)的最简单、最直接的方法是使用以下函数:

```C++
void PxVehicleDriveDynData::setAnalogInput(const PxReal analogVal, const PxU32 type);
```

游戏中载具动力学(vehicle dynamics)的困难之一是知道如何以一种令人愉悦的处理方式过滤(filter)原始控制器数据。例如，玩家经常通过非常快速地按下油门扳机来加速，这在真正的汽车中永远不会发生。这种快速加速可能会产生适得其反的效果，因为由此产生的车轮旋转会减少轮胎可能产生的横向和纵向力。为了帮助克服其中一些问题，提供了一些可选代码来筛选键盘和游戏手柄中的控制数据。

过滤控制器输入数据(filtering controller input data)问题的一个解决方案是为每个button或pad分配上升和下降速率。对于数字控制(digital control)下的模拟值，可以简单地以指定的速率增加或减少模拟值，具体取决于数字输入是打开还是关闭。对于模拟控制(analog control )下的模拟值，以指定的速率从以前的输入值混合(blend)到当前输入更有意义。这个简单模型的一个稍微复杂之处在于，还必须对在大速度下实现大转向角的难度进行建模。实现这一目标的一种技术是对轮胎对准力矩的力进行建模，并将其应用于转向连杆模型。这听起来相当复杂，很难调整。更简单的解决方案可能是将过滤后的转向值缩放为范围 (0，1) 中另一个在高速下减小的值。此更简单的方法已在帮助程序类和函数中实现。

数字和模拟控制的上升和下降率已在SnippetVehicle4W中硬编码:

```C++
PxVehicleKeySmoothingData gKeySmoothingData=
{
    {
        3.0f,    //rise rate eANALOG_INPUT_ACCEL
        3.0f,    //rise rate eANALOG_INPUT_BRAKE
        10.0f,    //rise rate eANALOG_INPUT_HANDBRAKE
        2.5f,    //rise rate eANALOG_INPUT_STEER_LEFT
        2.5f,    //rise rate eANALOG_INPUT_STEER_RIGHT
    },
    {
        5.0f,    //fall rate eANALOG_INPUT__ACCEL
        5.0f,    //fall rate eANALOG_INPUT__BRAKE
        10.0f,    //fall rate eANALOG_INPUT__HANDBRAKE
        5.0f,    //fall rate eANALOG_INPUT_STEER_LEFT
        5.0f    //fall rate eANALOG_INPUT_STEER_RIGHT
    }
};

PxVehiclePadSmoothingData gPadSmoothingData=
{
    {
        6.0f,    //rise rate eANALOG_INPUT_ACCEL
        6.0f,    //rise rate eANALOG_INPUT_BRAKE
        12.0f,    //rise rate eANALOG_INPUT_HANDBRAKE
        2.5f,    //rise rate eANALOG_INPUT_STEER_LEFT
        2.5f,    //rise rate eANALOG_INPUT_STEER_RIGHT
    },
    {
        10.0f,    //fall rate eANALOG_INPUT_ACCEL
        10.0f,    //fall rate eANALOG_INPUT_BRAKE
        12.0f,    //fall rate eANALOG_INPUT_HANDBRAKE
        5.0f,    //fall rate eANALOG_INPUT_STEER_LEFT
        5.0f    //fall rate eANALOG_INPUT_STEER_RIGHT
    }
};
```

还指定了一个查找表，以将最大转向描述为速度的函数:

```C++
PxF32 gSteerVsForwardSpeedData[2*8]=
{
    0.0f,        0.75f,
    5.0f,        0.75f,
    30.0f,        0.125f,
    120.0f,        0.1f,
    PX_MAX_F32, PX_MAX_F32,
    PX_MAX_F32, PX_MAX_F32,
    PX_MAX_F32, PX_MAX_F32,
    PX_MAX_F32, PX_MAX_F32
};
PxFixedSizeLookupTable<8> gSteerVsForwardSpeedTable(gSteerVsForwardSpeedData,4);
```

使用 `PxVehicleDrive4WRawInputData` 实例，在使用键盘时记录用户输入非常简单:

```C++
gVehicleInputData.setDigitalAccel(true);
gVehicleInputData.setDigitalBrake(true);
gVehicleInputData.setDigitalHandbrake(true);
gVehicleInputData.setDigitalSteerLeft(true);
gVehicleInputData.setDigitalSteerRight(true);
gVehicleInputData.setGearUp(true);
gVehicleInputData.setGearDown(true);
```

或者在使用游戏手柄的情况下:

```C++
gVehicleInputData.setAnalogAccel(1.0f);
gVehicleInputData.setAnalogBrake(1.0f);
gVehicleInputData.setAnalogHandbrake(1.0f);
gVehicleInputData.setAnalogSteer(1.0f);
gVehicleInputData.setGearUp(1.0f);
gVehicleInputData.setGearDown(1.0f);
```

在这里， `gVehicleInputData` 是载具SDK帮助器类 `PxVehicleDrive4WRawInputData` 的一个实例。

`vehicle SDK` 提供两个可选功能，用于平滑键盘或游戏手柄数据，并将平滑的输入值应用于 `PhysX` 载具。如果载具由数字输入控制，则使用以下功能:

```C++
PxVehicleDrive4WSmoothDigitalRawInputsAndSetAnalogInputs(gKeySmoothingData,
    gSteerVsForwardSpeedTable, carRawInputs,timestep,isInAir,(PxVehicleDrive4W&)focusVehicle);
```

而游戏手柄控制器使用以下代码:

```C++
PxVehicleDrive4WSmoothAnalogRawInputsAndSetAnalogInputs(gCarPadSmoothingData,
    gSteerVsForwardSpeedTable, carRawInputs,timestep,(PxVehicleDrive4W&)focusVehicle);
```

上面的代码平滑了控制器输入，并将其应用于 `PxVehicleDrive4W` 实例。对于其他载具类型，该过程非常相似，除了为每种载具类型设计的补充类和功能。

### Vehicle Update
-----------------------
已经提到载具分两个阶段更新(Update):

+ 特定的载具代码，用于更新载具internal dynamics并计算forces/torques以应用于载具的刚体表示
+ SDK 更新，用于说明施加的forces/torques以及与其他场景实体的碰撞。

在载具更新简介部分中，介绍了用于执行raycast和载具更新的功能。在本节中，将更详细地讨论这些单独的更新阶段。

#### Raycast and Update Ordering
---------------------------------
在首次在 `PxVehicleUpdates` 中更新载具之前，它必须已经使用 `PxVehicleSuspensionRaycasts` 至少执行了一次suspension line raycasts。在随后的更新中，严格来说没有必要发出新的raycast，因为每辆车都会缓存可以重复使用的raycast命中平面。建议在每辆车的raycast完成(completion)和更新(updates)之间有一对一的对应关系，但只需要低细节水平的载具的情况除外。这可能包括远离相机的汽车，或者已知载具在具有高空间相干性的几何图形上行驶的汽车。对仅需要较低详细级别的载具的支持将在Level of Detail部分中讨论。

相对于载具动态更新，raycasts的发布顺序有一定的自由度。在实际情况下，raycast可能会在更新循环结束时在单独的线程上发出，以便它们为下一个更新循环的开始做好准备。但是，这实际上完全取决于线程环境和刚体更新的顺序。

#### Wheel Pose
----------------
`PxVehicleUpdates` 会将载具`Actor`的wheel shape呈现出来，以考虑转向(steer)、外倾角(camber)和旋转角度(rotation angles)。计算出的pose还尝试将车轮几何`Shape`精确地放置在沿suspension line发出的raycast所标识的接触平面上。要执行此功能，`PhysX Vehicles SDK` 需要知道 `Actor` 的哪些`Shape`对应于载具的每个车轮。这是通过 `PxVehicleWheelsSimData::setWheelShapeMapping` 功能实现的。

**`vehicle SDK` 为每个车轮提供一个默认映射，该映射等效于 `PxVehicleWheelsSimData::setWheelShapeMapping(i，i)`。如果`Shape`的布局与默认模式不同，则需要更正此问题。**

**可以调用 `PxVehicleWheelsSimData::setWheelShapeMapping(i，-1)` 来禁用设置本地车轮pose。如果轮子没有相应的actor geometry，则此功能特别有用。**

Wheel Pose始终在 `PxVehicleSuspensionData::mMaxDroop` 和 `PxVehicleSuspensionData::mMaxCompression`施加的`limit`内。如果suspension raycast命中平面要求将车轮放置在超过compression limit的位置，则车轮将被放置在compression limit处，并且刚体约束将在下一个`SDK simulate(`)调用中处理差异。

#### Vehicle State Queries
--------------------------
每辆车都存储每次调用 `PxVehicleUpdate` 时更新的持久性仿真数据(persistent simulation data)。持久性数据的示例包括车轮转速(wheel rotation speeds)、车轮旋转角度(wheel rotation angle)和车轮旋转速度(wheel rotation speed)。此外，在每次更新期间都会计算大量非持久性数据。这种非持久性数据不会存储在载具自身的数据结构中。相反，数据缓冲区被传递给 `PxVehicleUpdates` ，并在 `PxVehicleUpdates` 完成后进行查询。非持久性数据的示例包括悬架抖动(suspension jounce)、轮胎力(tire force)和raycast hit actor。这两种数据类型的组合几乎可以完整地拍摄载具状态，并可用于触发二次效果，例如打滑痕迹(raycast hit actor)，发动机和离合器音频( engine and clutch audio)以及烟雾颗粒(smoke particles)。

持久性车轮数据存储在 `PxVehicleWheelsDynData` 中，而持久性驱动器模型数据(drive model data)存储在 `PxVehicleDriveDynData` 中。最有用的功能如下:

```C++
PX_FORCE_INLINE PxReal PxVehicleDriveDynData::getEngineRotationSpeed() const;
PxReal PxVehicleWheelsDynData::getWheelRotationSpeed(const PxU32 wheelIdx) const;
PxReal PxVehicleWheelsDynData::getWheelRotationAngle(const PxU32 wheelIdx) const;
```

要记录非持久性模拟数据以便以后可以查询，必须将额外的函数参数传递给 `PxVehicleUpdates` 。以下伪代码记录单个四轮汽车的非持久性数据:

```C++
PxWheelQueryResult wheelQueryResults[4];
PxVehicleWheelQueryResult vehicleWheelQueryResults[1] = {{wheelQueryResults, 4}};
PxVehicleUpdates(timestep, gravity, frictionPairs, 1, vehicles, vehicleWheelQueryResults);
```

在这里，一个 `PxVehicleWheelQueryResult` 数组(其长度至少等于批处理载具数组中的载具数量)被传递给 `PxVehicleUpdates` 。每个 `PxVehicleWheelQueryResult` 实例都有一个指向 `PxWheelQueryResult` 缓冲区的指针，其长度至少等于载具中的车轮数。在 `PxVehicleUpdates` 完成后，可以检查每个车轮的状态。

没有义务记录非持久性数据以供以后查询。事实上，将载具与NULL数据块相关联是完全合法的，以避免存储非持久性车轮数据。此功能允许内存预算针对最感兴趣的载具。

## More Advanced Concepts
----------------------------
### Vehicle Telemetry
---------------------
遥测数据(telemetry data)的目的是公开汽车的内部动态(inner dynamics)，并通过使用遥测图(telemetry graphs)来帮助处理调整。在本节中，将讨论telemetry data的初始化(initialization)、收集(collection)和呈现(rendering)。

通过调用以下函数来记录telemetry data:

```C++
void PxVehicleUpdateSingleVehicleAndStoreTelemetryData
    (const PxReal timestep, const PxVec3& gravity,
     const PxVehicleDrivableSurfaceToTireFrictionPairs& vehicleDrivableSurfaceToTireFrictionPairs,
     PxVehicleWheels* focusVehicle, PxVehicleWheelQueryResult* vehicleWheelQueryResults,
     PxVehicleTelemetryData& telemetryData);
```

上面的函数与 `PxVehicleUpdates` 相同，只是它一次只能更新一辆车，并采用额外的函数参数 `telemetryData` 。

设置telemetry data相对简单。除了存储telemetry data流外， `PxVehicleTelemetryData` 结构还存储描述图形大小、位置和配色方案的数据。以下伪代码初始化和配置四轮载具的telemetry data:

```C++
PxVehicleTelemetryData* myTelemetryData = PxVehicleTelemetryData::allocate(4);

const PxF32 graphSizeX=0.25f;
const PxF32 graphSizeY=0.25f;
const PxF32 engineGraphPosX=0.5f;
const PxF32 engineGraphPosY=0.5f;
const PxF32 wheelGraphPosX[4]={0.75f,0.25f,0.75f,0.25f};
const PxF32 wheelGraphPosY[4]={0.75f,0.75f,0.25f,0.25f};
const PxVec3 backgroundColor(255,255,255);
const PxVec3 lineColorHigh(255,0,0);
const PxVec3 lineColorLow(0,0,0);
myTelemetryData->setup
     (graphSizeX,graphSizeY,
      engineGraphPosX,engineGraphPosY,
      wheelGraphPosX,wheelGraphPosY,
      backgroundColor,lineColorHigh,lineColorLow);
```

大小、位置和颜色都是将用于呈现图形的值。这些字段的确切值将取决于用于可视化telemetry data的坐标系和颜色编码。

在上面的示例中，坐标已配置为在屏幕中心呈现与引擎相关的图形，假设 (1，1) 是屏幕的左上角，(0，0) 是屏幕的右下角。还指定了屏幕坐标，用于渲染与四个轮子中的每个轮子关联的数据。

以下枚举列表详细介绍了收集的telemetry data:

```C++
enum
{
    eCHANNEL_JOUNCE=0,
    eCHANNEL_SUSPFORCE,
    eCHANNEL_TIRELOAD,
    eCHANNEL_NORMALISED_TIRELOAD,
    eCHANNEL_WHEEL_OMEGA,
    eCHANNEL_TIRE_FRICTION,
    eCHANNEL_TIRE_LONG_SLIP,
    eCHANNEL_NORM_TIRE_LONG_FORCE,
    eCHANNEL_TIRE_LAT_SLIP,
    eCHANNEL_NORM_TIRE_LAT_FORCE,
    eCHANNEL_NORM_TIRE_ALIGNING_MOMENT,
    eMAX_NUM_WHEEL_CHANNELS
};

enum
{
    eCHANNEL_ENGINE_REVS=0,
    eCHANNEL_ENGINE_DRIVE_TORQUE,
    eCHANNEL_CLUTCH_SLIP,
    eCHANNEL_ACCEL_CONTROL,
    eCHANNEL_BRAKE_CONTROL,
    eCHANNEL_HANDBRAKE_CONTROL,
    eCHANNEL_STEER_CONTROL,
    eCHANNEL_GEAR_RATIO,
    eMAX_NUM_ENGINE_CHANNELS
};
```

收集悬架悬架抖动(suspension jounce)、悬架力(suspension force)、轮胎负载(tire load)、归一化轮胎负载(normalized tire load)、车轮转速(wheel rotation speed)、轮胎摩擦力(tire friction)、轮胎纵向滑移( tire longitudinal slip)、轮胎纵向力(tire longitudinal force)、轮胎侧滑(tire lateral slip)、轮胎侧向力(tire lateral force)和轮胎对准力矩(tire aligning moment)的数据。此外，还单独收集发动机转速(engine revs)、发动机驱动扭矩(engine drive torque)、离合器打滑(clutch slip)、施加的加速/制动/手刹/转向(applied acceleration/brake/handbrake/steer)和齿轮比(gear ratio)的数据。对于每个图形，所有相关数据都收集在单独的图形通道中，更新完成后可以访问这些通道。

在呈现特定轮子和通道的图形之前，需要以下伪代码:

```C++
PxF32 xy[2*PxVehicleGraph::eMAX_NB_SAMPLES];
PxVec3 color[PxVehicleGraph::eMAX_NB_SAMPLES];
char title[PxVehicleGraph::eMAX_NB_TITLE_CHARS];
myTelemetryData->getWheelGraph(wheel).computeGraphChannel(PxVehicleWheelGraphChannel::eJOUNCE,
    xy, color, title);
```

此代码以 [x0，y0，x1，y1，x2，y2,....xn，yn]格式计算屏幕坐标序列，表示引擎图形数据的指定图形通道的点。它还通过根据样品的值在 `lineColorHigh` 和 `lineColorLow` 之间进行选择来存储每个样品的颜色。每个图形通道存储最后 256 个样本，以便可以在屏幕上呈现每个参数的历史记录。

`PhysX Vehicles SDK`不会呈现图形。这是留给应用程序的练习，因为每个应用程序都有自己的系统来呈现调试信息。

### Vehicle Update Multi-Threaded
---------------------------------
`PhysX Vehicles SDK`可以在多线程环境中使用，以利用并行性带来的性能改进。更新步骤几乎完全按照Vehicle Update部分中所述进行，但在对 `PxVehicleSuspendies` 的所有并发调用完成后，还需要对功能 `PxVehiclePostUpdates` `进行额外的顺序调用` Raycasts 和 `PxVehicleUpdates` 。 `PxVehiclePostUpdates` 执行通常在 `PxVehicleUpdates` 中执行的写入操作，但在采用并发时无法有效或安全地调用。

`PxVehicleSuspensionRaycasts` 是一个线程安全的函数，可以并发调用，而无需对调用代码进行任何修改，当然，任何管理同时执行raycast的任务(task)和线程(thread)的代码除外。另一方面，在载具更新部分中使用的`PxVehicleUpdates` 不是线程安全的，需要指定一个额外的 `PxVehicleConcurrentUpdateData` 数组才能并发执行。当指定此额外数据时， `PxVehicleUpdates` 会向载具更新中涉及的 `PhysX Actor`延迟许多写入操作。这些延迟的写入存储在 `PxVehicleConcurrentUpdateData` 数组中，在对 `PxVehicleUpdates` 的所有并发调用期间，然后在 `PxVehiclePostUpdates` 中按顺序执行。

示例代码可以在 SnippetVehicleMultiThreading 中找到。

### Tire Shaders
----------------
可以将 `PhysX vehicles` 使用的默认轮胎模型替换为自定义模型。这需要一个可以按载具设置的着色器方法，以及必须按车轮设置的着色器数据:

```C++
void PxVehicleWheelsDynData::setTireForceShaderFunction
    (PxVehicleComputeTireForce tireForceShaderFn)
void PxVehicleWheelsDynData::setTireForceShaderData
    (const PxU32 tireId, const void* tireForceShaderData)
```

着色器函数必须实现以下函数原型:

```C++
typedef void (*PxVehicleComputeTireForce)
(const void* shaderData,
 const PxF32 tireFriction,
 const PxF32 longSlip, const PxF32 latSlip, const PxF32 camber,
 const PxF32 wheelOmega, const PxF32 wheelRadius, const PxF32 recipWheelRadius,
 const PxF32 restTireLoad, const PxF32 normalisedTireLoad, const PxF32 tireLoad,
 const PxF32 gravity, const PxF32 recipGravity,
 PxF32& wheelTorque, PxF32& tireLongForceMag, PxF32& tireLatForceMag, PxF32& tireAlignMoment);
```

载具更新代码将使用该车轮的着色器数据为每个车轮调用着色器函数。

### Vehicle Types
---------------------
`PhysX Vehicle SDK`支持四种类型的载具: `PxVehicleDrive4W` ， `PxVehicleDriveNW` ， `PxVehicleDriveTank` 和 `PxVehicleNoDrive` 。在大多数情况下， `PxVehicleDrive4W` 将是拉力赛车，街车和赛车的最佳选择。 `PxVehicleDriveNW` 与 `PxVehicleDrive4W` 非常相似，除了 `PxVehicleDriveNW` 的优势在于它允许所有车轮都耦合到差速器上。这种普遍性意味着 `PxVehicleDriveNW` 的differential models无法与 `PxVehicleDrive4W` 支持的范围或细节相匹配。 `PxVehicleDriveTank` 通过限制左右车轮速度来模仿坦克履带的影响，从而实现了简单但高效的坦克模型。最后， `PxVehicleNoDrive` 实现了一种载具，它只是一个带有悬架，车轮和轮胎的刚性车身。这里的想法是允许使用PhysX vehicles实现滑板和气垫船等自定义驱动模型。

#### PxVehicleDrive4W
------------------------
`PxVehicleDrive4W` 类已经进行了一些详细的讨论，但到目前为止，讨论集中在四轮车上。在以下各节中， `PxVehicleDrive4W` 将特别提及少于和超过4个车轮的实例。

##### 3-Wheeled Cars
----------------------
提供了实用程序功能，可以快速配置三轮车。基本思想是从四轮车开始，然后禁用其中一个轮子:

```C++
void PxVehicle4WEnable3WTadpoleMode(PxVehicleWheelsSimData& wheelsSimData,
    PxVehicleWheelsDynData& wheelsDynData, PxVehicleDriveSimData4W& driveSimData);
void PxVehicle4WEnable3WDeltaMode(PxVehicleWheelsSimData& wheelsSimData,
    PxVehicleWheelsDynData& wheelsDynData, PxVehicleDriveSimData4W& driveSimData);
```

这些功能可确保禁用的车轮不会返回raycast命中，并执行一些其他工作，以将禁用的车轮与差速器分离，禁用阿克曼校正，将相反的剩余车轮重新定位到轴的中心，并调整剩余车轮的悬架以补偿禁用车轮的缺失悬架。理论上，可以使用自定义代码删除更多的车轮，以创建具有1或2个有效车轮的载具。但是，在这一点上，需要额外的平衡代码来防止载具翻倒。

拆卸车轮时必须小心，因为 `PxVehicleUpdates` 功能具有许多必须满足所有载具的要求。第一个要求是，任何已禁用的轮子都不得与 `PxShape` 关联。这是一项安全功能，可防止 `PxVehicleUpdates` 尝试设置可能不再有效的 `PxShape` 的local pose。`PxVehicleWheelsSimData::setWheelShapeMapping` 函数可用于满足此要求。第二个要求是，任何已禁用的车轮必须具有零车轮转速。这可以通过调用 `PxVehicleWheelsDynData::setWheelRotationSpeed` 来满足相关车轮。最后一项要求是，禁用的车轮必须没有驱动扭矩。对于坦克，这个要求实际上可以被忽略，因为它是用 `PxVehicleUpdates` 函数调用的自定义坦克代码自动强制执行的。对于 `PxVehicleNoDrive` 类型的载具，通过确保`PxVehicleNoDrive::setDriveTorque`永远不会以非零扭矩值调用，可以满足对驱动扭矩的要求。此外，通过确保差速器与禁用的车轮断开连接，可以很容易地满足`PxVehicleDriveNW`型载具的驱动扭矩要求。这是使用`PxVehicleDifferentialNWData::setDrivenWheel`函数实现的。

配置 `PxVehicle4W` 的差速器以确保没有驱动扭矩传递给禁用的车轮有点复杂，因为有许多不同的方法可以实现这一点。如果车轮不是从动轮，则禁用车轮会自动满足驱动扭矩要求，因为这样的车轮永远无法连接到差速器。另一方面，如果车轮具有index `eFRONT_LEFT` 或 `eFRONT_RIGHT` 或 `eREAR_LEFT` 或 `eREAR_RIGHT` 则确实需要修改差速器以执行要求。一种方法是设置差速器，以便在前(后)轮被禁用时仅向后(前)轮提供扭矩。这可以通过根据需要选择前轮驱动模式或后轮驱动模式来轻松实现:

```C++
PxVehicleDifferential4WData diff = myVehicle.getDiffData();
if(PxVehicleDrive4WWheelOrder::eFRONT_LEFT == wheelToDisable ||
    PxVehicleDrive4WWheelOrder::eFRONT_RIGHT == wheelToDisable)
{
    if(PxVehicleDifferential4WData::eDIFF_TYPE_LS_4WD == diff.mType ||
      PxVehicleDifferential4WData::eDIFF_TYPE_LS_FRONTWD == diff.mType ||
      PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_4WD == diff.mType ||
      PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_FRONTWD == diff.mType)
      {
          diff.mBias = 1.3f;
          diff.mRearLeftRightSplit = 0.5f;
          diff.mType = PxVehicleDifferential4WData::eDIFF_TYPE_LS_REARWD;
          //could also be PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_REARWD;
      }
}
else if(PxVehicleDrive4WWheelOrder::eREAR_LEFT == wheelToDisable ||
    PxVehicleDrive4WWheelOrder::eREAR_RIGHT == wheelToDisable)
{
    if(PxVehicleDifferential4WData::eDIFF_TYPE_LS_4WD == diff.mType ||
       PxVehicleDifferential4WData::eDIFF_TYPE_LS_REARWD == diff.mType ||
       PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_4WD == diff.mType ||
       PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_REARWD == diff.mType)
       {
          diff.mBias = 1.3f;
          diff.mFronteftRightSplit = 0.5f;
          diff.mType = PxVehicleDifferential4WData::eDIFF_TYPE_LS_FRONTWD;
          //could also be PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_FRONTWD;
       }
}
myVehicle.setDiffData(diff);
```

在某些情况下，将驱动扭矩限制在前轮或后轮上可能是不可接受的。如果只禁用了一个车轮，则可以启动驱动3个车轮的驱动模式。这可以通过修改向所有四个车轮( `eDIFF_TYPE_LS_4WD` 或 `eDIFF_TYPE_OPEN_4WD` )提供扭矩的差速器来实现，以便扭矩仅传递到3个车轮:

```C++
PxVehicleDifferential4WData diff = myVehicle.getDiffData();
if(PxVehicleDrive4WWheelOrder::eFRONT_LEFT == wheelToDisable ||
    PxVehicleDrive4WWheelOrder::eFRONT_RIGHT == wheelToDisable)
{
    if(PxVehicleDifferential4WData::eDIFF_TYPE_LS_4WD == diff.mType ||
      PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_4WD == diff.mType)
      {
          if(PxVehicleDrive4WWheelOrder::eFRONT_LEFT == wheelToDisable)
          {
              diff.mFrontLeftRightSplit = 0.0f;
          }
          else
          {
              diff.mFrontLeftRightSplit = 1.0f;
          }
      }
}
else if(PxVehicleDrive4WWheelOrder::eREAR_LEFT == wheelToDisable ||
    PxVehicleDrive4WWheelOrder::eREAR_RIGHT == wheelToDisable)
{
    if(PxVehicleDifferential4WData::eDIFF_TYPE_LS_4WD == diff.mType ||
       PxVehicleDifferential4WData::eDIFF_TYPE_OPEN_4WD == diff.mType)
       {
          if(PxVehicleDrive4WWheelOrder::eREAR_LEFT == wheelToDisable)
          {
              diff.mRearLeftRightSplit = 0.0f;
          }
          else
          {
              diff.mRearLeftRightSplit = 1.0f;
          }
       }
}
myVehicle.setDiffData(diff);
```

在某些情况下，如果禁用的车轮能够转向，则禁用Ackermann转向校正(Ackermann steer correction)是有意义的。特别是，如果前桥或后轴的剩余车轮被重新定位，使其位于车轴的中心，那么几乎可以肯定的是，阿克曼校正将被禁用。这可以通过将精度设置为零来实现(`PxVehicleAckermannGeometryData::mAccuracy`)。然而，阿克曼转向校正的作用确实需要逐案确定。

##### N-Wheeled Cars
-----------------------
除了从载具上取下车轮外，还可以构建一个超过4个车轮的 `PxVehicleDrive4W` ，但需要注意的是，只能驱动4个车轮。由于这一警告，与前4个车轮相比，额外车轮的功能略有限制。更具体地说，只有前4个车轮连接到差速器或转向器;也就是说，只有4个车轮的第一个块可以体验到驱动扭矩或转向角，只有4个车轮的第一个块参与Ackermann转向校正。因此，额外的车轮与具有前轮驱动的4轮汽车的后轮或具有后轮驱动的4轮汽车的后轮具有相同的作用。添加额外的轮子并不排除调用 `PxVehicle4WEnable3WTadpoleMode` 或 `PxVehicle4WEnable3WDeltaMode` 的能力。但是，这些功能是硬编码的，以禁用可以连接到转向并通过差速器驱动的4个车轮之一。

以下伪代码说明了创建 6 轮 `PxVehicleDrive4W` 载具的关键步骤:

```C++
PxVehicleWheelsSimData* wheelsSimData=PxVehicleWheelsSimData::allocate(6);
PxVehicleDriveSimData4W driveSimData;
setupSimData(wheelsSimData,driveSimData);
PxVehicleDrive4W* car = PxVehicleDrive4W::allocate(6);
PxRigidDynamic* vehActor=createVehicleActor6W();
car->setup(&physics,vehActor,*wheelsSimData,driveSimData,2);
```

#### PxVehicleDriveNW
---------------------
虽然 `PxVehicleDrive4W` 允许创建和模拟具有任意数量车轮的汽车，但它只允许其中4个车轮通过差速器由发动机扭矩驱动。 `PxVehicleDriveNW` 型载具已被引入以解决这一特定限制。该载具类别使用差速器类型`PxVehicleDifferentialNW` ，该类允许任何或所有载具的车轮与差速器耦合，但限制是差速器上可用的扭矩始终在连接到差速器的车轮之间平均分配。 `PxVehicleNW` 的通用性排除了限滑差速器和阿克曼转向校正等高级功能，这意味着目前只能提供简单的等分差速器模型(equal-split differential model)。

以下伪代码说明了创建6轮 `PxVehicleDriveNW` 载具的关键步骤:

```C++
PxVehicleWheelsSimData* wheelsSimData=PxVehicleWheelsSimData::allocate(6);
PxVehicleDriveSimDataNW driveSimData;
setupSimData(wheelsSimData,driveSimData);
PxVehicleDriveNW* car = PxVehicleDriveNW::allocate(6);
PxRigidDynamic* vehActor=createVehicleActorNW();
car->setup(&physics,vehActor,*wheelsSimData,driveSimData,6);
```

#### PxVehicleDriveTank
-------------------------
`PhysX vehicle SDK`还通过使用 `PxVehicleDriveTank` 类来支持坦克。坦克与多轮载具的不同之处在于，车轮全部通过差速器驱动，以确保左侧的所有车轮具有相同的速度，而右侧的所有车轮具有相同的速度。这种对车轮速度的额外限制模仿了履带的影响，但避免了模拟连接履带结构的费用。添加履带的几何`Shape`非常简单，只需在每侧添加一个`Actor Shape`，然后为履带设置适当的碰撞和查询过滤器即可。履带的运动可以用滚动纹理(scrolling texture)渲染，因为知道所有车轮都具有相同的速度，就像它们受到履带旋转的适当约束一样。

创建 `PxVehicleDriveTank` 实例与创建 `PxVehicleDrive4W` 实例非常相似，不同之处在于Tanks没有未连接到差速器的额外车轮的概念:所有坦克车轮都由驱动。以下代码说明了如何设置 12 轮坦克:

```C++
PxVehicleWheelsSimData* wheelsSimData = PxVehicleWheelsSimData::allocate(12);
PxVehicleDriveSimData4W driveSimData;
setupTankSimData(wheelsSimData,driveSimData);
PxVehicleDriveTank* tank = PxVehicleDriveTank::allocate(12);
PxRigidDynamic* vehActor=createVehicleActor12W();
tank->setup(&physics,vehActor,*wheelsSimData,tankDriveSimData,12);
```

控制Tank与控制汽车完全不同，因为Tank具有完全不同的转向机制:Tank的转弯动作源于左右车轮速度的差异，而汽车则通过方向盘的动作转动，该方向盘相对于载具的前向运动方向。这需要一组完全不同的帮助程序类和函数来平滑控制输入:

```C++
1. PxVehicleDriveTankRawInputData
2. PxVehicleDriveTankSmoothDigitalRawInputsAndSetAnalogInputs
3. PxVehicleDriveTankSmoothAnalogRawInputsAndSetAnalogInputs
```

`PhysX` 坦克目前支持两种驱动模型: `eSTANDARD` 和 `eSPECIAL` 。驱动模型 `eSPECIAL` 允许坦克履带向不同方向旋转，而 `eSTANDARD` 则不允许。这两种模式导致完全不同的转弯动作。驾驶模型 `eSTANDARD` 模拟坦克的通常转弯动作:在左(右)摇杆上向前推动左(右)车轮向前，而在右(左)摇杆上向后拉动，将制动器施加到右(左)车轮上。另一方面， `eSPECIAL` 模拟了一种更具异国情调的转弯动作，其中向右(左)摇杆向后推右(左)车轮。这可能导致一个聚焦在坦克中心的转弯圈。在 `eSTANDARD` 中，Tank的最小可能转弯半径将聚焦在沿着履带之一的某个点，具体取决于Tank是向左转还是向右转。

#### PxVehicleNoDrive
----------------------
`PhysX` 引入了 `PxVehicleNoDrive` 类，以提供与 2.8.x `NxWheelShape` 类接口的向后兼容性的近似值。它本质上是一个刚性车身，连接了N个悬架/车轮/轮胎(suspension/wheel/tire)单元。它的行为与 `PxVehicleDrive4W` 的行为相同，后者永久处于空档，因此发动机对车轮没有影响，车轮仅通过刚体的运动进行耦合。当然，这不需要阿克曼转向校正数据、发动机扭矩曲线数据等的存储开销。这个想法是，用户可以在现有载具代码的基础上开发自己的驱动模型，以管理suspension raycasts，轮胎和悬架力计算以及PhysX SDK integration。

主要功能是设置每个车轮驱动和制动扭矩以及每个车轮的方向角:

```C++
/**
\brief Set the brake torque to be applied to a specific wheel
*/
void setBrakeTorque(const PxU32 id, const PxReal brakeTorque);

/**
\brief Set the drive torque to be applied to a specific wheel
*/
void setDriveTorque(const PxU32 id, const PxReal driveTorque);

/**
\brief Set the steer angle to be applied to a specific wheel
*/
void setSteerAngle(const PxU32 id, const PxReal steerAngle);
```

### SI Units
-----------------
到目前为止的讨论假设距离以米为单位，质量以千克为单位，时间以秒为单位。此外，所有相关载具部件的默认值都是在假设将采用SI Units的情况下设置的。此类默认参数的一个示例是最大制动扭矩值(maximum braking torque value)。检查 `PxVehicleWheelData` 的构造函数，发现 `mMaxBrakeTorque` 的值为 1500。这个数字实际上表示1500"千克米平方/秒平方"的值(另一种表示方式是1500"牛顿米")。一个重要的问题是，如果不采用SI Units，如何设置具有有意义值的载具。本节的目的是说明所需的步骤。特别是，以厘米而不是米为单位测量距离的情况将作为示例。这种与采用SI Units的特殊偏差可能是游戏开发中最常见的偏差，由所选3D建模package中的距离单位引起。

其值取决于长度刻度(length scale)的载具参数分为两类:理论上可以用尺子测量的参数，以及涉及质量或时间甚至距离幂(powers of distance)等其他属性组合的更复杂单位的参数。前一类包括车轮半径(wheel radius)或最大悬架下垂(maximum suspension droop)等数据字段，而后一类包括最大制动扭矩(maximum braking torque)或车轮惯性矩(wheel moment of inertia)等数据字段。

以下是载具参数的详尽列表，理论上只能从`vehicle geometry`中测量:

```C++
PxVehicleChassisData::mCMOffset

PxVehicleAckermannGeometryData::mFrontWidth

PxVehicleAckermannGeometryData::mRearWidth

PxVehicleAckermannGeometryData::mAxleSeparation

PxVehicleWheelData::mRadius

PxVehicleWheelData::mWidth

PxVehicleSuspensionData::mMaxCompression

PxVehicleSuspensionData::mMaxDroop

PxVehicleWheelsSimData::setSuspForceAppPointOffset

PxVehicleWheelsSimData::setTireForceAppPointOffset

PxVehicleWheelsSimData::setWheelCentreOffset
```

值得注意的是，上述所有参数的默认值均为零。也就是说，与长度尺度(length scale)无关，如果要成功实例化合法载具，则必须始终使用相应长度尺度中的测量值来设置它们。

与上面列表中的参数相比，设置涉及更复杂的长度刻度组合的参数需要稍微多一些思考。一个简单的经验法则是，任何具有与距离线性(linear with distance)单位的参数都必须按相当于 1 米进行缩放，而任何具有涉及距离平方(square of distance)单位的参数必须按相当于 1 米的平方进行缩放。例如，车轮制动扭矩为1500千克平方米/秒平方，相当于1500 * 100 * 100千克平方厘米/秒平方。因此，当使用厘米作为长度刻度时，车轮制动扭矩的良好初始预测是15000000[ 千克厘米平方/秒平方 ]。如果使用英寸作为长度刻度，那么对车轮制动扭矩的良好初始预测将是1500 * 39.37 * 39.37(= 2324995.35)[ 千克英寸平方/秒平方 ]。

每个非无量纲(non-dimensionless)参数都用 `PxVehicleComponents.h` 中的相应SI Units进行了描述。以下是作为距离刻度的间接表达式的载具参数的详尽列表:

```C++
PxVehicleEngineData::mMOI (kg m^2)

PxVehicleEngineData::mPeakTorque (kg m^2 s^-2)

PxVehicleEngineData::mDampingRateFullThrottle (kg m^2 s^-1)

PxVehicleEngineData::mDampingRateZeroThrottleClutchEngaged (kg m^2 s^-1)

PxVehicleEngineData::mDampingRateZeroThrottleClutchDisengaged (kg m^2 s^-1)

PxVehicleClutchData::mStrength (kg m^2 s^-1)

PxVehicleWheelData::mDampingRate (kg m^2 s^-1)

PxVehicleWheelData::mMaxBrakeTorque (kg m^2 s^-2)

PxVehicleWheelData::mMaxHandBrakeTorque (kg m^2 s^-2)

PxVehicleWheelData::mMOI (kg m^2)

PxVehicleChassisData::mMOI (kg m^2)
```

除最后三个参数外，上述所有参数在其关联的构造函数中均具有非零初始值。这意味着，通过将以SI Unit表示的值乘以相当于1米的长度单位数或相当于1米的平方，可以找到对其初始值的良好预测。

需要注意的是，车轮手刹扭矩(wheel handbrake torque)的默认值为零，因为并非所有车轮都响应手刹扭矩(handbrake torque)。对手刹扭矩的一个很好的预测是车轮制动扭矩(wheel braking torque)的值，也许乘以1.0到2.0之间，以确保手刹比制动器更强。

车轮惯性矩(wheel moment of inertia)和底盘惯性矩(chassis moment of inertia)通常根据车轮半径和底盘尺寸计算，因此自然会反映仿真中使用的长度刻度。如果值取自制造商数据，则确保制造商数据的单位与载具数据字段的其余部分相对应,执行适当的单位转换非常重要。

许多函数还具有作为长度刻度函数的参数。以下是此类函数的详尽列表:

```C++
PxVehicleWheelsSimData::setSubStepCount

PxVehicleWheelsSimData::setMinLongSlipDenominator

PxVehicleSetMaxHitActorAcceleration
```

在`PxVehicleWheels::setSubStepCount`中设置阈值速度需要小心。在这里，默认阈值速度为每秒5.0米。对于厘米，选择的长度刻度应传递 500 [ 厘米/秒 ] 的值以实现等效行为，或者将英寸作为所选长度刻度的值为 5 * 39.37 (= 196.85) [ 英寸/秒 ]。同样的过程也必须应用于`PxVehicleWheelsSimData::setMinLongSlipDenominator`。此处的默认值为每秒 4.0 米。如果厘米是采用的刻度，则等效值为 400 [ 厘米/秒 ]，而如果选择刻度为英寸，则需要 4 * 39.37 (=157.48) [ 英寸/秒 ]。 `PxVehicleSetMaxHitActorAcceleration` 采用一个与长度刻度线性缩放的值。如果所需的最大加速度为每秒10米，则将按厘米比例缩放至每秒10 * 100厘米。以英寸作为长度刻度，等效值为每秒 10*39.37 英寸。

`PhysX Vehicle SDK`支持任何单元系统，但需要注意的是，提供的所有数据必须符合相同的单元系统。此外，虽然严格以SI单位制表示，但默认数据值可以用作估计几乎任何可以想象的载具的任何单位制的合理值的指南。要做到这一点，一个快速的方法是决定，比如说，一辆卡车的手刹是否比家用汽车的手刹更强。现在，默认数据接近标准家用汽车的数据，因此从卡车的手刹可能强25%开始可能是一个很好的估计;即 5000 千克平方米/秒平方。如果厘米是所选的长度刻度，则可以通过注意1米等于100厘米来执行快速转换，从而导致制动扭矩设置为5000 * 100 * 100千克厘米平方/秒平方。如果质量的自然单位是克，那么注意1公斤是1000克会导致等效值为5000 * 1000克 - 平方米每秒平方。只需在相关类构造函数中记下默认值和 SI 单位，然后执行到所选单位制的转换，即可对所有载具数据字段重复此规则。

`PhysX Vehicle SDK`依赖于许多阈值，这些阈值是长度刻度的函数。这些是使用 `PxInitVehicleSDK` 函数设置的，并使用已为 `PhysX SDK` 配置的 `PxTolerancesScale` 值。如果在第一次调用 `PxVehicleUpdates` 之前未调用 `PxInitVehicleSDK` ，则会将警告传递给 `PhysX error stream`。

### Level of Detail
--------------------
尝试为载具保存宝贵的时钟周期(valuable clock cycles)似乎是明智的，这些载具要么在屏幕上不可见，要么离相机足够远，以至于很难判断它们的运动是否与世界几何体完全一致。`PhysX vehicles SDK` 提供了许多选项，用于减少仅需要低级别细节(Level of Detail)的载具的计算负载。

#### Extrapolation
-------------------
对于只需要低水平细节的载具来说，最明显的策略是简单地停止对该载具执行射线投射(`PxVehicleSuspensionRaycasts`)和更新(`PxVehicleUpdates`)。与其计算载具轮胎下方的地面并计算悬架和轮胎力，不如完全避免这些步骤，让`PhysX SDK`使用刚性车身的传统动量更新刚性车身。经过几帧后，载具的车轮可能会盘旋在地面以上或与地面相交，因此需要有一个策略来决定在载具再次正确更新之前可以通过多少个PhysX SDK更新，方法是将其包含在传递给`PxVehicleSuspensionRaycasts/PxVehicleUpdates`的载具数组中。任何此类策略的细节都留给vehicles SDK的用户，因为它取决于许多因素，例如与摄像头的距离;载具附近世界几何的空间相干性;载具的速度;以及载具的音频或图形fx是否起着重要作用。

#### Disable Wheels
--------------------
如果存在车轮数量较多的载具，也可以通过调用`PxVehicleWheelsSimData::disableWheel`来减少参与模拟的车轮数量。例如，一辆有 18 个轮子的卡车。现在，这样的卡车显然需要执行18次raycast，18次轮胎力计算和18次车轮转速更新，才能完成载具更新。如果卡车可以减少到只有4个启用的车轮，那么很明显需要更少的计算工作。重要的是要注意，当车轮被禁用时，它们不再参与支撑载具刚性车身的质量。在18轮卡车减少到只有4个主动车轮的极端情况下，这意味着剩余的悬架弹簧仅配置为支持载具刚性车身质量的约4/18。为了解决这个问题，刚体的质量将需要在启用的车轮和悬架之间重新分配，也许使用 `PxVehicleComputeSprungMasses` 。有关不完整车轮问题的更完整描述，请参见3-Wheeled Cars部分。

#### Swapping Multiple Vehicle Versions
---------------------------------------
与其禁用车轮，也许降低计算成本的更简单，更有效的方法是实例化具有不同车轮数量的两个版本的载具。
随着所需细节级别的增加和减少，这两个车辆可以在传递给 PxVehicleSuspensionRaycasts/PxVehicleUpdates 的车辆阵列中轻松交换。值得考虑的是，在前面提到的18轮卡车的情况下，这可能是如何运作的。最简单的策略是首先构建所需的刚体，并为18轮卡车的18个车轮中的每一个安装一个 `PxShape` 实例。使用 `PxVehicleNW::create` 或 `PxVehicleNW::setup` 实例化所需的 18 轮卡车版本，将自动以其余pose摆出所有 18 个车轮的`Shape`。下一步是选择18个车轮中的4个，以形成卡车的4轮版本。有许多选择可供选择，但最明显的选择是18轮卡车的前左/前右/后左/后右轮。然后，可以使用与18轮版本相同的刚体来实例化4轮版本。这将使 4 个 `PxShape` 实例与 4 轮卡车的其余部分pose相同。如果4轮版本的车轮设置正确，则其余pose应与18轮版本中的车轮相同。需要注意的一个关键点是，两个版本的载具都对同一刚性车身施加力。另一个需要注意的关键点是，当选择4轮载具时，18个 `PxShape` 实例中只有4个将更新其pose，留下14个 `PxShape` 实例，其余的使用本地pose或上次使用18轮版本时提供给它们的本地pose。在可见精度方面，这些未摆放的`Shape`是较低LOD载具的主要缺点。操控性的差异更难衡量。

有许多有用的功能，可以很容易地在同一载具的两个或多个版本之间切换:

```C++
void PxVehicleComputeSprungMasses(const PxU32 nbSprungMasses,
    const PxVec3* sprungMassCoordinates, const PxVec3& centreOfMass, const PxReal totalMass,
    const PxU32 gravityDirection, PxReal* sprungMasses);
void PxVehicleWheelsSimData::copy(const PxVehicleWheelsSimData& src, const PxU32 srcWheel,
    const PxU32 trgWheel);
void PxVehicleSuspensionData::setMassAndPreserveNaturalFrequency(const PxReal newSprungMass);
void PxVehicleCopyDynamicsData(const PxVehicleCopyDynamicsMap& wheelMap,
    const PxVehicleWheels& src, PxVehicleWheels* trg);
```

以下伪代码希望能够清楚地说明如何应用这些函数，以便首先构造较低的LOD载具，然后在不同版本之间进行交换:

```C++
PxVehicleDriveNW* instantiate4WVersion(const PxVehicleDriveNW& vehicle18W, PxPhysics& physics)
{
    //Compute the sprung masses of the 4-wheeled version.
    PxReal sprungMasses[4];
    {
        const PxReal rigidBodyMass = vehicle18W.getRigidDynamicActor()->getMass();
        const PxVec3 wheelCoords[4] =
        {
            vehicle18W.mWheelsSimData.getWheelCentreOffset(0),
            vehicle18W.mWheelsSimData.getWheelCentreOffset(1),
            vehicle18W.mWheelsSimData.getWheelCentreOffset(2),
            vehicle18W.mWheelsSimData.getWheelCentreOffset(3)
        };
        const PxU32 upDirection = 1;
        PxVehicleComputeSprungMasses(4, wheelCoords, PxVec3(0,0,0), rigidBodyMass, upDirection,
            sprungMasses);
    }

    //Set up the wheels simulation data.
    PxVehicleWheelsSimData* wheelsSimData4W = PxVehicleWheelsSimData::allocate(4);
    for(PxU32 i = 0; i < 4; i++)
    {
        wheelsSimData4W->copy(vehicle18W.mWheelsSimData, i, i);

        PxVehicleSuspensionData suspData = wheelsSimData4W->getSuspensionData(i);
        suspData.setMassAndPreserveNaturalFrequency(sprungMasses[i]);
        wheelsSimData4W->setSuspensionData(i, suspData);
    }
    wheelsSimData4W->setTireLoadFilterData(vehicle18W.mWheelsSimData.getTireLoadFilterData());

    //Make sure the correct shapes are posed.
    wheelsSimData4W->setWheelShapeMapping(0,0);
    wheelsSimData4W->setWheelShapeMapping(1,1);
    wheelsSimData4W->setWheelShapeMapping(2,2);
    wheelsSimData4W->setWheelShapeMapping(3,3);

    //Set up the drive simulation data.
    PxVehicleDriveSimDataNW driveSimData4W = vehicle18W.mDriveSimData;
    PxVehicleDifferentialNWData diff4W;
    diff4W.setDrivenWheel(0, true);
    diff4W.setDrivenWheel(1, true);
    diff4W.setDrivenWheel(2, true);
    diff4W.setDrivenWheel(3, true);
    driveSimData4W.setDiffData(diff4W);

    //Instantiate the 4-wheeled version.
    PxRigidDynamic* rigidDynamic =
        const_cast<PxRigidDynamic*>(vehicle18W.getRigidDynamicActor());
    PxVehicleDriveNW* vehicle4W =
        PxVehicleDriveNW::create(&physics, rigidDynamic, *wheelsSimData4W, driveSimData4W, 4);

    //Delete the wheels simulation data now that we have copied the data to the instantiated
    //vehicle.
    wheelsSimData4W->free();

    //Finished.
    return vehicle4W;
}

void swapToLowLodVersion(const PxVehicleDriveNW& vehicle18W, PxVehicleDrive4W* vehicle4W,
    PxVehicleWheels** vehicles, PxU32 vehicleId)
{
    vehicles[vehicleId] = vehicle4W;

    PxVehicleCopyDynamicsMap wheelMap;
    wheelMap.sourceWheelIds[0]=0;
    wheelMap.sourceWheelIds[1]=1;
    wheelMap.sourceWheelIds[2]=2;
    wheelMap.sourceWheelIds[3]=3;
    wheelMap.targetWheelIds[0]=0;
    wheelMap.targetWheelIds[1]=1;
    wheelMap.targetWheelIds[2]=2;
    wheelMap.targetWheelIds[3]=3;

    PxVehicleCopyDynamicsData(wheelMap, vehicle18W, vehicle4W);
}

void swapToHighLowVersion(const PxVehicleDriveNW& vehicle4W, PxVehicleDrive4W* vehicle18W,
    PxVehicleWheels** vehicles, PxU32 vehicleId)
{
    vehicles[vehicleId] = vehicle18W;

    PxVehicleCopyDynamicsMap wheelMap;
    wheelMap.sourceWheelIds[0]=0;
    wheelMap.sourceWheelIds[1]=1;
    wheelMap.sourceWheelIds[2]=2;
    wheelMap.sourceWheelIds[3]=3;
    wheelMap.targetWheelIds[0]=0;
    wheelMap.targetWheelIds[1]=1;
    wheelMap.targetWheelIds[2]=2;
    wheelMap.targetWheelIds[3]=3;

    PxVehicleCopyDynamicsData(wheelMap, vehicle4W, vehicle18W);
}
```

#### Disable Raycasts
-------------------------
在某些场景中，在每次更新之前，可能不为每辆车发出raycast。根据几何`Shape`的不同，这可以带来显著的增益。

`PhysX vehicles SDK` 提供了一种简单的机制，通过在 `PxVehicleSuspensionRaycasts` 中指定一个布尔值数组作为函数参数，为每个更新和每个载具禁用或启用raycast。使用布尔数组禁用raycast的替代方法是更改传递给 `PxVehicleSuspensionRaycasts` 的载具array，以便计划在 `PxVehicle` 更新中的某些载具在更新之前不参与batched raycast。预计使用布尔数组将允许将相同的载具数组传递给raycast和更新函数，从而简化载具管理。

参与batched raycast的载具会自动存储raycast命中平面，这些平面在每次后续更新中都会重复使用，直到它们被下一次raycast的命中平面替换。这意味着没有必要在每次更新时都执行raycast，特别是如果载具移动缓慢或载具远离相机或载具连续多次更新保持在同一平面上。随着raycast之前的更新频率降低，缓存命中平面(cached hit planes)的精度也会降低，这意味着轮子放置明显较差的可能性会增加。缓存的命中平面缺乏准确性意味着，如果在每次更新之前未执行raycast，某些车轮最终可能会悬停或与地面相交。SDK的用户需要制定自己的策略来决定载具是否需要新的raycast。

如果在更新之前未执行raycast，则载具将只能报告其与场景交互的部分描述。例如，由于删除了最后一次suspension raycast所击中的`Actor`或`Shape`或材料，在几次更新之后的场景中可能不再存在。因此，如果使用缓存平面而不是新raycast的命中平面，则载具会报告`Shape`/`Actor`/材质的 NULL 指针。 PxWheelQueryResult 的文档对此进行了详细描述。

任何载具的首次更新都需要在更新之前执行raycast。如果在第一次更新之前没有进行raycast，那么载具将没有机会缓存其raycast命中的平面。此外，在每次调用 `setToRestState` 之后，载具还需要在下次更新之前执行raycast。这样做的原因是 `setToRestState` 会清除缓存的命中平面，这意味着需要再次重新计算它们。

#### Use The Clutch in Estimate Mode
-------------------------------------
`vehicle SDK` 为离合器(Clutch)实现了一个数学模型，该模型具有两种可选的操作精度模式: `eESTIMATE` 和 `eBEST_POSSIBLE` 。如果选择了 `eBEST_POSSIBLE` ，SDK 会尝试通过离合器精确地更新车轮和发动机的转速。值得一提的是， `PxVehicleDriveTank` 中的离合器模型简化为一组特别简单的方程，具有快速解析解。因此，载具 SDK 忽略了Tank的离合器精度模型，而是始终选择计算最佳解决方案。在 `PxVehicle4W` 的情况下，通过切换到 `eESTIMATE` 只能产生边际性能提升，因为最多只有4个车轮可以连接到离合器。估计的解决方案带来的真正性能提升将来自具有高车轮数的 `PxVehicleNW` 实例。

如果选择 `eESTIMATE` ，则可以使用 `PxVehicleClutchData::mEstimateIterations` 调整估计的质量。随着该变量值的增加，计算成本也会增加，并且估计的解决方案接近可能的最佳解决方案。在特别大的 `mEstimateIterations` 值下，估计解决方案的成本甚至可能超过最佳解决方案的成本，但不会增加任何精度。另一方面，特别低的值(例如 1)可能会导致引擎和车轮之间的耦合弱或不准确。这在换档后或在站立时或在急刹车时尤为明显。在这种情况下，离合器处的大角速度差异会导致需要大量计算才能解决的大扭矩。例如，一个错误的估计可能会导致在换档后发动机转速出现波动，而不是预期的平稳过渡。精度损失的幅度及其对载具行为的后续影响很难量化，确实需要针对每个载具和场景进行测试。

建议为需要高度细节的载具选择 `eBEST_POSSIBLE` ，而仅为需要较低细节级别的载具选择 `eESTIMATE` 。 在调整 `PxVehicleClutchData::mEstimateIterations` 时必须小心，以确保精度损失对于所需的细节级别是可以接受的。 在许多情况下，最低可能值 1 将证明完全可以接受。 然而，只有在采用 `eBEST_POSSIBLE` 时才能保证流畅且物理上可信的行为。

### Wheel Contact Beyond Raycasts
------------------------------------
本节描述了使用场景查询sweep(scene query sweeps)和接触修改(contact modification)来模拟车轮体积所需的步骤。 示例代码可以在 SnippetVehicleContactMod 中找到。

Algorithm 部分描述了如何使用scene query raycasts来计算载具悬架力(vehicle suspension forces)。 对此主题进行扩展，Filtering部分描述了如何使用场景查询(scene query)和模拟过滤(simulation filtering)将场景`Shape`分类为可驱动(drivable surfaces)或不可驱动(non-drivable surfaces)的表面:可驱动的表面仅与suspension raycasts交互，而不可驱动的表面仅通过rigid body contact与车轮交互 .

上述raycast和filter系统会产生各种问题。 一个问题是，将场景中的每个`Shape`都指定为可驱动的或不可驱动的可能是不切实际的:很容易想象使用部分可驱动和部分不可驱动的单个网格建模的景观。 另一个问题是raycast忽略了车轮在横向和纵向上的范围。 这在图 2a 和 2b 中进行了说明。

<img src=".\image\Vehicle_4.png" alt="Vehicle_4" style="zoom:100%;" />

图 2a:raycast忽略了车轮体积与倾斜地平面的重叠。

<img src=".\image\Vehicle_5.png" alt="Vehicle_5" style="zoom:100%;" />

图 2b:轮子在帧 1 中向墙壁滚动，并立即被推到帧 2 中的升高表面。

图 2a 中所示的问题可以通过用`sweep`替换raycast来解决。代替沿悬挂方向通过轮子中心执行raycast，代表轮子的`Shape`从其最大压缩时的transform`sweep`到其最大伸长时的transform。 在场景中`sweep`一个体积意味着考虑所有可能的接触平面。 如图 3 所示。

<img src=".\image\Vehicle_6.png" alt="Vehicle_6" style="zoom:100%;" />

图 3:sweep拾取车轮下方的所有接触平面。

在图 3 中，我们很容易看到车轮下方有多个接触点，每个接触点都有不同的法线。我们需要决定接受哪些接触作为驱动面(driving surface)，忽略哪些接触。在某些情况下，只需要sweep遇到的第一个接触并忽略所有其他接触就足够了。对于这种情况，建议发出blocking sweep。 `PhysX` 支持两种类型的场景查询:阻塞和非阻塞。阻塞和非阻塞查询的详细描述可以在Filtering部分中找到。然而，总而言之，阻塞sweep将返回sweep体积遇到的第一个接触，而非阻塞sweep返回sweep遇到的所有接触。图 3 中的场景表明阻塞sweep就足够了，因为它将返回斜面而不是水平面。结果，载具将开始在斜面上行驶。一些场景，例如图 2b 中描绘的场景，更为复杂，需要无阻塞sweep。

<img src=".\image\Vehicle_7.png" alt="Vehicle_7" style="zoom:100%;" />

图 4:在复杂场景中导航轮子需要明智地选择sweep contacts和 rigid body contacts 。

图 4 显示了一个车轮沿水平面朝垂直面滚动。预期的行为是车轮继续在水平面上行驶并被垂直面挡住。事实证明，这可以通过明智地选择sweep contacts和rigid body contacts来轻松实现。首先要注意的是，sweep将返回图 4 中标记为 A、B 和 C 的三个接触平面。如果我们启用了车轮和环境之间的刚体接触，我们将同时将接触平面 B 和 C 作为刚体actor。下一步是设计一种策略，接受接触平面 B 用于sweep，接触平面 C 用于rigid body contacts。这种组合将确保车轮从垂直平面反弹并继续在较低的水平平面上行驶。`PhysX vehicles`采用的策略是通过将接触法线和点与悬架方向进行比较来对sweep和刚体接触进行分类。目的是将与环境的接触分为可驱动的接触面(drivable contact planes)和不可驱动的接触面(non-drivable contact planes)。这可以通过将阈值角度引入类别接触点(categories contact points)和法线来实现。

<img src=".\image\Vehicle_8.png" alt="Vehicle_8" style="zoom:100%;" />

图 5:sweep和rigid body contact points相对于悬挂方向的位置用于过滤sweep和rigid body contacts。 浅蓝色区域中的sweep contacts被接受为驱动平面(driving planes)，而粉红色区域中的rigid body contacts被接受为刚体接触平面(rigid body contact planes)。

<img src=".\image\Vehicle_9.png" alt="Vehicle_9" style="zoom:100%;" />

图 6:contact normal和suspension direction之间的角度用于将contact planes分类为either rigid body contacts或sweep contacts。 接近悬浮方向的接触法线被接受为驱动平面，而远离悬浮方向的法线被接受为刚体接触平面。

图 5 和图 6 引入了两个阈值角度，它们一起允许使用它们的位置和法线对sweep和 rigid body contacts 进行分类。 对drivable和non-drivable的接触点和法线进行数值测试(numerical test)可以放宽Filtering部分中描述的严格过滤规则。 现在的想法是设置模拟过滤器数据(simulation filter data)，以便轮子`Shape`sweep场景中的几乎所有东西并与之碰撞。 两个阈值角度将filter,categorise sweep 和rigid body contacts，以生成所需的行为。

图 5 和图 6 所示的阈值角度通过以下函数调用进行配置:

```C++
void PxVehicleSetSweepHitRejectionAngles(const PxF32 pointRejectAngle, const PxF32 normalRejectAngle);
```

代码片段 SnippetVehicleContactMod 演示了如何配置阻塞和非阻塞sweep。 通过修改 `BLOCKING_SWEEPS` 定义，可以将此代码段配置为使用任一类型的sweep运行。 运行带有 `BLOCKING_SWEEPS` 的代码片段表明图 4 中描述的情况需要非阻塞sweep以确保不选择升高的水平面作为行驶面。

使用以下代码发出suspension sweeps:

```C++
//Suspension sweeps (instead of raycasts).
//Sweeps provide more information about the geometry under the wheel.
PxVehicleWheels* vehicles[NUM_VEHICLES] = {gVehicle4W[0], gVehicle4W[1]};
PxSweepQueryResult* sweepResults = gVehicleSceneQueryData->getSweepQueryResultBuffer(0);
const PxU32 sweepResultsSize = gVehicleSceneQueryData->getQueryResultBufferSize();
PxVehicleSuspensionSweeps(gBatchQuery, NUM_VEHICLES, vehicles, sweepResultsSize, sweepResults, gNbQueryHitsPerWheel, NULL, 1.0f, 1.01f);
```

在实现非阻塞sweep的情况下，函数 `PxVehicleUpdates` 使用 `PxVehicleSetSweepHitRejectionAngles` 中设置的阈值角度拒绝和接受sweep命中。 当执行阻塞sweep时，只记录一次sweep contact。 因此， `PxVehicleUpdates` 会忽略阈值角度并自动处理blocking sweep hit。 是使用阻塞sweep还是非阻塞sweep由开发人员决定，因为这取决于有关载具将遇到的几何类型的知识。 在某些应用中，选择计算上更便宜的阻塞sweep选项就足够了，而其他应用程序可能期望载具在复杂的几何`Shape`上行驶，并准备接受非阻塞sweep的额外成本。

rigid body contacts的分类是使用contact modification callbacks实现的。Contact modification在Contact Modification部分中进行了描述。 `PhysX Vehicles SDK` 提供了函数 `PxVehicleModifyWheelContacts` 来使用定义的阈值角度接受或拒绝接触点。此函数应从应用程序拥有的contact modification callback中调用。contact modification callbacks的配置涉及simulation filter data和simulation shader的组合。因此，实现细节留给应用程序开发人员。 SnippetVehicleContactMod 说明了一种使用simulation filter data和 `PxShape` 和 `PxRigidDynamic` 的用户数据指针来实现contact modification callbacks的方法。在应用程序中使用本地知识的其他技术是可用的。除了添加sweep和contact modification之外，该片段还将连续碰撞检测 (CCD) 应用于wheel shape。 CCD 是在断面连续碰撞检测中引入的。

## Tuning Guide
--------------------
本节描述了 PxVehicleComponents.h 中数据结构的可编辑载具参数的影响。

### PxVehicleWheelData
-----------------------
mRadius:

+ 这是车轮中心与轮胎外缘之间的距离(以米为单位)。 半径值与轮子渲染网格的半径非常匹配，这一点很重要。 任何不匹配都会导致车轮悬停在地面上方或与地面相交。 理想情况下，此参数将从 3D 建模器导出。

mWidth:

+ 这是车轮的全宽(以米为单位)。 此参数与操控无关，但在尝试呈现与车轮/轮胎/悬架相关的调试数据时是一个非常有用的参数。 如果没有这个参数，就很难计算渲染点和线的坐标以确保它们的可见性。 理想情况下，此参数将从 3D 建模器导出。

mMass:

+ 这是车轮和轮胎的总质量(kg)。 通常，车轮的质量在 20Kg 到 80Kg 之间，但根据载具的不同，车轮的质量可能会越来越低。

mMOI:

+ 这是车轮绕滚动轴转动惯量的分量。 较大的值使车轮更难绕此轴旋转，而较低的值使车轮更容易绕滚动轴旋转。 另一种表达方式是，当踩油门时，高 MOI 将导致车轮打滑较少，因为更难使车轮打滑。 相反，当踩油门时，较低的 MOI 值会导致更多的车轮打滑。如果轮子近似圆柱形，那么可以使用一个简单的公式来计算 MOI:$MOI = 0.5 * Mass * Radius * Radius$ 然而，没有理由依赖方程来计算这个值。 调整这个数字的一个好策略可能是从上面的等式开始，然后对值进行小的调整，直到处理达到所需为止。

mDampingRate:

+ 该值描述了自由旋转的轮子静止的速度。 阻尼率描述了自由旋转的车轮失去旋转速度的速率。 在这里，自由旋转的车轮是指除了车轮内部轴承产生的阻尼力之外不承受任何力的车轮。 较高的阻尼率导致车轮在更短的时间内停止，而较低的阻尼率导致车轮保持速度更长时间。 范围 (0.25, 2) 中的值似乎是合理的值。 实验总是一个好主意，即使在这个范围之外。 对于非常小的阻尼率，请务必小心谨慎。 尤其应避免阻尼率恰好为 0。

mMaxBrakeTorque:

+ 这是最大程度地施加制动器时施加到车轮的扭矩值。 制动时，较高的扭矩将更快地锁定车轮，而较低的扭矩将需要更长的时间来锁定车轮。 该值与车轮 MOI 密切相关，因为 MOI 决定了车轮对施加的扭矩的反应速度。
+ 大约 1500 的值是普通车轮的一个很好的起点，但网络搜索将显示典型的制动扭矩。 一个困难是制造商通常将这些表示为制动马力或“磅英寸”。 此处所需的值以“牛顿米”为单位。

mMaxHandBrakeTorque:

+ 除了手刹而不是制动器之外，mMaxHandBrakeTorque与最大制动扭矩(max brake torque)相同。 通常情况下，对于四轮汽车，手刹比刹车强，只作用于后轮。 后轮的值为 4000 是一个很好的起点，而前轮的值必须为 0，以确保它们不会对手刹做出反应。

mMaxSteer:

+ 这是方向盘完全锁定时车轮转向角的值(以弧度为单位)。通常，对于四轮汽车，只有前轮响应转向。在这种情况下，后轮需要值为 0。然而，更奇特的汽车可能希望前后轮对转向做出反应。相当于 30 度到 90 度之间的某个弧度值似乎是一个很好的起点，但它实际上取决于被模拟的载具。较大的最大转向值将导致更紧的转弯，而较小的值将导致更宽的转弯。但请注意，高速下的大转向角可能会导致汽车失去牵引力并失控，就像真车会发生的那样。避免这种情况的一个好方法是过滤在运行时传递给汽车的转向角，以在较大的速度下生成较小的转向角。该策略将模拟在高速下实现大转向角的难度(在高速下，车轮抵抗方向盘施加的转向力)。

mToeAngle:

+ 这是在未施加转向的情况下发生的车轮角度(以弧度为单位)。 toe angle可用于帮助汽车在出弯后直立。 这是一个很好的实验数字，但最好保留为 0，除非需要进行详细的调整。
+ 为了帮助汽车直立，对一个前轮应用一个小的负角，对另一个前轮应用一个小的正角。 通过选择哪个车轮采用正角度，哪个车轮采用负角度，很容易使车轮“toe-in”或“toe-out”。 “toe-in”配置，前轮略微指向彼此，应该有助于汽车在转弯后直立，但代价是首先使转弯变得有点困难。 “toe-out”配置可能会产生相反的效果。 最好避免toe angle大于几度。

### PxVehicleWheelsSimData
----------------------------
void setSuspTravelDirection(const PxU32 id, const PxVec3& dir):

+ 这是在载具的静止配置中的向下方向的悬架方向。 一个直接指向下方的向量是一个很好的起点。

void setSuspForceAppPointOffset(const PxU32 id, const PxVec3& offset):

+ 这是悬架力的应用点，表示为与载具刚体质心的偏移向量。另一种表达方式是从刚体的质心开始，然后沿着偏移向量移动。偏移矢量末端的点是将施加悬架力的点。

+ 在实际载具中，悬架力通过悬架支柱调节。这些通常是极其复杂的机械系统，模拟起来的计算成本很高。因此，与其对悬架支柱的细节进行建模，不如假设悬架支柱具有一个有效点，在该点处它将力施加到刚体上。然而，选择那个点需要仔细考虑。同时，它开辟了各种调整的可能性，摆脱了现实世界的束缚。

+ 决定悬挂力施加点需要一些思考。悬架非常靠近车轮，因此车轮中心是一个很好的起点。考虑一条穿过车轮中心并沿着悬架行驶方向的线。沿着这条线的某个地方对于应用点似乎是一个更好的主意，尽管不完全科学。对于标准的 4 轮汽车，应用点位于车轮中心上方但在刚体质量中心下方的某个位置是有意义的。它可能在车轮中心上方，因为悬架大多在此点上方。可以假设它位于刚体质心下方的某处，否则载具会倾斜出转弯而不是向转弯内倾斜。这将应用点缩小到已知线路的一小部分。

+ 在编辑悬架力应用点时，重要的是要记住，将应用点降低太多会导致汽车更倾向于转弯。这可能会对操控产生负面影响，因为内轮可以承受如此多的负载以致响应饱和，而外轮最终会降低负载并降低转向力。结果是过弯不佳。相反，将应用程序点设置得太高会导致转弯看起来不自然。目的是达到良好的平衡。

void setTireForceAppPointOffset(const PxU32 id, const PxVec3& offset):

+ 除了在轮胎上产生的横向和纵向力之外，这几乎与悬架力应用点相同。 一个好的起点是复制悬挂力施加点。 仅对于真正详细的编辑，建议独立于悬架力应用偏移开始调整轮胎力应用偏移。

void setWheelCentreOffset(const PxU32 id, const PxVec3& offset):

+ 这是静止位置的车轮中心，表示为与载具质心的偏移向量。

### PxVehicleSuspensionData
-------------------------------
mSprungMass:

+ 这是由悬架弹簧支撑的质量(kg)。

+ 具有四个车轮中心的刚体质心的载具通常由每个悬架弹簧均等地支撑； 也就是说，每个悬架弹簧支撑载具总质量的 1/4。 如果质心向前移动，那么预计前轮需要比后轮支撑更多的质量。 相反，更靠近后轮的质量中心应该导致后悬架弹簧比前悬架支撑更多的质量。

**为了在所需的静止pose下获得稳定性，建议弹簧质量的集合与刚体的质量和质心相匹配。 有两种策略可以用来实现这一点。 第一种方法是确定单个弹簧质量的值，然后继续计算刚体质量和质心的等效值。 更具体地说，可以使用截面算法中提供的方程计算刚体质量和质心，然后将其应用于载具的 `PxRigidDynamic` 实例。 第二种方法从载具 `PxRigidDynamic` 实例的刚体质量和质心开始，然后反向计算和设置弹簧质量。 这利用了在 `setupWheelsSimulationData` 部分中引入的函数 `PxVehicleComputeSprungMasses` 。**

mMaxCompression:

mMaxDroop:

+ 这些值描述了弹簧可以支持的以米为单位的最大压缩和伸长率。 允许的沿弹簧方向的总行程距离是  `mMaxCompression` 和 `mMaxDroop` 的总和。

+ 说明最大下垂和压缩值的一个简单方法是考虑一辆悬挂在半空中的汽车，因为没有一个车轮接触地面。 轮子会自然地从它们的静止位置向下落下，直到达到最大下垂。 弹簧不能拉长超过该点。 现在考虑将轮子向上推，首先推到其静止位置，然后进一步推，直到弹簧不能再被压缩。 从静止位置的位移是弹簧的最大压缩量。

+ 选择最大压缩值很重要，这样车轮就不会被放置在车轮的视觉网格与汽车底盘的视觉网格相交的地方。 理想情况下，这些值将从 3d 建模器导出。

mSpringStrength:

+ 该值描述了悬架弹簧的强度。弹簧强度通过调节载具响应道路颠簸所需的时间以及轮胎承受的负载量对操控性产生深远的影响。 理解弹簧强度影响的关键是弹簧固有频率的概念。考虑一个简单的弹簧系统，例如来回摆动的钟摆。钟摆每秒从左到右再返回的次数称为钟摆的固有频率。更强大的摆簧会导致摆锤摆动得更快，从而增加自然频率。相反，增加摆锤质量会导致振荡变慢，从而降低固有频率。在悬架弹簧支撑载具质量的固定部分的情况下，弹簧的强度会影响固有频率；也就是说，弹簧可以响应载荷分布变化的速率。考虑一辆车拐弯。当汽车过弯时，它会向转弯处倾斜，从而将更多的重量放在转弯外侧的悬架上。弹簧通过施加力来重新分配负载的反应速度由固有频率控制。非常高的自然频率，例如赛车上的频率，自然会产生颤抖的操控性，因为轮胎上的负载以及它们可能产生的力变化非常迅速。另一方面，非常低的固有频率会导致汽车即使在转弯完成后也需要很长时间才能恢复正常。这将产生缓慢和反应迟钝的处理。强度和固有频率的另一个影响是汽车对道路颠簸的响应。高固有频率会导致汽车对颠簸做出非常强烈和快速的反应，车轮甚至可能会离开道路一小段时间。这不仅会造成颠簸，还会造成轮胎不产生力的时间段。较弱的弹簧将导致在颠簸时更平稳，轮胎力更弱但更恒定。必须找到一个平衡点来根据预期的转弯和地形类型调整汽车。弹簧的固有频率对计算机模拟提出了挑战。平滑和稳定的模拟需要以远大于弹簧固有频率的频率更新弹簧。另一种表达方式是考虑相对于模拟时间步长的弹簧周期。弹簧的周期是弹簧完成单次振荡所需的时间，在数学上等于固有频率的倒数。为了实现稳定的模拟，必须在每次振荡期间在多个点对弹簧进行采样。这种观察的一个自然结果是模拟时间步长必须明显小于弹簧的周期。为了进一步讨论这一点，引入一个比率来描述每个弹簧振荡期间将发生的模拟更新数量是有帮助的。这个比率只是弹簧周期除以时间步长$alpha = sqrt(mSprungMass/mSpringStrength)/timestep$其中 `sqrt(mSprungMass/mSpringStrength)` 是弹簧的周期。 alpha 值为 1.0 意味着所选的时间步长和弹簧属性在每次振荡期间仅允许弹簧的单个样本。如上所述，这几乎肯定会产生不稳定的行为。事实上，到目前为止提出的论点表明，明显大于 1.0 的 alpha 值对于生成平滑模拟至关重要。出现稳定性的 alpha 的确切值很难预测，并且取决于许多其他参数。但是，作为指导，建议选择时间步长和弹簧属性，以便它们产生大于 5.0 的 alpha 值；也就是说，每个弹簧周期至少有五个模拟更新。在调整悬架弹簧时，使用制造商数据来发现在一系列载具类型中使用的典型值非常有用。这些数据并不总是容易获得。另一种策略是通过想象汽车从 0.5m 的高度掉到地上时上下摆动的速度来考虑弹簧的固有频率。典型家用汽车的弹簧的固有频率介于 5 到 10 之间；也就是说，这样的汽车如果轻轻地掉到地上，每秒会产生 5-10 次振动。如果弹簧支撑的质量已知，则弹簧强度可以通过以下公式计算$mSpringStrength = naturalFrequency * naturalFrequency * mSprungMass$

**为获得理论上正确的弹簧，应选择 `mSprungMass` 、 `mSpringStrength` 和 `mMaxDroop` 的值，使它们遵守方程 $mSpringStrength*mMaxDroop = mSprungMass*gravitationalAcceleration$。当这个方程满足时，弹簧保证在最大伸长率时提供精确为零的力，并在静止pose(静止pose由 `PxVehicleWheelsSimDta::setWheelCentreOffset` 定义)支撑簧上质量。然而，通常情况下，汽车的视觉要求与其操控要求相冲突。一个例子可能是艺术家对静止pose和悬挂行程限制施加的视觉要求。为了满足这种视觉要求并获得理论上正确的弹簧， `mSpringStrength` 的值必须等于 $mSprungMass*gravitationalAcceleration/mMaxDroop$。如果这个 `mSpringStrength` 的值不满足游戏的处理要求，那么就会出现无法轻易解决的冲突。因此，`PhysX Vehicles` 模块不要求弹簧是理论上完美的弹簧。不完美弹簧的后果是弹簧要么在达到最大伸长率之前停止提供向上的力，要么在最大伸长率时仍提供非零力。对载具操控性或视觉外观的影响通常很难发现。特别是，在  `PxVehicleTireLoadFilterData` 部分讨论的轮胎负载过滤进一步掩盖了任何缺陷。**

mSpringDamperRate:

这描述了弹簧耗散存储在弹簧中的能量的速率。

理解阻尼率的关键是欠阻尼、过阻尼和临界阻尼的概念。从静止状态移开的过阻尼钟摆在其所有能量消散之前无法进行一次来回行程，而欠阻尼钟摆至少可以进行一次来回行程。一个临界阻尼的钟摆在消耗掉它所有的能量之前，只做了一次来回行程。

对于载具悬架弹簧，通常很重要的是确保弹簧具有产生过阻尼但不会过多的阻尼率。例如，在转弯时，重要的是弹簧不会因将重量从左悬架转移到右悬架然后又返回而过度响应。如果发生这种情况，轮胎负载和产生的力将变化极大，从而导致操控颤抖和无法控制。另一方面，一个非常严重的过阻尼弹簧会感觉迟钝和反应迟钝。

临界阻尼的概念可用于帮助调整弹簧的阻尼率。引入一个称为阻尼比的值是有帮助的，这有助于在数学上描述欠阻尼、临界阻尼和过阻尼状态。

$dampingRatio = mSpringDamperRate/[2 * sqrt(mSpringStrength * mSprungMass)]$

值大于 1.0 的 `dampingRatio` 会产生过阻尼，正好为 1.0 的值会产生临界阻尼，而小于 1.0 的值则是欠阻尼。首先考虑弹簧是欠阻尼还是过阻尼，然后再考虑它离临界阻尼有多远会很有用。该过程允许将数字主观地应用于阻尼比。从这里可以通过重新排列上面的等式直接计算阻尼率

$mSpringDamperRate = dampingRatio * 2 * sqrt(mSpringStrength * mSprungMass)$

一辆典型的家用车可能略微过阻尼，阻尼比值可能刚好超过 1.0。一个准则是，离临界阻尼很远的值可能是不切实际的，并且会产生缓慢或抽搐的处理。很难给出一个确切的数字，但介于 0.8 和 1.2 之间似乎是阻尼比的一个很好的起点。

mCamberAtRest:

mCamberAtMaxCompression:

mCamberAtMaxDroop:

+ 这些值将车轮的外倾角描述为悬架弹簧压缩的函数。加长弹簧的车轮向内弯曲是典型的现象；也就是说，从载具的前轴从前或后看，左右车轮几乎形成了一个 V 形的边缘。另一方面，压缩弹簧通常向外弯曲；也就是说，当沿着载具的前轴从前部或后部观察时，它们几乎形成了 A 形的外边缘。

+ 这三个值允许使用简单的线性插值计算任何弹簧压缩值的外倾角。在静止状态下，当弹簧既不伸长也不压缩时，外倾角等于 `mCamberAtRest` 。当弹簧被压缩时，弯曲度计算为 `mCamberAtRest` 和 `mCamberAtMaxCompression` 之间的线性插值。当弹簧被拉长时，弯曲度计算为 `mCamberAtRest` 和 `mCamberAtMaxDroop` 之间的线性插值。

+ 外倾角由默认轮胎模型使用，并作为函数参数传递给轮胎着色器。它还用于设置以几何方式表示车轮的 `PxShape` 的局部pose。

### PxVehicleAntiRollBar
----------------------------
当载具转弯时，转向力会导致汽车侧倾。通常，位于转弯外侧的悬架弹簧被压缩，而位于转弯内侧的悬架弹簧被拉长。如果侧倾如此严重以至于内侧车轮完全离开地面，则驾驶员可能会失去对载具的控制。在这种情况下，甚至存在载具侧翻的危险。对于不太严重的侧倾，仍然存在由内侧和外侧轮胎之间的载荷分布引起的处理问题。这里的问题是载具的不平衡会导致转向不足或转向过度。

防倾杆通常用于减少转弯时自然发生的侧倾。它们通常用作扭转弹簧，施加扭矩以最小化一对车轮的弹簧位移差异。标准的家用车可能配备前后防倾杆。前杆施加扭矩以减少左前轮和右前轮之间的差异。同样，后杆施加扭矩以减少左后轮和右后轮之间的差异。

防侧倾扭矩的大小与由杆连接的两个车轮的弹簧位移差成正比。幅度也与刚度参数成正比:更硬的杆会产生更多的抗侧倾扭矩。

一般来说，可以通过增加后防倾杆的刚度来减少转向不足。增加前防倾杆的刚度通常会减少过度转向。

mWheel0: mWheel1:

防倾杆连接由索引 mWheel0 和 mWheel1 描述的两个车轮。

mStiffness:

该参数描述了防倾杆的刚度。

### PxVehicleTireData
-------------------------
轮胎力计算分两个概念阶段进行。计算的第一阶段使用线性方程独立计算力的横向和纵向分量。这些独立的力是通过将轮胎视为线性系统来计算的，因此理论上每个方向的力都可以看作是轮胎每单位滑移强度和轮胎所经历的滑移的乘积。计算的第二阶段应用组合轮胎力受轮胎载荷和摩擦力的乘积限制的规则。就像刚体一样，当轮胎在具有高摩擦值的表面上承受较大的法向载荷时，它们能够抵抗更大的水平力。考虑到这一点，轮胎的最大阻力可以近似为法向载荷和摩擦值的乘积。默认的 `PhysX Vehicle` 轮胎模型采用一系列平滑函数来实现组合轮胎力的归一化。

除了力的横向和纵向分量外，还计算了由轮胎外倾角引起的外倾推力。通常，这仅对横向和纵向分量的影响提供很小的修正。外倾力参与归一化过程。

以下轮胎参数描述了独立的横向和纵向以及外倾分量的计算；也就是说，力计算的第一个概念阶段。通篇参考了规范化过程的处理结果。

mLongitudinalStiffnessPerUnitGravity:

纵向刚度描述了每单位纵向滑移(以弧度为单位)产生的纵向力。 在这里，引入了一个代表每单位重力的纵向刚度的变量，以使该变量对重力加速度值的任何编辑具有鲁棒性。 纵向轮胎力近似等于单位重力纵向刚度与纵向滑移和重力加速度大小的乘积:

$longitudinalTireForce = mLongitudinalStiffnessPerUnitGravity * longitudinalSlip * gravity$

增加该值将导致轮胎在打滑时尝试产生更大的纵向力。 通常，增加纵向刚度将有助于汽车加速和制动。 可用的总轮胎力受轮胎负载的限制，因此请注意，该值的增加可能没有影响，甚至会以降低横向力为代价。

mLatStiffX:

mLatStiffY:

这些值共同描述了轮胎每单位横向滑移(以弧度为单位)的横向刚度。轮胎的横向刚度的作用类似于纵向刚度 ( `mLongitudinalStiffnessPerUnitGravity` )，不同之处在于它控制横向轮胎力的发展，并且是轮胎载荷的函数。通常，增加横向刚度将有助于汽车更快地转弯。可用的总轮胎力受轮胎负载的限制，因此请注意，该值的增加可能没有效果，甚至会以降低纵向力为代价。

横向刚度比纵向刚度稍微复杂一些，因为轮胎在重载下通常会提供较差的响应。典型的汽车轮胎是横向力与负载的关系图，该图具有接近零负载的线性响应，但在更大负载时饱和。这意味着在低轮胎载荷下，横向刚度对载荷具有线性响应；也就是说，更多的负载会导致更大的刚度和更大的横向(转向)力。在更高的轮胎负载下，轮胎具有饱和响应，并且处于施加更多负载不会导致更高轮胎刚度的状态。在后一种情况下，预计轮胎会开始打滑。

两个值 `mLatStiffX` 和 `mLatStiffY` 的组合描述了每单位载荷的横向刚度作为归一化轮胎载荷的函数的图表。轮胎力计算采用平滑函数，该函数需要了解归一化轮胎载荷，此时轮胎对轮胎载荷具有饱和响应，以及出现在该饱和点的每单位载荷的横向刚度。下图中可以看到典型的曲线。

参数 `mLatStiffX` 描述了归一化轮胎载荷，高于该值，轮胎对轮胎载荷具有饱和响应。归一化轮胎载荷只是轮胎载荷除以载具完全静止时所承受的载荷。例如， `mLatStiffX` 的值为 2 意味着当轮胎的载荷超过其静止载荷的两倍时，无论对轮胎施加多少额外的载荷，它都无法提供更多的横向刚度。在下图中， `mLatStiffX` 的值为 3。

参数 `mLatStiffY` 描述了每单位静止载荷的每单位横向滑动(以弧度为单位)的最大刚度。当轮胎处于饱和载荷状态时提供最大刚度，依次由 `mLatStiffX` `控制。在下图中，mLatStiffY` 的值为 18。

横向刚度的计算首先计算轮胎上的载荷，然后计算归一化载荷，以计算轮胎承受的静止载荷数。这会将轮胎放置在下图 X 轴的某个位置。查询由 `mLatStiffX` 和 `mLatStiffY` 参数化的曲线 Y 轴上的相应值，以提供每单位静止载荷的横向刚度。然后通过将查询的图形值乘以静止载荷来计算横向刚度的最终值。该最终值描述了每单位横向滑移的横向刚度。

<img src=".\image\Vehicle_10.png" alt="Vehicle_10" style="zoom:100%;" />

`mLatStiffX` 的良好起始值介于 2 和 3 之间。 `mLatStiffY` 的良好起始值约为 18 左右。

mFrictionVsSlipGraph:

这六个值描述了作为纵向滑移函数的摩擦图。载具轮胎对纵向滑移的反应很复杂。该图试图近似这种关系。

通常，轮胎在小滑移时具有线性响应。这意味着当轮胎只是轻微打滑时，它能够产生随着打滑增加而增加的响应力。在较大的滑差值下，力实际上可以从出现在最佳滑差处的峰值开始减小。超过最佳滑移率，轮胎最终开始变得越来越低效，并进入低效平台。

表面类型和轮胎类型组合的摩擦值已在可行驶表面上的轮胎摩擦部分中讨论过。摩擦与纵向滑动的关系图用作对组合摩擦值的修正。特别地，最终摩擦值是根据组合摩擦值和图形的校正值的乘积计算得出的。轮胎模型然后响应最终摩擦值。

前两个值描述了零轮胎滑移时的摩擦力:mFrictionVsSlipGraph[ 0 ] [ 0 ] = 0，和 mFrictionVsSlipGraph [ 0 ] [ 1 ] = 零滑移时的摩擦力。

接下来的两个值描述了最佳滑动和最佳滑动时的摩擦:mFrictionVsSlipGraph [ 1 ] [ 0 ] = 最佳滑动，mFrictionVsSlipGraph [ 1 ] [ 1 ] = 最佳滑动时的摩擦。

最后两个值描述了低效率平台开始时的滑移和低效率平台可用的摩擦值: mFrictionVsSlipGraph [ 2 ] [ 0 ]  = 低效率平台开始时的滑动，mFrictionVsSlipGraph [ 2 ] [ 1 ] = 在低效率的平台上可用的摩擦。

在下图中，使用了以下值:

mFrictionVsSlipGraph[0][0] = 0.0

mFrictionVsSlipGraph[0][1] = 0.4

mFrictionVsSlipGraph[1][0] = 0.5

mFrictionVsSlipGraph[1][1] = 1.0

mFrictionVsSlipGraph[2][0] = 0.75

mFrictionVsSlipGraph[2][1] = 0.60

<img src=".\image\Vehicle_11.png" alt="Vehicle_11" style="zoom:100%;" />

此处描述的摩擦值用于缩放地面的摩擦力。 这意味着它们应该在 (0,1) 范围内，但这不是一个严格的要求。 通常，图中的摩擦力将接近 1.0，以便对地面摩擦力进行小幅修正。

一个很好的起点是具有以下值的摩擦与滑动的平面图:

mFrictionVsSlipGraph[0][0]=0.0

mFrictionVsSlipGraph[0][1]=1.0

mFrictionVsSlipGraph[1][0]=0.5

mFrictionVsSlipGraph[1][1]=1.0

mFrictionVsSlipGraph[2][0]=1.0

mFrictionVsSlipGraph[2][1]=1.0

mCamberStiffnessPerUnitGravity:

+ 外倾刚度类似于纵向和横向刚度，不同之处在于它描述了每单位外倾角(以弧度为单位)产生的外倾推力。 与纵向刚度类似，引入了每单位重力的外倾刚度，以使外倾刚度在不同的重力加速度值下都具有鲁棒性。 独立外倾力计算为外倾角乘以外倾刚度乘以重力加速度:$camberTireForce = mCamberStiffnessPerUnitGravity * camberAngle * gravity;$

mType:

此参数已在可行驶表面上的轮胎摩擦部分中进行了解释。

### PxVehicleEngineData
---------------------------
mMOI:

这是发动机绕旋转轴的转动惯量。 较大的值会使引擎更难加速，而较低的值会使引擎更容易加速。 1.0 的起始值是一个不错的选择。

mPeakTorque:

这是发动机所能提供的最大扭矩。 这是用牛顿米表示的。 起始值可能在 600 左右。

mMaxOmega:

这是发动机的最大转速，以弧度每秒表示。

mDampingRateFullThrottle:

mDampingRateZeroThrottleClutchEngaged:

mDampingRateZeroThrottleClutchDisengaged:

这三个值用于计算应用于发动机的阻尼率。如果离合器接合，则阻尼率是 `mDampingRateFullThrottle` 和 `mDampingRateZeroThrottleClutchEngaged` 之间的插值，其中插值由游戏手柄或键盘生成的加速度控制值控制。在全油门时应用 `mDampingRateFullThrottle` ，而在零油门时应用 `mDampingRateZeroThrottleClutchEngaged` 。在空档时，阻尼率是 `mDampingRateFullThrottle` 和 `mDampingRateZeroThrottleClutchDisengaged` 之间的插值。

这三个值允许产生一系列效果:不受强阻尼力阻碍的良好加速、换档期间临时处于空档时可调节的阻尼力，以及使载具快速静止的强阻尼力它不再由玩家驾驶。

范围内的典型值 (0.25,3)。当阻尼率为 0 时，模拟可能会变得不稳定。

mTorqueCurve:

这是峰值扭矩与发动机转速的关系图。 汽车通常具有产生良好驱动扭矩的发动机转速范围和产生较差扭矩的其他发动机转速范围。 熟练的驾驶员将充分利用齿轮以确保汽车保持在发动机响应最快的“良好”范围内。 调整此图可以对游戏玩法产生深远的影响。

曲线的 x 轴是归一化的发动机转速； 即发动机转速除以最大发动机转速。 曲线的 y 轴是范围 (0,1) 中的乘数，用于缩放峰值扭矩。

### PxVehicleGearsData
-------------------------
mNumRatios:

这是载具的档位数，包括倒档和空档。 因此，一辆有 5 个前进挡的标准汽车在考虑倒车和空挡后的值为 7。

mRatios:

每个齿轮都需要一个传动比。 更高的齿轮比会导致更大的扭矩，但该齿轮的最高速度会降低。 通常，档位越高，传动比越低。 空档必须始终为 0，而倒档必须具有负齿轮比。 一档的典型值可能是 4，五档的典型值可能是 1.1。

mFinalRatio:

模拟器中使用的齿轮比是当前齿轮的齿轮比乘以最终的齿轮比。 最终比率是一种快速而粗略的更改汽车传动比的方法，而无需编辑每个条目。 此外，制造商引用的传动比值通常会提及每个传动比以及最终传动比。 典型值可能约为 4。

mSwitchTime:

切换时间描述了完成换档所需的时间(以秒为单位)。 在真正的汽车中不可能立即换档。 例如，手动档位需要在接合所需的目标档位之前先挂空档一小段时间。 在完成换档过程中，汽车将处于空档。 一个很好的技巧可能是通过增加齿轮切换时间来惩罚使用自动齿轮箱的玩家。

如果启用了自动档，最好将此值设置为明显低于 `PxVehicleAutoBoxData::setLatency`。 如果 autobox 延迟小于换档时间，则 autobox 可能会决定在完成升档后立即开始降档。 这种情况会使汽车在空档和一档之间循环，而在 2 档中的间隔非常短。

### PxVehicleAutoBoxData
--------------------------
自动变速箱根据发动机的转速启动升档或降档。如果发动机旋转速度超过存储在 `PxVehicleAutoBoxData` 中的阈值，则将启动齿轮增量。另一方面，如果发动机转速低于阈值，则自动变速箱将启动齿轮减量。自动变速箱一次仅启动向上或向下一个档位的换档。

值得注意的是，如果自动变速箱启动换档，则在整个换档期间油门踏板会自动与发动机断开连接。手动换档(`PxVehicleDriveDynData::startGearChange` / `PxVehicleDriveDynData::mGearUpPressed` / `PxVehicleDriveDynData::mGearDownPressed`)不受此限制。这与典型的现实世界自动装箱行为保持一致。这背后的想法是在换档的空档阶段停止发动机疯狂加速，从而避免在换档结束时离合器重新接合时损坏离合器打滑。

当自动或手动换档仍处于活动状态时，自动变速箱不会尝试启动换档。

如果自动框对于应用程序的要求来说过于简单，那么可以很容易地禁用 `PxVehicleGearsData` 。此后的选择要么是恢复到手动齿轮模型，要么是在应用程序中实现自定义自动框。可以使用` PxVehicleDriveDynData::startGearChange` 启动到特定档位的转换，而可以使用 `PxVehicleDriveDynData::mGearUpPressed` / `PxVehicleDriveDynData::mGearDownPressed` 启动单个档位更改。

可以通过切换 `PxVehicleDriveDynData::mUseAutoGears` 来启用或禁用自动框。

PxReal mUpRatios[PxVehicleGearsData::eGEARSRATIO_COUNT]:

如果发动机转速与最大允许发动机转速的比率:`PxVehicleDriveDynData::getEngineRotationSpeed() / PxVehicleEngineData::mMaxOmega`大于存储在 mUpRatios[PxVehicleDriveDynData::getCurrentGear()] 中的值

PxReal mDownRatios[PxVehicleGearsData::eGEARSRATIO_COUNT]:

如果发动机转速与最大允许发动机转速的比率:`PxVehicleDriveDynData::getEngineRotationSpeed() / PxVehicleEngineData::mMaxOmega`小于存储在 mUpRatios[PxVehicleDriveDynData::getCurrentGear()] 中的值

void setLatency(const PxReal latency):

在自动变速箱启动换档后，它不会尝试启动另一个换档，直到等待时间过去。 将此值设置为明显高于 `PxVehicleGearsData::mSwitchTime` 是个好主意。 如果延迟小于换档时间，则自动变速箱可能会在完成升档后立即决定启动降档。 这种情况会使汽车在空档和一档之间循环，而在 2 档中的间隔非常短。

### PxVehicleClutchData
--------------------------

mStrength:

这描述了离合器将发动机连接到车轮的强度以及通过将扭矩分配给发动机和车轮来消除速度差异的速度。

较弱的值会导致更多的离合器打滑，尤其是在换档或踩油门后。 更高的值将导致离合器打滑减少，并提供更多的发动机扭矩传递到车轮。

该值只能在对载具进行非常精细的调整时进行编辑。 一些离合器打滑可归因于模拟中大时间步长的数值问题，而另一些则是以过于激进的方式驾驶汽车的自然结果。 值 10 是一个很好的起点。

### PxVehicleAckermannGeometryData
-------------------------------
mAccuracy:

`Ackermann` 校正通过用稍微不同的转向角转向左轮和右轮来实现更好的转弯，这是从简单的三角学计算出来的。 在实践中，不可能设计出能够实现完美阿克曼转向校正的转向连杆。 该值允许控制阿克曼转向校正的精度。 选择 0 值会完全禁用阿克曼转向校正。 另一方面，值 1.0 实现了完美阿克曼校正。

mFrontWidth:

这是两个前轮之间的距离(以米为单位)。

mRearWidth:

这是两个后轮之间的距离(以米为单位)。

mAxleSeparation:

这是前轴中心和后轴中心之间的距离(以米为单位)。

### PxVehicleTireLoadFilterData
-------------------------------
这是为了非常精细地控制处理，并纠正大时间步长模拟中固有的数值问题。

在较大的模拟时间步长下，悬架弹簧的运动幅度大于现实生活中的幅度。不幸的是，这是不可避免的。在崎岖不平的表面上，这可能意味着模拟将汽车抬离地面的距离比实际发生的要远。紧随其后的是弹簧比实际载具所经历的压缩更多。这种振荡的结果是轮胎上的负载比预期的变化更大，可用的轮胎力比预期的变化更大。该过滤器旨在通过平滑轮胎负载来纠正这个数值问题，目的是使操控更平滑、更可预测。

一个关键概念是归一化轮胎载荷的概念。归一化轮胎载荷只是实际载荷除以载具处于静止状态时所承受的载荷。如果轮胎承受的载荷大于静止时的载荷，则归一化轮胎载荷大于 1.0。类似地，如果轮胎的载荷小于静止时的载荷，则归一化轮胎载荷小于 1.0。在静止状态下，所有轮胎显然具有恰好 1.0 的归一化轮胎载荷。归一化轮胎载荷永远不会小于零。

此处的值描述了从原始轮胎负载生成过滤后的轮胎负载的 2d 图形上的点。图表的 x 轴是“归一化轮胎载荷”，而图表的 y 轴是“过滤归一化轮胎载荷”。小于 `mMinNormalisedLoad` 的归一化负载产生一个过滤的归一化负载 `mMinFilteredNormalisedLoad` 。大于 `mMaxNormalisedLoad` 的归一化负载产生 `mMaxFilteredNormalisedLoad` 的过滤归一化负载。介于 `mMinNormalisedLoad` 和 `mMaxNormalisedLoad` 之间的负载会在 `mMinFilteredNormalisedLoad` 和 `mMaxFilteredNormalisedLoad` 之间产生一个过滤的归一化负载，通过直接插值计算。

<img src=".\image\Vehicle_12.png" alt="Vehicle_12" style="zoom:100%;" />

选择 `mMaxNormalisedLoad` 和 `mMaxFilteredNormalisedLoad` 会限制模拟中将使用的最大负载。 另一方面，选择 mMinFilteredNormalisedLoad>0 和/或 mMinNormalisedLoad>0 允许轮胎潜在地产生非零轮胎力，即使轮胎刚刚以最大下垂接触地面。

通过设置，过滤后的负载可以与计算的轮胎负载相同:

mMinNormalisedLoad=mMaxFilteredNormalisedLoad=0 

mMaxNormalisedLoad=mMaxFilteredNormalisedLoad=1000.

**轮胎可能仅在轮胎接触地面时产生力:如果轮胎不能放在地面上，则轮胎力始终为零。 另一方面，在最大悬架下垂时接触地面的轮胎具有零测量负载，因为弹簧在最大下垂时产生零力。 通过编辑 `PxVehicleTireLoadFilterData` ，即使实际作用在轮胎上的负载非常小，也可以生成轮胎力。**

### PxVehicleDifferential4WData
--------------------------------
mType:

支持多种差速器类型:带开放式差速器的四轮驱动、带限滑的四轮驱动、带开放式差速器的前轮驱动、带限滑的前轮驱动、带开放式差速器的后轮驱动、后轮驱动 限滑轮驱动。

mFrontRearSplit:

如果选择了 4 轮驱动差速器(开放式或限滑)，则该选项允许驱动扭矩在前后轮之间不均匀地分配。 选择值 0.5 可在前轮和后轮之间分配相等的扭矩； 也就是说，传递到前轮的总扭矩等于传递到后轮的总扭矩。 选择大于 0.5 的值会为前轮提供更多扭矩，而选择小于 0.5 的值会为后轮提供更多扭矩。 对于前轮驱动和后轮驱动差速器，该值被忽略。

mFrontLeftRightSplit:

这类似于前后分离，但将前轮可用的扭矩分配到左前轮和右前轮之间。 大于 0.5 的值会为左前轮提供更多扭矩，而小于 0.5 的值会为右前轮提供更多扭矩。 该参数可用于防止任何扭矩传递到损坏或失效的车轮。 对于后轮驱动，该值被忽略。

mRearLeftRightSplit:

这与 `mFrontLeftRightSplit` 类似，不同之处在于它适用于后轮而不是前轮。 对于前轮驱动，该值被忽略。

mFrontBias:

限滑差速器的工作原理是仅允许累积一定的车轮转速差异。 这可以防止出现一个车轮打滑但最终占用所有可用功率的情况。 此外，通过允许车轮转速的微小差异累积，通过允许外轮比内轮更快地旋转，载具可以容易地转弯。

该参数描述了允许累积的车轮转速的最大差异。 前偏差是两个前轮转速中的最大值除以两个前轮转速中的最小值。 当该比率超过前偏置值时，差速器将扭矩从较快的车轮转移到较慢的车轮，以试图保持最大允许的车轮转速比。

除了前轮驱动或限滑四轮驱动外，该值被忽略。

一个好的起始值是 1.3 左右。

mRearBias:

这与 `mFrontBias` 类似，只是它指的是后轮。

除了后轮驱动或限滑四轮驱动外，该值被忽略。

一个好的起始值是 1.3 左右。

mCentreBias:

该值与 `mFrontBias` 和 `mRearBias` 类似，不同之处在于它指的是前轮转速之和和后轮转速之和。

除了带限滑的四轮驱动外，该值被忽略。

一个好的起始值是 1.3 左右。

### PxRigidDynamic
------------------------
Moment of Inertia:

刚体的转动惯量在编辑载具时是一个极其重要的参数，因为它会影响载具的转弯和侧倾。

刚体惯性矩的一个很好的起点是计算出限制底盘几何`Shape`的长方体的惯性矩。如果边界长方体宽 W、高 H、长 L，则质量为 M 的载具的惯性矩为:

$((L*L+H*H)*M/12, (W*W+L*L)*M/12, (H*H+W*W)*M/12)$然而，这只是一个粗略的指南。调整每个值将修改围绕相应轴的运动，更高的值会使轮胎和悬架力更难引起旋转速度。

为惯性矩提供非物理值将导致非常缓慢的行为或极度抽搐甚至不稳定的行为。惯性矩必须至少大致反映悬架和轮胎力施加点的长度尺度。

此参数应被视为第一个可编辑的值。

Center of mass:

与转动惯量一样，质心是第一个可编辑的值之一，因此对处理具有深远的影响。

为了讨论质心，考虑一个典型的四轮载具，其底盘网格的原点位于四个车轮的中心。原点在四个轮子的中心没有要求，但它确实使下面的讨论更简单一些。可以预期质心位于该原点附近的某个地方，因为载具的设计方式几乎可以在四个车轮之间均匀分布负载。更具体地说，可以预期质心需要略高于底盘底部而不是车轮的高度。毕竟，由于发动机和其他机械系统的密度，载具靠近底盘底部的质量密度更高。因此，预计质心比顶部更靠近底盘底部，但绝对高于底部。如果没有对底盘密度分布进行特别详细的分析，沿垂直轴的确切位置确实有点武断和主观。沿着向前的方向，由于前置发动机的质量，可以预期质心比后轮更靠近前轮。考虑这些因素可以沿垂直和向前方向调整质心。

调整质心实际上就是进行增量更改，以将处理调整为期望的目标。向前移动质心应该有助于转弯，因为更多的负载分配到前轮胎。然而，这是以减少后轮胎的负载为代价的，这意味着汽车可能会更快地转弯，只是因为后轮胎更快地失去抓地力而打滑。需要进行小改动，然后对处理进行测试。

在设置质心时，重要的是要记住悬架弹簧质量值可能需要同时更新。如果质心靠近前部，这意味着前悬架支撑的质量更多，后悬架支撑的质量更少。这种变化需要以一致的方式反映出来。可以用数学方法描述质量中心与悬架之间的质量分配之间的关系。然而，打破这种僵化的链接所提供的编辑可能性应该允许更多的调整选项。

Mass:

一辆典型的汽车可能有大约 1500 公斤的质量。

## Troubleshooting
-----------------
本节介绍载具调整常见问题的常见解决方案。

### Jittery Vehicles
-----------------------
+ 是否在第一次执行 `PxVehicleUpdates` 之前调用了 `PxInitVehicleSDK` 和 `PxVehicleSetBasisVectors` ？ 检查错误流是否有警告。
+ `PxTolerancesScale` 的长度刻度是否与载具的长度刻度匹配(例如，如果使用厘米，则为 100)？ 根据需要更新 `PxTolerancesScale::length`。
+ 弹簧的固有频率是否太高/模拟的时间步长太小而无法进行可靠的模拟？ 有关更多详细信息，请参阅 PxVehicleSuspensionData 部分并相应地更新自然频率或时间步长。 请记住，可以使用 `PxVehicleWheelsSimData::setSubStepCount` 更新每辆车的时间步长。
+ 最大悬架下垂和压缩是否设置为允许某些悬架运动的值？

### The Engine Rotation Refuses To Spin Quickly
-------------------------------------------------
+ 轮胎是否通过过大的摩擦力抵抗发动机运动？将汽车放置在离地面非常高的位置并加速发动机，看看发动机和车轮是否开始旋转。
+ 发动机的转动惯量、峰值扭矩和阻尼率是否反映了长度尺度？请注意记录的每个变量的 SI 单位，并根据需要重新计算值。
+ 转动惯量是否太大？ 1 或相关长度尺度中的等效值是测试目的的一个很好的估计。
+ 峰值扭矩是否太小而无法驱动发动机？在知道默认值将驱动大约 1500 公斤的标准汽车的情况下，用载具的质量缩放默认峰值扭矩值。
+ 扭矩曲线是否包含合理值？尝试使用每个数据点的 y 值为 1.0 的平坦曲线。
+ 最大发动机角速度是一个现实值吗？查阅任何可用的制造商数据以获取典型值或恢复到默认值以进行测试。
+ 是否有任何阻尼率过高？降低阻尼率并进行测试。

### The Engine Spins But the Wheels Refuse To Spin
-------------------------------------------------
+ 载具是否处于空档？ 通过将载具设置为一档并禁用自动变速箱，将发动机连接到车轮。
+ 差速器是否向车轮传递驱动扭矩(仅适用于 `PxVehicleNW` 载具)？ 确保差速器配置正确。
+ 刹车或手刹是否接合？ 确保刹车和手刹都为零。
+ 车轮的转动惯量和阻尼率是否反映了长度尺度？ 请注意记录的每个变量的 SI 单位，并根据需要重新计算值。
+ 车轮的转动惯量是否过高？ 重新计算车轮的转动惯量。
+ 车轮的阻尼率是否过高？ 降低车轮的阻尼率。
+ 轮胎是否通过过大的摩擦力抵抗发动机运动？ 将汽车放置在离地面非常高的位置并加速发动机，看看发动机和车轮是否开始旋转。

### The Wheels Are Spinning But The Vehicle Does Not Move Forwards
------------------------------------------------------------------
+ 过滤器是否配置为仅由悬架力支撑载具？检查附加到载具刚体actor的`Shape`的过滤配置，使用PVD搜索涉及附加到载具actor的`Shape`的接触。
+ 是否向轮胎接地面传递了足够的摩擦力？使用 `PxVehicleWheelsDynData::getTireFriction` 查询轮胎在执行 `PxVehicleUpdates` 期间经历的摩擦值。
+ 悬架力(以及轮胎上的负载)是否反映了刚体 actor 的质量？使用 `PxVehicleWheelsDynData::getSuspensionForce` 查询悬架力。四轮载具应产生大约为 actorMass*gravity/4 的悬架力。调整载具悬架的簧载质量以确保从动轮承受较大的轮胎负载。
+ 轮胎是否会产生显着的纵向轮胎力？使用 `PxVehicleWheelsDynData::getTireLongSlip` 来检查轮胎的纵向滑移是否为非零并在车轮快速旋转而没有向前运动时接近 1.0。如果纵向滑移非常小，请确保已使用正确的前向向量调用 `PxVehicleSetBasisVectors` 。使用 `PxVehicleWheelsDynData::getTireLongitudinalDir` 进一步测试前向向量是否设置正确。
+ 轮胎纵向刚度是否太小？将纵向刚度调整回默认值并进行测试。
+ 载具刚体actor的质量是否太大而无法由发动机峰值扭矩驱动？测试`Actor`的质量是一个合理的值并相应地设置。
+ `PhysX` 场景中的刚体`Actor`是否正在更新？确保`Actor`没有sleep并参与场景更新。

### The Vehicle Does Not Steer/Turn
--------------------------------
+ 载具的转动惯量是否太大以致于无法转动？检查载具刚体actor的惯性矩是否为合理值。使用具有载具宽度/高度/长度的盒子的惯性矩作为`Actor`惯性矩的起始预测。
+ 方向盘是否有转向角？使用 `PxVehicleWheelsDynData::getSteer` 检查转向角。如果转向角为零或小于预期，请检查转向角是否正在传递到载具以及转向轮的最大转向角是否为合理值。
+ 转向轮是否具有合理的横向侧偏角？使用 `PxVehicleWheelsDynData::getLatSlip` 查询侧偏角。如果横向滑动非常小，请确保已使用正确的前向和向上向量调用 `PxVehicleSetBasisVectors` 。使用 `PxVehicleWheelsDynData::getTireLateralDir` 进一步测试基向量是否设置正确。
+ 轮胎的横向刚度是否配置正确？将轮胎重置为默认值并重新测试。

### The Acceleration Feels Sluggish
------------------------------------
+ 发动机和车轮的阻尼率是否过大？ 降低发动机阻尼率，然后是车轮阻尼率并每次重新测试。
+ 载具是否一直卡在同一个档位？ 禁用自动变速箱并手动换档以测试自动变速箱是否无法换档。 检查自动变速箱设置以确保它会在合理的发动机转速下自动增加档位。
+ 发动机是否足够强大以快速加速汽车？ 增加发动机的峰值扭矩。
+ 车轮是否具有高转动惯量以防止发生显着的纵向滑动？ 减少车轮的转动惯量。

### The Vehicle Does Not Slow Down When Not Accelerating
----------------------------------------------------------
+ 车轮和发动机阻尼率是否太小？ 首先增加发动机阻尼率，然后是车轮阻尼率，每次都重新测试。
+ 载具的刚体 actor 是否具有速度阻尼值？ 适当增加。

### The Vehicle Turns Too Quickly/Too Slowly
---------------------------------------------

 rigid body actor的转动惯量需要调整吗？ 调整与绕up vector运动相对应的转动惯量分量。 增加转动惯量会减慢转动速度，减少转动惯量会增加转动速度。

### The Wheels Spin Too Much Under Acceleration
-------------------------------------------------
+ 油门踏板值是否从 0 增加到 1 太快？ 通过过滤控制器或键盘来减慢油门踏板值的增加速度。 请记住，在动力强劲的汽车上猛踩油门踏板应该会导致车轮打滑。
+ 车轮转动惯量是否太低？ 增加车轮转动惯量。

### The Wheels Spin Too Much When Cornering
-------------------------------------------
载具是否有限滑差速器？ 如果适用，将差速器类型设置为限滑并相应地调整差速器偏置(differential biases)。

### The Vehicle Never Goes Beyond First Gear
---------------------------------------------
载具是否在一档和空档之间循环？如果启用了 autobox，那么问题可能是 autobox 的延迟比执行换档所花费的时间短。 autobox 延迟控制在自动换档之间花费的最短时间。在启动自动换档后，自动变速箱将不会做出另一个换档决定，直到等待时间过去。在换档期间，载具进入空档并且油门踏板与发动机分离，这意味着发动机在换档期间会减速。当载具在换档结束时进入目标档位时，自动变速箱可能会立即决定发动机对于目标档位来说太慢，并立即开始向下换档。这将立即使汽车回到空档，这意味着汽车在空档中花费了很长时间并且从未达到其目标档位。如果将自动装箱延迟 (`PxVehicleAutoBoxData::setLatency`) 设置为明显大于换档时间 (`PxVehicleGearsData::mSwitchTime`)，则不会发生这种情况。

### The Vehicle Under-steers Then Over-steers
--------------------------------------------
载具是否在颠簸的路面上？ 编辑 `PxVehicleTireLoadFilterData` 中的值，以便过滤后的归一化轮胎负载对悬架压缩具有更平坦的响应。

### The Vehicle Slows Down Unnaturally
--------------------------------------
载具不平稳减速休息？ 查看纵向滑移值，看看它们是否在正负之间振荡。 如果存在强烈振荡，则有两个选项可以单独使用或相互结合使用。 第一个选项是使用 `PxVehicleWheelsSimData::setSubStepCount` 在载具前进速度接近零时强制执行更多载具更新子步骤。 第二个选项是使用 `PxVehicleWheelsSimData::setMinLongSlipDenominator` 来确保纵向滑动的分母永远不会低于指定值。

### The Vehicle Climbs Too Steep Slopes
-----------------------------------------
前轮打滑严重吗？ 修改 `PxVehicleTireData::mFrictionVsSlipGraph` 以减少滑动车轮的可用摩擦力。