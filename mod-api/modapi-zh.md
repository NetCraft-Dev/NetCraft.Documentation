# NetCraft-ModApi 参考

`NetCraft.ModApi` 是 NetCraft 暴露给模组的 API 表面。它有两重身份：对你而言它是一个 API 库；对它自己而言它就是一个普通模组（`id` 为 `netcraft-modapi`，自带 `ncmod.json` 和注入探针）。

本文随 API 一起增长。关于架构背景、与 Fabric 的差异以及如何编写模组，见 [modding-guide-zh.md](modding-guide-zh.md)。

- 程序集：`NetCraft.ModApi.dll`

- 依赖：`NetCraft`（主库）、`NetCraft.Game`

公共表面分为三个命名空间：

| 命名空间 | 内容 | 说明 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | 事件与订阅基类 `NcEvent<T>`、`Nc*` 门面、`Nc*` 对象句柄 | 包装层；公共表面上没有内核类型 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 注解 | 扩展点；规则绑定到内核的类名和方法名 |
| `NetCraft.ModApi.Internal` | 注入探针 | 不要直接引用 |

根命名空间 `NetCraft.ModApi` 只包含入口类 `ModApiEntry`。`Wrapper` 和 `Extension` 是两条并行路线；如何选择见 [modding-guide-zh.md 2.9](modding-guide-zh.md#29-两条路线包装层与扩展点)。

***

## 1. 快速开始

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //订阅返回一个句柄；释放它即取消订阅
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //你也可以用这个句柄做别的事
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

在 `ncmod.json` 中把 `entry` 指向这个类，并让 `hooks` 留空 —— 下面这些事件全部由 ModApi 自己的探针提供。

***

## 2. 事件

所有事件都在 `NetCraft.ModApi.Wrapper` 下；写了 `using NetCraft.ModApi.Wrapper;` 后即可使用。

### 2.1 汇总表

| 事件                           | Args 类型              | 触发时机                                                   | 侧   | ModApi 注入点                                        |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick`             | `ServerTickArgs`       | 服务端主循环的每个 tick                        | 服务端 | `DedicatedServer::Tick` (Mark)                           |
| `ServerEvents.Started`          | `ServerPhaseArgs`      | 主循环已启动，在打印 `Done (x.xxxs)!` 之后      | 服务端 | `MinecraftServer::Run` (Mark)                            |
| `ServerEvents.Stopping`         | `ServerPhaseArgs`      | 服务端开始关闭；玩家即将被断开连接 | 服务端 | `DedicatedServer::Stop` (Mark)                    |
| `ServerEvents.CommandRegister`  | `CommandRegisterArgs`  | 所有内置命令都已注册完成                | 服务端 | `EffectCommand::Register` call site (CallSite)           |
| `ServerEvents.PlayerJoin`       | `PlayerJoinArgs`       | 加入封包序列已发送                    | 服务端 | `PlayerList::PlaceNewPlayer` call site (CallSite)        |
| `ServerEvents.PlayerLeave`      | `PlayerLeaveArgs`      | 玩家从在线列表中被移除                       | 服务端 | `PlayerList::RemovePlayer` call site (CallSite)          |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | 断开连接封包已发送且连接已关闭              | 服务端 | `ServerPlayer::Disconnect` call site (CallSite)          |
| `ServerEvents.PlayerHurt`       | `PlayerHurtArgs`       | 伤害被实际结算                                     | 服务端 | `PlayerList::HurtPlayer` call site (CallSite)            |
| `ServerEvents.PlayerDeath`      | `PlayerDeathArgs`      | 生命值归零后立即触发                   | 服务端 | `PlayerList::RespawnPlayer` call site (CallSite)         |
| `ServerEvents.PlayerChat`       | `PlayerChatArgs`       | 聊天广播之后                                  | 服务端 | `ServerGamePacketListenerImpl::HandleChat` call site (CallSite) |
| `ServerEvents.ChunkLoaded`      | `ChunkLoadedArgs`      | 区块首次进入内存                    | 服务端 | `ServerChunkCache::set_ChunkLoaded` assignment site (CallSite) |
| `ServerEvents.ChunkUnloaded`    | `ChunkUnloadedArgs`    | 区块离开内存                                       | 服务端 | `ServerChunkCache::set_ChunkUnloaded` assignment site (CallSite) |
| `ServerEvents.ChunkSaved`       | `ChunkSavedArgs`       | 区块写入磁盘前拍摄快照        | 服务端 | `ServerChunkCache::set_ChunkSaveSink` assignment site (CallSite) |
| `ServerEvents.CommandExecuted`  | `CommandExecutedArgs`  | 一条命令运行结束；语法错误和权限拒绝也算 | 服务端 | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick`        | `LevelTickArgs`        | 世界 tick，每个 tick 对每个已加载世界各一次                | 服务端 | `PersistentServerLevel::Tick` (CallSite)                 |
| `ServerEvents.SavedDataSaving`  | `SavedDataSavingArgs`  | 存档数据写入磁盘，比区块保存晚一步 | 服务端 | `SavedDataStorage::ScheduleSave` (CallSite)          |
| `ServerEvents.BlockChanged`     | `BlockChangedArgs`     | 方块状态改变，即将同步给客户端             | 服务端 | `IBlockUpdateSink::BlockChanged` (CallSite)              |
| `ServerEvents.BlockBroken`      | `BlockBrokenArgs`      | 方块被破坏；玩家挖掘和红石自毁都算 | 服务端 | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped`      | `ItemDroppedArgs`      | 掉落物实体已生成，包括破坏方块的掉落物和烹饪产物 | 服务端 | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived`  | `PacketReceivedArgs`   | 每个入站封包被排入处理器，包括握手和状态阶段 | 两侧   | `PacketProcessor::ScheduleIfPossible` and `HandleNow` (CallSite) |
| `ClientEvents.Tick`             | `ClientTickArgs`       | 客户端主循环的每个 tick                        | 客户端 | `MinecraftClient::Tick` (Mark)                           |

玩家事件的顺序：死亡嵌套在受伤流程内部，所以 `PlayerDeath` 先于对应的 `PlayerHurt`；`PlayerLeave` 和 `PlayerDisconnect` 是两回事 —— 前者表示从在线列表移除（即便 `/kick` 也要等到连接断开才触发），后者表示连接断开本身，两者不保证成对出现。

### 2.2 订阅与取消订阅

```csharp
IDisposable Subscribe(Action<T> handler)
```

- 对同一个事件多次订阅，通知按订阅顺序送达。

- 派发时对回调列表拍快照，因此在回调内部订阅或取消订阅不影响当前这次派发。

- 不取消订阅就永远有效；模组没有卸载机制，所以通常不需要手动取消订阅。

### 2.3 Args 类型

**`ServerTickArgs`**

| 属性    | 类型   | 说明                                |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | 自本次启动以来的 tick 计数，从 1 开始 |

注意这是 ModApi 自己计数的，不是内核的 `TickCount`。

**`ClientTickArgs`**

| 属性    | 类型   | 说明                           |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | 同上，在客户端侧独立计数 |

**`ServerPhaseArgs`**

| 属性 | 类型     | 说明                        |
| -------- | -------- | ---------------------------- |
| `Phase`  | `string` | 阶段名，`started` 或 `stopping` |

该字段与事件本身重复；保留它是为了让日志能使用统一的格式。

**`CommandRegisterArgs`**

| 成员                               | 类型                                    | 说明            |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher`                         | `CommandDispatcher<CommandSourceStack>` | 内核的命令派发器 |
| `Register(name, description, build)` | method                                  | 注册一条命令并记入账本，见 3.1 |

**`PlayerJoinArgs`**

| 属性      | 类型           | 说明                                          |
| ------------- | -------------- | ---------------------------------------------- |
| `Player`      | `ServerPlayer` | 刚加入的玩家；加入封包已发送，状态可以安全读取 |
| `ProfileName` | `string`       | 玩家名                                    |

**`PlayerLeaveArgs`**

| 属性  | 类型           | 说明                                 |
| --------- | -------------- | ------------------------------------- |
| `Player`  | `ServerPlayer` | 正在离开的玩家，此时已不在在线列表中 |
| `Removed` | `bool`         | 是否真的被移除；重复移除时为 `false` |

**`PlayerDisconnectArgs`**

| 属性 | 类型           | 说明                                 |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | 已断开连接的玩家               |
| `Reason` | `string`       | 断开原因；以组件形式给出时是纯文本 |

断开连接封包已发送且连接已关闭；此时给该玩家发封包不会有任何效果。

**`PlayerHurtArgs`**

| 属性   | 类型            | 说明                                        |
| ---------- | --------------- | -------------------------------------------- |
| `Player`   | `ServerPlayer`  | 被伤害的玩家                      |
| `Attacker` | `ServerPlayer?` | 造成伤害的玩家；环境伤害和命令伤害时为 `null` |
| `Amount`   | `float`         | 本次伤害量                      |

无敌帧期间或死亡后不触发（内核的 `Hurt` 返回 `false`）。

**`PlayerDeathArgs`**

| 属性   | 类型            | 说明                          |
| ---------- | --------------- | ------------------------------ |
| `Player`   | `ServerPlayer`  | 死亡的玩家            |
| `Attacker` | `ServerPlayer?` | 击杀者；没有时为 `null` |

内核在生命值归零后立即重置，所以事件触发时玩家已经满血在重生点；死亡瞬间的坐标和掉落物拿不到。

**`PlayerChatArgs`**

| 属性     | 类型     | 说明                |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | 发送者名字          |
| `Message`    | `string` | 纯文本消息   |

这个事件是**只读通知**：原方法已经广播了消息，所以在这里修改 `Message` 没有任何效果。

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

这三个事件共用同一种 args 形状：

| 属性 | 类型  | 说明              |
| -------- | ----- | ------------------ |
| `X`      | `int` | 区块坐标 X |
| `Z`      | `int` | 区块坐标 Z |

最容易用错的是 `ChunkSaved`：内核要求这个回调**同步拍摄快照**，而序列化和写盘由内核自己异步完成。所以在这个回调里做耗时工作会直接拖慢区块卸载，破坏性操作（移除方块、改动容器）也不该放这里 —— 它只承诺快照这一刻。

`ChunkUnloaded` 触发时方块实体已经随区块一起清理完毕；如果你想读方块，用 `ChunkSaved`（它同样访问不到方块）或更早的时点。

**`CommandExecutedArgs`**

| 属性  | 类型                   | 说明                                 |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string`               | 原始命令文本；聊天命令不带前导斜杠 |
| `Result`  | `int`                  | 命令返回值；0 表示失败或被拒绝 |
| `Source`  | `CommandSourceStack?`  | 命令来源；走玩家重载路径时为 `null` |
| `Player`  | `ServerPlayer?`        | 发出命令的玩家；从控制台发出时为 `null` |

事件在命令结束**之后**触发，无法改变执行。语法错误和权限拒绝也会走这里；用 `Result` 区分它们。玩家从聊天栏发出的命令走 `Execute(ServerPlayer, string)` 重载，此时内核在内部构造命令来源，所以这种情况下 `Source` 为 `null`，只设置 `Player`。

**`LevelTickArgs`**

| 属性       | 类型                    | 说明                                        |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level`        | `NcLevel`               | 本 tick 推进的世界                 |
| `RunsNormally` | `bool`                  | 是否正常推进；`/tick freeze` 期间为 `false` |

每个 tick 对每个已加载世界触发一次，所以一个多世界的存档每 tick 会收到多次。触发点在世界 tick **已完成**之后；它是观察点，不是拦截点。

**`SavedDataSavingArgs`**

| 属性  | 类型                | 说明                       |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage`  | 正在持久化的存档数据表 |

这与 `ServerEvents.ChunkSaved` 是不同路径：区块那个只拍快照并异步写入，而这个在同步写完之后触发。世界时钟、游戏规则和世界边界数据都走这里。

**`BlockChangedArgs`**

| 属性 | 类型         | 说明                       |
| -------- | ------------ | --------------------------- |
| `Pos`    | `BlockPos`   | 被改变方块的位置 |
| `State`  | `BlockState` | 改变后的方块状态  |

状态已经写入区块，即将同步给客户端，所以无法在这里修改这次改变本身。红石元件在行为回调中改变自身状态也走这条路径；频率很高，不要在回调里做耗时工作。

**`BlockBrokenArgs`**

| 属性 | 类型            | 说明                                      |
| -------- | --------------- | ------------------------------------------ |
| `Pos`    | `BlockPos`      | 被破坏方块的位置                |
| `Player` | `ServerPlayer?` | 破坏者；红石等非玩家原因时为 `null` |

只在方块被实际替换时触发；空位置和被拒绝的破坏不触发。破坏效果和掉落物已经处理完毕，所以你在事件里读到的是结果。

**`ItemDroppedArgs`**

| 属性 | 类型        | 说明                   |
| -------- | ----------- | ----------------------- |
| `Pos`    | `BlockPos`  | 掉落物出现的位置 |
| `Stack`  | `ItemStack` | 掉落物物品堆   |

破坏方块的掉落物和营火烹饪产物都走这里。空的物品堆不会生成实体，所以没有事件。

**`PacketReceivedArgs`**

| 属性        | 类型     | 说明                    |
| --------------- | -------- | ------------------------ |
| `Listener`      | `object` | 接收该封包的监听器 |
| `Packet`        | `object` | 封包对象本身  |
| `IsServerbound` | `bool`   | 是否为服务端绑定（serverbound）封包 |

每个入站封包都会触发，覆盖全部四个阶段：握手、状态、配置和游戏。移动封包每 tick 会到好几次，不要在回调里做耗时工作。封包已经解码成对象但还没进入业务层；要区分类型，自己检查 `Packet`。出站封包不在这个事件的范围内。

***

## 3. 扩展点

### 3.1 命令注册

时机是 `ServerEvents.CommandRegister`。不要缓存这个事件的 args；内部命令树只在启动时构建一次。

```csharp
public void Register(
    string name,                                          //命令字面量，不带斜杠
    string description,                                   //一行描述，显示在 /ncmapi 账本中
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //挂载参数和执行器
```

你拿到的 `build` 是内核的 brigadier builder；按内核的方式写参数、子命令和权限判定：

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //权限判定
            .Executes(context => { /* ... */ return 1; })));
```

带参数时：

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //示意
                return 1;
            })));
```

要点：

- 直接调用 `args.Dispatcher.Register(...)` 也能装上命令，但它不进账本，`/ncmapi` 不会显示。想让它出现在列表里就用 `args.Register`。

- 命令默认没有权限限制；需要的话自己加 `.Requires(...)`。

- 执行时的行为完全由你决定；ModApi 不拦截。

### 3.2 查看已注册的命令

内置了一个 `/ncmapi`，需要权限等级 2：

```
/ncmapi
Commands registered via NetCraft-ModApi: 2 total
/ncmapi  list commands registered via NetCraft-ModApi
  /ncmapi
/tpall  teleport all players to the executor
  /tpall
/heal  heal the target players
  /heal <targets>
```

用法行是根据命令树的节点结构即时计算的：字面量按名字写出，参数用尖括号包裹，本身可执行的中间节点也会有自己的一行。

***

## 4. 服务端门面

本章的门面都在 `NetCraft.ModApi.Wrapper` 下；写了 `using NetCraft.ModApi.Wrapper;` 后即可使用。

门面是把散落在内核各处的能力汇集到少数几个入口点的 `Nc*` 静态类。内核实例会在主循环启动时被探针捕获；当 `NcServer.IsAvailable` 为 false 时下面的一切都会抛异常 —— 只在事件回调里用它们，不要在 `Init` 里用。

| 门面 | 用途 |
| --- | --- |
| `NcServer` | 服务端实例、tick 速率、命令、实体追踪、玩家数据、游戏规则、广播、命令执行 |
| `NcPlayers` | 在线玩家查询与操作（踢出、传送、生命值、游戏模式、权限） |
| `NcWorld` | 主世界方块读写与破坏、天气、时间、边界、时钟、音效、世界事件；用 `NcLevel` 句柄访问其它维度，坐标是普通的 `x y z` 整数 |
| `NcRegistries` | 按名字查找内置注册表（方块、物品、流体、状态效果、生物群系、粒子、实体、方块实体） |
| `NcRecipes` | 配方查询（网格合成、切石、烹饪；按 id 取配方） |
| `NcLists` | 列表与配置（白名单、管理员、封禁、`server.properties`） |
| `NcStartup` | 启动参数（内核不认识的 token 和按名字订阅） |

`NcPlayer` 不是静态门面，而是**对象句柄**：`NcPlayers.All` / `Find` 返回它，玩家事件里的 `Player` / `Attacker` 也是它。句柄只读且由探针构造；模组拿不到内核的 `ServerPlayer` —— 这是「公共表面上没有内核类型」的第一块基石。同一个内核玩家总是映射到同一个句柄，内部用弱引用缓存，玩家一登出就自动失效。

`NcLevel` 对世界遵循同样的形状。`NcWorld.Overworld` / `Nether` / `End` 和 `NcWorld.Get("minecraft:the_nether")` 返回它，`LevelTickArgs.Level` 也是一个。它携带维度 id、时间、天气、建筑高度、tick 计数和区块强制加载；方块操作留在 `NcWorld` 上，接收句柄加 `x y z`。`BlockPos` 从不出现，所以模组的 dll 不携带对内核世界类型的引用。

### 4.1 注册表

`NcRegistries` 既提供整张表，也提供按名字查找。整张表用于遍历和基于标签的查找；按名字查找用于获取单个元素：

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //方块的默认状态
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//遍历整张表
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

注册表在启动过程中逐步组装，而模组在组装完成前就已加载，所以不要在 `Init` 里缓存任何查出来的东西 —— 组装仍在进行，缓存下来的值会是空引用或过期值。目前 `BuiltInRegistries.BootStrap` 还是空实现；每个注册表由各自的 Bootstrap 分别填充，数据驱动的那些（生物群系、配方等）在数据包加载接通之前条目很少。

### 4.2 配方

`NcRecipes` 由一个从数据包加载的配方表支撑；`/reload` 会替换整张表，所以不要在跨重载时持有 `RecipeHolder`。

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //计算合成网格的输出
    var recipes = NcRecipes.StonecuttingFor(stack);         //该输入可用的切石配方
    var smelting = NcRecipes.CookingFor("smelting", stack); //按烹饪类型查找
    var byId = NcRecipes.Find("minecraft:oak_planks");      //按 id 取配方
}
```

***

## 5. 内部实现

写模组不需要这一节，但调试时可能有帮助。

### 5.1 探针

| 类                                                    | 形式        | 职责                                              |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)`                  | Mark ×4     | 所有「某事发生了」的信号汇聚到一个方法，按 `label` 分发到对应事件 |
| `Internal.CommandProbe.OnCommandsReady(object)`          | CallSite    | 替换对 `EffectCommand::Register` 的调用；在恢复原调用后触发 `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)`                | CallSite ×6 | 玩家事件，每个注入点一个方法；在恢复原调用后发布事件 |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | 区块事件，注入在 `ServerChunkCache` 三个回调属性的赋值点；在把控制权交还内核前叠加一层包装委托 |
| `Internal.BlockProbe.OnXxx(...)`                         | CallSite ×3 | 方块事件；破坏和掉落注入 `ServerBlockUpdates`，状态改变注入 `IBlockUpdateSink` 上的接口方法 |

`SignalProbe` 的签名只接收 `string`，而 `CommandProbe`、`PlayerProbe` 和 `LevelProbe` 的参数声明为 `object` —— 这是刻意的：组装期间 `Lead.Hook` 会解析替换方法的签名，一旦其中出现内核类型，解析它就会提前拉起内核程序集，注入便错过窗口。内核类型只出现在方法体内部，那时代码已经在运行了。

签名中唯一不能用 `object` 的是值类型参数和返回值：`object` 在栈上是引用，而 `float`/`bool` 是值，不匹配就是非法 IL。所以 `PlayerProbe.OnHurtPlayer` 的伤害量保留 `float`，`OnRemovePlayer` 和 `OnHurtPlayer` 保留 `bool` 返回值。

`BlockProbe` 是这条约束的延伸：方块位置和状态是两个值类型 `BlockPos`/`BlockState`，只能以它们本身写进签名。这两个类型来自 `NetCraft.Primitives` 和 `NetCraft.Registry`，两者都不在注入名单上，所以组装期间解析它们不会提前拉起待改写的程序集。

### 5.2 注入点清单

ModApi 的 `ncmod.json` 包含二十四条规则，与 2.1 中的表格一一对应。要改动注入点或新增规则，就编辑这个文件；改完后重新构建（它是嵌入资源），再把生成的 dll 放回 `mods/` —— 后者已由 `NetCraft.ModApi.csproj` 中的 `DeployModToHosts` 自动完成，漏掉它表现为规则完全不生效。

`CommandManager::Execute` 有两个重载共用一条规则。`Lead.Hook` 的 CallSite 按「类型 + 方法名」匹配调用点，而不是按参数列表，两个重载都接收两个参数，所以探针可以把第一个参数取为 `object`，再按真实类型分发。

两条 `PacketProcessor` 规则是互补的：游戏阶段的封包经 `ScheduleIfPossible` 进入主线程队列，而握手和状态阶段经 `HandleNow` 立即处理；任何一个给定的封包只会经过其中一条。只注入前者会漏掉握手和状态阶段 —— 而这两个阶段恰好最容易用脚本探测，所以调试时这容易被误读为「规则没有生效」。

三条区块规则注入的是 `ServerChunkCache` 三个回调属性的**赋值点**，不是读取点。原因是这三个属性是单播的，且在 `PersistentServerLevel` 构造时已被内核自己占用（它们注入了保存和方块实体清理逻辑）；模组直接赋值会覆盖内核的那份 —— 卸载不落盘、方块实体不清理，而且毫无报错。注入赋值点可以让内核回调和探针在那一刻串起来；赋值只发生一次，之后每次触发都多一层委托转发。

`PlayerList::RespawnPlayer` 是私有的，所以探针无法恢复原调用；这一条走反射（每次死亡调用一次，开销可忽略）。这也给内核对齐留了扇门：将来为它加上 `InternalsVisibleTo`，就能换成直接调用。

### 5.3 账本

`Internal.NcCommandRegistry` 记录通过 `args.Register` 注册的命令。它只是一个账本，不参与命令执行；命令本身装在内核派发器上，所以即便账本出问题，命令也照常工作。

***

## 6. 待补充

以下是位置已确认但尚未做成事件的注入点（该清单由仓库根目录的 `__scan_mod_api.py` 生成）：

| 方向         | 候选注入点                                                        |
| ----------------- | ---------------------------------------------------------------------------- |
| 实体          | `ClientLevel::AddEntity`, `Entity::Die`                                      |
| 世界             | 世界加载与卸载、`ServerChunkCache` 区块批处理                     |
| 地形生成 | 按 `ChunkStatus` 分阶段的 `ChunkGenerator::Generate`、`WorldGenRegion::SetBlockState` |
| 命令执行 | `CommandSourceStack::SendSuccess` / `SendFailure`（响应那一半，调用点很多） |
| 网络           | 按封包类型的 `ServerGamePacketListenerImpl::HandleXxx`（目前只有统一入口） |

已完成的方向：世界 tick 变成了 `ServerEvents.LevelTick`，存档数据持久化变成了 `ServerEvents.SavedDataSaving`，命令执行变成了 `ServerEvents.CommandExecuted`，方块变成了 `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`。

方块这一格在 `SetBlock` 上注入行不通：它有两个默认参数 `notifyNeighbors` 和 `strict`，所以编译后的调用点会带 4 到 6 个参数不等，而 CallSite 按「类型 + 方法名」匹配、不看参数列表，一个替换方法无法应对三种栈形状。于是改为注入两个地方：状态同步注入 `IBlockUpdateSink::BlockChanged`（`ServerLevel.SetBlock` 里唯一的接口调用，覆盖每一次带客户端同步的改动），破坏和掉落注入 `ServerBlockUpdates` 自己的方法。

实体这一格比较麻烦，因为 `Entity` 定义在 `NetCraft.Registry` 中，而它不在注入名单上，所以以它为目标的调用点无法被改写。`ClientLevel::AddEntity` 在客户端程序集里，可行；死亡事件则需要先解决 `Registry` 能否被改写。

网络这一格方面，`HandleChat` 早已存在，`NetworkEvents.PacketReceived` 也提供了统一入口，所以按 `HandleXxx` 逐个注入的价值低得多；只有需要按封包类型细粒度过滤的场景才值得增加。

世界加载与卸载在 NC 这侧没有汇聚点：`DedicatedServer::CreateLevel` 是私有的，只能像 `PlayerList::RespawnPlayer` 那样用反射恢复原调用；卸载路径更加分散。要做的话，先想清楚事件 args 应该是什么。

剩下这些都需要对象引用（实体实例等），所以必须用 `CallSite` 而不是 `Mark`；如果参数中出现内核值类型，只能以它本身写进替换方法的签名。

接口表面清单（`__modapi_api.txt`，由 `__scan_mod_api.py --api` 生成）也审查过了：够得上能力入口的条目已按领域归入第 4 章的门面，其余未暴露的分为三类 —— 协议与封包处理（`Network.Protocol.*`）、渲染与模型（`Client.Render.*`）、地形生成与密度函数（`LevelGen.*`）。这些属于内核内部；直接使用会把模组绑死在实现细节上，所以应先在核心里开出稳定接口。

服务端侧还有两样没有包装成门面：`ReloadableServerResources` 实例挂在 `DedicatedServer` 上，而 ModApi 不引用 `NetCraft.Server`，要包装它得先在内核基类上开一个属性；`ChunkSender` 和 `ServerWorldBorderListener` 是内部流程，模组没有使用场景。
