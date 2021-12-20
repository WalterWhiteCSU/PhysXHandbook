# GPU Rigid Bodies

## Introduction
---------------
GPU刚体(GPU Rigid Bodies)是 `PhysX 3.4` 中引入的一项新功能。它支持整个刚体管线特征集(rigid body pipeline feature-set)，但目前不支持`Articulations`。GPU加速刚(GPU-accelerated rigid bodies)体的状态可以使用与 CPU 刚体的修改和查询完全相同的 API 进行修改(modify)和查询(query)。GPU 刚体可以与服装(clothing)和粒子(particles)交互，就像 CPU 刚体可以轻松地与角色控制器 (CCT) 和车辆结合使用一样。

## Using GPU Rigid Bodies
--------------------------
GPU 刚体并不比 CPU 刚体更难使用。GPU 刚体使用与 CPU 刚体完全相同的 API 和类。GPU 刚体加速基于每个场景启用。如果启用，则占据场景的所有刚体都将由 GPU 处理。此功能在 CUDA 中实现，需要 SM3.0(Kepler)或更高版本的兼容 GPU。如果未找到兼容设备，则模拟将回退到 CPU 上，并提供相应的错误消息。

该特征分为两个部分:刚体动力学(rigid body dynamics)和粗检测(broad phase)。这些是使用`PxSceneFlag::eENABLE_GPU_DYNAMICS`并将`PxSceneDesc::broadphaseType`设置为`PxBroadPhaseType::eGPU`来启用的。这些属性是场景的不可变属性。此外，您必须初始化 CUDA 上下文管理器(CUDA context manager)，并在 `PxSceneDesc` 上设置 GPU 调度程序(GPU dispatcher)。这也是使用GPU加速粒子或服装的要求。SnippetHelloGRB 中提供了演示如何启用 GPU 刚体仿真的代码段。下面的代码示例用作简要参考:

```C++
PxCudaContextManagerDesc cudaContextManagerDesc;

gCudaContextManager = PxCreateCudaContextManager(*gFoundation, cudaContextManagerDesc);

PxSceneDesc sceneDesc(gPhysics->getTolerancesScale());
sceneDesc.gravity = PxVec3(0.0f, -9.81f, 0.0f);
gDispatcher = PxDefaultCpuDispatcherCreate(4);
sceneDesc.cpuDispatcher = gDispatcher;
sceneDesc.filterShader  = PxDefaultSimulationFilterShader;
sceneDesc.gpuDispatcher = gCudaContextManager->getGpuDispatcher();

sceneDesc.flags |= PxSceneFlag::eENABLE_GPU_DYNAMICS;
sceneDesc.broadPhaseType = PxBroadPhaseType::eGPU;

gScene = gPhysics->createScene(sceneDesc);
```

启用 GPU 刚体动态(GPU rigid body dynamics)可启用 GPU 加速的接触生成(GPU-accelerated contact generation)、shape/body管理和 GPU 加速的约束求解器。这加速了大部分离散刚体管道。

打开 GPU 粗检测会将 CPU 粗检测(CPU broad phase)替换为 GPU 加速粗检测(GPU-accelerated broad phase)。

每个都可以独立启用，因此，例如，您可以在 CPU rigid body dynamics 上启用 GPU 粗检测，在 GPU rigid body dynamics 上启用 CPU 粗检测(SAP 或 MBP)，或者在 GPU rigid body dynamics 上启用 GPU 粗检测。

## What is GPU accelerated?
-----------------------------
GPU 刚体功能提供 GPU 加速的实现:

+ Broad Phase 
+ Contact generation 
+ Shape and body management 
+ Constraint solver 

所有其他功能都在 CPU 上执行。

GPU 接触生成(Contact generation)有几个注意事项。具体如下:

+ GPU 接触生成仅支持`boxes`、`convex hulls`、`triangle meshes`和`heightfields`。任何`spheres`，`capsules`或`planes`都将具有涉及在CPU而不是GPU上处理的那些形状的接触生成。
+ `convex hulls`需要将 `PxCookingParam::buildGRBData` 设置为 true，以构建在 GPU 上执行接触生成所需的数据。如果使用具有 64 个以上顶点或每个面超过 32 个顶点的`convex hulls`，则将在 CPU 上对其进行处理。如果在创建`convex hulls`时使用` PxConvexFlag::eGPU_COMPATIBLE` 标志，则应用 `limit` 以确保生成的`convex hulls`可以在 GPU 上使用。
+ `triangle meshes`需要将 `PxCookingParam::buildGRBData` 设置为 true，以构建在 GPU 上处理网格所需的数据。如果在`cooking`过程中未设置此标志，则网格的 GPU 数据将不存在，并且将在 CPU 上处理涉及此网格的任何接触生成。
+ 任何请求触点修改的对都将在 CPU 上进行处理。
+ 必须启用 `PxSceneFlag::eENABLE_PCM` 才能执行 GPU 接触生成。这是在 GPU 上实现的唯一接触生成形式。如果未引发`eENABLE_PCM`，则将使用非基于距离的传统接触生成在 CPU 上处理所有对的接触生成。

无论给定对的触点生成是在 CPU 上还是在 GPU 上处理的，GPU 求解器都将处理所有具有在其筛选器着色器(filter shader)中请求碰撞响应(collision response)的触点的对。

如上所述，GPU 刚体目前不支持`Articulations`。如果在场景中启用了 `eENABLE_GPU_DYNAMICS` ，则任何向场景添加`Articulations`的尝试都将导致显示错误消息，并且不会将该`Articulations`添加到场景中。

GPU 刚体求解器为`joint`和`contact`提供全面支持。但是，使用`D6 joints`可以获得最佳性能，因为 GPU 本身支持`D6 joints`，即从准备到求解的完整求解器管道在 GPU 上实现。GPU 求解器支持其他关节类型，但它们的joint shaders在 CPU 上运行。与`D6 joints`相比，这会产生一些额外的主机端性能开销。

## Tuning
------------
与 `CPU PhysX` 不同，GPU 刚体功能无法动态增大所有缓冲区。因此，有必要为 GPU 刚体功能提供一些固定的缓冲区大小。如果可用内存不足，系统将发出警告并丢弃触点/约束/对，这意味着该行为可能会受到不利影响。以下缓冲区在 `PxSceneDesc::gpuDynamicsConfig` 中可调:

```C++
struct PxgDynamicsMemoryConfig
{
        PxU32 constraintBufferCapacity; //!< Capacity of constraint buffer allocated in GPU global memory
        PxU32 contactBufferCapacity;    //!< Capacity of contact buffer allocated in GPU global memory
        PxU32 tempBufferCapacity;       //!< Capacity of temp buffer allocated in pinned host memory.
        PxU32 contactStreamCapacity;    //!< Capacity of contact stream buffer allocated in pinned host memory. This is double-buffered so total allocation size = 2* contactStreamCapacity.
        PxU32 patchStreamCapacity;      //!< Capacity of the contact patch stream buffer allocated in pinned host memory. This is double-buffered so total allocation size = 2 * patchStreamCapacity.
        PxU32 forceStreamCapacity;      //!< Capacity of force buffer allocated in pinned host memory.
        PxU32 heapCapacity;             //!< Initial capacity of the GPU and pinned host memory heaps. Additional memory will be allocated if more memory is required.
        PxU32 foundLostPairsCapacity;   //!< Capacity of found and lost buffers allocated in GPU global memory. This is used for the found/lost pair reports in the BP.


        PxgDynamicsMemoryConfig() :
                constraintBufferCapacity(32 * 1024 * 1024),
                contactBufferCapacity(24 * 1024 * 1024),
                tempBufferCapacity(16 * 1024 * 1024),
                contactStreamCapacity(6 * 1024 * 1024),
                patchStreamCapacity(5 * 1024 * 1024),
                forceStreamCapacity(1 * 1024 * 1024),
                heapCapacity(64 * 1024 * 1024),
                foundLostPairsCapacity(256 * 1024)
        {
        }
};
```

对于模拟大约 10,000 个刚体的场景，默认值通常已足够。

+ `constraintBufferCapacity` 定义求解器中的约束(`constraints`)可以占用的内存总量。如果需要更多内存，则会发出警告，并且不会创建其他约束。
+ `contactBufferCapacity` 定义约束求解器中使用的临时接触(`temporary contact`)缓冲区的大小。如果需要更多内存，则会发出警告并丢弃触点。
+ `tempBufferCapacity` 定义用于约束求解器(`constraint solver`)中使用的各种瞬态内存分配的缓冲区的大小。
+ `contactStreamCapacity` 定义用于在触点流(contact stream)中存储触点的缓冲区的大小。此数据在固定主机内存中分配，并经过双重缓冲。如果分配的内存不足，将发出警告并删除触点。
+ `patchStreamCapacity` 定义用于在触点流中存储`contact patches`的缓冲区的大小。此数据在固定主机内存中分配，并经过双重缓冲。如果分配的内存不足，将发出警告并删除触点。
+ `forceStreamCapacity` 定义了用于向用户报告施加的接触力的缓冲区的大小。此数据在固定主机内存中分配。如果分配的内存不足，将发出警告并删除触点。
+ `heapCapacity`定义 GPU 和固定主机内存堆的初始大小。如果需要更多内存，将分配额外的内存。物理分配内存的成本可能相对较高，因此使用自定义堆分配器来降低这些成本。
+ `foundLostPairsCapacity` 定义了 GPU 粗检测在单个帧中可以产生的最大发现或丢失的对数。这不会限制对的总数，而只会限制在单个帧中可以检测到的新对或丢失的对数。如果在帧中检测到或丢失了更多对，则会发出错误，并且粗检测会丢弃对。

## Performance Considerations
-----------------------------
在具有数千个活动刚体的场景中，GPU 刚体可以提供比 CPU 刚体非常大的性能优势。但是，需要考虑一些性能注意事项。

GPU刚体目前仅加速涉及`convex hulls`和`boxes`的接触生成(针对`boxes`、`convex hulls`、`triangle meshes`和`heightfields`)。如果您大量使用其他形状，例如`capsules`或`spheres`，则涉及这些形状的接触生成将仅在CPU上处理。

`D6 joints`在与 GPU 刚体一起使用时将提供最佳性能。其他关节类型将部分GPU加速，但性能优势将小于`D6 joints`所表现出的性能优势。

具有超过 64 个顶点或每个面具有超过 32 个顶点的`convex hulls`将由 CPU 而不是 GPU 处理其触点，因此，如果可能，请将顶点计数保持在这些限制范围内。可以在`cooking`中定义顶点限制，以确保`cook`好的`convex hulls`不超过这些限制。

如果您的应用程序大量使用触点修改(contact modification)，这可能会限制在 GPU 上执行触点生成的对数。

修改`Actor`的状态会强制将数据重新同步到GPU，例如，如果应用程序调整全局pose，则必须更新`Actor`的`transform`，如果应用程序修改`body`的速度，则必须更新速度等。将数据重新同步到GPU的相关成本相对较低，但应予以考虑。

joint projection、CCD 和触发器等功能不能使用 GPU 加速，而是在 CPU 上处理的。
