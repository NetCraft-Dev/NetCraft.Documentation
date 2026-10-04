# NetCraft 改装指南


此处不提供各个 API 的详细信息；参见 [mod-api.md](mod-api.md)。

---

## 1.首先运行时结构

### 1.1 三个进程入口点

NC 具有三个入口点，并且所有三个入口点的 mod 注入管道都是相同的：

|进入|目的|
| --- | --- |
| `NetCraft.Loader` |双方各一个 exe： `--server` 启动服务器； `--client` 或无模式标志启动客户端 |
| `NetCraft.Server.Exe` |独立服务器可执行文件|
| `NetCraft.Client.Exe` |独立客户端可执行文件|

`Main` 本身是一个薄壳，仅注册回调并将工作交给下一个方法。取`NetCraft.Server.Exe`：

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // register the kernel assembly resolution callback
    BootMods(args);                        // run the mod bootstrap
    return Launch(args);                   // only now enter the business implementation
}
```

这种分离不是风格问题。当 JIT 编译一个方法时，它会解析该方法主体中出现的**所有类型**，并且这发生在该方法执行之前。如果`Main`直接调用`ServerMain.Run(args)`，`NetCraft.Server.dll`将在`Main`被JIT编译的瞬间被拉起，在mod引导程序运行之前，重写窗口将消失。因此 `BootMods` 和 `Launch` 都必须标记为 `MethodImplOptions.NoInlining` — 如果没有标记，JIT 将它们内联回 `Main`，从而阻止拆分。

`NetCraft.Loader`具有相同的结构，只是其模式检测和mod bootstrap都在`Launch`中，而`Main`仅保留`Initialize`和`Launch`两个步骤。

### 1.2 kernel/子目录中的内核程序集

构建后的输出目录如下所示：

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- main library, embeds all lower-level sub-libraries
NetCraft.ModLoader.dll    <- the loader itself
NetCraft.Server.Exe.dll   <- entry assembly
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... remaining kernel assemblies
mods/
  your-mod.dll
```

为什么将它们移动到 `kernel/` 而不是将它们留在根目录中？

.NET 主机将 `deps.json` 中注册的程序集视为 TPA（可信平台程序集）。对于 TPA 中的程序集，运行时通过路径解析 - 传递到 `AssemblyLoadContext.LoadFromStream` 的字节将被忽略。也就是说，即使我们提前输入重写的字节，运行时仍然从磁盘读取未重写的副本。只有从 `deps.json` 中删除内核程序集并将文件移走，运行时才会在解析失败时回调到 `AssemblyLoadContext.Resolving`，从而使我们有机会移交重写的字节。

留在根目录中的三种类型不能移动：主库（它是嵌入主机，必须首先启动）、加载器本身（引导代码位于其中）和入口程序集（apphost 从它启动）。

**成本**：入口组件本身无法注入。如果你的钩子目标恰好位于 `NetCraft.Server.Exe.dll` 程序集中，那么它是无效的。内核业务代码都在`kernel/`下，所以一般情况下这不是问题。

### 1.3 Mod加载顺序

```
EmbeddedAssemblyLoader.Initialize()
  └─ install the Resolving callback
BootMods → ModBootstrap.Run(current side)
  ├─ statically scan mods/*.dll (MetadataReader reads embedded ncmod.json, no assembly is loaded)
  ├─ filter out mods whose environment does not match the current side
  ├─ assemble injection rules and hand the rewriter to the main library
  ├─ preload target assemblies: read bytes → run through rewriter → LoadFromStream
  └─ call each mod entry's Init()
Launch → ServerMain/ClientMain.Run(args)
  └─ kernel business starts running; the probes are already inside
```

请注意顺序：**扫描声明，加载重写的字节，最后运行入口代码**。当 mod 的 `Init()` 执行时，内核程序集已经被替换。

---

## 2. 与 Fabric 的主要区别

|尺寸|面料|网络争霸|
| --- | --- | --- |
|语言/运行时 | Java/JVM | C# / .NET 10 (CoreCLR) |
|模组载体 |包含 `fabric.mod.json` | 的 jar dll 嵌入 `ncmod.json` |
|宣言阅读|读取 jar 内的文件 | `MetadataReader` 静态读取嵌入资源而不加载程序集 |
|代码注入| Mixin（源中的注释；成员在类加载时混合到目标类中）| `Lead.Hook`（在清单或注释中声明的规则；在程序集解析期间就地重写字节）|
|注射粒度|方法体中的任何行，包括局部变量和中间表达式值 |十三种形式（调用站点、字段读/写、构造函数、类型检查、装箱、局部变量、常量、整个方法体替换、探针等），带有 insert-before 或 insert-after |
|加载模型| Fabric Loader + Knot 类加载器 |单个默认 ALC + `AssemblyLoadContext.Resolving` |
|官方API范围| Fabric API 有很多模块 | NetCraft-ModApi 目前只有事件和命令扩展点 |

### 2.1 注入样式：注释或清单，任选其一

Fabric的Mixin在源码中注释为：

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NC支持这两种风格，但它们的先决条件不同：注释形式依赖于`NetCraft.ModApi.Extension`中的`InjectAttribute`，因此不引用它的mod无法使用注释；清单形式是用`ncmod.json`编写的纯数据，不需要参考注入规则。

**注解**，放在自己的替换方法上：

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**清单**，写在`ncmod.json`中的`hooks`中：

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

在组装过程中，两条路线合并成一个规则表，并且**如果两者中都写入相同的注入点，则注释获胜**。合并后，它们无法区分；区别在于先决条件和人体工程学：

| |注释|清单 |
| --- | --- | --- |
|写在哪里|关于更换方法|在 `hooks` 在 `ncmod.json` |
|先决条件|必须引用 `NetCraft.ModApi` |无，纯数据|
|类型名称 | `typeof` / `nameof`，由编译器检查 |手写字符串；仅在组装过程中才发现拼写错误 |
|它可以携带什么 |仅注入规则 | mod身份（id、条目、环境、显示信息）和注入规则|

所以无论你是否使用注解，都必须写`ncmod.json`；它是 mod 身份的唯一来源。注释只会使规则编写起来更不容易出错。清单没有依赖字段；依赖关系是从程序集引用推断出来的（参见 6.1），不需要声明。

相反，**不引用 `NetCraft.ModApi` 的 mod 只能使用清单** - 这影响的不仅仅是注入规则：像 `ServerEvents` 和 `Nc*` 外观的事件也在 ModApi 中（在 `NetCraft.ModApi.Wrapper` 下，参见 [3.3](#33-a-side-by-side-example)），因此不能使用注释的 mod 也无法使用它们。

与 Fabric 的区别仍然是：

- **改变如何以及何时发生**：Mixin 有一个变压器 **在**类加载**时将 mixin 类的成员混合到**目标类中，因此加载的是合成的新类，原始类不再存在； NC 在**程序集进入内存之前**就地重写了目标方法的指令，因此该类仍然是同一个类，只是其方法体发生了变化。两者都在加载时重写，都不在编译时修改字节码——Mixin 的注释处理器仅生成 refmap（混淆映射）并在构建时执行验证，而 NC 没有混淆并且根本没有这样的层。
- Mixin可以注入**方法体中间的任意位置**； NC 可以针对指定宿主方法内的特定调用点、字段访问、构造、局部变量读/写或常量，并且可以在其之前或之后插入（`InType`/`InMethod` 缩小范围，`Placement` 决定插入或替换），但它**无法到达任意行号**，并且无法更改堆栈上的跳转目标或中间表达式值。
- Mixin 目标使用字符串方法名称加描述符； NC 使用“完整类型名+方法名”，因此同名重载所有匹配，并且精确到单个需要 `InType`/`InMethod`。

**哪个层处理注释**：注释类型（`InjectAttribute`）由`NetCraft.ModApi.Extension`提供，并由`NetCraft.ModLoader`解析——扫描mod时，它使用`MetadataReader`静态读取`CustomAttribute`表，而不加载程序集。 **`Lead.Hook` 无法识别注释**；它只看到合并的规则表，并且本机注入层仅识别在托管端编译的描述字节，甚至不读取`ncmod.json`。

这决定了注释可以表达什么：你能写什么完全取决于`InjectAttribute`有哪些字段。目前有七个 — 目标类型、方法名称、`HookType`、`Label`、`Environment`、`PatchMode`、`Ordinal` — 以及 [2.4](#24-narrowing-to-one-site-host-scoping-and-placement) 中的 `InType`/`InMethod`/`Placement` 和 [2.5](#25-in-method-body-anchors-local-variables-and-constants) 中的 `LocalIndex`/`ConstantValue` **无法写入在注释中**；使用 C# API 或等待清单赶上。清单方面也缺少这些——除了注释之外它唯一接受的是`ordinal`。

对于十三种注入形式，请参阅[改装指南附录](#appendix-hooktype-overview)和[mod-api.md](mod-api.md)。

### 2.2 一个重要的约束：探测类的签名中不能携带内核类型

NC 重写发生在内核程序集加载之前。在组装规则时，`Lead.Hook`使用反射来查找替换方法并构建方法引用，并且此过程解析签名中的每个参数类型和返回类型。

因此： **替换方法的签名只能使用 BCL 类型和 `object`**。一旦 `NetCraft.*` 类型出现在签名中，解析它就会提前提取内核程序集，并且注入会彻底失败。

当您需要内核对象时，将参数声明为 `object` 并在方法体内进行强制转换：

```csharp
//assembly only sees object; the method body is JIT-compiled after the kernel starts
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite是替换，不是插入

`CallSite` 规则的替换方法**替换**原始调用，因此原始方法不会执行。要保留原始行为，您必须在替换方法中自行恢复：

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              // restore the replaced call
    ServerEvents.CommandRegister.Publish(...);  // then add the mod's own logic
}
```

错过这一步，原来的功能就会完全消失。

有关执行此操作的一些细节：

- **实例方法上的`this`也算作参数**。目标方法是否是实例方法决定了替换方法是否需要额外的前导参数。在 IL 中，`call` 和 `callvirt` 都算作实例调用 — 对于 `sealed` 类型上的非虚拟方法，编译器会发出 `call`。
- **一个规则键涵盖该类名下的所有重载**；具有相同参数数量的重载共享一种替换方法。 `Disconnect(string)`和`Disconnect(Component)`共享这种方式，替换方法按实参类型调度。
- **私有方法无法从外部调用**，因此替换方法无法恢复原始调用。要么放弃这个钩子点，要么通过反射调用一次（低频可以接受）。
- **值类型参数和返回值不能声明为`object`**，`object`是堆栈上的引用，而`float`/`bool`是值，不匹配是无效IL。将这两个位置保留为其真实类型。
- **单播回调不能直接分配**。一些内核回调属性（例如，`ServerChunkCache`上的三个块回调）是`Action<T>`而不是`event`，并且内核已经占用它们。 mod 分配直接覆盖内核的副本，完全没有错误。正确的方法是挂钩属性的 setter，并在赋值时将逻辑和内核回调链接到一个包装器委托中。

### 2.4 缩小到一个站点：主机范围和放置

指令级规则的默认范围是**整个程序集** - 调用目标方法或读取/写入目标字段匹配的每个位置。要缩小到一个站点，请使用两个可选参数：

|参数|效果|
| --- | --- |
| `InType` / `InMethod` |仅在指定的宿主方法体内匹配锚点；均为空表示不受限制 |
| `Placement` | `Replace` 替换锚点（默认）； `Before` / `After` 保留锚点并在其之前或之后插入一个呼叫 |
| `Ordinal` |当同一个锚点匹配宿主方法中的多个位置时，选择哪一个，从 0 开始。省略表示每个地方都修改了 |

```csharp
//example: instrument only when LevelChunk reads block state; PalettedContainer::Get elsewhere is untouched
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

两种模式对回调签名有不同的要求：

- **替换模式**与替换调用的参数一致（包括实例调用的 `this`）；回调是否恢复原始呼叫取决于您。
- **插入模式**传递**宿主方法的参数**（包括`this`），与`MethodBody`约定一致。插入不会干扰锚点已经建立的堆栈；原始调用照常运行，只是在其之前或之后有一个额外的回调。

一些界限：

- `InType`和`InMethod`是独立的；您可以只指定一个。两者都为空相当于没有作用域。
- 多个规则可能会挂钩同一个锚点，每个规则的范围都不同主机； **第一场主场比赛获胜**。
- `Placement`仅适用于指令级形式（`CallSite`、`NewObj`、字段读/写、`TypeCheck`、`Box`、`FunctionPointer`和[2.5](#25-in-method-body-anchors-local-variables-and-constants)中的三种）； `MethodBody` 总是取代整个事情。
- `Ordinal` 计算**匹配顺序**，无论该站点最终是否被修改；如果该规则在宿主方法中出现的次数不够多，则该规则不会落地。与 Mixin 的 `@At(ordinal)` 相同的想法。
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` 目前仅在 C# API 上可用； `ncmod.json` 和 `[Inject]` 都不支持它们（清单接受 `ordinal`），因此基于清单的 mod 不能使用前几个。

### 2.5 方法体内锚点：局部变量和常量

前面的类型锚定到**引用的实体**（方法、字段或构造函数），而 `LocalRead` / `LocalWrite` / `Constant` 锚定到 **宿主方法体内的位置**，对应于 Mixin 的 `@ModifyVariable` 和 `@ModifyConstant`。对于这三个，`OriginalType` / `OriginalMethod` 命名**主机方法**，而不是引用的实体。

|表格 |额外参数 |所选职位 |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` |每次读取该槽（从 0 开始）|
| `LocalWrite` | `LocalIndex` |每次写入该插槽|
| `Constant` | `ConstantValue` |该常数的每个负载，按盒装类型进行比较 |

```csharp
//example: insert one callback before the write to slot 0 in G
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//example: replace the constant 5 in G with the return value of OnConst()
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

在替换模式下，回调签名与**指令的堆栈效果**一致，而不是主机参数：

|锚|堆栈效应|替换方法签名 |
| --- | --- | --- |
| `LocalRead` |推一个值|零参数，返回该值 |
| `LocalWrite` |弹出一个值 |一个参数|
| `Constant` |推一个值|零参数，返回该值 |

`ConstantValue`是通过盒装类型进行比较的，所以`5`（int）和`5L`（long）是两个不同的anchor；要匹配 `ldc.i8`，您必须通过 `long`。

一个slot是编译后的局部变量索引；同一源在不同的编译器版本下可能会改变它，因此跨版本移植时不要将其视为稳定标识符。

### 2.6 运行时注入：修改已经运行的代码

到目前为止讨论的注入都发生在**程序集加载之前** - 首先重写字节，然后交给运行时。前提是目标程序集还没有加载。

`Lead.Hook` 还有另一种途径：使用 CLR 的 Profiler 接口 (ReJIT) 修改**已加载的代码，或其方法已运行**的代码。两者共享相同的 `HookRule`，并且 [2.4](#24-narrowing-to-one-site-host-scoping-and-placement) 和 [2.5](#25-in-method-body-anchors-local-variables-and-constants) 中的参数仍然可用：

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//one call and injection is done; no restart and no file changes
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| |加载时重写 |运行时注入 |
| --- | --- | --- |
|清单 `patchMode` | `ILRewrite`（默认）| `RuntimeInject` |
|时间 |在程序集进入内存之前|进程开始后的任何时间 |
|基金会| Mono.Cecil 字节重写 | CLR 分析器 ReJIT |
|先决条件|目标尚未加载|目标已经在过程中 |
|修改已经 JIT 编译的代码 |不可能|可能 |

**为什么规则是共享的**：Cecil 仍然在这里进行重写，但结果没有写入磁盘；相反，它被编译成传递给本机层的描述，本机层在运行时将新方法体提交给 CLR，而剩余的版本管理则留给 CLR。

**mod 是如何做的**：在 `ncmod.json` 中的规则项中添加 `patchMode` 条目； `[Inject]` 注释有一个同名参数。

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**程序集的作用**：这些规则不经过加载时重写路径（`ModHooks.Rewrite` 仅适用于 `ILRewrite`）；在汇编时，会记录一个单独的运行时目标表。在预加载内核程序集之后和 mods 的 `Init()` 之前，加载器会获取每个**已加载的**实例，并将重写的方法体提交给 CLR。如果目标当时没有加载，则会跳过警告，并且不会提前加载它 - 来自 [2.2](#22-an-important-constraint-probe-classes-must-not-carry-kernel-types-in-signatures) 的约束“提前拉起内核会错过窗口”在这里方向相反，得出相同的结论：如果它不存在，则无法完成。

**挂钩本机注入层**：ReJIT 开关只能在进程启动时通过环境变量设置（`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`）；启动后设置它们没有效果。当加载程序在启动早期检测到 `RuntimeInject` 规则并且进程尚未 hook 时，它会使用携带这三个变量的相同命令行重新启动进程（`NC_PROFILER_ATTACHED=1` 防止在“hooked 但无效”时重复重新启动）。本机库`lead_hook_native`必须放置在程序根目录中，或通过`NC_PROFILER_PATH`指向其他地方；如果两者都不存在，则整批规则将降级为单个警告，并且不会阻止启动。

**不要将其与同一目标上的加载时重写混合**：通过运行时注入提交的方法体源自**原始字节**，并且不包括对同一方法进行的加载时重写更改 - 如果同一方法被两种规则类型命中，则加载时版本将被完全覆盖。 Assembly无法判断两条规则是否命中了同一个方法，只能通过目标Assembly进行粗略判断并记录警告。

**注释不直接参与此路径**：本机注入层无法识别 `InjectAttribute` 或读取 `ncmod.json` — 它仅识别描述字节。注释和清单都是**组装时**的东西（参见[2.1]（#21-injection-styles-annotations-or-manifest-pick-one）），由`NetCraft.ModLoader`解析为`HookRule`，然后交给`RuntimeInjector`； mod 方不需要以不同的方式处理它们。

**限制**（比加载时重写更窄）：不支持带有异常处理表的方法，无法更改局部变量表，不支持泛型类型和方法，操作数仅识别方法引用（字段引用、字符串常量和类型标记抛出 `NotSupportedException`）。

**性能**：注入仅在注册时发生；之后的方法是普通的 JIT 代码，其调用开销与没有注入时相同。连接分析器需要一次性成本——启用 ReJIT 需要同时禁用 ReadyToRun 映像，并且测得的进程启动速度大约慢 80–110 毫秒；稳态计算显示没有差异。对于像 NC 这样的启动已经以秒为单位的服务器来说，这个可以忽略不计。

### 2.7 两个修改同一个类的mod会像Mixin一样冲突吗？

首先，为什么 Mixin 方面会发生冲突。 Mixin **将成员混合到目标类中**并在类加载时应用它们：当多个 mixins 混合到同一个类中时，诸如在同一位置重复注入或将同名成员添加到同一类的情况会抛出 `MixinApplyError`，默认的失败硬 **会直接杀死游戏**；此外，这种检测发生在类加载的瞬间，此时游戏可能已经运行了一半。

NC的模型不同，可能发生冲突的面要小得多：

| |米辛 |网络争霸|
| --- | --- | --- |
|登陆方式|将成员混合到目标类中+重写字节码 |仅重写IL指令；没有类型合成，没有添加成员 |
|结构冲突（同名成员、继承冲突）|是的 |无 |
|规则何时验证 |班级负荷 |在组装时，静态读取元数据|
|两条规则击中同一个地方 |抛出|先到先得；后者默默地失败了|
|一个模组失败 |可能会拖累整个负载|仅影响其自身 |

**静态验证**：规则不是通过加载程序集和反射类型来构建的，而是通过读取PE元数据表来构建的。因此，诸如“目标类型不在任何已知程序集中”或“注入形式拼写错误”之类的问题都会被记录并在**启动初期**跳过，而无需等待类在爆炸之前加载。

**故障隔离**：当某个 mod 的规则解析失败、替换类加载失败或条目 `Init()` 抛出时，只有 **该 mod** 被标记为失败（状态 `Error`，在 MODS 页面上显示为“加载失败”），而其他 mod 照常加载。这里需要纠正一个措辞：NC **没有运行时卸载** - mods 加载一次，并且 `ModManager` 明确不提供动态加载/卸载。所谓“失败时自动卸载”实际上是**加载时隔离**：失败的mod不会被初始化，但也不会被“卸载”。

**动态注入**：在来自[2.6](#26-runtime-injection-modifying-already-running-code)的ReJIT路线上，竞争同一方法的多个mod的语义与静态情况匹配——第一个注册的获胜，随后的请求被发送但无法声明它（`GetReJITParameters`通过“模块+方法”声明，取第一个）。

我们在这条路线上遇到过一次陷阱，值得记录一下：在早期的实现中，`FindTypeRef` 的 **解析范围参数作为 `mdTokenNil`** 传递，其语义是“仅匹配没有解析范围的 TypeRef”——我们的引用都挂在 `AssemblyRef` 上，因此我们构建的引用从未找到。表现为：第一次注入成功，但第二次注入引用解析失败，无法构建新的方法体，CLR回落到原始IL，**第一次注入随之丢失**（目标方法恢复到其未注入的行为）。修复后，它不再重现，但限制仍然存在：**引用的元数据注入必须在目标模块加载后立即在窗口内完成；越晚，失败的可能性就越大**。

**必须明确说明成本**：NC 的不崩溃行为是以容易错过冲突为代价的——Mixin 至少会中断加载，而 NC 让后者默默地失败。为了解决这个问题，程序集会执行**相同锚点冲突检查**：当多个 mod 声明相同的注入点时，稍后组装的注入点会记录在 `ModHooks.Warnings` 中，并在启动日志中报告为警告（不会阻止加载）：

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

检测关键是“目标类型+方法+注入形式+补丁模式”。 **主机作用域不区分** - 清单和注释都不能写 `InType`/`InMethod`，因此来自 m​​ods 的规则在作用域中自然是整个程序集，并且相同的键意味着冲突。通过 C# API 直接添加的规则会绕过此检查，因为在这种情况下，两个规则可能各自访问不同的主机，并且本质上不冲突。

**运行时注入也经过此检查**：它进入相同的程序集入口点，并且检测键中的补丁模式使其与加载时重写分开；当两种类型落在同一目标程序集上时，会出现单独的覆盖通知（请参阅[6.4]（#64-two-mods-injecting-the-same-target））。

### 2.8 Mixins：向目标类型添加成员

前面的部分都修改了现有代码中的指令，并且不能创建任何新内容。要将字段、方法或接口添加到目标类型，请使用 mixin。

与Mixin的语法关系如下：

|米辛 |数控|
| --- | --- |
| `@Mixin(X.class)` 关于 mixin 类 | `[Mixin(typeof(X))]` 关于源类 |
| mixin 类的成员被混合到目标类中 |源类的字段和方法被移动到目标类型 |
| `@Unique` 添加私有字段 |在源类中写入一个普通字段；它以同样的方式移动|
| `@Shadow` 引用目标类的现有成员 |不需要；直接写`X`的成员然后hook它们 |
| `@Implements` / `implements` | `Interfaces` |

也可以在manifest中这样写：

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
    //after being moved over, this is an instance field on the target type
    public int MyCounter = 5;

    //a mixed-in method; it reads/writes the field moved along with it
    public int Bump() => MyCounter + 1;

    //the implementation the interface requires; after being moved over the target type implements ITagged
    public string Describe() => $"tagged:{MyCounter}";
}
```

**移动，而不是复制**。这些成员从源类中删除，只留下一个空壳——就像 Mixin 的 mixin 类一样，**mod 代码不应再使用该类**（`new SomeEntityMixin()` 或调用其方法将无法找到成员）。

一些登陆规则：

- **仅适用于加载时重写**。目标类型必须位于尚未进入内存的内核程序集或 mod 程序集中。运行时注入仅提交方法体，无法更改类型布局，因此该表单不能存在于其上。
- **构造函数初始化器出现**。源类字段初始值设定项中写入的值将合并到目标类型的每个实例构造函数中；静态字段初始值设定项被合并到静态构造函数中（如果目标没有则创建）。源类构造函数内的基类链接调用被剥离，因此基构造函数不会运行两次。
- **带有接口的规则将移动的公共实例方法标记为虚拟**。接口调度仅识别 vtable，如果没有标记，CLR 将确定接口未实现并且无法完全加载。因此，不要期望这些方法在接口中混合时保持非虚拟状态。
- **同名成员被跳过**。当目标类型已经具有同名的字段或方法时，该项目不会移动，其余项目照常进行。当两个 mod 混合到相同的目标类型中时，两者都会着陆，并且只会跳过后者的名称冲突部分 - 比 [2.7](#27-do-two-mods-modifying-the-same-class-conflict-like-mixin) 中的 hooks 的“后者完全失败”行为更温和。
- **嵌套类型不会移动**，并且源类中的嵌套类型或泛型方法当前也在该路径的覆盖范围之外。

源类型必须位于 **mod 自己的程序集**中，因此清单和注释都不会写入程序集名称。

### 2.9 两条路线：包装层和扩展点

`NetCraft.ModApi`的公共表面被分为两个命名空间，对应于两种用法：

|命名空间|内容 |你得到什么 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`、`NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup` 以及对象句柄，如 `NcPlayer` / `NcLevel` |包装类型；公共表面上没有内核类型 |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 注释 |规则绑定到内核类和方法名称|

这两条路线是**平行**路线，而不是一条叠加在另一条之上：

- **为了稳定性，请使用 `Wrapper`**。外观为您处理内核繁琐的调用顺序（在触摸关卡时编写单个块、玩家列表和同步链就是一个例子），并且事件参数都是包装器类型。代价是您无法使用外观未公开的功能。
- **为了完整起见，请使用 `Extension`**。注入规则直接修改内核类和方法，但你编写的目标名称是内核的名称，因此当内核更改时，规则必须随之更改。

两者都可以参考。 `Wrapper` 线正在被巩固为“公共表面上没有内核类型”；玩家和关卡部分已完成 - 玩家事件中的 `Player` / `Attacker` 和 `NcPlayers` 的输入/输出参数是 `NcPlayer` 句柄，`NcWorld.Overworld` / `Nether` / `End` / `Get` 和 `LevelTickArgs.Level` 是 `NcLevel` 句柄，块坐标是普通的 `x y z` 整数；实体和其余值类型（`BlockPos` / `BlockState` / `Vec3`）尚未包装。

需要指出的另一个边界是：**包装层不屏蔽注入**。您编写的 `hooks` 规则或 `[Inject]` 注释仍然绑定到内核类和方法名称，并且在内核更改时同样会中断。

---

## 3. 从 Fabric 迁移

### 3.1 概念图

|面料|网络争霸|
| --- | --- |
| `fabric.mod.json` |嵌入在 dll 中的 `ncmod.json` |
| `ModInitializer.onInitialize()` |入口类的 `public Task Init()` |
| `@Inject` / `@Redirect` | `[Inject]` 注释，或 `hooks` | 中的 `Mark` / `Probe` / `CallSite` 等规则
| `@ModifyVariable` | `LocalRead` / `LocalWrite`，参见[2.5](#25-in-method-body-anchors-local-variables-and-constants)；不可写为注释 |
| `@ModifyConstant` | `Constant`，见[2.5](#25-in-method-body-anchors-local-variables-and-constants)；不可写为注释 |
| `@Accessor` |尚无等效项（`private` 成员不需要扩大可见性；只需编写一条规则）|
| `Registry.register(...)` |内核注册表 (`BuiltInRegistries`) |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` |尚无等效项（`ModManager` 尚未向 mods 开放）|
| `@Mixin` / `@Unique` / `@Implements` | `[Mixin]` 注释或清单的 `mixins`，请参阅 [2.8](#28-mixins-adding-members-to-a-target-type) |

### 3.2 哪些内容不能延续

- **Mixin 的注释系统**：NC 有两个注释，`[Inject]` 和 `[Mixin]`，这两个注释都只是 **声明样式**，相当于 `ncmod.json` 中的 `hooks` / `mixins`，并在汇编时合并（它们由加载器解析，而不是 `Lead.Hook`，参见 [2.1](#21-injection-styles-annotations-or-manifest-pick-one)）。指令重写默认在加载时落地，并且可以根据[2.6]（#26-runtime-injection-modifying-already-running-code）更改为运行时提交；添加成员和接口通过 [2.8](#28-mixins-adding-members-to-a-target-type) 的 mixin。定位没有 `@At` 样式的字符串语法，但 `CallSite`/`FieldRead`/`LocalWrite`/`Constant` 等形式，以及 `Ordinal`、`InType`/`InMethod` 和 `Placement` 可以涵盖以下用法`HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE`和`shift`；缺少的是`JUMP`。
- **AccessWidener**：无。可见性并不是 NC 中 IL 重写的障碍； `private` 方法可以以相同的方式挂钩（重写器在字节级别工作）。
- **Yarn / Mojang 映射**：不需要。 NC 是直接从 vanilla 翻译过来的 C# 源代码，类型和方法名称与 vanilla 相对应，只是命名风格遵循 C#。
- **绝大多数 Fabric API 模块**：仅 `NetCraft-ModApi` 涵盖的功能可用；其余的，请编写您自己的注入规则或等待 API 赶上。

### 3.3 并排示例

Fabric：服务器启动时记录一行并注册命令。

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

数控：

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

清单（`ncmod.json`，作为嵌入式资源）：

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

注意：即使是空的`hooks`也可以在这里工作 - 像`ServerEvents.Started`这样的事件是由`NetCraft-ModApi`自己的探针提供的，并且你的mod只需要订阅（`ServerEvents`在`NetCraft.ModApi.Wrapper`下，参见[2.9]（#29-two-routes-wrapper-layer-and-extension-points））。当你想钩住内核中 ModApi 尚未提供事件的地方时，你只需要编写自己的钩子规则。

---

## 4. ncm 的基本要求

ncm 是指 NetCraft mod。 ncm 是嵌入 `ncmod.json` 的 .NET 类库 dll，放置在 `mods/` 目录中。

最简单的开始方法是模板：

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

如果本地包尚未发布，请使用 `dotnet new install <nupkg path>`，或在存储库中运行 `dotnet pack`，然后安装输出。

`-e`采用`both`（默认）/`server`/`client`，确定清单的`environment`以及在条目类中生成哪一方的订阅代码。该模板附带 NC 参考组件，因此不需要项目参考，并且 `ncmod.json` 的 id/条目从项目名称中填充。

从4.1开始，以下内容涵盖了手写项目必须满足的内容。

### 4.1 项目文件

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private must be turned off, otherwise the NC assemblies get embedded into the mod dll by EmbedDependencies in 4.6 -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName` 必须是 `ncmod.json`；扫描仪仅识别该名称。

该模板采用不同的路线：`libs/` (`Reference Include="libs\*.dll" Private="false"`) 下的装配引用。当您没有 NC 源时，请遵循此操作。两种方法的共同点是 **NC 自己的 dll 绝不能进入输出目录** - [4.6](#46-third-party-dependencies) 中的 `EmbedDependencies` 将输出目录中的第三方 dll 嵌入到 mod 中，如果 NC 的程序集也被嵌入，则会有两组类型标识。

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

|领域 |必填 |笔记|
| --- | --- | --- |
| `id` |是的 | mod 标识符，由依赖项和查找使用。扫描仪跳过空字符串 |
| `version` |推荐|版本号；被其他mod依赖时，由其判断版本限制，见下文|
| `name` |没有 |显示名称；这是 mod 页面显示的内容，默认返回 `id` |
| `description` |没有 |一行描述 |
| `authors` / `contributors` |没有 |学分，字符串数组 |
| `license` |没有 |许可证标识符|
| `contact` |没有 |外部链接；可能需要 `homepage` / `sources` / `issues` |
| `icon` |没有 |图标嵌入的资源名称，参见4.7 |
| `environment` |没有 | `both`/`client`/`server`，默认`both`。当它与当前面不匹配时，整个模组不会加载 |
| `entry` |是的 |参赛班级的全名；该类必须有 `public Task Init()` |
| `depends` |没有 |它所依赖的其他模组和所需版本，请参见下文 |
| `hooks` |没有 |注入规则列表；空数组意味着仅订阅 ModApi 已有的事件 |
| `mixins` |没有 | mixin 规则列表；将此 mod 类之一的成员移动到目标类型中，请参阅 [2.8](#28-mixins-adding-members-to-a-target-type) |

`depends`声明它所依赖的其他mod和所需的版本；键是 mod id，值是版本约束：

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**未写入 `depends` 的依赖项没有版本约束**。模块间依赖关系已经从编译时引用中自动推断出来（参见 [6.1](#61-mod-dependency--injecting-the-depended-on-mod)），并且 `depends` 仅在顶部添加版本约束。当依赖的 mod 不在当前一侧时，将跳过检查 - 这种情况将留给程序集解析来报告。

|约束语法 |意义|
| --- | --- |
| `*` |任何版本，相当于省略条目 |
| `1.2.3` |段前缀； `1.2` 匹配 `1.2` 和 `1.2.9`，但不匹配 `1.3` |
| `^1.2.3` |相同的主要版本且不低于基线；当主版本为 0 时，将使用次版本，因此 `0.1` 和 `0.2` 算作不兼容 |
| `>=1.2.3` |不低于基线|

版本号仅采用每个段的前导数字，因此 `26.2-netcraft` 作为 `26.2` 参与。当版本不匹配时，**仅跳过声明的 mod**，其余部分照常加载；启动日志指出“依赖项 X 需要版本...，实际版本...”。

判断依据是依赖mod清单中的`version`字段。因此**想要依赖的 mod 必须正确设置 `version`**——空版本号不满足任何特定约束。

### 4.3 钩子规则字段

|领域 |笔记|
| --- | --- |
| `target` |目标类型的全名；必须位于 `mods/` | 下的内核程序集或 mod 程序集之一中
| `method` |目标方法名称；同名重载所有匹配 |
| `type` |注射剂形式见附录|
| `patchMode` |登陆方式，`ILRewrite`（默认）或`RuntimeInject`，参见[2.6](#26-runtime-injection-modifying-already-running-code) |
| `ordinal` |当同一个锚点匹配宿主方法中的多个位置时，选择哪个，从0开始，参见[2.4](#24-narrowing-to-one-site-host-scoping-and-placement) |
| `replaceType` |包含替换方法的类的全名 |
| `replaceMethod` |替换方法名称 |
| `label` |探针标签，仅由 `Mark` 和 `Probe` 使用
| `environment` |规则适用的一侧，默认`both`；针对在客户端上运行的服务器类型的规则根本没有目标，并且被 `environment` | 过滤掉。

同样的规则也可以写成替换方法上的注解；对应关系是：

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

|清单字段 |注释表格|
| --- | --- |
| `target` |第一个构造函数参数，写为 `typeof(...)` |
| `method` |第二个构造函数参数，最好是 `nameof(...)` |
| `type` |命名参数 `HookType`，默认 `CallSite` |
| `patchMode` |命名参数 `PatchMode`，默认 `ILRewrite` |
| `ordinal` |命名参数 `Ordinal` |
| `label` |命名参数 `Label` |
| `environment` |命名参数 `Environment`，默认 `both` |
| `replaceType` |没有写；自动从它注释的类中获取 |
| `replaceMethod` |没有写；从它注释的方法中自动获取 |

注释来自`NetCraft.ModApi.Extension`中的`InjectAttribute`，因此使用注释的mod必须引用它。当注释和清单都存在时，它们会合并，并且如果在两者中声明相同的注入点**注释获胜**。

Mixin规则使用不同的字段集：`target`（目标类型的全名），`source`（源类型的全名，必须在mod自己的程序集中），`interfaces`（可选，接口全名数组）；语义请参见[2.8](#28-mixins-adding-members-to-a-target-type)。源类上的 `[Mixin(typeof(target))]` 相当于清单条目。

### 4.4 什么可以被 hook，什么不能被 hook

可以挂钩：`kernel/`下的内核程序集，以及`mods/`下的其他mod。

无法挂钩：

- 主库`NetCraft.dll`
- 装载机`NetCraft.ModLoader.dll`
- 入口组件（`NetCraft.Server.Exe.dll` 等）

每个钩子的 `target` 必须在内核或某些 mod 程序集中可以找到，否则程序集会报告“注入目标不在任何已知程序集中”。请注意，**名称空间并不意味着程序集** - `NetCraft.Game.Server.DedicatedServer` 实际上位于 `NetCraft.Server.dll` 中；加载程序通过从元数据表构建的索引来查找它，因此只需写下全名即可。

Mod 的 Mod 注入遵循相同的系统：目标 mod 在它本身加载时被重写，无论 `mods/` 中的顺序如何。循环规则（A 注入 B，B 注入 A）在加载时报告“循环加载”；此类规则在IL重写级别没有解决方案，因此只需删除其中一条即可。

请注意，注入的 mod **不能是您自己的 mod ** - 自注入也算作一个循环。

### 4.5 部署

将编译后的 dll 复制到输出目录的 `mods/`（顶级，无子目录递归）中并重新启动该进程。

**此步骤已由构建自动执行**：`NetCraft.ModApi.csproj` 中的 `DeployModToHosts` 在构建后将 dll 复制到每个主机项目输出目录的 `mods/` 目录中。主机列表是`ModHostProjects`属性；创建自己的宿主项目时，只需添加其名称即可。如果没有规则生效并且完全没有错误，请首先检查宿主项目的`mods/`中的dll是否已过时。

启动日志打印：

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

故障排除顺序：

- 没有规则生效：首先检查`mods/`中的dll是否陈旧（最常见）。
- 如果您没有看到 `Rewrote and preloaded assembly`，则目标程序集已在引导程序之前加载，因此规则编写得太晚。
- 如果您看到`injection targets`，但目标是错误的，通常是输入错误的`target`；按照[4.4](#44-what-can-and-cannot-be-hooked)检查该类型属于哪个程序集。
- `environment` 设置为 `client` 的规则会在服务器上静默跳过（mod 级别不匹配意味着整个 mod 未加载）；这是预期行为，而不是错误。

### 4.6 第三方依赖

对应于 Fabric 的 Jar-in-Jar。

从模板创建的项目**不需要担心这个**：通过 `dotnet add package` 添加的库会在构建时自动嵌入到 mod dll 中，并且当加载程序无法解析程序集时，它会查看 mod 的嵌入 `.dll` 资源。

```
dotnet add package Newtonsoft.Json
```

就这样; `ncmod.json` 无需更改。

手写项目必须从模板 csproj 复制 `EmbedDependencies` 目标，或自行嵌入：

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

依赖关系不会写入清单中；解析仅查看 mod 的嵌入 `.dll` 资源，并且只需要资源名称与程序集名称匹配。

三点注意：

- **根据程序集名称进行匹配**。加载程序比较请求的程序集名称：删除 `.dll` 后，资源名称要么等于程序集名称，要么以 `.` + 程序集名称结尾。因此 `MyLib.dll` 和默认的 `MyProject.deps.MyLib.dll` 都可以工作。
- **仅识别模组自己的嵌入资源**。未嵌入的库无法解析，也不会在其他地方搜索。
- **解析顺序是内核优先**。 `EmbeddedAssemblyLoader` 首先在内核程序集和嵌入式子库中查找，如果没有找到，则仅回退到 mods，因此 mods 不应嵌入与内核程序集同名的程序集。

模板的目标使用`WithMetadataValue`过滤`.dll`而不是写入`Condition`，因为模板引擎在模板时评估`.csproj`中的`Condition`，当`%(...)`没有值时，整行将被删除。

### 4.7 图标和显示信息

名称、描述、作者、链接和图标都写在`ncmod.json`中，对应于原版`fabric.mod.json`中的`name` / `description` / `authors` / `contact` / `icon`。

**图标是嵌入式资源**，而不是外部文件，使用与嵌入式依赖项相同的资源命名约定：

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

该模板已附带配置了两个位置的 `icon.png`；只需替换该图像即可。建议使用 64×64 或 128×128 PNG。

图标查找顺序是：清单的 `icon` 指向的内容 → 名为 `icon.png` 的嵌入资源 → 如果两者都不存在，则为 UI 的默认图像（灰色问号）。

此信息在服务器 GUI 的 **MODS** 页面上可见。左侧是mod列表（小图标+显示名称+版本+状态）；右侧是所选项目的描述、制作人员、链接、依赖关系、初始化时间和注入规则。依赖列中的 Mod 名称可单击并直接跳转到该条目。

加载失败或被跳过的模组也在列表中，并在状态栏中标记——当诊断“为什么我的模组没有生效”时，请先检查此栏。

---

## 5. 已知限制

- **探测签名只能使用 BCL 类型和 `object`**，原因见 2.2；值类型参数和返回值是例外，必须保持其真实类型。
- **`CallSite`是替换**，替换方法必须恢复原来的调用本身，见2.3。私有方法无法恢复，需要反射。
- **`mods/`下的DLL由`DeployModToHosts`自动部署**；创建自己的宿主项目时，请记住将其名称添加到`ModHostProjects`。
- **入口程序集无法注入**：如果你的目标和钩子落在入口程序集中，则无效。
- **主库和加载器无法hook**；这是一个设计约束，阻止模组更改加载过程本身。
- **不匹配的`environment`意味着整个模组未加载**，而不是“某些规则失败”。
- **运行时注入需要本机库**：使用 `RuntimeInject` 的规则要求在进程启动时附加 `lead_hook_native`，并且加载器会自行重新启动以执行此操作；如果找不到库或者重启失败，则这批规则降级为警告，启动不被阻止。对于能力边界和成本，请参见[2.6](#26-runtime-injection-modifying-already-running-code)。
- **注释的字段少于 C# API**：`InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` 不能写入 `[Inject]`（支持 `PatchMode` 和 `Ordinal`），请参见 [2.1](#21-injection-styles-annotations-or-manifest-pick-one)。
- **Mixin仅适用于加载时重写**，并且源类中的成员被移动而不是复制；源类中的嵌套类型和泛型方法不在覆盖范围内，与接口混合的方法被标记为虚拟。参见[2.8](#28-mixins-adding-members-to-a-target-type)。
- 测试和调试时，如果使用语言表或模型资源，则需要 `assets` 目录（从 vanilla jar 中提取），否则相关功能将降级为翻译键或占位符纹理。

---

## 6. 已知行为

本节是观察到的行为的记录，而不是规范。

### 6.1 Mod依赖+注入依赖的mod

**场景**：b 依赖于 a 并且也注入了 a。

**结论**：有效且不形成循环。

该链分为三个步骤：

1. 规则表纯粹通过读取PE元数据构建，无需加载任何程序集。规则“b 注入 a”要求 a 和 b 都不存在。
2. `PreloadReplacers`加载`ModManager`之前的所有替换类（包括b），此时a尚未加载。 `LoadFromStream` 只读取元数据，不读取 JIT 方法体，因此此时 b 对 a 的引用是惰性的，加载不会失败。
3. `ModManager` 然后按 `AssemblyRef` 拓扑排序，a 在 b 之前。当重写a时，替换类b已经在`Default`中，因此直接获取类型并将一个指向b的`AssemblyRef`添加到a的元数据中。 **a 根本不需要知道 b 存在。**

**硬性要求**：在csproj中引用注入的mod时，必须写`Private="false"`。默认情况下，它将 `a.dll` 复制到输出目录中，然后由 `EmbedDependencies` 作为嵌入式依赖项嵌入到 `b.dll` 中，并且在运行时 `ModLibs` 解析 `a` 时会拾取第二个副本，导致两者之间的类型标识检查失败。

**验证案例**：两个模板项目`NetCraft.Test1`（a）和`NetCraft.Test2`（b）； a 提供 `Test1Api.Greet` 并在自己的 `ModEntry.Server` 中调用它，而 b 用 `Test2Probe.OnGreet` 替换该调用站点，并在 b 的 `ModEntry.Server` 中调用一次 `Greet` 作为控制。实际运行日志：

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← injection took effect
Test2 internal greeting: Test1 original greeting test2    ← control group: the rule only rewrites the target assembly, so b's own internal call site is untouched
```

依赖关系仅从 `AssemblyRef` 派生顺序，并且不带有版本约束。要限制版本，请在清单的 `depends` 中声明它们，请参阅 [4.2](#42-ncmodjson-fields)。

### 6.2 注解注入

**场景**：`[Inject(typeof(X), nameof(X.M))]`采用替换方法，`ncmod.json`中没有规则。

**结论**：它有效并且不需要明显的声明。注释和清单共享一个源，在组装时合并，并且注释在同一注入点上获胜。

**为什么不需要声明**：注释只是元数据中的 `CustomAttribute` 表，加载器在扫描 mod（清单、嵌入式资源和 AssemblyRef — 三项）时已经读取了相同的元数据，因此再读取一个表不会引入新的加载或计时约束。唯一的前提条件是 mod 引用 `NetCraft.ModApi`（注释的主机）。

**前提是静态读取**：读取注释必须经过`MetadataReader`，**绝对不能使用`Assembly.Load` + `GetCustomAttributes`**——后者拉起mod组件只是为了读取规则，重写窗口当场就没了。

**`typeof` 不构成类型引用**：`typeof(X)` 编译到参数中的是类型的序列化名称 (`full name, assembly, Version=…`)，它仅解析为字符串，不需要 `X` 存在。因此，在注释中写入 `typeof(injected-mod)` 并不会违反 2.2 中的约束，即“替换类不得引用注入的 mod 类型”——出现在元数据中的名称和在运行时解析类型是两件不同的事情。

**验证案例**：`NetCraft.Test2`针对`NetCraft.Test1`的两种方法； `Greet` 遍历注释，`Farewell` 遍历清单。两条规则均已安装，并且两个调用站点均已替换：

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← annotation
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← manifest
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← control group: the rule only rewrites the target assembly
Test2 internal farewell: Test1 original farewell test2     ← control group
```

同样的情况也顺便验证了注释的命名参数（`Environment = "server"`）也已解析。

### 6.3 附加分析器的性能成本

**场景**：相同的纯计算程序（一亿次模累加迭代），在没有分析器的情况下运行一次，在附加分析器的情况下运行一次（三个 `CORECLR_ENABLE_PROFILING` 环境变量）。每人跑五次。

**结论**：稳态无差异；成本完全是在启动时。

| |总处理时间（5 次运行，毫秒）|程序内计算时间 |
| --- | --- | --- |
|没有 | 306 / 260 / 278 / 253 / 300 | 225 毫秒 |
|与 | 406 / 370 / 340 / 325 / 354 | 193 毫秒 |

总时间大约长 80–110 毫秒。来源是`COR_PRF_DISABLE_ALL_NGEN_IMAGES`——启用ReJIT需要同时禁用ReadyToRun镜像，因此框架代码只能经过JIT； JIT 编译后，程序内的定时循环是相同的，没有显示任何差异（使用探查器的运行实际上稍快一些，这是噪音）。

**一路上也修复了一个浪费**：初始实现订阅了 `COR_PRF_MONITOR_JIT_COMPILATION`，每个方法编译后都会触发一个本机回调，但我们从未使用过。删除后，事件掩码从 `0x80040024` 更改为 `0x80040004`；上表是去除后的数据。

**验证案例**：`__hookverify/BenchProbe`。

### 6.4 两个 mod 注入同一目标

**场景**：两个mod各自声明一条命中相同目标方法的规则（`MethodBody`形式，不同的替换方法）。

**结论**：没有错误，没有崩溃； **第一个组装的规则获胜，而后一个则默默失败**。

两条规则都会进入规则表 - 不会进行跨模式重复数据删除。重写时，将采用 `OriginalType::OriginalMethod` 相同键列表中的**第一个**条目；主机作用域的行为方式相同，`InType`/`InMethod` 表示“第一场比赛获胜”。首先取决于汇编顺序，汇编顺序来自mods目录的枚举顺序； **没有优先级字段，无法通过声明依赖项来控制**（依赖项仅影响`Init()`的顺序，而不影响注入规则的组装）。

**运行时注入也是先到先得**：后面的注册请求正常发送，但是`GetReJITParameters`通过“模块+方法”声明，总是匹配第一个请求，所以后面的注册没有落地。在第二次注入后测量，目标方法的行为保持在第一个结果。

**一个错误的规则不会影响其他规则**：诸如目标类型不在任何已知程序集中或拼写错误的注入形式之类的问题会在汇编时记录在 `ModHooks.Errors` 中，并且会跳过该条目，而其他 mods 的规则会照常汇编。

**记录冲突**：当汇编检测到多个mod声明相同的注入点时，后汇编的会写入`ModHooks.Warnings`，并在启动日志中输出为`Mod injection conflict ...`，命名哪两个mod发生冲突，哪一个不生效。这是警告，不是错误，不影响加载，并且规则本身保留在表中（只是无法访问）。

**运行时注入也经过此检查**：冲突检测键包括 `patchMode`，因此一个写 `ILRewrite` 的 mod 和另一个写 `RuntimeInject` 不算冲突（两条独立路径，每个路径做自己的事情）；只有两个相同的模式被判断为冲突并发出警告。它的实际落地同样是先到先得——`GetReJITParameters`通过“模块+方法”声明，匹配第一个请求，所以后面的请求被发送但不落地。

**一个硬崩溃点**：当两个mod都在`RuntimePatch`模式下修补相同的方法时，第二个mod会遇到`RuntimeHookEngine`的重复注册检查并抛出`InvalidOperationException`，并且该路径未被捕获，因此启动彻底失败。 `RuntimePatch` 即将退出（参见 [2.6](#26-runtime-injection-modifying-already-running-code)）；不要在新规则中使用它。

**验证案例**：`NetCraft.Test`的`modinjection`模块，条目`same anchor first mod wins quietly`和`one bad rule does not sink the rest`。

### 6.5 尚未验证

- **在真实服务器上运行的运行时注入**：`RuntimeInject`模式已在`__hookverify/RuntimeProbe`中进行了端到端验证（注册后目标方法的行为将交换为替换方法），并且mod组装端也具有路由和降级的测试覆盖率；但当前没有构建步骤将 `lead_hook_native` 放入 NC 的运行目录中，因此在真实服务器上运行此链需要首先将库放入程序根目录中（或使用 `NC_PROFILER_PATH` 指向它）。这一步没有完成。
- **引用注入 mod 类型的替换类**：通过推理，在重写 a 时，它将解析 a，而 a 在加载完成之前被卡住（它尚未位于 `Default` 中，并且 `ModLibs` 和内核解析回调都无法识别 mod 程序集），因此 `PrepareMod` 预计会抛出并记录到 `result.Errors` 中。还没有真正运行。注6.2仅证明注释中的**`typeof`不是引用； **方法签名中出现的类型**是另一回事。

---

## 附录：HookType 概述

|表格 |效果|替换方法签名的要求 |
| --- | --- | --- |
| `CallSite` |将目标方法的调用站点替换为您的方法 |参数计数与被调用方法匹配（实例调用+1）|
| `MethodBody` |替换整个目标方法体 |匹配替换的方法 |
| `NewObj` |替换 `new X(...)` |参数计数与构造函数 | 匹配
| `FieldRead` |仪器字段读取|按读取类型 |
| `FieldWrite` |仪器领域写|按写入类型 |
| `TypeCheck` |仪器 `isinst` / `castclass` |按检查类型 |
| `Box` |仪器装箱/拆箱 |按元素类型 |
| `FunctionPointer` |仪表功能指针负载|按委托类型 |
| `LocalRead` |仪器局部变量读取 |零参数，返回变量的值 |
| `LocalWrite` |仪器局部变量写入 |一个参数，接收写入的值 |
| `Constant` |仪器恒定负载|零参数，返回常量的值 |
| `Probe` |保留原始方法主体，对入口和每个出口进行检测；使用 `LabelArgumentIndex`，一个参数可以折叠到标签 | 中`Begin()` 返回长整型，`End(string, long)` |
| `Mark` |仅在方法入口处报告一次，不计时 | `void method(string label)` |

`Probe` 和 `Mark` 仅传递标签文本（`Probe` 也可以包含一个参数的 `ToString()`）；他们无法获取对象引用。要获取实际参数，请使用 `CallSite`。

`InType`/`InMethod`、`Placement`和`Ordinal`（参见[2.4]（#24-narrowing-to-one-site-host-scoping-and-placement））仅对指令级形式有意义：上表中除`MethodBody`、`Probe`和`Mark`之外的十个条目可以选择replace或insert-before/after，并且可以使用`Ordinal`选择单个出现； `MethodBody` 总是替换整个内容，而 `Probe`/`Mark` 忽略这些参数。

对于`LocalRead`/`LocalWrite`/`Constant`三种，宿主方法写在`target`而不是引用实体中，另外需要`localIndex`或`constantValue`；参见[2.5](#25-in-method-body-anchors-local-variables-and-constants)。
