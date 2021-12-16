# Character Controllers
----------------------------
## Introduction
-------------------
角色控制器 (CCT) SDK 是构建在 PhysX SDK 之上的外部组件，其方式类似于 PhysXExtensions。

CCT 可以通过多种方式实现：CCT 模块中的 PhysX 实现只是其中之一。

从本质上讲，CCT 通常是针对特定游戏的，并且它们可以在每个游戏中具有许多独特的功能。 例如，角色的边界体积在一个游戏中可能是一个胶囊，而在另一个游戏中可能是一个倒金字塔。 CCT SDK 并不试图提供一种适用于所有可能游戏的开箱即用的解决方案。 但它提供了所有 CCT 共有的基本功能：角色控制和角色交互。 它是用户的默认起点，是一个可以构建的强大基础，可以在以后根据需要进行修改或自定义。

## Kinematic Character Controller
--------------------------------
PhysX CCT 是一个运动控制器。传统上，角色控制器可以是运动的或动态的。运动控制器直接处理输入位移向量（一阶控制）。动态控制器与输入速度（二阶控制）或力（三阶控制）一起工作。

过去，游戏并没有使用像 PhysX SDK 这样的“真实”物理引擎。但他们仍然使用角色控制器在关卡中移动玩家。这些游戏，比如 Quake 甚至 Doom，都有专门的、定制的代码来实现碰撞检测和响应，这通常是整个游戏中唯一的物理部分。它实际上几乎没有物理特性，但有很多精心调整的值，以在控制玩家的同时提供良好的感觉。它实现的特定行为通常被称为“碰撞和滑动”算法，并且已经“调整了十多年”。 PhysX CCT 模块是此类算法的实现，为角色控制提供了稳健且众所周知的行为。

运动学控制器的主要优点是它们不会遇到以下问题，这些问题对于动态控制器来说是典型的：

+ （缺乏）连续碰撞检测：典型的物理引擎使用离散碰撞检查，导致臭名昭著的“隧道效应”，多年来一直困扰着各种商业和非商业物理包。 这导致三个主要问题：
  + 隧道效应本身：如果角色走得太快，它可能会穿过墙壁
  + 因此，角色的最大速度受到限制（因此也限制了游戏的可能性）
  + 即使它没有隧道，例如在角落里向前推时，角色可能会抖动，因为物理引擎不断将其前后移动到略有不同的位置。
+ 没有直接控制：刚体通常由脉冲或力控制。 通常不可能将其直接移动到其最终位置：相反，必须将 delta 位置向量转换为脉冲/力，应用它们，并希望角色最终到达所需位置。 这并不总是有效，特别是当物理引擎使用不完美的线性求解器时。
+ 摩擦问题：当角色站在斜坡上时，它不应该滑动。所以这里需要无限摩擦。当角色在同一个斜坡上向前移动时，它不应减速。这里不需要任何摩擦。同样，当角色靠墙滑动时，它也不应该放慢速度。因此，对于 CCT，摩擦通常为 0 或无穷大。不幸的是，物理引擎中的摩擦模型可能并不完美，很容易以少量摩擦（角色减速一点）或非常大但不是无限大的摩擦（无论人为的摩擦参数有多大，角色在该斜坡上的滑动速度都非常缓慢）。斜坡的相互矛盾的要求也意味着通常根本无法完美地模拟所需的行为。

+ 恢复原状的问题：通常，对于 CCT 应避免恢复原状。当角色快速移动并与墙壁碰撞时，它不应弹开。当角色从高处坠落并着地时，弯曲双腿，应防止任何弹跳。但是再一次，即使恢复完全为零，物理引擎仍然可以使 CCT 反弹一点。这不仅与线性求解器的不完美本质有关，还与典型的基于穿透深度的引擎如何从重叠情况中恢复有关，有时会施加过多的力将对象分开太多。

+ 不希望的跳跃：角色必须经常粘在地上，无论身体行为应该是什么。例如，动作游戏中的角色往往以不切实际的速度快速移动。当他们到达坡道顶部时，物理引擎通常会让他们跳跃一点，就像一辆快速的汽车在旧金山的街道上跳跃一样。但这通常不是想要的行为：相反，角色应该经常粘在地面上，而不管其当前的速度如何。这有时是使用固定关节来实现的，但对于使用运动控制器可以轻松避免的问题，这是一种不必要的复杂解决方案。

+ 不希望的旋转：一个典型的角色总是站着，从不旋转。然而，物理引擎通常对这种约束的支持很差，并且通常会付出很大的努力来防止角色周围的胶囊掉落（它应该始终在其尖端站立）。这通常是使用人工关节来实现的，由此产生的系统既不是很健壮也不是很快。

总而言之，可以花费大量精力来调整和禁用物理引擎的功能，只是为了模拟不太复杂的自定义代码。 继续使用那段简单的自定义代码是很自然的。

## Creating a character controller
------------------------------------
首先，在应用程序中的某处创建一个控制器管理器。 该对象跟踪所有创建的控制器，并允许来自同一管理器的角色相互交互。 使用 PxCreateControllerManager 函数创建管理器：

```C++
PxScene* scene;    // Previously created scene
PxControllerManager* manager = PxCreateControllerManager(*scene);
```

然后，为游戏中的每个角色创建一个控制器。 在撰写本文时，仅支持盒 (PxBoxController) 和胶囊 (PxCapsuleController)。 例如，胶囊控制器是这样创建的：

```C++
PxCapsuleControllerDesc desc;
...
<fill the descriptor here>
...
PxController* c = manager->createController(desc);
```

manager 类将跟踪所有创建的控制器。 可以使用以下功能随时检索它们:

```C++
PxU32          PxControllerManager::getNbControllers()   const = 0;
PxController*  PxControllerManager::getController(PxU32 index) = 0;
```

要释放角色控制器，只需调用它的释放函数：

```C++
void    PxController::release() = 0;
```

要一次释放所有创建的角色控制器，请释放管理器对象本身，或者如果您打算继续使用管理器，则使用以下函数：

```C++
void    PxControllerManager::purgeControllers() = 0;
```

SampleBridges 中说明了控制器管理器及其后续控制器的创建。

## Overlap Recovery Module
------------------------------
理想情况下，不应在初始重叠状态下创建角色，即它们应创建在不与周围几何体重叠的位置。 在创建角色之前，可以使用各种 PxScene 重叠函数来检查所需的空间体积是否为空。 默认情况下，CCT 模块不检查重叠本身，并且创建最初与世界静态几何体重叠的角色可能会产生不希望的和未定义的行为 - 例如角色穿过地面。

但是，重叠恢复模块可用于自动纠正字符的初始位置。 只要重叠量合理，恢复模块就应该能够将角色重新定位到适当的、无碰撞的位置。

重叠恢复模块在其他几种情况下也很有用。 主要有以下三种情况：

+ 当 CCT 直接生成或传送到另一个对象时
+ 当 CCT 算法由于 FPU 精度有限而失败时
+ 当“向上向量”被修改时，使旋转的 CCT 形状与周围的对象重叠

激活后，CCT 模块将自动尝试解决穿透问题，并将 CCT 移至不再与其他物体重叠的安全位置。 这仅涉及静态对象，该模块忽略动态对象。

使用此功能启用或禁用重叠恢复模块：

```C++
void PxControllerManager::setOverlapRecoveryModule(bool flag);
```

默认情况下，角色控制器使用精确的扫描测试，其精度通常足以避免所有穿透 - 前提是接触偏移不是太小。 因此，在大多数情况下不需要重叠恢复模块。 但是，当使用它时，可以使用以下函数将扫描测试切换到不太准确但可能更快的版本：

```C++
void PxControllerManager::setPreciseSweeps(bool flag);
```

## Character Volume
---------------------
角色使用独立于 SDK 中现有形状的边界体积。 我们目前支持围绕角色的两种不同形状：

+ AABB，由中心位置和范围向量定义。 AABB 不旋转。 即使玩家在（视觉上）旋转，它也总是有固定的旋转。 这样可以避免卡在太紧的地方而无法让 AABB 旋转。
+ 一个胶囊，由中心位置、垂直高度和半径定义。 高度是胶囊末端的两个球心之间的距离。 例如，当爬楼梯时，胶囊具有更好的行为。 这是推荐的默认选择。

<img src=".\image\Char_01.png" alt="Char_01" style="zoom:100%;" />

注意：2.3 之前的版本也支持球体。 这已被删除，因为 PxCapsuleController 更健壮并提供相同的功能（零长度胶囊）。

在角色的体积周围保留一个小皮肤，以避免在角色接触其他形状时会发生的数字问题。 此皮肤的大小是用户定义的。 出于调试目的渲染角色的体积时，请记住将体积扩大此皮肤的大小以获得准确的调试可视化。 此皮肤在 PxControllerDesc::contactOffset 中定义，稍后可通过 PxController::getContactOffset() 函数使用。

## Volume Update
---------------------
有时在运行时更改字符体积的大小很有用。 例如，如果角色可以蹲下，则可能需要降低其边界体积的高度，以便它可以移动到他无法以其他方式到达的地方。

对于盒子控制器，相关功能有：

```C++
bool PxBoxController::setHalfHeight(PxF32 halfHeight)               = 0;
bool PxBoxController::setHalfSideExtent(PxF32 halfSideExtent)       = 0;
bool PxBoxController::setHalfForwardExtent(PxF32 halfForwardExtent) = 0;
```

对于胶囊控制器：

```C++
bool PxCapsuleController::setRadius(PxF32 radius) = 0;
bool PxCapsuleController::setHeight(PxF32 height) = 0;
```

使用上述功能更改控制器的大小实际上不会更改其位置。 因此，如果角色站在地面上（触摸它），并且在没有更新其位置的情况下突然降低其高度，则该角色最终会悬浮在地面上几帧，直到重力使其再次坠落并再次接触地面。 发生这种情况是因为控制器位置位于形状的中心，而不是底部。 因此，要修改控制器的高度并保留其底部位置，必须同时更改控制器的高度和位置。 以下辅助函数会自动执行此操作：

```C++
void PxController::resize(PxF32 height) = 0;
```
<img src=".\image\Char_02.png" alt="Char_02" style="zoom:100%;" />

重要的是要注意，无需任何额外测试即可直接修改体积，因此可能会发生生成的体积与附近的某些几何图形重叠的情况。 例如，当调整角色大小以离开蹲伏姿势时，即当角色的大小增加时，首先检查角色是否确实可以“站立”很重要：角色上方的空间体积必须是空的（碰撞 自由）。 为此，建议使用各种 PxScene 重叠查询：

```C++
bool PxScene::overlap(...) = 0;
```

SampleNorthPole 中说明了在运行时更新角色的音量以实现“蹲伏”动作。 使用重叠查询离开蹲姿是在 SampleNorthPole::tryStandup() 函数中完成的。

## Moving a Character Controller
---------------------------------
CCT 算法的核心是实际移动字符的函数：

```C++
PxControllerCollisionFlags collisionFlags =
    PxController::move(const PxVec3& disp, PxF32 minDist, PxF32 elapsedTime,
    const PxControllerFilters& filters, const PxObstacleContext* obstacles=NULL);
```

disp 是当前帧的位移向量。当您的角色移动时，它通常是由于重力引起的垂直运动和横向运动的组合。请注意，用户有责任在此处对角色施加重力。

minDist 是一个最小长度，用于在剩余行驶距离低于此限制时尽早停止递归位移算法。

elapsedTime 是自上次调用移动函数以来经过的时间量。

过滤器是类似于 SDK 中使用的过滤参数。使用这些来控制角色应该碰撞什么。

障碍物是角色应该与之碰撞的可选附加障碍物对象。这些对象完全由用户控制，不需要对应的 SDK 对象。注意，接触到的障碍物会被缓存，这意味着如果障碍物的集合发生变化，缓存需要失效。

碰撞标志是返回给用户的位掩码，用于定义移动过程中发生的碰撞事件。这是 PxControllerCollisionFlag 标志的组合。它可用于触发各种角色动画。例如，您的角色可能会在播放下降空闲动画时下降，并且您可能会在返回 PxControllerCollisionFlag::eCOLLISION_DOWN 后立即开始着陆动画。

了解 PxController::move 和 PxController::setPosition 之间的区别很重要。 PxController::move 函数是 CCT 模块的核心。这就是前面提到的“碰撞和滑动”算法发生的地方。因此该功能将从 CCT 的当前位置开始，并使用扫描测试尝试向所需方向移动。如果发现障碍物，它可能会使 CCT 平滑地靠在它们上面。或者 CCT 可能会被墙挡住：移动调用的结果取决于周围的几何形状。相反，PxController::setPosition 是一个简单的“传送”函数，无论 CCT 从哪里开始，无论周围几何形状如何，即使所需位置在中间，它都会将 CCT 移动到所需位置的另一个对象。

在 SampleBridges 中演示了 PxController::move 和 PxController::setPosition。

## Graphics Update
---------------------
每一帧，在 PxController::move 调用之后，图形对象必须与新的 CCT 位置保持同步。 可以使用以下方法访问控制器的位置：

```C++
const PxExtendedVec3& PxController::getPosition() const;
```

此函数返回从碰撞形状中心开始的位置，因为这是 PhysX SDK 内部和常用图形 API 内部使用的位置。 SampleBridges 中说明了检索此位置并将其传递给渲染器的过程。 请注意，位置使用双精度，以使 CCT 模块与大世界一起工作。 另请注意，控制器永远不会旋转，因此您只能访问其位置。

提供了替代的辅助函数来使用角色的底部位置，也就是脚的位置：

```C++
const PxExtendedVec3& PxController::getFootPosition() const;
bool                  PxController::setFootPosition(const PxExtendedVec3& position);
```

请注意，脚位置考虑了接触偏移。

<img src=".\image\Char_03.png" alt="Char_03" style="zoom:100%;" />

## Auto Stepping
-------------------
如果没有自动步进，框控角色很容易卡在地面网格的轻微高度上。 在下图中，一小步将完全停止角色。 感觉很不自然，因为在现实世界中，角色会不假思索地越过这个小障碍。

<img src=".\image\Char_04.png" alt="Char_04" style="zoom:100%;" />

这就是自动步进使我们能够做到的。 没有玩家的任何干预（即没有他们考虑），盒子正确地越过小障碍物。

<img src=".\image\Char_05.png" alt="Char_05" style="zoom:100%;" />

但是，如果障碍物太大，即其高度大于stepOffset参数，则控制器无法自动爬升，角色卡住（这次是正确的）：

<img src=".\image\Char_06.png" alt="Char_06" style="zoom:100%;" />

“攀爬”（例如，越过这个更大的障碍）也可能在未来实现，作为自动步进的延伸。 步进偏移在 PxControllerDesc::stepOffset 中定义，稍后可通过 PxController::getStepOffset() 函数获得。

一般来说，步长偏移量应尽可能小。

## Climbing Mode
--------------------
自动步进功能最初是为盒子控制器设计的，它们很容易被地面上的小障碍物挡住。 Capsule 控制器由于其圆形特性，不一定需要该功能。

即使步距偏移为 0.0，胶囊也能够越过小障碍物，因为它们的圆形底部在与小障碍物碰撞后会产生向上运动。

由于自动步进功能及其圆形的综合效果，具有非零步距偏移的胶囊可以越过高于步距偏移的障碍物。在这种情况下，胶囊可以爬过的最大高度很难预测，因为它取决于自动步长值、胶囊的半径，甚至位移矢量的大小。

这就是为什么胶囊有两种不同的攀爬模式：

+ PxCapsuleClimbingMode::eEASY：在这种模式下，胶囊不受步长偏移值的约束。他们可能会越过高于此值的障碍物。
+ PxCapsuleClimbingMode::eCONSTRAINED：在此模式下，尝试确保胶囊无法越过高于步距偏移的障碍物。

## Up Vector
-------------
为了实现自动步进功能，SDK 需要知道“向上”向量。向上向量在 PxControllerDesc::upDirection 中定义，稍后可通过 PxController::getUpDirection() 函数获得。

向上向量不需要轴对齐。它可以是任意的，使用 PxController::setUpDirection() 函数修改每一帧，允许角色在球形世界中导航。这在 SampleCustomGravity 中进行了演示。

修改向上向量会改变 CCT 库查看字符卷的方式。例如，胶囊由 PxCapsuleControllerDesc::height 定义，它是沿向上向量的“垂直高度”。因此，从库的角度来看，改变向上向量可以有效地旋转胶囊。修改会立即发生，无需测试以验证角色是否与附近的几何体不重叠。然后角色可以在调用后立即穿透某些几何体。建议使用重叠恢复模块来解决这些问题。

<img src=".\image\Char_07.png" alt="Char_07" style="zoom:100%;" />

在上图中，左侧的胶囊使用垂直向上矢量并且不会与周围的几何体发生碰撞。在右侧，向上矢量已设置为 45 度，胶囊现在穿透附近的墙壁。对于大多数应用程序，向上向量将是常数，并且对于所有字符都相同。这些问题只会出现在在球形世界中导航的角色（例如小行星等）。

## Walkable Parts & Invisible Walls
-------------------------------------
默认情况下，角色可以随处移动。这可能并不总是一件好事。特别是，经常希望防止在坡度陡峭的多边形上行走。由于用户定义的斜率限制，SDK 可以自动执行此操作。所有坡度高于限制坡度的多边形将被标记为不可行走，并且 SDK 不会让角色去那里。

有两种模式可用于定义触摸不可行走部件时发生的情况。使用 PxControllerDesc::nonWalkableMode 枚举选择所需的模式：

+ PxControllerNonWalkableMode::ePREVENT_CLIMBING 阻止角色向上移动，否则不会移动角色。角色仍然可以在这些多边形上横向行走，并沿着斜坡向下移动。
+ PxControllerNonWalkableMode::ePREVENT_CLIMBING_AND_FORCE_SLIDING 不仅阻止角色向上移动不可行走的斜坡，而且还迫使它从这些斜坡滑下。
  
斜率限制在 PxControllerDesc::slopeLimit 中定义，稍后可通过 PxController::getSlopeLimit() 函数获得。极限表示为所需极限角的余弦。例如，这使用 45 度的斜率限制：

```C++
slopeLimit = cosf(PxMath::degToRad(45.0f));
```

使用 slopeLimit = 0.0f 自动禁用该功能（即字符可以去任何地方）。

并非总是需要此功能。一个常见的策略是禁用它并在关卡中放置隐形墙，以限制玩家的移动。如果 PxControllerDesc::invisibleWallHeight 不为零，字符模块还可以为您创建这些墙。在这种情况下，库会动态创建那些额外的三角形，并且该参数控制它们的高度（在用户定义的向上方向上挤出）。一个常见的问题是那些看不见的墙只有在找到不可行走的三角形时才会创建。如果跳跃角色的边界体积太小并且不与他下方不可行走的三角形发生碰撞，则跳跃角色可能会越过它们。 PxControllerDesc::maxJumpHeight 参数通过向下扩展边界体积的大小来解决这个问题。这样，碰撞查询会正确返回所有可能不可行走的三角形，并正确创建隐形墙 - 防止角色跳过它们。

一个已知的限制是斜率限制机制目前仅对静态对象启用。它不适用于动态对象，尤其是运动学对象。静态球体或静态胶囊也不支持它。

## Obstacle Objects
----------------------
有时可以方便地为 CCT 碰撞创建额外的障碍物，而无需创建实际的 SDK 对象。这在许多情况下都很有用。例如：

+ 障碍可能只存在于几个帧中，在这种情况下，创建和删除 SDK 对象并不总是有效的。
+ 障碍可能只存在于停止角色，而不是 SDK 的动态对象。例如，这将是几何体周围的隐形墙，只有角色应该与之碰撞。在这种情况下，将不可见墙创建为 SDK 对象可能不是很有效，因为它们的交互必须被过滤掉，除了角色之外的所有内容。将那些额外的隐形墙创建为外部障碍物可能更有效，只有角色才能与之交互。
+ 障碍物可能是动态的并使用可变时间步长更新，而 SDK 使用固定时间步长。例如，这可以是角色可以站立的移动平台。

在撰写本文时，角色控制器支持盒和胶囊 PxObstacle 对象，即 PxBoxObstacle 和 PxCapsuleObstacle。要创建这些，首先使用以下函数创建一个 PxObstacleContext 对象：

```C++
PxObstacleContext* PxControllerManager::createObstacleContext()    = 0;
```

然后通过以下方式管理障碍：

```C++
ObstacleHandle PxObstacleContext::addObstacle(const PxObstacle& obstacle) = 0;
bool PxObstacleContext::removeObstacle(ObstacleHandle handle) = 0;
bool PxObstacleContext::updateObstacle(ObstacleHandle handle, const PxObstacle& obstacle) = 0;
```

通常在控制器的移动调用之前调用 updateObstacle。

当 PLATFORMS_AS_OBSTACLES 在 SampleBridgesSettings.h 中定义时，SampleBridges 中说明了使用移动平台的障碍物。

## Hit Callback
---------------
PxUserControllerHitReport 对象用于检索有关控制器演变的一些信息。特别是，当一个角色碰到一个形状、另一个角色或用户定义的障碍物时，它会被调用。

当角色击中一个形状时，会调用 PxUserControllerHitReport::onShapeHit 回调 - 对于静态和动态形状。各种影响参数被发送到回调，然后它们可以用来做各种事情，比如播放声音、渲染轨迹、施加力量等等。 SampleBridges 中说明了 PxUserControllerHitReport::onShapeHit 的使用。请注意，此回调只会在角色移动到某个形状时被调用。如果（动态）形状与其他不移动的角色发生碰撞，则不会调用它。换句话说，这只会在 PxController::move 调用期间被调用。

当角色击中另一个角色时，即另一个由角色控制器控制的对象，会调用 PxUserControllerHitReport::onControllerHit 回调。例如，当玩家与 NPC 碰撞时就会发生这种情况。

最后，当角色碰到用户定义的障碍物时，会调用 PxUserControllerHitReport::onObstacleHit 回调。

## Behavior Callback
-----------------------
PxControllerBehaviorCallback 对象用于自定义角色在触摸 PxShape、PxController 或 PxObstacle 后的行为。 这是使用以下函数完成的：

```C++
PxControllerBehaviorFlags PxControllerBehaviorCallback::getBehaviorFlags
    (const PxShape& shape, const PxActor& actor) = 0;
PxControllerBehaviorFlags PxControllerBehaviorCallback::getBehaviorFlags
    (const PxController& controller)             = 0;
PxControllerBehaviorFlags PxControllerBehaviorCallback::getBehaviorFlags
    (const PxObstacle& obstacle)                 = 0;
```

在撰写本文时，支持以下返回标志：

PxControllerBehaviorFlag::eCCT_CAN_RIDE_ON_OBJECT 定义角色是否可以有效地与它所站立的物体一起移动。例如，站在动态桥上的角色应该跟随它所站立的 PxShape 的运动（例如在 SampleBridges 中）。但如果角色站在上面，情况就不应该如此，比如在地面上滚动的 PxShape 瓶子（例如 SampleNorthPole 中的雪球）。请注意，此标志仅控制从对象传输到控制器的水平位移。垂直运动略有不同，因为许多因素都会导致这种位移：用于自动走过小颠簸的步距偏移，底层动态演员的垂直运动，例如SampleBridges 中的桥梁，可能应该始终考虑在内，等等。

PxControllerBehaviorFlag::eCCT_SLIDE 定义角色站在物体上时是否应该滑动。这可以用作先前讨论的坡度限制特征的替代方案，以定义不可行走的对象而不是不可行走的部分。当胶囊的中心穿过平台的边缘时，它还可用于使胶囊角色自动从平台的边缘脱落。

PxControllerBehaviorFlag::eCCT_USER_DEFINED_RIDE 只是禁用与骑在对象上的控制器相关的所有内置代码。这对于恢复旧行为很有用，在将围绕 PhysX 2.x 角色控制器构建的一段代码移植到 PhysX 3.x 时，这有时是必要的。该标志只是跳过新的代码路径，让用户在他们自己的应用程序中处理这个特定的问题，在 CCT 库之外。

SampleBridges 中演示了行为回调。

## Character Interactions: CCT-vs-dynamic actors
--------------------------------------------------
通过在接触点施加力来让物理引擎推动动态对象是很诱人的。然而，这通常不是一个非常有说服力的解决方案。

角色周围的边界体积是人造的（盒子、胶囊等）并且是不可见的，因此物理引擎计算出的边界体积与其周围对象之间的力无论如何都不现实。它们不会正确模拟实际角色与这些对象之间的交互。如果边界体积与可见角色相比较大，可能是为了确保其肢体永远不会穿透周围的静态几何体，动态对象将在实际角色接触它们之前开始移动（由边界体积推动） - 使其看起来像角色被某种力场包围。

此外，当从盒子控制器切换到胶囊控制器时，推动效果不应改变。理想情况下，它应该独立于边界体积。

推动效果通常由游戏玩法决定，有时需要额外的代码，如反向运动解算器，这超出了 CCT 模块的范围。即使对于简单的用例，例如很难用胶囊控制器向前推动动态盒子：因为胶囊永远不会正好碰到盒子的中间，施加的力往往会旋转盒子 - 即使游戏玩法要求它应该移动在一条直线上。

因此，这是一个 CCT 模块最好与特定游戏代码耦合的领域，以实现特定游戏的特定解决方案。这种耦合可以通过许多不同的方式完成。对于简单的用例，使用 PxUserControllerHitReport::onShapeHit 回调将人工作用力应用于周围的动态对象就足够了。 SampleBridges 中说明了这种方法。

请注意，字符控制器确实使用重叠查询来确定附近的形状。因此，应该与角色交互的 SDK 形状（例如，角色应该推动的对象）必须将 PxShapeFlag::eSCENE_QUERY_SHAPE 标志设置为 true，否则 CCT 将无法检测到它们并且角色将直接穿过这些形状。

## Character Interactions: CCT-vs-CCT
--------------------------------------
CCT 之间（即两个 PxController 对象之间）的交互是有限的，因为在这种情况下，两个对象都是有效的运动学对象。 换句话说，它们的运动应该完全由用户控制，不允许 PhysX SDK 和 CCT 模块移动它们。

PxControllerFilterCallback 对象用于定义角色之间的基本交互。 它的 PxControllerFilterCallback::filter 函数可用于确定两个 PxController 对象是否应该相互碰撞：

```C++
bool PxControllerFilterCallback::filter(const PxController& a, const PxController& b) = 0;
```

要使 CCT 始终相互碰撞和滑动，只需返回 true。

要使 CCT 始终在彼此之间自由移动，只需返回 false。

否则，可以在此回调中实现自定义的或游戏驱动的过滤规则。 有时过滤会在运行时发生变化，并且可能只允许两个字符在有限的时间内相互通过。 当该有限时间到期时，角色可能会处于重叠状态，直到它们分开并再次相互靠近。 要自动分离重叠字符，可以使用以下功能：

```C++
void PxControllerManager::computeInteractions(PxF32 elapsedTime,
    PxControllerFilterCallback* cctFilterCb=NULL) = 0;
```

此函数是一个可选的助手，用于正确解决字符之间的重叠问题。 它应该在 PxController::move 调用之前每帧调用一次。 该函数不会直接移动字符，但会计算将在下一个 PxController::move 调用中使用的每个字符的重叠信息。

## Hidden Kinematic Actors
------------------------
CCT 库在幕后为每个受控角色创建了一个运动演员。 当调用 PxController::move 函数时，底层隐藏的运动学 PxActor 也会更新以反映物理场景中的 CCT 位置。

用户应该注意这些隐藏实体，因为场景中的演员总数将高于他们自己创建的人数。 此外，他们可能会从场景级碰撞查询中取回这些可能令人困惑的未知演员。

一种可能的策略是使用以下函数检索控制器的运动角色：

```C++
PxRigidDynamic* PxController::getActor() const;
```

然后使用 PxRigidDynamic::userData 字段用特殊标签标记这些角色。 这样，可以在碰撞查询或联系报告中轻松识别（并可能忽略）CCT 参与者。

## Time Stepping
-----------------
CCT 库内部使用的 Actor 遵循与任何其他 PhysX 对象相同的规则。 特别是，它们使用固定或可变的时间步长进行更新。 这可能很麻烦，因为 PxController 对象通常使用可变时间步长（通常使用两个渲染帧之间的经过时间）进行更新。

因此，PxController 对象（使用可变时间步长）可能并不总是与其运动actor（使用固定时间步长）完全同步。 SampleBridges 中显示了这种现象。

## Invalidating Internal Geometry Caches
----------------------------------------
CCT 库缓存每个字符周围的几何图形，以加快碰撞查询。角色的时间边界框是围绕角色运动的 AABB（它包含角色在开始和结束位置的体积）。缓存的空间体积由角色的时间边界框的大小乘以一个常数因子决定。这个常数因子是由 PxControllerDesc::volumeGrowth 为每个字符定义的。每次角色移动时，都会针对缓存的空间量测试其时间边界框。如果运动完全包含在该空间体积内，缓存的内容将被重用，而不是通过 PxScene 级查询重新生成。

<img src=".\image\Char_08.png" alt="Char_08" style="zoom:100%;" />

在 PhysX 3.3 及更高版本中，当缓存对象被更新或删除时，这些缓存应该自动失效。但是，也可以使用以下函数手动刷新这些缓存：

```C++
void PxController::invalidateCache();
```

在决定角色是否会随着与角色接触的物体的运动而移动之前，会自动执行许多测试以确定缓存的触摸物体是否仍然有效。这些自动有效性测试意味着在以下情况下，并非绝对需要使缓存无效：

+ 如果形状演员被释放
+ 如果形状被释放
+ 如果从演员身上移除形状
+ 如果演员从场景中移除或移动到另一个场景
+ 如果形状场景查询标志发生变化
+ 如果形状或场景的过滤参数发生了变化。

如果缓存的触摸对象不再实际接触角色并且希望角色不再随着该缓存对象的运动而行进，则必须使缓存无效。 如果由于更新的全局姿势或修改的几何体而使该对分离，则这适用。

## Runtime Tessellation
----------------------------
CCT 库非常强大，但有时会在角色与大三角形碰撞时遇到 FPU 精度问题。 这可能会导致角色不能顺利地滑过这些三角形，甚至无法穿透它们。 有效解决这些问题的一种方法是在运行时细分大三角形，用一组较小的三角形即时替换它们。 该库支持内置的曲面细分功能，通过此功能启用：

```C++
void PxControllerManager::setTessellation(bool flag, float maxEdgeLength);
```

第一个参数启用或禁用该功能。 第二个参数定义了三角形的最大允许边长，在它被镶嵌之前。 显然，较小的边长会导致在运行时创建更多的三角形，生成的三角形越多，碰撞它们的速度就越慢。

因此建议首先禁用该功能，只有在遇到碰撞问题时才启用它。 启用该功能时，建议使用最大可能的 maxEdgeLength 来解决遇到的问题。

<img src=".\image\Char_09.png" alt="Char_09" style="zoom:100%;" />

在屏幕截图中，角色站立的大洋红色三角形被镶嵌模块替换为较小的绿色三角形。 内部几何缓存由蓝色边界框表示。 请注意，仅保留接触该空间体积的绿色三角形。 因此，细分代码生成的三角形的确切数量取决于 maxEdgeLength 参数和 PxControllerDesc::volumeGrowth 参数。

## Troubleshooting
-------------------
本节介绍 CCT 库常见问题的常见解决方案。

### Character goes through walls in rare cases
------------------------------------------------
+ 尝试增加 PxControllerDesc::contactOffset。
+ 尝试使用 PxControllerManager::setTessellation 启用运行时曲面细分。 先从一个小的 maxEdgeLength 开始，看看是否能解决问题。 然后尽可能地增加该值。
+ 尝试使用 PxControllerManager::setOverlapRecoveryModule 启用重叠恢复模块。

### Tessellation performance issue
---------------------------------
+ 尝试微调 maxEdgeLength 参数。 使用仍然可以防止隧道问题的最大可能值。
+ 尝试减少 PxControllerDesc::volumeGrowth。

### The capsule controller manages to climb over obstacles higher than the step offset value
-------------------------------------------------------------------------------------------------
胶囊控制器设法越过高于步距偏移值的障碍物