# Plantilla de mod de NetCraft

Código de ejemplo para el desarrollo de mods de NetCraft. Cada entrada en `NetCraftTemplate.yaml` apunta a un archivo bajo `examples/`, y `ncm template` los extrae bajo demanda.

## Uso

```
ncm template view              listar cada entrada
ncm template view Wrapper.?    filtrar por id, ? y * son comodines
ncm template example <api id>  extraer un archivo de ejemplo al directorio actual
```

## Las dos rutas

`NetCraft.ModApi` expone dos espacios de nombres; elige el que te convenga:

| Espacio de nombres | Qué obtienes |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Eventos, fachadas `Nc*` y handles `Nc*`. Ningún tipo del kernel aparece en la superficie pública, así que un cambio de nombre del kernel no obliga a recompilar tu mod. |
| `NetCraft.ModApi.Extension` | Atributos `[Inject]` y `[Mixin]`. Las reglas nombran directamente tipos y métodos del kernel, lo cual es más potente y más frágil. |

## Handles

`NcPlayer` y `NcLevel` son handles de solo lectura. Todo lo que devuelven es una cadena, un número o un booleano simples (`NcLevel.Dimension`, `NcPlayer.X`), nunca un tipo del kernel, así que un cambio de nombre del kernel no obliga a recompilar tu mod. Las operaciones de bloques toman el handle de nivel más `x y z`.

## Llamadas comunes

Abre este panel desde dentro de un proyecto de mod y cada llamada de abajo que tu código use realmente se evalúa contra `NetCraftTemplate.yaml`: verde cuando el miembro está declarado, ámbar cuando el miembro no existe, rojo cuando el tipo no está declarado en absoluto. Pasa el cursor sobre un nombre resaltado para ver el motivo.

| Llamada | Qué hace |
| --- | --- |
| `NcServer.IsAvailable` | si el servidor está activo y capturado |
| `NcServer.Broadcast` | mensaje del sistema a todos los que están en línea |
| `NcServer.Execute` | ejecutar un comando como consola |
| `NcWorld.GetBlock` | leer un bloque, null cuando el chunk no está cargado |
| `NcWorld.SetBlock` | escribir un bloque, ejecuta la cadena de actualización completa |
| `NcWorld.BreakBlock` | romper un bloque como lo haría un jugador |
| `NcWorld.Overworld` | el handle de nivel del Overworld |
| `NcLevel.Dimension` | id de dimensión de un handle de nivel, por ejemplo minecraft:overworld |
| `NcLevel.DayTime` | leer o establecer la hora de una dimensión |
| `NcPlayer.Name` | el nombre del jugador |
| `NcPlayer.Health` | salud actual |
| `NcPlayers.Find` | buscar un jugador en línea por nombre |
| `NcPlayers.Send` | mensaje privado del sistema |
| `NcRegistries.FindState` | estado de bloque por id con espacio de nombres |
| `NcRegistries.FindItem` | ítem por id con espacio de nombres |
| `ServerEvents.Tick` | se ejecuta en cada tick del servidor |
| `ServerEvents.PlayerJoin` | un jugador terminó de unirse |
| `ServerEvents.BlockBroken` | un bloque fue realmente reemplazado |