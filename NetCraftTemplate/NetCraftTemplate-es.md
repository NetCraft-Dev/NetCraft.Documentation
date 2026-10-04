# Plantilla de mod de NetCraft

Código de ejemplo para el desarrollo de mods de NetCraft. Cada entrada en `NetCraftTemplate.yaml`
apunta a un archivo bajo `examples/`, y `ncm template` los extrae bajo demanda.

## Uso

```
ncm template view              lista todas las entradas
ncm template view Wrapper.?    filtra por id, ? y * son comodines
ncm template example <api id>  extrae un archivo de ejemplo al directorio actual
```

## Las dos rutas

`NetCraft.ModApi` expone dos espacios de nombres, elige el que te convenga:

| Espacio de nombres | Lo que obtienes |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Eventos, fachadas `Nc*` y manejadores `Nc*`. Ningún tipo del kernel aparece en la superficie pública, así que un cambio de nombre en el kernel no obliga a recompilar tu mod. |
| `NetCraft.ModApi.Extension` | Atributos `[Inject]` y `[Mixin]`. Las reglas nombran directamente tipos y métodos del kernel, lo cual es más potente y más frágil. |

## Llamadas comunes

Abre este panel desde dentro de un proyecto de mod y cada llamada de abajo que tu código
use realmente se evalúa contra `NetCraftTemplate.yaml`: verde cuando el miembro
está declarado, ámbar cuando el miembro no lo está, rojo cuando el tipo no está declarado
en absoluto. Pasa el cursor sobre un nombre resaltado para ver el motivo.

| Llamada | Qué hace |
| --- | --- |
| `NcServer.IsAvailable` | si el servidor está activo y capturado |
| `NcServer.Broadcast` | mensaje de sistema a todos los conectados |
| `NcServer.Execute` | ejecuta un comando como la consola |
| `NcWorld.GetBlock` | lee un bloque, null cuando el chunk está descargado |
| `NcWorld.SetBlock` | escribe un bloque, ejecuta toda la cadena de actualización |
| `NcWorld.BreakBlock` | rompe un bloque tal como lo haría un jugador |
| `NcPlayer.Name` | el nombre del jugador |
| `NcPlayer.Health` | salud actual |
| `NcPlayers.Find` | busca un jugador conectado por nombre |
| `NcPlayers.Send` | mensaje de sistema privado |
| `NcRegistries.FindState` | estado de bloque por id con espacio de nombres |
| `NcRegistries.FindItem` | objeto por id con espacio de nombres |
| `ServerEvents.Tick` | se ejecuta en cada tick del servidor |
| `ServerEvents.PlayerJoin` | un jugador terminó de entrar |
| `ServerEvents.BlockBroken` | un bloque fue realmente reemplazado |
