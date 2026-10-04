# NetCraft mod template

Sample code for NetCraft mod development. Every entry in `NetCraftTemplate.yaml`
points at one file under `examples/`, and `ncm template` pulls them on demand.

## Usage

```
ncm template view              list every entry
ncm template view Wrapper.?    filter by id, ? and * are wildcards
ncm template example <api id>  pull one example file into the current directory
```

## The two routes

`NetCraft.ModApi` exposes two namespaces, pick the one that fits:

| Namespace | What you get |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Events, `Nc*` facades and `Nc*` handles. No kernel type shows up in the public surface, so a kernel rename does not force a rebuild of your mod. |
| `NetCraft.ModApi.Extension` | `[Inject]` and `[Mixin]` attributes. Rules name kernel types and methods directly, which is more powerful and more fragile. |

## Handles

`NcPlayer` and `NcLevel` are read only handles. Everything they hand back is a
plain string, number or boolean (`NcLevel.Dimension`, `NcPlayer.X`), never a
kernel type, so a kernel rename does not force a rebuild of your mod. Block
operations take the level handle plus `x y z`.

## Common calls

Open this panel from inside a mod project and every call below that your code
actually uses is graded against `NetCraftTemplate.yaml`: green when the member
is declared, amber when the member is not, red when the type is not declared
at all. Hover a highlighted name to see the reason.

| Call | What it does |
| --- | --- |
| `NcServer.IsAvailable` | whether the server is up and captured |
| `NcServer.Broadcast` | system message to everyone online |
| `NcServer.Execute` | run a command as the console |
| `NcWorld.GetBlock` | read one block, null when the chunk is unloaded |
| `NcWorld.SetBlock` | write one block, runs the full update chain |
| `NcWorld.BreakBlock` | break a block the way a player would |
| `NcWorld.Overworld` | the overworld level handle |
| `NcLevel.Dimension` | dimension id of a level handle, for example minecraft:overworld |
| `NcLevel.DayTime` | read or set the time of one dimension |
| `NcPlayer.Name` | the player name |
| `NcPlayer.Health` | current health |
| `NcPlayers.Find` | look up an online player by name |
| `NcPlayers.Send` | private system message |
| `NcRegistries.FindState` | block state by namespaced id |
| `NcRegistries.FindItem` | item by namespaced id |
| `ServerEvents.Tick` | runs every server tick |
| `ServerEvents.PlayerJoin` | a player finished joining |
| `ServerEvents.BlockBroken` | a block was actually replaced |
