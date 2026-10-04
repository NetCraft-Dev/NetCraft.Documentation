# NetCraft mod 模板

NetCraft mod 开发的示例代码。 `NetCraftTemplate.yaml` 中的每个条目
指向 `examples/` 下的一个文件，`ncm template` 按需提取它们。

＃＃ 用法

```
ncm template view              list every entry
ncm template view Wrapper.?    filter by id, ? and * are wildcards
ncm template example <api id>  pull one example file into the current directory
```

## 两条路线

`NetCraft.ModApi` 公开两个命名空间，选择适合的一个：

|命名空间|你得到什么 |
| --- | --- |
| `NetCraft.ModApi.Wrapper` |事件、`Nc*` 外观和 `Nc*` 句柄。公共表面上不会显示任何内核类型，因此内核重命名不会强制重建您的 mod。 |
| `NetCraft.ModApi.Extension` | `[Inject]` 和 `[Mixin]` 属性。规则直接命名内核类型和方法，功能更强大，但也更脆弱。 |

## 句柄

`NcPlayer` 和 `NcLevel` 是只读句柄。他们返还的一切都是
纯字符串、数字或布尔值（`NcLevel.Dimension`、`NcPlayer.X`），绝不是
内核类型，因此内核重命名不会强制重建您的 mod。块
操作采用级别句柄加 `x y z`。

## 常用调用

从 mod 项目内部打开此面板以及您的代码下面的每个调用
实际使用的是根据 `NetCraftTemplate.yaml` 进行评分：当成员为绿色时
已声明，如果未声明成员，则为琥珀色；如果未声明类型，则为红色
根本不。将鼠标悬停在突出显示的名称上即可查看原因。

|致电 |它有什么作用 |
| --- | --- |
| `NcServer.IsAvailable` |服务器是否已启动并被捕获 |
| `NcServer.Broadcast` |系统给大家在线留言|
| `NcServer.Execute` |作为控制台运行命令 |
| `NcWorld.GetBlock` |读取一个块，卸载该块时为 null |
| `NcWorld.SetBlock` |写一个块，运行完整的更新链 |
| `NcWorld.BreakBlock` |像玩家一样打破方块 |
| `NcWorld.Overworld` |主世界关卡手柄|
| `NcLevel.Dimension` |关卡句柄的维度 ID，例如 minecraft:overworld |
| `NcLevel.DayTime` |读取或设置一维时间|
| `NcPlayer.Name` |玩家姓名 |
| `NcPlayer.Health` |目前健康状况 |
| `NcPlayers.Find` |按姓名查找在线玩家 |
| `NcPlayers.Send` |私人系统留言|
| `NcRegistries.FindState` |通过命名空间 id 阻止状态 |
| `NcRegistries.FindItem` |命名空间 id 的项目 |
| `ServerEvents.Tick` |运行每个服务器tick |
| `ServerEvents.PlayerJoin` |一名玩家完成加入 |
| `ServerEvents.BlockBroken` |一个块实际上被替换了 |
