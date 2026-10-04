# NetCraft 模组模板

NetCraft 模组开发的示例代码。`NetCraftTemplate.yaml` 中的每个条目都指向
`examples/` 下的一个文件，`ncm template` 按需拉取。

## 用法

```
ncm template view              列出所有条目
ncm template view Wrapper.?    按 id 过滤，? 和 * 是通配符
ncm template example <api id>  拉取一个示例文件到当前目录
```

## 两条路线

`NetCraft.ModApi` 暴露两个命名空间，挑合适的那个：

| 命名空间 | 你能得到什么 |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | 事件、`Nc*` 门面和 `Nc*` 句柄。公共表面上不出现任何内核类型，因此内核改名不会迫使你重新构建模组。 |
| `NetCraft.ModApi.Extension` | `[Inject]` 和 `[Mixin]` 特性。规则直接写出内核类型和方法名，功能更强也更脆弱。 |

## 常用调用

在模组项目内打开这个面板，下面每一个你的代码实际用到的调用都会
对照 `NetCraftTemplate.yaml` 评级：成员已声明为绿色，成员不存在为琥珀色，
类型完全未声明为红色。悬停在高亮的名称上可查看原因。

| 调用 | 作用 |
| --- | --- |
| `NcServer.IsAvailable` | 服务器是否已启动并捕获 |
| `NcServer.Broadcast` | 向所有在线玩家发送系统消息 |
| `NcServer.Execute` | 以控制台身份执行一条命令 |
| `NcWorld.GetBlock` | 读取一个方块，区块未加载时返回 null |
| `NcWorld.SetBlock` | 写入一个方块，走完整的更新链 |
| `NcWorld.BreakBlock` | 像玩家那样破坏一个方块 |
| `NcPlayer.Name` | 玩家名 |
| `NcPlayer.Health` | 当前生命值 |
| `NcPlayers.Find` | 按名字查找在线玩家 |
| `NcPlayers.Send` | 私发系统消息 |
| `NcRegistries.FindState` | 按命名空间 id 获取方块状态 |
| `NcRegistries.FindItem` | 按命名空间 id 获取物品 |
| `ServerEvents.Tick` | 每个服务端 tick 运行 |
| `ServerEvents.PlayerJoin` | 一名玩家完成加入 |
| `ServerEvents.BlockBroken` | 一个方块被实际替换 |
