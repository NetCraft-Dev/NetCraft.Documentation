# NetCraft-Mod-Vorlage

Beispielcode für die Entwicklung von NetCraft-Mods. Jeder Eintrag in `NetCraftTemplate.yaml` verweist auf eine Datei unter `examples/`, und `ncm template` ruft sie bei Bedarf ab.

## Verwendung

```
ncm template view              jeden Eintrag auflisten
ncm template view Wrapper.?    nach id filtern, ? und * sind Platzhalter
ncm template example <api id>  eine Beispieldatei ins aktuelle Verzeichnis holen
```

## Die zwei Wege

`NetCraft.ModApi` stellt zwei Namespaces bereit; wähle den, der passt:

| Namespace | Was du bekommst |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Events, `Nc*`-Fassaden und `Nc*`-Handles. In der öffentlichen Oberfläche taucht kein Kernel-Typ auf, daher erzwingt eine Kernel-Umbenennung keinen Neubau deines Mods. |
| `NetCraft.ModApi.Extension` | Attribute `[Inject]` und `[Mixin]`. Regeln benennen Kernel-Typen und -Methoden direkt, was mächtiger und fragiler ist. |

## Handles

`NcPlayer` und `NcLevel` sind schreibgeschützte Handles. Alles, was sie zurückgeben, ist ein einfacher String, eine Zahl oder ein Boolean (`NcLevel.Dimension`, `NcPlayer.X`), niemals ein Kernel-Typ, daher erzwingt eine Kernel-Umbenennung keinen Neubau deines Mods. Blockoperationen nehmen das Level-Handle plus `x y z`.

## Häufige Aufrufe

Öffne dieses Panel aus einem Mod-Projekt heraus; jeder Aufruf unten, den dein Code tatsächlich verwendet, wird gegen `NetCraftTemplate.yaml` geprüft: grün, wenn das Member deklariert ist, gelb, wenn das Member nicht existiert, rot, wenn der Typ überhaupt nicht deklariert ist. Fahre mit der Maus über einen hervorgehobenen Namen, um den Grund zu sehen.

| Aufruf | Was er tut |
| --- | --- |
| `NcServer.IsAvailable` | ob der Server läuft und erfasst wurde |
| `NcServer.Broadcast` | Systemnachricht an alle Online-Spieler |
| `NcServer.Execute` | einen Befehl als Konsole ausführen |
| `NcWorld.GetBlock` | einen Block lesen, null wenn der Chunk nicht geladen ist |
| `NcWorld.SetBlock` | einen Block schreiben, führt die vollständige Update-Kette aus |
| `NcWorld.BreakBlock` | einen Block so abbauen, wie ein Spieler es tun würde |
| `NcWorld.Overworld` | das Overworld-Level-Handle |
| `NcLevel.Dimension` | Dimensions-ID eines Level-Handles, zum Beispiel minecraft:overworld |
| `NcLevel.DayTime` | die Zeit einer Dimension lesen oder setzen |
| `NcPlayer.Name` | der Spielername |
| `NcPlayer.Health` | aktuelle Gesundheit |
| `NcPlayers.Find` | einen Online-Spieler nach Namen suchen |
| `NcPlayers.Send` | private Systemnachricht |
| `NcRegistries.FindState` | Blockzustand nach Namespace-ID |
| `NcRegistries.FindItem` | Item nach Namespace-ID |
| `ServerEvents.Tick` | läuft bei jedem Server-Tick |
| `ServerEvents.PlayerJoin` | ein Spieler hat das Beitreten abgeschlossen |
| `ServerEvents.BlockBroken` | ein Block wurde tatsächlich ersetzt |