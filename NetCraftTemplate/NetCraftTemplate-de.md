# NetCraft Mod-Vorlage

Beispielcode für die NetCraft-Mod-Entwicklung. Jeder Eintrag in `NetCraftTemplate.yaml`
verweist auf eine Datei unter `examples/`, und `ncm template` lädt sie bei Bedarf.

## Verwendung

```
ncm template view              jeden Eintrag auflisten
ncm template view Wrapper.?    nach id filtern, ? und * sind Platzhalter
ncm template example <api id>  eine Beispieldatei ins aktuelle Verzeichnis laden
```

## Die zwei Routen

`NetCraft.ModApi` stellt zwei Namespaces bereit; wähle den passenden:

| Namespace | Was du bekommst |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Events, `Nc*`-Fassaden und `Nc*`-Handles. Kein Kernel-Typ erscheint in der öffentlichen Schnittstelle, daher erzwingt eine Umbenennung im Kernel keinen Neubau deines Mods. |
| `NetCraft.ModApi.Extension` | `[Inject]`- und `[Mixin]`-Attribute. Regeln benennen Kernel-Typen und -Methoden direkt, was mächtiger und zugleich brüchiger ist. |

## Häufige Aufrufe

Öffne dieses Panel aus einem Mod-Projekt heraus, und jeder der folgenden Aufrufe,
den dein Code tatsächlich verwendet, wird gegen `NetCraftTemplate.yaml` bewertet:
grün, wenn das Mitglied deklariert ist, gelb, wenn das Mitglied fehlt, rot, wenn der
Typ überhaupt nicht deklariert ist. Fahre über einen hervorgehobenen Namen, um den Grund zu sehen.

| Aufruf | Was er bewirkt |
| --- | --- |
| `NcServer.IsAvailable` | ob der Server läuft und erfasst ist |
| `NcServer.Broadcast` | Systemnachricht an alle Online-Spieler |
| `NcServer.Execute` | einen Befehl als Konsole ausführen |
| `NcWorld.GetBlock` | einen Block lesen, null wenn der Chunk entladen ist |
| `NcWorld.SetBlock` | einen Block schreiben, führt die vollständige Update-Kette aus |
| `NcWorld.BreakBlock` | einen Block abbrechen, wie es ein Spieler täte |
| `NcPlayer.Name` | der Spielername |
| `NcPlayer.Health` | aktuelle Gesundheit |
| `NcPlayers.Find` | einen Online-Spieler nach Namen nachschlagen |
| `NcPlayers.Send` | private Systemnachricht |
| `NcRegistries.FindState` | Blockzustand nach namensraumbehafteter id |
| `NcRegistries.FindItem` | Item nach namensraumbehafteter id |
| `ServerEvents.Tick` | läuft bei jedem Server-Tick |
| `ServerEvents.PlayerJoin` | ein Spieler hat den Beitritt abgeschlossen |
| `ServerEvents.BlockBroken` | ein Block wurde tatsächlich ersetzt |
