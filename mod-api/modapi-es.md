# Referencia de NetCraft-ModApi

`NetCraft.ModApi` es la superficie API que NetCraft expone a las modificaciones. Tiene dos identidades: para ti es una biblioteca API; por sí mismo es un mod ordinario (`id` es `netcraft-modapi`, se envía su propio `ncmod.json` y sondas de inyección).

Este archivo crece a medida que crece la API. Para conocer los antecedentes arquitectónicos, las diferencias con Fabric y cómo escribir un mod, consulte [modding-guide.md](modding-guide.md).

- Montaje: `NetCraft.ModApi.dll`

- Dependencias: `NetCraft` (la biblioteca principal), `NetCraft.Game`

La superficie pública se divide en tres espacios de nombres:

| Espacio de nombres | Contenidos | Notas |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | Clase base de eventos y suscripción `NcEvent<T>`, `Nc*` fachadas, `Nc*` identificadores de objetos | Capa envolvente; no hay tipos de kernel en la superficie pública |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` anotaciones | Puntos de extensión; reglas se unen a los nombres de métodos y clases del núcleo |
| `NetCraft.ModApi.Internal` | Sondas de inyección | No hacer referencia directamente |

El espacio de nombres raíz `NetCraft.ModApi` contiene solo la clase de entrada `ModApiEntry`. `Wrapper` y `Extension` son ​​dos rutas paralelas; Para saber cómo elegir, consulte [modding-guide.md 2.9](modding-guide.md#29-two-routes-wrapper-layer-and-extension-points).

***

## 1. Inicio rápido

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

En `ncmod.json`, apunte `entry` a esta clase y deje `hooks` vacío; todos los eventos siguientes son proporcionados por las propias sondas de ModApi.

***

## 2. Eventos

Todos los eventos se viven bajo `NetCraft.ModApi.Wrapper`; después de `using NetCraft.ModApi.Wrapper;` están disponibles.

### 2.1 Tabla resumen

| Evento | Tipo de argumentos | Gatillo | Lado | Punto de gancho ModApi |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick` | `ServerTickArgs` | cada tic del bucle principal del servidor | servidor | `DedicatedServer::Tick` (Marca) |
| `ServerEvents.Started` | `ServerPhaseArgs` | bucle principal iniciado, después de imprimir `Done (x.xxxs)!` | servidor | `MinecraftServer::Run` (Marca) |
| `ServerEvents.Stopping` | `ServerPhaseArgs` | el servidor comienza a cerrarse; jugadores están a punto de ser desconectados | servidor | `DedicatedServer::Stop` (Marca) |
| `ServerEvents.CommandRegister` | `CommandRegisterArgs` | todos los comandos integrados han sido registrados | servidor | `EffectCommand::Register` sitio de llamada (CallSite) |
| `ServerEvents.PlayerJoin` | `PlayerJoinArgs` | se ha enviado la secuencia del paquete de unión | servidor | `PlayerList::PlaceNewPlayer` sitio de llamada (CallSite) |
| `ServerEvents.PlayerLeave` | `PlayerLeaveArgs` | jugador eliminado de la lista online | servidor | `PlayerList::RemovePlayer` sitio de llamada (CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | desconectar paquete enviado y conexión cerrada | servidor | `ServerPlayer::Disconnect` sitio de llamada (CallSite) |
| `ServerEvents.PlayerHurt` | `PlayerHurtArgs` | daño realmente causado | servidor | `PlayerList::HurtPlayer` sitio de llamada (CallSite) |
| `ServerEvents.PlayerDeath` | `PlayerDeathArgs` | inmediatamente después de que la salud se restablezca a cero | servidor | `PlayerList::RespawnPlayer` sitio de llamada (CallSite) |
| `ServerEvents.PlayerChat` | `PlayerChatArgs` | después de la transmisión del chat | servidor | `ServerGamePacketListenerImpl::HandleChat` sitio de llamada (CallSite) |
| `ServerEvents.ChunkLoaded` | `ChunkLoadedArgs` | trozo entra en la memoria por primera vez | servidor | `ServerChunkCache::set_ChunkLoaded` sitio de asignación (CallSite) |
| `ServerEvents.ChunkUnloaded` | `ChunkUnloadedArgs` | trozo deja memoria | servidor | `ServerChunkCache::set_ChunkUnloaded` sitio de asignación (CallSite) |
| `ServerEvents.ChunkSaved` | `ChunkSavedArgs` | instantánea tomada antes de que el fragmento se escriba en el disco | servidor | `ServerChunkCache::set_ChunkSaveSink` sitio de asignación (CallSite) |
| `ServerEvents.CommandExecuted` | `CommandExecutedArgs` | un comando terminó de ejecutarse; los errores de sintaxis y las denegaciones de permisos también cuentan | servidor | `CommandManager::Execute` (Sitio de llamada) |
| `ServerEvents.LevelTick` | `LevelTickArgs` | tick de nivel, una vez por nivel cargado por tick | servidor | `PersistentServerLevel::Tick` (Sitio de llamada) |
| `ServerEvents.SavedDataSaving` | `SavedDataSavingArgs` | datos guardados escritos en el disco, un paso después del guardado de fragmentos | servidor | `SavedDataStorage::ScheduleSave` (Sitio de llamada) |
| `ServerEvents.BlockChanged` | `BlockChangedArgs` | estado del bloque cambiado, a punto de sincronizarse con los clientes | servidor | `IBlockUpdateSink::BlockChanged` (Sitio de llamada) |
| `ServerEvents.BlockBroken` | `BlockBrokenArgs` | bloque roto; La minería de jugadores y la autodestrucción de Redstone cuentan | servidor | `ServerBlockUpdates::BreakBlock` (Sitio de llamada) |
| `ServerEvents.ItemDropped` | `ItemDroppedArgs` | entidad de artículo caído generada, incluidas caídas de ruptura de bloques y productos de cocina | servidor | `ServerBlockUpdates::SpawnDrop` (Sitio de llamada) |
| `NetworkEvents.PacketReceived` | `PacketReceivedArgs` | cada paquete entrante puesto en cola para un controlador, incluidas las fases de protocolo de enlace y estado | ambos | `PacketProcessor::ScheduleIfPossible` y `HandleNow` (CallSite) |
| `ClientEvents.Tick` | `ClientTickArgs` | cada tic del bucle principal del cliente | cliente | `MinecraftClient::Tick` (Marca) |

Orden de los eventos del jugador: la muerte está anidada dentro del flujo de daño, por lo que `PlayerDeath` precede al `PlayerHurt` correspondiente; `PlayerLeave` y `PlayerDisconnect` son ​​dos cosas diferentes: la primera significa la eliminación de la lista en línea (incluso después de `/kick` solo se activa una vez que se interrumpe la conexión), la segunda significa que la conexión se interrumpe sola y no se garantiza que los dos aparezcan en pares.

### 2.2 Suscripción y baja

```csharp
IDisposable Subscribe(Action<T> handler)
```

- Al suscribirse al mismo evento varias veces se envían notificaciones en el orden de suscripción.

- El envío toma una instantánea de la lista de devolución de llamadas, por lo que suscribirse o cancelar el registro desde dentro de una devolución de llamada no afecta el envío actual.

- Sin darse de baja, permanece efectivo para siempre; Los mods no proporcionan ningún mecanismo de descarga, por lo que la cancelación del registro manual suele ser innecesaria.

### 2.3 Tipos de argumentos

**`ServerTickArgs`**

| Propiedad | Tipo | Notas |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | recuento de ticks desde este lanzamiento, a partir de 1 |

Tenga en cuenta que esto lo cuenta ModApi, no el `TickCount` del kernel.

**`ClientTickArgs`**

| Propiedad | Tipo | Notas |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | igual que el anterior, contado independientemente del lado del cliente |

**`ServerPhaseArgs`**

| Propiedad | Tipo | Notas |
| -------- | -------- | ---------------------------- |
| `Phase` | `string` | nombre de fase, `started` o `stopping` |

El campo duplica el evento en sí; se mantiene para que el registro pueda utilizar un formato uniforme.

**`CommandRegisterArgs`**

| Miembro | Tipo | Notas |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher` | `CommandDispatcher<CommandSourceStack>` | el despachador de comandos del kernel |
| `Register(name, description, build)` | método | registrar un comando y registrarlo en el libro mayor, ver 3.1 |

**`PlayerJoinArgs`**

| Propiedad | Tipo | Notas |
| ------------- | -------------- | ---------------------------------------------- |
| `Player` | `ServerPlayer` | el jugador que acaba de unirse; unirse a paquetes ya enviados, el estado es seguro de leer |
| `ProfileName` | `string` | nombre del jugador |

**`PlayerLeaveArgs`**

| Propiedad | Tipo | Notas |
| --------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | el jugador saliente, que ya no está en la lista online en este momento |
| `Removed` | `bool` | si realmente se eliminó; `false` sobre la eliminación repetida |

**`PlayerDisconnectArgs`**

| Propiedad | Tipo | Notas |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | el jugador desconectado |
| `Reason` | `string` | motivo de desconexión; texto plano cuando se proporciona como componente |

Se envió el paquete de desconexión y se cerró la conexión; Enviar paquetes a este jugador ahora no tiene ningún efecto.

**`PlayerHurtArgs`**

| Propiedad | Tipo | Notas |
| ---------- | --------------- | -------------------------------------------- |
| `Player` | `ServerPlayer` | el jugador que resultó herido |
| `Attacker` | `ServerPlayer?` | el jugador que causó el daño; `null` por daños ambientales y de mando |
| `Amount` | `float` | cantidad de daño esta vez |

No se activa durante fotogramas de invulnerabilidad o después de la muerte (el `Hurt` del kernel devuelve `false`).

**`PlayerDeathArgs`**

| Propiedad | Tipo | Notas |
| ---------- | --------------- | ------------------------------ |
| `Player` | `ServerPlayer` | el jugador que murió |
| `Attacker` | `ServerPlayer?` | el asesino; `null` cuando no hay ninguno |

El kernel se reinicia inmediatamente después de que la salud llega a cero, por lo que cuando se activa el evento, el jugador ya tiene la salud completa en el punto de reaparición; Las coordenadas y caídas en el momento de la muerte no están disponibles.

**`PlayerChatArgs`**

| Propiedad | Tipo | Notas |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | nombre del remitente |
| `Message` | `string` | mensaje de texto plano |

Este evento es una **notificación de solo lectura**: el método original ya transmitió el mensaje, por lo que cambiar `Message` aquí no tiene ningún efecto.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

Los tres eventos comparten la misma forma de argumentos:

| Propiedad | Tipo | Notas |
| -------- | ----- | ------------------ |
| `X` | `int` | coordenada del fragmento X |
| `Z` | `int` | coordenada del fragmento Z |

El más fácil de equivocarse es `ChunkSaved`: el kernel requiere esta devolución de llamada para **tomar la instantánea sincrónicamente**, mientras que la serialización y las escrituras en el disco las realiza el propio kernel de forma asincrónica. Por lo tanto, el trabajo que requiere mucho tiempo en esta devolución de llamada ralentiza directamente la descarga de fragmentos, y las operaciones destructivas (eliminación de bloques, cambio de inventarios) tampoco deberían realizarse aquí: solo promete el momento de la instantánea.

Cuando se activa `ChunkUnloaded`, las entidades del bloque ya se han limpiado junto con el fragmento; si desea leer bloques, use `ChunkSaved` (que tampoco puede acceder a los bloques) o un punto anterior.

**`CommandExecutedArgs`**

| Propiedad | Tipo | Notas |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string` | texto de comando sin formato; los comandos de chat no llevan barra diagonal inicial |
| `Result` | `int` | valor de retorno del comando; 0 significa fracaso o negación |
| `Source` | `CommandSourceStack?` | fuente de comando; `null` en la ruta de sobrecarga del reproductor |
| `Player` | `ServerPlayer?` | el jugador que dio la orden; `null` cuando se emite desde la consola |

El evento se activa **después** de que finaliza el comando; no puede cambiar la ejecución. Los errores de sintaxis y las denegaciones de permisos también aparecen aquí; usa `Result` para diferenciarlos. Los comandos que un jugador envía desde la barra de chat pasan por la sobrecarga `Execute(ServerPlayer, string)`, donde el kernel construye la fuente del comando internamente, por lo que en ese caso `Source` es `null` y solo se configura `Player`.

**`LevelTickArgs`**

| Propiedad | Tipo | Notas |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level` | `NcLevel` | el nivel avanzó este tick |
| `RunsNormally` | `bool` | si avanzó normalmente; `false` durante `/tick freeze` |

Se dispara una vez por nivel cargado por tick, por lo que un mundo de varios niveles recibe varios por tick. El punto de activación es después de que el tic de nivel **se haya completado**; es un punto de observación, no un punto de intercepción.

**`SavedDataSavingArgs`**

| Propiedad | Tipo | Notas |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage` | la tabla de datos guardados se conserva |

Esta es una ruta diferente de `ServerEvents.ChunkSaved`: el fragmento solo toma una instantánea y escribe de forma asincrónica, mientras que este se activa después de que se completa una escritura sincrónica. Por él pasan el reloj mundial, las reglas del juego y los datos de las fronteras mundiales.

**`BlockChangedArgs`**

| Propiedad | Tipo | Notas |
| -------- | ------------ | --------------------------- |
| `Pos` | `BlockPos` | posición del bloque modificado |
| `State` | `BlockState` | estado del bloque después del cambio |

El estado ya se escribió en el fragmento y está a punto de sincronizarse con los clientes, por lo que el cambio en sí no se puede modificar aquí. Los componentes de Redstone que cambian su propio estado en las devoluciones de llamadas de comportamiento también salen por esta ruta; es de alta frecuencia, así que no realice trabajos que requieran mucho tiempo en la devolución de llamada.

**`BlockBrokenArgs`**

| Propiedad | Tipo | Notas |
| -------- | --------------- | ------------------------------------------ |
| `Pos` | `BlockPos` | posición del bloque roto |
| `Player` | `ServerPlayer?` | el interruptor; `null` por causas que no son de jugadores como redstone |

Se activa sólo cuando el bloque realmente se reemplaza; las posiciones vacías y las pausas rechazadas no lo activan. Los efectos de ruptura y las caídas ya se han manejado, por lo que lo que lees en el evento es el resultado.

**`ItemDroppedArgs`**

| Propiedad | Tipo | Notas |
| -------- | ----------- | ----------------------- |
| `Pos` | `BlockPos` | donde apareció el artículo caído |
| `Stack` | `ItemStack` | la pila de elementos caídos |

Los productos para romper bloques y los productos para cocinar en fogatas pasan por aquí. Una pila de elementos vacía no genera ninguna entidad, por lo que no hay ningún evento.

**`PacketReceivedArgs`**

| Propiedad | Tipo | Notas |
| --------------- | -------- | ------------------------ |
| `Listener` | `object` | el oyente que recibe este paquete |
| `Packet` | `object` | el objeto del paquete en sí |
| `IsServerbound` | `bool` | si se trata de un paquete destinado al servidor |

Se activa para cada paquete entrante y cubre las cuatro fases: protocolo de enlace, estado, configuración y reproducción. Los paquetes de movimiento llegan varias veces por tick, por lo que no debe realizar un trabajo que requiera mucho tiempo en la devolución de llamada. El paquete ya está decodificado en un objeto pero no ha ingresado a la capa empresarial; para distinguir tipos, inspeccione `Packet` usted mismo. Los paquetes salientes están fuera del alcance de este evento.

***

## 3. Puntos de extensión

### 3.1 Registro de comandos

El momento es `ServerEvents.CommandRegister`. No almacene en caché los argumentos de este evento; el árbol de comandos interno se construye solo una vez al inicio.

```csharp
public void Register(
    string name,                                          //command literal, without the slash
    string description,                                   //one-line description, shown in the /ncmapi ledger
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attach arguments and the executor
```

El `build` que recibes es el constructor de brigada del núcleo; escriba argumentos, subcomandos y predicados de permisos a la manera del kernel:

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //permission predicate
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
                    player.Heal(20f);                     //illustrative
                return 1;
            })));
```

Puntos clave:

- Llamar a `args.Dispatcher.Register(...)` directamente también instala un comando, pero no ingresa al libro mayor y `/ncmapi` no lo mostrará. Utilice `args.Register` si desea que aparezca en la lista.

- Los comandos no tienen restricción de permisos de forma predeterminada; agregue `.Requires(...)` usted mismo si es necesario.

- El comportamiento en el momento de la ejecución depende totalmente de usted; ModApi no lo intercepta.

### 3.2 Ver comandos registrados

Hay un `/ncmapi` incorporado, que requiere nivel de permiso 2:

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

Las líneas de uso se calculan sobre la marcha desde la estructura de nodos del árbol de comandos: los literales se escriben por nombre, los argumentos se encierran entre corchetes angulares y los nodos intermedios que son ejecutables también obtienen su propia línea.

***

## 4. Fachadas del servidor

Todas las fachadas de este capítulo viven bajo `NetCraft.ModApi.Wrapper`; después de `using NetCraft.ModApi.Wrapper;` están disponibles.

Las fachadas son `Nc*` clases estáticas que reúnen capacidades dispersas por el núcleo en unos pocos puntos de entrada. La instancia del kernel es capturada por una sonda cuando se inicia el bucle principal; cuando `NcServer.IsAvailable` es falso, todo lo que se muestra a continuación arroja: utilícelos solo dentro de devoluciones de llamada de eventos, no desde `Init`.

| Fachada | Propósito |
| --- | --- |
| `NcServer` | instancia del servidor, tasa de ticks, comandos, seguimiento de entidades, datos del jugador, reglas del juego, transmisión, ejecución de comandos |
| `NcPlayers` | consultas y operaciones de jugadores en línea (patada, teletransporte, salud, modo de juego, permisos) |
| `NcWorld` | lectura/escritura y ruptura de bloques del mundo, clima, hora, frontera, reloj, sonidos, eventos de nivel; toma `NcLevel` controladores para alcanzar otras dimensiones, las coordenadas son simples `x y z` ints |
| `NcRegistries` | registros integrados buscados por nombre (bloques, elementos, fluidos, efectos, biomas, partículas, entidades, entidades de bloques) |
| `NcRecipes` | consultas de recetas (elaboración de cuadrículas, corte de piedra, cocina; buscar recetas por identificación) |
| `NcLists` | listas y configuración (lista blanca, operaciones, prohibiciones, `server.properties`) |
| `NcStartup` | argumentos de inicio (tokens no reconocidos por el kernel y suscripción basada en nombre) |

`NcPlayer` no es una fachada estática sino un **identificador de objeto**: `NcPlayers.All` / `Find` lo devuelve, y `Player` / `Attacker` en los eventos del jugador también lo son. Los identificadores son de solo lectura y están construidos mediante sondas; los mods no pueden obtener el `ServerPlayer` del kernel, el primer ancla de "no hay tipos de kernel en la superficie pública". El mismo reproductor del kernel siempre se asigna al mismo identificador, se almacena en caché internamente por una referencia débil y se invalida automáticamente una vez que el reproductor cierra la sesión.

`NcLevel` sigue la misma forma para los niveles. `NcWorld.Overworld` / `Nether` / `End` y `NcWorld.Get("minecraft:the_nether")` lo devuelven, y `LevelTickArgs.Level` es uno también. Lleva la identificación de la dimensión, el tiempo, el clima, la altura de construcción, el recuento de ticks y la carga de fuerza de los fragmentos; las operaciones de bloque permanecen en `NcWorld` y toman el mango más `x y z`. `BlockPos` nunca aparece, por lo que la dll de un mod no contiene ninguna referencia al tipo de nivel de kernel.

### 4.1 Registros

`NcRegistries` proporciona tablas completas y búsquedas por nombre. Las tablas completas son para iteración y búsqueda basada en etiquetas; las búsquedas por nombre sirven para obtener un solo elemento:

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //block default state
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//iterate the whole table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

Los registros se ensamblan gradualmente durante el inicio y las modificaciones se cargan antes de que se complete el ensamblaje, por lo que no almacene en caché nada de lo buscado en `Init`; el ensamblaje aún está en curso y un valor almacenado en caché será una referencia nula o un valor obsoleto. Actualmente `BuiltInRegistries.BootStrap` sigue siendo una implementación vacía; cada registro se completa por separado con su propio Bootstrap, y los basados ​​en datos (biomas, recetas, etc.) tienen muy pocas entradas antes de que se conecte la carga del paquete de datos.

### 4.2 Recetas

`NcRecipes` está respaldado por una tabla de recetas cargada desde paquetes de datos; `/reload` reemplaza toda la tabla, así que no mantengas un `RecipeHolder` entre recargas.

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

## 5. Internos

No necesitas esta sección para escribir mods, pero puede ayudarte al depurar.

### 5.1 Sondas

| Clase | Formulario | Responsabilidad |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)` | Marca ×4 | todas las señales de "algo sucedió" se canalizan en un método, enviado al evento coincidente mediante `label` |
| `Internal.CommandProbe.OnCommandsReady(object)` | Sitio de llamada | reemplaza la llamada a `EffectCommand::Register`; después de restaurar la llamada original, se activa `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)` | Sitio de llamada ×6 | eventos de jugador, un método por punto de gancho; luego de restaurar la convocatoria original publica el evento |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | Sitio de llamada ×3 | eventos de fragmentos, enganchados en los sitios de asignación de las tres propiedades de devolución de llamada de `ServerChunkCache`; se superpone un delegado contenedor antes de devolver el control al núcleo |
| `Internal.BlockProbe.OnXxx(...)` | Sitio de llamada ×3 | bloquear eventos; rotura y caída del gancho `ServerBlockUpdates`, cambios de estado enganchan el método de interfaz en `IBlockUpdateSink` |

La firma de `SignalProbe` toma solo `string`, y los parámetros de `CommandProbe`, `PlayerProbe` y `LevelProbe` se declaran como `object`; esto es deliberado: durante el ensamblaje `Lead.Hook` resuelve la firma del método de reemplazo, y una vez que aparece un tipo de kernel en él, al resolverlo, se extrae el ensamblaje del kernel antes de tiempo y la inyección pierde su ventana. Los tipos de kernel aparecen solo dentro de los cuerpos de los métodos, momento en el que el código ya se está ejecutando.

Lo único que no puede ser `object` en una firma son los parámetros de tipo valor y los valores de retorno: `object` es una referencia en la pila mientras que `float`/`bool` son ​​valores, y una falta de coincidencia es IL no válida. Entonces `PlayerProbe.OnHurtPlayer` mantiene `float` por el monto del daño, y `OnRemovePlayer` y `OnHurtPlayer` mantienen `bool` valores de retorno.

`BlockProbe` es una extensión de esta restricción: la posición y el estado del bloque son los dos tipos de valores `BlockPos`/`BlockState`, que solo se pueden escribir en la firma como ellos mismos. Estos dos tipos provienen de `NetCraft.Primitives` y `NetCraft.Registry`, ninguno de los cuales está en la lista de inyección, por lo que resolverlos durante el ensamblaje no elimina los ensamblajes que se reescribirán antes de tiempo.

### 5.2 Lista de puntos de gancho

`ncmod.json` de ModApi contiene veinticuatro reglas, que coinciden con la tabla en 2.1 uno a uno. Para cambiar un punto de enlace o agregar una regla, edite este archivo; después de editar, reconstruya (es un recurso integrado) y vuelva a colocar el dll resultante en `mods/`; esto último ya lo hace automáticamente `DeployModToHosts` en `NetCraft.ModApi.csproj`, y su falta se manifiesta como que las reglas no tienen efecto en absoluto.

`CommandManager::Execute` tiene dos sobrecargas que comparten una regla. CallSite de `Lead.Hook` coincide con los sitios de llamadas por "tipo + nombre de método", no por lista de parámetros, y ambas sobrecargas toman dos parámetros, por lo que la sonda puede tomar `object` como el primer parámetro y enviar por el tipo real.

Las dos reglas `PacketProcessor` son ​​complementarias: los paquetes de la fase de reproducción pasan por `ScheduleIfPossible` a la cola del hilo principal, mientras que las fases de protocolo de enlace y estado pasan por `HandleNow` para su manejo inmediato; cualquier paquete determinado pasa a través de solo uno de ellos. Al conectar solo el primero, se pierden las fases de protocolo de enlace y estado, que resultan ser las más fáciles de probar con scripts, por lo que durante la depuración esto se malinterpreta fácilmente como "la regla no tuvo efecto".

Las tres reglas de fragmentos enlazan los **sitios de asignación** de las tres propiedades de devolución de llamada de `ServerChunkCache`, no los sitios de lectura. La razón es que esas tres propiedades son de unidifusión y ya están ocupadas por el propio núcleo cuando se construye `PersistentServerLevel` (inyectan lógica de limpieza de entidad de bloqueo y guardado); una asignación directa de mod anularía la copia del kernel: las descargas no persisten, las entidades de bloque no se limpian y sin ningún error. Enganchar el sitio de asignación permite que la devolución de llamada del kernel y la sonda se encadenen en ese momento; la asignación ocurre solo una vez y cada activador posterior agrega una capa de reenvío de delegados.

`PlayerList::RespawnPlayer` es privado, por lo que la sonda no puede restaurar la llamada original; ese pasa por reflexión (llamada una vez por muerte, por lo que los gastos generales son insignificantes). Esto también deja una puerta abierta para la alineación del kernel: si se agrega `InternalsVisibleTo` en el futuro, se puede cambiar a una llamada directa.

### 5.3 Libro mayor

`Internal.NcCommandRegistry` graba comandos registrados a través de `args.Register`. Es sólo un libro de contabilidad y no participa en la ejecución de comandos; Los comandos en sí están instalados en el despachador del kernel, por lo que incluso si el libro mayor tiene problemas, los comandos siguen funcionando.

***

## 6. Para agregar

Los siguientes son puntos de enlace cuyas ubicaciones están confirmadas pero que aún no se han convertido en eventos (la lista fue producida por `__scan_mod_api.py` en la raíz del repositorio):

| Dirección | Puntos de gancho candidatos |
| ----------------- | ---------------------------------------------------------------------------- |
| Entidades | `ClientLevel::AddEntity`, `Entity::Die` |
| Mundo | carga y descarga de nivel, `ServerChunkCache` procesamiento por lotes de fragmentos |
| Generación de terreno | `ChunkGenerator::Generate` etapas por `ChunkStatus`, `WorldGenRegion::SetBlockState` |
| Ejecución de comandos | `CommandSourceStack::SendSuccess` / `SendFailure` (la mitad de respuesta, con muchos sitios de llamada) |
| Red | tipo por paquete `ServerGamePacketListenerImpl::HandleXxx` (actualmente solo un punto de entrada unificado) |

Instrucciones ya hechas: los ticks de nivel se convirtieron en `ServerEvents.LevelTick`, la persistencia de datos guardados se convirtió en `ServerEvents.SavedDataSaving`, la ejecución de comandos se convirtió en `ServerEvents.CommandExecuted` y los bloques se convirtieron en `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`.

Enganchar la celda de bloque en `SetBlock` no funciona: tiene dos parámetros predeterminados, `notifyNeighbors` y `strict`, por lo que los sitios de llamadas compilados toman de 4 a 6 parámetros, y dado que CallSite coincide por "tipo + nombre de método" sin mirar la lista de parámetros, un método de reemplazo no puede manejar las tres formas de pila. En su lugar, enlaza dos lugares: enlaces de sincronización de estado `IBlockUpdateSink::BlockChanged` (la única llamada de interfaz en `ServerLevel.SetBlock`, que cubre cada cambio con la sincronización del cliente) y métodos propios de enlace de interrupción y caída `ServerBlockUpdates`.

La celda de entidades es problemática porque `Entity` está definida en `NetCraft.Registry`, que no está en la lista de inyección, por lo que las llamadas a sitios que apuntan a ella no se pueden reescribir. `ClientLevel::AddEntity` está en el ensamblado del cliente y es factible; un evento de muerte primero requiere resolver si `Registry` se puede reescribir.

Para la celda de red, `HandleChat` existe desde hace mucho tiempo y `NetworkEvents.PacketReceived` proporciona un punto de entrada unificado, por lo que los ganchos por `HandleXxx` son ​​mucho menos valiosos; sólo vale la pena agregar escenarios que necesitan un filtrado detallado por tipo de paquete.

La carga y descarga de niveles no tienen punto de convergencia en el lado NC: `DedicatedServer::CreateLevel` es privado, por lo que la llamada original solo se puede restaurar mediante reflexión como `PlayerList::RespawnPlayer`; el camino de descarga está aún más disperso. Para hacerlo, primero establezca cuáles deberían ser los argumentos del evento.

Todos los restantes necesitan referencias a objetos (instancias de entidad, etc.), por lo que deben usar `CallSite` en lugar de `Mark`; Si aparece un tipo de valor del kernel entre los parámetros, solo se puede escribir en la firma del método de reemplazo como sí mismo.

También se revisó la lista de interfaz-superficie (`__modapi_api.txt`, producida por `__scan_mod_api.py --api`): las entradas calificadas como puntos de entrada de capacidad se reunieron en las fachadas del capítulo 4 por dominio, y el resto que no están expuestos se dividen en tres categorías: protocolo y manejo de paquetes (`Network.Protocol.*`), renderizado y modelos (`Client.Render.*`), y funciones de densidad y generación de terreno (`LevelGen.*`). Estos son elementos internos del núcleo; usarlos directamente vincularía las modificaciones con los detalles de implementación, por lo que primero se debe abrir una interfaz estable en el kernel.

En el lado del servidor hay dos cosas más que no están empaquetadas como fachadas: la instancia `ReloadableServerResources` cuelga de `DedicatedServer`, y dado que ModApi no hace referencia a `NetCraft.Server`, empaquetarla requiere primero abrir una propiedad en la clase base del kernel; `ChunkSender` y `ServerWorldBorderListener` son ​​flujos internos sin caso de uso para mods.
