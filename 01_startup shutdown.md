# startup shutdown

## introduction

----

使用`physx sdk`的第一步是初始化一个全局对象。这些对象会在`physx`不在需要这些资源时relese。

## Foundation and Physics

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

##  Cooking

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

# Extension

extensions library 包含了很多方法，这些方法对于很多用户都有用，但是某些用户可能出于代码量的大小或者避免使用某些子系统的原因省略掉不使用。初始化extensions library需要`PxPhysics`对象

```c++
if (!PxInitExtensions(*mPhysics, mPvd))
    fatalError("PxInitExtensions failed!");
```

## Optional SDK Components

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



## Delay-Loading DLLs

------------------------------

在`PhysX` ,`PhysXCooking` ,`PhysXCommon`和`PxPvdSDK`项目中，`PhysXCommon DLL` `PxFoundation DLL`和`PxPvdSDK DLL`被标记为延迟装载(Delay-Loading)

### PhysXCommon DLL and PxFoundation DLL load

应用程序链接到PhysXCommon DLL，会在装在其他DLL之前先装载`PxFoundation.dll`、`PxPvdSDK `和`PhysXCommon.dll`。应用程序装载的DLL必须和`PhysX`和`PhysXCooking DLLs`使用的一样。在`PhysX`和`PhysXCooking DLLs`中关于`PhysXCommon`,`PxFoundation`和`PxPvdSDK`由如下规则构成：

+ 如果延迟加载hook指定了`PhysXCommon`名称，则使用用户提供的`PhysXCommon`或`PxPvdSDK`名称
  + 如果延迟加载hook没有指定，则会使用相应的`PhysXCommon``PxFoundation`或者`PxPvdSDK`

### PxDelayLoadHook

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

###  PxGpuLoadHook

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

###  PhysXCommon Secure Load

所有由Nvidia分布的`PhysX DLL`都被签名过。通过`PhysX`或者`PhysXCooking`装载时，会检查`PhysXCommon DLL`的签名。如果签名验证失败会终止应用程序。

## Shutting Down

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

