# Guía de modding de NetCraft


Los detalles de cada API individual no están aquí; consulta [modapi-es.md](modapi-es.md).

---

## 1. Primero, la estructura en tiempo de ejecución

### 1.1 Tres puntos de entrada de proceso

NC tiene tres puntos de entrada, y el proceso de inyección de mods es el mismo para los tres:

| Entrada | Propósito |
| --- | --- |
| `NetCraft.Loader` | un solo exe para ambos lados: `--server` arranca el servidor; `--client` o sin indicador de modo arranca el cliente |
| `NetCraft.Server.Exe` | ejecutable de servidor independiente |
| `NetCraft.Client.Exe` | ejecutable de cliente independiente |

`Main` en sí es una capa fina que solo registra callbacks y pasa el trabajo al siguiente método. Tomemos `NetCraft.Server.Exe`:

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // registra el callback de resolución del ensamblado del kernel
    BootMods(args);                        // ejecuta el arranque de los mods
    return Launch(args);                   // solo ahora entra en la implementación de negocio
}
```

Esta separación no es una cuestión de estilo. Cuando el JIT compila un método, resuelve **todos los tipos** que aparecen en el cuerpo de ese método, y esto ocurre antes de que el método se ejecute. Si `Main` llamara directamente a `ServerMain.Run(args)`, `NetCraft.Server.dll` se arrastraría en el instante en que se compila el JIT de `Main`, antes de que se haya ejecutado el arranque de mods, y la ventana de reescritura desaparecería. Por eso tanto `BootMods` como `Launch` deben marcarse con `MethodImplOptions.NoInlining` — sin el marcador, el JIT los vuelve a insertar en `Main`, echando por tierra la división.

`NetCraft.Loader` tiene la misma estructura, salvo que su detección de modo y el arranque de mods están ambos en `Launch`, y `Main` conserva solo los dos pasos `Initialize` y `Launch`.

### 1.2 Ensamblados del kernel en el subdirectorio kernel/

El directorio de salida tras una compilación tiene este aspecto:

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- biblioteca principal, incrusta todas las sub-bibliotecas de nivel inferior
NetCraft.ModLoader.dll    <- el propio cargador
NetCraft.Server.Exe.dll   <- ensamblado de entrada
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... ensamblados restantes del kernel
mods/
  your-mod.dll
```

¿Por qué moverlos a `kernel/` en lugar de dejarlos en la raíz?

El host de .NET trata los ensamblados registrados en `deps.json` como TPA (Trusted Platform Assemblies). Para los ensamblados en el TPA, el runtime resuelve **por ruta** — los bytes pasados a `AssemblyLoadContext.LoadFromStream` simplemente se ignoran. Es decir, aunque introduzcamos bytes reescritos de antemano, el runtime sigue leyendo del disco la copia no reescrita. Solo eliminando los ensamblados del kernel de `deps.json` y moviendo los archivos lejos, el runtime vuelve a llamar a `AssemblyLoadContext.Resolving` cuando falla la resolución, dándonos la oportunidad de entregar los bytes reescritos.

Los tres tipos que quedan en la raíz no pueden moverse: la biblioteca principal (es el host de incrustación y debe iniciarse primero), el propio cargador (el código de arranque vive en él) y el ensamblado de entrada (el apphost parte de él).

**Coste**: no se puede inyectar en el propio ensamblado de entrada. Si tu objetivo de enganche resulta estar en el ensamblado `NetCraft.Server.Exe.dll`, es ineficaz. El código de negocio del kernel está todo bajo `kernel/`, así que normalmente esto no es un problema.

### 1.3 Secuencia de carga de mods

```
EmbeddedAssemblyLoader.Initialize()
  └─ instala el callback Resolving
BootMods → ModBootstrap.Run(lado actual)
  ├─ escanea estáticamente mods/*.dll (MetadataReader lee el ncmod.json incrustado, no se carga ningún ensamblado)
  ├─ filtra los mods cuyo entorno no coincide con el lado actual
  ├─ ensambla las reglas de inyección y entrega el reescritor a la biblioteca principal
  ├─ precarga los ensamblados objetivo: lee bytes → pasa por el reescritor → LoadFromStream
  └─ llama a Init() de la entrada de cada mod
Launch → ServerMain/ClientMain.Run(args)
  └─ el negocio del kernel empieza a ejecutarse; las sondas ya están dentro
```

Fíjate en el orden: **se escanean las declaraciones, se cargan los bytes reescritos y el código de entrada se ejecuta al final**. Cuando se ejecuta el `Init()` de un mod, los ensamblados del kernel ya se han reemplazado.

---

## 2. Diferencias clave con Fabric

| Dimensión | Fabric | NetCraft |
| --- | --- | --- |
| Lenguaje / runtime | Java / JVM | C# / .NET 10 (CoreCLR) |
| Portador del mod | jar que contiene `fabric.mod.json` | dll que incrusta `ncmod.json` |
| Lectura de declaraciones | leer un archivo dentro del jar | `MetadataReader` lee estáticamente los recursos incrustados sin cargar ensamblados |
| Inyección de código | Mixin (anotaciones en el código fuente; los miembros se mezclan en la clase objetivo al cargar la clase) | `Lead.Hook` (reglas declaradas en un manifiesto o en anotaciones; los bytes se reescriben in situ durante la resolución del ensamblado) |
| Granularidad de inyección | cualquier línea del cuerpo de un método, incluidas variables locales y valores de expresiones intermedias | trece formas (sitio de llamada, lectura/escritura de campo, constructor, comprobación de tipo, boxing, variable local, constante, reemplazo de todo el cuerpo del método, sondas, etc.), con inserción antes o después |
| Modelo de carga | Fabric Loader + cargador de clases Knot | único ALC por defecto + `AssemblyLoadContext.Resolving` |
| Alcance de la API oficial | Fabric API tiene muchísimos módulos | NetCraft-ModApi actualmente solo tiene puntos de extensión de eventos y comandos |

### 2.1 Estilos de inyección: anotaciones o manifiesto, elige uno

El Mixin de Fabric se anota **en el código fuente**:

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NC admite ambos estilos, pero sus requisitos previos difieren: la forma de anotación depende de `InjectAttribute` en `NetCraft.ModApi.Extension`, así que un mod que no lo referencie no puede usar anotaciones; la forma de manifiesto son datos puros escritos en `ncmod.json` y no requiere referencia alguna para las reglas de inyección.

**Anotación**, colocada en tu propio método de reemplazo:

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**Manifiesto**, escrito en `hooks` dentro de `ncmod.json`:

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

Durante el ensamblado las dos rutas se fusionan en una única tabla de reglas, y **si el mismo punto de inyección se escribe en ambas, gana la anotación**. Tras fusionarlas son indistinguibles; la diferencia está en los requisitos previos y en la ergonomía:

| | Anotación | Manifiesto |
| --- | --- | --- |
| Dónde se escribe | en el método de reemplazo | en `hooks` dentro de `ncmod.json` |
| Requisito previo | debe referenciar `NetCraft.ModApi` | ninguno, datos puros |
| Nombres de tipo | `typeof` / `nameof`, comprobados por el compilador | cadenas escritas a mano; las erratas solo se descubren durante el ensamblado |
| Qué puede llevar | solo reglas de inyección | identidad del mod (id, entry, environment, información de visualización) y reglas de inyección |

Así que `ncmod.json` debe escribirse tanto si usas anotaciones como si no; es la única fuente de identidad del mod. Las anotaciones solo hacen que las reglas sean menos propensas a errores al escribirlas. El manifiesto no tiene campo de dependencias; las relaciones de dependencia se infieren de las referencias de ensamblado (ver 6.1) y no necesitan declaración.

A la inversa, **un mod que no referencia `NetCraft.ModApi` solo puede usar el manifiesto** — esto afecta a más que a las reglas de inyección: eventos como `ServerEvents` y las fachadas `Nc*` también están en ModApi (bajo `NetCraft.ModApi.Wrapper`, ver [3.3](#33-un-ejemplo-lado-a-lado)), así que un mod que no puede usar anotaciones tampoco puede usarlos.

Las diferencias con Fabric permanecen:

- **Cómo y cuándo ocurren los cambios**: Mixin tiene un transformador que **mezcla los miembros de la clase mixin en** la clase objetivo en el momento de **cargar la clase**, así que lo que se carga es una nueva clase sintetizada y la original ya no existe; NC reescribe in situ las instrucciones del método objetivo **antes de que el ensamblado entre en memoria**, así que la clase sigue siendo la misma clase, solo cambia su cuerpo de método. Ambos reescriben en tiempo de carga, ninguno modifica el bytecode en tiempo de compilación — el procesador de anotaciones de Mixin solo genera un refmap (mapeo de ofuscación) y realiza validación en tiempo de compilación, mientras que NC no está ofuscado y no tiene esa capa en absoluto.
- Mixin puede inyectarse en **cualquier posición en medio del cuerpo de un método**; NC puede apuntar a un sitio de llamada, acceso a campo, construcción, lectura/escritura de variable local o constante específicos dentro de un método anfitrión dado, y puede insertar antes o después de él (`InType`/`InMethod` acotan el alcance, `Placement` decide insertar o reemplazar), pero **no puede alcanzar un número de línea arbitrario** ni cambiar el destino de un salto o el valor de una expresión intermedia en la pila.
- Los objetivos de Mixin usan un nombre de método de cadena más un descriptor; NC usa "nombre de tipo completo + nombre de método", así que todas las sobrecargas con el mismo nombre coinciden, y la precisión a una sola requiere `InType`/`InMethod`.

**Qué capa gestiona las anotaciones**: el tipo de anotación (`InjectAttribute`) lo proporciona `NetCraft.ModApi.Extension`, y lo resuelve `NetCraft.ModLoader` — al escanear mods lee estáticamente la tabla `CustomAttribute` con `MetadataReader`, sin cargar ensamblados. **`Lead.Hook` no reconoce anotaciones**; solo ve la tabla de reglas fusionada, y la capa de inyección nativa solo reconoce los bytes de descripción compilados en el lado administrado, sin siquiera leer `ncmod.json`.

Esto determina qué pueden expresar las anotaciones: lo que puedas escribir depende enteramente de qué campos tiene `InjectAttribute`. Actualmente hay siete — tipo objetivo, nombre de método, `HookType`, `Label`, `Environment`, `PatchMode`, `Ordinal` — y `InType`/`InMethod`/`Placement` de [2.4](#24-reducir-a-un-solo-sitio-alcance-del-host-y-colocación) y `LocalIndex`/`ConstantValue` de [2.5](#25-anclas-dentro-del-cuerpo-del-método-variables-locales-y-constantes) **no pueden escribirse en anotaciones**; usa la API de C# o espera a que el manifiesto se ponga al día. Al lado del manifiesto también le faltan estos — lo único que acepta más allá de las anotaciones es `ordinal`.

Para las trece formas de inyección, consulta el [apéndice de la guía de modding](#apéndice-resumen-de-hooktype) y [modapi-es.md](modapi-es.md).

### 2.2 Una restricción importante: las clases de sonda no deben llevar tipos del kernel en las firmas

La reescritura de NC ocurre **antes** de que se carguen los ensamblados del kernel. Al ensamblar reglas, `Lead.Hook` usa reflexión para encontrar tu método de reemplazo y construir una referencia de método, y este proceso resuelve cada tipo de parámetro y tipo de retorno de la firma.

Por lo tanto: **la firma de un método de reemplazo solo puede usar tipos BCL y `object`**. En cuanto aparece un tipo `NetCraft.*` en la firma, resolverlo arrastra demasiado pronto los ensamblados del kernel y la inyección falla directamente.

Cuando necesites un objeto del kernel, declara el parámetro como `object` y haz el cast dentro del cuerpo del método:

```csharp
//el ensamblado solo ve object; el cuerpo del método se compila con JIT después de que arranque el kernel
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite es reemplazo, no inserción

El método de reemplazo de una regla `CallSite` **reemplaza** la llamada original, así que el método original no se ejecuta. Para preservar el comportamiento original debes restaurarlo tú mismo en el método de reemplazo:

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              //restaura la llamada reemplazada
    ServerEvents.CommandRegister.Publish(...);  //luego añade la lógica propia del mod
}
```

Omite este paso y la funcionalidad original desaparece por completo.

Algunos detalles sobre cómo llevarlo a cabo:

- **`this` en un método de instancia también cuenta como parámetro**. Que el método objetivo sea un método de instancia determina si el método de reemplazo necesita un parámetro inicial extra. En IL tanto `call` como `callvirt` cuentan como llamadas de instancia — para un método no virtual en un tipo `sealed` el compilador emite `call`.
- **Una clave de regla cubre todas las sobrecargas bajo ese nombre de clase**; las sobrecargas con el mismo número de parámetros comparten un único método de reemplazo. `Disconnect(string)` y `Disconnect(Component)` se comparten así, y el método de reemplazo despacha por el tipo real del argumento.
- **Un método privado no puede llamarse desde fuera**, así que el método de reemplazo no puede restaurar la llamada original. O renuncias a este punto de enganche o lo llamas una vez mediante reflexión (aceptable a baja frecuencia).
- **Los parámetros y valores de retorno de tipo valor no pueden declararse como `object`**, `object` es una referencia en la pila mientras que `float`/`bool` son valores, y una falta de coincidencia es IL inválido. Mantén estas dos posiciones con sus tipos reales.
- **Los callbacks unicast no pueden asignarse directamente**. Algunas propiedades de callback del kernel (p. ej., los tres callbacks de chunk en `ServerChunkCache`) son `Action<T>` en lugar de `event`, y el kernel ya las ocupa. Un mod que asigne directamente sobrescribe la copia del kernel sin ningún error. El enfoque correcto es enganchar el setter de la propiedad y, en el momento de la asignación, encadenar tu lógica y el callback del kernel en un único delegado envoltorio.

### 2.4 Reducir a un solo sitio: alcance del host y colocación

El alcance por defecto de una regla a nivel de instrucción es **todo el ensamblado** — coincide cada lugar que llama al método objetivo o lee/escribe el campo objetivo. Para reducirlo a un solo sitio, usa dos parámetros opcionales:

| Parámetro | Efecto |
| --- | --- |
| `InType` / `InMethod` | hace coincidir las anclas solo dentro del cuerpo del método anfitrión especificado; ambos vacíos significa sin restricción |
| `Placement` | `Replace` reemplaza el ancla (por defecto); `Before` / `After` conservan el ancla e insertan una llamada antes o después de ella |
| `Ordinal` | cuando la misma ancla coincide en varios lugares del método anfitrión, cuál elegir, basado en 0. Omitido significa que se modifica cada lugar |

```csharp
//ejemplo: instrumentar solo cuando LevelChunk lee el estado del bloque; PalettedContainer::Get en otros lugares queda intacto
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

Los dos modos imponen requisitos distintos a la firma del callback:

- **El modo Replace** se alinea con los argumentos de la llamada reemplazada (incluido `this` para llamadas de instancia); que el callback restaure la llamada original depende de ti.
- **El modo Insert** pasa los **parámetros del método anfitrión** (incluido `this`), de forma consistente con la convención de `MethodBody`. La inserción no altera la pila que el ancla ya ha construido; la llamada original se ejecuta como de costumbre, solo que con un callback extra antes o después.

Algunos límites:

- `InType` e `InMethod` son independientes; puedes especificar solo uno. Ambos vacíos equivale a sin alcance.
- Varias reglas pueden enganchar la misma ancla, cada una acotada a un anfitrión diferente; **gana la primera coincidencia de anfitrión**.
- `Placement` solo se aplica a las formas a nivel de instrucción (`CallSite`, `NewObj`, lectura/escritura de campo, `TypeCheck`, `Box`, `FunctionPointer`, y las tres clases de [2.5](#25-anclas-dentro-del-cuerpo-del-método-variables-locales-y-constantes)); `MethodBody` siempre reemplaza todo.
- `Ordinal` cuenta el **orden de las coincidencias**, independientemente de si ese sitio acaba modificándose; si la regla no aparece suficientes veces en el método anfitrión, la regla no aterriza. La misma idea que `@At(ordinal)` de Mixin.
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` actualmente solo están disponibles en la API de C#; ni `ncmod.json` ni `[Inject]` los admiten (el manifiesto acepta `ordinal`), así que los mods basados en manifiesto no pueden usar los primeros.

### 2.5 Anclas dentro del cuerpo del método: variables locales y constantes

Las clases anteriores se anclan a una **entidad referenciada** (un método, campo o constructor), mientras que `LocalRead` / `LocalWrite` / `Constant` se anclan a **una posición dentro del cuerpo del método anfitrión**, correspondiendo a `@ModifyVariable` y `@ModifyConstant` de Mixin. Para estas tres, `OriginalType` / `OriginalMethod` nombran el **método anfitrión**, no una entidad referenciada.

| Forma | Parámetro extra | Posiciones seleccionadas |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` | cada lectura de esa ranura (basado en 0) |
| `LocalWrite` | `LocalIndex` | cada escritura a esa ranura |
| `Constant` | `ConstantValue` | cada carga de esa constante, comparada por tipo boxeado |

```csharp
//ejemplo: insertar un callback antes de la escritura a la ranura 0 en G
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//ejemplo: reemplazar la constante 5 en G por el valor de retorno de OnConst()
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

En el modo replace la firma del callback se alinea con el **efecto de pila de la instrucción**, no con los parámetros del anfitrión:

| Ancla | Efecto de pila | Firma del método de reemplazo |
| --- | --- | --- |
| `LocalRead` | empuja un valor | cero parámetros, devuelve ese valor |
| `LocalWrite` | saca un valor | un parámetro |
| `Constant` | empuja un valor | cero parámetros, devuelve ese valor |

`ConstantValue` se compara por tipo boxeado, así que `5` (int) y `5L` (long) son dos anclas diferentes; para coincidir con `ldc.i8` debes pasar `long`.

Una ranura es el índice compilado de la variable local; el mismo código fuente puede cambiarlo con una versión distinta del compilador, así que no lo trates como un identificador estable al portar entre versiones.

### 2.6 Inyección en tiempo de ejecución: modificar código ya en ejecución

La inyección tratada hasta ahora ocurre toda **antes de la carga del ensamblado** — los bytes se reescriben primero y luego se entregan al runtime. La premisa es que el ensamblado objetivo aún no se ha cargado.

`Lead.Hook` tiene otra ruta: usar la interfaz de Profiler del CLR (ReJIT) para modificar código que **ya está cargado, o cuyos métodos ya se han ejecutado**. Ambas comparten el mismo `HookRule`, y los parámetros de [2.4](#24-reducir-a-un-solo-sitio-alcance-del-host-y-colocación) y [2.5](#25-anclas-dentro-del-cuerpo-del-método-variables-locales-y-constantes) siguen disponibles:

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//una llamada y la inyección está hecha; sin reiniciar y sin cambios en archivos
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| | Reescritura en tiempo de carga | Inyección en tiempo de ejecución |
| --- | --- | --- |
| `patchMode` del manifiesto | `ILRewrite` (por defecto) | `RuntimeInject` |
| Momento | antes de que el ensamblado entre en memoria | cualquier momento después de que el proceso haya arrancado |
| Base | reescritura de bytes con Mono.Cecil | CLR Profiler ReJIT |
| Requisito previo | el objetivo no se ha cargado | el objetivo ya está en el proceso |
| Modificar código ya compilado con JIT | no es posible | posible |

**Por qué se comparten las reglas**: Cecil sigue haciendo aquí la reescritura, pero el resultado no se escribe en disco; en su lugar se compila en una descripción que se entrega a la capa nativa, la cual envía el nuevo cuerpo del método al CLR en tiempo de ejecución, dejando el resto de la gestión de versiones al CLR.

**Cómo lo hace un mod**: añade una entrada `patchMode` al elemento de regla en `ncmod.json`; la anotación `[Inject]` tiene un parámetro del mismo nombre.

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**Qué hace el ensamblado**: estas reglas no pasan por la ruta de reescritura en tiempo de carga (`ModHooks.Rewrite` aplica solo `ILRewrite`); en tiempo de ensamblado se registra una tabla separada de objetivos en tiempo de ejecución. Después de precargar los ensamblados del kernel y antes del `Init()` de los mods, el cargador toma cada una de sus instancias **ya cargadas** y envía los cuerpos de método reescritos al CLR. Si un objetivo no está cargado en ese momento se omite con una advertencia, y no se cargará antes en su nombre — la restricción de [2.2](#22-una-restricción-importante-las-clases-de-sonda-no-deben-llevar-tipos-del-kernel-en-las-firmas) de que "arrastrar el kernel demasiado pronto pierde la ventana" se invierte de dirección aquí, con la misma conclusión: si no está presente, no se puede hacer.

**Para enganchar la capa de inyección nativa**: el interruptor de ReJIT solo puede activarse mediante variables de entorno al arrancar el proceso (`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`); activarlas después del arranque no tiene efecto. Cuando el cargador detecta reglas `RuntimeInject` al principio del arranque y el proceso aún no está enganchado, **reinicia el proceso con la misma línea de comandos** llevando estas tres variables (`NC_PROFILER_ATTACHED=1` protege contra reinicios repetidos cuando "está enganchado pero no surte efecto"). La biblioteca nativa `lead_hook_native` debe colocarse en la raíz del programa, o apuntarse en otro lugar mediante `NC_PROFILER_PATH`; si no está ninguna de las dos, todo el lote de reglas se degrada a una única advertencia y no se bloquea el arranque.

**No lo mezcles con la reescritura en tiempo de carga sobre el mismo objetivo**: un cuerpo de método enviado por inyección en tiempo de ejecución se deriva de **bytes originales** y no incluye los cambios que la reescritura en tiempo de carga hizo al mismo método — si el mismo método es alcanzado por ambos tipos de regla, la versión de tiempo de carga se sobrescribe por completo. El ensamblado no puede saber si dos reglas alcanzan el mismo método, así que solo puede hacer un juicio tosco por ensamblado objetivo y registrar una advertencia.

**Las anotaciones no participan directamente en esta ruta**: la capa de inyección nativa no reconoce `InjectAttribute` ni lee `ncmod.json` — solo reconoce bytes de descripción. Las anotaciones y el manifiesto son cosas de **tiempo de ensamblado** (ver [2.1](#21-estilos-de-inyección-anotaciones-o-manifiesto-elige-uno)), analizadas por `NetCraft.ModLoader` en `HookRule` y luego entregadas a `RuntimeInjector`; el lado del mod no necesita tratarlas de forma diferente.

**Limitaciones** (más estrechas que la reescritura en tiempo de carga): no se admiten métodos con tablas de manejo de excepciones, no puede cambiarse la tabla de variables locales, no se admiten tipos y métodos genéricos, y los operandos solo reconocen referencias de método (las referencias de campo, las constantes de cadena y los tokens de tipo lanzan `NotSupportedException`).

**Rendimiento**: la inyección ocurre solo en el registro; después el método es código JIT ordinario con la misma sobrecarga de llamada que sin inyección. Adjuntar el profiler tiene un coste único — habilitar ReJIT requiere deshabilitar al mismo tiempo las imágenes ReadyToRun, y el arranque del proceso medido es unos 80–110 ms más lento; el cómputo en estado estable no muestra diferencia. Para un servidor como NC cuyo arranque ya se mide en segundos, esto es insignificante.

### 2.7 ¿Dos mods que modifican la misma clase entran en conflicto como en Mixin?

Primero, por qué las cosas entran en conflicto en el lado de Mixin. Mixin **mezcla miembros en la clase objetivo** y los aplica al cargar la clase: cuando varios mixins se mezclan en la misma clase, casos como inyectar repetidamente en el mismo lugar o añadir miembros con el mismo nombre a la misma clase lanzan `MixinApplyError`, y el fail-hard por defecto **mata el juego directamente**; además esta detección ocurre en el instante de cargar la clase, cuando el juego puede estar ya a medio ejecutar.

El modelo de NC es diferente, y la superficie donde pueden ocurrir conflictos es mucho menor:

| | Mixin | NetCraft |
| --- | --- | --- |
| Método de aterrizaje | mezclar miembros en la clase objetivo + reescribir bytecode | reescribir solo instrucciones IL; sin síntesis de tipos, sin añadir miembros |
| Conflictos estructurales (miembros con el mismo nombre, conflictos de herencia) | sí | ninguno |
| Cuándo se validan las reglas | al cargar la clase | en tiempo de ensamblado, leyendo estáticamente metadatos |
| Dos reglas que alcanzan el mismo lugar | lanza excepción | el primero que llega se queda con él; el segundo falla silenciosamente |
| Falla un mod | puede arrastrar toda la carga | afecta solo a sí mismo |

**Validación estática**: las reglas no se construyen cargando ensamblados y reflectando sobre tipos, sino leyendo tablas de metadatos PE. Así que problemas como "el tipo objetivo no está en ningún ensamblado conocido" o "la forma de inyección está mal escrita" se registran y se omiten **al principio del arranque**, sin esperar a que se cargue una clase antes de explotar.

**Aislamiento de fallos**: cuando las reglas de un mod no se pueden analizar, la clase de reemplazo no se carga, o el `Init()` de la entrada lanza excepción, solo **ese mod** se marca como fallido (estado `Error`, se muestra como "load failed" en la página MODS) y los demás mods se cargan como de costumbre. Aquí hay que corregir una formulación: NC **no tiene descarga en tiempo de ejecución** — los mods se cargan una vez, y `ModManager` explícitamente no ofrece carga/descarga dinámica. La llamada "descarga automática al fallar" es en realidad **aislamiento en tiempo de carga**: un mod fallido no se inicializa, pero tampoco se "descarga".

**Inyección dinámica**: en la ruta de ReJIT de [2.6](#26-inyección-en-tiempo-de-ejecución-modificar-código-ya-en-ejecución), la semántica de varios mods compitiendo por el mismo método coincide con el caso estático — gana el primero registrado, y las solicitudes posteriores se envían pero no pueden reclamarlo (`GetReJITParameters` reclama por "módulo + método", tomando el primero).

En esta ruta tropezamos una vez con un escollo, que merece la pena registrar: en una implementación temprana, el **parámetro de ámbito de resolución de `FindTypeRef` se pasaba como `mdTokenNil`**, cuya semántica es "coincidir solo con TypeRefs sin ámbito de resolución" — nuestras referencias cuelgan todas de `AssemblyRef`, así que la que habíamos construido nunca se encontraba. Se manifestaba así: la primera inyección tenía éxito, pero en la segunda inyección fallaba la resolución de referencias, no se podía construir ningún cuerpo de método nuevo, el CLR recurría al IL original, y **la primera inyección se perdía junto con él** (el método objetivo volvía a su comportamiento no inyectado). Tras la corrección ya no se reprodujo, pero la limitación permanece: **la inyección de referencias en metadatos debe completarse dentro de la ventana justo después de que se cargue el módulo objetivo; cuanto más tarde, más probable es que falle**.

**El coste debe declararse claramente**: el comportamiento de NC de no colapsar tiene el coste de que los conflictos se pasan por alto fácilmente — Mixin al menos interrumpe la carga, mientras que NC deja que el segundo falle silenciosamente. Para abordar esto, el ensamblado realiza una **comprobación de conflictos de la misma ancla**: cuando varios mods declaran el mismo punto de inyección, el que se ensambla después se registra en `ModHooks.Warnings` y se informa como advertencia en el registro de arranque (sin bloquear la carga):

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

La clave de detección es "tipo objetivo + método + forma de inyección + modo de patch". **El alcance del host no se distingue** — ni el manifiesto ni las anotaciones pueden escribir `InType`/`InMethod`, así que las reglas que provienen de mods son naturalmente de alcance de todo el ensamblado, y la misma clave significa una colisión. Las reglas añadidas directamente mediante la API de C# se saltan esta comprobación, ya que en ese caso dos reglas pueden alcanzar cada una un anfitrión diferente y son inherentemente no conflictivas.

**La inyección en tiempo de ejecución también pasa por esta comprobación**: entra por el mismo punto de entrada de ensamblado, y el modo de patch en la clave de detección la mantiene separada de la reescritura en tiempo de carga; cuando ambos tipos recaen en el mismo ensamblado objetivo hay un aviso de sobrescritura aparte (ver [6.4](#64-dos-mods-inyectando-el-mismo-objetivo)).

### 2.8 Mixins: añadir miembros a un tipo objetivo

Las secciones anteriores modifican todas instrucciones de código existente y no pueden crear nada nuevo. Para **añadir campos, métodos o interfaces** a un tipo objetivo, usa un mixin.

La relación con la sintaxis de Mixin es la siguiente:

| Mixin | NC |
| --- | --- |
| `@Mixin(X.class)` en la clase mixin | `[Mixin(typeof(X))]` en la clase fuente |
| los miembros de la clase mixin se mezclan en la clase objetivo | los campos y métodos de la clase fuente se mueven al tipo objetivo |
| `@Unique` añade un campo privado | escribe un campo ordinario en la clase fuente; se mueve del mismo modo |
| `@Shadow` referencia un miembro existente de la clase objetivo | no es necesario; escribe directamente los miembros de `X` y luego engánchalos |
| `@Implements` / `implements` | `Interfaces` |

También puede escribirse en el manifiesto:

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
    //tras ser movido, este es un campo de instancia en el tipo objetivo
    public int MyCounter = 5;

    //un método mezclado; lee/escribe el campo que se movió junto con él
    public int Bump() => MyCounter + 1;

    //la implementación que requiere la interfaz; tras ser movido, el tipo objetivo implementa ITagged
    public string Describe() => $"tagged:{MyCounter}";
}
```

**Movimiento, no copia**. Estos miembros se eliminan de la clase fuente, dejando solo una cáscara vacía — igual que la clase mixin de Mixin, **el código del mod ya no debería usar esa clase** (`new SomeEntityMixin()` o llamar a sus métodos no encontrará los miembros).

Algunas reglas de aterrizaje:

- **Solo se aplica a la reescritura en tiempo de carga**. El tipo objetivo debe estar en un ensamblado del kernel o en un ensamblado de mod que aún no haya entrado en memoria. La inyección en tiempo de ejecución solo envía cuerpos de método y no puede cambiar la disposición de tipos, así que esta forma no puede existir en ella.
- **Los inicializadores del constructor vienen con ello**. Los valores escritos en los inicializadores de campo de la clase fuente se fusionan en cada constructor de instancia del tipo objetivo; los inicializadores de campo estático se fusionan en el constructor estático (se crea si el objetivo no tiene ninguno). La llamada de encadenamiento a la clase base dentro del constructor de la clase fuente se elimina, así que el constructor base no se ejecuta dos veces.
- **Las reglas con interfaces marcan como virtuales los métodos de instancia públicos movidos**. El despacho de interfaces solo reconoce la vtable, y sin el marcador el CLR determinaría que la interfaz no está implementada y no la cargaría directamente. Así que no esperes que esos métodos sigan siendo no virtuales al mezclar una interfaz.
- **Los miembros con el mismo nombre se omiten**. Cuando el tipo objetivo ya tiene un campo o método con el mismo nombre, ese elemento no se mueve y el resto procede como de costumbre. Cuando dos mods se mezclan en el mismo tipo objetivo, ambos aterrizan, y solo se omite la parte del segundo que colisiona por nombre — más suave que el comportamiento de "el segundo falla por completo" de los hooks en [2.7](#27-dos-mods-que-modifican-la-misma-clase-entran-en-conflicto-como-en-mixin).
- **Los tipos anidados no se mueven**, y los tipos anidados o métodos genéricos en la clase fuente también quedan actualmente fuera de la cobertura de esta ruta.

El tipo fuente debe estar en **el propio ensamblado del mod**, así que ni el manifiesto ni la anotación escriben un nombre de ensamblado.

### 2.9 Dos rutas: capa wrapper y puntos de extensión

La superficie pública de `NetCraft.ModApi` se divide en dos espacios de nombres, correspondientes a dos usos:

| Espacio de nombres | Contenido | Lo que obtienes |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`, `NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup`, y manejadores de objeto como `NcPlayer` / `NcLevel` | tipos wrapper; sin tipos del kernel en la superficie pública |
| `NetCraft.ModApi.Extension` | anotaciones `[Inject]` / `[Mixin]` | las reglas se enlazan a nombres de clases y métodos del kernel |

Las dos son rutas **paralelas**, no una superpuesta a la otra:

- **Para estabilidad, usa `Wrapper`**. Las fachadas gestionan por ti el tedioso orden de llamadas del kernel (escribir un solo bloque tocando el nivel, la lista de jugadores y la cadena de sincronización es un ejemplo), y los args de los eventos son todos tipos wrapper. El coste es que las capacidades que las fachadas no exponen no están disponibles para ti.
- **Para completitud, usa `Extension`**. Las reglas de inyección modifican directamente clases y métodos del kernel, pero los nombres objetivo que escribes son los nombres del kernel, así que cuando el kernel cambia las reglas deben cambiar con él.

Puedes referenciar ambas. La línea `Wrapper` se está consolidando hacia "sin tipos del kernel en la superficie pública"; las partes de jugador y nivel están hechas — `Player` / `Attacker` en los eventos de jugador y los parámetros de entrada/salida de `NcPlayers` son manejadores `NcPlayer`, `NcWorld.Overworld` / `Nether` / `End` / `Get` y `LevelTickArgs.Level` son manejadores `NcLevel`, y las coordenadas de bloque son enteros `x y z` simples; las entidades y los tipos valor restantes (`BlockPos` / `BlockState` / `Vec3`) aún no están envueltos.

Hay que señalar un límite más: **la capa wrapper no protege contra la inyección**. Las reglas de `hooks` o las anotaciones `[Inject]` que escribas siguen enlazándose a nombres de clases y métodos del kernel, y se rompen igual cuando el kernel cambia.

---

## 3. Migrar desde Fabric

### 3.1 Correspondencia de conceptos

| Fabric | NetCraft |
| --- | --- |
| `fabric.mod.json` | el `ncmod.json` incrustado en el dll |
| `ModInitializer.onInitialize()` | el `public Task Init()` de la clase de entrada |
| `@Inject` / `@Redirect` | la anotación `[Inject]`, o reglas como `Mark` / `Probe` / `CallSite` en `hooks` |
| `@ModifyVariable` | `LocalRead` / `LocalWrite`, ver [2.5](#25-anclas-dentro-del-cuerpo-del-método-variables-locales-y-constantes); no escribible como anotación |
| `@ModifyConstant` | `Constant`, ver [2.5](#25-anclas-dentro-del-cuerpo-del-método-variables-locales-y-constantes); no escribible como anotación |
| `@Accessor` | aún no hay equivalente (los miembros `private` no necesitan ampliación de visibilidad; basta con escribir una regla) |
| `Registry.register(...)` | el registro del kernel (`BuiltInRegistries`) |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` | aún no hay equivalente (`ModManager` no está abierto a los mods) |
| `@Mixin` / `@Unique` / `@Implements` | la anotación `[Mixin]` o los `mixins` del manifiesto, ver [2.8](#28-mixins-añadir-miembros-a-un-tipo-objetivo) |

### 3.2 Qué no se traslada

- **El sistema de anotaciones de Mixin**: NC tiene dos anotaciones, `[Inject]` y `[Mixin]`, que son solo **estilos de declaración**, equivalentes a `hooks` / `mixins` en `ncmod.json` y fusionadas al ensamblar (las resuelve el cargador, no `Lead.Hook`, ver [2.1](#21-estilos-de-inyección-anotaciones-o-manifiesto-elige-uno)). La reescritura de instrucciones aterriza en tiempo de carga por defecto, y puede cambiarse a envío en tiempo de ejecución según [2.6](#26-inyección-en-tiempo-de-ejecución-modificar-código-ya-en-ejecución); añadir miembros e interfaces pasa por los mixins de [2.8](#28-mixins-añadir-miembros-a-un-tipo-objetivo). El direccionamiento no tiene una sintaxis de cadenas tipo `@At`, pero formas como `CallSite`/`FieldRead`/`LocalWrite`/`Constant`, junto con `Ordinal`, `InType`/`InMethod` y `Placement`, pueden cubrir los usos de `HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE` y `shift`; lo que falta es `JUMP`.
- **AccessWidener**: ninguno. La visibilidad no es un obstáculo para la reescritura de IL en NC; los métodos `private` pueden engancharse igual (el reescritor trabaja a nivel de bytes).
- **Yarn / Mojang mappings**: no es necesario. NC es código C# traducido directamente desde vanilla, con tipos y nombres de método correspondientes a vanilla, solo con un estilo de nomenclatura que sigue a C#.
- **La gran mayoría de los módulos de Fabric API**: solo están disponibles las capacidades cubiertas por `NetCraft-ModApi`; para el resto, escribe tus propias reglas de inyección o espera a que la API se ponga al día.

### 3.3 Un ejemplo lado a lado

Fabric: registrar una línea de log cuando arranca el servidor y registrar un comando.

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

NC:

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

Manifiesto (`ncmod.json`, como recurso incrustado):

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

Nota: incluso un `hooks` vacío funciona aquí — eventos como `ServerEvents.Started` los proporcionan las propias sondas de `NetCraft-ModApi`, y tu mod solo necesita suscribirse (`ServerEvents` está bajo `NetCraft.ModApi.Wrapper`, ver [2.9](#29-dos-rutas-capa-wrapper-y-puntos-de-extensión)). Solo necesitas escribir tus propias reglas de enganche cuando quieras enganchar un lugar del kernel donde ModApi aún no proporciona un evento.

---

## 4. Requisitos básicos de un ncm

ncm significa mod de NetCraft. Un ncm es un dll de biblioteca de clases de .NET que incrusta un `ncmod.json`, colocado en el directorio `mods/`.

La forma más fácil de empezar es la plantilla:

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

Si el paquete local aún no se ha publicado, usa `dotnet new install <nupkg path>`, o ejecuta `dotnet pack` en el repositorio y luego instala la salida.

`-e` toma `both` (por defecto) / `server` / `client`, determinando el `environment` del manifiesto y qué código de suscripción del lado correspondiente se genera en la clase de entrada. La plantilla incluye los ensamblados de referencia de NC, así que no se necesita una referencia de proyecto, y el id / entry de `ncmod.json` se rellenan a partir del nombre del proyecto.

A partir de 4.1, lo siguiente cubre lo que debe cumplir un proyecto escrito a mano.

### 4.1 Archivo de proyecto

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private debe desactivarse, si no los ensamblados de NC se incrustan en el dll del mod mediante EmbedDependencies en 4.6 -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName` debe ser `ncmod.json`; el escáner solo reconoce ese nombre.

La plantilla toma una ruta diferente: referencias de ensamblado bajo `libs/` (`Reference Include="libs\*.dll" Private="false"`). Síguela cuando no tengas el código fuente de NC. Lo que ambos enfoques comparten es que **los propios dll de NC nunca deben entrar en el directorio de salida** — `EmbedDependencies` de [4.6](#46-dependencias-de-terceros) incrusta los dll de terceros del directorio de salida en el mod, y si los ensamblados de NC también se incrustan habrá dos conjuntos de identidad de tipos.

### 4.2 Campos de ncmod.json

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

| Campo | Obligatorio | Notas |
| --- | --- | --- |
| `id` | Sí | identificador del mod, usado por las dependencias y las búsquedas. Una cadena vacía es omitida por el escáner |
| `version` | Recomendado | número de versión; cuando otros mods dependen de él, las restricciones de versión se juzgan por él, ver abajo |
| `name` | No | nombre para mostrar; esto es lo que muestra la página del mod, y por defecto vuelve a `id` |
| `description` | No | descripción de una línea |
| `authors` / `contributors` | No | créditos, un array de cadenas |
| `license` | No | identificador de licencia |
| `contact` | No | enlaces externos; puede tomar `homepage` / `sources` / `issues` |
| `icon` | No | el nombre del recurso incrustado del icono, ver 4.7 |
| `environment` | No | `both` / `client` / `server`, por defecto `both`. Cuando no coincide con el lado actual no se carga el mod entero |
| `entry` | Sí | nombre completo de la clase de entrada; la clase debe tener `public Task Init()` |
| `depends` | No | otros mods de los que depende y las versiones requeridas, ver abajo |
| `hooks` | No | lista de reglas de inyección; un array vacío significa suscribirse solo a los eventos que ModApi ya tiene |
| `mixins` | No | lista de reglas de mixin; mueve miembros de una de las clases de este mod al tipo objetivo, ver [2.8](#28-mixins-añadir-miembros-a-un-tipo-objetivo) |

`depends` declara otros mods de los que depende y las versiones requeridas; la clave es un id de mod y el valor una restricción de versión:

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**Las dependencias no escritas en `depends` no conllevan restricción de versión**. Las dependencias entre mods ya se infieren automáticamente de las referencias en tiempo de compilación (ver [6.1](#61-dependencia-entre-mods--inyectar-el-mod-del-que-se-depende)), y `depends` solo añade restricciones de versión por encima. Cuando el mod del que se depende no está en el lado actual se omite la comprobación — ese caso se deja que lo informe la resolución del ensamblado.

| Sintaxis de restricción | Significado |
| --- | --- |
| `*` | cualquier versión, equivalente a omitir la entrada |
| `1.2.3` | prefijo de segmento; `1.2` coincide con `1.2` y `1.2.9` pero no con `1.3` |
| `^1.2.3` | misma versión mayor y no menor que la base; cuando la versión mayor es 0 se usa la menor en su lugar, así que `0.1` y `0.2` cuentan como incompatibles |
| `>=1.2.3` | no menor que la base |

Los números de versión toman solo los dígitos iniciales de cada segmento, así que `26.2-netcraft` participa como `26.2`. Cuando las versiones no coinciden, **solo se omite el mod que lo declara** y el resto se carga como de costumbre; el registro de arranque indica "dependency X requires version …, actual version …".

La base del juicio es el campo `version` del manifiesto del mod del que se depende. Así que **un mod destinado a ser dependido debe establecer `version` correctamente** — un número de versión vacío no satisface ninguna restricción concreta.

### 4.3 Campos de una regla de hook

| Campo | Notas |
| --- | --- |
| `target` | nombre completo del tipo objetivo; debe estar en un ensamblado del kernel o en uno de los ensamblados de mod bajo `mods/` |
| `method` | nombre del método objetivo; todas las sobrecargas con el mismo nombre coinciden |
| `type` | forma de inyección, ver el apéndice |
| `patchMode` | método de aterrizaje, `ILRewrite` (por defecto) o `RuntimeInject`, ver [2.6](#26-inyección-en-tiempo-de-ejecución-modificar-código-ya-en-ejecución) |
| `ordinal` | cuando la misma ancla coincide en varios lugares del método anfitrión, cuál elegir, basado en 0, ver [2.4](#24-reducir-a-un-solo-sitio-alcance-del-host-y-colocación) |
| `replaceType` | nombre completo de la clase que contiene el método de reemplazo |
| `replaceMethod` | nombre del método de reemplazo |
| `label` | etiqueta de la sonda, usada solo por `Mark` y `Probe` |
| `environment` | el lado al que se aplica la regla, por defecto `both`; una regla dirigida a un tipo del servidor ejecutada en el cliente no tiene ningún objetivo y se filtra por `environment` |

La misma regla también puede escribirse como anotación en el método de reemplazo; la correspondencia es:

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

| Campo del manifiesto | Forma de anotación |
| --- | --- |
| `target` | el primer parámetro del constructor, escrito como `typeof(...)` |
| `method` | el segundo parámetro del constructor, preferiblemente `nameof(...)` |
| `type` | parámetro con nombre `HookType`, por defecto `CallSite` |
| `patchMode` | parámetro con nombre `PatchMode`, por defecto `ILRewrite` |
| `ordinal` | parámetro con nombre `Ordinal` |
| `label` | parámetro con nombre `Label` |
| `environment` | parámetro con nombre `Environment`, por defecto `both` |
| `replaceType` | no se escribe; se toma automáticamente de la clase que anota |
| `replaceMethod` | no se escribe; se toma automáticamente del método que anota |

Las anotaciones provienen de `InjectAttribute` en `NetCraft.ModApi.Extension`, así que los mods que usan anotaciones deben referenciarlo. Cuando existen tanto las anotaciones como el manifiesto, se fusionan, y si el mismo punto de inyección se declara en ambos **gana la anotación**.

Las reglas de mixin usan un conjunto diferente de campos: `target` (nombre completo del tipo objetivo), `source` (nombre completo del tipo fuente, que debe estar en el propio ensamblado del mod), `interfaces` (opcional, un array de nombres completos de interfaces); para la semántica ver [2.8](#28-mixins-añadir-miembros-a-un-tipo-objetivo). `[Mixin(typeof(target))]` en la clase fuente equivale a la entrada del manifiesto.

### 4.4 Qué se puede y qué no se puede enganchar

Se puede enganchar: los ensamblados del kernel bajo `kernel/`, y otros mods bajo `mods/`.

No se puede enganchar:

- la biblioteca principal `NetCraft.dll`
- el cargador `NetCraft.ModLoader.dll`
- los ensamblados de entrada (`NetCraft.Server.Exe.dll`, etc.)

El `target` de cada hook debe poder encontrarse en el kernel o en algún ensamblado de mod, de lo contrario el ensamblado informa "the injection target is not in any known assembly". Ten en cuenta que **un espacio de nombres no implica un ensamblado** — `NetCraft.Game.Server.DedicatedServer` vive en realidad en `NetCraft.Server.dll`; el cargador lo busca mediante un índice construido a partir de las tablas de metadatos, así que basta con escribir el nombre completo.

La inyección de mods sobre mods sigue el mismo sistema: el mod objetivo se reescribe **en el momento en que él mismo se carga**, independientemente del orden en `mods/`. Las reglas cíclicas (A inyecta B y B inyecta A) informan "cyclic loading" en tiempo de carga; esas reglas no tienen solución a nivel de reescritura de IL, así que basta con eliminar una de ellas.

Fíjate en que el mod inyectado **no puede ser tu propio mod** — la autoinyección también cuenta como ciclo.

### 4.5 Despliegue

Copia el dll compilado en el `mods/` del directorio de salida (nivel superior, sin recursión en subdirectorios) y reinicia el proceso.

**Este paso ya está automatizado por la compilación**: `DeployModToHosts` en `NetCraft.ModApi.csproj` copia el dll al directorio `mods/` del directorio de salida de cada proyecto anfitrión tras compilar. La lista de anfitriones es la propiedad `ModHostProjects`; al crear tu propio proyecto anfitrión, basta con añadir su nombre. Si ninguna regla surte efecto y no hay ningún error, comprueba primero si el dll en el `mods/` del proyecto anfitrión está obsoleto.

El registro de arranque imprime:

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

Orden de resolución de problemas:

- Ninguna regla surte efecto: comprueba primero si el dll en `mods/` está obsoleto (lo más común).
- Si no ves `Rewrote and preloaded assembly`, el ensamblado objetivo ya estaba cargado antes del arranque, así que la regla se escribió demasiado tarde.
- Si ves `injection targets` pero el objetivo es erróneo, normalmente es un `target` mal escrito; sigue [4.4](#44-qué-se-puede-y-qué-no-se-puede-enganchar) para comprobar a qué ensamblado pertenece el tipo.
- Una regla con `environment` puesto a `client` se omite silenciosamente en el servidor (una falta de coincidencia a nivel de mod significa que no se carga el mod entero); este es el comportamiento esperado, no un fallo.

### 4.6 Dependencias de terceros

Corresponde al Jar-in-Jar de Fabric.

Los proyectos creados a partir de la plantilla **no necesitan preocuparse por esto**: las bibliotecas añadidas mediante `dotnet add package` se incrustan automáticamente en el dll del mod en tiempo de compilación, y cuando el cargador no puede resolver un ensamblado busca entre los recursos `.dll` incrustados del mod.

```
dotnet add package Newtonsoft.Json
```

Eso es todo; `ncmod.json` no necesita cambios.

Un proyecto escrito a mano debe copiar el destino `EmbedDependencies` del csproj de la plantilla, o incrustarlo tú mismo:

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

Las dependencias no se escriben en el manifiesto; la resolución solo mira los recursos `.dll` incrustados del mod, y solo necesita que el nombre del recurso coincida con el nombre del ensamblado.

Tres notas:

- **La coincidencia es por nombre de ensamblado**. El cargador compara el nombre de ensamblado solicitado: tras eliminar `.dll`, el nombre del recurso o bien es igual al nombre del ensamblado o bien termina en `.` + el nombre del ensamblado. Así que tanto `MyLib.dll` como el nombre por defecto `MyProject.deps.MyLib.dll` funcionan.
- **Solo se reconocen los recursos incrustados del propio mod**. Las bibliotecas no incrustadas no pueden resolverse y no se buscarán en otro lugar.
- **El orden de resolución es primero el kernel**. `EmbeddedAssemblyLoader` busca primero entre los ensamblados del kernel y las sub-bibliotecas incrustadas, y solo recurre a los mods si no encuentra nada, así que los mods no deben incrustar ensamblados con el mismo nombre que los del kernel.

El destino de la plantilla usa `WithMetadataValue` para filtrar `.dll` en lugar de escribir `Condition`, porque el motor de plantillas evalúa `Condition` en `.csproj` en tiempo de plantilla, cuando `%(...)` no tiene valor, y se eliminaría toda la línea.

### 4.7 Icono e información de visualización

El nombre, la descripción, los autores, los enlaces y el icono se escriben todos en `ncmod.json`, correspondiendo a `name` / `description` / `authors` / `contact` / `icon` en el `fabric.mod.json` de vanilla.

**El icono es un recurso incrustado**, no un archivo externo, usando la misma convención de nombres de recursos que las dependencias incrustadas:

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

La plantilla ya incluye un `icon.png` con ambos lugares configurados; solo reemplaza esa imagen. Se recomienda un PNG de 64×64 o 128×128.

El orden de búsqueda del icono es: lo que apunta el `icon` del manifiesto → un recurso incrustado llamado `icon.png` → si no existe ninguno, la imagen por defecto de la interfaz (un signo de interrogación gris).

Esta información es visible en la página **MODS** de la interfaz gráfica del servidor. A la izquierda está la lista de mods (icono pequeño + nombre para mostrar + versión + estado); a la derecha están la descripción, los créditos, los enlaces, las dependencias, el tiempo de inicialización y las reglas de inyección del seleccionado. Los nombres de mod en la columna de dependencias son clicables y saltan directamente a esa entrada.

Los mods que fallaron al cargar o se omitieron también están en la lista, marcados en la columna de estado — al diagnosticar "por qué mi mod no surtió efecto", comprueba primero esta columna.

---

## 5. Limitaciones conocidas

- **Las firmas de sonda solo pueden usar tipos BCL y `object`**, por la razón de 2.2; los parámetros y valores de retorno de tipo valor son la excepción y deben conservar sus tipos reales.
- **`CallSite` es un reemplazo**, y el método de reemplazo debe restaurar la llamada original él mismo, ver 2.3. Los métodos privados no pueden restaurarse y requieren reflexión.
- **Los dll bajo `mods/` los despliega automáticamente `DeployModToHosts`**; al crear tu propio proyecto anfitrión, recuerda añadir su nombre a `ModHostProjects`.
- **No se puede inyectar en los ensamblados de entrada**: si tu objetivo y tu hook caen en un ensamblado de entrada, es ineficaz.
- **No se puede enganchar la biblioteca principal ni el cargador**; esto es una restricción de diseño que impide que los mods cambien el propio proceso de carga.
- **Un `environment` que no coincide significa que no se carga el mod entero**, no "algunas reglas fallan".
- **La inyección en tiempo de ejecución requiere la biblioteca nativa**: las reglas que usan `RuntimeInject` requieren que `lead_hook_native` se adjunte al arrancar el proceso, y el cargador se reinicia a sí mismo para lograrlo; si no se encuentra la biblioteca o falla el reinicio, este lote de reglas se degrada a una advertencia y no se bloquea el arranque. Para los límites de capacidad y el coste ver [2.6](#26-inyección-en-tiempo-de-ejecución-modificar-código-ya-en-ejecución).
- **Las anotaciones tienen menos campos que la API de C#**: `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` no pueden escribirse en `[Inject]` (`PatchMode` y `Ordinal` sí se admiten), ver [2.1](#21-estilos-de-inyección-anotaciones-o-manifiesto-elige-uno).
- **Los mixins se aplican solo a la reescritura en tiempo de carga**, y los miembros de la clase fuente se mueven en lugar de copiarse; los tipos anidados y los métodos genéricos en la clase fuente quedan fuera de cobertura, y los métodos mezclados con una interfaz se marcan como virtuales. Ver [2.8](#28-mixins-añadir-miembros-a-un-tipo-objetivo).
- Al probar y depurar, si se usan tablas de idioma o recursos de modelos, se requiere un directorio `assets` (extraído del jar de vanilla), de lo contrario las funciones relacionadas se degradan a claves de traducción o texturas de marcador de posición.

---

## 6. Comportamientos conocidos

Esta sección es un registro de comportamientos observados, no una especificación.

### 6.1 Dependencia entre mods + inyectar el mod del que se depende

**Escenario**: b depende de a y también inyecta a.

**Conclusión**: funciona y no forma un ciclo.

La cadena tiene tres pasos:

1. La tabla de reglas se construye puramente leyendo metadatos PE, sin cargar ningún ensamblado. La regla "b inyecta a" no requiere que ni a ni b estén presentes.
2. `PreloadReplacers` carga todas las clases de reemplazo (incluida b) antes de `ModManager`, momento en el que a aún no está cargado. `LoadFromStream` lee solo metadatos y no compila con JIT los cuerpos de método, así que la referencia de b a a es perezosa en este momento y la carga no falla.
3. `ModManager` luego ordena topológicamente por `AssemblyRef`, con a antes que b. Al reescribir a, la clase de reemplazo b ya está en `Default`, así que el tipo se obtiene directamente y se añade un `AssemblyRef` que apunta a b a los metadatos de a. **a no necesita saber en absoluto que b existe.**

**Requisito estricto**: al referenciar el mod inyectado en el csproj, debes escribir `Private="false"`. Por defecto copia `a.dll` al directorio de salida, que luego `EmbedDependencies` incrusta en `b.dll` como dependencia incrustada, y en tiempo de ejecución `ModLibs` al resolver `a` toma la segunda copia, haciendo que fallen las comprobaciones de identidad de tipos entre ambas.

**Caso de verificación**: dos proyectos de plantilla `NetCraft.Test1` (a) y `NetCraft.Test2` (b); a proporciona `Test1Api.Greet` y lo llama en su propio `ModEntry.Server`, mientras que b reemplaza ese sitio de llamada por `Test2Probe.OnGreet` y también llama a `Greet` una vez en el `ModEntry.Server` de b como control. Registro de ejecución real:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← la inyección surtió efecto
Test2 internal greeting: Test1 original greeting test2    ← grupo de control: la regla solo reescribe el ensamblado objetivo, así que el propio sitio de llamada interno de b queda intacto
```

Las dependencias derivan el orden solo de `AssemblyRef` y no conllevan restricción de versión. Para restringir versiones, decláralas en el `depends` del manifiesto, ver [4.2](#42-campos-de-ncmodjson).

### 6.2 Inyección por anotación

**Escenario**: `[Inject(typeof(X), nameof(X.M))]` en el método de reemplazo, sin regla en `ncmod.json`.

**Conclusión**: funciona y no necesita declaración en el manifiesto. Las anotaciones y el manifiesto comparten una única fuente, se fusionan al ensamblar, y la anotación gana para el mismo punto de inyección.

**Por qué no se necesita declaración**: las anotaciones son solo la tabla `CustomAttribute` en los metadatos, y el cargador ya lee los mismos metadatos al escanear mods (manifiesto, recursos incrustados y AssemblyRef — tres elementos), así que leer una tabla más no introduce ninguna nueva restricción de carga ni de tiempo. La única precondición es que el mod referencie `NetCraft.ModApi` (el host de las anotaciones).

**La premisa es la lectura estática**: leer anotaciones debe pasar por `MetadataReader` y **no debe usar `Assembly.Load` + `GetCustomAttributes`** — esto último arrastra el ensamblado del mod solo para leer reglas, y la ventana de reescritura desaparece al instante.

**`typeof` no constituye una referencia de tipo**: lo que `typeof(X)` compila en el parámetro es el nombre serializado del tipo (`full name, assembly, Version=…`), que se resuelve como una simple cadena y no requiere que `X` esté presente. Así que escribir `typeof(injected-mod)` en una anotación **no** viola la restricción de 2.2 de que "una clase de reemplazo no debe referenciar los tipos del mod inyectado" — que un nombre aparezca en metadatos y que resuelva un tipo en tiempo de ejecución son dos cosas diferentes.

**Caso de verificación**: `NetCraft.Test2` contra dos métodos de `NetCraft.Test1`; `Greet` va por la anotación y `Farewell` por el manifiesto. Ambas reglas se instalan y ambos sitios de llamada se reemplazan:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← anotación
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← manifiesto
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← grupo de control: la regla solo reescribe el ensamblado objetivo
Test2 internal farewell: Test1 original farewell test2     ← grupo de control
```

El mismo caso también verificó de paso que el parámetro con nombre de la anotación (`Environment = "server"`) también se resuelve.

### 6.3 El coste de rendimiento de adjuntar el profiler

**Escenario**: el mismo programa de cómputo puro (cien millones de iteraciones de módulo y acumulación), ejecutado una vez sin el profiler y otra con él adjunto (las tres variables de entorno `CORECLR_ENABLE_PROFILING`). Cinco ejecuciones cada una.

**Conclusión**: ninguna diferencia en estado estable; el coste está enteramente en el arranque.

| | Tiempo total del proceso (5 ejecuciones, ms) | Tiempo de cómputo dentro del programa |
| --- | --- | --- |
| sin | 306 / 260 / 278 / 253 / 300 | 225 ms |
| con | 406 / 370 / 340 / 325 / 354 | 193 ms |

El tiempo total es unos 80–110 ms más largo. La fuente es `COR_PRF_DISABLE_ALL_NGEN_IMAGES` — habilitar ReJIT requiere deshabilitar al mismo tiempo las imágenes ReadyToRun, así que el código del framework solo puede pasar por el JIT; el bucle cronometrado dentro del programa es idéntico una vez compilado con JIT, sin mostrar diferencia (la ejecución con el profiler fue en realidad algo más rápida, lo cual es ruido).

**Un desperdicio también corregido de paso**: la implementación inicial se suscribía a `COR_PRF_MONITOR_JIT_COMPILATION`, un callback nativo que se disparaba tras compilar cada método, que nunca usábamos. Tras eliminarlo, la máscara de eventos cambió de `0x80040024` a `0x80040004`; la tabla de arriba son los datos tras la eliminación.

**Caso de verificación**: `__hookverify/BenchProbe`.

### 6.4 Dos mods inyectando el mismo objetivo

**Escenario**: dos mods declaran cada uno una regla que alcanza el mismo método objetivo (forma `MethodBody`, métodos de reemplazo diferentes).

**Conclusión**: sin error, sin colapso; **gana la regla ensamblada primero y la segunda falla silenciosamente**.

Ambas reglas entran en la tabla de reglas — no se hace ninguna deduplicación entre mods. Al reescribir, se toma la **primera** entrada de la lista de la misma clave para `OriginalType::OriginalMethod`; el alcance del host se comporta igual, con `InType`/`InMethod` siendo "gana la primera coincidencia". Cuál es la primera depende del orden de ensamblado, y el orden de ensamblado proviene del orden de enumeración del directorio de mods; **no hay campo de prioridad y no puede controlarse declarando dependencias** (las dependencias afectan solo al orden de `Init()`, no al ensamblado de las reglas de inyección).

**La inyección en tiempo de ejecución también es por orden de llegada**: una solicitud de registro posterior se envía normalmente, pero `GetReJITParameters` reclama por "módulo + método" y siempre coincide con la primera solicitud, así que el registro posterior no aterriza. Medido tras una segunda inyección, el comportamiento del método objetivo se queda en el primer resultado.

**Una regla mala no afecta a las demás**: problemas como que el tipo objetivo no esté en ningún ensamblado conocido o que una forma de inyección esté mal escrita se registran en `ModHooks.Errors` en tiempo de ensamblado y esa entrada se omite, mientras que las reglas de los otros mods se ensamblan como de costumbre.

**Los conflictos se registran**: cuando el ensamblado detecta el mismo punto de inyección declarado por varios mods, el ensamblado después se escribe en `ModHooks.Warnings` y se emite en el registro de arranque como `Mod injection conflict ...`, nombrando qué dos mods colisionaron y cuál no surtirá efecto. Es una advertencia, no un error, no afecta a la carga, y la regla en sí permanece en la tabla (solo inalcanzable).

**La inyección en tiempo de ejecución también pasa por esta comprobación**: la clave de detección de conflictos incluye `patchMode`, así que que un mod escriba `ILRewrite` y otro escriba `RuntimeInject` no cuenta como colisión (dos rutas independientes, cada una hace lo suyo); solo se juzgan conflictivos y se advierten dos del mismo modo. Su aterrizaje real también es por orden de llegada — `GetReJITParameters` reclama por "módulo + método", coincidiendo con la primera solicitud, así que las posteriores se envían pero no aterrizan.

**El único punto de colapso duro**: cuando dos mods parchean ambos el mismo método en modo `RuntimePatch`, el segundo choca con la comprobación de registro duplicado de `RuntimeHookEngine` y lanza `InvalidOperationException`, y esta ruta no se captura, así que el arranque falla directamente. `RuntimePatch` está en vías de desaparición (ver [2.6](#26-inyección-en-tiempo-de-ejecución-modificar-código-ya-en-ejecución)); no lo uses en reglas nuevas.

**Caso de verificación**: el módulo `modinjection` de `NetCraft.Test`, entradas `same anchor first mod wins quietly` y `one bad rule does not sink the rest`.

### 6.5 Aún no verificado

- **La inyección en tiempo de ejecución funcionando en un servidor real**: el modo `RuntimeInject` se ha verificado de extremo a extremo en `__hookverify/RuntimeProbe` (tras el registro el comportamiento del método objetivo se intercambia por el del reemplazo), y el lado del ensamblado del mod también tiene cobertura de pruebas para el enrutamiento y la degradación; pero ningún paso de compilación coloca actualmente `lead_hook_native` en el directorio de ejecución de NC, así que ejecutar esta cadena en un servidor real requiere primero poner la biblioteca en la raíz del programa (o apuntarla con `NC_PROFILER_PATH`). Este paso no está hecho.
- **Una clase de reemplazo que referencia los tipos del mod inyectado**: por razonamiento, en el momento de reescribir a resolvería a, mientras que a queda atascado justo antes de completar la carga (aún no está en `Default`, y ni `ModLibs` ni el callback de resolución del kernel reconocen los ensamblados de mod), así que se espera que `PrepareMod` lance y registre en `result.Errors`. Aún no ejecutado realmente. Ten en cuenta que 6.2 solo prueba que **`typeof` en una anotación** no es una referencia; **el tipo que aparece en una firma de método** es otra cosa.

---

## Apéndice: resumen de HookType

| Forma | Efecto | Requisito sobre la firma del método de reemplazo |
| --- | --- | --- |
| `CallSite` | reemplaza los sitios de llamada al método objetivo por tu método | el número de parámetros coincide con el método llamado (llamada de instancia +1) |
| `MethodBody` | reemplaza todo el cuerpo del método objetivo | coincide con el método reemplazado |
| `NewObj` | reemplaza `new X(...)` | el número de parámetros coincide con el constructor |
| `FieldRead` | instrumenta las lecturas de campo | según el tipo de lectura |
| `FieldWrite` | instrumenta las escrituras de campo | según el tipo de escritura |
| `TypeCheck` | instrumenta `isinst` / `castclass` | según el tipo comprobado |
| `Box` | instrumenta el boxing/unboxing | según el tipo del elemento |
| `FunctionPointer` | instrumenta las cargas de punteros a función | según el tipo de delegado |
| `LocalRead` | instrumenta las lecturas de variables locales | cero parámetros, devuelve el valor de la variable |
| `LocalWrite` | instrumenta las escrituras de variables locales | un parámetro, recibe el valor escrito |
| `Constant` | instrumenta las cargas de constantes | cero parámetros, devuelve el valor de la constante |
| `Probe` | conserva el cuerpo del método original, instrumentando la entrada y cada salida; con `LabelArgumentIndex`, un argumento puede plegarse en la etiqueta | `Begin()` devuelve long, `End(string, long)` |
| `Mark` | informa una vez solo en la entrada del método, sin cronometraje | `void method(string label)` |

`Probe` y `Mark` pasan solo el texto de la etiqueta (`Probe` también puede incluir el `ToString()` de un argumento); no pueden obtener referencias de objeto. Para obtener los argumentos reales, usa `CallSite`.

`InType`/`InMethod`, `Placement` y `Ordinal` (ver [2.4](#24-reducir-a-un-solo-sitio-alcance-del-host-y-colocación)) solo tienen sentido para las formas a nivel de instrucción: las diez entradas de la tabla de arriba distintas de `MethodBody`, `Probe` y `Mark` pueden elegir reemplazar o insertar-antes/después, y pueden usar `Ordinal` para elegir una sola ocurrencia; `MethodBody` siempre reemplaza todo, y `Probe`/`Mark` ignoran estos parámetros.

Para las tres clases `LocalRead` / `LocalWrite` / `Constant`, el método anfitrión se escribe en `target` en lugar de una entidad referenciada, y se requiere además `localIndex` o `constantValue`; ver [2.5](#25-anclas-dentro-del-cuerpo-del-método-variables-locales-y-constantes).
