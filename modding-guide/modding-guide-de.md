# NetCraft Modding-Leitfaden


Details zu einzelnen APIs stehen nicht hier; siehe [modapi-de.md](modapi-de.md).

---

## 1. Zuerst die Laufzeitstruktur

### 1.1 Drei Prozess-Einstiegspunkte

NC hat drei Einstiegspunkte, und die Mod-Injektionspipeline ist für alle drei gleich:

| Einstieg | Zweck |
| --- | --- |
| `NetCraft.Loader` | eine exe für beide Seiten: `--server` startet den Server; `--client` oder kein Modus-Flag startet den Client |
| `NetCraft.Server.Exe` | eigenständige Server-Executable |
| `NetCraft.Client.Exe` | eigenständige Client-Executable |

`Main` selbst ist eine dünne Hülle, die nur Callbacks registriert und die Arbeit an die nächste Methode übergibt. Nimm `NetCraft.Server.Exe`:

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // den Kernel-Assembly-Auflösungs-Callback registrieren
    BootMods(args);                        // den Mod-Bootstrap ausführen
    return Launch(args);                   // erst jetzt in die Geschäftsimplementierung eintreten
}
```

Diese Trennung ist keine Stilfrage. Wenn der JIT eine Methode kompiliert, löst er **alle Typen** auf, die im Methodenkörper vorkommen, und dies geschieht, bevor die Methode ausgeführt wird. Wenn `Main` direkt `ServerMain.Run(args)` aufriefe, würde `NetCraft.Server.dll` in dem Moment heraufgezogen, in dem `Main` JIT-kompiliert wird, bevor der Mod-Bootstrap gelaufen ist, und das Umschreibungsfenster wäre weg. Daher müssen sowohl `BootMods` als auch `Launch` mit `MethodImplOptions.NoInlining` markiert werden — ohne die Markierung inlined der JIT sie zurück in `Main`, was die Aufteilung zunichtemacht.

`NetCraft.Loader` hat dieselbe Struktur, außer dass seine Modus-Erkennung und sein Mod-Bootstrap beide in `Launch` liegen und `Main` nur die beiden Schritte `Initialize` und `Launch` behält.

### 1.2 Kernel-Assemblies im Unterverzeichnis kernel/

Das Ausgabeverzeichnis nach einem Build sieht so aus:

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- Hauptbibliothek, bettet alle tieferliegenden Unterbibliotheken ein
NetCraft.ModLoader.dll    <- der Loader selbst
NetCraft.Server.Exe.dll   <- Einstiegs-Assembly
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... übrige Kernel-Assemblies
mods/
  your-mod.dll
```

Warum sie nach `kernel/` verschieben, statt sie im Stammverzeichnis zu belassen?

Der .NET-Host behandelt in `deps.json` registrierte Assemblies als TPA (Trusted Platform Assemblies). Für Assemblies in der TPA löst die Laufzeit **anhand des Pfads** auf — Bytes, die an `AssemblyLoadContext.LoadFromStream` übergeben werden, werden einfach ignoriert. Das heißt, selbst wenn wir im Voraus umgeschriebene Bytes einspeisen, liest die Laufzeit trotzdem die nicht umgeschriebene Kopie von der Platte. Erst wenn die Kernel-Assemblies aus `deps.json` entfernt und die Dateien weggeschoben werden, ruft die Laufzeit bei Auflösungsfehler wieder in `AssemblyLoadContext.Resolving` zurück und gibt uns die Chance, die umgeschriebenen Bytes zu übergeben.

Die drei Arten, die im Stamm belassen werden, können nicht verschoben werden: die Hauptbibliothek (sie ist der einbettende Host und muss zuerst starten), der Loader selbst (der Bootstrap-Code liegt darin) und die Einstiegs-Assembly (der Apphost startet von ihr).

**Kosten**: In die Einstiegs-Assembly selbst kann nicht injiziert werden. Wenn dein Hook-Ziel zufällig in der Assembly `NetCraft.Server.Exe.dll` liegt, ist es wirkungslos. Kernel-Geschäftscode liegt vollständig unter `kernel/`, daher ist dies normalerweise kein Problem.

### 1.3 Mod-Ladereihenfolge

```
EmbeddedAssemblyLoader.Initialize()
  └─ den Resolving-Callback installieren
BootMods → ModBootstrap.Run(aktuelle Seite)
  ├─ mods/*.dll statisch scannen (MetadataReader liest eingebettete ncmod.json, keine Assembly wird geladen)
  ├─ Mods herausfiltern, deren environment nicht zur aktuellen Seite passt
  ├─ Injektionsregeln zusammenstellen und den Rewriter an die Hauptbibliothek übergeben
  ├─ Ziel-Assemblies vorladen: Bytes lesen → durch den Rewriter laufen lassen → LoadFromStream
  └─ die Init() jedes Mod-Einstiegs aufrufen
Launch → ServerMain/ClientMain.Run(args)
  └─ Kernel-Geschäft beginnt zu laufen; die Probes sind bereits darin
```

Beachte die Reihenfolge: **Deklarationen werden gescannt, umgeschriebene Bytes werden geladen, und Einstiegscode läuft zuletzt**. Wenn `Init()` eines Mods ausgeführt wird, sind die Kernel-Assemblies bereits ersetzt.

---

## 2. Zentrale Unterschiede zu Fabric

| Dimension | Fabric | NetCraft |
| --- | --- | --- |
| Sprache / Laufzeit | Java / JVM | C# / .NET 10 (CoreCLR) |
| Mod-Träger | jar, das `fabric.mod.json` enthält | dll, die `ncmod.json` einbettet |
| Deklarationslesen | eine Datei im jar lesen | `MetadataReader` liest eingebettete Ressourcen statisch, ohne Assemblies zu laden |
| Code-Injektion | Mixin (Annotationen im Quellcode; Mitglieder werden beim Laden der Klasse in die Zielklasse eingemischt) | `Lead.Hook` (Regeln in einem Manifest oder in Annotationen deklariert; Bytes werden während der Assembly-Auflösung an Ort und Stelle umgeschrieben) |
| Injektionsgranularität | jede Zeile in einem Methodenkörper, einschließlich lokaler Variablen und Zwischenausdruckswerten | dreizehn Formen (Aufrufstelle, Feld lesen/schreiben, Konstruktor, Typüberprüfung, Boxing, lokale Variable, Konstante, Ersetzung des gesamten Methodenkörpers, Probes usw.), mit Einfügen davor oder danach |
| Lademodell | Fabric Loader + Knot-Klassenlader | einzelne Standard-ALC + `AssemblyLoadContext.Resolving` |
| Umfang der offiziellen API | Fabric API hat sehr viele Module | NetCraft-ModApi hat derzeit nur Event- und Befehls-Erweiterungspunkte |

### 2.1 Injektionsstile: Annotationen oder Manifest, eines wählen

Fabrics Mixin wird **im Quellcode** annotiert:

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NC unterstützt beide Stile, aber ihre Voraussetzungen unterscheiden sich: Die Annotationsform setzt `InjectAttribute` in `NetCraft.ModApi.Extension` voraus, sodass ein Mod, der es nicht referenziert, keine Annotationen verwenden kann; die Manifestform ist reine Daten, die in `ncmod.json` geschrieben werden, und benötigt für Injektionsregeln keine Referenz.

**Annotation**, platziert auf deiner eigenen Ersatzmethode:

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**Manifest**, geschrieben in `hooks` in `ncmod.json`:

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

Während der Assemblierung verschmelzen die beiden Routen zu einer Regeltabelle, und **wenn derselbe Injektionspunkt in beiden geschrieben ist, gewinnt die Annotation**. Nach dem Zusammenführen sind sie nicht mehr unterscheidbar; der Unterschied liegt in den Voraussetzungen und der Ergonomie:

| | Annotation | Manifest |
| --- | --- | --- |
| Wo sie geschrieben wird | auf der Ersatzmethode | in `hooks` in `ncmod.json` |
| Voraussetzung | muss `NetCraft.ModApi` referenzieren | keine, reine Daten |
| Typnamen | `typeof` / `nameof`, vom Compiler geprüft | handgeschriebene Zeichenketten; Tippfehler werden erst während der Assemblierung gefunden |
| Was sie tragen kann | nur Injektionsregeln | Mod-Identität (id, entry, environment, Anzeigeinformationen) und Injektionsregeln |

Also muss `ncmod.json` geschrieben werden, ob du Annotationen verwendest oder nicht; sie ist die einzige Quelle der Mod-Identität. Annotationen machen das Schreiben von Regeln nur weniger fehleranfällig. Das Manifest hat kein Abhängigkeitsfeld; Abhängigkeitsbeziehungen werden aus Assembly-Referenzen abgeleitet (siehe 6.1) und müssen nicht deklariert werden.

Umgekehrt **kann ein Mod, der `NetCraft.ModApi` nicht referenziert, nur das Manifest verwenden** — dies betrifft mehr als Injektionsregeln: Events wie `ServerEvents` und die `Nc*`-Fassaden sind ebenfalls in ModApi (unter `NetCraft.ModApi.Wrapper`, siehe [3.3](#33-ein-gegenüberstellungsbeispiel)), sodass ein Mod, der keine Annotationen verwenden kann, sie ebenfalls nicht verwenden kann.

Die Unterschiede zu Fabric bleiben bestehen:

- **Wie und wann Änderungen geschehen**: Mixin hat einen Transformer, der **Mitglieder der Mixin-Klasse in** die Zielklasse **beim Laden der Klasse einmischt**, sodass das Geladene eine synthetisierte neue Klasse ist und das Original nicht mehr existiert; NC schreibt die Instruktionen der Zielmethode an Ort und Stelle **um, bevor die Assembly in den Speicher gelangt**, sodass die Klasse dieselbe bleibt und nur ihr Methodenkörper sich ändert. Beide schreiben zur Ladezeit um, keines ändert Bytecode zur Kompilierzeit — der Annotation Processor von Mixin erzeugt nur eine Refmap (Obfuskationszuordnung) und führt zur Buildzeit eine Validierung durch, während NC nicht obfuskiert ist und überhaupt keine solche Schicht hat.
- Mixin kann **an jeder Position in der Mitte eines Methodenkörpers** injizieren; NC kann eine bestimmte Aufrufstelle, einen Feldzugriff, eine Konstruktion, ein Lesen/Schreiben einer lokalen Variablen oder eine Konstante innerhalb einer festgelegten Host-Methode anvisieren und davor oder danach einfügen (`InType`/`InMethod` grenzen den Geltungsbereich ein, `Placement` entscheidet Einfügen oder Ersetzen), aber es **kann keine beliebige Zeilennummer erreichen** und kann kein Sprungziel oder einen Zwischenausdruckswert auf dem Stack ändern.
- Mixin-Ziele verwenden einen String-Methodennamen plus Deskriptor; NC verwendet „vollständiger Typname + Methodenname", sodass gleichnamige Overloads alle übereinstimmen und für die Genauigkeit auf einen einzelnen `InType`/`InMethod` erforderlich ist.

**Welche Schicht die Annotationen verarbeitet**: Der Annotationstyp (`InjectAttribute`) wird von `NetCraft.ModApi.Extension` bereitgestellt und von `NetCraft.ModLoader` aufgelöst — beim Scannen von Mods liest er die `CustomAttribute`-Tabelle statisch mit `MetadataReader`, ohne Assemblies zu laden. **`Lead.Hook` erkennt Annotationen nicht**; er sieht nur die zusammengeführte Regeltabelle, und die native Injektionsschicht erkennt nur die auf der Managed-Seite kompilierten Beschreibungsbytes und liest nicht einmal `ncmod.json`.

Dies bestimmt, was Annotationen ausdrücken können: Was du schreiben kannst, hängt vollständig davon ab, welche Felder `InjectAttribute` hat. Derzeit sind es sieben — Zieltyp, Methodenname, `HookType`, `Label`, `Environment`, `PatchMode`, `Ordinal` — und `InType`/`InMethod`/`Placement` aus [2.4](#24-auf-eine-stelle-eingrenzen-host-scoping-und-platzierung) sowie `LocalIndex`/`ConstantValue` aus [2.5](#25-anker-im-methodenkörper-lokale-variablen-und-konstanten) **können nicht in Annotationen geschrieben werden**; verwende die C#-API oder warte, bis das Manifest nachzieht. Auch auf der Manifest-Seite fehlen diese — das Einzige, was sie über Annotationen hinaus akzeptiert, ist `ordinal`.

Für die dreizehn Injektionsformen siehe den [Anhang des Modding-Leitfadens](#anhang-hooktype-übersicht) und [modapi-de.md](modapi-de.md).

### 2.2 Eine wichtige Einschränkung: Probe-Klassen dürfen keine Kernel-Typen in Signaturen tragen

Das Umschreiben von NC geschieht, **bevor** die Kernel-Assemblies geladen werden. Beim Zusammenstellen von Regeln verwendet `Lead.Hook` Reflexion, um deine Ersatzmethode zu finden und eine Methodenreferenz aufzubauen, und dieser Vorgang löst jeden Parametertyp und Rückgabetyp in der Signatur auf.

Daher: **Die Signatur einer Ersatzmethode darf nur BCL-Typen und `object` verwenden**. Sobald ein `NetCraft.*`-Typ in der Signatur auftaucht, zieht dessen Auflösung die Kernel-Assemblies frühzeitig herauf, und die Injektion schlägt schlicht fehl.

Wenn du ein Kernel-Objekt benötigst, deklariere den Parameter als `object` und caste innerhalb des Methodenkörpers:

```csharp
//die Assembly sieht nur object; der Methodenkörper wird JIT-kompiliert, nachdem der Kernel startet
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite ist Ersetzung, nicht Einfügung

Die Ersatzmethode einer `CallSite`-Regel **ersetzt** den Originalaufruf, sodass die Originalmethode nicht ausgeführt wird. Um das ursprüngliche Verhalten zu bewahren, musst du es in der Ersatzmethode selbst wiederherstellen:

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              // den ersetzten Aufruf wiederherstellen
    ServerEvents.CommandRegister.Publish(...);  // dann die eigene Logik des Mods hinzufügen
}
```

Verpasst du diesen Schritt, verschwindet die ursprüngliche Funktionalität vollständig.

Ein paar Details zur Umsetzung:

- **`this` bei einer Instanzmethode zählt ebenfalls als Parameter**. Ob die Zielmethode eine Instanzmethode ist, bestimmt, ob die Ersatzmethode einen zusätzlichen führenden Parameter benötigt. In IL zählen sowohl `call` als auch `callvirt` als Instanzaufrufe — für eine nicht-virtuelle Methode auf einem `sealed`-Typ gibt der Compiler `call` aus.
- **Ein Regelschlüssel deckt alle Overloads unter diesem Klassennamen ab**; Overloads mit derselben Parameteranzahl teilen sich eine Ersatzmethode. `Disconnect(string)` und `Disconnect(Component)` teilen sich auf diese Weise, und die Ersatzmethode verteilt nach dem echten Argumenttyp.
- **Eine private Methode kann nicht von außen aufgerufen werden**, daher kann die Ersatzmethode den Originalaufruf nicht wiederherstellen. Entweder gibst du diesen Hook-Punkt auf oder rufst ihn einmal per Reflexion auf (bei geringer Frequenz akzeptabel).
- **Werttyp-Parameter und Rückgabewerte können nicht als `object` deklariert werden**, `object` ist eine Referenz auf dem Stack, während `float`/`bool` Werte sind, und eine Nichtübereinstimmung ist ungültige IL. Behalte diese beiden Positionen als ihre echten Typen.
- **Unicast-Callbacks können nicht direkt zugewiesen werden**. Einige Kernel-Callback-Eigenschaften (z. B. die drei Chunk-Callbacks auf `ServerChunkCache`) sind `Action<T>` statt `event`, und der Kernel belegt sie bereits. Ein Mod, der direkt zuweist, überschreibt die Kopie des Kernels ganz ohne Fehler. Der richtige Ansatz ist, den Setter der Eigenschaft zu hooken und im Moment der Zuweisung deine Logik und den Kernel-Callback in einen Wrapper-Delegate zu verketten.

### 2.4 Auf eine Stelle eingrenzen: Host-Scoping und Platzierung

Der Standard-Geltungsbereich einer Instruktionsregel ist **die gesamte Assembly** — jede Stelle, die die Zielmethode aufruft oder das Zielfeld liest/schreibt, passt. Um auf eine Stelle einzugrenzen, verwende zwei optionale Parameter:

| Parameter | Wirkung |
| --- | --- |
| `InType` / `InMethod` | passen Anker nur innerhalb des angegebenen Host-Methodenkörpers an; beide leer bedeutet uneingeschränkt |
| `Placement` | `Replace` ersetzt den Anker (Standard); `Before` / `After` behalten den Anker und fügen davor oder danach einen Aufruf ein |
| `Ordinal` | wenn derselbe Anker an mehreren Stellen in der Host-Methode passt, wähle welche, 0-basiert. Weggelassen bedeutet, jede Stelle wird modifiziert |

```csharp
//Beispiel: nur instrumentieren, wenn LevelChunk den Blockzustand liest; PalettedContainer::Get anderswo bleibt unberührt
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

Die beiden Modi stellen unterschiedliche Anforderungen an die Callback-Signatur:

- **Replace-Modus** richtet sich nach den Argumenten des ersetzten Aufrufs (einschließlich `this` bei Instanzaufrufen); ob der Callback den Originalaufruf wiederherstellt, liegt bei dir.
- **Insert-Modus** übergibt die **Parameter der Host-Methode** (einschließlich `this`), konsistent mit der `MethodBody`-Konvention. Das Einfügen stört den Stack nicht, den der Anker bereits aufgebaut hat; der Originalaufruf läuft wie gewohnt, nur mit einem zusätzlichen Callback davor oder danach.

Ein paar Grenzen:

- `InType` und `InMethod` sind unabhängig; du kannst nur eines angeben. Beide leer entspricht keiner Eingrenzung.
- Mehrere Regeln können denselben Anker hooken, jede auf einen anderen Host beschränkt; **der erste Host-Treffer gewinnt**.
- `Placement` gilt nur für Instruktionsformen (`CallSite`, `NewObj`, Feld lesen/schreiben, `TypeCheck`, `Box`, `FunctionPointer` und die drei Arten in [2.5](#25-anker-im-methodenkörper-lokale-variablen-und-konstanten)); `MethodBody` ersetzt immer das Ganze.
- `Ordinal` zählt die **Reihenfolge der Treffer**, unabhängig davon, ob diese Stelle letztlich modifiziert wird; wenn die Regel in der Host-Methode nicht oft genug vorkommt, greift die Regel nicht. Dieselbe Idee wie Mixins `@At(ordinal)`.
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` sind derzeit nur in der C#-API verfügbar; weder `ncmod.json` noch `[Inject]` unterstützen sie (das Manifest akzeptiert `ordinal`), sodass manifestbasierte Mods die ersten nicht verwenden können.

### 2.5 Anker im Methodenkörper: lokale Variablen und Konstanten

Die bisherigen Arten verankern sich an einer **referenzierten Entität** (einer Methode, einem Feld oder einem Konstruktor), während `LocalRead` / `LocalWrite` / `Constant` sich an **einer Position innerhalb des Host-Methodenkörpers** verankern, entsprechend Mixins `@ModifyVariable` und `@ModifyConstant`. Bei diesen dreien benennen `OriginalType` / `OriginalMethod` die **Host-Methode**, nicht eine referenzierte Entität.

| Form | Zusätzlicher Parameter | Ausgewählte Positionen |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` | jedes Lesen dieses Slots (0-basiert) |
| `LocalWrite` | `LocalIndex` | jedes Schreiben in diesen Slot |
| `Constant` | `ConstantValue` | jedes Laden dieser Konstante, verglichen nach Boxed-Typ |

```csharp
//Beispiel: einen Callback vor das Schreiben in Slot 0 in G einfügen
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//Beispiel: die Konstante 5 in G durch den Rückgabewert von OnConst() ersetzen
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

Im Replace-Modus richtet sich die Callback-Signatur nach der **Stack-Wirkung der Instruktion**, nicht nach den Host-Parametern:

| Anker | Stack-Wirkung | Signatur der Ersatzmethode |
| --- | --- | --- |
| `LocalRead` | schiebt einen Wert auf den Stack | null Parameter, gibt diesen Wert zurück |
| `LocalWrite` | nimmt einen Wert vom Stack | ein Parameter |
| `Constant` | schiebt einen Wert auf den Stack | null Parameter, gibt diesen Wert zurück |

`ConstantValue` wird nach Boxed-Typ verglichen, daher sind `5` (int) und `5L` (long) zwei verschiedene Anker; um `ldc.i8` zu treffen, musst du `long` übergeben.

Ein Slot ist der kompilierte Index der lokalen Variablen; derselbe Quellcode kann ihn unter einer anderen Compiler-Version ändern, also behandle ihn beim Portieren über Versionen hinweg nicht als stabilen Bezeichner.

### 2.6 Laufzeit-Injektion: bereits laufenden Code ändern

Die bisher besprochene Injektion geschieht stets **vor dem Laden der Assembly** — Bytes werden zuerst umgeschrieben und dann an die Laufzeit übergeben. Die Voraussetzung ist, dass die Ziel-Assembly noch nicht geladen wurde.

`Lead.Hook` hat eine weitere Route: die Profiler-Schnittstelle der CLR (ReJIT) verwenden, um Code zu ändern, der **bereits geladen ist oder dessen Methoden bereits gelaufen sind**. Beide teilen dieselbe `HookRule`, und die Parameter aus [2.4](#24-auf-eine-stelle-eingrenzen-host-scoping-und-platzierung) und [2.5](#25-anker-im-methodenkörper-lokale-variablen-und-konstanten) bleiben verfügbar:

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//ein Aufruf und die Injektion ist erledigt; kein Neustart und keine Dateiänderungen
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| | Umschreiben zur Ladezeit | Laufzeit-Injektion |
| --- | --- | --- |
| Manifest-`patchMode` | `ILRewrite` (Standard) | `RuntimeInject` |
| Zeitpunkt | bevor die Assembly in den Speicher gelangt | jederzeit nachdem der Prozess gestartet ist |
| Grundlage | Mono.Cecil-Byte-Umschreibung | CLR Profiler ReJIT |
| Voraussetzung | das Ziel wurde noch nicht geladen | das Ziel ist bereits im Prozess |
| Bereits JIT-kompilierten Code ändern | nicht möglich | möglich |

**Warum Regeln geteilt werden**: Cecil erledigt hier weiterhin das Umschreiben, aber das Ergebnis wird nicht auf die Platte geschrieben; stattdessen wird es in eine Beschreibung kompiliert, die an die native Schicht übergeben wird, welche den neuen Methodenkörper zur Laufzeit an die CLR übergibt, wobei die übrige Versionsverwaltung der CLR überlassen bleibt.

**Wie ein Mod es macht**: Füge dem Regeleintrag in `ncmod.json` einen `patchMode`-Eintrag hinzu; die `[Inject]`-Annotation hat einen Parameter desselben Namens.

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**Was die Assemblierung tut**: Diese Regeln laufen nicht über den Ladezeit-Umschreibungspfad (`ModHooks.Rewrite` wendet nur `ILRewrite` an); zur Assemblierungszeit wird eine separate Laufzeit-Zieltabelle aufgezeichnet. Nachdem die Kernel-Assemblies vorgeladen wurden und bevor `Init()` der Mods läuft, nimmt der Loader jede ihrer **bereits geladenen** Instanzen und übergibt die umgeschriebenen Methodenkörper an die CLR. Ist ein Ziel in diesem Moment nicht geladen, wird es mit einer Warnung übersprungen, und es wird nicht seinetwegen frühzeitig geladen — die Einschränkung aus [2.2](#22-eine-wichtige-einschränkung-probe-klassen-dürfen-keine-kernel-typen-in-signaturen-tragen), dass „das frühzeitige Heraufziehen des Kernels das Zeitfenster verpasst", ist hier in der Richtung umgekehrt, mit demselben Ergebnis: Wenn es nicht vorhanden ist, kann es nicht gemacht werden.

**Um die native Injektionsschicht zu hooken**: Der ReJIT-Schalter kann nur über Umgebungsvariablen beim Prozessstart gesetzt werden (`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`); sie nach dem Start zu setzen hat keine Wirkung. Wenn der Loader früh während des Starts `RuntimeInject`-Regeln erkennt und der Prozess noch nicht gehookt ist, **startet er den Prozess mit derselben Kommandozeile** neu und trägt diese drei Variablen mit (`NC_PROFILER_ATTACHED=1` schützt vor wiederholten Neustarts bei „gehookt, aber nicht wirksam"). Die native Bibliothek `lead_hook_native` muss im Programmstamm liegen oder per `NC_PROFILER_PATH` an eine andere Stelle verwiesen werden; wenn keines von beiden vorhanden ist, wird der ganze Regelsatz auf eine einzelne Warnung herabgestuft und der Start nicht blockiert.

**Nicht mit Ladezeit-Umschreibung auf demselben Ziel mischen**: Ein durch Laufzeit-Injektion übergebener Methodenkörper leitet sich aus **Originalbytes** ab und enthält keine Änderungen, die die Ladezeit-Umschreibung an derselben Methode vorgenommen hat — wenn dieselbe Methode von beiden Regeltypen getroffen wird, wird die Ladezeit-Version vollständig überschrieben. Die Assemblierung kann nicht erkennen, ob zwei Regeln dieselbe Methode treffen, also kann sie nur eine grobe Beurteilung nach der Ziel-Assembly vornehmen und eine Warnung aufzeichnen.

**Annotationen nehmen an diesem Pfad nicht direkt teil**: Die native Injektionsschicht erkennt `InjectAttribute` nicht und liest `ncmod.json` nicht — sie erkennt nur Beschreibungsbytes. Annotationen und das Manifest sind beide **Assemblierungszeit**-Dinge (siehe [2.1](#21-injektionsstile-annotationen-oder-manifest-eines-wählen)), werden von `NetCraft.ModLoader` in `HookRule` geparst und dann an `RuntimeInjector` übergeben; die Mod-Seite muss sie nicht unterschiedlich behandeln.

**Einschränkungen** (enger als die Ladezeit-Umschreibung): Methoden mit Ausnahmebehandlungstabellen werden nicht unterstützt, die Tabelle der lokalen Variablen kann nicht geändert werden, generische Typen und Methoden werden nicht unterstützt, und Operanden erkennen nur Methodenreferenzen (Feldreferenzen, String-Konstanten und Typ-Tokens werfen `NotSupportedException`).

**Leistung**: Die Injektion geschieht nur bei der Registrierung; danach ist die Methode gewöhnlicher JIT-Code mit demselben Aufruf-Overhead wie ohne Injektion. Das Anhängen des Profilers hat einmalige Kosten — das Aktivieren von ReJIT erfordert gleichzeitig das Deaktivieren von ReadyToRun-Images, und der gemessene Prozessstart ist etwa 80–110 ms langsamer; die Berechnung im eingeschwungenen Zustand zeigt keinen Unterschied. Für einen Server wie NC, dessen Start bereits in Sekunden gemessen wird, ist dies vernachlässigbar.

### 2.7 Kollidieren zwei Mods, die dieselbe Klasse ändern, wie bei Mixin?

Zuerst: Warum es auf der Mixin-Seite zu Konflikten kommt. Mixin **mischt Mitglieder in die Zielklasse ein** und wendet sie beim Laden der Klasse an: Wenn mehrere Mixins in dieselbe Klasse einmischen, werfen Fälle wie wiederholtes Injizieren an derselben Stelle oder das Hinzufügen gleichnamiger Mitglieder zur selben Klasse `MixinApplyError`, und das standardmäßige Fail-Hard **beendet das Spiel kurzerhand**; außerdem geschieht diese Erkennung im Moment des Klassenladens, wenn das Spiel möglicherweise bereits halb läuft.

NCs Modell ist anders, und die Fläche, auf der Konflikte auftreten können, ist viel kleiner:

| | Mixin | NetCraft |
| --- | --- | --- |
| Umsetzungsart | Mitglieder in die Zielklasse einmischen + Bytecode umschreiben | nur IL-Instruktionen umschreiben; keine Typsynthese, keine hinzugefügten Mitglieder |
| Strukturelle Konflikte (gleichnamige Mitglieder, Vererbungskonflikte) | ja | keine |
| Wann Regeln validiert werden | beim Laden der Klasse | zur Assemblierungszeit, statisches Lesen von Metadaten |
| Zwei Regeln, die dieselbe Stelle treffen | wirft | wer zuerst kommt, mahlt zuerst; Letztere schlägt still fehl |
| Ein Mod schlägt fehl | kann den gesamten Ladevorgang mitreißen | betrifft nur sich selbst |

**Statische Validierung**: Regeln werden nicht durch Laden von Assemblies und Reflektieren über Typen aufgebaut, sondern durch Lesen von PE-Metadatentabellen. Daher werden Probleme wie „der Zieltyp ist in keiner bekannten Assembly" oder „die Injektionsform ist falsch geschrieben" **früh beim Start** aufgezeichnet und übersprungen, ohne darauf zu warten, dass eine Klasse geladen wird, bevor es knallt.

**Fehlerisolierung**: Wenn die Regeln eines Mods nicht geparst werden können, die Ersatzklasse nicht geladen werden kann oder der Einstieg `Init()` wirft, wird nur **dieser eine Mod** als fehlgeschlagen markiert (Status `Error`, auf der MODS-Seite als „Laden fehlgeschlagen" angezeigt), und die anderen Mods laden wie gewohnt. Eine Formulierung muss hier korrigiert werden: NC hat **kein Entladen zur Laufzeit** — Mods werden einmal geladen, und `ModManager` bietet ausdrücklich kein dynamisches Laden/Entladen an. Das sogenannte „automatische Entladen bei Fehler" ist tatsächlich **Isolierung zur Ladezeit**: Ein fehlgeschlagener Mod wird nicht initialisiert, aber er ist auch nicht „entladen".

**Dynamische Injektion**: Auf der ReJIT-Route aus [2.6](#26-laufzeit-injektion-bereits-laufenden-code-ändern) entspricht die Semantik, wenn mehrere Mods um dieselbe Methode konkurrieren, dem statischen Fall — der zuerst Registrierte gewinnt, und spätere Anfragen werden gesendet, können sie aber nicht beanspruchen (`GetReJITParameters` beansprucht nach „Modul + Methode" und nimmt die erste).

Auf dieser Route sind wir einmal in eine Falle getappt, die es wert ist, festgehalten zu werden: In einer frühen Implementierung wurde der **Auflösungsbereich-Parameter von `FindTypeRef` als `mdTokenNil` übergeben**, dessen Semantik „nur TypeRefs ohne Auflösungsbereich abgleichen" ist — unsere Referenzen hängen alle an `AssemblyRef`, sodass die von uns aufgebaute nie gefunden wurde. Es äußerte sich so: Die erste Injektion gelang, aber bei der zweiten Injektion schlug die Referenzauflösung fehl, es konnte kein neuer Methodenkörper aufgebaut werden, die CLR fiel auf die Original-IL zurück, und **die erste Injektion ging damit verloren** (die Zielmethode kehrte zu ihrem uninjizierten Verhalten zurück). Nach der Behebung trat es nicht mehr auf, aber die Einschränkung bleibt: **Die Metadaten-Injektion von Referenzen muss innerhalb des Fensters direkt nach dem Laden des Zielmoduls abgeschlossen sein; je später, desto wahrscheinlicher schlägt sie fehl**.

**Die Kosten müssen klar benannt werden**: NCs nicht abstürzendes Verhalten hat den Preis, dass Konflikte leicht übersehen werden — Mixin unterbricht zumindest das Laden, während NC die letztere still fehlschlagen lässt. Um dem zu begegnen, führt die Assemblierung eine **Konfliktprüfung auf denselben Anker** durch: Wenn derselbe Injektionspunkt von mehreren Mods deklariert wird, wird der später assemblierte in `ModHooks.Warnings` aufgezeichnet und im Startprotokoll als Warnung gemeldet (ohne das Laden zu blockieren):

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

Der Erkennungsschlüssel ist „Zieltyp + Methode + Injektionsform + Patch-Modus". **Host-Scoping wird nicht unterschieden** — weder das Manifest noch Annotationen können `InType`/`InMethod` schreiben, daher sind von Mods stammende Regeln naturgemäß im gesamten Assembly-Geltungsbereich, und derselbe Schlüssel bedeutet eine Kollision. Regeln, die direkt über die C#-API hinzugefügt werden, umgehen diese Prüfung, da in diesem Fall zwei Regeln jeweils einen anderen Host treffen können und von Natur aus nicht kollidieren.

**Auch die Laufzeit-Injektion durchläuft diese Prüfung**: Sie tritt in denselben Assemblierungseinstiegspunkt ein, und der Patch-Modus im Erkennungsschlüssel hält sie getrennt von der Ladezeit-Umschreibung; wenn beide Typen auf derselben Ziel-Assembly landen, gibt es einen separaten Überschreibhinweis (siehe [6.4](#64-zwei-mods-die-dasselbe-ziel-injizieren)).

### 2.8 Mixins: Mitglieder zu einem Zieltyp hinzufügen

Die vorherigen Abschnitte ändern alle Instruktionen in vorhandenem Code und können nichts Neues erzeugen. Um **Felder, Methoden oder Schnittstellen** zu einem Zieltyp hinzuzufügen, verwende ein Mixin.

Die Beziehung zur Mixin-Syntax ist wie folgt:

| Mixin | NC |
| --- | --- |
| `@Mixin(X.class)` auf der Mixin-Klasse | `[Mixin(typeof(X))]` auf der Quellklasse |
| Mitglieder der Mixin-Klasse werden in die Zielklasse eingemischt | Felder und Methoden der Quellklasse werden in den Zieltyp verschoben |
| `@Unique` fügt ein privates Feld hinzu | schreibe ein gewöhnliches Feld in der Quellklasse; es wird genauso herüberverschoben |
| `@Shadow` referenziert ein vorhandenes Mitglied der Zielklasse | nicht nötig; schreibe die Mitglieder von `X` direkt und hooke sie dann |
| `@Implements` / `implements` | `Interfaces` |

Es kann auch im Manifest geschrieben werden:

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
    //nach dem Herüberverschieben ist dies ein Instanzfeld auf dem Zieltyp
    public int MyCounter = 5;

    //eine eingemischte Methode; sie liest/schreibt das mitverschobene Feld
    public int Bump() => MyCounter + 1;

    //die vom Interface geforderte Implementierung; nach dem Herüberverschieben implementiert der Zieltyp ITagged
    public string Describe() => $"tagged:{MyCounter}";
}
```

**Verschieben, nicht kopieren**. Diese Mitglieder werden aus der Quellklasse entfernt, sodass nur eine leere Hülle bleibt — genau wie bei Mixins Mixin-Klasse, **sollte Mod-Code diese Klasse nicht mehr verwenden** (`new SomeEntityMixin()` oder der Aufruf ihrer Methoden findet die Mitglieder nicht).

Ein paar Umsetzungsregeln:

- **Gilt nur für die Ladezeit-Umschreibung**. Der Zieltyp muss in einer Kernel- oder Mod-Assembly sein, die noch nicht in den Speicher gelangt ist. Die Laufzeit-Injektion übergibt nur Methodenkörper und kann das Typ-Layout nicht ändern, daher kann diese Form darauf nicht existieren.
- **Konstruktor-Initialisierer kommen mit**. Werte, die in den Feldinitialisierern der Quellklasse geschrieben werden, werden in jeden Instanzkonstruktor des Zieltyps eingemischt; statische Feldinitialisierer werden in den statischen Konstruktor eingemischt (erstellt, falls das Ziel keinen hat). Der Basisketten-Aufruf innerhalb des Konstruktors der Quellklasse wird entfernt, sodass der Basiskonstruktor nicht zweimal ausgeführt wird.
- **Regeln mit Interfaces markieren die verschobenen öffentlichen Instanzmethoden als virtual**. Der Interface-Dispatch erkennt nur die Vtable, und ohne die Markierung würde die CLR feststellen, dass das Interface nicht implementiert ist, und schlicht das Laden verweigern. Erwarte also nicht, dass diese Methoden nicht-virtuell bleiben, wenn ein Interface eingemischt wird.
- **Gleichnamige Mitglieder werden übersprungen**. Wenn der Zieltyp bereits ein Feld oder eine Methode mit demselben Namen hat, wird dieses eine Element nicht verschoben und der Rest läuft wie gewohnt. Wenn zwei Mods in denselben Zieltyp einmischen, landen beide, und nur der namenskollidierende Teil des Letzteren wird übersprungen — sanfter als das Verhalten „Letzterer schlägt vollständig fehl" bei Hooks in [2.7](#27-kollidieren-zwei-mods-die-dieselbe-klasse-ändern-wie-bei-mixin).
- **Verschachtelte Typen werden nicht verschoben**, und verschachtelte Typen oder generische Methoden in der Quellklasse liegen derzeit ebenfalls außerhalb der Abdeckung dieses Pfads.

Der Quelltyp muss in **der Assembly des Mods selbst** liegen, daher schreiben weder das Manifest noch die Annotation einen Assembly-Namen.

### 2.9 Zwei Routen: Wrapper-Schicht und Erweiterungspunkte

Die öffentliche Schnittstelle von `NetCraft.ModApi` ist in zwei Namespaces aufgeteilt, die zwei Verwendungsweisen entsprechen:

| Namespace | Inhalt | Was du bekommst |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`, `NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup` und Objekthandles wie `NcPlayer` / `NcLevel` | Wrapper-Typen; keine Kernel-Typen in der öffentlichen Schnittstelle |
| `NetCraft.ModApi.Extension` | `[Inject]`- / `[Mixin]`-Annotationen | Regeln binden an Kernel-Klassen- und -Methodennamen |

Die beiden sind **parallele** Routen, nicht eine über die andere geschichtet:

- **Für Stabilität verwende `Wrapper`**. Die Fassaden übernehmen die mühsame Aufrufreihenfolge des Kernels für dich (einen einzelnen Block zu schreiben und dabei das Level, die Spielerliste und die Sync-Kette zu berühren, ist ein Beispiel), und Event-Args sind alle Wrapper-Typen. Die Kosten sind, dass Fähigkeiten, die die Fassaden nicht offenlegen, für dich nicht verfügbar sind.
- **Für Vollständigkeit verwende `Extension`**. Injektionsregeln modifizieren Kernel-Klassen und -Methoden direkt, aber die Zielnamen, die du schreibst, sind die Namen des Kernels, also müssen sich die Regeln mit ihm ändern, wenn sich der Kernel ändert.

Du kannst beide referenzieren. Die `Wrapper`-Linie wird in Richtung „keine Kernel-Typen in der öffentlichen Schnittstelle" konsolidiert; die Spieler- und Level-Teile sind erledigt — `Player` / `Attacker` in Spieler-Events und die In/Out-Parameter von `NcPlayers` sind `NcPlayer`-Handles, `NcWorld.Overworld` / `Nether` / `End` / `Get` und `LevelTickArgs.Level` sind `NcLevel`-Handles, und Blockkoordinaten sind einfache `x y z`-Ganzzahlen; Entities und die übrigen Werttypen (`BlockPos` / `BlockState` / `Vec3`) sind noch nicht verpackt.

Noch eine Grenze ist zu nennen: **Die Wrapper-Schicht schirmt die Injektion nicht ab**. Die `hooks`-Regeln oder `[Inject]`-Annotationen, die du schreibst, binden weiterhin an Kernel-Klassen- und -Methodennamen und brechen genauso, wenn sich der Kernel ändert.

---

## 3. Von Fabric migrieren

### 3.1 Konzeptzuordnung

| Fabric | NetCraft |
| --- | --- |
| `fabric.mod.json` | die in die dll eingebettete `ncmod.json` |
| `ModInitializer.onInitialize()` | das `public Task Init()` der Einstiegsklasse |
| `@Inject` / `@Redirect` | die `[Inject]`-Annotation oder Regeln wie `Mark` / `Probe` / `CallSite` in `hooks` |
| `@ModifyVariable` | `LocalRead` / `LocalWrite`, siehe [2.5](#25-anker-im-methodenkörper-lokale-variablen-und-konstanten); nicht als Annotation schreibbar |
| `@ModifyConstant` | `Constant`, siehe [2.5](#25-anker-im-methodenkörper-lokale-variablen-und-konstanten); nicht als Annotation schreibbar |
| `@Accessor` | noch kein Äquivalent (`private`-Mitglieder müssen nicht erweitert werden; schreibe einfach eine Regel) |
| `Registry.register(...)` | die Kernel-Registry (`BuiltInRegistries`) |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` | noch kein Äquivalent (`ModManager` ist nicht für Mods geöffnet) |
| `@Mixin` / `@Unique` / `@Implements` | die `[Mixin]`-Annotation oder die `mixins` des Manifests, siehe [2.8](#28-mixins-mitglieder-zu-einem-zieltyp-hinzufügen) |

### 3.2 Was nicht übernommen wird

- **Das Annotationssystem von Mixin**: NC hat zwei Annotationen, `[Inject]` und `[Mixin]`, die beide nur **Deklarationsstile** sind, äquivalent zu `hooks` / `mixins` in `ncmod.json` und bei der Assemblierung zusammengeführt (sie werden vom Loader aufgelöst, nicht von `Lead.Hook`, siehe [2.1](#21-injektionsstile-annotationen-oder-manifest-eines-wählen)). Das Umschreiben von Instruktionen landet standardmäßig zur Ladezeit und kann gemäß [2.6](#26-laufzeit-injektion-bereits-laufenden-code-ändern) auf Laufzeit-Übergabe umgestellt werden; das Hinzufügen von Mitgliedern und Interfaces läuft über die Mixins aus [2.8](#28-mixins-mitglieder-zu-einem-zieltyp-hinzufügen). Die Zielauswahl hat keine `@At`-artige String-Syntax, aber Formen wie `CallSite`/`FieldRead`/`LocalWrite`/`Constant` können zusammen mit `Ordinal`, `InType`/`InMethod` und `Placement` die Verwendungen von `HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE` und `shift` abdecken; was fehlt, ist `JUMP`.
- **AccessWidener**: keines. Sichtbarkeit ist in NC kein Hindernis für IL-Umschreibung; `private`-Methoden können auf dieselbe Weise gehookt werden (der Rewriter arbeitet auf Byte-Ebene).
- **Yarn / Mojang-Mappings**: nicht nötig. NC ist direkt aus Vanilla übersetzter C#-Quellcode, wobei Typ- und Methodennamen Vanilla entsprechen, nur der Namensstil folgt C#.
- **Die überwiegende Mehrheit der Fabric-API-Module**: Nur Fähigkeiten, die von `NetCraft-ModApi` abgedeckt werden, sind verfügbar; für den Rest schreibe eigene Injektionsregeln oder warte, bis die API nachzieht.

### 3.3 Ein Gegenüberstellungsbeispiel

Fabric: eine Zeile protokollieren, wenn der Server startet, und einen Befehl registrieren.

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

Manifest (`ncmod.json`, als eingebettete Ressource):

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

Hinweis: Sogar ein leeres `hooks` funktioniert hier — Events wie `ServerEvents.Started` werden von den eigenen Probes von `NetCraft-ModApi` bereitgestellt, und dein Mod muss nur abonnieren (`ServerEvents` liegt unter `NetCraft.ModApi.Wrapper`, siehe [2.9](#29-zwei-routen-wrapper-schicht-und-erweiterungspunkte)). Du musst nur dann eigene Hook-Regeln schreiben, wenn du eine Stelle im Kernel hooken möchtest, für die ModApi noch kein Event bereitstellt.

---

## 4. Grundanforderungen an ein ncm

ncm bedeutet NetCraft-Mod. Ein ncm ist eine .NET-Klassenbibliothek-dll, die eine `ncmod.json` einbettet und im Verzeichnis `mods/` liegt.

Der einfachste Einstieg ist die Vorlage:

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

Wenn das lokale Paket noch nicht veröffentlicht wurde, verwende `dotnet new install <nupkg path>`, oder führe `dotnet pack` im Repository aus und installiere dann die Ausgabe.

`-e` nimmt `both` (Standard) / `server` / `client` und bestimmt das `environment` des Manifests sowie, welcher Abonnementcode der Seite in der Einstiegsklasse erzeugt wird. Die Vorlage liefert die NC-Referenz-Assemblies mit, sodass kein Projektverweis nötig ist, und id / entry von `ncmod.json` werden aus dem Projektnamen befüllt.

Ab 4.1 behandelt das Folgende, was ein handgeschriebenes Projekt erfüllen muss.

### 4.1 Projektdatei

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private muss deaktiviert werden, sonst werden die NC-Assemblies von EmbedDependencies in 4.6 in die Mod-dll eingebettet -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName` muss `ncmod.json` sein; der Scanner erkennt nur diesen Namen.

Die Vorlage nimmt eine andere Route: Assembly-Referenzen unter `libs/` (`Reference Include="libs\*.dll" Private="false"`). Folge dem, wenn du den NC-Quellcode nicht hast. Beiden Ansätzen gemeinsam ist, dass **NCs eigene dlls niemals ins Ausgabeverzeichnis gelangen dürfen** — `EmbedDependencies` aus [4.6](#46-drittanbieter-abhängigkeiten) bettet dlls von Drittanbietern aus dem Ausgabeverzeichnis in den Mod ein, und wenn NCs Assemblies ebenfalls eingebettet werden, gibt es zwei Sätze von Typidentität.

### 4.2 Felder von ncmod.json

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

| Feld | Erforderlich | Hinweise |
| --- | --- | --- |
| `id` | Ja | Mod-Bezeichner, von Abhängigkeiten und Nachschlagen verwendet. Eine leere Zeichenkette wird vom Scanner übersprungen |
| `version` | Empfohlen | Versionsnummer; wenn andere Mods davon abhängen, werden Versionsbeschränkungen daran beurteilt, siehe unten |
| `name` | Nein | Anzeigename; dies zeigt die Mod-Seite, standardmäßig fällt es auf `id` zurück |
| `description` | Nein | einzeilige Beschreibung |
| `authors` / `contributors` | Nein | Danksagungen, ein String-Array |
| `license` | Nein | Lizenzbezeichner |
| `contact` | Nein | externe Links; kann `homepage` / `sources` / `issues` annehmen |
| `icon` | Nein | der Name der eingebetteten Ressource des Symbols, siehe 4.7 |
| `environment` | Nein | `both` / `client` / `server`, Standard `both`. Wenn es nicht zur aktuellen Seite passt, wird der ganze Mod nicht geladen |
| `entry` | Ja | vollständiger Name der Einstiegsklasse; die Klasse muss `public Task Init()` haben |
| `depends` | Nein | andere Mods, von denen es abhängt, und deren erforderliche Versionen, siehe unten |
| `hooks` | Nein | Liste der Injektionsregeln; ein leeres Array bedeutet, nur Events zu abonnieren, die ModApi bereits hat |
| `mixins` | Nein | Liste der Mixin-Regeln; verschiebt Mitglieder einer der Klassen dieses Mods in den Zieltyp, siehe [2.8](#28-mixins-mitglieder-zu-einem-zieltyp-hinzufügen) |

`depends` deklariert andere Mods, von denen es abhängt, und deren erforderliche Versionen; der Schlüssel ist eine Mod-id und der Wert eine Versionsbeschränkung:

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**Abhängigkeiten, die nicht in `depends` geschrieben sind, tragen keine Versionsbeschränkung**. Zwischen-Mod-Abhängigkeiten werden bereits automatisch aus Compile-Zeit-Referenzen abgeleitet (siehe [6.1](#61-mod-abhängigkeit--injektion-des-abhängigen-mods)), und `depends` fügt nur Versionsbeschränkungen obendrauf hinzu. Wenn der abhängige Mod nicht auf der aktuellen Seite ist, wird die Prüfung übersprungen — dieser Fall wird der Assembly-Auflösung zur Meldung überlassen.

| Beschränkungssyntax | Bedeutung |
| --- | --- |
| `*` | jede Version, äquivalent zum Weglassen des Eintrags |
| `1.2.3` | Segmentpräfix; `1.2` passt auf `1.2` und `1.2.9`, aber nicht auf `1.3` |
| `^1.2.3` | gleiche Hauptversion und nicht kleiner als die Basis; wenn die Hauptversion 0 ist, wird stattdessen die Nebenversion verwendet, sodass `0.1` und `0.2` als inkompatibel gelten |
| `>=1.2.3` | nicht kleiner als die Basis |

Versionsnummern nehmen nur die führenden Ziffern jedes Segments, daher nimmt `26.2-netcraft` als `26.2` teil. Wenn Versionen nicht passen, wird **nur der deklarierende Mod übersprungen**, und der Rest lädt wie gewohnt; das Startprotokoll gibt „dependency X requires version …, actual version …" an.

Die Beurteilungsgrundlage ist das `version`-Feld im Manifest des abhängigen Mods. Also **muss ein Mod, der als Abhängigkeit dienen soll, `version` korrekt setzen** — eine leere Versionsnummer erfüllt keine spezifische Beschränkung.

### 4.3 Felder von Hook-Regeln

| Feld | Hinweise |
| --- | --- |
| `target` | vollständiger Name des Zieltyps; muss in einer Kernel-Assembly oder einer der Mod-Assemblies unter `mods/` liegen |
| `method` | Name der Zielmethode; gleichnamige Overloads passen alle |
| `type` | Injektionsform, siehe den Anhang |
| `patchMode` | Umsetzungsart, `ILRewrite` (Standard) oder `RuntimeInject`, siehe [2.6](#26-laufzeit-injektion-bereits-laufenden-code-ändern) |
| `ordinal` | wenn derselbe Anker an mehreren Stellen in der Host-Methode passt, wähle welche, 0-basiert, siehe [2.4](#24-auf-eine-stelle-eingrenzen-host-scoping-und-platzierung) |
| `replaceType` | vollständiger Name der Klasse, die die Ersatzmethode enthält |
| `replaceMethod` | Name der Ersatzmethode |
| `label` | Probe-Label, nur von `Mark` und `Probe` verwendet |
| `environment` | die Seite, für die die Regel gilt, Standard `both`; eine Regel, die auf einen Server-Typ abzielt und auf dem Client läuft, hat überhaupt kein Ziel und wird durch `environment` herausgefiltert |

Dieselbe Regel kann auch als Annotation auf der Ersatzmethode geschrieben werden; die Entsprechung ist:

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

| Manifest-Feld | Annotationsform |
| --- | --- |
| `target` | der erste Konstruktorparameter, geschrieben als `typeof(...)` |
| `method` | der zweite Konstruktorparameter, vorzugsweise `nameof(...)` |
| `type` | benannter Parameter `HookType`, Standard `CallSite` |
| `patchMode` | benannter Parameter `PatchMode`, Standard `ILRewrite` |
| `ordinal` | benannter Parameter `Ordinal` |
| `label` | benannter Parameter `Label` |
| `environment` | benannter Parameter `Environment`, Standard `both` |
| `replaceType` | nicht geschrieben; automatisch von der Klasse übernommen, die sie annotiert |
| `replaceMethod` | nicht geschrieben; automatisch von der Methode übernommen, die sie annotiert |

Die Annotationen stammen aus `InjectAttribute` in `NetCraft.ModApi.Extension`, daher müssen Mods, die Annotationen verwenden, es referenzieren. Wenn Annotationen und das Manifest beide existieren, werden sie zusammengeführt, und wenn derselbe Injektionspunkt in beiden deklariert ist, **gewinnt die Annotation**.

Mixin-Regeln verwenden einen anderen Satz von Feldern: `target` (vollständiger Name des Zieltyps), `source` (vollständiger Name des Quelltyps, der in der Assembly des Mods selbst liegen muss), `interfaces` (optional, ein Array vollständiger Interface-Namen); zur Semantik siehe [2.8](#28-mixins-mitglieder-zu-einem-zieltyp-hinzufügen). `[Mixin(typeof(target))]` auf der Quellklasse entspricht dem Manifest-Eintrag.

### 4.4 Was gehookt werden kann und was nicht

Gehookt werden können: Kernel-Assemblies unter `kernel/` und andere Mods unter `mods/`.

Nicht gehookt werden können:

- die Hauptbibliothek `NetCraft.dll`
- der Loader `NetCraft.ModLoader.dll`
- Einstiegs-Assemblies (`NetCraft.Server.Exe.dll` usw.)

Das `target` jedes Hooks muss im Kernel oder in einer Mod-Assembly auffindbar sein, andernfalls meldet die Assemblierung „the injection target is not in any known assembly". Beachte, dass **ein Namespace keine Assembly impliziert** — `NetCraft.Game.Server.DedicatedServer` liegt tatsächlich in `NetCraft.Server.dll`; der Loader schlägt es über einen aus Metadatentabellen aufgebauten Index nach, also schreibe einfach den vollständigen Namen.

Die Mod-Injektion von Mods folgt demselben System: Der Ziel-Mod wird **in dem Moment umgeschrieben, in dem er selbst geladen wird**, unabhängig von der Reihenfolge in `mods/`. Zyklische Regeln (A injiziert B und B injiziert A) melden zur Ladezeit „cyclic loading"; solche Regeln haben auf der Ebene der IL-Umschreibung keine Lösung, also entferne einfach eine davon.

Beachte, dass der injizierte Mod **nicht dein eigener Mod sein darf** — Selbst-Injektion zählt ebenfalls als Zyklus.

### 4.5 Bereitstellung

Kopiere die kompilierte dll in das `mods/` des Ausgabeverzeichnisses (oberste Ebene, keine Rekursion in Unterverzeichnisse) und starte den Prozess neu.

**Dieser Schritt ist bereits durch den Build automatisiert**: `DeployModToHosts` in `NetCraft.ModApi.csproj` kopiert die dll nach dem Build in das `mods/`-Verzeichnis des Ausgabeverzeichnisses jedes Host-Projekts. Die Host-Liste ist die `ModHostProjects`-Eigenschaft; wenn du dein eigenes Host-Projekt erstellst, füge einfach seinen Namen hinzu. Wenn keine Regel wirkt und überhaupt kein Fehler auftritt, prüfe zuerst, ob die dll im `mods/` des Host-Projekts veraltet ist.

Das Startprotokoll gibt aus:

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

Reihenfolge der Fehlersuche:

- Keine Regel wirkt: Prüfe zuerst, ob die dll in `mods/` veraltet ist (am häufigsten).
- Wenn du `Rewrote and preloaded assembly` nicht siehst, wurde die Ziel-Assembly bereits vor dem Bootstrap geladen, also wurde die Regel zu spät geschrieben.
- Wenn du `injection targets` siehst, aber das Ziel falsch ist, ist es meist ein falsch geschriebenes `target`; folge [4.4](#44-was-gehookt-werden-kann-und-was-nicht), um zu prüfen, zu welcher Assembly der Typ gehört.
- Eine Regel mit `environment` gleich `client` wird auf dem Server still übersprungen (eine Nichtübereinstimmung auf Mod-Ebene bedeutet, dass der ganze Mod nicht geladen wird); dies ist erwartetes Verhalten, kein Fehler.

### 4.6 Drittanbieter-Abhängigkeiten

Entspricht Jar-in-Jar von Fabric.

Projekte, die aus der Vorlage erstellt werden, **müssen sich darum nicht kümmern**: Über `dotnet add package` hinzugefügte Bibliotheken werden zur Buildzeit automatisch in die Mod-dll eingebettet, und wenn der Loader eine Assembly nicht auflösen kann, durchsucht er die eingebetteten `.dll`-Ressourcen des Mods.

```
dotnet add package Newtonsoft.Json
```

Das ist alles; `ncmod.json` benötigt keine Änderungen.

Ein handgeschriebenes Projekt muss das `EmbedDependencies`-Target aus der csproj der Vorlage kopieren oder selbst einbetten:

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

Abhängigkeiten werden nicht ins Manifest geschrieben; die Auflösung sieht nur die eingebetteten `.dll`-Ressourcen des Mods an, und es genügt, dass der Ressourcenname zum Assembly-Namen passt.

Drei Hinweise:

- **Abgeglichen wird nach Assembly-Namen**. Der Loader vergleicht den angeforderten Assembly-Namen: Nach Entfernen von `.dll` ist der Ressourcenname entweder gleich dem Assembly-Namen oder endet mit `.` + dem Assembly-Namen. Daher funktionieren sowohl `MyLib.dll` als auch das Standard-`MyProject.deps.MyLib.dll`.
- **Nur die eigenen eingebetteten Ressourcen des Mods werden erkannt**. Nicht eingebettete Bibliotheken können nicht aufgelöst werden und werden auch nicht anderswo gesucht.
- **Die Auflösungsreihenfolge ist Kernel zuerst**. `EmbeddedAssemblyLoader` sucht zuerst unter Kernel-Assemblies und eingebetteten Unterbibliotheken und greift erst dann auf Mods zurück, wenn nichts gefunden wird, daher sollten Mods keine Assemblies mit demselben Namen wie Kernel-Assemblies einbetten.

Das Target der Vorlage verwendet `WithMetadataValue`, um `.dll` zu filtern, statt `Condition` zu schreiben, weil die Template-Engine `Condition` in der `.csproj` zur Vorlagenzeit auswertet, wenn `%(...)` keinen Wert hat, und die ganze Zeile gelöscht würde.

### 4.7 Symbol und Anzeigeinformationen

Name, Beschreibung, Autoren, Links und Symbol werden alle in `ncmod.json` geschrieben und entsprechen `name` / `description` / `authors` / `contact` / `icon` in Vanillas `fabric.mod.json`.

**Das Symbol ist eine eingebettete Ressource**, keine externe Datei, und verwendet dieselbe Ressourcenbenennungskonvention wie eingebettete Abhängigkeiten:

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

Die Vorlage liefert bereits eine `icon.png` mit beiden Stellen konfiguriert; ersetze einfach dieses Bild. Ein PNG mit 64×64 oder 128×128 wird empfohlen.

Die Suchreihenfolge für das Symbol ist: worauf das `icon` des Manifests zeigt → eine eingebettete Ressource namens `icon.png` → wenn keines von beiden existiert, das Standardbild der UI (ein graues Fragezeichen).

Diese Informationen sind auf der **MODS**-Seite der Server-GUI sichtbar. Links ist die Mod-Liste (kleines Symbol + Anzeigename + Version + Status); rechts stehen Beschreibung, Danksagungen, Links, Abhängigkeiten, Initialisierungszeit und Injektionsregeln des ausgewählten Mods. Mod-Namen in der Abhängigkeitsspalte sind anklickbar und springen direkt zu jenem Eintrag.

Mods, die nicht geladen werden konnten oder übersprungen wurden, sind ebenfalls in der Liste und in der Statusspalte markiert — wenn du diagnostizierst, „warum hat mein Mod nicht gewirkt", prüfe zuerst diese Spalte.

---

## 5. Bekannte Einschränkungen

- **Probe-Signaturen dürfen nur BCL-Typen und `object` verwenden**, aus dem in 2.2 genannten Grund; Werttyp-Parameter und Rückgabewerte sind Ausnahmen und müssen ihre echten Typen behalten.
- **`CallSite` ist eine Ersetzung**, und die Ersatzmethode muss den Originalaufruf selbst wiederherstellen, siehe 2.3. Private Methoden können nicht wiederhergestellt werden und erfordern Reflexion.
- **dlls unter `mods/` werden automatisch von `DeployModToHosts` bereitgestellt**; wenn du dein eigenes Host-Projekt erstellst, denke daran, seinen Namen zu `ModHostProjects` hinzuzufügen.
- **In Einstiegs-Assemblies kann nicht injiziert werden**: Wenn dein Ziel und Hook in eine Einstiegs-Assembly fallen, ist es wirkungslos.
- **Die Hauptbibliothek und der Loader können nicht gehookt werden**; dies ist eine Design-Einschränkung, die verhindert, dass Mods den Ladeprozess selbst ändern.
- **Nicht passendes `environment` bedeutet, dass der ganze Mod nicht geladen wird**, nicht „einige Regeln schlagen fehl".
- **Laufzeit-Injektion erfordert die native Bibliothek**: Regeln, die `RuntimeInject` verwenden, erfordern, dass `lead_hook_native` beim Prozessstart angehängt wird, und der Loader startet sich selbst neu, um dies zu tun; wenn die Bibliothek nicht gefunden wird oder der Neustart fehlschlägt, wird dieser Regelsatz auf eine Warnung herabgestuft und der Start nicht blockiert. Zu Fähigkeitsgrenzen und Kosten siehe [2.6](#26-laufzeit-injektion-bereits-laufenden-code-ändern).
- **Annotationen haben weniger Felder als die C#-API**: `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` können nicht in `[Inject]` geschrieben werden (`PatchMode` und `Ordinal` werden unterstützt), siehe [2.1](#21-injektionsstile-annotationen-oder-manifest-eines-wählen).
- **Mixins gelten nur für die Ladezeit-Umschreibung**, und Mitglieder in der Quellklasse werden verschoben statt kopiert; verschachtelte Typen und generische Methoden in der Quellklasse liegen außerhalb der Abdeckung, und mit einem Interface eingemischte Methoden werden als virtual markiert. Siehe [2.8](#28-mixins-mitglieder-zu-einem-zieltyp-hinzufügen).
- Beim Testen und Debuggen ist, wenn Sprachdateien oder Modellressourcen verwendet werden, ein `assets`-Verzeichnis (aus dem Vanilla-jar extrahiert) erforderlich, andernfalls verschlechtern sich die betreffenden Funktionen zu Übersetzungsschlüsseln oder Platzhaltertexturen.

---

## 6. Bekannte Verhaltensweisen

Dieser Abschnitt ist eine Aufzeichnung beobachteten Verhaltens, keine Spezifikation.

### 6.1 Mod-Abhängigkeit + Injektion des abhängigen Mods

**Szenario**: b hängt von a ab und injiziert a außerdem.

**Fazit**: Es funktioniert und bildet keinen Zyklus.

Die Kette hat drei Schritte:

1. Die Regeltabelle wird rein durch Lesen von PE-Metadaten aufgebaut, ohne eine Assembly zu laden. Die Regel „b injiziert a" erfordert, dass weder a noch b vorhanden ist.
2. `PreloadReplacers` lädt alle Ersatzklassen (einschließlich b) vor `ModManager`, zu welchem Zeitpunkt a noch nicht geladen ist. `LoadFromStream` liest nur Metadaten und JIT-kompiliert keine Methodenkörper, daher ist b's Referenz auf a in diesem Moment lazy, und das Laden schlägt nicht fehl.
3. `ModManager` sortiert dann topologisch nach `AssemblyRef`, mit a vor b. Beim Umschreiben von a befindet sich die Ersatzklasse b bereits in `Default`, also wird der Typ direkt geholt und eine `AssemblyRef`, die auf b zeigt, zu den Metadaten von a hinzugefügt. **a muss überhaupt nicht wissen, dass b existiert.**

**Harte Anforderung**: Wenn du den injizierten Mod in der csproj referenzierst, musst du `Private="false"` schreiben. Standardmäßig wird `a.dll` ins Ausgabeverzeichnis kopiert, die dann von `EmbedDependencies` als eingebettete Abhängigkeit in `b.dll` eingebettet wird, und zur Laufzeit greift `ModLibs` beim Auflösen von `a` auf die zweite Kopie zu, wodurch die Typidentitätsprüfungen zwischen den beiden fehlschlagen.

**Verifizierungsfall**: zwei Vorlagenprojekte `NetCraft.Test1` (a) und `NetCraft.Test2` (b); a stellt `Test1Api.Greet` bereit und ruft es in seinem eigenen `ModEntry.Server` auf, während b diese Aufrufstelle durch `Test2Probe.OnGreet` ersetzt und `Greet` außerdem einmal in b's `ModEntry.Server` als Kontrolle aufruft. Tatsächliches Laufprotokoll:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← die Injektion hat gewirkt
Test2 internal greeting: Test1 original greeting test2    ← Kontrollgruppe: die Regel schreibt nur die Ziel-Assembly um, daher bleibt b's eigene interne Aufrufstelle unberührt
```

Abhängigkeiten leiten die Reihenfolge nur aus `AssemblyRef` ab und tragen keine Versionsbeschränkung. Um Versionen zu beschränken, deklariere sie im `depends` des Manifests, siehe [4.2](#42-felder-von-ncmodjson).

### 6.2 Annotations-Injektion

**Szenario**: `[Inject(typeof(X), nameof(X.M))]` auf der Ersatzmethode, ohne Regel in `ncmod.json`.

**Fazit**: Es funktioniert und benötigt keine Manifest-Deklaration. Annotationen und das Manifest teilen eine Quelle, werden bei der Assemblierung zusammengeführt, und die Annotation gewinnt für denselben Injektionspunkt.

**Warum keine Deklaration nötig ist**: Annotationen sind nur die `CustomAttribute`-Tabelle in den Metadaten, und der Loader liest bereits dieselben Metadaten beim Scannen von Mods (Manifest, eingebettete Ressourcen und AssemblyRef — drei Elemente), also führt das Lesen einer weiteren Tabelle zu keiner neuen Lade- oder Zeitbeschränkung. Die einzige Voraussetzung ist, dass der Mod `NetCraft.ModApi` referenziert (den Host der Annotationen).

**Die Voraussetzung ist statisches Lesen**: Das Lesen von Annotationen muss über `MetadataReader` erfolgen und **darf nicht `Assembly.Load` + `GetCustomAttributes` verwenden** — Letzteres zieht die Mod-Assembly herauf, nur um Regeln zu lesen, und das Umschreibungsfenster ist auf der Stelle weg.

**`typeof` stellt keine Typreferenz dar**: Was `typeof(X)` in den Parameter kompiliert, ist der serialisierte Name des Typs (`full name, assembly, Version=…`), der sich zu bloß einer Zeichenkette auflöst und nicht erfordert, dass `X` vorhanden ist. Daher verstößt das Schreiben von `typeof(injected-mod)` in einer Annotation **nicht** gegen die Einschränkung in 2.2, dass „eine Ersatzklasse nicht die Typen des injizierten Mods referenzieren darf" — ein Name, der in Metadaten erscheint, und ein Typ, der zur Laufzeit aufgelöst wird, sind zwei verschiedene Dinge.

**Verifizierungsfall**: `NetCraft.Test2` gegen zwei Methoden von `NetCraft.Test1`; `Greet` läuft über die Annotation und `Farewell` über das Manifest. Beide Regeln werden installiert und beide Aufrufstellen werden ersetzt:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← Annotation
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← Manifest
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← Kontrollgruppe: die Regel schreibt nur die Ziel-Assembly um
Test2 internal farewell: Test1 original farewell test2     ← Kontrollgruppe
```

Derselbe Fall verifizierte nebenbei auch, dass der benannte Parameter der Annotation (`Environment = "server"`) ebenfalls aufgelöst wird.

### 6.3 Die Leistungskosten beim Anhängen des Profilers

**Szenario**: Dasselbe reine Rechenprogramm (einhundert Millionen Modulo-und-Akkumulations-Iterationen), einmal ohne Profiler und einmal mit angehängtem Profiler ausgeführt (die drei `CORECLR_ENABLE_PROFILING`-Umgebungsvariablen). Jeweils fünf Läufe.

**Fazit**: Kein Unterschied im eingeschwungenen Zustand; die Kosten liegen vollständig beim Start.

| | Gesamtprozesszeit (5 Läufe, ms) | Rechenzeit im Programm |
| --- | --- | --- |
| ohne | 306 / 260 / 278 / 253 / 300 | 225 ms |
| mit | 406 / 370 / 340 / 325 / 354 | 193 ms |

Die Gesamtzeit ist etwa 80–110 ms länger. Die Quelle ist `COR_PRF_DISABLE_ALL_NGEN_IMAGES` — das Aktivieren von ReJIT erfordert gleichzeitig das Deaktivieren von ReadyToRun-Images, sodass Framework-Code nur über JIT laufen kann; die zeitgesteuerte Schleife innerhalb des Programms ist einmal JIT-kompiliert identisch und zeigt keinen Unterschied (der Lauf mit dem Profiler war sogar etwas schneller, was Rauschen ist).

**Eine nebenbei behobene Verschwendung**: Die ursprüngliche Implementierung abonnierte `COR_PRF_MONITOR_JIT_COMPILATION`, einen nativen Callback, der nach jeder Methodenkompilierung ausgelöst wird, den wir nie verwendet haben. Nach dem Entfernen änderte sich die Event-Maske von `0x80040024` auf `0x80040004`; die obige Tabelle enthält die Daten nach der Entfernung.

**Verifizierungsfall**: `__hookverify/BenchProbe`.

### 6.4 Zwei Mods, die dasselbe Ziel injizieren

**Szenario**: Zwei Mods deklarieren jeweils eine Regel, die dieselbe Zielmethode trifft (`MethodBody`-Form, unterschiedliche Ersatzmethoden).

**Fazit**: Kein Fehler, kein Absturz; **die zuerst assemblierte Regel gewinnt und die spätere schlägt still fehl**.

Beide Regeln gelangen in die Regeltabelle — es wird keine mod-übergreifende Deduplizierung durchgeführt. Beim Umschreiben wird der **erste** Eintrag in der gleichschlüsseligen Liste für `OriginalType::OriginalMethod` genommen; Host-Scoping verhält sich genauso, wobei `InType`/`InMethod` „der erste Treffer gewinnt" bedeutet. Welcher zuerst kommt, hängt von der Assemblierungsreihenfolge ab, und die Assemblierungsreihenfolge stammt aus der Enumerationsreihenfolge des Mods-Verzeichnisses; **es gibt kein Prioritätsfeld und es kann nicht durch das Deklarieren von Abhängigkeiten gesteuert werden** (Abhängigkeiten beeinflussen nur die Reihenfolge von `Init()`, nicht die Assemblierung der Injektionsregeln).

**Auch die Laufzeit-Injektion gilt nach dem Prinzip wer zuerst kommt**: Eine spätere Registrierungsanfrage wird normal gesendet, aber `GetReJITParameters` beansprucht nach „Modul + Methode" und passt immer auf die erste Anfrage, sodass die spätere Registrierung nicht greift. Gemessen bleibt nach einer zweiten Injektion das Verhalten der Zielmethode beim ersten Ergebnis.

**Eine schlechte Regel beeinflusst die anderen nicht**: Probleme wie der Zieltyp in keiner bekannten Assembly oder eine falsch geschriebene Injektionsform werden zur Assemblierungszeit in `ModHooks.Errors` aufgezeichnet und dieser Eintrag wird übersprungen, während die Regeln anderer Mods wie gewohnt assembliert werden.

**Konflikte werden aufgezeichnet**: Wenn die Assemblierung denselben Injektionspunkt von mehreren Mods deklariert erkennt, wird der später assemblierte in `ModHooks.Warnings` geschrieben und im Startprotokoll als `Mod injection conflict ...` ausgegeben, wobei benannt wird, welche zwei Mods kollidierten und welche nicht wirksam wird. Es ist eine Warnung, kein Fehler, beeinträchtigt das Laden nicht, und die Regel selbst bleibt in der Tabelle (nur unerreichbar).

**Auch die Laufzeit-Injektion durchläuft diese Prüfung**: Der Konflikterkennungsschlüssel enthält `patchMode`, also zählt ein Mod, der `ILRewrite` schreibt, und ein anderer, der `RuntimeInject` schreibt, nicht als Kollision (zwei unabhängige Pfade, jeder macht sein eigenes Ding); nur zwei desselben Modus werden als konfliktär beurteilt und gewarnt. Ihr tatsächliches Greifen ist ebenfalls nach dem Prinzip wer zuerst kommt — `GetReJITParameters` beansprucht nach „Modul + Methode" und passt auf die erste Anfrage, sodass spätere gesendet werden, aber nicht greifen.

**Der eine harte Absturzpunkt**: Wenn zwei Mods dieselbe Methode im `RuntimePatch`-Modus patchen, trifft der zweite auf die Duplikatregistrierungsprüfung von `RuntimeHookEngine` und wirft `InvalidOperationException`, und dieser Pfad wird nicht abgefangen, sodass der Start schlicht fehlschlägt. `RuntimePatch` wird auslaufen (siehe [2.6](#26-laufzeit-injektion-bereits-laufenden-code-ändern)); verwende es nicht in neuen Regeln.

**Verifizierungsfall**: das `modinjection`-Modul von `NetCraft.Test`, Einträge `same anchor first mod wins quietly` und `one bad rule does not sink the rest`.

### 6.5 Noch nicht verifiziert

- **Laufzeit-Injektion, die auf einem echten Server funktioniert**: Der `RuntimeInject`-Modus wurde in `__hookverify/RuntimeProbe` durchgängig verifiziert (nach der Registrierung wird das Verhalten der Zielmethode auf das der Ersatzmethode umgestellt), und die Mod-Assembly-Seite hat ebenfalls Testabdeckung für Routing und Herabstufung; aber kein Build-Schritt platziert derzeit `lead_hook_native` in NCs Ausführungsverzeichnis, sodass das Ausführen dieser Kette auf einem echten Server zuerst erfordert, die Bibliothek in den Programmstamm zu legen (oder per `NC_PROFILER_PATH` darauf zu verweisen). Dieser Schritt ist nicht erledigt.
- **Eine Ersatzklasse, die die Typen des injizierten Mods referenziert**: Laut Überlegung würde sie im Moment des Umschreibens von a das a auflösen, während a unmittelbar vor Abschluss des Ladens feststeckt (es ist noch nicht in `Default`, und weder `ModLibs` noch der Kernel-Auflösungs-Callback erkennen Mod-Assemblies), sodass `PrepareMod` voraussichtlich wirft und in `result.Errors` aufzeichnet. Bisher nicht tatsächlich ausgeführt. Beachte, dass 6.2 nur beweist, dass **`typeof` in einer Annotation** keine Referenz ist; **der in einer Methodensignatur erscheinende Typ** ist eine andere Sache.

---

## Anhang: HookType-Übersicht

| Form | Wirkung | Anforderung an die Signatur der Ersatzmethode |
| --- | --- | --- |
| `CallSite` | Aufrufstellen zur Zielmethode durch deine Methode ersetzen | Parameteranzahl passt zur aufgerufenen Methode (Instanzaufruf +1) |
| `MethodBody` | den gesamten Zielmethodenkörper ersetzen | passt zur ersetzten Methode |
| `NewObj` | `new X(...)` ersetzen | Parameteranzahl passt zum Konstruktor |
| `FieldRead` | Feldlesevorgänge instrumentieren | nach dem Lesetyp |
| `FieldWrite` | Feldschreibvorgänge instrumentieren | nach dem Schreibtyp |
| `TypeCheck` | `isinst` / `castclass` instrumentieren | nach dem geprüften Typ |
| `Box` | Boxing/Unboxing instrumentieren | nach dem Elementtyp |
| `FunctionPointer` | Laden von Funktionszeigern instrumentieren | nach dem Delegattyp |
| `LocalRead` | Lesen lokaler Variablen instrumentieren | null Parameter, gibt den Wert der Variablen zurück |
| `LocalWrite` | Schreiben lokaler Variablen instrumentieren | ein Parameter, empfängt den geschriebenen Wert |
| `Constant` | Laden von Konstanten instrumentieren | null Parameter, gibt den Wert der Konstante zurück |
| `Probe` | den Originalmethodenkörper beibehalten und Ein- und jeden Ausstieg instrumentieren; mit `LabelArgumentIndex` kann ein Argument in das Label eingefügt werden | `Begin()` gibt long zurück, `End(string, long)` |
| `Mark` | einmalig nur beim Methodeneintritt melden, ohne Zeitmessung | `void method(string label)` |

`Probe` und `Mark` übergeben nur den Labeltext (`Probe` kann auch das `ToString()` eines Arguments einschließen); sie können keine Objektreferenzen erhalten. Um tatsächliche Argumente zu erhalten, verwende `CallSite`.

`InType`/`InMethod`, `Placement` und `Ordinal` (siehe [2.4](#24-auf-eine-stelle-eingrenzen-host-scoping-und-platzierung)) sind nur für Instruktionsformen sinnvoll: Die zehn Einträge in der obigen Tabelle außer `MethodBody`, `Probe` und `Mark` können Ersetzen oder Einfügen davor/danach wählen und `Ordinal` verwenden, um ein einzelnes Vorkommen auszuwählen; `MethodBody` ersetzt immer das Ganze, und `Probe`/`Mark` ignorieren diese Parameter.

Bei den drei Arten `LocalRead` / `LocalWrite` / `Constant` wird die Host-Methode in `target` statt in einer referenzierten Entität geschrieben, und es ist zusätzlich `localIndex` oder `constantValue` erforderlich; siehe [2.5](#25-anker-im-methodenkörper-lokale-variablen-und-konstanten).
