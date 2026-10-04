# NetCraft-ModApi 参考

`NetCraft.ModApi` 是 NetCraft 向 mod 公开的 API 表面。它有两个身份：对你来说它是一个API库；对你来说它是一个API库；对你来说它是一个API库。就其本身而言，它是一个普通的 mod（`id` 是 `netcraft-modapi`，运送自己的 `ncmod.json` 和注入探针）。

该文件随着 API 的增长而增长。有关架构背景、与 Fabric 的差异以及如何编写 mod，请参阅 [modding-guide.md](modding-guide.md)。

- 装配：`NetCraft.ModApi.dll`

- 依赖项：`NetCraft`（主库）、`NetCraft.Game`

公共表面分为三个命名空间：

|命名空间|内容 |笔记|
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` |事件和订阅基类 `NcEvent<T>`、`Nc*` 外观、`Nc*` 对象句柄 |包裹层；公共表面上没有内核类型 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 注释 |扩展点；规则绑定到内核类和方法名称|
| `NetCraft.ModApi.Internal` |注射探针|不要直接引用 |

根命名空间`NetCraft.ModApi`仅包含入口类`ModApiEntry`。 `Wrapper`和`Extension`是两条平行的路线；如何选择请参见[modding-guide.md 2.9](modding-guide.md#29-two-routes-wrapper-layer-and-extension-points)。

***

## 1. 快速启动

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //subscription returns a handle; disposing it unregisters
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //you can also do something else with the handle
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

在`ncmod.json`中，将`entry`指向该类，并将`hooks`留空——下面的事件都是由ModApi自己的探针提供的。

***

## 2. 活动

所有事件均位于 `NetCraft.ModApi.Wrapper` 下；在 `using NetCraft.ModApi.Wrapper;` 之后它们可用。

### 2.1 汇总表

|活动 |参数类型 |触发|侧面| ModApi挂钩点|
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick` | `ServerTickArgs` |服务器主循环的每个滴答声|服务器 | `DedicatedServer::Tick`（马克）|
| `ServerEvents.Started` | `ServerPhaseArgs` |打印 `Done (x.xxxs)!` 后主循环开始 |服务器 | `MinecraftServer::Run`（马克）|
| `ServerEvents.Stopping` | `ServerPhaseArgs` |服务器开始关闭；玩家即将断线|服务器 | `DedicatedServer::Stop`（马克）|
| `ServerEvents.CommandRegister` | `CommandRegisterArgs` |所有内置命令均已注册 |服务器 | `EffectCommand::Register` 呼叫站点 (CallSite) |
| `ServerEvents.PlayerJoin` | `PlayerJoinArgs` |加入数据包序列已发送 |服务器 | `PlayerList::PlaceNewPlayer` 呼叫站点 (CallSite) |
| `ServerEvents.PlayerLeave` | `PlayerLeaveArgs` |玩家已从在线列表中删除|服务器 | `PlayerList::RemovePlayer` 呼叫站点 (CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` |断开数据包发送并关闭连接 |服务器 | `ServerPlayer::Disconnect` 呼叫站点 (CallSite) |
| `ServerEvents.PlayerHurt` | `PlayerHurtArgs` |实际造成的伤害|服务器 | `PlayerList::HurtPlayer` 呼叫站点 (CallSite) |
| `ServerEvents.PlayerDeath` | `PlayerDeathArgs` |生命值重置为零后立即 |服务器 | `PlayerList::RespawnPlayer` 呼叫站点 (CallSite) |
| `ServerEvents.PlayerChat` | `PlayerChatArgs` |聊天直播后 |服务器 | `ServerGamePacketListenerImpl::HandleChat` 呼叫站点 (CallSite) |
| `ServerEvents.ChunkLoaded` | `ChunkLoadedArgs` | chunk 第一次进入内存 |服务器 | `ServerChunkCache::set_ChunkLoaded` 分配站点 (CallSite) |
| `ServerEvents.ChunkUnloaded` | `ChunkUnloadedArgs` |块离开内存|服务器 | `ServerChunkCache::set_ChunkUnloaded` 分配站点 (CallSite) |
| `ServerEvents.ChunkSaved` | `ChunkSavedArgs` |在将块写入磁盘之前拍摄的快照服务器 | `ServerChunkCache::set_ChunkSaveSink` 分配站点 (CallSite) |
| `ServerEvents.CommandExecuted` | `CommandExecutedArgs` |命令运行完毕；语法错误和权限拒绝也很重要服务器 | `CommandManager::Execute`（呼叫站点）|
| `ServerEvents.LevelTick` | `LevelTickArgs` |级别刻度，每个刻度每个加载级别一次 |服务器 | `PersistentServerLevel::Tick`（呼叫站点）|
| `ServerEvents.SavedDataSaving` | `SavedDataSavingArgs` |保存的数据写入磁盘，比块保存晚一步 |服务器 | `SavedDataStorage::ScheduleSave`（呼叫站点）|
| `ServerEvents.BlockChanged` | `BlockChangedArgs` |区块状态已更改，即将同步到客户端 |服务器 | `IBlockUpdateSink::BlockChanged`（呼叫站点）|
| `ServerEvents.BlockBroken` | `BlockBrokenArgs` |块损坏；玩家挖矿和红石自毁都算数 |服务器 | `ServerBlockUpdates::BreakBlock`（呼叫站点）|
| `ServerEvents.ItemDropped` | `ItemDroppedArgs` |生成掉落物品实体，包括方块破坏掉落物和烹饪产品 |服务器 | `ServerBlockUpdates::SpawnDrop`（呼叫站点）|
| `NetworkEvents.PacketReceived` | `PacketReceivedArgs` |每个入站数据包都排队等待处理程序，包括握手和状态阶段 |两者 | `PacketProcessor::ScheduleIfPossible` 和 `HandleNow` (CallSite) |
| `ClientEvents.Tick` | `ClientTickArgs` |客户端主循环的每个周期 |客户| `MinecraftClient::Tick`（马克）|

玩家事件的顺序：死亡嵌套在伤害流中，因此 `PlayerDeath` 位于相应的 `PlayerHurt` 之前； `PlayerLeave` 和 `PlayerDisconnect` 是两个不同的东西 - 前者意味着从在线列表中删除（即使在 `/kick` 之后它也只会在连接断开时触发），后者意味着连接断开本身，并且不能保证两者成对出现。

### 2.2 订阅和注销

```csharp
IDisposable Subscribe(Action<T> handler)
```

- 多次订阅同一事件会按订阅顺序发送通知。

- 调度会获取回调列表的快照，因此从回调内部订阅或取消注册不会影响当前的调度。

- 无需注销，永久有效； mods 不提供卸载机制，因此通常不需要手动注销。

### 2.3 参数类型

**`ServerTickArgs`**

|物业 |类型 |笔记|
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` |自本次发布以来的滴答计数，从 1 开始 |

请注意，这是由 ModApi 本身计算的，而不是内核的 `TickCount`。

**`ClientTickArgs`**

|物业 |类型 |笔记|
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` |同上，在客户端独立计数|

**`ServerPhaseArgs`**

|物业 |类型 |笔记|
| -------- | -------- | ---------------------------- |
| `Phase` | `string` |阶段名称，`started` 或 `stopping` |

该字段复制事件本身；保留它是为了使日志记录可以使用一种统一的格式。

**`CommandRegisterArgs`**

|会员|类型 |笔记|
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher` | `CommandDispatcher<CommandSourceStack>` |内核的命令调度程序|
| `Register(name, description, build)` |方法|注册一个命令并将其记录在账本中，参见3.1 |

**`PlayerJoinArgs`**

|物业 |类型 |笔记|
| ------------- | -------------- | ---------------------------------------------- |
| `Player` | `ServerPlayer` |刚刚加入的玩家；加入数据包已发送，状态可以安全读取 |
| `ProfileName` | `string` |玩家姓名 |

**`PlayerLeaveArgs`**

|物业 |类型 |笔记|
| --------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` |离开的玩家，此时已不再在线列表中 |
| `Removed` | `bool` |是否实际移除； `false`关于重复删除|

**`PlayerDisconnectArgs`**

|物业 |类型 |笔记|
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` |断开连接的玩家 |
| `Reason` | `string` |断开原因；作为组件给出时的纯文本 |

断开数据包已发送，连接已关闭；现在向该播放器发送数据包没有任何效果。

**`PlayerHurtArgs`**

|物业 |类型 |笔记|
| ---------- | --------------- | -------------------------------------------- |
| `Player` | `ServerPlayer` |受伤的球员|
| `Attacker` | `ServerPlayer?` |造成伤害的玩家； `null` 环境和指挥损害 |
| `Amount` | `float` |这次的伤害量|

在无敌帧期间或死亡后不会触发（内核的 `Hurt` 返回 `false`）。

**`PlayerDeathArgs`**

|物业 |类型 |笔记|
| ---------- | --------------- | ------------------------------ |
| `Player` | `ServerPlayer` |死去的玩家|
| `Attacker` | `ServerPlayer?` |凶手； `null` 当没有 |

内核在生命值达到零后立即重置，因此当事件触发时，玩家在重生点已经处于完全生命值；死亡时的坐标和掉落物不可用。

**`PlayerChatArgs`**

|物业 |类型 |笔记|
| ------------ | -------- | -------------------- |
| `SenderName` | `string` |发件人姓名 |
| `Message` | `string` |纯文本消息 |

这个事件是一个**只读通知**：原来的方法已经广播了消息，所以这里改变`Message`没有效果。

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

这三个事件共享相同的参数形状：

|物业 |类型 |笔记|
| -------- | ----- | ------------------ |
| `X` | `int` |块坐标 X |
| `Z` | `int` |块坐标 Z |

最容易出错的是`ChunkSaved`：内核需要此回调**同步拍摄快照**，而序列化和磁盘写入由内核本身异步完成。因此，此回调中的耗时工作会直接减慢块卸载速度，并且破坏性操作（删除块、更改库存）也不应该放在这里 - 它只承诺快照时刻。

当 `ChunkUnloaded` 触发时，块实体已经与块一起被清理；如果要读取块，请使用 `ChunkSaved`（也不能访问块）或更早的点。

**`CommandExecutedArgs`**

|物业 |类型 |笔记|
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string` |原始命令文本；聊天命令不带前导斜杠 |
| `Result` | `int` |命令返回值； 0 表示失败或拒绝 |
| `Source` | `CommandSourceStack?` |命令源； `null` 在玩家过载路径上 |
| `Player` | `ServerPlayer?` |发出命令的玩家； `null` 从控制台发出时 |

该事件在命令完成后**触发；它不能改变执行。语法错误和权限拒绝也会出现在这里；使用 `Result` 来区分它们。玩家从聊天栏发送的命令会经过 `Execute(ServerPlayer, string)` 重载，其中内核在内部构建命令源，因此在这种情况下 `Source` 是 `null`，并且仅设置 `Player`。

**`LevelTickArgs`**

|物业 |类型 |笔记|
| -------------- | ----------------------- | -------------------------------------------- |
| `Level` | `NcLevel` |这个价格上涨了 |
| `RunsNormally` | `bool` |是否正常进展； `/tick freeze` 期间的 `false` |

每个加载级别每个刻度触发一次，因此多级世界每个刻度接收多个。触发点是在水平刻度**完成**之后；它是一个观察点，而不是拦截点。

**`SavedDataSavingArgs`**

|物业 |类型 |笔记|
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage` |保存的数据表被持久化|

这是与 `ServerEvents.ChunkSaved` 不同的路径：块仅拍摄快照并异步写入，而该块在同步写入完成后触发。世界时钟、游戏规则和世界边界数据都经过它。

**`BlockChangedArgs`**

|物业 |类型 |笔记|
| -------- | ------------ | --------------------------- |
| `Pos` | `BlockPos` |改变块的位置 |
| `State` | `BlockState` |更改后的块状态 |

状态已写入块并即将同步到客户端，因此此处无法修改更改本身。红石组件在行为回调中改变自己的状态也通过此路径退出；它是高频的，所以不要在回调中做耗时的工作。

**`BlockBrokenArgs`**

|物业 |类型 |笔记|
| -------- | --------------- | ------------------------------------------ |
| `Pos` | `BlockPos` |损坏块的位置|
| `Player` | `ServerPlayer?` |断路器； `null` 对于非玩家原因，例如红石 |

仅当块实际被替换时才触发；空仓和拒绝突破不会触发它。中断效果和掉落已经处理完毕，因此您在事件中看到的就是结果。

**`ItemDroppedArgs`**

|物业 |类型 |笔记|
| -------- | ----------- | ----------------------- |
| `Pos` | `BlockPos` |掉落物品出现的位置 |
| `Stack` | `ItemStack` |掉落的物品堆栈 |

方块掉落物和篝火烹饪产品都会经过这里。空的项目堆栈不会产生任何实体，因此没有事件。

**`PacketReceivedArgs`**

|物业 |类型 |笔记|
| --------------- | -------- | ------------------------ |
| `Listener` | `object` |接收此数据包的侦听器 |
| `Packet` | `object` |数据包对象本身 |
| `IsServerbound` | `bool` |是否是服务器绑定数据包 |

针对每个入站数据包触发，涵盖所有四个阶段：握手、状态、配置和播放。每个时钟周期移动数据包都会到达几次，因此不要在回调中执行耗时的工作。数据包已解码为对象，但尚未进入业务层；要区分类型，请自行检查 `Packet`。出站数据包超出此事件的范围。

***

## 3. 扩展点

### 3.1 命令注册

时间是`ServerEvents.CommandRegister`。不缓存该事件的参数；内部命令树仅在启动时构建一次。

```csharp
public void Register(
    string name,                                          //command literal, without the slash
    string description,                                   //one-line description, shown in the /ncmapi ledger
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attach arguments and the executor
```

您收到的 `build` 是内核的准将构建器； write 参数、子命令和权限谓词内核的方式：

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //permission predicate
            .Executes(context => { /* ... */ return 1; })));
```

带参数：

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //illustrative
                return 1;
            })));
```

要点：

- 直接调用`args.Dispatcher.Register(...)`也会安装命令，但不会进入账本，`/ncmapi`也不会显示。如果您希望列出它，请使用 `args.Register`。

- 命令默认无权限限制；如果需要，请自行添加 `.Requires(...)`。

- 执行时的行为完全取决于你； ModApi 不会拦截它。

### 3.2 查看注册命令

有一个内置的`/ncmapi`，需要权限级别2：

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

用法行是根据命令树的节点结构动态计算的：文字按名称编写，参数包含在尖括号中，本身可执行的中间节点也有自己的行。

***

## 4. 服务器外观

本章中的外墙都位于 `NetCraft.ModApi.Wrapper` 下；在 `using NetCraft.ModApi.Wrapper;` 之后它们可用。

外观是 `Nc*` 静态类，它将分散在内核中的功能收集到几个入口点。当主循环开始时，探针捕获内核实例；当 `NcServer.IsAvailable` 为 false 时，下面的所有内容都会抛出 - 仅在事件回调中使用它们，而不是在 `Init` 中使用它们。

|门面 |目的|
| --- | --- |
| `NcServer` |服务器实例、滴答率、命令、实体跟踪、玩家数据、游戏规则、广播、命令执行 |
| `NcPlayers` |在线玩家查询及操作（踢腿、传送、生命值、游戏模式、权限）|
| `NcWorld` |主世界区块读/写和破坏、天气、时间、边界、时钟、声音、关卡事件；需要 `NcLevel` 句柄来到达其他维度，坐标是普通的 `x y z` 整数 |
| `NcRegistries` |按名称查找内置注册表（块、物品、流体、效果、生物群系、粒子、实体、块实体）|
| `NcRecipes` |食谱查询（网格制作、石料切割、烹饪；通过 ID 获取食谱）|
| `NcLists` |列表和配置（白名单、操作、禁令、`server.properties`）|
| `NcStartup` |启动参数（内核无法识别的令牌和基于名称的订阅）|

`NcPlayer` 不是一个静态外观，而是一个 **对象句柄**： `NcPlayers.All` / `Find` 返回它，玩家事件中的 `Player` / `Attacker` 也是它。句柄是只读的，由探针构造； mods 无法获取内核的 `ServerPlayer` — “公共表面上没有内核类型”的第一个锚点。相同的内核播放器始终映射到相同的句柄，通过弱引用在内部缓存，并在播放器注销后自动失效。

`NcLevel` 的级别遵循相同的形状。 `NcWorld.Overworld` / `Nether` / `End` 和 `NcWorld.Get("minecraft:the_nether")` 返回它，`LevelTickArgs.Level` 也是 1。它携带维度 ID、时间、天气、构建高度、滴答计数和块力加载；块操作保留在 `NcWorld` 上并使用句柄加 `x y z`。 `BlockPos` 永远不会出现，因此 mod 的 dll 不包含对内核级类型的引用。

### 4.1 注册表

`NcRegistries` 提供整个表和按名称查找。整个表用于迭代和基于标签的查找；按名称查找用于获取单个元素：

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //block default state
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//iterate the whole table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

注册表在启动期间逐渐组装，并且 mods 在组装完成之前加载，因此不要缓存在 `Init` 中查找的任何内容 - 组装仍在进行中，并且缓存的值将是空引用或过时的值。目前 `BuiltInRegistries.BootStrap` 仍然是一个空的实现；每个注册表都由自己的引导程序单独填充，并且数据驱动的注册表（生物群落、配方等）在数据包加载连接之前只有很少的条目。

### 4.2 食谱

`NcRecipes` 由从数据包加载的配方表支持； `/reload` 替换整个表，因此在重新加载时不要保留 `RecipeHolder`。

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //compute the output for a crafting grid
    var recipes = NcRecipes.StonecuttingFor(stack);         //stonecutting recipes available for this input
    var smelting = NcRecipes.CookingFor("smelting", stack); //look up by cooking type
    var byId = NcRecipes.Find("minecraft:oak_planks");      //fetch a recipe by id
}
```

***

## 5. 内部结构

编写 mod 时不需要此部分，但在调试时可能会有所帮助。

### 5.1 探针

|班级 |表格 |责任|
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)` |马克×4 |所有“发生了什么事”信号都集中到一个方法中，由 `label` | 分派到匹配的事件。
| `Internal.CommandProbe.OnCommandsReady(object)` |呼叫站点 |替换对 `EffectCommand::Register` 的调用；恢复原始调用后，它会触发 `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)` |呼叫站点 ×6 |玩家事件，每个挂钩点一个方法；恢复原始调用后，它会发布事件 |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` |呼叫站点 ×3 |块事件，挂接在 `ServerChunkCache` 的三个回调属性的赋值点；在将控制权交还给内核之前，会分层包装委托 |
| `Internal.BlockProbe.OnXxx(...)` |呼叫站点 ×3 |阻止事件；断开并丢弃挂钩 `ServerBlockUpdates`，状态更改挂钩 `IBlockUpdateSink` 上的接口方法 |

`SignalProbe` 的签名仅采用 `string`，`CommandProbe`、`PlayerProbe` 和 `LevelProbe` 的参数被声明为 `object` — 这是故意的：在汇编期间 `Lead.Hook` 解析替换方法的签名，并且一旦内核类型出现在其中，解析它会提前拉出内核程序集，并且注入会错过其窗口。内核类型仅出现在方法体内部，此时代码已经在运行。

签名中唯一不能是 `object` 的是值类型参数和返回值：`object` 是堆栈上的引用，而 `float`/`bool` 是值，不匹配是无效的 IL。因此，`PlayerProbe.OnHurtPlayer` 保留 `float` 作为伤害量，`OnRemovePlayer` 和 `OnHurtPlayer` 保留 `bool` 返回值。

`BlockProbe`是该约束的扩展：块位置和状态是两种值类型`BlockPos`/`BlockState`，只能将其本身写入签名中。这两种类型来自`NetCraft.Primitives`和`NetCraft.Registry`，它们都不在注入列表中，因此在汇编期间解析它们不会拉出要提前重写的程序集。

### 5.2 挂钩点列表

ModApi的`ncmod.json`包含24条规则，与2.1中的表一一匹配。要更改挂钩点或添加规则，请编辑此文件；编辑后，重建（它是嵌入式资源）并将生成的dll放回到`mods/`中——后者已经由`NetCraft.ModApi.csproj`中的`DeployModToHosts`自动完成，缺少它则表现为规则根本不生效。

`CommandManager::Execute` 有两个共享一条规则的重载。 `Lead.Hook` 的 CallSite 通过“类型 + 方法名称”而不是参数列表来匹配调用站点，并且两个重载都采用两个参数，因此探针可以采用 `object` 作为第一个参数并按实际类型分派。

两个`PacketProcessor`规则是互补的：播放阶段数据包通过`ScheduleIfPossible`进入主线程队列，而握手和状态阶段通过`HandleNow`立即处理；任何给定的数据包仅通过其中一个。仅挂钩前者会错过握手和状态阶段 - 这恰好是最容易用脚本探测的阶段，因此在调试过程中很容易被误读为“规则未生效”。

三个块规则挂钩 `ServerChunkCache` 的三个回调属性的 **分配站点**，而不是读取站点。原因是这三个属性是单播的，并且在构造 `PersistentServerLevel` 时已经被内核本身占用（它们注入保存和块实体清理逻辑）；直接分配的 mod 将覆盖内核的副本 - 卸载不会持久，块实体不会被清理，并且根本没有错误。挂钩分配站点可以让内核的回调和探测器在那一刻链接起来；该分配仅发生一次，并且每个后续触发器都会添加一层委托转发。

`PlayerList::RespawnPlayer`是私有的，因此探针无法恢复原始呼叫；一个会经历反射（每次死亡调用一次，因此开销可以忽略不计）。这也为内核对齐打开了一扇门：如果将来为其添加`InternalsVisibleTo`，则可以将其交换为直接调用。

### 5.3 账本

`Internal.NcCommandRegistry` 记录通过`args.Register` 注册的命令。它只是一个账本，不参与命令执行；命令本身安装在内核调度程序上，因此即使分类帐出现问题，命令仍然有效。

***

## 6. 待补充

以下是位置已确认但尚未成为事件的挂钩点（该列表由存储库根目录中的 `__scan_mod_api.py` 生成）：

|方向 |候选挂钩点 |
| ----------------- | ---------------------------------------------------------------------------- |
|实体| `ClientLevel::AddEntity`，`Entity::Die` |
|世界 |级别加载和卸载，`ServerChunkCache` 块批处理 |
|地形生成 |每个 `ChunkStatus`、`WorldGenRegion::SetBlockState` | `ChunkGenerator::Generate` 阶段
|命令执行 | `CommandSourceStack::SendSuccess` / `SendFailure`（响应一半，有许多调用点）|
|网络| per-packet-type `ServerGamePacketListenerImpl::HandleXxx`（目前只有统一的入口点）|

已经完成的方向：级别刻度变为`ServerEvents.LevelTick`，保存的数据持久性变为`ServerEvents.SavedDataSaving`，命令执行变为`ServerEvents.CommandExecuted`，块变为`ServerEvents.BlockChanged`/`BlockBroken`/`ItemDropped`。

将块单元挂在 `SetBlock` 上不起作用：它有两个默认参数，`notifyNeighbors` 和 `strict`，因此编译后的调用站点采用 4 到 6 个参数，并且由于 CallSite 通过“类型 + 方法名称”进行匹配而不查看参数列表，因此一种替换方法无法处理所有三种堆栈形状。相反，它挂钩两个位置：状态同步挂钩 `IBlockUpdateSink::BlockChanged`（`ServerLevel.SetBlock` 中唯一的接口调用，覆盖客户端同步的每个更改），以及破坏和删除挂钩 `ServerBlockUpdates` 自己的方法。

实体单元很麻烦，因为 `Entity` 是在 `NetCraft.Registry` 中定义的，而 `NetCraft.Registry` 不在注入列表中，因此无法重写以它为目标的调用站点。 `ClientLevel::AddEntity` 在客户端程序集中并且是可行的；死亡事件首先需要解决`Registry`是否可以重写。

对于网络单元来说，`HandleChat`早已存在，`NetworkEvents.PacketReceived`提供了统一的入口点，因此per-`HandleXxx`钩子的价值要低得多；只有需要按数据包类型进行细粒度过滤的场景才值得添加。

级别加载和卸载在NC侧没有收敛点：`DedicatedServer::CreateLevel`是私有的，所以只能像`PlayerList::RespawnPlayer`一样通过反射来恢复原来的调用；卸载路径更加分散。为此，首先确定事件参数应该是什么。

其余的都需要对象引用（实体实例等），因此必须使用 `CallSite` 而不是 `Mark`；如果参数中出现内核值类型，则只能将其本身写入替换方法的签名中。

还审查了界面表面列表（`__modapi_api.txt`，由`__scan_mod_api.py --api`生成）：符合能力入口点的条目按领域收集到第4章外观中，其余未公开的分为三类——协议和数据包处理（`Network.Protocol.*`）、渲染和模型（`Client.Render.*`）以及地形生成和密度函数（`LevelGen.*`）。这些是内核内部结构；直接使用它们会将 mods 与实现细节联系起来，因此应该首先在内核中打开一个稳定的接口。

在服务器端，还有两个东西没有包装为外观：`ReloadableServerResources`实例挂在`DedicatedServer`上，并且由于ModApi不引用`NetCraft.Server`，包装它需要首先在内核基类上打开一个属性； `ChunkSender` 和 `ServerWorldBorderListener` 是内部流程，没有 mod 的用例。
