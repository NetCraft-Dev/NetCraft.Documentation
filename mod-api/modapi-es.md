# Referencia de NetCraft-ModApi

`NetCraft.ModApi` es la superficie de API que NetCraft expone a los mods. Tiene dos identidades: para ti es una biblioteca de API; para sí mismo es un mod común (`id` es `netcraft-modapi`, y distribuye su propio `ncmod.json` y sus propias sondas de inyección).

Este archivo crece a medida que crece la API. Para conocer los antecedentes arquitectónicos, las diferencias con Fabric y cómo escribir un mod, consulta [modding-guide-es.md](modding-guide-es.md).

- Ensamblado: `NetCraft.ModApi.dll`

- Dependencias: `NetCraft` (la biblioteca principal), `NetCraft.Game`

La superficie pública se divide en tres espacios de nombres:

| Espacio de nombres | Contenido | Notas |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | Clase base de eventos y suscripciones `NcEvent<T>`, fachadas `Nc*`, manejadores de objeto `Nc*` | Capa wrapper; sin tipos del kernel en la superficie pública |
| `NetCraft.ModApi.Extension` | Anotaciones `[Inject]` / `[Mixin]` | Puntos de extensión; las reglas se enlazan a nombres de clases y métodos del kernel |
| `NetCraft.ModApi.Internal` | Sondas de inyección | No referenciar directamente |

El espacio de nombres raíz `NetCraft.ModApi` contiene solo la clase de entrada `ModApiEntry`. `Wrapper` y `Extension` son dos rutas paralelas; para saber cómo elegir, consulta [modding-guide-es.md 2.9](modding-guide-es.md#29-dos-rutas-capa-wrapper-y-puntos-de-extensión).

***

## 1. Inicio rápido

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //la suscripción devuelve un manejador; liberarlo lo da de baja
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //también puedes hacer otra cosa con el manejador
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

En `ncmod.json`, apunta `entry` a esta clase y deja `hooks` vacío — los eventos de abajo los proporcionan todos las propias sondas de ModApi.

***

## 2. Eventos

Todos los eventos viven bajo `NetCraft.ModApi.Wrapper`; tras `using NetCraft.ModApi.Wrapper;` están disponibles.

### 2.1 Tabla resumen

| Evento                          | Tipo de args           | Disparador                                                | Lado   | Punto de enganche de ModApi                              |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick`             | `ServerTickArgs`       | cada tick del bucle principal del servidor                | servidor | `DedicatedServer::Tick` (Mark)                         |
| `ServerEvents.Started`          | `ServerPhaseArgs`      | el bucle principal se inició, después de imprimirse `Done (x.xxxs)!` | servidor | `MinecraftServer::Run` (Mark)                 |
| `ServerEvents.Stopping`         | `ServerPhaseArgs`      | el servidor comienza a apagarse; los jugadores están a punto de ser desconectados | servidor | `DedicatedServer::Stop` (Mark)         |
| `ServerEvents.CommandRegister`  | `CommandRegisterArgs`  | ya se han registrado todos los comandos integrados        | servidor | sitio de llamada de `EffectCommand::Register` (CallSite) |
| `ServerEvents.PlayerJoin`       | `PlayerJoinArgs`       | se ha enviado la secuencia de paquetes de entrada         | servidor | sitio de llamada a `PlayerList::PlaceNewPlayer` (CallSite) |
| `ServerEvents.PlayerLeave`      | `PlayerLeaveArgs`      | jugador eliminado de la lista de conectados               | servidor | sitio de llamada a `PlayerList::RemovePlayer` (CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | paquete de desconexión enviado y conexión cerrada         | servidor | sitio de llamada a `ServerPlayer::Disconnect` (CallSite) |
| `ServerEvents.PlayerHurt`       | `PlayerHurtArgs`       | daño realmente infligido                                  | servidor | sitio de llamada a `PlayerList::HurtPlayer` (CallSite)   |
| `ServerEvents.PlayerDeath`      | `PlayerDeathArgs`      | justo después de que la salud se restablece a cero        | servidor | sitio de llamada a `PlayerList::RespawnPlayer` (CallSite) |
| `ServerEvents.PlayerChat`       | `PlayerChatArgs`       | después de la difusión del chat                           | servidor | sitio de llamada a `ServerGamePacketListenerImpl::HandleChat` (CallSite) |
| `ServerEvents.ChunkLoaded`      | `ChunkLoadedArgs`      | el chunk entra en memoria por primera vez                 | servidor | sitio de asignación de `ServerChunkCache::set_ChunkLoaded` (CallSite) |
| `ServerEvents.ChunkUnloaded`    | `ChunkUnloadedArgs`    | el chunk sale de memoria                                  | servidor | sitio de asignación de `ServerChunkCache::set_ChunkUnloaded` (CallSite) |
| `ServerEvents.ChunkSaved`       | `ChunkSavedArgs`       | instantánea tomada antes de que el chunk se escriba en disco | servidor | sitio de asignación de `ServerChunkCache::set_ChunkSaveSink` (CallSite) |
| `ServerEvents.CommandExecuted`  | `CommandExecutedArgs`  | un comando terminó de ejecutarse; los errores de sintaxis y las denegaciones de permisos también cuentan | servidor | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick`        | `LevelTickArgs`        | tick de nivel, una vez por nivel cargado por tick         | servidor | `PersistentServerLevel::Tick` (CallSite)                 |
| `ServerEvents.SavedDataSaving`  | `SavedDataSavingArgs`  | datos guardados escritos en disco, un paso más tarde que el guardado del chunk | servidor | `SavedDataStorage::ScheduleSave` (CallSite) |
| `ServerEvents.BlockChanged`     | `BlockChangedArgs`     | estado de bloque cambiado, a punto de sincronizarse con los clientes | servidor | `IBlockUpdateSink::BlockChanged` (CallSite)    |
| `ServerEvents.BlockBroken`      | `BlockBrokenArgs`      | bloque roto; tanto la minería de un jugador como la autodestrucción por redstone cuentan | servidor | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped`      | `ItemDroppedArgs`      | entidad de objeto soltado generada, incluidos los objetos soltados al romper bloques y los productos de cocción | servidor | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived`  | `PacketReceivedArgs`   | cada paquete entrante encolado a un manejador, incluidas las fases de handshake y status | ambos  | `PacketProcessor::ScheduleIfPossible` y `HandleNow` (CallSite) |
| `ClientEvents.Tick`             | `ClientTickArgs`       | cada tick del bucle principal del cliente                 | cliente | `MinecraftClient::Tick` (Mark)                         |

Orden de los eventos de jugador: la muerte está anidada dentro del flujo de daño, por lo que `PlayerDeath` precede al `PlayerHurt` correspondiente; `PlayerLeave` y `PlayerDisconnect` son dos cosas diferentes — el primero significa la eliminación de la lista de conectados (incluso tras `/kick` solo se dispara cuando la conexión cae), el segundo significa la caída de la conexión en sí, y no se garantiza que aparezcan en pares.

### 2.2 Suscripción y baja

```csharp
IDisposable Subscribe(Action<T> handler)
```

- Suscribirse al mismo evento varias veces entrega las notificaciones en orden de suscripción.

- El despacho toma una instantánea de la lista de callbacks, por lo que suscribirse o darse de baja desde dentro de un callback no afecta al despacho actual.

- Sin darse de baja permanece vigente para siempre; los mods no ofrecen ningún mecanismo de descarga, por lo que la baja manual normalmente es innecesaria.

### 2.3 Tipos de args

**`ServerTickArgs`**

| Propiedad   | Tipo   | Notas                                        |
| ----------- | ------ | -------------------------------------------- |
| `TickCount` | `long` | recuento de ticks desde este arranque, empezando en 1 |

Nota: esto lo cuenta ModApi mismo, no el `TickCount` del kernel.

**`ClientTickArgs`**

| Propiedad   | Tipo   | Notas                                        |
| ----------- | ------ | -------------------------------------------- |
| `TickCount` | `long` | igual que arriba, contado de forma independiente en el lado del cliente |

**`ServerPhaseArgs`**

| Propiedad | Tipo     | Notas                        |
| --------- | -------- | ---------------------------- |
| `Phase`   | `string` | nombre de la fase, `started` o `stopping` |

El campo duplica el evento en sí; se mantiene para que el registro pueda usar un formato uniforme.

**`CommandRegisterArgs`**

| Miembro                              | Tipo                                    | Notas            |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher`                         | `CommandDispatcher<CommandSourceStack>` | el despachador de comandos del kernel |
| `Register(name, description, build)` | método                                  | registra un comando y lo anota en el libro de registro, ver 3.1 |

**`PlayerJoinArgs`**

| Propiedad     | Tipo           | Notas                                          |
| ------------- | -------------- | ---------------------------------------------- |
| `Player`      | `ServerPlayer` | el jugador que acaba de entrar; los paquetes de entrada ya se enviaron, el estado es seguro de leer |
| `ProfileName` | `string`       | nombre del jugador                             |

**`PlayerLeaveArgs`**

| Propiedad | Tipo           | Notas                                 |
| --------- | -------------- | ------------------------------------- |
| `Player`  | `ServerPlayer` | el jugador que se va, ya no está en la lista de conectados en este punto |
| `Removed` | `bool`         | si realmente se eliminó; `false` en una eliminación repetida |

**`PlayerDisconnectArgs`**

| Propiedad | Tipo           | Notas                                 |
| --------- | -------------- | ------------------------------------- |
| `Player`  | `ServerPlayer` | el jugador desconectado                |
| `Reason`  | `string`       | motivo de la desconexión; texto plano cuando se da como componente |

El paquete de desconexión ya se ha enviado y la conexión se ha cerrado; enviar paquetes a este jugador ahora no tiene efecto.

**`PlayerHurtArgs`**

| Propiedad  | Tipo            | Notas                                        |
| ---------- | --------------- | -------------------------------------------- |
| `Player`   | `ServerPlayer`  | el jugador que recibió el daño                |
| `Attacker` | `ServerPlayer?` | el jugador que infligió el daño; `null` para daño ambiental y de comandos |
| `Amount`   | `float`         | cantidad de daño esta vez                     |

No se dispara durante los fotogramas de invulnerabilidad ni después de la muerte (el `Hurt` del kernel devuelve `false`).

**`PlayerDeathArgs`**

| Propiedad  | Tipo            | Notas                          |
| ---------- | --------------- | ------------------------------ |
| `Player`   | `ServerPlayer`  | el jugador que murió           |
| `Attacker` | `ServerPlayer?` | el asesino; `null` cuando no hay ninguno |

El kernel se restablece inmediatamente después de que la salud llega a cero, así que cuando el evento se dispara el jugador ya está con la salud completa en el punto de reaparición; las coordenadas y los objetos soltados en el momento de la muerte no están disponibles.

**`PlayerChatArgs`**

| Propiedad    | Tipo     | Notas                |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | nombre del remitente |
| `Message`    | `string` | mensaje en texto plano |

Este evento es una **notificación de solo lectura**: el método original ya ha difundido el mensaje, por lo que cambiar `Message` aquí no tiene efecto.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

Los tres eventos comparten la misma forma de args:

| Propiedad | Tipo  | Notas              |
| --------- | ----- | ------------------ |
| `X`       | `int` | coordenada X del chunk |
| `Z`       | `int` | coordenada Z del chunk |

El más fácil de equivocar es `ChunkSaved`: el kernel requiere que este callback **tome la instantánea de forma síncrona**, mientras que la serialización y la escritura en disco las hace el propio kernel de forma asíncrona. Así que el trabajo que consume tiempo en este callback ralentiza directamente la descarga del chunk, y las operaciones destructivas (eliminar bloques, cambiar inventarios) tampoco deberían ir aquí — solo promete el momento de la instantánea.

Cuando `ChunkUnloaded` se dispara, las entidades de bloque ya se han limpiado junto con el chunk; si quieres leer bloques, usa `ChunkSaved` (que tampoco puede acceder a los bloques) o un punto anterior.

**`CommandExecutedArgs`**

| Propiedad | Tipo                  | Notas                                 |
| --------- | --------------------- | ------------------------------------- |
| `Command` | `string`              | texto bruto del comando; los comandos de chat no llevan barra inicial |
| `Result`  | `int`                 | valor de retorno del comando; 0 significa fallo o denegación |
| `Source`  | `CommandSourceStack?` | fuente del comando; `null` en la ruta de sobrecarga de jugador |
| `Player`  | `ServerPlayer?`       | el jugador que emitió el comando; `null` cuando se emite desde la consola |

El evento se dispara **después** de que el comando termina; no puede cambiar la ejecución. Los errores de sintaxis y las denegaciones de permisos también pasan por aquí; usa `Result` para distinguirlos. Los comandos que un jugador envía desde la barra de chat pasan por la sobrecarga `Execute(ServerPlayer, string)`, donde el kernel construye la fuente del comando internamente, así que en ese caso `Source` es `null` y solo `Player` está establecido.

**`LevelTickArgs`**

| Propiedad      | Tipo                    | Notas                                        |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level`        | `NcLevel`               | el nivel avanzado este tick                   |
| `RunsNormally` | `bool`                  | si avanzó normalmente; `false` durante `/tick freeze` |

Se dispara una vez por nivel cargado por tick, así que un mundo con varios niveles recibe varios por tick. El punto de disparo es después de que el tick de nivel **se ha completado**; es un punto de observación, no un punto de intercepción.

**`SavedDataSavingArgs`**

| Propiedad | Tipo                | Notas                       |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage`  | la tabla de datos guardados que se está persistiendo |

Esta es una ruta diferente de `ServerEvents.ChunkSaved`: la del chunk solo toma una instantánea y escribe de forma asíncrona, mientras que esta se dispara tras completarse una escritura síncrona. El reloj del mundo, las reglas de juego y los datos de la barrera del mundo pasan por aquí.

**`BlockChangedArgs`**

| Propiedad | Tipo         | Notas                       |
| --------- | ------------ | --------------------------- |
| `Pos`     | `BlockPos`   | posición del bloque cambiado |
| `State`   | `BlockState` | estado del bloque tras el cambio |

El estado ya se ha escrito en el chunk y está a punto de sincronizarse con los clientes, así que el cambio en sí no puede modificarse aquí. Los componentes de redstone que cambian su propio estado en callbacks de comportamiento también salen por esta ruta; es de alta frecuencia, así que no hagas trabajo que consuma tiempo en el callback.

**`BlockBrokenArgs`**

| Propiedad | Tipo            | Notas                                      |
| --------- | --------------- | ------------------------------------------ |
| `Pos`     | `BlockPos`      | posición del bloque roto                    |
| `Player`  | `ServerPlayer?` | quien lo rompió; `null` para causas no de jugador como la redstone |

Se dispara solo cuando el bloque es realmente reemplazado; las posiciones vacías y las roturas rechazadas no lo disparan. Los efectos de rotura y los objetos soltados ya se han gestionado, así que lo que lees en el evento es el resultado.

**`ItemDroppedArgs`**

| Propiedad | Tipo        | Notas                   |
| --------- | ----------- | ----------------------- |
| `Pos`     | `BlockPos`  | dónde apareció el objeto soltado |
| `Stack`   | `ItemStack` | la pila de objetos soltada |

Los objetos soltados al romper bloques y los productos de cocción de la hoguera pasan ambos por aquí. Una pila de objetos vacía no genera ninguna entidad, así que no hay evento.

**`PacketReceivedArgs`**

| Propiedad       | Tipo     | Notas                    |
| --------------- | -------- | ------------------------ |
| `Listener`      | `object` | el listener que recibe este paquete |
| `Packet`        | `object` | el objeto paquete en sí   |
| `IsServerbound` | `bool`   | si es un paquete serverbound |

Se dispara para cada paquete entrante, cubriendo las cuatro fases: handshake, status, configuration y play. Los paquetes de movimiento llegan varias veces por tick, así que no hagas trabajo que consuma tiempo en el callback. El paquete ya está decodificado en un objeto pero no ha entrado en la capa de negocio; para distinguir tipos, inspecciona `Packet` tú mismo. Los paquetes salientes están fuera del alcance de este evento.

***

## 3. Puntos de extensión

### 3.1 Registro de comandos

El momento es `ServerEvents.CommandRegister`. No guardes en caché los args de este evento; el árbol de comandos interno se construye solo una vez al arrancar.

```csharp
public void Register(
    string name,                                          //literal del comando, sin la barra
    string description,                                   //descripción de una línea, mostrada en el registro de /ncmapi
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //adjunta argumentos y el ejecutor
```

El `build` que recibes es el constructor brigadier del kernel; escribe argumentos, subcomandos y predicados de permisos al estilo del kernel:

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //predicado de permiso
            .Executes(context => { /* ... */ return 1; })));
```

Con argumentos:

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //ilustrativo
                return 1;
            })));
```

Puntos clave:

- Llamar directamente a `args.Dispatcher.Register(...)` también instala un comando, pero no entra en el libro de registro y `/ncmapi` no lo mostrará. Usa `args.Register` si quieres que aparezca listado.

- Los comandos no tienen restricción de permisos por defecto; añade `.Requires(...)` tú mismo si es necesario.

- El comportamiento en tiempo de ejecución depende enteramente de ti; ModApi no lo intercepta.

### 3.2 Ver los comandos registrados

Hay un `/ncmapi` integrado, que requiere nivel de permiso 2:

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

Las líneas de uso se calculan al vuelo a partir de la estructura de nodos del árbol de comandos: los literales se escriben por nombre, los argumentos se envuelven entre paréntesis angulares, y los nodos intermedios que son ejecutables por sí mismos también obtienen su propia línea.

***

## 4. Fachadas del servidor

Las fachadas de este capítulo viven todas bajo `NetCraft.ModApi.Wrapper`; tras `using NetCraft.ModApi.Wrapper;` están disponibles.

Las fachadas son clases estáticas `Nc*` que reúnen capacidades dispersas por el kernel en unos pocos puntos de entrada. La instancia del kernel la captura una sonda cuando arranca el bucle principal; cuando `NcServer.IsAvailable` es false todo lo de abajo lanza excepción — úsalas solo dentro de callbacks de eventos, no desde `Init`.

| Fachada | Propósito |
| --- | --- |
| `NcServer` | instancia del servidor, tasa de ticks, comandos, seguimiento de entidades, datos de jugador, reglas de juego, difusión, ejecución de comandos |
| `NcPlayers` | consultas y operaciones sobre jugadores conectados (kick, teletransporte, salud, modo de juego, permisos) |
| `NcWorld` | lectura/escritura y rotura de bloques del mundo normal, clima, tiempo, barrera, reloj, sonidos, eventos de nivel; toma manejadores `NcLevel` para llegar a otras dimensiones, las coordenadas son enteros `x y z` simples |
| `NcRegistries` | registros integrados buscados por nombre (bloques, objetos, fluidos, efectos, biomas, partículas, entidades, entidades de bloque) |
| `NcRecipes` | consultas de recetas (fabricación en rejilla, corte de piedra, cocción; obtener recetas por id) |
| `NcLists` | listas y configuración (whitelist, ops, bans, `server.properties`) |
| `NcStartup` | argumentos de arranque (tokens no reconocidos por el kernel y suscripción basada en nombre) |

`NcPlayer` no es una fachada estática sino un **manejador de objeto**: `NcPlayers.All` / `Find` lo devuelven, y `Player` / `Attacker` en los eventos de jugador también lo son. Los manejadores son de solo lectura y los construyen las sondas; los mods no pueden obtener el `ServerPlayer` del kernel — el primer ancla de "sin tipos del kernel en la superficie pública". El mismo jugador del kernel siempre se asigna al mismo manejador, cacheado internamente por referencia débil e invalidado automáticamente una vez que el jugador se desconecta.

`NcLevel` sigue la misma forma para los niveles. `NcWorld.Overworld` / `Nether` / `End` y `NcWorld.Get("minecraft:the_nether")` lo devuelven, y `LevelTickArgs.Level` también es uno. Lleva el id de la dimensión, el tiempo, el clima, la altura de construcción, el recuento de ticks y la carga forzada de chunks; las operaciones de bloques se quedan en `NcWorld` y toman el manejador más `x y z`. `BlockPos` nunca aparece, así que el dll de un mod no lleva ninguna referencia al tipo de nivel del kernel.

### 4.1 Registros

`NcRegistries` proporciona tanto tablas completas como búsquedas por nombre. Las tablas completas son para iterar y para búsquedas basadas en etiquetas; las búsquedas por nombre son para obtener un solo elemento:

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //estado por defecto del bloque
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//recorre toda la tabla
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

Los registros se ensamblan gradualmente durante el arranque, y los mods se cargan antes de que el ensamblado se complete, así que no guardes en caché nada buscado en `Init` — el ensamblado sigue en curso, y un valor cacheado será una referencia nula o un valor obsoleto. Actualmente `BuiltInRegistries.BootStrap` sigue siendo una implementación vacía; cada registro se rellena por separado mediante su propio Bootstrap, y los que dependen de datos (biomas, recetas, etc.) tienen muy pocas entradas antes de que se conecte la carga del paquete de datos.

### 4.2 Recetas

`NcRecipes` está respaldado por una tabla de recetas cargada desde los paquetes de datos; `/reload` reemplaza toda la tabla, así que no mantengas un `RecipeHolder` entre recargas.

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //calcula la salida para una rejilla de fabricación
    var recipes = NcRecipes.StonecuttingFor(stack);         //recetas de corte de piedra disponibles para esta entrada
    var smelting = NcRecipes.CookingFor("smelting", stack); //busca por tipo de cocción
    var byId = NcRecipes.Find("minecraft:oak_planks");      //obtiene una receta por id
}
```

***

## 5. Internos

No necesitas esta sección para escribir mods, pero puede ayudar al depurar.

### 5.1 Sondas

| Clase                                                    | Forma       | Responsabilidad                                             |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)`                  | Mark ×4     | todas las señales de "algo ocurrió" confluyen en un método, despachadas al evento correspondiente por `label` |
| `Internal.CommandProbe.OnCommandsReady(object)`          | CallSite    | reemplaza la llamada a `EffectCommand::Register`; tras restaurar la llamada original dispara `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)`                | CallSite ×6 | eventos de jugador, un método por punto de enganche; tras restaurar la llamada original publica el evento |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | eventos de chunk, enganchados en los sitios de asignación de las tres propiedades de callback de `ServerChunkCache`; se coloca un delegado envoltorio delante antes de devolver el control al kernel |
| `Internal.BlockProbe.OnXxx(...)`                         | CallSite ×3 | eventos de bloque; la rotura y los objetos soltados enganchan `ServerBlockUpdates`, los cambios de estado enganchan el método de interfaz en `IBlockUpdateSink` |

La firma de `SignalProbe` solo toma `string`, y los parámetros de `CommandProbe`, `PlayerProbe` y `LevelProbe` están declarados como `object` — esto es deliberado: durante el ensamblado `Lead.Hook` resuelve la firma del método de reemplazo, y en cuanto aparece un tipo del kernel en ella, resolverlo arrastra el ensamblado del kernel demasiado pronto y la inyección pierde su ventana. Los tipos del kernel aparecen solo dentro de los cuerpos de los métodos, cuando el código ya se está ejecutando.

Lo único que no puede ser `object` en una firma son los parámetros y valores de retorno de tipo valor: `object` es una referencia en la pila mientras que `float`/`bool` son valores, y una falta de coincidencia es IL inválido. Por eso `PlayerProbe.OnHurtPlayer` conserva `float` para la cantidad de daño, y `OnRemovePlayer` y `OnHurtPlayer` conservan valores de retorno `bool`.

`BlockProbe` es una extensión de esta restricción: la posición y el estado del bloque son los dos tipos valor `BlockPos`/`BlockState`, que solo pueden escribirse en la firma como ellos mismos. Estos dos tipos vienen de `NetCraft.Primitives` y `NetCraft.Registry`, ninguno de los cuales está en la lista de inyección, así que resolverlos durante el ensamblado no arrastra demasiado pronto los ensamblados que hay que reescribir.

### 5.2 Lista de puntos de enganche

El `ncmod.json` de ModApi contiene veinticuatro reglas, que coinciden una a una con la tabla de 2.1. Para cambiar un punto de enganche o añadir una regla, edita este archivo; tras editarlo, recompila (es un recurso incrustado) y vuelve a poner el dll resultante en `mods/` — esto último ya lo hace automáticamente `DeployModToHosts` en `NetCraft.ModApi.csproj`, y omitirlo se manifiesta como que las reglas no surten efecto en absoluto.

`CommandManager::Execute` tiene dos sobrecargas que comparten una regla. El CallSite de `Lead.Hook` hace coincidir los sitios de llamada por "tipo + nombre de método", no por lista de parámetros, y ambas sobrecargas toman dos parámetros, así que la sonda puede tomar `object` para el primer parámetro y despachar por el tipo real.

Las dos reglas de `PacketProcessor` son complementarias: los paquetes de la fase play pasan por `ScheduleIfPossible` a la cola del hilo principal, mientras que las fases handshake y status pasan por `HandleNow` para su manejo inmediato; cualquier paquete dado pasa solo por una de ellas. Enganchar solo la primera se pierde las fases handshake y status — que además son las más fáciles de sondear con scripts, así que al depurar esto se malinterpreta fácilmente como "la regla no surtió efecto".

Las tres reglas de chunk enganchan los **sitios de asignación** de las tres propiedades de callback de `ServerChunkCache`, no los sitios de lectura. La razón es que esas tres propiedades son unicast y ya están ocupadas por el propio kernel cuando se construye `PersistentServerLevel` (inyectan la lógica de guardado y de limpieza de entidades de bloque); un mod que asignara directamente sobrescribiría la copia del kernel — descargas no persistidas, entidades de bloque no limpiadas, y sin ningún error. Enganchar el sitio de asignación permite encadenar el callback del kernel y la sonda en ese momento; la asignación ocurre solo una vez, y cada disparo posterior añade una capa de reenvío de delegado.

`PlayerList::RespawnPlayer` es privado, así que la sonda no puede restaurar la llamada original; esa pasa por reflexión (se llama una vez por muerte, así que la sobrecarga es insignificante). Esto también deja una puerta abierta para la alineación con el kernel: si en el futuro se añade `InternalsVisibleTo` para él, puede cambiarse a una llamada directa.

### 5.3 Libro de registro

`Internal.NcCommandRegistry` registra los comandos registrados mediante `args.Register`. Es solo un libro de registro y no participa en la ejecución de comandos; los comandos en sí se instalan en el despachador del kernel, así que aunque el libro de registro tenga problemas, los comandos siguen funcionando.

***

## 6. Por añadir

Los siguientes son puntos de enganche cuya ubicación está confirmada pero que aún no se han convertido en eventos (la lista la produjo `__scan_mod_api.py` en la raíz del repositorio):

| Dirección         | Puntos de enganche candidatos                                                |
| ----------------- | ---------------------------------------------------------------------------- |
| Entidades         | `ClientLevel::AddEntity`, `Entity::Die`                                      |
| Mundo             | carga y descarga de nivel, agrupación de chunks de `ServerChunkCache`        |
| Generación de terreno | etapas de `ChunkGenerator::Generate` por `ChunkStatus`, `WorldGenRegion::SetBlockState` |
| Ejecución de comandos | `CommandSourceStack::SendSuccess` / `SendFailure` (la mitad de la respuesta, con muchos sitios de llamada) |
| Red               | `ServerGamePacketListenerImpl::HandleXxx` por tipo de paquete (actualmente solo un punto de entrada unificado) |

Direcciones ya hechas: los ticks de nivel se convirtieron en `ServerEvents.LevelTick`, la persistencia de datos guardados se convirtió en `ServerEvents.SavedDataSaving`, la ejecución de comandos se convirtió en `ServerEvents.CommandExecuted`, y los bloques se convirtieron en `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`.

Enganchar la celda de bloque en `SetBlock` no funciona: tiene dos parámetros por defecto, `notifyNeighbors` y `strict`, así que los sitios de llamada compilados toman entre 4 y 6 parámetros, y como CallSite hace coincidir por "tipo + nombre de método" sin mirar la lista de parámetros, un método de reemplazo no puede manejar las tres formas de pila. En su lugar engancha dos sitios: la sincronización de estado engancha `IBlockUpdateSink::BlockChanged` (la única llamada de interfaz en `ServerLevel.SetBlock`, que cubre todo cambio con sincronización de cliente), y la rotura y los objetos soltados enganchan los propios métodos de `ServerBlockUpdates`.

La celda de entidades es problemática porque `Entity` está definido en `NetCraft.Registry`, que no está en la lista de inyección, así que los sitios de llamada dirigidos a él no pueden reescribirse. `ClientLevel::AddEntity` está en el ensamblado del cliente y es factible; un evento de muerte requiere primero resolver si `Registry` puede reescribirse.

Para la celda de red, `HandleChat` existe desde hace tiempo y `NetworkEvents.PacketReceived` proporciona un punto de entrada unificado, así que los enganches por `HandleXxx` tienen mucho menos valor; solo merece la pena añadirlos en escenarios que necesiten un filtrado fino por tipo de paquete.

La carga y descarga de nivel no tienen punto de convergencia en el lado de NC: `DedicatedServer::CreateLevel` es privado, así que la llamada original solo puede restaurarse por reflexión como `PlayerList::RespawnPlayer`; la ruta de descarga está aún más dispersa. Para hacerlo, primero hay que decidir cómo deberían ser los args del evento.

Todos los restantes necesitan referencias de objeto (instancias de entidad, etc.), así que deben usar `CallSite` en lugar de `Mark`; si entre los parámetros aparece un tipo valor del kernel, solo puede escribirse en la firma del método de reemplazo como él mismo.

También se revisó la lista de superficie de interfaz (`__modapi_api.txt`, producida por `__scan_mod_api.py --api`): las entradas calificadas como puntos de entrada de capacidad se reunieron en las fachadas del capítulo 4 por dominio, y el resto que no se expone cae en tres categorías — manejo de protocolo y paquetes (`Network.Protocol.*`), renderizado y modelos (`Client.Render.*`), y generación de terreno y funciones de densidad (`LevelGen.*`). Estos son internos del kernel; usarlos directamente ataría los mods a detalles de implementación, así que primero debería abrirse una interfaz estable en el kernel.

En el lado del servidor hay dos cosas más no envueltas como fachadas: la instancia `ReloadableServerResources` cuelga de `DedicatedServer`, y como ModApi no referencia `NetCraft.Server`, envolverla requiere primero abrir una propiedad en la clase base del kernel; `ChunkSender` y `ServerWorldBorderListener` son flujos internos sin caso de uso para los mods.
