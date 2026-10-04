# NetCraft-ModApi-Referenz

`NetCraft.ModApi` ist die API-Oberfläche, die NetCraft Mods zur Verfügung stellt. Es hat zwei Identitäten: Für Sie ist es eine API-Bibliothek; Für sich genommen ist es ein gewöhnlicher Mod (`id` ist `netcraft-modapi` und liefert seine eigenen `ncmod.json` und Injektionssonden).

Diese Datei wächst mit dem Wachstum der API. Informationen zum architektonischen Hintergrund, zu Unterschieden zu Fabric und zum Schreiben eines Mods finden Sie unter [modding-guide.md](modding-guide.md).

- Montage: `NetCraft.ModApi.dll`

- Abhängigkeiten: `NetCraft` (die Hauptbibliothek), `NetCraft.Game`

Die öffentliche Oberfläche ist in drei Namensräume aufgeteilt:

| Namensraum | Inhalt | Notizen |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | Ereignis- und Abonnement-Basisklasse `NcEvent<T>`, `Nc*` Fassaden, `Nc*` Objekthandles | Wrapper-Schicht; Keine Kerneltypen auf der öffentlichen Oberfläche |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` Anmerkungen | Erweiterungspunkte; Regeln binden an Kernel-Klassen- und Methodennamen |
| `NetCraft.ModApi.Internal` | Injektionssonden | Nicht direkt verweisen |

Der Root-Namespace `NetCraft.ModApi` enthält nur die Eintragsklasse `ModApiEntry`. `Wrapper` und `Extension` sind zwei parallele Routen; Informationen zur Auswahl finden Sie unter [modding-guide.md 2.9](modding-guide.md#29-two-routes-wrapper-layer-and-extension-points).

***

## 1. Schnellstart

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

Zeigen Sie in `ncmod.json` mit `entry` auf diese Klasse und lassen Sie `hooks` leer – die folgenden Ereignisse werden alle von ModApis eigenen Sonden bereitgestellt.

***

## 2. Ereignisse

Alle Veranstaltungen live unter `NetCraft.ModApi.Wrapper`; nach `using NetCraft.ModApi.Wrapper;` sind sie verfügbar.

### 2.1 Übersichtstabelle

| Veranstaltung | Args-Typ | Auslöser | Seite | ModApi-Hook-Punkt |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick` | `ServerTickArgs` | jeder Tick der Server-Hauptschleife | Server | `DedicatedServer::Tick` (Markierung) |
| `ServerEvents.Started` | `ServerPhaseArgs` | Hauptschleife gestartet, nachdem `Done (x.xxxs)!` gedruckt wurde | Server | `MinecraftServer::Run` (Markierung) |
| `ServerEvents.Stopping` | `ServerPhaseArgs` | Der Server beginnt mit dem Herunterfahren. Spieler werden bald getrennt | Server | `DedicatedServer::Stop` (Markierung) |
| `ServerEvents.CommandRegister` | `CommandRegisterArgs` | alle integrierten Befehle wurden registriert | Server | `EffectCommand::Register` Anrufseite (CallSite) |
| `ServerEvents.PlayerJoin` | `PlayerJoinArgs` | die Join-Paketsequenz wurde gesendet | Server | `PlayerList::PlaceNewPlayer` Anrufseite (CallSite) |
| `ServerEvents.PlayerLeave` | `PlayerLeaveArgs` | Spieler aus der Online-Liste entfernt | Server | `PlayerList::RemovePlayer` Anrufseite (CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | Paket gesendet und Verbindung getrennt | Server | `ServerPlayer::Disconnect` Anrufseite (CallSite) |
| `ServerEvents.PlayerHurt` | `PlayerHurtArgs` | tatsächlich verursachter Schaden | Server | `PlayerList::HurtPlayer` Anrufseite (CallSite) |
| `ServerEvents.PlayerDeath` | `PlayerDeathArgs` | unmittelbar nachdem die Gesundheit auf Null zurückgesetzt wurde | Server | `PlayerList::RespawnPlayer` Anrufseite (CallSite) |
| `ServerEvents.PlayerChat` | `PlayerChatArgs` | nach der Chat-Übertragung | Server | `ServerGamePacketListenerImpl::HandleChat` Anrufseite (CallSite) |
| `ServerEvents.ChunkLoaded` | `ChunkLoadedArgs` | Chunk gelangt zum ersten Mal in den Speicher | Server | `ServerChunkCache::set_ChunkLoaded` Zuweisungsseite (CallSite) |
| `ServerEvents.ChunkUnloaded` | `ChunkUnloadedArgs` | Chunk verlässt die Erinnerung | Server | `ServerChunkCache::set_ChunkUnloaded` Zuweisungsseite (CallSite) |
| `ServerEvents.ChunkSaved` | `ChunkSavedArgs` | Snapshot erstellt, bevor der Block auf die Festplatte geschrieben wird | Server | `ServerChunkCache::set_ChunkSaveSink` Zuweisungsseite (CallSite) |
| `ServerEvents.CommandExecuted` | `CommandExecutedArgs` | Die Ausführung eines Befehls wurde beendet. Syntaxfehler und Berechtigungsverweigerungen zählen ebenfalls | Server | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick` | `LevelTickArgs` | Level-Tick, einmal pro geladenem Level pro Tick | Server | `PersistentServerLevel::Tick` (CallSite) |
| `ServerEvents.SavedDataSaving` | `SavedDataSavingArgs` | gespeicherte Daten werden auf die Festplatte geschrieben, einen Schritt später als die Blockspeicherung | Server | `SavedDataStorage::ScheduleSave` (CallSite) |
| `ServerEvents.BlockChanged` | `BlockChangedArgs` | Blockstatus geändert, Synchronisierung mit Clients im Begriff | Server | `IBlockUpdateSink::BlockChanged` (CallSite) |
| `ServerEvents.BlockBroken` | `BlockBrokenArgs` | Block gebrochen; Player-Mining und Redstone-Selbstzerstörung zählen beide | Server | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped` | `ItemDroppedArgs` | Gespawnte Entität für fallengelassene Gegenstände, einschließlich Block-Break-Drops und Kochprodukten | Server | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived` | `PacketReceivedArgs` | jedes eingehende Paket, das in die Warteschlange eines Handlers gestellt wird, einschließlich Handshake- und Statusphasen | beide | `PacketProcessor::ScheduleIfPossible` und `HandleNow` (CallSite) |
| `ClientEvents.Tick` | `ClientTickArgs` | jeder Tick der Client-Hauptschleife | Kunde | `MinecraftClient::Tick` (Markierung) |

Reihenfolge der Spielerereignisse: Der Tod ist im Verletzungsfluss verschachtelt, daher steht `PlayerDeath` vor dem entsprechenden `PlayerHurt`; `PlayerLeave` und `PlayerDisconnect` sind zwei verschiedene Dinge – Ersteres bedeutet das Entfernen aus der Online-Liste (auch nach `/kick` wird es erst ausgelöst, wenn die Verbindung unterbrochen wird), Letzteres bedeutet, dass die Verbindung selbst unterbrochen wird, und es ist nicht garantiert, dass die beiden paarweise erscheinen.

### 2.2 Abonnieren und Abmelden

```csharp
IDisposable Subscribe(Action<T> handler)
```

- Wenn Sie dasselbe Ereignis mehrmals abonnieren, werden Benachrichtigungen in der Reihenfolge des Abonnements gesendet.

- Dispatch erstellt einen Schnappschuss der Rückrufliste, sodass das Abonnieren oder Aufheben der Registrierung innerhalb eines Rückrufs keinen Einfluss auf den aktuellen Versand hat.

- Ohne Abmeldung bleibt es für immer gültig; Mods bieten keinen Entlademechanismus, sodass eine manuelle Aufhebung der Registrierung normalerweise nicht erforderlich ist.

### 2.3 Args-Typen

**`ServerTickArgs`**

| Eigentum | Geben Sie | ein Notizen |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | Tick-Anzahl seit diesem Start, beginnend bei 1 |

Beachten Sie, dass dies von ModApi selbst gezählt wird, nicht vom `TickCount` des Kernels.

**`ClientTickArgs`**

| Eigentum | Geben Sie | ein Notizen |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | wie oben, unabhängig auf der Clientseite gezählt |

**`ServerPhaseArgs`**

| Eigentum | Geben Sie | ein Notizen |
| -------- | -------- | ---------------------------- |
| `Phase` | `string` | Phasenname, `started` oder `stopping` |

Das Feld dupliziert das Ereignis selbst; Es wird beibehalten, damit die Protokollierung ein einheitliches Format verwenden kann.

**`CommandRegisterArgs`**

| Mitglied | Geben Sie | ein Notizen |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher` | `CommandDispatcher<CommandSourceStack>` | der Befehlsverteiler des Kernels |
| `Register(name, description, build)` | Methode | Registrieren Sie einen Befehl und zeichnen Sie ihn im Hauptbuch auf, siehe 3.1 |

**`PlayerJoinArgs`**

| Eigentum | Geben Sie | ein Notizen |
| ------------- | -------------- | ---------------------------------------------- |
| `Player` | `ServerPlayer` | der Spieler, der gerade beigetreten ist; Bereits gesendete Pakete zusammenführen, Status ist sicher lesbar |
| `ProfileName` | `string` | Spielername |

**`PlayerLeaveArgs`**

| Eigentum | Geben Sie | ein Notizen |
| --------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | der ausscheidende Spieler, zu diesem Zeitpunkt nicht mehr in der Online-Liste |
| `Removed` | `bool` | ob tatsächlich entfernt; `false` bei wiederholter Entfernung |

**`PlayerDisconnectArgs`**

| Eigentum | Geben Sie | ein Notizen |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | der getrennte Spieler |
| `Reason` | `string` | Trennungsgrund; Klartext, wenn als Komponente angegeben |

Das Disconnect-Paket wurde gesendet und die Verbindung geschlossen; Das Senden von Paketen an diesen Player hat jetzt keine Auswirkung.

**`PlayerHurtArgs`**

| Eigentum | Geben Sie | ein Notizen |
| ---------- | --------------- | -------------------------------------------- |
| `Player` | `ServerPlayer` | der Spieler, der verletzt wurde |
| `Attacker` | `ServerPlayer?` | der Spieler, der den Schaden verursacht hat; `null` für Umwelt- und Kommandoschäden |
| `Amount` | `float` | Schadenshöhe dieses Mal |

Wird nicht während Unverwundbarkeits-Frames oder nach dem Tod ausgelöst (`Hurt` des Kernels gibt `false` zurück).

**`PlayerDeathArgs`**

| Eigentum | Geben Sie | ein Notizen |
| ---------- | --------------- | ------------------------------ |
| `Player` | `ServerPlayer` | der Spieler, der gestorben ist |
| `Attacker` | `ServerPlayer?` | der Mörder; `null` wenn es keines gibt |

Der Kernel wird sofort zurückgesetzt, wenn die Gesundheit Null erreicht. Wenn also das Ereignis ausgelöst wird, ist der Spieler am Respawn-Punkt bereits bei voller Gesundheit. Die Koordinaten und Drops zum Zeitpunkt des Todes sind nicht verfügbar.

**`PlayerChatArgs`**

| Eigentum | Geben Sie | ein Notizen |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | Absendername |
| `Message` | `string` | Klartextnachricht |

Bei diesem Ereignis handelt es sich um eine **schreibgeschützte Benachrichtigung**: Die ursprüngliche Methode hat die Nachricht bereits gesendet, daher hat eine Änderung von `Message` hier keine Auswirkung.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

Die drei Ereignisse haben die gleiche Argumentform:

| Eigentum | Geben Sie | ein Notizen |
| -------- | ----- | ------------------ |
| `X` | `int` | Blockkoordinate X |
| `Z` | `int` | Blockkoordinate Z |

Der einfachste Fehler ist `ChunkSaved`: Der Kernel benötigt diesen Rückruf, um **den Snapshot synchron zu erstellen**, während Serialisierung und Festplattenschreibvorgänge asynchron vom Kernel selbst durchgeführt werden. Zeitaufwändige Arbeit in diesem Rückruf verlangsamt also direkt das Entladen von Blöcken, und destruktive Vorgänge (Blöcke entfernen, Lagerbestände ändern) sollten hier ebenfalls nicht berücksichtigt werden – sie versprechen nur den Snapshot-Moment.

Wenn `ChunkUnloaded` ausgelöst wird, wurden die Blockentitäten bereits zusammen mit dem Chunk bereinigt; Wenn Sie Blöcke lesen möchten, verwenden Sie `ChunkSaved` (das ebenfalls nicht auf Blöcke zugreifen kann) oder einen früheren Punkt.

**`CommandExecutedArgs`**

| Eigentum | Geben Sie | ein Notizen |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string` | roher Befehlstext; Chat-Befehle enthalten keinen führenden Schrägstrich |
| `Result` | `int` | Befehlsrückgabewert; 0 bedeutet Misserfolg oder Ablehnung |
| `Source` | `CommandSourceStack?` | Befehlsquelle; `null` auf dem Player-Overload-Pfad |
| `Player` | `ServerPlayer?` | der Spieler, der den Befehl gegeben hat; `null` bei Ausgabe über die Konsole |

Das Ereignis wird ausgelöst, **nachdem** der Befehl beendet ist; Es kann die Ausführung nicht ändern. Auch Syntaxfehler und Berechtigungsverweigerungen kommen hier vor; Verwenden Sie `Result`, um sie voneinander zu unterscheiden. Befehle, die ein Spieler über die Chatleiste sendet, durchlaufen die Überlastung `Execute(ServerPlayer, string)`, wobei der Kernel die Befehlsquelle intern erstellt. In diesem Fall ist also `Source` `null` und nur `Player` festgelegt.

**`LevelTickArgs`**

| Eigentum | Geben Sie | ein Notizen |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level` | `NcLevel` | das in diesem Tick fortgeschrittene Level |
| `RunsNormally` | `bool` | ob es normal voranschritt; `false` während `/tick freeze` |

Wird einmal pro geladenem Level pro Tick ausgelöst, sodass eine Welt mit mehreren Leveln mehrere pro Tick erhält. Der Triggerpunkt liegt, nachdem der Level-Tick **abgeschlossen ist**; Es ist ein Beobachtungspunkt, kein Abfangpunkt.

**`SavedDataSavingArgs`**

| Eigentum | Geben Sie | ein Notizen |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage` | die gespeicherte Datentabelle wird beibehalten |

Dies ist ein anderer Pfad als `ServerEvents.ChunkSaved`: Der Chunk One erstellt nur einen Snapshot und schreibt asynchron, während dieser nach Abschluss eines synchronen Schreibvorgangs ausgelöst wird. Weltzeituhr, Spielregeln und Weltgrenzdaten durchlaufen es.

**`BlockChangedArgs`**

| Eigentum | Geben Sie | ein Notizen |
| -------- | ------------ | --------------------------- |
| `Pos` | `BlockPos` | Position des geänderten Blocks |
| `State` | `BlockState` | Blockzustand nach der Änderung |

Der Status wurde bereits in den Chunk geschrieben und steht kurz vor der Synchronisierung mit Clients, sodass die Änderung selbst hier nicht geändert werden kann. Redstone-Komponenten, die ihren eigenen Status in Verhaltensrückrufen ändern, werden ebenfalls über diesen Pfad beendet. Da es sich um eine Hochfrequenz handelt, sollten Sie beim Rückruf keine zeitaufwändige Arbeit leisten.

**`BlockBrokenArgs`**

| Eigentum | Geben Sie | ein Notizen |
| -------- | --------------- | ------------------------------------------ |
| `Pos` | `BlockPos` | Position des gebrochenen Blocks |
| `Player` | `ServerPlayer?` | der Unterbrecher; `null` für Nicht-Spieler-Ursachen wie Redstone |

Wird nur ausgelöst, wenn der Block tatsächlich ersetzt wird; Leere Positionen und abgelehnte Pausen lösen es nicht aus. Break-Effekte und Drops wurden bereits behandelt, daher ist das, was Sie im Event lesen, das Ergebnis.

**`ItemDroppedArgs`**

| Eigentum | Geben Sie | ein Notizen |
| -------- | ----------- | ----------------------- |
| `Pos` | `BlockPos` | wo das abgelegte Element erschien |
| `Stack` | `ItemStack` | der fallengelassene Gegenstandsstapel |

Hier finden Sie Block-Break-Tropfen und Produkte zum Kochen am Lagerfeuer. Ein leerer Gegenstandsstapel erzeugt keine Entität, daher gibt es kein Ereignis.

**`PacketReceivedArgs`**

| Eigentum | Geben Sie | ein Notizen |
| --------------- | -------- | ------------------------ |
| `Listener` | `object` | der Listener, der dieses Paket empfängt |
| `Packet` | `object` | das Paketobjekt selbst |
| `IsServerbound` | `bool` | ob es sich um ein servergebundenes Paket handelt |

Wird für jedes eingehende Paket ausgelöst und deckt alle vier Phasen ab: Handshake, Status, Konfiguration und Wiedergabe. Bewegungspakete kommen mehrmals pro Tick an, machen Sie also keine zeitraubende Arbeit im Callback. Das Paket ist bereits in ein Objekt dekodiert, hat aber noch nicht die Business-Schicht erreicht; Um Typen zu unterscheiden, überprüfen Sie `Packet` selbst. Ausgehende Pakete liegen außerhalb des Geltungsbereichs dieses Ereignisses.

***

## 3. Erweiterungspunkte

### 3.1 Befehlsregistrierung

Der Zeitpunkt ist `ServerEvents.CommandRegister`. Die Argumente dieses Ereignisses nicht zwischenspeichern; Der interne Befehlsbaum wird beim Start nur einmal erstellt.

```csharp
public void Register(
    string name,                                          //command literal, without the slash
    string description,                                   //one-line description, shown in the /ncmapi ledger
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attach arguments and the executor
```

Das `build`, das Sie erhalten, ist der Brigadier-Builder des Kernels; Schreibargumente, Unterbefehle und Berechtigungsprädikate geben den Weg des Kernels vor:

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //permission predicate
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
                    player.Heal(20f);                     //illustrative
                return 1;
            })));
```

Kernpunkte:

- Durch den direkten Aufruf von `args.Dispatcher.Register(...)` wird ebenfalls ein Befehl installiert, dieser gelangt jedoch nicht in das Hauptbuch und `/ncmapi` zeigt ihn nicht an. Verwenden Sie `args.Register`, wenn Sie es aufgelistet haben möchten.

- Für Befehle gibt es standardmäßig keine Berechtigungseinschränkung. Fügen Sie `.Requires(...)` bei Bedarf selbst hinzu.

- Das Verhalten zur Ausführungszeit liegt ganz bei Ihnen; ModApi fängt es nicht ab.

### 3.2 Registrierte Befehle anzeigen

Es gibt eine integrierte `/ncmapi`, die Berechtigungsstufe 2 erfordert:

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

Verwendungszeilen werden im Handumdrehen aus der Knotenstruktur des Befehlsbaums berechnet: Literale werden nach Namen geschrieben, Argumente werden in spitze Klammern eingeschlossen und Zwischenknoten, die selbst ausführbar sind, erhalten ebenfalls ihre eigene Zeile.

***

## 4. Serverfassaden

Die Fassaden in diesem Kapitel leben alle unter `NetCraft.ModApi.Wrapper`; nach `using NetCraft.ModApi.Wrapper;` sind sie verfügbar.

Fassaden sind `Nc*` statische Klassen, die über den Kernel verteilte Funktionen an einigen wenigen Einstiegspunkten sammeln. Die Kernel-Instanz wird von einer Probe erfasst, wenn die Hauptschleife startet; Wenn `NcServer.IsAvailable` falsch ist, wird alles unten ausgelöst – verwenden Sie sie nur innerhalb von Ereignisrückrufen, nicht von `Init`.

| Fassade | Zweck |
| --- | --- |
| `NcServer` | Serverinstanz, Tickrate, Befehle, Entitätsverfolgung, Spielerdaten, Spielregeln, Broadcast, Befehlsausführung |
| `NcPlayers` | Online-Spielerabfragen und -Operationen (Kick, Teleport, Gesundheit, Spielmodus, Berechtigungen) |
| `NcWorld` | Overworld-Block Lesen/Schreiben und Brechen, Wetter, Zeit, Grenze, Uhr, Geräusche, Level-Ereignisse; benötigt `NcLevel`-Griffe, um andere Dimensionen zu erreichen, die Koordinaten sind einfach `x y z` ints |
| `NcRegistries` | integrierte Register, die nach Namen gesucht werden (Blöcke, Gegenstände, Flüssigkeiten, Effekte, Biome, Partikel, Entitäten, Blockentitäten) |
| `NcRecipes` | Rezeptabfragen (Gitterherstellung, Steinmetzarbeiten, Kochen; Rezepte anhand der ID abrufen) |
| `NcLists` | Listen und Konfiguration (Whitelist, Ops, Bans, `server.properties`) |
| `NcStartup` | Startargumente (vom Kernel nicht erkannte Token und namensbasiertes Abonnement) |

`NcPlayer` ist keine statische Fassade, sondern ein **Objekthandle**: `NcPlayers.All` / `Find` gibt es zurück, und `Player` / `Attacker` in Spielerereignissen ist es auch. Handles sind schreibgeschützt und werden von Sonden erstellt; Mods können den `ServerPlayer` des Kernels nicht erhalten – den ersten Anker von „keine Kerneltypen auf der öffentlichen Oberfläche“. Derselbe Kernel-Player wird immer demselben Handle zugeordnet, intern durch schwache Referenzen zwischengespeichert und automatisch ungültig gemacht, sobald sich der Player abmeldet.

`NcLevel` folgt der gleichen Form für Ebenen. `NcWorld.Overworld` / `Nether` / `End` und `NcWorld.Get("minecraft:the_nether")` geben es zurück, und `LevelTickArgs.Level` ist auch eines. Es trägt die Dimensions-ID, Zeit, Wetter, Bauhöhe, Tick-Anzahl und Chunk-Force-Loading; Blockoperationen bleiben auf `NcWorld` und nehmen den Griff plus `x y z`. `BlockPos` wird nie angezeigt, daher enthält die DLL eines Mods keinen Verweis auf den Kernel-Level-Typ.

### 4.1 Register

`NcRegistries` bietet sowohl vollständige Tabellen als auch namentliche Suchvorgänge. Ganze Tabellen dienen der Iteration und tagbasierten Suche; Suchvorgänge nach Namen dienen dazu, ein einzelnes Element abzurufen:

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //block default state
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//iterate the whole table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

Registrierungen werden während des Startvorgangs nach und nach zusammengestellt, und Mods werden geladen, bevor die Zusammenstellung abgeschlossen ist. Speichern Sie daher nichts im Cache, was in `Init` gesucht wird. Die Assemblierung läuft noch und ein zwischengespeicherter Wert ist eine Nullreferenz oder ein veralteter Wert. Derzeit ist `BuiltInRegistries.BootStrap` noch eine leere Implementierung; Jedes Register wird separat von seinem eigenen Bootstrap bevölkert, und die datengesteuerten (Biome, Rezepte usw.) haben nur sehr wenige Einträge, bevor das Laden des Datenpakets verkabelt ist.

### 4.2 Rezepte

`NcRecipes` wird durch eine Rezepttabelle unterstützt, die aus Datenpaketen geladen wird; `/reload` ersetzt die gesamte Tabelle, also halten Sie `RecipeHolder` nicht über mehrere Neuladevorgänge hinaus.

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

## 5. Interna

Sie benötigen diesen Abschnitt nicht, um Mods zu schreiben, er kann jedoch beim Debuggen hilfreich sein.

### 5.1 Sonden

| Klasse | Formular | Verantwortung |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)` | Markiere ×4 | Alle „etwas ist passiert“-Signale werden in eine Methode geleitet und von `label` | an das entsprechende Ereignis gesendet
| `Internal.CommandProbe.OnCommandsReady(object)` | CallSite | ersetzt den Aufruf von `EffectCommand::Register`; Nach dem Wiederherstellen des ursprünglichen Aufrufs wird `CommandRegister` | ausgelöst
| `Internal.PlayerProbe.OnXxx(object, ...)` | CallSite ×6 | Spielerereignisse, eine Methode pro Hook-Punkt; Nach der Wiederherstellung des ursprünglichen Aufrufs wird das Ereignis veröffentlicht |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | Chunk-Ereignisse, die an den Zuweisungsstellen der drei Callback-Eigenschaften von `ServerChunkCache` eingebunden sind; Ein Wrapper-Delegat wird überlagert, bevor die Kontrolle wieder an den Kernel übergeben wird
| `Internal.BlockProbe.OnXxx(...)` | CallSite ×3 | Blockereignisse; Breaking and Drops Hook `ServerBlockUpdates`, Zustandsänderungen Hook die Schnittstellenmethode auf `IBlockUpdateSink` |

Die Signatur von `SignalProbe` akzeptiert nur `string`, und die Parameter von `CommandProbe`, `PlayerProbe` und `LevelProbe` werden als `object` deklariert – dies ist beabsichtigt: Während der Assembly löst `Lead.Hook` die Signatur der Ersetzungsmethode auf, und sobald ein Kerneltyp darin erscheint, wird durch die Auflösung die Kernel-Assembly vorzeitig aufgerufen und die Injektion verfehlt ihr Fenster. Kerneltypen erscheinen nur innerhalb von Methodenkörpern, zu diesem Zeitpunkt wird der Code bereits ausgeführt.

Die einzigen Dinge, die in einer Signatur nicht `object` sein dürfen, sind Werttypparameter und Rückgabewerte: `object` ist eine Referenz auf dem Stapel, während `float`/`bool` Werte sind und eine Nichtübereinstimmung eine ungültige IL ist. `PlayerProbe.OnHurtPlayer` behält also `float` für die Schadenshöhe und `OnRemovePlayer` und `OnHurtPlayer` behalten die `bool`-Rückgabewerte.

`BlockProbe` ist eine Erweiterung dieser Einschränkung: Blockposition und Status sind die beiden Werttypen `BlockPos`/`BlockState`, die nur als sie selbst in die Signatur geschrieben werden können. Diese beiden Typen stammen von `NetCraft.Primitives` und `NetCraft.Registry`, von denen keiner auf der Injektionsliste steht. Wenn sie also während der Assemblierung aufgelöst werden, werden die neu zu schreibenden Baugruppen nicht frühzeitig aufgerufen.

### 5.2 Hakenpunktliste

`ncmod.json` von ModApi enthält vierundzwanzig Regeln, die eins zu eins mit der Tabelle in 2.1 übereinstimmen. Um einen Hook-Punkt zu ändern oder eine Regel hinzuzufügen, bearbeiten Sie diese Datei. Erstellen Sie nach der Bearbeitung neu (es handelt sich um eine eingebettete Ressource) und fügen Sie die resultierende DLL wieder in `mods/` ein. Letzteres wird bereits automatisch von `DeployModToHosts` in `NetCraft.ModApi.csproj` durchgeführt, und das Fehlen führt dazu, dass die Regeln überhaupt nicht wirksam werden.

`CommandManager::Execute` hat zwei Überladungen, die eine Regel teilen. CallSite von `Lead.Hook` gleicht Aufrufseiten nach „Typ + Methodenname“ ab, nicht nach Parameterliste, und beide Überladungen nehmen zwei Parameter an, sodass die Probe `object` als ersten Parameter annehmen und nach dem echten Typ weiterleiten kann.

Die beiden `PacketProcessor`-Regeln ergänzen sich: Play-Phase-Pakete gelangen über `ScheduleIfPossible` in die Haupt-Thread-Warteschlange, während Handshake- und Statusphasen zur sofortigen Verarbeitung über `HandleNow` laufen; Jedes Paket passiert nur einen von ihnen. Wenn man nur Ersteres einbindet, werden die Handshake- und Statusphasen übersehen – die mit Skripten am einfachsten zu prüfen sind, sodass dies beim Debuggen leicht als „die Regel wurde nicht wirksam“ missverstanden wird.

Die drei Chunk-Regeln binden die **Zuweisungs-Sites** der drei Callback-Eigenschaften von `ServerChunkCache` ein, nicht die Lese-Sites. Der Grund dafür ist, dass diese drei Eigenschaften Unicast sind und bereits vom Kernel selbst belegt werden, wenn `PersistentServerLevel` erstellt wird (sie fügen Speicher- und Blockentitätsbereinigungslogik ein); Ein Mod, der direkt zuweist, würde die Kopie des Kernels überschreiben – Entladungen werden nicht beibehalten, Blockentitäten werden nicht bereinigt und es gibt keinerlei Fehler. Durch das Einbinden der Zuweisungsseite können der Rückruf des Kernels und die Sonde in diesem Moment verkettet werden. Die Zuweisung erfolgt nur einmal und jeder nachfolgende Auslöser fügt eine Ebene der Delegate-Weiterleitung hinzu.

`PlayerList::RespawnPlayer` ist privat, daher kann die Probe den ursprünglichen Anruf nicht wiederherstellen; dass man eine Reflexion durchläuft (einmal pro Tod aufgerufen, daher ist der Aufwand vernachlässigbar). Dies lässt auch eine Tür für die Kernel-Ausrichtung offen: Wenn `InternalsVisibleTo` in Zukunft dafür hinzugefügt wird, kann es auf einen direkten Aufruf umgestellt werden.

### 5.3 Hauptbuch

`Internal.NcCommandRegistry` zeichnet Befehle auf, die über `args.Register` registriert wurden. Es ist nur ein Hauptbuch und nimmt nicht an der Befehlsausführung teil; Die Befehle selbst werden auf dem Kernel-Dispatcher installiert, sodass die Befehle auch dann noch funktionieren, wenn das Ledger Probleme hat.

***

## 6. Wird hinzugefügt

Im Folgenden sind Hook-Punkte aufgeführt, deren Standorte bestätigt sind, die aber noch nicht zu Ereignissen geworden sind (die Liste wurde von `__scan_mod_api.py` im Repository-Stammverzeichnis erstellt):

| Richtung | Kandidaten-Hook-Punkte |
| ----------------- | ---------------------------------------------------------------------------- |
| Entitäten | `ClientLevel::AddEntity`, `Entity::Die` |
| Welt | Level laden und entladen, `ServerChunkCache` Chunk-Batching |
| Geländegenerierung | `ChunkGenerator::Generate` Stufen pro `ChunkStatus`, `WorldGenRegion::SetBlockState` |
| Befehlsausführung | `CommandSourceStack::SendSuccess` / `SendFailure` (die Antworthälfte, mit vielen Anrufstellen) |
| Netzwerk | pro Pakettyp `ServerGamePacketListenerImpl::HandleXxx` (derzeit nur ein einheitlicher Einstiegspunkt) |

Bereits durchgeführte Anweisungen: Level-Ticks wurden zu `ServerEvents.LevelTick`, gespeicherte Datenpersistenz wurde zu `ServerEvents.SavedDataSaving`, Befehlsausführung wurde zu `ServerEvents.CommandExecuted` und Blöcke wurden zu `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`.

Das Anhängen der Blockzelle an `SetBlock` funktioniert nicht: Sie verfügt über zwei Standardparameter, `notifyNeighbors` und `strict`, sodass kompilierte Aufrufseiten zwischen 4 und 6 Parameter annehmen, und da CallSite nach „Typ + Methodenname“ abgleicht, ohne sich die Parameterliste anzusehen, kann eine Ersetzungsmethode nicht alle drei Stapelformen verarbeiten. Stattdessen hakt es an zwei Stellen: State-Sync-Hooks `IBlockUpdateSink::BlockChanged` (der einzige Schnittstellenaufruf in `ServerLevel.SetBlock`, der jede Änderung mit Client-Synchronisierung abdeckt) und Break-and-Drop-Hooks `ServerBlockUpdates`s eigene Methoden.

Die Entities-Zelle ist problematisch, da `Entity` in `NetCraft.Registry` definiert ist, das nicht auf der Injektionsliste steht, sodass Aufrufseiten, die darauf abzielen, nicht neu geschrieben werden können. `ClientLevel::AddEntity` befindet sich in der Client-Assembly und ist machbar; Bei einem Todesereignis muss zunächst geklärt werden, ob `Registry` umgeschrieben werden kann.

Für die Netzwerkzelle gibt es `HandleChat` schon seit langem und `NetworkEvents.PacketReceived` stellt einen einheitlichen Einstiegspunkt dar, sodass Pro-`HandleXxx`-Hooks viel weniger wertvoll sind; Nur Szenarien, die eine feinkörnige Filterung nach Pakettyp erfordern, sind eine Hinzufügung wert.

Das Laden und Entladen von Ebenen hat keinen Konvergenzpunkt auf der NC-Seite: `DedicatedServer::CreateLevel` ist privat, daher kann der ursprüngliche Aufruf nur durch Reflektion wiederhergestellt werden, wie `PlayerList::RespawnPlayer`; der Entladeweg ist noch weiter verstreut. Legen Sie dazu zunächst fest, wie die Ereignisargumente lauten sollen.

Alle übrigen benötigen Objektreferenzen (Entitätsinstanzen usw.), daher müssen sie `CallSite` anstelle von `Mark` verwenden; Wenn ein Kernel-Werttyp unter den Parametern erscheint, kann er nur als er selbst in die Signatur der Ersetzungsmethode geschrieben werden.

Die Schnittstellen-Oberflächenliste (`__modapi_api.txt`, erstellt von `__scan_mod_api.py --api`) wurde ebenfalls überprüft: Einträge, die als Fähigkeitseinstiegspunkte qualifiziert sind, wurden in den Kapitel-4-Fassaden nach Domäne gesammelt, und der Rest, der nicht offengelegt ist, fällt in drei Kategorien – Protokoll- und Paketverarbeitung (`Network.Protocol.*`), Rendering und Modelle (`Client.Render.*`) und Geländegenerierungs- und Dichtefunktionen (`LevelGen.*`). Dies sind Kernel-Interna; Ihre direkte Verwendung würde Mods an Implementierungsdetails binden, daher sollte zunächst eine stabile Schnittstelle im Kernel geöffnet werden.

Auf der Serverseite gibt es zwei weitere Dinge, die nicht als Fassaden verpackt sind: Die `ReloadableServerResources`-Instanz hängt an `DedicatedServer`, und da ModApi nicht auf `NetCraft.Server` verweist, erfordert das Umschließen zunächst das Öffnen einer Eigenschaft in der Kernel-Basisklasse; `ChunkSender` und `ServerWorldBorderListener` sind interne Flüsse ohne Anwendungsfall für Mods.
