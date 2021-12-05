# physx学习

## startup shutdown

### introduction

----

使用`physx sdk`的第一步是初始化一个全局对象。这些对象会在`physx`不在需要这些资源时relese。

### Foundation and Physics

---------

首先我们需要创建`PxFoundation`对象

```c++
static PxDefaultErrorCallback gDefaultErrorCallback;
static PxDefaultAllocator gDefaultAllocatorCallback;

mFoundation = PxCreateFoundation(PX_FOUNDATION_VERSION, gDefaultAllocatorCallback,
    gDefaultErrorCallback);
if(!mFoundation)
    fatalError("PxCreateFoundation failed!");
```

每一个`physx`需要一个可用的`PxFoundation`实例。需要传入的参数是一个version ID、一个allocator callback和一个error callback。`PX_PHYSICS_VERSION`是一个定义在头文件中的宏，，它能够检查头文件个相关联的SDK DLLs是否象匹配。

通常来说，allocator callback和error callback是特定的，`physx`为了让我们能够轻松地开始也提供了一个默认的实现。



接下来我们需要创建最顶层的`PxPhysics`对象

```c++
bool recordMemoryAllocations = true;

mPvd = PxCreatePvd(*gFoundation);
PxPvdTransport* transport = PxDefaultPvdSocketTransportCreate(PVD_HOST, 5425, 10);
mPvd->connect(*transport,PxPvdInstrumentationFlag::eALL);


mPhysics = PxCreatePhysics(PX_PHYSICS_VERSION, *mFoundation,
    PxTolerancesScale(), recordMemoryAllocations, mPvd);
if(!mPhysics)
    fatalError("PxCreatePhysics failed!");
```

+ version  ID同样需要传入。

+ `PxTolerancesScale()`能够在不同的scale下标注上下文(author context)并使得`physx`在传入默认对象的情况下能够正常工作。
+ `recordMemoryAllocations`指明是否进行内存分析。
+ PVD实例是可选项，能够开启debug并使用`Physx Visual Debugger`进行分析。

###  Cooking

---------------------

`physx cooking library`提供创建、转换序列化大量数据的工具。根据自己的香炉需求，你可以希望链接cooking library以在运行时处理这些数据。当然你也可以预先将所有数据处理好然后在需要时经数据装入内存中。初始化cooking library的过程如下:

```c++
mCooking = PxCreateCooking(PX_PHYSICS_VERSION, *mFoundation, PxCookingParams(scale));
if (!mCooking)
    fatalError("PxCreateCooking failed!");
```

`PxCookingParams`是一个对cooking library进行配置的结构体，它将cooking library指向不同的平台（target different platform）、是否使用非默认的tolerances以及是否创建可选的输出。注意，在工程中需要使用一致的PxTolerancesScale。

cooking library使用streaming interface来生成数据。在例子中，`PxToolKit`库提供了streams的实现，streams用于文件和buffer的读写。

通过使用`PxPhysicsInsertionCallback`，高程信息(Heightfield)或者三角网格(Trianglemesh)cooking的网格能够不通过序列化之间插入`PxPhysics`中。这种情况下我们必须使用默认的回调，默认回调通过`PxPhysics::getPhysicsInsertionCallback()`获取。

## Extension

extensions library 包含了很多方法，这些方法对于很多用户都有用，但是某些用户可能出于代码量的大小或者避免使用某些子系统的原因省略掉不使用。初始化extensions library需要`PxPhysics`对象

```c++
if (!PxInitExtensions(*mPhysics, mPvd))
    fatalError("PxInitExtensions failed!");
```

### Optional SDK Components

---------------------------

当我们在内存受限的平台上将`physx`作为静态库链接时，我们可以避免链接部分`physx`的feature来节约内存。目前可供原则的部分为

+ Articulations
+ Height Fields
+ Cloth
+ Particles

如果你的程序需要这些功能的子集，建议你调用`PxCreateBasePhysics`而不是`PxCreatePhysics`并且手动注册你所需要的组件。下面的例子就注册了部分功能：

```c++
physx::PxPhysics* customCreatePhysics(physx::PxU32 version,
    physx::PxFoundation& foundation,
    const physx::PxTolerancesScale& scale,
    bool trackOutstandingAllocations
    physx::PxPvd* pvd)
{
    physx::PxPhysics* physics = PxCreateBasePhysics(version, foundation, scale,
        trackOutstandingAllocations, pvd);

    if(!physics)
        return NULL;

    PxRegisterArticulations(*physics);
    PxRegisterHeightFields(*physics);

    return physics;
}
```

注意只有当将`physx`作为静态链接库时才能节省内存，因为我们依靠链接器来去除未使用的代码。



### Delay-Loading DLLs

------------------------------

在`PhysX``PhysXCooking``PhysXCommon`和`PxPvdSDK`项目中，`PhysXCommon DLL` `PxFoundation DLL`和``PxPvdSDK DLL`被标记为延迟装载(Delay-Loading)

#### PhysXCommon DLL and PxFoundation DLL load

应用程序链接到PhysXCommon DLL，会在装在其他DLL之前先装载`PxFoundation.dll`、`PxPvdSDK `和`PhysXCommon.dll`。应用程序装载的DLL必须和`PhysX`和`PhysXCooking DLLs`使用的一样。在`PhysX`和`PhysXCooking DLLs`中关于`PhysXCommon``PxFoundation`和`PxPvdSDK`由如下规则构成：

+ 如果延迟加载hook指定了`PhysXCommon`名称，则使用用户提供的`PhysXCommon`或`PxPvdSDK`名称
  + 如果延迟加载hook没有指定，则会使用相应的`PhysXCommon``PxFoundation`或者`PxPvdSDK`

#### PxDelayLoadHook

`PxDelayLoadHook`支持装载不同版本的`PhysXCommon DLL`` PxFoundation DLL`或者` PxPvdSDK DLL`。通过自定义的`PxDelayLoadHook`的子类向`PhysX SDK`提供不同的DLL名字来实现，见如下的例子

```c++
class SampleDelayLoadHook: public PxDelayLoadHook
{
    virtual const char* getPhysXCommonDEBUGDllName() const
        { return "PhysX3CommonDEBUG_x64_Test.dll"; }
    virtual const char* getPhysXCommonCHECKEDDllName() const
        { return "PhysX3CommonCHECKED_x64_Test.dll"; }
    virtual const char* getPhysXCommonPROFILEDllName() const
        { return "PhysX3CommonPROFILE_x64_Test.dll"; }
    virtual const char* getPhysXCommonDllName() const
        { return "PhysX3Common_x64_Test.dll"; }
    virtual const char* getPxFoundationDEBUGDllName() const
        { return "PxFoundationDEBUG_x64_Test.dll"; }
    virtual const char* getPxFoundationCHECKEDDllName() const
        { return "PxFoundationCHECKED_x64_Test.dll"; }
    virtual const char* getPxFoundationPROFILEDllName() const
        { return "PxFoundationPROFILE_x64_Test.dll"; }
    virtual const char* getPxFoundationDllName() const
        { return "PxFoundation_x64_Test.dll"; }
    virtual const char* getPxPvdSDKDEBUGDllName() const
        { return "PxPvdSDKDEBUG_x64_Test.dll"; }
    virtual const char* getPxPvdSDKCHECKEDDllName() const
        { return "PxPvdSDKCHECKED_x64_Test.dll"; }
    virtual const char* getPxPvdSDKPROFILEDllName() const
        { return "PxPvdSDKPROFILE_x64_Test.dll"; }
    virtual const char* getPxPvdSDKDllName() const
        { return "PxPvdSDK_x64_Test.dll"; }
} gDelayLoadHook;
```

接下来需要设置`PhysX`, `PhysXCooking`, `PhysXCommon`, `PxPvdSDK`的hook

```c++
PxSetPhysXDelayLoadHook(&gDelayLoadHook);
PxSetPhysXCookingDelayLoadHook(&gDelayLoadHook);
PxSetPhysXCommonDelayLoadHook(&gDelayLoadHook);
PxPvdSetFoundationDelayLoadHook(&gDelayLoadHook);
```

####  PxGpuLoadHook

PxGpuLoadHook类支持装载不同版本的DLL。可以通过自定义的`PxGpuLoadHook`子类向`PhysX SDK`提供不同的DLL名字来实现，见下

```c++
class SampleGpuLoadHook: public PxGpuLoadHook
{
    virtual const char* getPhysXGpuDEBUGDllName() const
        { return "PhysX3GpuDEBUG_x64_Test.dll"; }
    virtual const char* getPhysXGpuCHECKEDDllName() const
        { return "PhysX3GpuCHECKED_x64_Test.dll"; }
    virtual const char* getPhysXGpuPROFILEDllName() const
        { return "PhysX3GpuPROFILE_x64_Test.dll"; }
    virtual const char* getPhysXGpuDllName() const
        { return "PhysX3Gpu_x64_Test.dll"; }
} gGpuLoadHook;
```

设置`PhysX`的hook

```c++
PxSetPhysXGpuLoadHook(&gGpuLoadHook);
```

####  PhysXCommon Secure Load

所有由Nvidia分布的`PhysX DLL`都被签名过。通过`PhysX`或者`PhysXCooking`装载时，会检查`PhysXCommon DLL`的签名。如果签名验证失败会终止应用程序。

### Shutting Down

调用`release()`方法能够dispose任何的`PhysX`对象。这个方法会销毁对象和其包含的所有对象。具体的行为取决于dispose的对象。

关闭extension library需要调用`PxCloseExtension()`。

关闭`physics`，需要在PxPhysics对象上调用release()，它会清理所有的physics对象。

```c++
mPhysics->release();
```

也不要忘记释放foundation对象，最后释放

```c++
mFoundation->release();
```





## Threading

### Introduction

-------------

本章介绍了在多线程应用程序中如何使用`PhysX`。主要有三个方面使用多线程：

+ 如何在不产生竞争的情况下调用`PhysX API`进行读写
+ 如何使用多线程加速模拟过程
+ 如何进行同步模拟，以及如何在模拟过程中进行读和写

### Data Access from Multiple Threads

由于各种原因，`PhysX`不会在内部锁定应用程序对其数据结构的访问，所以当我们在多线程应用中调用API时需要小心。规则如下：

+ 标记为'const'的API接口方法是只读的， 其他的接口是写的。
+ 多线程下可以同时调用多个只读接口。
+ 不同场景中的对象能够通过不同的线程安全地获取。
+ 场景外的不同对象可以从不同的线程安全地访问，注意访问一个对象可能会通过持久性引用间接获取另一个对象。（一个actor引用一个shape，一个shape引用一个mesh，joint和actors相互引用）

不符合上述规则的访问模式可能会导致数据损坏、死锁或程序崩溃。注意在实践中不能一边对场景中的对象进行写操作一边对相同场景中的对象进行读操作。

构建好的工程(check build)包含跟踪应用程序线程对场景中对象的访问的代码，他们会在用户进行非法调用API时进行检测。

#### Scene Locking

每一个`PxScene`对象提供了多个reader、一个writer来在多线程中控制场景的读写。他们适用于在多个系统间共享`PhysX scene`的情况。例如在APEX和游戏引擎中的物理代码。`scene lock`提供了一个在这些系统中定位彼此的方式。

锁不是强制使用的。如果对场景的所有访问都来自单个线程，则使用锁会增加不必要的开销。甚至在多线程下访问场景时，你也可以通过一个更简单或更快速的特定应用程序的机制进行同步来保证你的应用程序避免这些问题。但是使用`scene lock`由如下的好处：

+ 如果设置了`PxSceneFlag::eREQUIRE_RW_LOCK`，构建好的工程(check build)将阵对未首先获取锁而进行的任何API调用或者仅在获取读取锁进行写入时发出警告。
+ APEX SDK使用`scene lock`保证在应用程序中安全地共享场景。

四个方法可以获取/释放锁：

```c++
void PxScene::lockRead(const char* file=NULL, PxU32 line=0);
void PxScene::unlockRead();

void PxScene::lockWrite(const char* file=NULL, PxU32 line=0);
void PxScene::unlockWrite();
```

####  Locking Semantics

关于`scene lock`地使用有精确地描述：

+ 多线程可以同时读。
+ 只能有一个线程写，读是不能写。
+ 一个线程持有写锁它能够同时读和写。
+ 支持Re-entrant 读锁，一个线程已经获取一个读锁仍然允许调用`lockRead()`。每一个`lockRead()`必须有一个`unlockRead()`。
+ 支持Re-entrant写锁， 一个线程已经获取一个写锁仍然允许调用`lockWrite()`。每一个`lockWrite()`必须有一个`unlockWrite()`。
+ 允许已经获取了写锁的线程调用`lockRead()`，线程将仍然能读和写。每一个`lock*()`必须有`unlock*()`。
+ 不支持lock upgrading。不允许持有读锁的线程`lockWrite()`。在checked build中这样做会报错，在release build中会导致deadlock。
+ Writers are favored。如果线程获取到读锁后`lockWrite()`会阻断所有的reader离开。如果新的reader在writer线程被阻塞时到达，它们(reader)将被置于睡眠，并且writer先访问场景。
+ 如果多个writer在队列中，第一个writer优先级最高，后续的writer将根据OS的scheduling被授予访问权限。

注意：*PxScene::release()* 会自动尝试获取写锁，不需要在调用release()之前手动获取写锁。

#### Locking Best Practices

安排应用程序一次性获取锁以执行多个操作通常很有用。这最大限度地减少了锁的开销，此外还可以防止诸如在一个做sweep test的线程中看到布娃娃在另一个线程进行部分插入之类的情况。

群集写入还有助于减少对锁的争用，因为获取写锁会使尝试读的任何线程停止。

### Asynchronous Simulation

---------------

`PhysX`默认进行异步模拟。通过调用`simulate()`开始模拟

```c++
scene->simulate(dt);
```

当调用返回时，模拟的step已经在其他线程中开始进行。你仍然可以在模拟进行时调用API。当这些调用影响到模拟的状态时，结果会在模拟step完成后被缓存并于模拟的结果进行协调。

为了等待到模拟结束，可以调用：

```c++
scene->fetchResults(true);
```

其中的布尔值表示调式是否应该等待模拟完成，或者在当前的状态下立即返回。

需要区分两种访问数据的slot：

+ 在fetchResults之后，下一个simulate之前。
+ 在simulate后还没有调用fetchResults前。

在第一种slot中，模拟并没有运行，对于对象资源的读写没有限制。例如：改变对象的位置会被立即生效，写一次scene query或simulate step会使用新的状态。

在第二种slot中，模拟正在运行且在读写数据。来自用户的并发访问会修改对象的状态或者导致数据竞争或导致模拟代码中的试图不一致（inconsistent views in the simulation code）。因此，模拟中的代码中的对象受到保护，不会受到API写入的影响，并且模拟更新的任何属性都会被缓冲以允许API读取。

注意`simulate()``fetchResults()`是写调用，被调用禁止对于任何对象的访问。

#### Double Buffering

----------------------------

`PhysX`支持在模拟运行时对场景中的对象进行读写操作。这包括添加/移除到场景。

从用户的角度来看，API 更改会立即反映出来。例如：如果设定了刚体的速度并进行查询，则会返回新的速度。相似的，如果在模拟进行中时创建了对象，它能够被其他任何对象访问或者修改。当然了，这些改变都在缓存中所以模拟的代码指挥看到调用`PxScene::simulate()`时的对象状态。改变对象filter data只会影响下一步模拟step，运行中的模拟对正在运行的step会忽略掉。

当`PxScene::fetchResults()`被调用时，任何buffer的改变会被flush：模拟对对象的改变会被反射到API视图上，API的改变会在模拟的下一个step中可见。用户的修改会更优先：当模拟运行时用户修改了一个对象的position，则用户修改的数据会覆盖模拟的结构写入结果中。延迟应用更新不会影响场景查询，场景查询始终会考虑最新的更改。

#### Events involving removed objects

-------------

模拟运行时删除对象或者将对象移除出场景会影响发送给`PxScene::fetchResults()`的事件。

+ 运行期间如果一个对象被删除或者移除将不会移除出`PxSimulationEventCallback::onWake()``PxSimulationEventCallback()::onSleep()`
+ 但是会移除出`PxSimulationEventCallback::onContact()``PxSimulationEventCallback::onTrigger()`。对象将被标记为`PxContactPairHeaderFlag::eREMOVED_ACTOR_0``PxContactPairFlag::eREMOVED_SHAPE_0``PxTriggerPairFlag::eREMOVED_SHAPE_TRIGGER`。此外，如果为包含已删除/已移除对象的对请求 `PxPairFlag::eNOTIFY_TOUCH_LOST`，`PxPairFlag::eNOTIFY_THRESHOLD_FORCE_LOST`事件，则将创建这些事件。

#### Support

不是所有的`PhysX`对象有所有的buffering support。API 文档中提到了在模拟过程中无法运行的操作，SDK 会中止此类操作并报告错误。最重要的exceptions如下：

+ Particle：在模拟运行时粒子数据不能被读取和修改，包括读写粒子的位置或速度，创建删除例子，加力等等。
+ Cloth：唯一允许的双缓冲(double buffered)操作是创建删除cloth和将他添加移除出场景。

#### Memory Considerations

在模拟运行时，用于存储对象更改的缓冲区是按需创建的。如果内存使用时出现的问题超过了与模拟并行读取/写入对象的好处，则不要在模拟运行时写入对象。

### Multithreaded Simulation

----------

`PhysX`拥有一个管理CPU和GPU的计算资源的任务系统。任务通过依赖关系创建，这样能够按照给定顺序解析。解析完成后，他们将会提交给用户实现的调度程序执行。中间件产品（Middleware products）通常不希望创建 CPU 线程供自己使用，在执行线程可能产生大量开销的控制台上尤其如此。在任务模型（task model）中，计算工作（computational work）被分解为job，这些jobs在准备运行时提交到游戏的线程池中。以下类组成了 CPU 任务管理：

#### TaskManager

----------

TaskManager管理任务间相关性，并将就绪任务分派给各自的dispatcher。存在一个用于分配给 TaskManager 的 CPU 任务和 GPU 任务的dispatcher。TaskManager通过SDK创建并持有，每一个`PxScene`会分配一个自己的TaskManager实例，用户可以通过 `PxSceneDesc `或直接通过 `TaskManager `界面使用dispatcher进行配置。

#### CpuDispatcher

----------

CpuDispatcher 是 SDK 用于与应用程序的线程池交互的抽象类，通常，整个应用程序将有一个 CpuDispatcher，因为很少需要多个线程池。一个 CpuDispatcher 实例可能由多个 TaskManager 共享，例如，如果正在使用多个场景。`PhysX `包含默认的 CpuDispatcher 实现，但我们更喜欢应用程序自己实现此类，这样 `PhysX `和 `APEX `可以有效地与应用程序共享 CPU 资源。

**注意：TaskManager 将从 API 调用的上下文（scene::simulate()）或其他正在运行的任务中调用 CpuDispatcher::submitTask()，因此该函数必须是线程安全的。**

CpuDispatcher 接口的实现必须对每个提交的任务调用以下两个方法，才能正确运行:

```c++
baseTask->run();    // optionally call runProfiled() to wrap with PVD profiling events
baseTask->release();
```

`PxExtensions `库具有所有调度程序类型的默认实现，以下代码片段取自 `SampleParticles` 和 `SampleBase`，并显示如何创建默认调度程序。

```c++
PxSceneDesc sceneDesc(mPhysics->getTolerancesScale());
    [...]
    // create CPU dispatcher which mNbThreads worker threads
    mCpuDispatcher = PxDefaultCpuDispatcherCreate(mNbThreads);
    if(!mCpuDispatcher)
        fatalError("PxDefaultCpuDispatcherCreate failed!");
    sceneDesc.cpuDispatcher = mCpuDispatcher;
#if PX_WINDOWS
    // create GPU dispatcher
    PxCudaContextManagerDesc cudaContextManagerDesc;
    mCudaContextManager = PxCreateCudaContextManager(cudaContextManagerDesc);
    sceneDesc.gpuDispatcher = mCudaContextManager->getGpuDispatcher();
#endif
    [...]
    mScene = mPhysics->createScene(sceneDesc);
```

**注意：如果线程数小于或等于正在运行的平台的可用硬件线程，则通常会获得最佳性能，因此创建比硬件线程更多的工作线程通常会导致性能下降。对于具有单个执行内核的平台，可以使用零工作线程创建 CPU 调度程序 （PxDefaultCpuDispatcherCreate(0))）。在这种情况下，所有工作都将在调用PxScene：：simulate())的线程上执行，这比使用多个线程更有效。**

#### CpuDispatcher Implementation Guidelines

-----------

在场景的 TaskManager 找到一个ready-to-run的任务并将其提交给相应的调度程序后，由调度程序实现来决定如何以及何时运行该任务。通常在游戏场景中，刚体模拟是时间关键型的，目标是减少从 simulate() 到 fetchResults() 完成的延迟。当 `PhysX` 任务在更新期间对 CPU 资源具有独占访问权限时，将实现尽可能低的延迟，但是实际上，PhysX将不得不与其他游戏任务共享计算资源。下面是一些准则，可帮助确保在将 PhysX 更新与其他工作混合时在吞吐量和延迟之间取得平衡：

+ 避免长时间运行的任务与 `PhysX `任务交错，这将有助于减少延迟。
+ 避免将工作线程分配给与优先级较高的线程相同的执行核心。如果在执行期间切换了 PhysX 任务的上下文，则处理刚体的pipeline的其余部分可能会停止，从而增加延迟。
+ PhysX偶尔会提交任务，然后立即等待它们完成，因此，以后进后出（stack）顺序执行任务可能比FIFO（queue）顺序执行任务更好。
+ PhysX 不是一个完全并行的 SDK，因此交错执行中小粒度任务（interleaving small to medium granularity tasks）通常会导致更高的总体吞吐量。
+ 如果您的线程池具有每线程作业队列（per-thread job-queues），则在它们提交的线程上排队任务（queuing tasks on the thread they were submitted ）可能会导致更优化的 CPU 缓存一致性，但这不是必需的。

#### BaseTask

------------------------

`BaseTask`是所有任务类型的抽象基类。所有任务的 run() 函数都将在应用程序线程上执行，因此它们需要小心其堆栈用法，尽可能使用少量堆栈，并且堆栈永远不应该出于任何原因阻塞。

#### Task

-----------

`Task`类时标准的task类型。每个模拟step都必须将`Task`提交给 TaskManager 才能执行这些任务。`Task`可以在提交时命名，这样它们能被发现。提交`Task`时，将为其提供引用计数 1，并且 TaskManager::startSimulation() 函数会递减所有任务的引用计数，并调度引用计数为零的所有任务。在调用 TaskManager::startSimulation() 之前，`Task`可以设置彼此之间的依赖关系，以控制它们的调度顺序。一旦模拟开始，仍然可以提交新任务并添加依赖项，但由程序员来避免race hazards。您不能向已分派的任务添加依赖关系，并且新提交的任务的引用计数必须递减，然后才能允许执行该任务。

此外还可以使用任务名称定义同步点（Synchronization points），`TaskManager`将为该名称分配一个不带任务实现(Task implementation)的TaskID。当满足所有命名 TaskID 的依赖项时，它将递减具有该名称的所有任务的引用计数。`APEX `几乎专门使用 `Task `类来管理 CPU 资源。`ApexScene `定义了许多命名任务，模块使用这些任务来计划自己的任务（例如：在 LOD 计算完成后开始，在 PhysX 场景逐步执行之前完成）。

#### LightCpuTask

`LightCpuTask `是 `BaseTask `的另一个子类，由程序员显式调度。LightCpuTasks 在初始化时的引用计数为 1，因此在调度之前必须递减其引用计数。LightCpuTasks 在初始化时递增其延续任务引用计数（continuation task reference count），并在释放时递减引用计数（在完成其 run（） 函数之后）。

PhysX 3.x 几乎专门使用 `LightCpuTasks `来管理 CPU 资源。例如，模拟更新的每个阶段可能由多个并行任务组成，当每个任务都完成执行时，它将减少更新链中下一个任务的引用计数，然后，当其引用计数达到零时，将自动调度以执行。

**注意：即使仅使用 `LightCpuTasks `来管理 CPU 资源，也必须在每个模拟步骤中调用 `TaskManager startSimulation()` 和 `stopSimulation()` 来保持 `GpuDispatcher `的同步。**

以下代码片段显示了 `SampleSubmarine `中的Crab A.I. 如何作为 CPU 任务运行。通过这样做，Crab A.I.作为后台任务与`PhysX`模拟更新并行运行。对于不需要处理多个延续的 CPU 任务，可以对 `LightCpuTask` 进行子类化。`LightCpuTask` 子类要求定义 `getName `和 `run `方法：

```c++
class Crab: public ClassType, public physx::PxLightCpuTask, public SampleAllocateable
{
public:
    Crab(SampleSubmarine& sample, const PxVec3& crabPos, RenderMaterial* material);
    ~Crab();
    [...]

    // Implements LightCpuTask
    virtual  const char*    getName() const { return "Crab AI Task"; }
    virtual  void           run();

    [...]
}
```

在调用 `PxScene::simulate() `并启动模拟后，应用程序在每个 Crab 任务上调用 `removeReference()`，这反过来又会导致它被提交给 `CpuDispatcher `进行更新。请注意，也可以直接向调度程序提交任务（无需操作引用计数），如下所示：

```c++
PxLightCpuTask& task = &mCrab;
mCpuDispatcher->submitTask(task);
```

一旦 `CpuDispatcher `排队等待执行，线程池的一个工作线程最终将调用任务的 `run `方法。在此示例中，Crab 任务将对场景执行光线投射并更新其内部状态机：

```c++
void Crab::run()
{
    // run as a separate task/thread
    scanForObstacles();
    updateState();
}
```

在` simulate()` 运行时，从多个线程执行 API 读取调用（如场景查询）是安全的。但是，必须注意不要重叠来自多个线程的 API 读取和写入调用。在这种情况下，SDK 将发出错误，有关详细信息，请参阅线程处理。



显式引用计数修改和任务相关性设置的示例：

```c++
// assume all tasks have a refcount of 1 and are submitted to the task manager
// 3 task chains a0-a2, b0-b2, c0-c2
// b0 shall start after a1
// the a and c chain have no dependencies and shall run in parallel
//
// a0-a1-a2
//      \
//       b0-b1-b2
// c0-c1-c2

// setup the 3 chains
for(PxU32 i = 0; i < 2; i++)
{
    a[i].setContinuation(&a[i+1]);
    b[i].setContinuation(&b[i+1]);
    c[i].setContinuation(&c[i+1]);
}

// b0 shall start after a1
b[0].startAfter(a[1].getTaskID());

// setup is done, now start all task by decrementing their refcount by 1
// tasks with refcount == 0 will be submitted to the dispatcher (a0 & c0 will start).
for(PxU32 i = 0; i < 3; i++)
{
    a[i].removeReference();
    b[i].removeReference();
    c[i].removeReference();
}
```





## Geometry

### introduction

这部分主要讨论`Physx`的几何类。`geometry`主要用来构建刚体的形状，刚体的形状主要用来作碰撞的触发和physx中的屏幕查询系统的量。Physx也提供独立的function用来做两个`geometry`的相交性检测，射线扫描和几何体扫描（sweep geometry）。

`geometry`是值类型，并且所有的`geometry`类都继承自共同的基类`PxGeometry`。每一个`geometry`类都定义了一个可以定位的体积或者表面。`transform`指明了`geometry`在当前帧的位置。`Physx`提供了对于`plane`和`capsule`几何类型的`helper function`来通过相同的可替代的表达构建他们的`transform`。

`Geometry`分为两类：

+ 包含所有数据的原始结构，包括`PxBoxGeometry`、`PxCapsuleGeometry`、`PxPlaneGeometry`
+ 一个包含指向更多数据指针的结构。包括网格和高度场`PxConvexMeshGeometry`、`PxTriangleMeshGeometry`、`PxHeightFieldGeometry`。指针指向的大数据为`PxConvexMesh`、`PxTriangleMesh`、`PxHeightField`。我们能在每一个被`PxGeometry Type`引用的大数据中使用不同的比例。大数据必须使用`cooking process`来创建。

当我们使用这些`geometry`作为`simulation geometry`来传入或者传出SDk时，`geometry`会拷贝进或者复制出`PxShape Class`。这样在不知道`geometry`的类型的情况下取回是非常危险的，`Physx`提供了一个`union-like wrapper class(PxGeometryHolder)`来通过`geometry`的值传递类型。每一个网格（或者高度场）有一个`reference count`来追踪指向mesh的`PxShape`的编号。

### Geometry Type

---

#### Spheres

---



<img src=".\image\Geo_1.PNG" alt="Geo_1" style="zoom:100%;" />

`PxSphereGeometry `由一个属性（即其半径）指定，并以原点为中心。

#### Capsule

---

<img src=".\image\Geo_2.PNG" alt="Geo_2" style="zoom:75%;" />

`PxCapsuleGeometry`的中心在原点。它通过声明半径和X轴向两边伸长的半程高来创建。

为了创建一个`geometry`为向上站立的`dynamic actor`，`shape`需要一个相对的`transform`。这个`transform`绕着Z轴旋转四分之一个圆。这样`capsule`就通过Y轴来延展`actor`。设置`shape`和`actor`和`sphere`是一样的。

```c++
PxRigidDynamic* aCapsuleActor = thePhysics->createRigidDynamic(PxTransform(position));
PxTransform relativePose(PxQuat(PxHalfPi, PxVec(0,0,1)));
PxShape* aCapsuleShape = PxRigidActorExt::createExclusiveShape(*aCapsuleActor,
    PxCapsuleGeometry(radius, halfHeight), aMaterial);
aCapsuleShape->setLocalPose(relativePose);
PxRigidBodyExt::updateMassAndInertia(*aCapsuleActor, capsuleDensity);
aScene->addActor(aCapsuleActor);

```

函数`PxTransformFromSegment`将定义`capsule`轴的线段转换成`transform`和`halfheight`。

#### Boxs

---

<img src=".\image\Geo_3.PNG" alt="Geo_3" style="zoom:100%;" />

`pXBoxGeometry`有三个attribute

```c++
PxShape* aBoxShape = PxRigidActorExt::createExclusiveShape(*aBoxActor,
    PxBoxGeometry(a/2, b/2, c/2), aMaterial);
```

a、b 和 c 是生成的vox的边长。

#### Planes

---

![Geo_4](.\image\Geo_4.PNG)

`plane`将空间分为上面和下面。下面的东西都会与`plane`产生碰撞

平面一般与YZ平面重合的平面且向朝上的`pointer`指向X正半轴。我们可以使用`PxTransformFormPlaneEquation()`将平面方程转换成等价的`transform`。

`PxPlaneEquationFromTransform()`提供相反的转换。

`PxPlaneGeometry`没有attribute，因为形状的位置完全定义了平面的碰撞体积。

`PxPlaneGeometry`的shape只能为静态actor创建。

#### Convex Meshes

---

<img src=".\image\Geo_5.png" alt="Geo_5" style="zoom:150%;" />

对于一个多边形，如果给定两个其内部的点，这两点的连线一定在这个多边形的内部，那么我们说这个多边形是凸多边形。`PxConvexMesh`是由一系列的顶点和多边形面表示的凸多边形。`Physx`中凸多边形的顶点和面的数量限制在255内。

创建一个`PxConvexMesh`需要cooking。这里我们假设cooking library已经被初始化。接下来我们来看看如何创建`square pyramid`。

+ 首先定义凸包的顶点

  ```c++
  static const PxVec3 convexVerts[] = {PxVec3(0,1,0),PxVec3(1,0,0),PxVec3(-1,0,0),PxVec3(0,0,1),
      PxVec3(0,0,-1)};
  ```

  凸包数据的描述布局结构为

  ```c++
  PxConvexMeshDesc convexDesc;
  convexDesc.points.count     = 5;
  convexDesc.points.stride    = sizeof(PxVec3);
  convexDesc.points.data      = convexVerts;
  convexDesc.flags            = PxConvexFlag::eCOMPUTE_CONVEX;
  ```

+ 现在我们可以使用`cooking library`来构建`PxConvexMesh`

  ```c++
  PxDefaultMemoryOutputStream buf;
  PxConvexMeshCookingResult::Enum result;
  if(!cooking.cookConvexMesh(convexDesc, buf, &result))
      return NULL;
  PxDefaultMemoryInputData input(buf.getData(), buf.getSize());
  PxConvexMesh* convexMesh = physics->createConvexMesh(input);
  ```

+ 最后我们使用`PxConvexMeshGeometry`来创建形状。`PxConvexMeshGeometry`声明网格的实例

  ```c++
  PxShape* aConvexShape = PxRigidActorExt::createExclusiveShape(*aConvexActor,
      PxConvexMeshGeometry(convexMesh), aMaterial);
  ```



我们也可以不进行序列化直接cook`PxConvexMesh`然后将`PxConvexMesh`添加进`PxPhysics`。如果需要实时cooking时这很有用。当然，还是建议使用离线的cooking和streams。下面的代码展示了如何提升cooking的速度

```c++
PxConvexMeshDesc convexDesc;
convexDesc.points.count     = 5;
convexDesc.points.stride    = sizeof(PxVec3);
convexDesc.points.data      = convexVerts;
convexDesc.flags            = PxConvexFlag::eCOMPUTE_CONVEX | PxConvexFlag::eDISABLE_MESH_VALIDATION | PxConvexFlag::eFAST_INERTIA_COMPUTATION;

#ifdef _DEBUG
    // mesh should be validated before cooking without the mesh cleaning
    bool res = theCooking->validateConvexMesh(convexDesc);
    PX_ASSERT(res);
#endif

PxConvexMesh* aConvexMesh = theCooking->createConvexMesh(convexDesc,
    thePhysics->getPhysicsInsertionCallback());
```

我们需要在bebug和check build时确认网格的有效性。如果通过不确定的descriptor来创建网格可能导致不确定的行为。提供的`PxConvexFlag::eFAST_INERTIA_COMPUTATION`标志volume integration能够使用SIMD代码路径，这样能够使用更少的代价获得更快的计算速度。

使用者能够在`PxConvexMeshGeometry`中提供一个`per-instance PxMeshScale`。==The scale defaults to identity==。不支持负值。

`PxConvexMeshGeometry`也包含一些能够对convex object进行调整的flags。默认情况下系统会计算convex object周围的近似的边界(bounds)。我们可以使用`PxConvexMeshGeometryFlag::eTIGHT_BOUNDS`，能够得到更紧致的边界。当然，这样做会产生更多的计算，但是在非常多的object相交的场景下这样能得到更好的模拟结果。

`PxConvexMeshGeometry`包含一个叫`maxMargin`的变量。默认设为3.4e38f。如果 `maxMargin `小于 PCM 接触生成（PCM contact gen）计算的边距量，它将为缩小的形状选择最小边距，以使用 GJK 算法执行增量更新。在这种情况下，应用程序可能会注意到顶点碰撞周围的一些伪影。如果 `maxMargin `设置为较小的值，则可能会降低这些伪影的可见性。如果 `maxMargin `设置为零，PCM 将使用 GJK 算法的原始形状，这将导致此方法不会出现伪影。但是，在性能和准确性之间存在权衡。

#### Convex Mesh cooking

-------

`Convex Mesh cooking`将网格数据转换为一种允许 SDK 执行高效的碰撞检测的形式。cooking的输入是通过输入 `PxConvexMeshDesc `结构体定义的。填充此结构体的方法不同，具体取决于是要仅从点云开始，还是从已经具有多面体的顶点和面开始生成`Convex Mesh`。

##### If Only Vertex Points are Provided（只有点云）

当仅提供顶点时，设置 `PxConvexFlag::eCOMPUTE_CONVEX` 标志来计算网格：

```c++
PxConvexMeshDesc convexDesc;
convexDesc.points.count     = 20;
convexDesc.points.stride    = sizeof(PxVec3);
convexDesc.points.data      = convexVerts;
convexDesc.flags            = PxConvexFlag::eCOMPUTE_CONVEX;
convexDesc.maxVerts         = 10;

PxDefaultMemoryOutputStream buf;
if(!cooking.cookConvexMesh(convexDesc, buf))
    return NULL;
```

该算法尝试从源顶点创建`convex mesh`。字段 `convexDesc.vertexLimit` 指定生成的外壳中最大顶点数的限制。当源数据在几何上具有挑战性时，例如，如果它包含许多彼此靠近的顶点，则此例程有时会失败。如果cooking失败，则会向错误流报告错误，并且例程返回 false。\

如果使用` PxConvexFlag::eCHECK_ZERO_AREA_TRIANGLES`，则算法不包括面积小于 `PxCookingParams::areaTestEpsilon` 的三角形。如果算法找不到 4 个没有小三角形的初始顶点，则返回 `PxConvexMeshCookingResult::eZERO_AREA_TEST_FAILED`。这意味着提供的顶点位于非常小的区域内，cooker无法产生有效的hull。`toolkit helper`函数 `PxToolkit::createConvexMeshSafe` 说明了`Convex Mesh cooking`的最可靠策略：首先，它试图在没有inflation的情况下制造`Hull`。如果失败，它会尝试inflation，如果也失败了，则使用AABB或OBB。

建议在原点周围提供顶点，并将转换放在 `PxShape `中，否则添加` PxConvexFlag::eSHIFT_VERTICES`网格计算的flag。如果提供了大量的输入顶点，量化输入顶点可能会很有用，在这种情况下，请使用`PxConvexFlag::eQUANTIZE_INPUT`并设置所需的`PxConvexMeshDesc::quantizedCount`

Convex cooking 支持两种不同的算法：

######  Quickhull Algorithm

此算法不使用inflation。它将创建一个`Convex Hull`，其顶点是原始顶点的子集，并且顶点数保证不超过指定的最大值。

Quickhull 算法执行以下步骤：

+ 清理顶点 - 删除重复项等。
+ 查找包含输入集的顶点子集，不超过`vertexLimit`。
+ 如果达到`vertexLimit`，请围绕输入顶点展开有限的外壳，以确保我们封装所有输入顶点。
+ 计算顶点map table。（每个顶点至少需要 3 个相邻面。）
+ 检查多边形数据 - 验证所有顶点是否都在hull上或其内部。
+ 计算质量和惯性张量，假设密度为 1。
+ Saves data to stream.

构造hull时，添加的每个新顶点必须比 `PxCookingParams::planeTolerance` 距hull更远，如果不是，则丢弃该顶点。

###### Inflation Based Incremental Algorithm

-----------

此算法始终使用 `PxConvexFlag::eINFLATE_CONVEX`标志，并通过 `PxCookingParams::skinWidth` 对hull planes进行膨胀。

Inflation Incremental Algorithm 执行以下步骤：

+ 清理顶点 - 删除重复项等。
+ 查找包含输入集的顶点子集，不超过`vertexLimit`
+ 从生产的封闭Hull中创建平面。
+ 通过定义的 `PxCookingParams::skinWidth` 来膨胀平面。
+ 通过膨胀的平面裁剪AABB并产生新的hull。
+ 计算顶点map table。（每个顶点至少需要 3 个相邻面。）
+ 检查多边形数据 - 验证所有顶点是否都在hull上或其内部。
+ 计算质量和惯性张量，假设密度为 1。
+ Saves data to stream.

请注意，Inflation Incremental Algorithm可以生成具有更多输入顶点的hull.该算法明显慢于Quickhull，并且产生的结果明显不那么稳定。建议使用Quickhull算法。

###### Vertex Limit Algorithms

如果提供了顶点限制，则有两种算法可以处理顶点限制。

默认算法计算完整的外壳，以及输入顶点周围的 OBB。然后用hull平面对该OBB进行切片，直到达到顶点极限。默认算法要求将顶点限制至少设置为 8，并且通常生成的质量比平面平移(plane shifting)产生的结果要好得多。

启用平面移位(plane shifting ) （`PxConvexFlag::ePLANE_SHIFTING`） 后，当达到顶点限制时，hull计算将停止。然后，将hull平面平移以包含所有输入顶点，然后使用新的平面交点生成具有给定顶点限制的最终hull。平面偏移可能会在远离输入点云的点上产生尖锐的边缘，并且不能保证所有输入顶点都在生成的hull内。但是，它可以在顶点限制低至4的情况下使用，因此对于顶点计数非常低的小碎片等情况，它可能是更好的选择。

##### Vertex Points, Indices and Polygons are Provided（顶点、顶点索引和多边形）

------

要创建给定一组输入顶点（凸顶点）和多边形（hullPolygons）的`PxConvexMesh`，请执行以下操作：

```c++
PxConvexMeshDesc convexDesc;
convexDesc.points.count             = 12;
convexDesc.points.stride            = sizeof(PxVec3);
convexDesc.points.data              = convexVerts;
convexDescPolygons.polygons.count   = 20;
convexDescPolygons.polygons.stride  = sizeof(PxHullPolygon);
convexDescPolygons.polygons.data    = hullPolygons;
convexDesc.flags                    = 0;

PxDefaultMemoryOutputStream buf;
if(!cooking.cookConvexMesh(convexDesc, buf))
    return NULL;
```

提供点和面后，SDK 会验证网格并直接创建 `PxConvexmesh`。这是创建`Convex Mesh`的最快方法。请注意，SDK 要求每个顶点至少有 3 个相邻面。否则，不会创建 PCM 的加速结构，如果启用了 PCM，则会导致性能下降。

convex cooking过程中的内部步骤：

+ 计算顶点映射表（vertex map table），每个顶点至少需要 3 个相邻面。
+ 检查多边形数据 - 检查所有顶点是否都在hull上或hull内部。
+ 计算质量和惯性张量，假设密度为 1。
+ Save data to stream。

#### Triangle Meshes

-------------

<img src=".\image\Geo_6.png" alt="Geo_6" style="zoom:100%;" />

与图形三角形网格一样，碰撞三角形网格由顶点和三角形索引的集合组成。创建三角形网格需要使用cooking库。此处假定cooking库已初始化

```c++
PxTriangleMeshDesc meshDesc;
meshDesc.points.count           = nbVerts;
meshDesc.points.stride          = sizeof(PxVec3);
meshDesc.points.data            = verts;

meshDesc.triangles.count        = triCount;
meshDesc.triangles.stride       = 3*sizeof(PxU32);
meshDesc.triangles.data         = indices32;

PxDefaultMemoryOutputStream writeBuffer;
PxTriangleMeshCookingResult::Enum result;
bool status = cooking.cookTriangleMesh(meshDesc, writeBuffer,result);
if(!status)
    return NULL;

PxDefaultMemoryInputData readBuffer(writeBuffer.getData(), writeBuffer.getSize());
return physics.createTriangleMesh(readBuffer);
```

此外，`PxTriangleMesh`可以cooked并直接插入`PxPhysics`中，而无需流序列化。如果需要实时cooking，这很有用。强烈建议使用离线cooking和流。如果需要，如何提高cooking速度的示例：

```c++
PxTolerancesScale scale;
PxCookingParams params(scale);
// disable mesh cleaning - perform mesh validation on development configurations
params.meshPreprocessParams |= PxMeshPreprocessingFlag::eDISABLE_CLEAN_MESH;
// disable edge precompute, edges are set for each triangle, slows contact generation
params.meshPreprocessParams |= PxMeshPreprocessingFlag::eDISABLE_ACTIVE_EDGES_PRECOMPUTE;
// lower hierarchy for internal mesh
params.meshCookingHint = PxMeshCookingHint::eCOOKING_PERFORMANCE;

theCooking->setParams(params);

PxTriangleMeshDesc meshDesc;
meshDesc.points.count           = nbVerts;
meshDesc.points.stride          = sizeof(PxVec3);
meshDesc.points.data            = verts;

meshDesc.triangles.count        = triCount;
meshDesc.triangles.stride       = 3*sizeof(PxU32);
meshDesc.triangles.data         = indices32;

#ifdef _DEBUG
    // mesh should be validated before cooked without the mesh cleaning
    bool res = theCooking->validateTriangleMesh(meshDesc);
    PX_ASSERT(res);
#endif

PxTriangleMesh* aTriangleMesh = theCooking->createTriangleMesh(meshDesc,
    thePhysics->getPhysicsInsertionCallback());
```

索引可以是 16 位或 32 位。此处使用的步幅假定顶点和索引分别是 PxVec3 和 32 位整数的数组，数据布局中没有间隙。

返回的结果枚举 `PxTriangleMeshCookingResult::eLARGE_TRIANGLE` 确实会在网格包含大三角形时警告用户，应进行细分以确保更好的仿真和 CCT 稳定性。

与`height fields`一样，三角网格支持每个三角形的材料索引。要将每个三角形材质用于网格，请在网格描述符中向cooking库提供每个三角形的索引。稍后，在创建 `PxShape `时，请提供一个将网格中的索引值映射到材质实例的表。

##### 	Triangle Mesh cooking

----------

三角形网格cooking过程如下

+ 检查输入顶点的有效性。
+ 焊接顶点并检查三角形尺寸。
+ 为queries创建加速结构。
+ 计算边缘凸性信息和邻接。
+ Save data to stream. 

请注意，网格清理可能导致cooking产生的三角形集与原始输入集不同，网格清理可移除无效三角形（包含超出范围的顶点参照）、重复的三角形和零区域三角形。发生这种情况时，PhysX 可以选择输出一个网格重新映射表（mesh remapping table），该表将每个内部三角形链接到用户数据中的源三角形。

有多个参数可用于控制网格创建。

+ In *PxTriangleMeshDesc*:
  + `materialIndices` 定义每个三角形材质。当三角形网格与另一个对象碰撞时，碰撞点处需要材质。如果 `materialIndices `为 NULL，则使用 `PxShape` 实例的材料。

+ In *PxCookingParams*:
  + `scale` 定义公差刻度用于检查cooking好的的三角形是否不太大。此检查将有助于提高模拟稳定性。
  + `suppressTriangleMeshRemapTable `指定是否创建表面重映射表（face remap table）。否则，这将节省大量内存，但 SDK 将无法提供有关在碰撞、扫描或光线投射命中时击中哪个原始网格三角形的信息。
  + `buildTriangleAdjacies `指定是否创建了三角形邻接信息。可以使用 `getTriangle `检索给定三角形的相邻三角形。
  + `meshPreprocessParams `指定网格预处理参数。
    + `PxMeshPreprocessingFlag::eWELD_VERTICES` 可在三角形网格cooking过程中实现顶点焊接。
    + `PxMeshPreprocessingFlag::eDISABLE_CLEAN_MESH` 禁用网格清理过程。顶点重复不被搜索，巨大的三角形测试不做。顶点焊接未完成。能够加快cooking速度。
    + `PxMeshPreprocessingFlag::eDISABLE_ACTIVE_EDGES_PRECOMPUTE` 禁用顶点边缘预计算。使cooking更快，但减少接触生成（contact generation）。
  + `meshWeldTolerance `- 如果启用了网格焊接，则控制顶点焊接的距离。如果未启用网格焊接，则此值定义网格验证的验收距离（acceptance distance ）。
  + `midphaseDesc`指定所需的中相加速度结构描述符（midphase acceleration structure descriptor）。
    + `PxBVH33MidphaseDesc - PxMeshMidPhase::eBVH33` 是默认结构。它是在最近的`PhysX`版本中使用的版本，直到`PhysX 3.3`。它具有出色的性能，并且在所有平台上都受支持。
    + `PxBVH34MidphaseDesc - PxMeshMidPhase::eBVH34` 是 `PhysX 3.4` 中引入的重新访问的实现。在cooking性能和运行时性能方面，它的速度都明显更快，但目前仅在支持 SSE2 指令集的平台上可用。
+ *PxBVH33MidphaseDesc params*
  + `meshCookingHint `指定网格层次结构构造首选项。与碰撞性能相比，可实现更好的cooking性能，适用于cooking性能比创建最高质量的网格更重要的应用。
  + `meshSizePerformanceTradeOff `指定网格大小和运行时性能之间的权衡。
+ *PxBVH34MidphaseDesc params*:
  + numTrisPerLeaf 指定每片leaf的三角形数。每片leaf的三角形越少，产生的网格越大，运行时性能通常越好，cooking性能越差。

#### Height Fields

--------

<img src=".\image\Geo_7.png" alt="Geo_7" style="zoom:100%;" />

`Height Fields`的局部空间轴为：

+ Row - X axis 
+ Column - Z axis 
+ Height - Y axis 

顾名思义，地形只能通过常规矩形采样网格上的高度值来描述：

```c++
PxHeightFieldSample* samples = (PxHeightFieldSample*)alloc(sizeof(PxHeightFieldSample)*
    (numRows*numCols));
```

每个样本都由一个 16 位整数高度值、两个材质（对于样本矩形中的两个三角形）和一个曲面细分标志组成。

标志和材质是指下方和采样点右侧的采样点，并指示沿哪个对角线将其拆分为三角形，以及这些三角形的材料。特殊的预定义材料 `PxHeightFieldMaterial::eHOLE` 指定高度字段中的孔。有关更多详细信息，请参阅 `PxHeightFieldSample `的参考文档。

<img src=".\image\Geo_8.PNG" alt="Geo_8" style="zoom:75%;" />

| Tesselation flags                                            | Result                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src=".\image\Geo_9.PNG" alt="Geo_9" style="zoom:75%;" /> | <img src=".\image\Geo_10.PNG" alt="Geo_10" style="zoom:75%;" /> |
| <img src=".\image\Geo_11.PNG" alt="Geo_11" style="zoom:75%;" /> | <img src=".\image\Geo_12.PNG" alt="Geo_12" style="zoom:75%;" /> |
| <img src=".\image\Geo_13.PNG" alt="Geo_13" style="zoom:75%;" /> | <img src=".\image\Geo_14.PNG" alt="Geo_14" style="zoom:75%;" /> |

要告诉系统每个方向上的采样高度数，请使用描述符来实例化 `PxHeightField `对象： 

```c++
PxHeightFieldDesc hfDesc;
hfDesc.format             = PxHeightFieldFormat::eS16_TM;
hfDesc.nbColumns          = numCols;
hfDesc.nbRows             = numRows;
hfDesc.samples.data       = samples;
hfDesc.samples.stride     = sizeof(PxHeightFieldSample);

PxHeightField* aHeightField = theCooking->createHeightField(hfDesc,
    thePhysics->getPhysicsInsertionCallback());
```

现在创建一个`PxHeightFieldGeometry`和一个形状：

```c++
PxHeightFieldGeometry hfGeom(aHeightField, PxMeshGeometryFlags(), heightScale, rowScale,
    colScale);
PxShape* aHeightFieldShape = PxRigidActorExt::createExclusiveShape(*aHeightFieldActor,
    hfGeom, aMaterialArray, nbMaterials);
```

行和列刻度告诉系统采样点在关联方向上的距离有多远。高度刻度将整数高度值缩放到浮点范围。

此处使用的 `createExclusiveShape()`的变体为高度字段指定了一个材质数组，该数组将由每个单元格的材料索引进行索引，以解决与该单元格的冲突。可以改用单材料变体，但高度字段材料索引必须全部为单个值或特殊值 `eHOLE`。

使用 `PxHeightFieldFlag::eNO_BOUNDARY_EDGES` 标志可以禁用地形边界处具有三角形边缘的接触生成，从而在排列了多个高度场形状以使其边缘接触时，可以更有效地生成接触。

##### Heightfield cooking

--------------------------

`Heightfield`数据可以在离线状态下进行cooking，然后用于`createHeightField`。cooking预先计算并存储边缘信息。这样可以更快地创建`heightfield`，因为边缘已经预先计算过了。如果需要在运行时中创建`heightfields `，这将非常有用，因为它确实可以显著提高创建`HeightField`的速度。

Heightfield cooking 过程如下：

+ 将`heightfield`样本加载到内部存储器中。
+ 预先计算边缘碰撞信息。
+ Save data to stream. 

##### Unified Heightfields

-------------------

`PhysX`为`heightfields`提供了两种接触生成方法。这些是：

+ 默认统一`heightfield `接触生成。
+ 传统`heightfield`接触生成。

默认的统一高度场接触生成方法（default unified heightfield contact generation approach）从高度场中提取三角形，并利用用于针对三角形网格生成接触的相同低级接触生成代码。如果可互换使用三角形网格或高度场，则此方法可确保等效的行为和性能。但是，通过这种方法，高度场表面没有厚度，因此如果不启用CCD，快速移动的物体可能会被穿过(tunnel)。经典高度场碰撞代码（在以前版本的 PhysX 中是默认的）的工作方式与三角形网格接触生成不同。除了生成与接触高度场表面的形状的接触外，它还生成与表面下方形状的接触。高度场的"厚度"用于控制在表面接触下方生成多远。这是通过沿垂直轴的"厚度"在宽相中挤出高度场的AABB来工作的。将为边界与高度场拉伸边界相交的曲面以下的任何形状生成触点。

通过调用以下命令启用统一高度场联系生成(heightfield contact generation)：

```c++
PxRegisterHeightFields(PxPhysics& physics);
```

通过调用以下命令启用传统高度场联系人生成：

```c++
PxRegisterLegacyHeightFields(PxPhysics& physics);
```

这些调用必须在创建场景之前进行，否则将发出警告。高度场碰撞设置是全局设置，适用于所有场景。

如果调用 `PxCreatePhysics（...）`，这将自动调用 `PxRegisterHeightFields（...）` 来注册默认的统一高度场碰撞方法。如果调用 `PxCreateBasePhysics（...）`，则默认情况下不会注册任何高度场接触生成。如果使用高度场，应用程序必须调用相应的高度场注册函数。

### Deformable meshes

`PhysX`支持可变形网格(`Deformable meshes`)，即其顶点随时间移动的网格（而拓扑，即三角形索引，保持固定）。可变形网格仅支持 `PxMeshMidPhase::eBVH33` 数据结构。由于网格顶点将被更新，因此用户定义的顶点与 PhysX 的内部顶点之间的映射也必须保留。也就是说，`PhysX `不应在cooking过程中对顶点重新排序。因此，应禁用所有可以对顶点重新排序的cooking操作，并且用户有责任确保传递的顶点是正确的，并且禁用了操作。例如，应禁用网格清洁阶段:

```c++
cookingParams.midphaseDesc.setToDefault(PxMeshMidPhase::eBVH33);
    cookingParams.meshPreprocessParams      = PxMeshPreprocessingFlag::eDISABLE_CLEAN_MESH;
```

可以在同一场景中混合`eBVH33`和`eBVH34`网格，因此默认的cooking参数仍可用于不可变形/静态网格。

要修改顶点，请在模拟场景之前使用`PxTriangleMesh::getVerticesForModification()`和`PxTriangleMesh::refitBVH()`函数：

```c++
// get vertex array
PxVec3* verts = mesh->getVerticesForModification();

// update the vertices here
...

// tell PhysX to update the mesh structure
PxBounds3 newBounds = mesh->refitBVH();
```

然后使用`PxScene::resetFiltering()` 作为相应的网格执行器，告诉宽相其边界已被修改：

```c++
scene->resetFiltering(*actor);
```

当网格变形并远离位于其上的物体时，所述网格会在网格上轻微反弹和抖动。对网格形状使用略微负的静止偏移有助于减少此影响：

```c++
PxShape* shape;
mesh->getShapes(&shape, 1);
shape->setRestOffset(-0.5f);   // something negative, value depends on your game's scale
```

这将使对象"沉入"到动态网格中。这样，接触就不会立即丢失，并且运动保持平稳。有关更多详细信息，请参阅 SDK 中的可变形网格片段。

### Mesh Scaling

---------------

共享的 `PxTriangleMesh `或 `PxConvexMesh `在被几何体实例化时可能会被拉伸或压缩。这允许对应用不同比例因子的同一网格进行多次实例化。缩放是使用 `PxMeshScale `类指定的，该类定义要沿 3 个正交轴应用的比例因子。因子大于 1.0 会导致拉伸，而因子小于 1.0 会导致压缩。轴的方向由四元数控制，并在形状的局部框架中指定。支持负网格缩放，负值沿每个相应轴产生相反方向的缩放。此外，当 scale.x*scale.y*scale.z < 0 时，`PhysX` 将翻转网格三角形的法线。

下面的代码创建一个形状，其 `PxTriangleMesh `沿 x 轴缩放 x 倍，沿 y 轴缩放 y，沿 z 轴缩放 z：

```c++
// created earlier
PxRigidActor* myActor;
PxTriangleMesh* myTriMesh;
PxMaterial* myMaterial;

// create a shape instancing a triangle mesh at the given scale
PxMeshScale scale(PxVec3(x,y,z), PxQuat(PxIdentity));
PxTriangleMeshGeometry geom(myTriMesh,scale);
PxShape* myTriMeshShape = PxRigidActorExt::createExclusiveShape(*myActor,geom,*myMaterial);
```

凸网格以类似的方式使用 PxMeshScale 类进行缩放。下面的代码创建一个形状，其 `PxConvexMesh `沿 (sqrt(1/2)、1.0、-sqrt(1/2))缩放 x 因子，沿 (0，1，0) 缩放 y 因子，沿 (sqrt(1/2)、1.0、sqrt(1/2))缩放z 因子 ：

```c++
PxMeshScale scale(PxVec3(x,y,z), PxQuat quat(PxPi*0.25f, PxVec3(0,1,0)));
PxConvexMeshGeometry geom(myTriMesh,scale);
PxShape* myConvexMeshShape = PxRigidActorExt::createExclusiveShape(*myActor,geom,*myMaterial);
```

还可以使用存储在 `PxHeightFieldGeometry `中的比例因子来缩放Height fields。在这种情况下，假定比例尺沿行、列和高度字段的高度方向的轴。缩放在 `SampleNorthPoleBuilder `中的 `SampleNorthPole.cpp` 中演示：

```c++
PxHeightFieldGeometry hfGeom(heightField, PxMeshGeometryFlags(), heightScale, hfScale, hfScale);
PxShape* hfShape = PxRigidActorExt::createExclusiveShape(*hfActor, hfGeom, getDefaultMaterial());
```

在此示例中，沿 x 轴和 z 轴的坐标按 `hfScale `缩放，而样本高度按 `heightScale `缩放。

### PxGeometryHolder

------------------

为`Shape`提供几何图形时，无论是在创建时还是使用 `PxShape::setGeometry()` 时，该几何图形都会复制到 SDK 的内部结构中。如果您知道形状几何形状的类型，则可以直接检索它：

```c++
PxBoxGeometry boxGeom;
bool status = shape->getBoxGeometry(geometry);
```

如果`Shape`的几何图形不是预期类型，则状态返回代码设置为 false。但是，在未首先知道其类型的情况下从`Shape`中检索几何对象通常很方便 - 例如，调用将 `PxGeometry `引用作为参数的函数。`PxGeometryHolder `是一个类似并集的类，它允许按值返回 `PxGeometry `对象，而不管类型如何。它的用法在`PhysXSample.cpp`中的`createRenderObjectFromShape()`函数中进行了说明：

```c++
PxGeometryHolder geom = shape->getGeometry();

switch(geom.getType())
{
case PxGeometryType::eSPHERE:
    shapeRenderActor = SAMPLE_NEW(RenderSphereActor)(renderer, geom.sphere().radius);
    break;
case PxGeometryType::eCAPSULE:
    shapeRenderActor = SAMPLE_NEW(RenderCapsuleActor)(renderer, geom.capsule().radius,
        geom.capsule().halfHeight);
    break;
...
}
```

函数` PxGeometryHolder::any()` 返回对 `PxGeometry `对象的引用。例如，要比较场景中的两个形状是否重叠：

```c++
bool testForOverlap(const PxShape& s0, const PxShape& s1)
{
    return PxGeometryQuery::overlap(s0.getGeometry().any(), PxShapeExt::getGlobalPose(s0),
                                    s1.getGeometry().any(), PxShapeExt::getGlobalPose(s1));
}
```

### Vertex and Face Data

----------

Convex meshes、triangle meshes和height fields都可以查询顶点和面数据。例如，在渲染凸形状的网格时，这特别有用。该方法为：

```c++
RenderBaseActor* PhysXSample::createRenderObjectFromShape(PxShape* shape,
    RenderMaterial* material)
```

在 `PhysXSample.cc` 包含一个 switch 语句，其中包含每个形状类型的大小写，说明了查询顶点和面所需的步骤。可以使用`PxMeshQuery::getTriangle`函数从triangle meshes或height fields中获取有关三角形的信息。您还可以检索给定三角形的相邻三角形索引（三角形的triangleNeighbour[i]共享edge vertex[i]-vertex[(i+1)%3]，其中三角形索引为"triangleIndex"，其中顶点在0到2的范围内）。要启用此功能，三角形网格在构建三角形相邻参数设置为 true 的情况下进行cooking。

#### Convex Meshes

凸网格包含一个顶点数组、一个面数组和一个索引缓冲区，该缓冲区连接每个面的顶点索引。要解压缩Convex meshes，第一步是提取共享Convex meshes：

```C++
PxConvexMesh* convexMesh = geom.convexMesh().convexMesh;
```

然后获取对顶点和索引缓冲区的引用：

```c++
PxU32 nbVerts = convexMesh->getNbVertices();
const PxVec3* convexVerts = convexMesh->getVertices();
const PxU8* indexBuffer = convexMesh->getIndexBuffer();
```

现在循环访问面数组以对它们进行triangulate：

```C++
PxU32 offset = 0;
for(PxU32 i=0;i<nbPolygons;i++)
{
    PxHullPolygon face;
    bool status = convexMesh->getPolygonData(i, face);
    PX_ASSERT(status);

    const PxU8* faceIndices = indexBuffer + face.mIndexBase;
    for(PxU32 j=0;j<face.mNbVerts;j++)
    {
        vertices[offset+j] = convexVerts[faceIndices[j]];
        normals[offset+j] = PxVec3(face.mPlane[0], face.mPlane[1], face.mPlane[2]);
    }

    for(PxU32 j=2;j<face.mNbVerts;j++)
    {
        *triangles++ = PxU16(offset);
        *triangles++ = PxU16(offset+j);
        *triangles++ = PxU16(offset+j-1);
    }

    offset += face.mNbVerts;
}
```

观察多边形的顶点索引从` indexBuffer[face.mIndexBase] `开始，顶点计数由 `face.mNbVerts` 给出。

#### Triangle Meshes

-----------

Triangle meshes包含顶点和索引三元组的数组，这些三元组通过索引到顶点缓冲区来定义三角形。这些数组可以直接从共享Triangle meshes访问：

```c++
PxTriangleMesh* tm = geom.triangleMesh().triangleMesh;
const PxU32 nbVerts = tm->getNbVertices();
const PxVec3* verts = tm->getVertices();
const PxU32 nbTris = tm->getNbTriangles();
const void* tris = tm->getTriangles();
```

索引可以使用 16 位或 32 位值存储，这些值在网格最初cooking时指定。要确定运行时的存储格式，请使用 API 调用：

```c++
const bool has16bitIndices = tm->has16BitTriangleIndices();
```

假设三角形索引以 16 位格式存储，请通过以下方式找到第 i 个三角形的第 j 个顶点：

```c++
const PxU16* triIndices = (const PxU16*)tris;
const PxU16 index = triIndices[3*i +j];
```

对应的顶点是：

```c++
const PxVec3& vertex = verts[index];
```

#### Height Fields

Height Fields样本可以在运行时以矩形块的形式进行修改。在以下代码片段中，我们创建一个 Height Fields 并修改其示例：

```c++
// create a 5x5 HF with height 100 and materials 2,3
PxHeightFieldSample samples1[25];
for (PxU32 i = 0; i < 25; i ++)
{
    samples1[i].height = 100;
    samples1[i].materialIndex0 = 2;
    samples1[i].materialIndex1 = 3;
}

PxHeightFieldDesc heightFieldDesc;
heightFieldDesc.nbColumns = 5;
heightFieldDesc.nbRows = 5;
heightFieldDesc.thickness = -10;
heightFieldDesc.convexEdgeThreshold = 3;
heightFieldDesc.samples.data = samples1;
heightFieldDesc.samples.stride = sizeof(PxHeightFieldSample);

PxPhysics* physics = getPhysics();
PxHeightField* pHeightField = cooking->createHeightField(heightFieldDesc, physics->getPhysicsInsertionCallback());

// create modified HF samples, this 10-sample strip will be used as a modified row
// Source samples that are out of range of target heightfield will be clipped with no error.
PxHeightFieldSample samplesM[10];
for (PxU32 i = 0; i < 10; i ++)
{
    samplesM[i].height = 1000;
    samplesM[i].materialIndex0 = 1;
    samplesM[i].materialIndex1 = 127;
}

PxHeightFieldDesc desc10Rows;
desc10Rows.nbColumns = 1;
desc10Rows.nbRows = 10;
desc10Rows.samples.data = samplesM;
desc10Rows.samples.stride = sizeof(PxHeightFieldSample);

pHeightField->modifySamples(1, 0, desc10Rows); // modify row 1 with new sample data
```

`PhysX `不会保留从Height Fields到引用它的Height Fields Shape的映射。调用 `PxShape::set`引用高度字段的每个形状的几何图形的几何图形，以确保更新内部数据结构以反映新的几何图形：

```c++
PxShape *hfShape = userGetHfShape(); // the user is responsible for keeping track of
                                     // shapes associated with modified HF
hfShape->setGeometry(PxHeightFieldGeometry(pHeightField, ...));
```

另请注意，当对象位于旧几何或新几何体之上时，`PxShape::setGeometry() `不保证正确/连续的行为。方法 `PxHeightField::getTimestamp()` 返回Height Fields被修改的次数。







## Rigid Body Overview

### Introduction

----

本章将介绍使用 `NVIDIA PhysX ` 引擎模拟刚体动力学的基础知识。

### Rigid Body Object Model

-------------

`PhysX` 使用分层刚体对象/actor 模型，如下所示：

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

### The Simulation Loop

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

每当应用完成事件处理并开始空闲时，就会从示例框架中调用此操作。它累积经过的实时时间，直到大于六十分之一秒，然后调用 `simulate()`，它将场景中的所有对象向前移动该间隔时间。这可能是在推进模拟时处理时间的众多不同方法中最简单的方法。

要允许模拟完成并返回结果，只需调用：

```c++
mScene->fetchResults(true);
```

`True` 表示模拟应阻塞，直到`simulate()`完成，以便在返回时保证结果可用。当 fetchResults 完成时，您定义的任何模拟事件回调函数也将被调用。请参阅回调序列一章。在模拟过程中，可以从场景中读取和写入。示例利用这一点与物理场并行执行渲染工作。在 fetchResults（） 返回之前，当前模拟步骤的结果不可用。因此，与模拟并行运行渲染会将演员渲染为调用 `simulate()`前的样子。在 `fetchResults()` 返回后，所有这些函数都将返回新的模拟后的状态。有关在模拟运行时读取和写入的更多详细信息，请参阅线程处理一章。为了使人眼将动画运动感知为平滑，每秒至少使用二十个离散帧，每帧对应于一个物理时间步长。要对更复杂的物理场景进行流畅、逼真的模拟，请至少每秒使用五十帧。

**注意： 如果您正在进行实时交互式模拟，则可能会尝试采用不同大小的时间步长，这些步长对应于自上次模拟帧以来经过的实时量。如果这样做，请非常小心，与采用恒定大小的时间步长不同的是：模拟代码对非常小和很大的时间步长都很敏感，并且对时间步长之间的太大变化也很敏感。在这些情况下，它可能会产生抖动模拟（jittery simulation）。**





## Rigid Body Collision

### Introduction

----------

本节将介绍刚体碰撞的基础知识。

### Shapes

---------

`Shape`描述actor的空间范围（spatial extent ）和碰撞属性（collision properties ）。它们在` PhysX` 中用于三个目的：确定刚性对象的接触特征的相交性测试(intersection tests )、场景查询测试(scene query tests )（如光线投射）以及定义触发体积(defining trigger volumes )（当其他`Shape`与它们相交时生成通知）。

每个`Shape`都包含一个` PxGeometry` 对象和一个对 `PxMaterial` 的引用，这两个对象都必须在创建时指定。下面的代码创建一个具有球体几何图形和特定材质的`Shape`：

```c++
PxShape* shape = physics.createShape(PxSphereGeometry(1.0f), myMaterial, true);
myActor.attachShape(*shape);
shape->release();
```

方法 `PxRigidActorExt::createExclusiveShape()` 等效于上面的三行。

用于 `createShape()` 的参数 "true" 通知 SDK 该`Shape`不会与其他参与者共享。当有许多具有相同几何图形的 `Actor` 时，可以使用形状共享来降低模拟的内存成本，但共享形状具有非常严格的限制：在共享形状附加到 `Actor` 时，您无法更新共享形状的属性。

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

### Simulation Shapes and Scene Query Shapes

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

### Kinematic Triangle Meshes (Planes, Heighfields)

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

### Broad-phase Algorithms

---------

`PhysX` 支持多种宽相算法（broad-phase algorithms）：

+ *sweep-and-prune (SAP)* 
+ *multi box pruning (MBP)* 

`PxBroadPhaseTyp::eSAP` 是 `PhysX 3.2` 之前使用的默认算法。这是一个很好的通用选择，当许多物体处于睡眠状态时，它具有出色的性能。但是，当所有对象都在移动时，或者当在宽相(broad-phase)中添加或删除大量对象时，性能可能会显著下降。此算法不需要定义世界边界(world bounds)即可工作。`PxBroadPhaseType::eMBP` 是 `PhysX 3.3 `中引入的一种新算法。它是一种替代的宽相(broad-phase)算法，当所有对象都在移动或插入大量对象时，它不会遇到与`eSAP`相同的性能问题。但是，当许多对象处于休眠状态时，其通用性能可能不如 eSAP，并且它要求用户定义世界边界(world bounds)才能工作。所需的宽相(broad-phase)算法由`PxSceneDesc`结构中的`PxBroadPhaseType`枚举控制。

### Regions of Interest

------------

感兴趣的区域(Regions of Interest)是围绕宽相(broad-phase)控制的体积空间中的世界空间AABB包围盒。这些区域中包含的对象由宽相(broad-phase)正确处理。落在这些区域之外的对象将丢失所有碰撞检测。理想情况下，这些区域应覆盖整个游戏空间，同时限制覆盖的空白空间的数量。区域可以重叠，但为了获得最大效率，建议尽可能减少区域之间的重叠量。请注意，AABB 仅接触的两个区域不被视为重叠。例如，`PxBroadPhaseExt::createRegionsFromWorldBounds` helper function通过将给定的世界 AABB 细分为常规 2D 网格来创建许多非重叠区域边界。区域可以由`PxBroadPhaseRegion` 结构定义，也可以由分配给它们的用户数据定义。它们可以在场景创建时或在运行时使用 `PxScene::addBroadPhaseRegion` 函数进行定义。SDK 返回分配给新创建区域的句柄，稍后可以使用 `PxScene::removeBroadPhaseRegion` 函数删除区域。新添加的区域可能会重叠已存在的对象。如果设置了 `PxScene::addBroadPhaseRegion` 调用中的 `populateRegion` 参数，SDK 可以自动将这些对象添加到新区域。但是，此操作并不便宜，并且可能会对性能产生很大影响，尤其是在同一帧中添加多个区域时。因此，建议尽可能禁用它。该区域将被创建为空，并且只会在创建区域后添加到场景中的对象填充，或者在更新时（即当它们移动时）使用先前存在的对象填充该区域。请注意，只有`PxBroadPhaseType::eMBP`需要定义区域。`PxBroadPhaseType::eSAP` 算法则不然。此信息在`PxBroadPhaseCaps`结构中捕获，该结构列出了每个宽相(broad-phase)算法的信息和功能。此结构可以通过 `PxScene::getBroadPhaseCaps` 函数检索。有关当前区域的运行时信息可以使用 `PxScene::getNbBroadPhaseRegions` 和 `PxScene::getBroadPhaseRegions` 函数进行检索。区域的最大数量目前限制为 256 个。

### Broad-phase Callback

----------

可以在 `PxSceneDesc` 结构中定义与宽相位相关(broad-phase-related )的事件的回调。当在指定的感兴趣区域（即"越界"）中发现对象时，将调用此`PxBroadPhaseCallback`对象。SDK 会禁用这些对象的冲突检测。一旦对象重新进入有效区域，它就会自动重新启用。由用户决定如何处理越界对象。典型选项包括：

+ 删除对象
+ 让他们继续运动而不会发生碰撞，直到他们重新进入有效区域
+ 手动将他们传送回有效位置

### Collision Filtering

-------------
在几乎所有应用程序中，除了微不足道的应用程序之外，都需要免除某些对象对的相交性（interacting），或者以特定方式为交互对（interacting pair）配置 SDK 冲突检测行为。在潜艇样本中，如上所述，当潜艇接触水雷或水雷链条时，我们需要得到通知，以便我们可以将它们炸毁。crab的AI还需要知道crab何时接触到高度场。在了解示例如何实现此目的之前，我们需要了解 SDK filtering系统的可能性。由于filtering可能交互对（interacting pair）发生在仿真引擎的最深处，并且需要应用于彼此靠近的所有对象对，因此它对性能特别敏感。实现它的最简单方法是始终为每个可能交互的对（interacting pair）调用回调函数，其中应用程序基于两个对象指针可以使用一些自定义逻辑（例如咨询其游戏数据库）确定该对是否应该进行交互。不幸的是，如果对于一个非常大的游戏世界来说，这会变得太慢，特别是如果冲突检测处理发生在远程处理器（如GPU）或具有本地内存的其他类型的矢量处理器上，这些处理器必须暂停其并行计算，中断运行游戏代码的主处理器，并让它在继续之前执行回调。即使它要在CPU上执行，它也可能会在多个内核或超线程上同时执行，并且必须放置线程安全代码以确保对共享数据的并发访问是安全的。更好的做法是使用某种可以在远程处理器上执行的固定函数逻辑。这就是我们在 PhysX 2.x 中所做的 - 不幸的是，我们提供的基于组的简单过滤规则（simple group based filtering rules）不够灵活，无法涵盖所有应用程序。在3.0中，我们引入了一个着色器系统，它允许开发人员使用在矢量处理器上运行的代码实现任意规则系统(因此无法访问主内存中的任何最终游戏数据库(eventual game data base ))，这比2.x固定函数过滤更灵活，但同样高效，还有一个完全灵活的回调机制，其中过滤器着色器调用能够访问任何应用程序数据的CPU回调函数， 以性能为代价 -- 有关详细信息，请参阅 `PxSimulationFilterCallback`。最好的部分是，应用程序可以基于每对(per-pair)进行决策，以进行速度与灵活性的权衡。

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
`SampleSubmarineFilterShader`是一个简单的着色器函数，它是`PxFiltering.h`中声明的`PxSimulationFilterShader`原型的实现。`shader filter function`（如上称为 `SampleSubmarineFilterShader`）可能不引用函数的参数及其自己的局部堆栈变量以外的任何内存，因为该函数可以在远程处理器上编译和执行。`SampleSubmarineFilterShader()` 将用于所有彼此靠近的形状对(pairs of shapes) - 更准确地说：对于在世界空间中发现轴对齐边界框首次相交的所有形状对。除此之外的所有行为都由 `SampleSubmarineFilterShader()` 返回的内容决定。`SampleSubmarineFilterShader())` 的参数包括两个对象的 `PxFilterObjectAttributes` 和 `PxFilterData`，以及一个常量内存块。请注意，指向这两个对象的指针不会传递，因为这些指针引用计算机的主内存，并且正如我们所说，这可能不适用于着色器，因此指针不是很有用，因为取消引用它们可能会导致崩溃。`PxFilterObjectAttributes`和`PxFilterData`旨在包含可以从指针中快速收集的所有有用信息。`PxFilterObjectAttributes`是32位数据，用于编码对象的类型：例如`PxFilterObjectType::eRIGID_STATIC`，`PxFilterObjectType::eRIGID_DYNAMIC`，甚至`PxFilterObjectType::ePARTICLE_SYSTEM`。此外，它还可以让您了解对象是运动学的(kinematic,)还是触发器。`PhysX`中的每个`PxShape`和`PxParticleBase`对象都有一个类型为`PxFilterData`的成员变量。这是 128 位用户定义的数据，可用于存储与冲突过滤(collision filtering)相关的应用程序特定信息。这是传递给每个对象的`SampleSubmarineFilterShader()` 的另一个变量。对于常量块。这是应用程序可以提供给着色器进行操作的每个场景的全局信息块。您将需要使用它来编码有关要过滤的内容和不过滤的内容的规则。最后，`SampleSubmarineFilterShader()` 还有一个 `PxPairFlags` 参数。这是一个输出，类似于返回值 `PxFilterFlags`，尽管用法略有不同。`PxFilterFlags` 告诉 SDK 它是否应该永久忽略该对 （eKILL），在重叠时忽略该对，但在过滤其中一个对象的相关数据更改时再次询问 （eSUPPRESS），或者在着色器无法决定时调用性能较低但更灵活的 CPU 回调（eCALLBACK）。`PxPairFlags` 指定了其他标志，这些标志代表模拟将来应针对此对执行的操作。例如，`eNOTIFY_TOUCH_FOUND`意味着当配对真正开始接触时通知用户，而不仅仅是潜在的。

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

这表示对于所有其他对象，请执行"默认"冲突处理。此外，还有一个基于 `filterDatas` 的规则，用于确定我们要求触摸通知(touch notifications)的特定对。要理解这意味着什么，我们需要知道样本赋予 `filterDatas` 的特殊含义。

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

### Aggregates
---------------
聚合是`actor`的集合。聚合不提供额外的模拟或查询功能，但允许您告诉 SDK 一组`actor`将聚类在一起，这反过来又允许 SDK 优化其空间数据操作。一个典型的用例是布娃娃，由多个不同的`actor`组成。如果没有聚集体，这会产生与布娃娃中的形状一样多的宽相实体(broad-phase entries)。通常，将布娃娃在广义阶段表示为单个实体，并在必要时在第二次通过时执行内部重叠测试会更有效。另一个潜在的用例是具有大量附加`Shape`的单个Actor。

### Creating an Aggregate
----------
从 `PxPhysics` 对象创建聚合：

```C++
PxPhysics* physics; // The physics SDK object

PxU32 nbActors;     // Max number of actors expected in the aggregate
bool selfCollisions = true;

PxAggregate* aggregate = physics->createAggregate(nbActors, selfCollisions);
```

目前，`actor`的最大数量限制为128个，为了提高效率，应尽可能低。如果永远不需要聚合的`actor`之间的碰撞检测，请在创建时禁用它们。这比使用场景过滤机制更有效，因为它绕过了所有内部过滤逻辑。典型的用例是静态或运动学`actor`的聚合。请注意，最大`actor`个数和自冲突属性（self-collision attribute）都是不可变的。

### Populating an Aggregate
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

### Releasing an Aggregate
------------
要释放聚合:
```C++
PxAggregate* aggregate;    // The aggregate we previously created
aggregate->release();
```

释放 `PxAggregate` 不会释放聚合的`actors`。如果 `PxAggregate` 属于某个场景，则 `Actor` 会自动重新插入到该场景中。如果您打算同时删除 `PxAggregate` 及其`actors`，则最有效的方法是先释放`actor`，然后在 `PxAggregate` 为空时释放它。

### Amortizing Insertion
-------
在一帧中向场景中添加多个对象可能是一项代价高昂的操作。布娃娃就是这种情况，如前所述，这是 `PxAggregate` 的良好候选者。另一种情况是局部碎片，其自碰撞(self-collisions)经常被禁用。要将对象插入宽相结构(broad-phase structure)的成本摊销到多个阶段结构中，请在 `PxAggregate` 中生成碎片，然后从聚合中删除每个 `Actor` ，然后通过这些帧将其重新插入到场景中。

### Trigger Shapes
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

### Interactions
--------------
SDK 在内部为宽相(broad-phase)报告的每个重叠对创建一个交互对象。这些对象不仅为成对碰撞的刚体创建，而且还为重叠的触发器对创建。一般来说，用户应该假设无论涉及对象的类型（刚体，触发器，布料等）以及所涉及的`PxFilterFlag`标志如何，都创建了此类对象。目前，每个参与者的此类交互对象限制为 65535 个。如果超过 65535 个交互涉及同一个执行组件，则 SDK 会输出一条错误消息，并忽略额外的交互。


## Rigid Body Dynamics
---------

在本章中，我们将介绍一些主题，当您熟悉了设置基本的刚体模拟世界后，这些主题很重要。

### Velocity
-------------------
刚体的运动分为线性和角速度分量。在仿真过程中，`PhysX`将根据重力、其他施加的力和扭矩以及各种约束（如碰撞或关节）来修改物体的速度。

可以使用以下方法读取物体的线性速度和角速度：

```C++
PxVec3 PxRigidBody::getLinearVelocity();
PxVec3 PxRigidBody::getAngularVelocity();
```

可以使用以下方法设置物体的线性速度和角速度：

```C++
void PxRigidBody::setLinearVelocity(const PxVec3& linVel, bool autowake);
void PxRigidBody::setAngularVelocity(const PxVec3& angVel, bool autowake);
```

### Mass Properties
---------------------------
动态actor需要质量属性(mass properties)：质量，惯性矩和质量中心框架，它指定Actor的质心位置及其主惯性轴。计算质量属性的最简单方法是使用 `PxRigidBodyExt::updateMassAndInertia（）` helper function，该函数将根据参与者的形状和均匀的密度值设置所有三个属性。此功能的变体允许组合每个形状的密度和手动指定某些质量属性。有关更多详细信息，请参阅 `PxRigidBodyExt` 的参考。North Pole Sample中摇摇晃晃的雪人说明了不同质量属性的使用。雪人就像罗利聚乙烯玩具，通常只是一个空壳，底部装满了一些沉重的材料。低质心导致它们在倾斜后移回直立位置。它们有不同的风格，具体取决于质量属性的设置方式：

+ 第一种基本上是无质量的。在Actor的底部只有一个质量相对较高的小球体。由于产生的惯性矩很小，这导致相当快速的运动。雪人感觉很轻。

+ 第二种仅使用底部雪球的质量，导致更大的惯性。后来，质心被移动到`actor`的底部。这种近似在物理上是不正确的，但由此产生的雪人感觉更饱满一些。

+ 第三个和第四个雪人使用形状来计算质量。不同之处在于，人们首先计算惯性矩（从实际质心），然后将质心移动到底部。另一个计算我们传递给计算例程的低质心的惯性矩。请注意第二种情况的摆动速度要慢得多，尽管两者的质量相同。这是因为头部在惯性矩（与质心的距离平方）中占了更多。

+ 最后一个雪人的质量属性是手动设置的。该示例使用惯性矩的粗略值来创建特定的所需行为。对角线张量在 X 中具有低值，在 Y 和 Z 中具有高值，从而产生围绕 X 轴旋转的低阻力和围绕 Y 和 Z 的高阻力。因此，雪人只会围绕X轴来回摆动。

如果您有一个 3x3 惯性矩阵（例如，您的对象有现实生活中的惯性张量），请使用 `PxDiagonalize()` 函数获取主轴和对角惯性张量来初始化 `PxRigidDynamic` `actor`。

当手动设置物体的质量/惯性张量时，`PhysX` 需要质量和每个主惯性轴的正值。但是，在这些值中提供 0 是合法的。
当提供0质量或惯性值时，PhysX将其解释为表示围绕该主轴的无限质量或惯性。这可用于创建抵抗所有线性运动或抵抗所有或某些角度运动的物体。使用此方法可以实现的效果示例如下：

+ 表现得好像是`kinematic`的`body`。

+ 其平移是`kinematic`的但其旋转是`dynamic`的`body`。

+ 其平移是`dynamic`的，但其旋转是`kinematic`的`body`。

+ 只能绕特定轴旋转的`body`。

下面详细介绍了可以取得的成就的一些例子。首先，让我们假设我们正在创建一个共同的结构 - 一个风车。下面提供了构建将成为风车一部分的`body`的代码：

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

此示例无限期地保留了风车以恒定角速度旋转的先前行为。然而，现在`body`的速度不能受到任何约束的影响，因为身体有无限的质量和惯性。如果一个物体与风车片相撞，碰撞的行为就好像风车片是一个`kinematic`物体。另一种选择是使风车具有无限的质量，并将其旋转限制在身体的局部z轴周围。这将提供与在风车和静态风车框架之间应用旋转接头相同的效果：

```C++
dyn->setMass(0.f);
dyn->setMassSpaceInertiaTensor(PxVec3(0.f, 0.f, 10.f));
```

在这两个例子中，`body`的质量都设置为0，表明身体具有无限的质量，因此其线速度不能被任何约束改变。但是，在此示例中，`body`的惯性被配置为允许主体的角速度受围绕一个主轴或惯性的约束的影响。这提供了与引入旋转接头类似的效果。z轴周围的惯性值可以增加或减少，以使风车对运动的抵抗力更大/更小。

## Rigid Body Collision
------------------------
### Introduction
------------------------

 本节将介绍刚体碰撞的基础知识。

 ### Shapes
 -----------------------
 `Shape`描述`actor`的空间范围和碰撞属性。它们在 `PhysX` 中用于三个目的：确定刚性对象的接触特征的相交性测试、场景查询测试（如raycast）以及定义触发体积（当其他`shape`与它们相交时生成通知）。每个`shape`都包含一个 `PxGeometry` 对象和一个对 `PxMaterial` 的引用，这两个对象都必须在创建时指定。下面的代码创建一个具有球体几何图形和特定材质的形状：
 ```C++
 PxShape* shape = physics.createShape(PxSphereGeometry(1.0f), myMaterial, true);
myActor.attachShape(*shape);
shape->release();
 ```

方法 `PxRigidActorExt::createExclusiveShape()` 等效于上面的三行。

* 注意： 有关反序列化`Shape`的参考计数行为，请参阅反序列化对象的引用计数。*

用于 `createShape()` 的参数 "true" 通知 `SDK` 该`Shape`不会与其他`Actor`共享。当有许多具有相同几何图形的 `Actor` 时，可以使用`Shape`共享来降低模拟的内存成本，但共享`Shape`具有非常严格的限制：在共享`Shape`附加到 `Actor` 时，您无法更新共享`Shape`的属性。

（可选）您可以通过指定类型为 `PxShapeFlags` 的形状标志来配置形状。默认情况下，形状配置为:
+ 模拟`Shape`（在模拟期间启用接触生成）

+ 场景查询`Shape`（为场景查询启用）

+ 如果启用了调试时渲染(debug rendering)，则可视化

为`Shape`指定几何对象时，该几何对象将复制到该`Shape`中。对于可以为`Shape`指定几何图形有一些限制，具体取决于形状标志和父角色的类型。

+ 附加到动态`Actor`的模拟`Shape`不支持`TriangleMesh`、`HeightField`和`Plane geometries`，除非动态`Actor`配置为`kinematic`。

+ 触发器形状(trigger shapes)不支持`TriangleMesh`和`HeightField`几何图形。

有关更多详细信息，请参阅以下部分。

将`Shape`与`Actor``Detach`，如下所示：

```C++
myActor.detachShape(*shape);
```

注意
在以前版本的 `PhysX` 中，`release()` 用于将`Shape`与其执行组件分离并销毁它。release（） 的这种用法在` PhysX 3.3` 中已弃用，在 `PhysX` 的未来版本中将不受支持。

### Simulation Shapes and Scene Query Shapes
--------------------------
`Shape`可以独立配置为参与场景查询和相交性测试中的一个或两个。默认情况下，`Shape`将同时参与这两项操作。

以下伪代码配置 `PxShape` 实例，使其不再参与`Shape`对相交性测试：

```C++
void disableShapeInContactTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSIMULATION_SHAPE,false);
}
```

可以将 `PxShape` 实例配置为参与`Shape`对相交性测试测试，如下所示：

```C++
void enableShapeInContactTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSIMULATION_SHAPE,true);
}
```

要从场景查询测试中禁用 `PxShape` 实例，请执行以下操作：

```C++
void disableShapeInSceneQueryTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSCENE_QUERY_SHAPE,false);
}
```
最后，可以在场景查询测试中重新启用 `PxShape` 实例：

```C++
void enableShapeInSceneQueryTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSCENE_QUERY_SHAPE,true);
}
```

