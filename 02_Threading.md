# Threading

## Introduction

-------------

本章介绍了在多线程应用程序中如何使用`PhysX`。主要有三个方面使用多线程：

+ 如何在不产生竞争的情况下调用`PhysX API`进行读写
+ 如何使用多线程加速模拟过程
+ 如何进行同步模拟，以及如何在模拟过程中进行读和写

## Data Access from Multiple Threads

由于各种原因，`PhysX`不会在内部锁定应用程序对其数据结构的访问，所以当我们在多线程应用中调用API时需要小心。规则如下：

+ 标记为`const`的API接口方法是只读的， 其他的接口是写的。
+ 多线程下可以同时调用多个只读接口。
+ 不同场景中的对象能够通过不同的线程安全地获取。
+ 场景外的不同对象可以从不同的线程安全地访问，注意访问一个对象可能会通过持久性引用间接获取另一个对象。（一个actor引用一个shape，一个shape引用一个mesh，joint和actors相互引用）

不符合上述规则的访问模式可能会导致数据损坏、死锁或程序崩溃。注意在实践中不能一边对场景中的对象进行写操作一边对相同场景中的对象进行读操作。

构建好的工程(check build)包含跟踪应用程序线程对场景中对象的访问的代码，他们会在用户进行非法调用API时进行检测。

### Scene Locking

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

###  Locking Semantics

关于`scene lock`地使用有精确地描述：

+ 多线程可以同时读。
+ 只能有一个线程写，读是不能写。
+ 一个线程持有写锁它能够同时读和写。
+ 支持Re-entrant 读锁，一个线程已经获取一个读锁仍然允许调用`lockRead()`。每一个`lockRead()`必须有一个`unlockRead()`。
+ 支持Re-entrant写锁， 一个线程已经获取一个写锁仍然允许调用`lockWrite()`。每一个`lockWrite()`必须有一个`unlockWrite()`。
+ 允许已经获取了写锁的线程调用`lockRead()`，线程将仍然能读和写。每一个`lock()`必须有`unlock()`。
+ 不支持lock upgrading。不允许持有读锁的线程`lockWrite()`。在checked build中这样做会报错，在release build中会导致deadlock。
+ Writers are favored。如果线程获取到读锁后`lockWrite()`会阻断所有的reader离开。如果新的reader在writer线程被阻塞时到达，它们(reader)将被置于睡眠，并且writer先访问场景。
+ 如果多个writer在队列中，第一个writer优先级最高，后续的writer将根据OS的scheduling被授予访问权限。

注意：*PxScene::release()* 会自动尝试获取写锁，不需要在调用release()之前手动获取写锁。

### Locking Best Practices

安排应用程序一次性获取锁以执行多个操作通常很有用。这最大限度地减少了锁的开销，此外还可以防止诸如在一个做sweep test的线程中看到布娃娃在另一个线程进行部分插入之类的情况。

群集写入还有助于减少对锁的争用，因为获取写锁会使尝试读的任何线程停止。

## Asynchronous Simulation

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

注意`simulate()`,`fetchResults()`是写调用，被调用禁止对于任何对象的访问。

### Double Buffering

----------------------------

`PhysX`支持在模拟运行时对场景中的对象进行读写操作。这包括添加/移除到场景。

从用户的角度来看，API 更改会立即反映出来。例如：如果设定了刚体的速度并进行查询，则会返回新的速度。相似的，如果在模拟进行中时创建了对象，它能够被其他任何对象访问或者修改。当然了，这些改变都在缓存中所以模拟的代码只会看到调用`PxScene::simulate()`时的对象状态。改变对象filter data只会影响下一步模拟step，运行中的模拟对正在运行的step会忽略掉。

当`PxScene::fetchResults()`被调用时，任何buffer的改变会被flush：模拟对对象的改变会被反射到API视图上，API的改变会在模拟的下一个step中可见。用户的修改会更优先：当模拟运行时用户修改了一个对象的position，则用户修改的数据会覆盖模拟的结构写入结果中。延迟应用更新不会影响场景查询，场景查询始终会考虑最新的更改。

### Events involving removed objects

-------------

模拟运行时删除对象或者将对象移除出场景会影响发送给`PxScene::fetchResults()`的事件。

+ 运行期间如果一个对象被删除或者移除将不会移除出`PxSimulationEventCallback::onWake()`,`PxSimulationEventCallback()::onSleep()`
+ 但是会移除出`PxSimulationEventCallback::onContact()``PxSimulationEventCallback::onTrigger()`。对象将被标记为`PxContactPairHeaderFlag::eREMOVED_ACTOR_0`,`PxContactPairFlag::eREMOVED_SHAPE_0`,`PxTriggerPairFlag::eREMOVED_SHAPE_TRIGGER`。此外，如果为包含已删除/已移除对象的对请求 `PxPairFlag::eNOTIFY_TOUCH_LOST`，`PxPairFlag::eNOTIFY_THRESHOLD_FORCE_LOST`事件，则将创建这些事件。

### Support

不是所有的`PhysX`对象有所有的buffering support。API 文档中提到了在模拟过程中无法运行的操作，SDK 会中止此类操作并报告错误。最重要的exceptions如下：

+ Particle：在模拟运行时粒子数据不能被读取和修改，包括读写粒子的位置或速度，创建删除例子，加力等等。
+ Cloth：唯一允许的双缓冲(double buffered)操作是创建删除cloth和将他添加移除出场景。

### Memory Considerations

在模拟运行时，用于存储对象更改的缓冲区是按需创建的。如果内存使用时出现的问题超过了与模拟并行读取/写入对象的好处，则不要在模拟运行时写入对象。

## Multithreaded Simulation

----------

`PhysX`拥有一个管理CPU和GPU的计算资源的任务系统。任务通过依赖关系创建，这样能够按照给定顺序解析。解析完成后，他们将会提交给用户实现的调度程序执行。中间件产品（Middleware products）通常不希望创建 CPU 线程供自己使用，在执行线程可能产生大量开销的控制台上尤其如此。在任务模型（task model）中，计算工作（computational work）被分解为job，这些jobs在准备运行时提交到游戏的线程池中。以下类组成了 CPU 任务管理：

### TaskManager

----------

TaskManager管理任务间相关性，并将就绪任务分派给各自的dispatcher。存在一个用于分配给 TaskManager 的 CPU 任务和 GPU 任务的dispatcher。TaskManager通过SDK创建并持有，每一个`PxScene`会分配一个自己的TaskManager实例，用户可以通过 `PxSceneDesc `或直接通过 `TaskManager `界面使用dispatcher进行配置。

### CpuDispatcher

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

### CpuDispatcher Implementation Guidelines

-----------

在场景的 TaskManager 找到一个ready-to-run的任务并将其提交给相应的调度程序后，由调度程序实现来决定如何以及何时运行该任务。通常在游戏场景中，刚体模拟是时间关键型的，目标是减少从 simulate() 到 fetchResults() 完成的延迟。当 `PhysX` 任务在更新期间对 CPU 资源具有独占访问权限时，将实现尽可能低的延迟，但是实际上，PhysX将不得不与其他游戏任务共享计算资源。下面是一些准则，可帮助确保在将 PhysX 更新与其他工作混合时在吞吐量和延迟之间取得平衡：

+ 避免长时间运行的任务与 `PhysX `任务交错，这将有助于减少延迟。
+ 避免将工作线程分配给与优先级较高的线程相同的执行核心。如果在执行期间切换了 PhysX 任务的上下文，则处理刚体的pipeline的其余部分可能会停止，从而增加延迟。
+ PhysX偶尔会提交任务，然后立即等待它们完成，因此，以后进后出（stack）顺序执行任务可能比FIFO（queue）顺序执行任务更好。
+ PhysX 不是一个完全并行的 SDK，因此交错执行中小粒度任务（interleaving small to medium granularity tasks）通常会导致更高的总体吞吐量。
+ 如果您的线程池具有每线程作业队列（per-thread job-queues），则在它们提交的线程上排队任务（queuing tasks on the thread they were submitted ）可能会导致更优化的 CPU 缓存一致性，但这不是必需的。

### BaseTask

------------------------

`BaseTask`是所有任务类型的抽象基类。所有任务的 run() 函数都将在应用程序线程上执行，因此它们需要小心其堆栈用法，尽可能使用少量堆栈，并且堆栈永远不应该出于任何原因阻塞。

### Task

-----------

`Task`类是标准的task类型。每个模拟step都必须将`Task`提交给 TaskManager 才能执行这些任务。`Task`可以在提交时命名，这样它们能被发现。提交`Task`时，将为其提供引用计数 1，并且 TaskManager::startSimulation() 函数会递减所有任务的引用计数，并调度引用计数为零的所有任务。在调用 TaskManager::startSimulation() 之前，`Task`可以设置彼此之间的依赖关系，以控制它们的调度顺序。一旦模拟开始，仍然可以提交新任务并添加依赖项，但由程序员来避免race hazards。您不能向已分派的任务添加依赖关系，并且新提交的任务的引用计数必须递减，然后才能允许执行该任务。

此外还可以使用任务名称定义同步点（Synchronization points），`TaskManager`将为该名称分配一个不带任务实现(Task implementation)的TaskID。当满足所有命名 TaskID 的依赖项时，它将递减具有该名称的所有任务的引用计数。`APEX `几乎专门使用 `Task `类来管理 CPU 资源。`ApexScene `定义了许多命名任务，模块使用这些任务来计划自己的任务（例如：在 LOD 计算完成后开始，在 PhysX 场景逐步执行之前完成）。

### LightCpuTask

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

