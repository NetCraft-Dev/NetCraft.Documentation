# NetCraft 模组开发指南


单个 API 的细节不在这里；见 [mod-api-zh.md](modapi-zh.md)。

---

## 1. 先看运行时结构

### 1.1 三个进程入口

NC 有三个入口点，三者的模组注入流水线完全相同：

| 入口 | 用途 |
| --- | --- |
| `NetCraft.Loader` | 一个 exe 兼顾两侧：`--server` 启动服务端；`--client` 或不带模式参数启动客户端 |
| `NetCraft.Server.Exe` | 独立的服务端可执行文件 |
| `NetCraft.Client.Exe` | 独立的客户端可执行文件 |

`Main` 本身只是一层薄壳，只注册回调并把工作交给下一个方法。以 `NetCraft.Server.Exe` 为例：

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // 注册内核程序集解析回调
    BootMods(args);                        // 运行模组引导
    return Launch(args);                   // 到这里才进入业务实现
}
```

这种分离不是风格问题。JIT 编译一个方法时会解析该方法体中出现的**所有类型**，且这发生在方法执行之前。如果 `Main` 直接调用 `ServerMain.Run(args)`，`NetCraft.Server.dll` 会在 `Main` 被 JIT 编译的瞬间就被拉起，早于模组引导运行，改写窗口就没了。所以 `BootMods` 和 `Launch` 都必须标注 `MethodImplOptions.NoInlining` —— 没有这个标注，JIT 会把它们内联回 `Main`，拆分的意义就没了。

`NetCraft.Loader` 结构相同，只是它的模式检测和模组引导都在 `Launch` 里，`Main` 只保留 `Initialize` 和 `Launch` 两步。

### 1.2 内核程序集放在 kernel/ 子目录

构建后的输出目录长这样：

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- 主库，内嵌所有更低层的子库
NetCraft.ModLoader.dll    <- 加载器本身
NetCraft.Server.Exe.dll   <- 入口程序集
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... 其余内核程序集
mods/
  your-mod.dll
```

为什么把它们挪进 `kernel/` 而不是留在根目录？

.NET 宿主把 `deps.json` 中注册的程序集视为 TPA（Trusted Platform Assemblies，受信任平台程序集）。对于 TPA 中的程序集，运行时**按路径**解析 —— 传给 `AssemblyLoadContext.LoadFromStream` 的字节会被直接忽略。也就是说，即便我们提前喂入改写后的字节，运行时仍会从磁盘读取未改写的那份。只有把内核程序集从 `deps.json` 中移除并把文件挪走，运行时才会在解析失败时回调 `AssemblyLoadContext.Resolving`，我们才有机会交出自改写后的字节。

留在根目录的三类不能挪：主库（它是内嵌宿主，必须最先启动）、加载器本身（引导代码住在里面）、入口程序集（apphost 从它启动）。

**代价**：入口程序集本身无法被注入。如果你的注入目标恰好住在 `NetCraft.Server.Exe.dll` 程序集里，那它不会生效。内核业务代码全在 `kernel/` 下，所以通常不是问题。

### 1.3 模组加载顺序

```
EmbeddedAssemblyLoader.Initialize()
  └─ 安装 Resolving 回调
BootMods → ModBootstrap.Run(当前侧)
  ├─ 静态扫描 mods/*.dll（MetadataReader 读取内嵌的 ncmod.json，不加载任何程序集）
  ├─ 过滤掉 environment 与当前侧不匹配的模组
  ├─ 汇总注入规则并把改写器交给主库
  ├─ 预加载目标程序集：读取字节 → 过改写器 → LoadFromStream
  └─ 调用每个模组入口的 Init()
Launch → ServerMain/ClientMain.Run(args)
  └─ 内核业务开始运行；探针已经在里面了
```

注意顺序：**先扫描声明，再加载改写后的字节，入口代码最后运行**。等模组的 `Init()` 执行时，内核程序集已经被替换完毕。

---

## 2. 与 Fabric 的关键差异

| 维度 | Fabric | NetCraft |
| --- | --- | --- |
| 语言 / 运行时 | Java / JVM | C# / .NET 10 (CoreCLR) |
| 模组载体 | 包含 `fabric.mod.json` 的 jar | 内嵌 `ncmod.json` 的 dll |
| 声明读取 | 读取 jar 内的一个文件 | `MetadataReader` 静态读取嵌入资源，不加载程序集 |
| 代码注入 | Mixin（源码中的注解；成员在类加载时混入目标类） | `Lead.Hook`（规则在清单或注解中声明；字节在程序集解析期间就地改写） |
| 注入粒度 | 方法体中的任意一行，包括局部变量和中间表达式值 | 十三种形式（调用点、字段读/写、构造、类型检查、装箱、局部变量、常量、整方法体替换、探针等），并支持前插或后插 |
| 加载模型 | Fabric Loader + Knot 类加载器 | 单一默认 ALC + `AssemblyLoadContext.Resolving` |
| 官方 API 范围 | Fabric API 有非常多模块 | NetCraft-ModApi 目前只有事件和命令扩展点 |

### 2.1 注入风格：注解或清单，二选一

Fabric 的 Mixin 是**在源码中**注解的：

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NC 两种风格都支持，但前提不同：注解形式依赖 `NetCraft.ModApi.Extension` 中的 `InjectAttribute`，所以不引用它的模组用不了注解；清单形式是写在 `ncmod.json` 里的纯数据，注入规则不需要任何引用。

**注解**，写在你自己的替换方法上：

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**清单**，写在 `ncmod.json` 的 `hooks` 中：

```json
{
  "target": "NetCraft.Game.Server.DedicatedServer",
  "method": "Tick",
  "type": "Mark",
  "label": "server_tick",
  "environment": "server",
  "replaceType": "MyMod.Probe",
  "replaceMethod": "OnTick"
}
```

组装时两条路线合并成一张规则表，**如果同一个注入点两边都写了，注解胜出**。合并之后二者无法区分；差别在于前提和易用性：

| | 注解 | 清单 |
| --- | --- | --- |
| 写在哪里 | 替换方法上 | `ncmod.json` 的 `hooks` 中 |
| 前提 | 必须引用 `NetCraft.ModApi` | 无，纯数据 |
| 类型名 | `typeof` / `nameof`，由编译器检查 | 手写字符串；拼错要到组装时才发现 |
| 能携带什么 | 只有注入规则 | 模组身份（id、entry、environment、显示信息）和注入规则 |

所以无论你用不用注解，`ncmod.json` 都必须写；它是模组身份的唯一来源。注解只是让规则写起来更不容易出错。清单没有依赖字段；依赖关系从程序集引用推断（见 6.1），无需声明。

反过来说，**不引用 `NetCraft.ModApi` 的模组只能用清单** —— 这影响的不只是注入规则：像 `ServerEvents` 这样的事件和 `Nc*` 门面也在 ModApi 中（在 `NetCraft.ModApi.Wrapper` 下，见 [3.3](#33-一个对照示例)），所以用不了注解的模组也用不了它们。

与 Fabric 的差异依然存在：

- **改变的方式和时机**：Mixin 有一个转换器，在**类加载**时把 mixin 类的成员**混入**目标类，所以加载进来的是一个合成的新类，原来的类不复存在；NC 则在**程序集进入内存之前**就地改写目标方法的指令，类还是同一个类，只是方法体变了。二者都在加载时改写，都不在编译期修改字节码 —— Mixin 的注解处理器只在构建时生成 refmap（混淆映射）并做校验，而 NC 不做混淆，根本没有这一层。
- Mixin 可以注入到**方法体中间的任意位置**；NC 可以在指定宿主方法内瞄准特定的调用点、字段访问、构造、局部变量读/写或常量，并能前插或后插（`InType`/`InMethod` 收窄范围，`Placement` 决定插入还是替换），但**无法触及任意行号**，也无法改变跳转目标或栈上的中间表达式值。
- Mixin 的目标用字符串方法名加描述符；NC 用「完整类型名 + 方法名」，所以同名重载全部匹配，要精确到某一个需要 `InType`/`InMethod`。

**注解由哪一层处理**：注解类型（`InjectAttribute`）由 `NetCraft.ModApi.Extension` 提供，由 `NetCraft.ModLoader` 解析 —— 扫描模组时它用 `MetadataReader` 静态读取 `CustomAttribute` 表，不加载程序集。**`Lead.Hook` 不认注解**；它只看合并后的规则表，而原生注入层只认托管侧编译出的描述字节，连 `ncmod.json` 都不读。

这决定了注解能表达什么：你能写什么完全取决于 `InjectAttribute` 有哪些字段。目前有七个 —— 目标类型、方法名、`HookType`、`Label`、`Environment`、`PatchMode`、`Ordinal` —— 而 [2.4](#24-收窄到一个位置宿主限定与放置) 中的 `InType`/`InMethod`/`Placement` 以及 [2.5](#25-方法体内部锚点局部变量与常量) 中的 `LocalIndex`/`ConstantValue` **无法写进注解**；要用 C# API，或者等清单跟上。清单这边也缺这些 —— 它超出注解能接受的只有 `ordinal`。

十三种注入形式见[本指南附录](#附录hooktype-一览)和 [mod-api-zh.md](modapi-zh.md)。

### 2.2 一条重要约束：探针类的签名不能带内核类型

NC 的改写发生在内核程序集加载**之前**。组装规则时，`Lead.Hook` 用反射找到你的替换方法并构建方法引用，这个过程会解析签名中的每一个参数类型和返回类型。

因此：**替换方法的签名只能使用 BCL 类型和 `object`**。一旦签名中出现 `NetCraft.*` 类型，解析它就会提前拉起内核程序集，注入直接失败。

当你需要内核对象时，把参数声明为 `object`，在方法体内转换：

```csharp
//程序集只看到 object；方法体在内核启动后才 JIT 编译
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite 是替换不是插入

`CallSite` 规则的替换方法**替换**原调用，所以原方法不会被执行。要保留原行为，你必须在替换方法里自己恢复：

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              // 恢复被替换的调用
    ServerEvents.CommandRegister.Publish(...);  // 再加上模组自己的逻辑
}
```

漏掉这一步，原功能就彻底消失了。

执行时的几个细节：

- **实例方法上的 `this` 也算一个参数**。目标方法是否为实例方法，决定了替换方法是否需要多一个前导参数。在 IL 中 `call` 和 `callvirt` 都算实例调用 —— 对于 `sealed` 类型上的非虚方法，编译器生成 `call`。
- **一个规则键覆盖该类名下的所有重载**；参数个数相同的重载共用一个替换方法。`Disconnect(string)` 和 `Disconnect(Component)` 就是这样共用的，替换方法按真实参数类型分发。
- **私有方法无法从外部调用**，所以替换方法不能恢复原调用。要么放弃这个注入点，要么用反射调用一次（低频时可接受）。
- **值类型参数和返回值不能声明为 `object`**，`object` 在栈上是引用而 `float`/`bool` 是值，不匹配就是非法 IL。这两个位置要保留它们的真实类型。
- **单播回调不能直接赋值**。有些内核回调属性（例如 `ServerChunkCache` 上的三个区块回调）是 `Action<T>` 而不是 `event`，且内核已经占用了它们。模组直接赋值会覆盖内核的那份，而且毫无报错。正确做法是注入该属性的 setter，在赋值那一刻把你的逻辑和内核回调串成一个包装委托。

### 2.4 收窄到一个位置：宿主限定与放置

指令级规则的默认范围是**整个程序集** —— 每一个调用目标方法、读写目标字段的地方都会匹配。要收窄到一个位置，用两个可选参数：

| 参数 | 作用 |
| --- | --- |
| `InType` / `InMethod` | 只在指定的宿主方法体内匹配锚点；两者都为空表示不限 |
| `Placement` | `Replace` 替换锚点（默认）；`Before` / `After` 保留锚点，在它前面或后面插入一个调用 |
| `Ordinal` | 同一个锚点在宿主方法中匹配到多处时，选择哪一处，从 0 开始。省略表示每一处都改 |

```csharp
//示例：只在 LevelChunk 读取方块状态时插桩；别处的 PalettedContainer::Get 不动
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

两种模式对回调签名有不同的要求：

- **替换模式**与被替换调用的参数对齐（实例调用包括 `this`）；回调是否恢复原调用由你决定。
- **插入模式**传递的是**宿主方法的参数**（包括 `this`），与 `MethodBody` 的约定一致。插入不会打扰锚点已经建好的栈；原调用照常运行，只是在它前面或后面多一个回调。

几条边界：

- `InType` 和 `InMethod` 相互独立；可以只指定一个。两者都为空等同于不限范围。
- 多条规则可以注入同一个锚点，各自限定到不同的宿主；**第一个匹配到的宿主胜出**。
- `Placement` 只适用于指令级形式（`CallSite`、`NewObj`、字段读/写、`TypeCheck`、`Box`、`FunctionPointer`，以及 [2.5](#25-方法体内部锚点局部变量与常量) 中的三种）；`MethodBody` 总是整段替换。
- `Ordinal` 数的是**匹配的顺序**，与那一处最终是否被修改无关；如果规则在宿主方法中出现的次数不够，规则就不会落地。与 Mixin 的 `@At(ordinal)` 思路相同。
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` 目前只在 C# API 上可用；`ncmod.json` 和 `[Inject]` 都不支持它们（清单接受 `ordinal`），所以基于清单的模组用不了前几个。

### 2.5 方法体内部锚点：局部变量与常量

前面几种锚定的是**被引用的实体**（方法、字段或构造函数），而 `LocalRead` / `LocalWrite` / `Constant` 锚定的是**宿主方法体内部的一个位置**，对应 Mixin 的 `@ModifyVariable` 和 `@ModifyConstant`。对于这三种，`OriginalType` / `OriginalMethod` 填的是**宿主方法**，不是被引用的实体。

| 形式 | 额外参数 | 选中的位置 |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` | 对该槽位的每一次读取（从 0 开始） |
| `LocalWrite` | `LocalIndex` | 对该槽位的每一次写入 |
| `Constant` | `ConstantValue` | 该常量的每一次加载，按装箱后的类型比较 |

```csharp
//示例：在 G 中写入槽位 0 之前插入一个回调
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//示例：把 G 中的常量 5 替换为 OnConst() 的返回值
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

在替换模式下，回调签名对齐的是**指令的栈效应**，而不是宿主参数：

| 锚点 | 栈效应 | 替换方法签名 |
| --- | --- | --- |
| `LocalRead` | 压入一个值 | 零参数，返回该值 |
| `LocalWrite` | 弹出一个值 | 一个参数 |
| `Constant` | 压入一个值 | 零参数，返回该值 |

`ConstantValue` 按装箱后的类型比较，所以 `5`（int）和 `5L`（long）是两个不同的锚点；要匹配 `ldc.i8` 必须传 `long`。

槽位是编译后的局部变量索引；同一份源码在不同编译器版本下可能改变它，所以跨版本移植时不要把它当作稳定标识。

### 2.6 运行时注入：修改已在运行的代码

前面讨论的注入都发生在**程序集加载之前** —— 先改写字节，再交给运行时。前提是目标程序集尚未加载。

`Lead.Hook` 还有另一条路线：用 CLR 的 Profiler 接口（ReJIT）修改**已经加载、或方法已经运行过**的代码。两者共用同一个 `HookRule`，[2.4](#24-收窄到一个位置宿主限定与放置) 和 [2.5](#25-方法体内部锚点局部变量与常量) 中的参数依然可用：

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//一次调用注入就完成了；无需重启也无需改文件
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| | 加载时改写 | 运行时注入 |
| --- | --- | --- |
| 清单 `patchMode` | `ILRewrite`（默认） | `RuntimeInject` |
| 时机 | 程序集进入内存之前 | 进程启动后的任意时刻 |
| 基础 | Mono.Cecil 字节改写 | CLR Profiler ReJIT |
| 前提 | 目标尚未加载 | 目标已在进程中 |
| 修改已 JIT 编译的代码 | 不可以 | 可以 |

**为什么规则共用**：这里仍然由 Cecil 做改写，但结果不写盘；而是编译成一份描述交给原生层，由它在运行时把新方法体提交给 CLR，剩余的版本管理留给 CLR。

**模组怎么做**：在 `ncmod.json` 的规则项里加一个 `patchMode` 字段；`[Inject]` 注解有同名参数。

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**组装做了什么**：这些规则不走加载时改写路径（`ModHooks.Rewrite` 只处理 `ILRewrite`）；组装时会单独记录一张运行时目标表。内核程序集预加载完毕、模组 `Init()` 之前，加载器取用它们各自**已加载**的实例，把改写后的方法体提交给 CLR。如果某个目标那一刻尚未加载，就跳过并给一条警告，也不会为它提前加载 —— [2.2](#22-一条重要约束探针类的签名不能带内核类型) 中「提前拉起内核会错过窗口」的约束在这里方向反过来，结论相同：不在场就做不了。

**挂上原生注入层**：ReJIT 开关只能在进程启动时通过环境变量设置（`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`）；启动后再设置没有效果。当加载器在启动早期检测到 `RuntimeInject` 规则而进程尚未挂上时，它会**带着这三个变量用同一命令行重启进程**（`NC_PROFILER_ATTACHED=1` 用于防止「已挂上却没生效」时的反复重启）。原生库 `lead_hook_native` 必须放在程序根目录，或用 `NC_PROFILER_PATH` 指向别处；两者都不在时，整批规则降级为一条警告，不阻断启动。

**不要和加载时改写混用在同一个目标上**：运行时注入提交的方法体来自**原始字节**，不包含加载时改写对同一方法所做的改动 —— 如果同一个方法同时被两类规则命中，加载时那份会被整体覆盖。组装无法判断两条规则是否命中同一个方法，只能按目标程序集做粗略判断并记录一条警告。

**注解不直接参与这条路径**：原生注入层不认 `InjectAttribute` 也不读 `ncmod.json` —— 它只认描述字节。注解和清单都是**组装期**的东西（见 [2.1](#21-注入风格注解或清单二选一)），由 `NetCraft.ModLoader` 解析成 `HookRule` 再交给 `RuntimeInjector`；模组侧无需分别处理。

**限制**（比加载时改写更窄）：不支持带异常处理表的方法，不能改变局部变量表，不支持泛型类型和方法，操作数只认方法引用（字段引用、字符串常量和类型 token 会抛 `NotSupportedException`）。

**性能**：注入只发生在注册时；之后方法就是普通的 JIT 代码，调用开销与不注入相同。挂上 profiler 有一次性的代价 —— 启用 ReJIT 需要同时禁用 ReadyToRun 镜像，实测进程启动慢约 80–110 ms；稳态计算没有差别。对 NC 这种启动本来就以秒计的服务器来说，可以忽略。

### 2.7 两个模组修改同一个类会像 Mixin 那样冲突吗？

先说 Mixin 那边为什么会冲突。Mixin 会把**成员混入目标类**并在类加载时应用：当多个 mixin 混入同一个类时，重复注入同一处、给同一个类加入同名成员等情况会抛 `MixinApplyError`，而默认的 fail-hard 会**直接把游戏干掉**；而且这个检测发生在类加载的一瞬间，此时游戏可能已经跑了一半。

NC 的模型不同，可能冲突的层面小得多：

| | Mixin | NetCraft |
| --- | --- | --- |
| 落地方式 | 把成员混入目标类 + 改写字节码 | 只改写 IL 指令；不合成类型、不增加成员 |
| 结构冲突（同名成员、继承冲突） | 有 | 无 |
| 规则何时校验 | 类加载时 | 组装时静态读取元数据 |
| 两条规则命中同一处 | 抛异常 | 先到先得；后者静默失败 |
| 一个模组失败 | 可能拖垮整个加载 | 只影响自己 |

**静态校验**：规则不是靠加载程序集并反射类型来构建的，而是靠读取 PE 元数据表。所以「目标类型不在任何已知程序集中」「注入形式拼错」之类的问题会在**启动早期**就被记录并跳过，不必等到某个类加载时才爆。

**故障隔离**：当某个模组的规则解析失败、替换类加载失败或入口 `Init()` 抛异常时，只有**这一个模组**被标记为失败（状态 `Error`，在 MODS 页面上显示为「加载失败」），其它模组照常加载。这里有一处措辞需要纠正：NC **没有运行时卸载** —— 模组只加载一次，`ModManager` 明确不提供动态加载/卸载。所谓「失败自动卸载」实际上是**加载时隔离**：失败的模组不会被初始化，但它也没有被「卸载」。

**动态注入**：[2.6](#26-运行时注入修改已在运行的代码) 的 ReJIT 路线上，多个模组争抢同一个方法时的语义与静态情形一致 —— 第一个注册的胜出，后来的请求虽然发出但抢不到（`GetReJITParameters` 按「模块 + 方法」认领，取第一个）。

我们在这条路线上踩过一次坑，值得记录：在早期实现中，`FindTypeRef` 的**解析范围参数传成了 `mdTokenNil`**，其语义是「只匹配没有解析范围的 TypeRef」 —— 我们的引用全都挂在 `AssemblyRef` 上，于是自己构建的那个永远找不到。表现是：第一次注入成功，但第二次注入时引用解析失败，构建不出新方法体，CLR 回退到原始 IL，**连同第一次注入一起丢了**（目标方法退回未注入的行为）。修复后不再复现，但限制仍在：**引用的元数据注入必须在目标模块加载后的窗口内完成；越晚越容易失败**。

**代价必须说清楚**：NC 不崩溃的行为，代价是冲突容易被漏掉 —— Mixin 至少会打断加载，而 NC 让后来的静默失败。为此，组装会做一次**同锚点冲突检查**：当同一个注入点被多个模组声明时，后组装的会记录进 `ModHooks.Warnings`，并在启动日志中作为警告报出（不阻断加载）：

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

检测键是「目标类型 + 方法 + 注入形式 + 补丁模式」。**宿主限定不参与区分** —— 清单和注解都写不了 `InType`/`InMethod`，所以来自模组的规则天然是整程序集范围，键相同就意味着冲突。通过 C# API 直接添加的规则绕过这项检查，因为那种情况下两条规则可能各自命中不同的宿主，本质上不冲突。

**运行时注入也走这项检查**：它进入同一个组装入口，检测键里的补丁模式让它与加载时改写区分开；当两类规则落在同一个目标程序集上时，另有一条覆盖提示（见 [6.4](#64-两个模组注入同一个目标)）。

### 2.8 Mixin：给目标类型增加成员

前面几节修改的都是既有代码中的指令，创造不出新东西。要**给目标类型增加字段、方法或接口**，用 mixin。

与 Mixin 语法的关系如下：

| Mixin | NC |
| --- | --- |
| mixin 类上的 `@Mixin(X.class)` | 源类上的 `[Mixin(typeof(X))]` |
| mixin 类的成员被混入目标类 | 源类的字段和方法被移入目标类型 |
| `@Unique` 添加一个私有字段 | 在源类里写一个普通字段；以同样方式移过去 |
| `@Shadow` 引用目标类的既有成员 | 不需要；直接写 `X` 的成员再注入即可 |
| `@Implements` / `implements` | `Interfaces` |

也可以写在清单里：

```json
{
  "target": "NetCraft.Game.World.Entity.SomeEntity",
  "source": "MyMod.SomeEntityMixin",
  "interfaces": ["MyMod.ITagged"]
}
```

```csharp
[Mixin(typeof(SomeEntity), Interfaces = new[] { typeof(ITagged) })]
public class SomeEntityMixin
{
    //移过去之后，这就是目标类型上的一个实例字段
    public int MyCounter = 5;

    //一个混入的方法；它读写随之移过来的字段
    public int Bump() => MyCounter + 1;

    //接口要求的实现；移过去之后目标类型就实现了 ITagged
    public string Describe() => $"tagged:{MyCounter}";
}
```

**是移动，不是复制**。这些成员从源类中移除，只留下一个空壳 —— 和 Mixin 的 mixin 类一样，**模组代码不应再使用那个类**（`new SomeEntityMixin()` 或调用它的方法会找不到成员）。

几条落地规则：

- **只适用于加载时改写**。目标类型必须在尚未进入内存的内核程序集或模组程序集中。运行时注入只提交方法体，无法改变类型布局，所以这种形式在它上面不存在。
- **构造函数初始化器会一起带过去**。写在源类字段初始化器里的值会并入目标类型的每一个实例构造函数；静态字段初始化器并入静态构造函数（目标没有就创建）。源类构造函数里的基类链接调用会被剥离，所以基类构造函数不会被运行两次。
- **带接口的规则会把移入的公共实例方法标记为虚方法**。接口派发只认虚表，没有这个标记 CLR 会判定接口未实现并直接加载失败。所以混入接口时，别指望那些方法保持非虚。
- **同名成员会被跳过**。当目标类型已有同名字段或方法时，那一项不移动，其余照常进行。两个模组混入同一个目标类型时，两者都会落地，只有后者的同名部分被跳过 —— 比 [2.7](#27-两个模组修改同一个类会像-mixin-那样冲突吗) 中 hook 的「后者整体失败」行为更温和。
- **嵌套类型不会被移动**，源类中的嵌套类型或泛型方法目前也不在这条路径的覆盖范围内。

源类型必须在**模组自己的程序集**中，所以清单和注解都不写程序集名。

### 2.9 两条路线：包装层与扩展点

`NetCraft.ModApi` 的公共表面分为两个命名空间，对应两种用法：

| 命名空间 | 内容 | 你能得到什么 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`，`NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup`，以及 `NcPlayer` / `NcLevel` 等对象句柄 | 包装类型；公共表面上没有内核类型 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 注解 | 规则绑定到内核的类名和方法名 |

两者是**平行**路线，不是一层叠在另一层上：

- **要稳定就用 `Wrapper`**。门面替你处理内核繁琐的调用顺序（写一个方块同时牵动世界、玩家列表和同步链就是一个例子），事件 args 也全是包装类型。代价是门面没有暴露的能力你就用不了。
- **要完整就用 `Extension`**。注入规则直接修改内核的类和方法，但你写的目标名就是内核的名字，所以内核一变规则就得跟着变。

两者都可以引用。`Wrapper` 这条线正在向「公共表面上没有内核类型」收敛；玩家和世界部分已完成 —— 玩家事件里的 `Player` / `Attacker` 以及 `NcPlayers` 的入/出参数都是 `NcPlayer` 句柄，`NcWorld.Overworld` / `Nether` / `End` / `Get` 和 `LevelTickArgs.Level` 都是 `NcLevel` 句柄，方块坐标是普通的 `x y z` 整数；实体和其余值类型（`BlockPos` / `BlockState` / `Vec3`）还没包装。

还有一条边界要指出：**包装层不隔离注入**。你写的 `hooks` 规则或 `[Inject]` 注解仍然绑定到内核的类名和方法名，内核一变照样断。

---

## 3. 从 Fabric 迁移

### 3.1 概念对照

| Fabric | NetCraft |
| --- | --- |
| `fabric.mod.json` | 内嵌在 dll 中的 `ncmod.json` |
| `ModInitializer.onInitialize()` | 入口类的 `public Task Init()` |
| `@Inject` / `@Redirect` | `[Inject]` 注解，或 `hooks` 中的 `Mark` / `Probe` / `CallSite` 等规则 |
| `@ModifyVariable` | `LocalRead` / `LocalWrite`，见 [2.5](#25-方法体内部锚点局部变量与常量)；无法写成注解 |
| `@ModifyConstant` | `Constant`，见 [2.5](#25-方法体内部锚点局部变量与常量)；无法写成注解 |
| `@Accessor` | 暂无对应物（`private` 成员无需放宽可见性；直接写规则即可） |
| `Registry.register(...)` | 内核注册表（`BuiltInRegistries`） |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` | 暂无对应物（`ModManager` 不对模组开放） |
| `@Mixin` / `@Unique` / `@Implements` | `[Mixin]` 注解或清单的 `mixins`，见 [2.8](#28-mixin给目标类型增加成员) |

### 3.2 不能照搬的部分

- **Mixin 的注解体系**：NC 有两个注解 `[Inject]` 和 `[Mixin]`，它们都只是**声明风格**，等价于 `ncmod.json` 中的 `hooks` / `mixins` 并在组装时合并（由加载器解析，不是 `Lead.Hook`，见 [2.1](#21-注入风格注解或清单二选一)）。指令改写默认在加载时落地，可按 [2.6](#26-运行时注入修改已在运行的代码) 改为运行时提交；增加成员和接口走 [2.8](#28-mixin给目标类型增加成员) 的 mixin。目标定位没有 `@At` 式的字符串语法，但 `CallSite`/`FieldRead`/`LocalWrite`/`Constant` 等形式，配合 `Ordinal`、`InType`/`InMethod` 和 `Placement`，能覆盖 `HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE` 以及 `shift` 的用法；缺的是 `JUMP`。
- **AccessWidener**：没有。在 NC 中可见性不是 IL 改写的障碍；`private` 方法照样能注入（改写器工作在字节层面）。
- **Yarn / Mojang 映射**：不需要。NC 是直接从原版翻译过来的 C# 源码，类型和方法名与原版对应，只是命名风格遵循 C#。
- **绝大多数 Fabric API 模块**：只有 `NetCraft-ModApi` 覆盖到的能力可用；其余的自己写注入规则，或者等 API 跟上。

### 3.3 一个对照示例

Fabric：服务器启动时打一行日志并注册一条命令。

```java
public class MyMod implements ModInitializer {
    @Override
    public void onInitialize() {
        ServerLifecycleEvents.SERVER_STARTED.register(server -> {
            System.out.println("server started");
        });
        CommandRegistrationCallback.EVENT.register((dispatcher, registry, env) -> {
            dispatcher.register(CommandManager.literal("mymod")
                .executes(ctx -> { ctx.getSource().sendSuccess(() -> Text.literal("hi"), false); return 1; }));
        });
    }
}
```

NC：

```csharp
public sealed class MyModEntry
{
    public Task Init()
    {
        ServerEvents.Started.Subscribe(_ => Log.Info("server started"));
        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    return 1;
                })));
        return Task.CompletedTask;
    }
}
```

清单（`ncmod.json`，作为嵌入资源）：

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

注意：这里即使 `hooks` 为空也能工作 —— 像 `ServerEvents.Started` 这样的事件由 `NetCraft-ModApi` 自己的探针提供，你的模组只需订阅（`ServerEvents` 在 `NetCraft.ModApi.Wrapper` 下，见 [2.9](#29-两条路线包装层与扩展点)）。只有当你想要注入内核中 ModApi 还没有提供事件的地方时，才需要自己写 hook 规则。

---

## 4. 一个 ncm 的基本要求

ncm 指 NetCraft 模组。一个 ncm 是一个内嵌 `ncmod.json` 的 .NET 类库 dll，放在 `mods/` 目录下。

最简单的起步方式是模板：

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

如果本地包还没发布，用 `dotnet new install <nupkg path>`，或者在仓库里运行 `dotnet pack` 再安装输出。

`-e` 接受 `both`（默认）/ `server` / `client`，决定清单的 `environment` 以及入口类中生成哪一侧的订阅代码。模板自带 NC 引用程序集，所以不需要项目引用，`ncmod.json` 的 id / entry 会根据项目名自动填入。

从 4.1 开始，下面讲手写项目必须满足什么。

### 4.1 项目文件

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private 必须关掉，否则 NC 程序集会被 4.6 中的 EmbedDependencies 嵌入到模组 dll 中 -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName` 必须是 `ncmod.json`；扫描器只认这个名字。

模板走的是另一条路：`libs/` 下的程序集引用（`Reference Include="libs\*.dll" Private="false"`）。没有 NC 源码时照那个来。两种方式共同的一点是**NC 自己的 dll 绝不能进入输出目录** —— [4.6](#46-第三方依赖) 的 `EmbedDependencies` 会把输出目录里的第三方 dll 嵌入模组，如果 NC 程序集也被嵌进去就会出现两套类型标识。

### 4.2 ncmod.json 字段

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "name": "My Mod",
  "description": "one-line description",
  "authors": ["someone"],
  "license": "MIT",
  "contact": { "homepage": "https://...", "sources": "https://..." },
  "icon": "icon.png",
  "environment": "both",
  "entry": "MyMod.ModEntry",
  "hooks": []
}
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `id` | 是 | 模组标识符，用于依赖和查找。空字符串会被扫描器跳过 |
| `version` | 推荐 | 版本号；被其他模组依赖时，版本约束按它判断，见下 |
| `name` | 否 | 显示名；模组页面显示的就是它，缺省回退到 `id` |
| `description` | 否 | 一行描述 |
| `authors` / `contributors` | 否 | 鸣谢，字符串数组 |
| `license` | 否 | 许可证标识符 |
| `contact` | 否 | 外部链接；可以取 `homepage` / `sources` / `issues` |
| `icon` | 否 | 图标的嵌入资源名，见 4.7 |
| `environment` | 否 | `both` / `client` / `server`，默认 `both`。与当前侧不匹配时整个模组不加载 |
| `entry` | 是 | 入口类的全名；该类必须有 `public Task Init()` |
| `depends` | 否 | 它依赖的其他模组及所需版本，见下 |
| `hooks` | 否 | 注入规则列表；空数组表示只订阅 ModApi 已有的事件 |
| `mixins` | 否 | mixin 规则列表；把本模组某个类的成员移入目标类型，见 [2.8](#28-mixin给目标类型增加成员) |

`depends` 声明它依赖的其他模组及所需版本；键是模组 id，值是版本约束：

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**没有写进 `depends` 的依赖不带版本约束**。模组间的依赖已经从编译期引用自动推断（见 [6.1](#61-模组依赖--注入被依赖的模组)），`depends` 只是在上面追加版本约束。被依赖的模组不在当前侧时跳过检查 —— 那种情况留给程序集解析去报告。

| 约束语法 | 含义 |
| --- | --- |
| `*` | 任意版本，等同于省略该项 |
| `1.2.3` | 段前缀；`1.2` 匹配 `1.2` 和 `1.2.9`，但不匹配 `1.3` |
| `^1.2.3` | 主版本相同且不低于基线；主版本为 0 时改用次版本判断，所以 `0.1` 和 `0.2` 视为不兼容 |
| `>=1.2.3` | 不低于基线 |

版本号只取每一段的前导数字，所以 `26.2-netcraft` 以 `26.2` 参与。版本不匹配时，**只跳过声明的那个模组**，其余照常加载；启动日志会写「依赖 X 需要版本 …，实际版本 …」。

判断依据是被依赖模组清单里的 `version` 字段。所以**打算被依赖的模组必须正确设置 `version`** —— 空版本号满足不了任何具体约束。

### 4.3 hook 规则字段

| 字段 | 说明 |
| --- | --- |
| `target` | 目标类型的全名；必须在某个内核程序集或 `mods/` 下的某个模组程序集中 |
| `method` | 目标方法名；同名重载全部匹配 |
| `type` | 注入形式，见附录 |
| `patchMode` | 落地方式，`ILRewrite`（默认）或 `RuntimeInject`，见 [2.6](#26-运行时注入修改已在运行的代码) |
| `ordinal` | 同一个锚点在宿主方法中匹配到多处时选哪一处，从 0 开始，见 [2.4](#24-收窄到一个位置宿主限定与放置) |
| `replaceType` | 包含替换方法的类的全名 |
| `replaceMethod` | 替换方法名 |
| `label` | 探针标签，只被 `Mark` 和 `Probe` 使用 |
| `environment` | 规则适用的侧，默认 `both`；针对服务端类型的规则在客户端上运行根本没有目标，会被 `environment` 过滤掉 |

同一条规则也可以写成替换方法上的注解；对应关系是：

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

| 清单字段 | 注解形式 |
| --- | --- |
| `target` | 第一个构造函数参数，写成 `typeof(...)` |
| `method` | 第二个构造函数参数，最好用 `nameof(...)` |
| `type` | 命名参数 `HookType`，默认 `CallSite` |
| `patchMode` | 命名参数 `PatchMode`，默认 `ILRewrite` |
| `ordinal` | 命名参数 `Ordinal` |
| `label` | 命名参数 `Label` |
| `environment` | 命名参数 `Environment`，默认 `both` |
| `replaceType` | 不写；自动取它所注解的类 |
| `replaceMethod` | 不写；自动取它所注解的方法 |

注解来自 `NetCraft.ModApi.Extension` 中的 `InjectAttribute`，所以使用注解的模组必须引用它。注解和清单同时存在时会合并，如果同一个注入点在两边都声明，**注解胜出**。

Mixin 规则使用另一组字段：`target`（目标类型全名）、`source`（源类型全名，必须在模组自己的程序集中）、`interfaces`（可选，接口全名数组）；语义见 [2.8](#28-mixin给目标类型增加成员)。源类上的 `[Mixin(typeof(target))]` 等价于清单项。

### 4.4 能注入什么、不能注入什么

能注入：`kernel/` 下的内核程序集，以及 `mods/` 下的其他模组。

不能注入：

- 主库 `NetCraft.dll`
- 加载器 `NetCraft.ModLoader.dll`
- 入口程序集（`NetCraft.Server.Exe.dll` 等）

每个 hook 的 `target` 必须能在内核或某个模组程序集中找到，否则组装会报「注入目标不在任何已知程序集中」。注意**命名空间不等于程序集** —— `NetCraft.Game.Server.DedicatedServer` 实际上住在 `NetCraft.Server.dll` 里；加载器用元数据表构建的索引来查找，所以把全名写对即可。

模组注入模组遵循同一套体系：目标模组在**它自己被加载的那一刻**被改写，与 `mods/` 中的顺序无关。循环规则（A 注入 B 且 B 注入 A）会在加载时报「循环加载」；这类规则在 IL 改写层面无解，去掉其中一条即可。

注意被注入的模组**不能是你自己的模组** —— 自我注入同样算循环。

### 4.5 部署

把编译好的 dll 复制到输出目录的 `mods/`（顶层，不递归子目录）并重启进程。

**这一步已由构建自动化**：`NetCraft.ModApi.csproj` 中的 `DeployModToHosts` 会在构建后把 dll 复制到每个宿主项目输出目录的 `mods/`。宿主列表是 `ModHostProjects` 属性；创建自己的宿主项目时，把它的名字加进去即可。如果没有任何规则生效且毫无报错，先检查宿主项目 `mods/` 里的 dll 是不是旧的。

启动日志会打印：

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

排查顺序：

- 没有任何规则生效：先检查 `mods/` 里的 dll 是不是旧的（最常见）。
- 看不到 `Rewrote and preloaded assembly`：目标程序集在引导之前就已加载，规则写得太晚。
- 看到 `injection targets` 但目标不对：通常是 `target` 写错了；按 [4.4](#44-能注入什么不能注入什么) 检查类型属于哪个程序集。
- `environment` 设为 `client` 的规则在服务端会被静默跳过（模组级别不匹配意味着整个模组不加载）；这是预期行为，不是故障。

### 4.6 第三方依赖

对应 Fabric 的 Jar-in-Jar。

从模板创建的项目**不需要操心这个**：通过 `dotnet add package` 添加的库会在构建时自动嵌入模组 dll，加载器解析不到程序集时会翻查模组内嵌的 `.dll` 资源。

```
dotnet add package Newtonsoft.Json
```

就这样；`ncmod.json` 不需要改动。

手写项目必须从模板 csproj 复制 `EmbedDependencies` 目标，或者自己嵌入：

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

依赖不写进清单；解析只看模组内嵌的 `.dll` 资源，只需要资源名与程序集名匹配。

三点说明：

- **按程序集名匹配**。加载器比较请求的程序集名：去掉 `.dll` 后，资源名要么等于程序集名，要么以 `.` + 程序集名结尾。所以 `MyLib.dll` 和默认的 `MyProject.deps.MyLib.dll` 都可以。
- **只认模组自己的嵌入资源**。没有嵌入的库解析不到，也不会去别处找。
- **解析顺序是内核优先**。`EmbeddedAssemblyLoader` 先在内核程序集和内嵌子库中找，找不到才回退到模组，所以模组不应嵌入与内核同名的程序集。

模板的目标用 `WithMetadataValue` 过滤 `.dll` 而不是写 `Condition`，因为模板引擎在模板生成时就会求值 `.csproj` 中的 `Condition`，那时 `%(...)` 还没有值，整行会被删掉。

### 4.7 图标与显示信息

名称、描述、作者、链接和图标都写在 `ncmod.json` 中，对应原版 `fabric.mod.json` 里的 `name` / `description` / `authors` / `contact` / `icon`。

**图标是嵌入资源**，不是外部文件，使用与嵌入依赖相同的资源命名约定：

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

模板已经自带一个 `icon.png` 且两处都配置好了；替换那张图即可。推荐 64×64 或 128×128 的 PNG。

图标查找顺序是：清单 `icon` 指向的 → 名为 `icon.png` 的嵌入资源 → 两者都不存在时，UI 的默认图像（一个灰色问号）。

这些信息在服务端 GUI 的 **MODS** 页面上可见。左边是模组列表（小图标 + 显示名 + 版本 + 状态）；右边是选中模组的描述、鸣谢、链接、依赖、初始化时间和注入规则。依赖列中的模组名可点击，直接跳到那一项。

加载失败或被跳过的模组也在列表里，状态列会标出 —— 诊断「我的模组为什么没生效」时，先看这一列。

---

## 5. 已知限制

- **探针签名只能使用 BCL 类型和 `object`**，原因见 2.2；值类型参数和返回值是例外，必须保留真实类型。
- **`CallSite` 是替换**，替换方法必须自己恢复原调用，见 2.3。私有方法无法恢复，需要反射。
- **`mods/` 下的 dll 由 `DeployModToHosts` 自动部署**；创建自己的宿主项目时，记得把它的名字加进 `ModHostProjects`。
- **入口程序集不能被注入**：如果你的目标和 hook 落在入口程序集里，那它不生效。
- **主库和加载器不能被注入**；这是防止模组改变加载过程本身的设计约束。
- **`environment` 不匹配意味着整个模组不加载**，而不是「部分规则失败」。
- **运行时注入需要原生库**：使用 `RuntimeInject` 的规则要求进程启动时挂上 `lead_hook_native`，加载器为此会重启自己；如果找不到这个库或重启失败，这批规则降级为警告，不阻断启动。能力边界和代价见 [2.6](#26-运行时注入修改已在运行的代码)。
- **注解的字段比 C# API 少**：`InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` 无法写进 `[Inject]`（`PatchMode` 和 `Ordinal` 支持），见 [2.1](#21-注入风格注解或清单二选一)。
- **Mixin 只适用于加载时改写**，源类中的成员是移动而非复制；源类中的嵌套类型和泛型方法不在覆盖范围内，带接口混入的方法会被标记为虚方法。见 [2.8](#28-mixin给目标类型增加成员)。
- 测试和调试时，如果用到语言表或模型资源，需要一个 `assets` 目录（从原版 jar 中提取），否则相关功能会退化为翻译键或占位贴图。

---

## 6. 已知行为

本节是对观察到的行为的记录，不是规范。

### 6.1 模组依赖 + 注入被依赖的模组

**场景**：b 依赖 a 且同时注入 a。

**结论**：可行，且不构成循环。

这条链有三步：

1. 规则表纯粹靠读取 PE 元数据构建，不加载任何程序集。「b 注入 a」这条规则既不要求 a 在场，也不要求 b 在场。
2. `PreloadReplacers` 在 `ModManager` 之前加载所有替换类（包括 b），此时 a 还未加载。`LoadFromStream` 只读元数据、不 JIT 方法体，所以此刻 b 对 a 的引用是惰性的，加载不会失败。
3. `ModManager` 随后按 `AssemblyRef` 做拓扑排序，a 在 b 之前。改写 a 时，替换类 b 已经在 `Default` 中，所以直接取到类型，并向 a 的元数据添加一个指向 b 的 `AssemblyRef`。**a 根本不需要知道 b 的存在。**

**硬性要求**：在 csproj 中引用被注入的模组时，必须写 `Private="false"`。默认情况下它会把 `a.dll` 复制到输出目录，随后被 `EmbedDependencies` 作为嵌入依赖嵌进 `b.dll`，运行时 `ModLibs` 解析 `a` 会取到第二份副本，导致两者之间的类型标识检查失败。

**验证用例**：两个模板项目 `NetCraft.Test1`（a）和 `NetCraft.Test2`（b）；a 提供 `Test1Api.Greet` 并在自己的 `ModEntry.Server` 中调用它，b 把该调用点替换为 `Test2Probe.OnGreet`，同时也在 b 的 `ModEntry.Server` 中调用一次 `Greet` 作为对照。实际运行日志：

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← 注入生效
Test2 internal greeting: Test1 original greeting test2    ← 对照组：规则只改写目标程序集，所以 b 自己内部的调用点不受影响
```

依赖只从 `AssemblyRef` 推导顺序，不带版本约束。要约束版本，在清单的 `depends` 中声明，见 [4.2](#42-ncmodjson-字段)。

### 6.2 注解注入

**场景**：替换方法上有 `[Inject(typeof(X), nameof(X.M))]`，`ncmod.json` 中没有规则。

**结论**：可行，且不需要清单声明。注解和清单同源，在组装时合并，同一个注入点注解胜出。

**为什么不需要声明**：注解就是元数据里的 `CustomAttribute` 表，而加载器扫描模组时已经在读同样的元数据（清单、嵌入资源和 AssemblyRef —— 三项），多读一张表不会引入新的加载或时机约束。唯一的前提是模组引用了 `NetCraft.ModApi`（注解的宿主）。

**前提是静态读取**：读取注解必须走 `MetadataReader`，**不能用 `Assembly.Load` + `GetCustomAttributes`** —— 后者为了读规则就会拉起模组程序集，改写窗口当场就没了。

**`typeof` 不构成类型引用**：`typeof(X)` 编译进参数的是类型的序列化名（`全名, 程序集, Version=…`），它解析出来只是一个字符串，不要求 `X` 在场。所以在注解里写 `typeof(被注入的模组)` **不**违反 2.2 中「替换类不得引用被注入模组的类型」的约束 —— 名字出现在元数据里和运行时解析出类型是两回事。

**验证用例**：`NetCraft.Test2` 针对 `NetCraft.Test1` 的两个方法；`Greet` 走注解，`Farewell` 走清单。两条规则都装上，两个调用点都被替换：

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← 注解
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← 清单
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← 对照组：规则只改写目标程序集
Test2 internal farewell: Test1 original farewell test2     ← 对照组
```

这个用例还顺带验证了注解的命名参数（`Environment = "server"`）也能被解析。

### 6.3 挂载 profiler 的性能代价

**场景**：同一个纯计算程序（一亿次取模累加迭代），一次不挂 profiler，一次挂上（三个 `CORECLR_ENABLE_PROFILING` 环境变量）。各跑五次。

**结论**：稳态没有差别；代价全在启动。

| | 进程总时间（5 次，ms） | 程序内计算时间 |
| --- | --- | --- |
| 不挂 | 306 / 260 / 278 / 253 / 300 | 225 ms |
| 挂上 | 406 / 370 / 340 / 325 / 354 | 193 ms |

总时间长了约 80–110 ms。来源是 `COR_PRF_DISABLE_ALL_NGEN_IMAGES` —— 启用 ReJIT 需要同时禁用 ReadyToRun 镜像，所以框架代码只能走 JIT；程序内部的计时循环一旦 JIT 编译就完全相同，看不出差别（挂了 profiler 那次反而略快，属于噪声）。

**顺带修掉的一处浪费**：最初的实现订阅了 `COR_PRF_MONITOR_JIT_COMPILATION`，一个每个方法编译后都会触发的原生回调，而我们从未用到。移除后事件掩码从 `0x80040024` 变为 `0x80040004`；上表是移除之后的数据。

**验证用例**：`__hookverify/BenchProbe`。

### 6.4 两个模组注入同一个目标

**场景**：两个模组各自声明一条命中同一目标方法的规则（`MethodBody` 形式，替换方法不同）。

**结论**：不报错、不崩溃；**先组装的规则胜出，后者静默失败**。

两条规则都进入规则表 —— 不做跨模组去重。改写时取 `OriginalType::OriginalMethod` 同键列表中的**第一条**；宿主限定同理，`InType`/`InMethod` 是「第一个匹配到的胜出」。谁在前取决于组装顺序，而组装顺序来自 mods 目录的枚举顺序；**没有优先级字段，也无法通过声明依赖来控制**（依赖只影响 `Init()` 的顺序，不影响注入规则的组装）。

**运行时注入也是先到先得**：后到的注册请求会正常发出，但 `GetReJITParameters` 按「模块 + 方法」认领，总是匹配第一个请求，所以后注册的不会落地。实测第二次注入后，目标方法的行为仍是第一次的结果。

**一条坏规则不影响其他**：目标类型不在任何已知程序集中、注入形式拼错之类的问题会在组装时记录进 `ModHooks.Errors` 并跳过该项，其他模组的规则照常组装。

**冲突会被记录**：当组装检测到同一个注入点被多个模组声明时，后组装的会写进 `ModHooks.Warnings`，并在启动日志中以 `Mod injection conflict ...` 输出，指明是哪两个模组碰撞以及哪一个不会生效。它是警告不是错误，不影响加载，规则本身也留在表里（只是不可达）。

**运行时注入也走这项检查**：冲突检测键包含 `patchMode`，所以一个模组写 `ILRewrite`、另一个写 `RuntimeInject` 不算碰撞（两条独立路径，各干各的）；只有同模式的两个才判为冲突并警告。它实际的落地同样先到先得 —— `GetReJITParameters` 按「模块 + 方法」认领，匹配第一个请求，所以后来的虽发出但不落地。

**唯一会硬崩的点**：两个模组都以 `RuntimePatch` 模式补丁同一个方法时，第二个会撞上 `RuntimeHookEngine` 的重复注册检查并抛 `InvalidOperationException`，这条路径没有捕获，启动直接失败。`RuntimePatch` 正在被淘汰（见 [2.6](#26-运行时注入修改已在运行的代码)）；新规则不要用它。

**验证用例**：`NetCraft.Test` 的 `modinjection` 模块，条目 `same anchor first mod wins quietly` 和 `one bad rule does not sink the rest`。

### 6.5 尚未验证

- **运行时注入在真实服务器上可用**：`RuntimeInject` 模式已在 `__hookverify/RuntimeProbe` 中端到端验证（注册后目标方法的行为被换成替换方法的），模组程序集侧也有路由和降级的测试覆盖；但当前没有任何构建步骤把 `lead_hook_native` 放到 NC 的运行目录，所以在真实服务器上跑通这条链需要先把库放进程序根目录（或用 `NC_PROFILER_PATH` 指向它）。这一步尚未完成。
- **替换类引用被注入模组的类型**：按推理，改写 a 的那一刻它会解析 a，而 a 卡在加载完成之前（它还没进 `Default`，`ModLibs` 和内核解析回调也都不认模组程序集），所以 `PrepareMod` 预期会抛异常并记录进 `result.Errors`。尚未实际运行。注意 6.2 只证明了**注解里的 `typeof`** 不算引用；**类型出现在方法签名里**是另一回事。

---

## 附录：HookType 一览

| 形式 | 作用 | 对替换方法签名的要求 |
| --- | --- | --- |
| `CallSite` | 用你的方法替换对目标方法的调用点 | 参数个数与被调用方法一致（实例调用 +1） |
| `MethodBody` | 替换整个目标方法体 | 与被替换方法一致 |
| `NewObj` | 替换 `new X(...)` | 参数个数与构造函数一致 |
| `FieldRead` | 对字段读取插桩 | 按读取的类型 |
| `FieldWrite` | 对字段写入插桩 | 按写入的类型 |
| `TypeCheck` | 对 `isinst` / `castclass` 插桩 | 按被检查的类型 |
| `Box` | 对装箱/拆箱插桩 | 按元素类型 |
| `FunctionPointer` | 对函数指针加载插桩 | 按委托类型 |
| `LocalRead` | 对局部变量读取插桩 | 零参数，返回该变量的值 |
| `LocalWrite` | 对局部变量写入插桩 | 一个参数，接收被写入的值 |
| `Constant` | 对常量加载插桩 | 零参数，返回该常量的值 |
| `Probe` | 保留原方法体，对入口和每个出口插桩；配合 `LabelArgumentIndex`，可以把一个参数折进标签 | `Begin()` 返回 long，`End(string, long)` |
| `Mark` | 只在方法入口上报一次，不计时 | `void method(string label)` |

`Probe` 和 `Mark` 只传标签文本（`Probe` 还可以带上一个参数的 `ToString()`）；它们拿不到对象引用。要拿到实际参数，用 `CallSite`。

`InType`/`InMethod`、`Placement` 和 `Ordinal`（见 [2.4](#24-收窄到一个位置宿主限定与放置)）只对指令级形式有意义：上表中除 `MethodBody`、`Probe` 和 `Mark` 之外的十个条目可以选择替换或前插/后插，并可用 `Ordinal` 挑选单次出现；`MethodBody` 总是整段替换，`Probe`/`Mark` 忽略这些参数。

对于 `LocalRead` / `LocalWrite` / `Constant` 这三种，宿主方法填在 `target` 里而不是被引用的实体，并额外需要 `localIndex` 或 `constantValue`；见 [2.5](#25-方法体内部锚点局部变量与常量)。
