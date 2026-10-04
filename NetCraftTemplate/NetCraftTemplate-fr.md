# Modèle de mod NetCraft

Code d’exemple pour le développement de mods NetCraft. Chaque entrée de `NetCraftTemplate.yaml` pointe vers un fichier sous `examples/`, et `ncm template` les récupère à la demande.

## Utilisation

```
ncm template view              lister chaque entrée
ncm template view Wrapper.?    filtrer par id, ? et * sont des jokers
ncm template example <api id>  récupérer un fichier d’exemple dans le répertoire courant
```

## Les deux voies

`NetCraft.ModApi` expose deux espaces de noms ; choisissez celui qui convient :

| Espace de noms | Ce que vous obtenez |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Événements, façades `Nc*` et handles `Nc*`. Aucun type du noyau n’apparaît dans la surface publique, donc un renommage du noyau ne force pas la reconstruction de votre mod. |
| `NetCraft.ModApi.Extension` | Attributs `[Inject]` et `[Mixin]`. Les règles nomment directement les types et méthodes du noyau, ce qui est plus puissant et plus fragile. |

## Handles

`NcPlayer` et `NcLevel` sont des handles en lecture seule. Tout ce qu’ils renvoient est une chaîne, un nombre ou un booléen simple (`NcLevel.Dimension`, `NcPlayer.X`), jamais un type du noyau, donc un renommage du noyau ne force pas la reconstruction de votre mod. Les opérations sur les blocs prennent le handle de niveau plus `x y z`.

## Appels courants

Ouvrez ce panneau depuis un projet de mod et chaque appel ci-dessous que votre code utilise réellement est évalué par rapport à `NetCraftTemplate.yaml` : vert lorsque le membre est déclaré, ambre lorsque le membre ne l’est pas, rouge lorsque le type n’est pas déclaré du tout. Survolez un nom mis en évidence pour voir la raison.

| Appel | Ce qu’il fait |
| --- | --- |
| `NcServer.IsAvailable` | si le serveur est démarré et capturé |
| `NcServer.Broadcast` | message système à tous les joueurs en ligne |
| `NcServer.Execute` | exécuter une commande en tant que console |
| `NcWorld.GetBlock` | lire un bloc, null lorsque le chunk n’est pas chargé |
| `NcWorld.SetBlock` | écrire un bloc, exécute toute la chaîne de mise à jour |
| `NcWorld.BreakBlock` | casser un bloc comme le ferait un joueur |
| `NcWorld.Overworld` | le handle de niveau de l’Overworld |
| `NcLevel.Dimension` | id de dimension d’un handle de niveau, par exemple minecraft:overworld |
| `NcLevel.DayTime` | lire ou définir l’heure d’une dimension |
| `NcPlayer.Name` | le nom du joueur |
| `NcPlayer.Health` | santé actuelle |
| `NcPlayers.Find` | rechercher un joueur en ligne par nom |
| `NcPlayers.Send` | message système privé |
| `NcRegistries.FindState` | état de bloc par id d’espace de noms |
| `NcRegistries.FindItem` | item par id d’espace de noms |
| `ServerEvents.Tick` | s’exécute à chaque tick serveur |
| `ServerEvents.PlayerJoin` | un joueur a terminé de rejoindre |
| `ServerEvents.BlockBroken` | un bloc a réellement été remplacé |