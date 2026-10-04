# Modèle de mod NetCraft

Exemples de code pour le développement de mods NetCraft. Chaque entrée de `NetCraftTemplate.yaml`
pointe vers un fichier sous `examples/`, et `ncm template` les récupère à la demande.

## Utilisation

```
ncm template view              lister toutes les entrées
ncm template view Wrapper.?    filtrer par id, ? et * sont des jokers
ncm template example <api id>  copier un fichier d'exemple dans le répertoire courant
```

## Les deux voies

`NetCraft.ModApi` expose deux espaces de noms, choisissez celui qui convient :

| Espace de noms | Ce que vous obtenez |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | Événements, façades `Nc*` et handles `Nc*`. Aucun type du noyau n'apparaît dans la surface publique, donc un renommage du noyau ne force pas à recompiler votre mod. |
| `NetCraft.ModApi.Extension` | Attributs `[Inject]` et `[Mixin]`. Les règles nomment directement les types et méthodes du noyau, ce qui est plus puissant et plus fragile. |

## Appels courants

Ouvrez ce panneau depuis un projet de mod : chaque appel ci-dessous réellement utilisé
par votre code est évalué par rapport à `NetCraftTemplate.yaml` : vert lorsque le membre
est déclaré, orange lorsqu'il ne l'est pas, rouge lorsque le type n'est pas déclaré du tout.
Survolez un nom surligné pour voir la raison.

| Appel | Ce qu'il fait |
| --- | --- |
| `NcServer.IsAvailable` | si le serveur est démarré et capturé |
| `NcServer.Broadcast` | message système à tous les joueurs en ligne |
| `NcServer.Execute` | exécuter une commande en tant que console |
| `NcWorld.GetBlock` | lire un bloc, null lorsque le chunk n'est pas chargé |
| `NcWorld.SetBlock` | écrire un bloc, exécute toute la chaîne de mise à jour |
| `NcWorld.BreakBlock` | casser un bloc comme le ferait un joueur |
| `NcPlayer.Name` | le nom du joueur |
| `NcPlayer.Health` | santé actuelle |
| `NcPlayers.Find` | rechercher un joueur en ligne par son nom |
| `NcPlayers.Send` | message système privé |
| `NcRegistries.FindState` | état de bloc par id namespacé |
| `NcRegistries.FindItem` | objet par id namespacé |
| `ServerEvents.Tick` | s'exécute à chaque tick du serveur |
| `ServerEvents.PlayerJoin` | un joueur a terminé de se connecter |
| `ServerEvents.BlockBroken` | un bloc a été réellement remplacé |
