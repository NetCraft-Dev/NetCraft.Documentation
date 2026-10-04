# NetCraft-ModApi-Referenz

`NetCraft.ModApi` ist die API-Oberfläche, die NetCraft für Mods bereitstellt. Sie hat zwei Identitäten: für dich ist sie eine API-Bibliothek; für sich selbst ist sie ein gewöhnlicher Mod (`id` ist `netcraft-modapi`, der seine eigene `ncmod.json` und Injektions-Probes mitbringt).

Diese Datei wächst mit der API. Für architektonischen Hintergrund, Unterschiede zu Fabric und das Schreiben eines Mods siehe [modding-guide-de.md](modding-guide-de.md).

- Assembly: `NetCraft.ModApi.dll`

- Abhängigkeiten: `NetCraft` (die Hauptbibliothek), `NetCraft.Game`

Die öffentliche Schnittstelle ist in drei Namespaces aufgeteilt:

| Namespace | Inhalt | Hinweise |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | Event- und Abonnement-Basisklasse `NcEvent<T>`, `Nc*`-Fassaden, `Nc*`-Objekthandles | Wrapper-Schicht; keine Kernel-Typen in der öffentlichen Schnittstelle |
| `NetCraft.ModApi.Extension` | `[Inject]`- / `[Mixin]`-Annotationen | Erweiterungspunkte; Regeln binden an Kernel-Klassen- und -Methodennamen |
| `NetCraft.ModApi.Internal` | Injektions-Probes | Nicht direkt referenzieren |

Der Wurzel-Namespace `NetCraft.ModApi` enthält nur die Einstiegsklasse `ModApiEntry`. `Wrapper` und `Extension` sind zwei parallele Routen; wie man wählt, steht in [modding-guide-de.md 2.9](modding-guide-de.md#29-zwei-routen-wrapper-schicht-und-erweiterungspunkte).

***

## 1. Schnellstart

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //das Abonnement gibt ein Handle zurück; das Verwerfen hebt die Registrierung auf
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //du kannst mit dem Handle auch etwas anderes tun
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

Richte in `ncmod.json` `entry` auf diese Klasse und lasse `hooks` leer — die untenstehenden Events werden alle von ModApis eigenen Probes bereitgestellt.

***

## 2. Events

Alle Events liegen unter `NetCraft.ModApi.Wrapper`; nach `using NetCraft.ModApi.Wrapper;` stehen sie zur Verfügung.

### 2.1 Übersichtstabelle

| Event                           | Args-Typ               | Auslöser                                                  | Seite  | ModApi-Hook-Punkt                                        |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick`             | `ServerTickArgs`       | jeder Tick der Server-Hauptschleife                       | server | `DedicatedServer::Tick` (Mark)                           |
| `ServerEvents.Started`          | `ServerPhaseArgs`      | Hauptschleife gestartet, nachdem `Done (x.xxxs)!` ausgegeben wurde | server | `MinecraftServer::Run` (Mark)                    |
| `ServerEvents.Stopping`         | `ServerPhaseArgs`      | Server beginnt herunterzufahren; Spieler werden gleich getrennt | server | `DedicatedServer::Stop` (Mark)                     |
| `ServerEvents.CommandRegister`  | `CommandRegisterArgs`  | alle eingebauten Befehle wurden registriert               | server | `EffectCommand::Register`-Aufrufstelle (CallSite)        |
| `ServerEvents.PlayerJoin`       | `PlayerJoinArgs`       | die Beitritts-Paketsequenz wurde gesendet                 | server | `PlayerList::PlaceNewPlayer`-Aufrufstelle (CallSite)     |
| `ServerEvents.PlayerLeave`      | `PlayerLeaveArgs`      | Spieler aus der Online-Liste entfernt                     | server | `PlayerList::RemovePlayer`-Aufrufstelle (CallSite)       |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | Trenn-Paket gesendet und Verbindung geschlossen           | server | `ServerPlayer::Disconnect`-Aufrufstelle (CallSite)       |
| `ServerEvents.PlayerHurt`       | `PlayerHurtArgs`       | Schaden tatsächlich zugefügt                              | server | `PlayerList::HurtPlayer`-Aufrufstelle (CallSite)         |
| `ServerEvents.PlayerDeath`      | `PlayerDeathArgs`      | unmittelbar nachdem die Gesundheit auf null zurückgesetzt wird | server | `PlayerList::RespawnPlayer`-Aufrufstelle (CallSite) |
| `ServerEvents.PlayerChat`       | `PlayerChatArgs`       | nach der Chat-Übertragung                                 | server | `ServerGamePacketListenerImpl::HandleChat`-Aufrufstelle (CallSite) |
| `ServerEvents.ChunkLoaded`      | `ChunkLoadedArgs`      | Chunk gelangt erstmals in den Speicher                    | server | `ServerChunkCache::set_ChunkLoaded`-Zuweisungsstelle (CallSite) |
| `ServerEvents.ChunkUnloaded`    | `ChunkUnloadedArgs`    | Chunk verlässt den Speicher                               | server | `ServerChunkCache::set_ChunkUnloaded`-Zuweisungsstelle (CallSite) |
| `ServerEvents.ChunkSaved`       | `ChunkSavedArgs`       | Snapshot wird erstellt, bevor der Chunk auf die Platte geschrieben wird | server | `ServerChunkCache::set_ChunkSaveSink`-Zuweisungsstelle (CallSite) |
| `ServerEvents.CommandExecuted`  | `CommandExecutedArgs`  | ein Befehl ist fertig ausgeführt; Syntaxfehler und Berechtigungsverweigerungen zählen ebenfalls | server | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick`        | `LevelTickArgs`        | Level-Tick, einmal pro geladenem Level und Tick           | server | `PersistentServerLevel::Tick` (CallSite)                 |
| `ServerEvents.SavedDataSaving`  | `SavedDataSavingArgs`  | gespeicherte Daten auf die Platte geschrieben, einen Schritt später als das Chunk-Speichern | server | `SavedDataStorage::ScheduleSave` (CallSite) |
| `ServerEvents.BlockChanged`     | `BlockChangedArgs`     | Blockzustand geändert, wird gleich an Clients synchronisiert | server | `IBlockUpdateSink::BlockChanged` (CallSite)           |
| `ServerEvents.BlockBroken`      | `BlockBrokenArgs`      | Block abgebaut; Spieler-Abbau und Redstone-Selbstzerstörung zählen beide | server | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped`      | `ItemDroppedArgs`      | fallengelassenes Item-Entity erzeugt, einschließlich Block-Abbau-Drops und Kochprodukte | server | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived`  | `PacketReceivedArgs`   | jedes eingehende Paket, das einem Handler zugeordnet wird, einschließlich Handshake- und Status-Phase | both   | `PacketProcessor::ScheduleIfPossible` und `HandleNow` (CallSite) |
| `ClientEvents.Tick`             | `ClientTickArgs`       | jeder Tick der Client-Hauptschleife                       | client | `MinecraftClient::Tick` (Mark)                           |

Reihenfolge der Spieler-Events: Der Tod ist in den Hurt-Ablauf eingebettet, daher geht `PlayerDeath` dem zugehörigen `PlayerHurt` voraus; `PlayerLeave` und `PlayerDisconnect` sind zwei verschiedene Dinge — Ersteres bedeutet das Entfernen aus der Online-Liste (auch nach `/kick` wird es erst ausgelöst, wenn die Verbindung abbricht), Letzteres bedeutet den Verbindungsabbruch selbst, und die beiden treten nicht garantiert paarweise auf.

### 2.2 Abonnieren und die Registrierung aufheben

```csharp
IDisposable Subscribe(Action<T> handler)
```

- Mehrfaches Abonnieren desselben Events liefert Benachrichtigungen in der Reihenfolge des Abonnierens.

- Die Verteilung erstellt einen Snapshot der Callback-Liste, daher beeinflusst Abonnieren oder Aufheben der Registrierung innerhalb eines Callbacks die aktuelle Verteilung nicht.

- Ohne Aufhebung der Registrierung bleibt es für immer wirksam; Mods bieten keinen Entlademechanismus, daher ist manuelles Abmelden normalerweise unnötig.

### 2.3 Args-Typen

**`ServerTickArgs`**

| Eigenschaft | Typ    | Hinweise                             |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | Tick-Zähler seit diesem Start, beginnend bei 1 |

Beachte, dass dies von ModApi selbst gezählt wird, nicht vom `TickCount` des Kernels.

**`ClientTickArgs`**

| Eigenschaft | Typ    | Hinweise                           |
| ----------- | ------ | ---------------------------------- |
| `TickCount` | `long` | wie oben, auf der Client-Seite unabhängig gezählt |

**`ServerPhaseArgs`**

| Eigenschaft | Typ      | Hinweise                     |
| ----------- | -------- | ---------------------------- |
| `Phase`     | `string` | Phasenname, `started` oder `stopping` |

Das Feld dupliziert das Event selbst; es wird beibehalten, damit die Protokollierung ein einheitliches Format verwenden kann.

**`CommandRegisterArgs`**

| Mitglied                             | Typ                                     | Hinweise         |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher`                         | `CommandDispatcher<CommandSourceStack>` | der Befehls-Dispatcher des Kernels |
| `Register(name, description, build)` | Methode                                 | registriert einen Befehl und trägt ihn ins Register ein, siehe 3.1 |

**`PlayerJoinArgs`**

| Eigenschaft   | Typ            | Hinweise                                       |
| ------------- | -------------- | ---------------------------------------------- |
| `Player`      | `ServerPlayer` | der gerade beigetretene Spieler; Beitrittspakete bereits gesendet, der Zustand kann sicher gelesen werden |
| `ProfileName` | `string`       | Spielername                                    |

**`PlayerLeaveArgs`**

| Eigenschaft | Typ            | Hinweise                              |
| ----------- | -------------- | ------------------------------------- |
| `Player`    | `ServerPlayer` | der verlassende Spieler, zu diesem Zeitpunkt nicht mehr in der Online-Liste |
| `Removed`   | `bool`         | ob tatsächlich entfernt; `false` beim wiederholten Entfernen |

**`PlayerDisconnectArgs`**

| Eigenschaft | Typ            | Hinweise                              |
| ----------- | -------------- | ------------------------------------- |
| `Player`    | `ServerPlayer` | der getrennte Spieler                 |
| `Reason`    | `string`       | Trennungsgrund; Klartext, wenn als Komponente angegeben |

Das Trenn-Paket wurde gesendet und die Verbindung geschlossen; das Senden von Paketen an diesen Spieler hat jetzt keine Wirkung.

**`PlayerHurtArgs`**

| Eigenschaft | Typ             | Hinweise                                     |
| ----------- | --------------- | -------------------------------------------- |
| `Player`    | `ServerPlayer`  | der Spieler, der verletzt wurde              |
| `Attacker`  | `ServerPlayer?` | der Spieler, der den Schaden zugefügt hat; `null` bei Umgebungs- und Befehlsschaden |
| `Amount`    | `float`         | Schadensmenge diesmal                        |

Wird während Unverwundbarkeitsframes oder nach dem Tod nicht ausgelöst (das `Hurt` des Kernels gibt `false` zurück).

**`PlayerDeathArgs`**

| Eigenschaft | Typ             | Hinweise                       |
| ----------- | --------------- | ------------------------------ |
| `Player`    | `ServerPlayer`  | der gestorbene Spieler         |
| `Attacker`  | `ServerPlayer?` | der Mörder; `null`, wenn keiner vorhanden ist |

Der Kernel setzt unmittelbar zurück, nachdem die Gesundheit null erreicht, daher ist der Spieler beim Auslösen des Events bereits mit voller Gesundheit am Respawn-Punkt; die Koordinaten und Drops im Moment des Todes sind nicht verfügbar.

**`PlayerChatArgs`**

| Eigenschaft  | Typ      | Hinweise             |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | Name des Absenders   |
| `Message`    | `string` | Klartextnachricht    |

Dieses Event ist eine **Nur-Lese-Benachrichtigung**: Die Originalmethode hat die Nachricht bereits übertragen, daher hat eine Änderung von `Message` hier keine Wirkung.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

Die drei Events haben dieselbe Args-Form:

| Eigenschaft | Typ   | Hinweise           |
| ----------- | ----- | ------------------ |
| `X`         | `int` | Chunk-Koordinate X |
| `Z`         | `int` | Chunk-Koordinate Z |

Am leichtesten falsch zu machen ist `ChunkSaved`: Der Kernel verlangt, dass dieser Callback **den Snapshot synchron erstellt**, während Serialisierung und Schreiben auf die Platte vom Kernel selbst asynchron erledigt werden. Daher verlangsamt zeitaufwändige Arbeit in diesem Callback direkt das Entladen von Chunks, und destruktive Operationen (Blöcke entfernen, Inventare ändern) gehören ebenfalls nicht hierher — er verspricht nur den Moment des Snapshots.

Wenn `ChunkUnloaded` ausgelöst wird, wurden die Block-Entities bereits zusammen mit dem Chunk aufgeräumt; wenn du Blöcke lesen möchtest, verwende `ChunkSaved` (das ebenfalls nicht auf Blöcke zugreifen kann) oder einen früheren Zeitpunkt.

**`CommandExecutedArgs`**

| Eigenschaft | Typ                    | Hinweise                              |
| ----------- | ---------------------- | ------------------------------------- |
| `Command`   | `string`               | Rohtext des Befehls; Chat-Befehle haben keinen führenden Schrägstrich |
| `Result`    | `int`                  | Rückgabewert des Befehls; 0 bedeutet Fehler oder Verweigerung |
| `Source`    | `CommandSourceStack?`  | Befehlsquelle; `null` auf dem Spieler-Overload-Pfad |
| `Player`    | `ServerPlayer?`        | der Spieler, der den Befehl ausgegeben hat; `null`, wenn von der Konsole ausgegeben |

Das Event wird ausgelöst, **nachdem** der Befehl beendet ist; es kann die Ausführung nicht ändern. Syntaxfehler und Berechtigungsverweigerungen laufen ebenfalls hier durch; verwende `Result`, um sie zu unterscheiden. Befehle, die ein Spieler über die Chatleiste sendet, durchlaufen den Overload `Execute(ServerPlayer, string)`, bei dem der Kernel die Befehlsquelle intern aufbaut, sodass in diesem Fall `Source` `null` ist und nur `Player` gesetzt ist.

**`LevelTickArgs`**

| Eigenschaft    | Typ                     | Hinweise                                     |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level`        | `NcLevel`               | das in diesem Tick fortgeschrittene Level    |
| `RunsNormally` | `bool`                  | ob es normal fortgeschritten ist; `false` während `/tick freeze` |

Wird einmal pro geladenem Level und Tick ausgelöst, daher erhält eine Welt mit mehreren Levels mehrere pro Tick. Der Auslösepunkt liegt, nachdem der Level-Tick **abgeschlossen ist**; er ist ein Beobachtungspunkt, kein Abfangpunkt.

**`SavedDataSavingArgs`**

| Eigenschaft | Typ                 | Hinweise                    |
| ----------- | ------------------- | --------------------------- |
| `Storage`   | `SavedDataStorage`  | die persistierte Tabelle gespeicherter Daten |

Dies ist ein anderer Pfad als `ServerEvents.ChunkSaved`: Bei den Chunks wird nur ein Snapshot erstellt und asynchron geschrieben, während dieses Event ausgelöst wird, nachdem ein synchrones Schreiben abgeschlossen ist. Weltuhr, Spielregeln und Weltgrenzen-Daten laufen hierüber.

**`BlockChangedArgs`**

| Eigenschaft | Typ          | Hinweise                    |
| -------- | ------------ | --------------------------- |
| `Pos`    | `BlockPos`   | Position des geänderten Blocks |
| `State`  | `BlockState` | Blockzustand nach der Änderung  |

Der Zustand wurde bereits in den Chunk geschrieben und wird gleich an Clients synchronisiert, daher kann die Änderung selbst hier nicht modifiziert werden. Redstone-Komponenten, die in Verhaltens-Callbacks ihren eigenen Zustand ändern, gehen ebenfalls über diesen Pfad hinaus; er ist hochfrequent, also verrichte keine zeitaufwändige Arbeit im Callback.

**`BlockBrokenArgs`**

| Eigenschaft | Typ             | Hinweise                                   |
| -------- | --------------- | ------------------------------------------ |
| `Pos`    | `BlockPos`      | Position des abgebauten Blocks              |
| `Player` | `ServerPlayer?` | der Abbaucher; `null` bei Nicht-Spieler-Ursachen wie Redstone |

Wird nur ausgelöst, wenn der Block tatsächlich ersetzt wird; leere Positionen und abgelehnte Abbaumaßnahmen lösen es nicht aus. Abbau-Effekte und Drops wurden bereits behandelt, daher liest du im Event das Ergebnis.

**`ItemDroppedArgs`**

| Eigenschaft | Typ         | Hinweise                |
| -------- | ----------- | ----------------------- |
| `Pos`    | `BlockPos`  | wo das fallengelassene Item erschienen ist |
| `Stack`  | `ItemStack` | der Stapel des fallengelassenen Items |

Block-Abbau-Drops und Lagerfeuer-Kochprodukte laufen beide hier durch. Ein leerer Item-Stapel erzeugt kein Entity, daher gibt es kein Event.

**`PacketReceivedArgs`**

| Eigenschaft     | Typ      | Hinweise                 |
| --------------- | -------- | ------------------------ |
| `Listener`      | `object` | der Listener, der dieses Paket empfängt |
| `Packet`        | `object` | das Paketobjekt selbst   |
| `IsServerbound` | `bool`   | ob es ein servergebundenes Paket ist |

Wird für jedes eingehende Paket ausgelöst und deckt alle vier Phasen ab: Handshake, Status, Konfiguration und Play. Bewegungs-Pakete kommen mehrmals pro Tick an, also verrichte keine zeitaufwändige Arbeit im Callback. Das Paket ist bereits in ein Objekt dekodiert, aber noch nicht in die Geschäftsschicht gelangt; um Typen zu unterscheiden, untersuche `Packet` selbst. Ausgehende Pakete liegen außerhalb des Geltungsbereichs dieses Events.

***

## 3. Erweiterungspunkte

### 3.1 Befehlsregistrierung

Der Zeitpunkt ist `ServerEvents.CommandRegister`. Zwischenspeichere die Args dieses Events nicht; der interne Befehlsbaum wird nur einmal beim Start aufgebaut.

```csharp
public void Register(
    string name,                                          //Befehls-Literal, ohne den Schrägstrich
    string description,                                   //einzeilige Beschreibung, angezeigt im /ncmapi-Register
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //Argumente und den Executor anhängen
```

Das `build`, das du erhältst, ist der Brigadier-Builder des Kernels; schreibe Argumente, Unterbefehle und Berechtigungsprädikate auf die Art des Kernels:

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //Berechtigungsprädikat
            .Executes(context => { /* ... */ return 1; })));
```

Mit Argumenten:

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //zur Veranschaulichung
                return 1;
            })));
```

Wichtige Punkte:

- Ein direkter Aufruf von `args.Dispatcher.Register(...)` installiert ebenfalls einen Befehl, aber er gelangt nicht ins Register und `/ncmapi` zeigt ihn nicht an. Verwende `args.Register`, wenn du ihn aufgeführt haben möchtest.

- Befehle haben standardmäßig keine Berechtigungsbeschränkung; füge bei Bedarf selbst `.Requires(...)` hinzu.

- Das Verhalten zur Ausführungszeit liegt ganz bei dir; ModApi fängt es nicht ab.

### 3.2 Registrierte Befehle anzeigen

Es gibt ein eingebautes `/ncmapi`, das Berechtigungsstufe 2 erfordert:

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

Verwendungszeilen werden zur Laufzeit aus der Knotenstruktur des Befehlsbaums berechnet: Literale werden mit Namen geschrieben, Argumente in spitze Klammern gesetzt, und Zwischenknoten, die selbst ausführbar sind, erhalten ihre eigene Zeile.

***

## 4. Server-Fassaden

Die Fassaden in diesem Kapitel liegen alle unter `NetCraft.ModApi.Wrapper`; nach `using NetCraft.ModApi.Wrapper;` stehen sie zur Verfügung.

Fassaden sind statische `Nc*`-Klassen, die über den Kernel verstreute Fähigkeiten an einigen Einstiegspunkten bündeln. Die Kernel-Instanz wird von einer Probe erfasst, wenn die Hauptschleife startet; wenn `NcServer.IsAvailable` false ist, wirft alles unten — verwende sie nur innerhalb von Event-Callbacks, nicht aus `Init`.

| Fassade | Zweck |
| --- | --- |
| `NcServer` | Server-Instanz, Tick-Rate, Befehle, Entity-Tracking, Spielerdaten, Spielregeln, Broadcast, Befehlsausführung |
| `NcPlayers` | Abfragen und Operationen zu Online-Spielern (Kick, Teleport, Gesundheit, Spielmodus, Berechtigungen) |
| `NcWorld` | Lesen/Schreiben und Abbauen von Blöcken in der Oberwelt, Wetter, Zeit, Grenze, Uhr, Klänge, Level-Events; nimmt `NcLevel`-Handles, um andere Dimensionen zu erreichen, Koordinaten sind einfache `x y z`-Ganzzahlen |
| `NcRegistries` | eingebaute Registries, nach Namen nachgeschlagen (Blöcke, Items, Flüssigkeiten, Effekte, Biome, Partikel, Entities, Block-Entities) |
| `NcRecipes` | Rezeptabfragen (Raster-Crafting, Steinschneiden, Kochen; Rezepte per id abrufen) |
| `NcLists` | Listen und Konfiguration (Whitelist, Ops, Bans, `server.properties`) |
| `NcStartup` | Startargumente (vom Kernel nicht erkannte Tokens und namensbasiertes Abonnement) |

`NcPlayer` ist keine statische Fassade, sondern ein **Objekthandle**: `NcPlayers.All` / `Find` geben es zurück, und `Player` / `Attacker` in Spieler-Events sind ebenfalls es. Handles sind schreibgeschützt und werden von Probes konstruiert; Mods können den `ServerPlayer` des Kernels nicht erhalten — der erste Ankerpunkt für „keine Kernel-Typen in der öffentlichen Schnittstelle". Derselbe Kernel-Spieler wird immer auf dasselbe Handle abgebildet, intern per Weak Reference zwischengespeichert und automatisch ungültig, sobald sich der Spieler abmeldet.

`NcLevel` folgt derselben Form für Levels. `NcWorld.Overworld` / `Nether` / `End` und `NcWorld.Get("minecraft:the_nether")` geben es zurück, und `LevelTickArgs.Level` ist ebenfalls eines. Es trägt die Dimensions-id, Zeit, Wetter, Bauhöhe, Tick-Zähler und das Zwangsladen von Chunks; Blockoperationen bleiben auf `NcWorld` und nehmen das Handle plus `x y z`. `BlockPos` taucht nie auf, daher trägt die dll eines Mods keine Referenz auf den Level-Typ des Kernels.

### 4.1 Registries

`NcRegistries` bietet sowohl vollständige Tabellen als auch Namensnachschlage. Vollständige Tabellen sind für Iteration und Tag-basiertes Nachschlagen gedacht; Namensnachschlage für das Abrufen eines einzelnen Elements:

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //Standardzustand des Blocks
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//die ganze Tabelle durchlaufen
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

Registries werden während des Starts schrittweise zusammengesetzt, und Mods werden geladen, bevor die Zusammenstellung abgeschlossen ist, also speichere nichts zwischen, was in `Init` nachgeschlagen wird — die Zusammenstellung läuft noch, und ein zwischengespeicherter Wert ist entweder eine Nullreferenz oder ein veralteter Wert. Derzeit ist `BuiltInRegistries.BootStrap` noch eine leere Implementierung; jede Registry wird separat von ihrem eigenen Bootstrap befüllt, und die datengetriebenen (Biome, Rezepte usw.) haben nur sehr wenige Einträge, bevor das Laden von Datenpaketen verdrahtet ist.

### 4.2 Rezepte

`NcRecipes` wird von einer aus Datenpaketen geladenen Rezepttabelle gestützt; `/reload` ersetzt die ganze Tabelle, also halte einen `RecipeHolder` nicht über Reloads hinweg.

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //die Ausgabe für ein Crafting-Raster berechnen
    var recipes = NcRecipes.StonecuttingFor(stack);         //für diese Eingabe verfügbare Steinschneide-Rezepte
    var smelting = NcRecipes.CookingFor("smelting", stack); //nach Kochtyp nachschlagen
    var byId = NcRecipes.Find("minecraft:oak_planks");      //ein Rezept per id abrufen
}
```

***

## 5. Interna

Du brauchst diesen Abschnitt nicht, um Mods zu schreiben, aber er kann beim Debuggen helfen.

### 5.1 Probes

| Klasse                                                   | Form        | Zuständigkeit                                               |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)`                  | Mark ×4     | alle „etwas ist passiert"-Signale laufen in einer Methode zusammen und werden per `label` an das passende Event verteilt |
| `Internal.CommandProbe.OnCommandsReady(object)`          | CallSite    | ersetzt den Aufruf von `EffectCommand::Register`; nach Wiederherstellung des Originalaufrufs wird `CommandRegister` ausgelöst |
| `Internal.PlayerProbe.OnXxx(object, ...)`                | CallSite ×6 | Spieler-Events, eine Methode pro Hook-Punkt; nach Wiederherstellung des Originalaufrufs wird das Event veröffentlicht |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | Chunk-Events, gehookt an den Zuweisungsstellen der drei Callback-Eigenschaften von `ServerChunkCache`; ein Wrapper-Delegate wird vorgeschaltet, bevor die Kontrolle an den Kernel zurückgegeben wird |
| `Internal.BlockProbe.OnXxx(...)`                         | CallSite ×3 | Block-Events; Abbau und Drops hooken `ServerBlockUpdates`, Zustandsänderungen hooken die Schnittstellenmethode auf `IBlockUpdateSink` |

Die Signatur von `SignalProbe` nimmt nur `string`, und die Parameter von `CommandProbe`, `PlayerProbe` und `LevelProbe` sind als `object` deklariert — das ist Absicht: Während der Assemblierung löst `Lead.Hook` die Signatur der Ersatzmethode auf, und sobald ein Kernel-Typ darin auftaucht, zieht dessen Auflösung die Kernel-Assembly frühzeitig herauf und die Injektion verpasst ihr Zeitfenster. Kernel-Typen erscheinen nur innerhalb von Methodenkörpern, zu welchem Zeitpunkt der Code bereits läuft.

Das Einzige, was in einer Signatur nicht `object` sein kann, sind Werttyp-Parameter und Rückgabewerte: `object` ist eine Referenz auf dem Stack, während `float`/`bool` Werte sind, und eine Nichtübereinstimmung ist ungültige IL. Daher behält `PlayerProbe.OnHurtPlayer` `float` für die Schadensmenge, und `OnRemovePlayer` und `OnHurtPlayer` behalten `bool`-Rückgabewerte.

`BlockProbe` ist eine Erweiterung dieser Einschränkung: Blockposition und -zustand sind die beiden Werttypen `BlockPos`/`BlockState`, die nur als sie selbst in die Signatur geschrieben werden können. Diese beiden Typen stammen aus `NetCraft.Primitives` und `NetCraft.Registry`, von denen keiner auf der Injektionsliste steht, daher zieht ihre Auflösung während der Assemblierung die umzuschreibenden Assemblies nicht frühzeitig herauf.

### 5.2 Liste der Hook-Punkte

Die `ncmod.json` von ModApi enthält vierundzwanzig Regeln, die eins zu eins zur Tabelle in 2.1 passen. Um einen Hook-Punkt zu ändern oder eine Regel hinzuzufügen, bearbeite diese Datei; baue nach der Bearbeitung neu (sie ist eine eingebettete Ressource) und lege die resultierende dll zurück in `mods/` — Letzteres erledigt bereits automatisch `DeployModToHosts` in `NetCraft.ModApi.csproj`, und wenn es fehlt, äußert sich das darin, dass die Regeln überhaupt nicht wirksam werden.

`CommandManager::Execute` hat zwei Overloads, die sich eine Regel teilen. Das CallSite von `Lead.Hook` gleicht Aufrufstellen nach „Typ + Methodenname" ab, nicht nach Parameterliste, und beide Overloads nehmen zwei Parameter, sodass die Probe `object` für den ersten Parameter nehmen und nach dem echten Typ verteilen kann.

Die beiden `PacketProcessor`-Regeln sind komplementär: Pakete der Play-Phase gehen über `ScheduleIfPossible` in die Hauptthread-Warteschlange, während Handshake- und Status-Phase über `HandleNow` zur sofortigen Verarbeitung laufen; ein gegebenes Paket durchläuft nur eine von beiden. Nur Erstere zu hooken verpasst die Handshake- und Status-Phase — die zufällig am leichtesten mit Skripten zu testen sind, weshalb dies beim Debuggen leicht als „die Regel hat nicht gewirkt" fehlinterpretiert wird.

Die drei Chunk-Regeln hooken die **Zuweisungsstellen** der drei Callback-Eigenschaften von `ServerChunkCache`, nicht die Lesestellen. Der Grund ist, dass diese drei Eigenschaften unicast sind und bereits vom Kernel selbst belegt werden, wenn `PersistentServerLevel` konstruiert wird (sie injizieren Speicher- und Block-Entity-Aufräumlogik); ein Mod, der direkt zuweist, würde die Kopie des Kernels überschreiben — Entladungen nicht persistiert, Block-Entities nicht aufgeräumt, und das ganz ohne Fehler. Die Zuweisungsstelle zu hooken erlaubt es, den Callback des Kernels und die Probe in diesem Moment zu verketten; die Zuweisung geschieht nur einmal, und jeder weitere Auslöser fügt eine Ebene Delegate-Weiterleitung hinzu.

`PlayerList::RespawnPlayer` ist privat, daher kann die Probe den Originalaufruf nicht wiederherstellen; dieser läuft über Reflexion (einmal pro Tod aufgerufen, also ist der Overhead vernachlässigbar). Das lässt auch eine Tür für die Angleichung an den Kernel offen: Falls künftig `InternalsVisibleTo` dafür hinzugefügt wird, kann es auf einen direkten Aufruf umgestellt werden.

### 5.3 Register

`Internal.NcCommandRegistry` zeichnet Befehle auf, die über `args.Register` registriert wurden. Es ist nur ein Register und nimmt nicht an der Befehlsausführung teil; die Befehle selbst sind auf dem Kernel-Dispatcher installiert, also funktionieren sie auch dann noch, wenn das Register Probleme hat.

***

## 6. Noch hinzuzufügen

Das Folgende sind Hook-Punkte, deren Positionen bestätigt sind, die aber noch keine Events geworden sind (die Liste wurde von `__scan_mod_api.py` im Repository-Stamm erzeugt):

| Richtung          | Kandidaten-Hook-Punkte                                                       |
| ----------------- | ---------------------------------------------------------------------------- |
| Entities          | `ClientLevel::AddEntity`, `Entity::Die`                                      |
| Welt              | Laden und Entladen von Levels, Chunk-Batching von `ServerChunkCache`         |
| Geländegenerierung | `ChunkGenerator::Generate`-Stufen pro `ChunkStatus`, `WorldGenRegion::SetBlockState` |
| Befehlsausführung | `CommandSourceStack::SendSuccess` / `SendFailure` (die Antwort-Hälfte, mit vielen Aufrufstellen) |
| Netzwerk          | `ServerGamePacketListenerImpl::HandleXxx` pro Pakettyp (derzeit nur ein einheitlicher Einstiegspunkt) |

Bereits erledigte Richtungen: Level-Ticks wurden `ServerEvents.LevelTick`, die Persistenz gespeicherter Daten wurde `ServerEvents.SavedDataSaving`, die Befehlsausführung wurde `ServerEvents.CommandExecuted`, und Blöcke wurden `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`.

Den Block-Bereich auf `SetBlock` zu hooken funktioniert nicht: Er hat zwei Standardparameter, `notifyNeighbors` und `strict`, sodass kompilierte Aufrufstellen zwischen 4 und 6 Parameter annehmen, und da CallSite nach „Typ + Methodenname" abgleicht, ohne die Parameterliste anzusehen, kann eine Ersatzmethode nicht alle drei Stack-Formen bewältigen. Stattdessen werden zwei Stellen gehookt: Die Zustandssynchronisierung hookt `IBlockUpdateSink::BlockChanged` (der einzige Schnittstellenaufruf in `ServerLevel.SetBlock`, der jede Änderung mit Client-Sync abdeckt), und Abbau und Drops hooken die eigenen Methoden von `ServerBlockUpdates`.

Die Entity-Zelle ist problematisch, weil `Entity` in `NetCraft.Registry` definiert ist, das nicht auf der Injektionsliste steht, sodass Aufrufstellen, die es anvisieren, nicht umgeschrieben werden können. `ClientLevel::AddEntity` liegt in der Client-Assembly und ist machbar; ein Todes-Event erfordert zuerst die Klärung, ob `Registry` umgeschrieben werden kann.

Für die Netzwerk-Zelle existiert `HandleChat` schon lange und `NetworkEvents.PacketReceived` bietet einen einheitlichen Einstiegspunkt, daher sind Hooks pro `HandleXxx` deutlich weniger wertvoll; nur Szenarien, die eine feinkörnige Filterung nach Pakettyp benötigen, lohnen sich hinzuzufügen.

Laden und Entladen von Levels haben auf der NC-Seite keinen Konvergenzpunkt: `DedicatedServer::CreateLevel` ist privat, sodass der Originalaufruf nur per Reflexion wiederhergestellt werden kann, wie bei `PlayerList::RespawnPlayer`; der Entladepfad ist noch verstreuter. Um es umzusetzen, muss zuerst geklärt werden, wie die Event-Args aussehen sollen.

Alle verbleibenden benötigen Objektreferenzen (Entity-Instanzen usw.), daher müssen sie `CallSite` statt `Mark` verwenden; wenn ein Kernel-Werttyp unter den Parametern auftaucht, kann er nur als er selbst in die Signatur der Ersatzmethode geschrieben werden.

Die Schnittstellenoberflächen-Liste (`__modapi_api.txt`, erzeugt von `__scan_mod_api.py --api`) wurde ebenfalls durchgesehen: Einträge, die sich als Fähigkeits-Einstiegspunkte qualifizierten, wurden nach Domäne in die Fassaden von Kapitel 4 gesammelt, und der Rest, der nicht offengelegt ist, fällt in drei Kategorien — Protokoll- und Paketverarbeitung (`Network.Protocol.*`), Rendering und Modelle (`Client.Render.*`) sowie Geländegenerierung und Dichtefunktionen (`LevelGen.*`). Dies sind Kernel-Interna; sie direkt zu verwenden würde Mods an Implementierungsdetails binden, daher sollte zuerst eine stabile Schnittstelle im Kernel geöffnet werden.

Auf der Server-Seite gibt es noch zwei Dinge, die nicht als Fassaden verpackt sind: Die `ReloadableServerResources`-Instanz hängt an `DedicatedServer`, und da ModApi nicht auf `NetCraft.Server` verweist, erfordert ihre Verpackung zuerst das Öffnen einer Eigenschaft in der Kernel-Basisklasse; `ChunkSender` und `ServerWorldBorderListener` sind interne Abläufe ohne Anwendungsfall für Mods.
