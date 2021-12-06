# Simulation
----------------
## Callback Sequence
---------------------
最简单的模拟回调(simulation callbacks)类型是事件。使用回调，应用程序可以简单地侦听事件并根据需要做出反应，前提是回调遵守禁止 SDK 状态更改的规则。鉴于 SDK 允许在模拟运行时写入非活动的后台缓冲区(inactive back-buffer)，此限制可能有点令人惊讶。但是，事件回调不是从模拟线程内部调用的，而是从 `fetchResults（）` 内部调用的。这里的关键点是`fetchResults（）`处理缓冲的写入，这意味着从事件回调写入SDK可能是一件特别脆弱的事情。为了避免这种脆弱性，有必要强加不允许从事件回调更改 SDK 状态的规则。

在 `fetchResults（）` 内部，缓冲区被`swap`。更具体地说，这意味着每个对象的内部模拟状态的属性被复制到 API 可见状态。有些事件回调发生在此`swap`之前，有些事件发生在此`swap`之后。之前发生的事件是：

+ onTrigger 
+ onContact 
+ onConstraintBreak 

当在回调中收到这些事件时，`Shape`、 `Actor` 等仍将处于模拟开始之前的状态。这是可取的，因为在模拟的早期，在对象被integrated（moved）之前，这些事件是检测到的。例如，一对获得 `onContact（）` 以报告它们正在联系的`Shape`在发出呼叫时仍将保持联系，即使它们在 `fetchResults` 返回后可能再次反弹。

另一方面，这些事件是在`swap`后发送的：

+ onSleep 
+ onWake 

睡眠信息在对象integrated后更新，因此在`swap`后发送这些事件是有意义的。

要"listen"这些事件中的任何一个，首先需要声明一个 `PxSimulationEventCallback` 的子类，以便可以根据需要实现各种虚函数。然后，可以使用 `PxScene::setSimulationEventCallback` 或 `PxSceneDesc::simulationEventCallback` 为每个场景注册此子类的实例。单独执行这些步骤将确保成功报告约束中断事件(constraint break events)。报告睡眠和唤醒事件还需要另一个步骤：为了避免报告所有睡眠和唤醒事件的费用，标识为值得睡眠/唤醒通知的`Actor`需要引发标志 `PxActorFlag::eSEND_SLEEP_NOTIFIES`。最后，若要接收 `onContact` 和 `onTrigger` 事件，必须在筛选器着色器回调(filter shader callback)中为需要事件的所有交互对象对设置一个标志。有关筛选器着色器回调(filter shader callback)的更多详细信息，请参阅Collision Filtering部分。

## Simulation memory
---------------------
`PhysX` 依赖于应用程序进行所有内存分配。主接口通过初始化 SDK 所需的 `PxAllocator` 回调接口：

```C++
class PxAllocatorCallback
{
public:
    virtual ~PxAllocatorCallback() {}
    virtual void* allocate(size_t size, const char* typeName, const char* filename,
        int line) = 0;
    virtual void deallocate(void* ptr) = 0;
};
```

在描述分配大小的self-explanatory function参数之后，接下来的三个函数参数是identifier name，用于标识分配的类型，以及__FILE__和__LINE__在进行分配的 SDK 代码中的位置。有关这些函数参数的更多详细信息，请参阅 PhysXAPI 文档。

**自 2.x 以来的一个重要更改：SDK 现在要求返回的内存以 16 字节对齐。在许多平台上，malloc（） 返回 16 字节对齐的内存，但在 Windows 上，系统函数 _aligned_malloc（） 提供了此功能。**

**在某些平台上， `PhysX` 使用系统库调用来确定正确的类型名称，并且返回类型名称的系统函数可能会调用系统内存分配器。如果您正在检测系统内存分配，您可能会观察到此行为。若要防止 `PhysX` 请求类型名称，请使用 `PxFoundation::setReportAllocationNames（）` 方法禁用分配名称。**

最大程度地减少动态分配是性能调优的一个重要方面。 `PhysX` 提供了多种机制来控制和分析内存使用情况。这些问题应依次讨论。

### Scene Limits
-----------------
通过预先设置场景数据结构的容量，可以在创建场景之前使用 `PxSceneDesc::limits` 或函数 `PxScene::setLimits（）`， 最大限度地减少跟踪对象的分配数量。需要注意的是，这些限制并不代表硬限制，这意味着如果对象数超过场景限制， `PhysX` 将自动执行进一步的分配。

### 16K Data Blocks
--------------------
`PhysX` 用于仿真的大部分内存都保存在一个块池中，每个块的大小为16K。分配给池的初始块数可以通过设置`PxSceneDesc::nbContactDataBlocks`来控制，而池中可以容纳的最大块数由`PxSceneDesc::maxNbContactDataBlocks`控制。如果 `PhysX` 内部需要的块数多于 `nbContactDataBlocks` ，则它将自动向池分配更多块，直到块数达到 `maxNbContactDataBlocks` 。如果 `PhysX` 随后需要的块数超过最大块数，那么它将简单地开始丢弃接触和关节约束(contacts and joint constraints)。发生这种情况时，警告将传递到 `PX_CHECKED` 配置中的错误流。

为了帮助调整 `nbContactDataBlocks` 和 `maxNbContactDataBlocks` ，使用函数 `PxScene::getNbContactDataBlocksUsed（）` 查询当前分配给池的块数会很有用。查询可以使用`PxScene::getMaxNbContactDataBlocksUsus`分配给池的最大块数也很有用。

可以使用 `PxScene::flushSimulation（）` 回收未使用的块。当调用此函数时，当前场景状态不需要的任何已分配块都将被删除，以便应用程序可以重用它们。此外，通过将许多其他内存资源收缩到场景配置所需的最小大小，可以释放这些内存资源。

### Scratch Buffer
------------------
scratch memory block可以作为函数参数传递给函数 `PxScene::simulate`。 `PhysX` 将尽可能在内部从暂存内存块(scratch memory block)中分配临时缓冲区，从而减少从 `PxAllocatorCallback` 执行临时分配的需要。在`PxScene::fetchResults（）`调用之后，应用程序可以重用该块，这标志着模拟的结束。暂存存储器块的一个限制是，它必须是 16K 的倍数，并且必须是 16 字节对齐的。

### In-place Serialization
---------------------------
`PhysX` 对象 `cab` 使用 `PhysX` 的二进制反序列化机制存储在应用程序拥有的内存中。有关详细信息，请参阅序列化。

### PVD Integration
-------------------
有关内存分配的详细信息可以记录并显示在 `PhysX` 可视调试器中。此内存分析功能可以通过在调用 `PxCreatePhysics（）` 时设置 `trackOutstandingAllocations` 标志来配置，并在使用 `PxVisualDebuggerExt::createConnection（）` 连接到调试器时引发标志 `PxVisualDebuggerConnectionFlag::eMEMORY`。

## Completion Tasks
--------------------
完成后任务(Completion Tasks)是在 `PxScene::simulate` 退出后立即执行的任务。如果 `PhysX` 已配置为使用工作线程，则 `PxScene::simulate` 将在工作线程上启动模拟任务，并且可能会在工作线程完成场景更新所需的工作之前退出。因此，典型的`completion task`首先需要调用 PxScene::fetchResults（true）以确保 fetchResults 阻塞，直到在 simulate（） 期间启动的所有工作线程都已完成其工作。调用 fetchResults（true） 后，`completion task`可以执行应用程序认为必要的任何其他物理处理后工作：

+ scene.fetchResults(true); game.updateA(); game.updateB(); ... game.updateZ();

完成后任务在 `PxScene::simulate` 中指定为函数参数。更多详细信息可以在 PhysAPI 文档中找到。

## Synchronizing with Other Threads
--------------------------------------
子步骤的一个重要考虑因素是，`simulate（）` 和 `fetchResults（）` 被归类为场景中的写入调用，因此在这些函数运行时读取或写入场景是非法的。对于 `simulate（）` 函数，区分running和ongoing的函数非常重要。在这种情况下，在 `simulate（）` 退出之前读取或写入场景是非法的。但是，在 `simulate（）` 退出后，在 `simulate（）` 调用期间启动的工作线程完成工作之前读取或写入场景是完全合法的。

**`PhysX` 不会锁定其场景图，但如果检测到多个线程对同一场景进行并发调用，它将在已检查的构建中报告错误，除非它们都是读取调用。**

## Substepping
----------------
出于保真度模拟(fidelity simulation)或更好的稳定性的原因，通常希望 `PhysX` 的模拟频率高于应用程序的更新速率。最简单的方法是多次调用 `simulate（）` 并 `fetchResults（）`：

```C++
for(PxU32 i=0; i<substepCount; i++)
{
    ... pre-simulation work (update controllers, etc) ...
    scene->simulate(substepSize);
    scene->fetchResults(true);
    ... post simulation work (process physics events, etc) ...
}
```

Substepping还可以与 `simulate（）` 函数的`completion task`功能集成(integrated)。为了说明这一点，请考虑在`graphics`组件发出信号，表明它已完成对场景渲染状态的更新之前，模拟场景的情况。在这种情况下，`completion task`将在 `simulate（）` 退出后自然运行。它的第一个工作是使用 fetchResults（true） 进行阻止，以确保它等到 `simulate（）` 和 `fetchResults（）` 都完成了它们的顺序工作。当`completion task`能够继续时，其下一个工作项将是查询`graphics`组件，以检查是否需要另一个 `simulate（）` 或是否可以退出。在需要另一个 `simulate（）` 步骤的情况下，它显然需要传递一个`completion task`来 `simulate（）`。这里的一个棘手点是，`completion task`不能将自己作为下一个`completion task`提交，因为它会导致非法递归。此问题的解决方案可能是有两个`completion task`，其中每个任务存储对另一个任务的引用。然后，每个`completion task`都可以通过其伙伴进行`simulate()`:

```C++
scene.fetchResults(true);
if(!graphics.isComplete())
{
    scene.simulate(otherCompletionTask);
}
```

## Split sim
-------------
作为 `simulate（）` 的替代方法，您可以将模拟拆分为两个不同的阶段，`collide（）` 和 `advance（）`。对于某些属性（称为write-through properties），`collide（）`阶段的修改将立即被随后的`advance（）`阶段看到。这允许 `collide（）` 在 `advance（）` 所需的数据可用之前开始，并与生成输入以` advance（）` 的游戏逻辑并行运行。这对于生成`kinematic targets`的动画逻辑以及将力施加到物体上的控制器特别有用。下面列出了write-through properties ：

```C++
addForce()/addTorque()/clearForce()/clearTorque()
setAngularVelocity()/setLinearVelocity()
setKinematicTarget()
wakeUp()
setWakeCounter()
```

使用Split sim时，物理模拟循环将如下所示：

```C++
scene.collide(dt)
scene.fetchCollision()
scene.advance()
scene.fetchResults()
```

任何其他 API 调用序列都是非法的。SDK 将发出错误消息。用户可以在` collide（）` 和 `fetchCollision()` 之间交错依赖于物理的游戏逻辑：

```C++
scene.collide(dt)
physics-dependent game logic(anmimation, rendering)
scene.fetchCollision()
```

`fetchCollision（）` 将等到 `collide（）` 完成后再更新 SDK 中的write-through properties 。一旦 `fetchCollision（）` 完成，对执行场景中的对象执行的任何状态修改都将被缓冲，并且在模拟和对 `fetchResults（）` 的调用完成之前不会反映。求解器在为被模拟的`Actor`计算新的速度和位置集时，将考虑write-through properties。

## Split fetchResults
------------------------
`fetchResults（）` 方法有标准格式和拆分格式。与标准的 `fetchResult（）` 方法相比，拆分格式(split format)具有一些优势，因为它允许用户并行处理contact reports，这在模拟复杂场景时可能很昂贵。

使用拆分 `fetchResults` 的简单方法如下所示：

```C++
gSharedIndex = 0;

gScene->simulate(1.0f / 60.0f);

//Call fetchResultsStart. Get the set of pair headers
const PxContactPairHeader* pairHeader;
PxU32 nbContactPairs;
gScene->fetchResultsStart(pairHeader, nbContactPairs, true);

//Set up continuation task to be run after callbacks have been processed in parallel
callbackFinishTask.setContinuation(*gScene->getTaskManager(), NULL);
callbackFinishTask.reset();

//process the callbacks
gScene->processCallbacks(&callbackFinishTask);

callbackFinishTask.removeReference();

callbackFinishTask.wait();

gScene->fetchResultsFinish();
```

用户可以自由使用自己的任务/线程(task/threading)系统来处理回调。但是， `PhysX` 场景提供了一个实用程序函数，该函数使用多个线程处理回调，此代码段使用该函数。此方法采用一个continuation task ，该task将在处理回调的任务完成时运行。在此示例中，完成任务会引发一个事件，可以等待该事件通知主线程回调处理已完成。

此功能在 `SnippetSplitFetchResults` 中演示。为了使用此方法，contact notification callbacks必须是线程安全的。此外，为了使这种方法有益，contact notification callbacks需要做大量的工作才能从多线程中受益。
