# Référence NetCraft-ModApi

`NetCraft.ModApi` est la surface d'API que NetCraft expose aux mods. Il a deux identités : pour vous c'est une bibliothèque d'API ; pour lui-même c'est un mod ordinaire (`id` vaut `netcraft-modapi`, et il fournit son propre `ncmod.json` ainsi que ses propres sondes d'injection).

Ce fichier s'étoffe à mesure que l'API grandit. Pour le contexte architectural, les différences avec Fabric et la façon d'écrire un mod, voir [modding-guide-fr.md](modding-guide-fr.md).

- Assembly : `NetCraft.ModApi.dll`

- Dépendances : `NetCraft` (la bibliothèque principale), `NetCraft.Game`

La surface publique est répartie en trois espaces de noms :

| Espace de noms | Contenu | Remarques |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | Classe de base pour les événements et les abonnements `NcEvent<T>`, façades `Nc*`, handles d'objets `Nc*` | Couche wrapper ; aucun type du noyau dans la surface publique |
| `NetCraft.ModApi.Extension` | Annotations `[Inject]` / `[Mixin]` | Points d'extension ; les règles se lient aux noms de classes et de méthodes du noyau |
| `NetCraft.ModApi.Internal` | Sondes d'injection | Ne pas référencer directement |

L'espace de noms racine `NetCraft.ModApi` ne contient que la classe d'entrée `ModApiEntry`. `Wrapper` et `Extension` sont deux voies parallèles ; pour savoir comment choisir, voir [modding-guide-fr.md 2.9](modding-guide-fr.md#29-deux-voies--couche-wrapper-et-points-dextension).

***

## 1. Démarrage rapide

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //l'abonnement renvoie un handle ; le libérer désenregistre
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //vous pouvez aussi faire autre chose avec le handle
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

Dans `ncmod.json`, faites pointer `entry` vers cette classe et laissez `hooks` vide — les événements ci-dessous sont tous fournis par les sondes de ModApi lui-même.

***

## 2. Événements

Tous les événements se trouvent sous `NetCraft.ModApi.Wrapper` ; après `using NetCraft.ModApi.Wrapper;` ils sont disponibles.

### 2.1 Tableau récapitulatif

| Événement                           | Type d'arguments       | Déclencheur                                                   | Côté   | Point de hook ModApi                                        |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick`             | `ServerTickArgs`       | à chaque tick de la boucle principale du serveur                        | serveur | `DedicatedServer::Tick` (Mark)                           |
| `ServerEvents.Started`          | `ServerPhaseArgs`      | la boucle principale a démarré, après l'affichage de `Done (x.xxxs)!`      | serveur | `MinecraftServer::Run` (Mark)                            |
| `ServerEvents.Stopping`         | `ServerPhaseArgs`      | le serveur commence à s'arrêter ; les joueurs vont être déconnectés | serveur | `DedicatedServer::Stop` (Mark)                    |
| `ServerEvents.CommandRegister`  | `CommandRegisterArgs`  | toutes les commandes intégrées ont été enregistrées                | serveur | site d'appel de `EffectCommand::Register` (CallSite)           |
| `ServerEvents.PlayerJoin`       | `PlayerJoinArgs`       | la séquence de paquets de connexion a été envoyée                    | serveur | site d'appel de `PlayerList::PlaceNewPlayer` (CallSite)        |
| `ServerEvents.PlayerLeave`      | `PlayerLeaveArgs`      | le joueur est retiré de la liste des connectés                       | serveur | site d'appel de `PlayerList::RemovePlayer` (CallSite)          |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | le paquet de déconnexion a été envoyé et la connexion fermée              | serveur | site d'appel de `ServerPlayer::Disconnect` (CallSite)          |
| `ServerEvents.PlayerHurt`       | `PlayerHurtArgs`       | des dégâts ont été réellement infligés                                     | serveur | site d'appel de `PlayerList::HurtPlayer` (CallSite)            |
| `ServerEvents.PlayerDeath`      | `PlayerDeathArgs`      | immédiatement après que la santé soit revenue à zéro                   | serveur | site d'appel de `PlayerList::RespawnPlayer` (CallSite)         |
| `ServerEvents.PlayerChat`       | `PlayerChatArgs`       | après la diffusion du chat                                  | serveur | site d'appel de `ServerGamePacketListenerImpl::HandleChat` (CallSite) |
| `ServerEvents.ChunkLoaded`      | `ChunkLoadedArgs`      | le chunk entre en mémoire pour la première fois                    | serveur | site d'assignation de `ServerChunkCache::set_ChunkLoaded` (CallSite) |
| `ServerEvents.ChunkUnloaded`    | `ChunkUnloadedArgs`    | le chunk quitte la mémoire                                       | serveur | site d'assignation de `ServerChunkCache::set_ChunkUnloaded` (CallSite) |
| `ServerEvents.ChunkSaved`       | `ChunkSavedArgs`       | un instantané est pris avant l'écriture du chunk sur disque        | serveur | site d'assignation de `ServerChunkCache::set_ChunkSaveSink` (CallSite) |
| `ServerEvents.CommandExecuted`  | `CommandExecutedArgs`  | une commande a fini de s'exécuter ; les erreurs de syntaxe et les refus de permission comptent aussi | serveur | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick`        | `LevelTickArgs`        | tick de niveau, une fois par niveau chargé et par tick                | serveur | `PersistentServerLevel::Tick` (CallSite)                 |
| `ServerEvents.SavedDataSaving`  | `SavedDataSavingArgs`  | les données sauvegardées sont écrites sur disque, une étape après la sauvegarde des chunks | serveur | `SavedDataStorage::ScheduleSave` (CallSite)          |
| `ServerEvents.BlockChanged`     | `BlockChangedArgs`     | l'état d'un bloc a changé, il va être synchronisé aux clients             | serveur | `IBlockUpdateSink::BlockChanged` (CallSite)              |
| `ServerEvents.BlockBroken`      | `BlockBrokenArgs`      | un bloc a été cassé ; le minage par un joueur et l'auto-destruction par redstone comptent tous les deux | serveur | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped`      | `ItemDroppedArgs`      | une entité d'objet lâché est apparue, y compris les drops de casse de bloc et les produits de cuisson | serveur | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived`  | `PacketReceivedArgs`   | chaque paquet entrant mis en file vers un gestionnaire, y compris les phases handshake et status | les deux   | `PacketProcessor::ScheduleIfPossible` et `HandleNow` (CallSite) |
| `ClientEvents.Tick`             | `ClientTickArgs`       | à chaque tick de la boucle principale du client                        | client | `MinecraftClient::Tick` (Mark)                           |

Ordre des événements joueur : la mort est imbriquée dans le flux de dégâts, donc `PlayerDeath` précède le `PlayerHurt` correspondant ; `PlayerLeave` et `PlayerDisconnect` sont deux choses différentes — le premier signifie le retrait de la liste des connectés (même après un `/kick` il ne se déclenche qu'une fois la connexion perdue), le second désigne la perte de connexion elle-même, et les deux ne sont pas garantis d'apparaître par paires.

### 2.2 S'abonner et se désabonner

```csharp
IDisposable Subscribe(Action<T> handler)
```

- S'abonner plusieurs fois au même événement délivre les notifications dans l'ordre d'abonnement.

- La distribution prend un instantané de la liste des callbacks, donc s'abonner ou se désabonner depuis l'intérieur d'un callback n'affecte pas la distribution en cours.

- Sans désabonnement, il reste effectif indéfiniment ; les mods n'offrent aucun mécanisme de déchargement, donc le désabonnement manuel est généralement inutile.

### 2.3 Types d'arguments

**`ServerTickArgs`**

| Propriété    | Type   | Remarques                                |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | nombre de ticks depuis ce lancement, en commençant à 1 |

À noter : ce compteur est tenu par ModApi lui-même, pas par le `TickCount` du noyau.

**`ClientTickArgs`**

| Propriété    | Type   | Remarques                           |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | identique à ci-dessus, compté indépendamment côté client |

**`ServerPhaseArgs`**

| Propriété | Type     | Remarques                        |
| -------- | -------- | ---------------------------- |
| `Phase`  | `string` | nom de la phase, `started` ou `stopping` |

Ce champ duplique l'événement lui-même ; il est conservé pour que la journalisation puisse utiliser un format uniforme.

**`CommandRegisterArgs`**

| Membre                               | Type                                    | Remarques            |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher`                         | `CommandDispatcher<CommandSourceStack>` | le dispatcher de commandes du noyau |
| `Register(name, description, build)` | méthode                                  | enregistre une commande et la consigne dans le journal, voir 3.1 |

**`PlayerJoinArgs`**

| Propriété      | Type           | Remarques                                          |
| ------------- | -------------- | ---------------------------------------------- |
| `Player`      | `ServerPlayer` | le joueur qui vient de se connecter ; les paquets de connexion sont déjà envoyés, l'état peut être lu sans risque |
| `ProfileName` | `string`       | nom du joueur                                    |

**`PlayerLeaveArgs`**

| Propriété  | Type           | Remarques                                 |
| --------- | -------------- | ------------------------------------- |
| `Player`  | `ServerPlayer` | le joueur qui part, plus présent dans la liste des connectés à ce stade |
| `Removed` | `bool`         | s'il a réellement été retiré ; `false` lors d'un retrait répété |

**`PlayerDisconnectArgs`**

| Propriété | Type           | Remarques                                 |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | le joueur déconnecté               |
| `Reason` | `string`       | raison de la déconnexion ; texte brut lorsqu'elle est fournie sous forme de composant |

Le paquet de déconnexion a été envoyé et la connexion fermée ; envoyer des paquets à ce joueur n'a désormais aucun effet.

**`PlayerHurtArgs`**

| Propriété   | Type            | Remarques                                        |
| ---------- | --------------- | -------------------------------------------- |
| `Player`   | `ServerPlayer`  | le joueur qui a été blessé                      |
| `Attacker` | `ServerPlayer?` | le joueur qui a infligé les dégâts ; `null` pour les dégâts environnementaux et de commande |
| `Amount`   | `float`         | montant des dégâts cette fois                      |

Ne se déclenche pas pendant les frames d'invulnérabilité ni après la mort (le `Hurt` du noyau renvoie `false`).

**`PlayerDeathArgs`**

| Propriété   | Type            | Remarques                          |
| ---------- | --------------- | ------------------------------ |
| `Player`   | `ServerPlayer`  | le joueur qui est mort            |
| `Attacker` | `ServerPlayer?` | le tueur ; `null` lorsqu'il n'y en a pas |

Le noyau réinitialise immédiatement après que la santé atteint zéro, donc lorsque l'événement se déclenche, le joueur est déjà à pleine santé au point de réapparition ; les coordonnées et les drops au moment de la mort ne sont pas disponibles.

**`PlayerChatArgs`**

| Propriété     | Type     | Remarques                |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | nom de l'expéditeur          |
| `Message`    | `string` | message en texte brut   |

Cet événement est une **notification en lecture seule** : la méthode d'origine a déjà diffusé le message, donc modifier `Message` ici n'a aucun effet.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

Les trois événements partagent la même forme d'arguments :

| Propriété | Type  | Remarques              |
| -------- | ----- | ------------------ |
| `X`      | `int` | coordonnée de chunk X |
| `Z`      | `int` | coordonnée de chunk Z |

Le plus facile à se tromper est `ChunkSaved` : le noyau exige que ce callback **prenne l'instantané de façon synchrone**, tandis que la sérialisation et l'écriture sur disque sont faites de façon asynchrone par le noyau lui-même. Donc un travail long dans ce callback ralentit directement le déchargement des chunks, et les opérations destructrices (retirer des blocs, modifier des inventaires) ne doivent pas non plus aller ici — il ne promet que le moment de l'instantané.

Quand `ChunkUnloaded` se déclenche, les block entities ont déjà été nettoyées en même temps que le chunk ; si vous voulez lire des blocs, utilisez `ChunkSaved` (qui ne peut pas non plus accéder aux blocs) ou un point plus précoce.

**`CommandExecutedArgs`**

| Propriété  | Type                   | Remarques                                 |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string`               | texte brut de la commande ; les commandes de chat n'ont pas de slash initial |
| `Result`  | `int`                  | valeur de retour de la commande ; 0 signifie échec ou refus |
| `Source`  | `CommandSourceStack?`  | source de la commande ; `null` sur le chemin de la surcharge joueur |
| `Player`  | `ServerPlayer?`        | le joueur qui a émis la commande ; `null` lorsqu'elle est émise depuis la console |

L'événement se déclenche **après** la fin de la commande ; il ne peut pas modifier l'exécution. Les erreurs de syntaxe et les refus de permission passent aussi par ici ; utilisez `Result` pour les distinguer. Les commandes qu'un joueur envoie depuis la barre de chat passent par la surcharge `Execute(ServerPlayer, string)`, où le noyau construit la source de commande en interne, donc dans ce cas `Source` vaut `null` et seul `Player` est renseigné.

**`LevelTickArgs`**

| Propriété       | Type                    | Remarques                                        |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level`        | `NcLevel`               | le niveau avancé à ce tick                 |
| `RunsNormally` | `bool`                  | s'il a avancé normalement ; `false` pendant `/tick freeze` |

Se déclenche une fois par niveau chargé et par tick, donc un monde multi-niveaux en reçoit plusieurs par tick. Le point de déclenchement se situe après que le tick de niveau **a été effectué** ; c'est un point d'observation, pas un point d'interception.

**`SavedDataSavingArgs`**

| Propriété  | Type                | Remarques                       |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage`  | la table de données sauvegardées en cours de persistance |

C'est un chemin différent de `ServerEvents.ChunkSaved` : celui des chunks ne prend qu'un instantané et écrit de façon asynchrone, tandis que celui-ci se déclenche après qu'une écriture synchrone est terminée. L'horloge du monde, les règles du jeu et les données de bordure du monde passent par lui.

**`BlockChangedArgs`**

| Propriété | Type         | Remarques                       |
| -------- | ------------ | --------------------------- |
| `Pos`    | `BlockPos`   | position du bloc modifié |
| `State`  | `BlockState` | état du bloc après la modification  |

L'état a déjà été écrit dans le chunk et est sur le point d'être synchronisé aux clients, donc la modification elle-même ne peut pas être altérée ici. Les composants redstone qui changent leur propre état dans des callbacks de comportement sortent aussi par ce chemin ; il est à haute fréquence, donc n'effectuez pas de travail long dans le callback.

**`BlockBrokenArgs`**

| Propriété | Type            | Remarques                                      |
| -------- | --------------- | ------------------------------------------ |
| `Pos`    | `BlockPos`      | position du bloc cassé                |
| `Player` | `ServerPlayer?` | le casseur ; `null` pour les causes non-joueur comme la redstone |

Se déclenche uniquement lorsque le bloc est réellement remplacé ; les positions vides et les cassures rejetées ne le déclenchent pas. Les effets de casse et les drops ont déjà été traités, donc ce que vous lisez dans l'événement est le résultat.

**`ItemDroppedArgs`**

| Propriété | Type        | Remarques                   |
| -------- | ----------- | ----------------------- |
| `Pos`    | `BlockPos`  | où l'objet lâché est apparu |
| `Stack`  | `ItemStack` | la pile d'objets lâchée   |

Les drops de casse de bloc et les produits de cuisson au feu de camp passent tous les deux par ici. Une pile d'objets vide ne fait apparaître aucune entité, donc il n'y a pas d'événement.

**`PacketReceivedArgs`**

| Propriété        | Type     | Remarques                    |
| --------------- | -------- | ------------------------ |
| `Listener`      | `object` | le listener qui reçoit ce paquet |
| `Packet`        | `object` | l'objet paquet lui-même  |
| `IsServerbound` | `bool`   | s'il s'agit d'un paquet serverbound |

Se déclenche pour chaque paquet entrant, couvrant les quatre phases : handshake, status, configuration et play. Les paquets de déplacement arrivent plusieurs fois par tick, donc n'effectuez pas de travail long dans le callback. Le paquet est déjà décodé en objet mais n'est pas encore entré dans la couche métier ; pour distinguer les types, inspectez `Packet` vous-même. Les paquets sortants sont hors du périmètre de cet événement.

***

## 3. Points d'extension

### 3.1 Enregistrement de commandes

Le moment est `ServerEvents.CommandRegister`. Ne mettez pas en cache les arguments de cet événement ; l'arbre de commandes interne n'est construit qu'une seule fois au démarrage.

```csharp
public void Register(
    string name,                                          //littéral de commande, sans le slash
    string description,                                   //description sur une ligne, affichée dans le journal de /ncmapi
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attacher les arguments et l'exécuteur
```

Le `build` que vous recevez est le builder brigadier du noyau ; écrivez les arguments, les sous-commandes et les prédicats de permission à la manière du noyau :

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //prédicat de permission
            .Executes(context => { /* ... */ return 1; })));
```

Avec des arguments :

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //à titre d'illustration
                return 1;
            })));
```

Points clés :

- Appeler directement `args.Dispatcher.Register(...)` installe aussi une commande, mais elle n'entre pas dans le journal et `/ncmapi` ne l'affichera pas. Utilisez `args.Register` si vous voulez qu'elle soit listée.

- Les commandes n'ont aucune restriction de permission par défaut ; ajoutez `.Requires(...)` vous-même si nécessaire.

- Le comportement à l'exécution vous appartient entièrement ; ModApi ne l'intercepte pas.

### 3.2 Consulter les commandes enregistrées

Il existe un `/ncmapi` intégré, nécessitant le niveau de permission 2 :

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

Les lignes d'utilisation sont calculées à la volée à partir de la structure des nœuds de l'arbre de commandes : les littéraux sont écrits par leur nom, les arguments sont encadrés par des chevrons, et les nœuds intermédiaires qui sont eux-mêmes exécutables obtiennent aussi leur propre ligne.

***

## 4. Façades serveur

Les façades de ce chapitre se trouvent toutes sous `NetCraft.ModApi.Wrapper` ; après `using NetCraft.ModApi.Wrapper;` elles sont disponibles.

Les façades sont des classes statiques `Nc*` qui rassemblent en quelques points d'entrée des capacités dispersées dans le noyau. L'instance du noyau est capturée par une sonde au démarrage de la boucle principale ; quand `NcServer.IsAvailable` est faux, tout ce qui suit lève une exception — utilisez-les uniquement à l'intérieur de callbacks d'événements, pas depuis `Init`.

| Façade | Rôle |
| --- | --- |
| `NcServer` | instance du serveur, fréquence de tick, commandes, suivi des entités, données des joueurs, règles du jeu, diffusion, exécution de commandes |
| `NcPlayers` | requêtes et opérations sur les joueurs en ligne (kick, téléportation, santé, mode de jeu, permissions) |
| `NcWorld` | lecture/écriture et casse de blocs dans l'overworld, météo, temps, bordure, horloge, sons, événements de niveau ; prend des handles `NcLevel` pour atteindre d'autres dimensions, les coordonnées sont de simples entiers `x y z` |
| `NcRegistries` | registres intégrés recherchés par nom (blocs, objets, fluides, effets, biomes, particules, entités, block entities) |
| `NcRecipes` | requêtes de recettes (artisanat sur grille, découpe de pierre, cuisson ; récupérer des recettes par id) |
| `NcLists` | listes et configuration (whitelist, ops, bans, `server.properties`) |
| `NcStartup` | arguments de démarrage (tokens non reconnus par le noyau et abonnement par nom) |

`NcPlayer` n'est pas une façade statique mais un **handle d'objet** : `NcPlayers.All` / `Find` le renvoient, et `Player` / `Attacker` dans les événements joueur le sont aussi. Les handles sont en lecture seule et construits par les sondes ; les mods ne peuvent pas obtenir le `ServerPlayer` du noyau — le premier point d'ancrage de « aucun type du noyau dans la surface publique ». Le même joueur du noyau correspond toujours au même handle, mis en cache en interne par référence faible et automatiquement invalidé une fois le joueur déconnecté.

`NcLevel` suit la même forme pour les niveaux. `NcWorld.Overworld` / `Nether` / `End` et `NcWorld.Get("minecraft:the_nether")` le renvoient, et `LevelTickArgs.Level` en est un aussi. Il porte l'id de dimension, le temps, la météo, la hauteur de construction, le nombre de ticks et le force-loading des chunks ; les opérations sur les blocs restent sur `NcWorld` et prennent le handle plus `x y z`. `BlockPos` n'apparaît jamais, donc la dll d'un mod ne porte aucune référence au type de niveau du noyau.

### 4.1 Registres

`NcRegistries` fournit à la fois des tables complètes et des recherches par nom. Les tables complètes servent à l'itération et à la recherche par tag ; les recherches par nom servent à obtenir un seul élément :

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //état par défaut du bloc
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//parcourir toute la table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

Les registres sont assemblés progressivement pendant le démarrage, et les mods sont chargés avant que l'assemblage soit terminé, donc ne mettez en cache rien de ce qui est recherché dans `Init` — l'assemblage est encore en cours, et une valeur mise en cache sera une référence nulle ou une valeur périmée. Actuellement `BuiltInRegistries.BootStrap` est encore une implémentation vide ; chaque registre est peuplé séparément par son propre Bootstrap, et ceux pilotés par les données (biomes, recettes, etc.) ont très peu d'entrées avant que le chargement des data packs soit branché.

### 4.2 Recettes

`NcRecipes` s'appuie sur une table de recettes chargée depuis les data packs ; `/reload` remplace toute la table, donc ne conservez pas de `RecipeHolder` à travers les rechargements.

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //calculer la sortie pour une grille d'artisanat
    var recipes = NcRecipes.StonecuttingFor(stack);         //recettes de découpe de pierre disponibles pour cette entrée
    var smelting = NcRecipes.CookingFor("smelting", stack); //rechercher par type de cuisson
    var byId = NcRecipes.Find("minecraft:oak_planks");      //récupérer une recette par id
}
```

***

## 5. Internes

Vous n'avez pas besoin de cette section pour écrire des mods, mais elle peut aider lors du débogage.

### 5.1 Sondes

| Classe                                                    | Forme        | Responsabilité                                              |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)`                  | Mark ×4     | tous les signaux « quelque chose s'est produit » convergent vers une seule méthode, distribuée vers l'événement correspondant via `label` |
| `Internal.CommandProbe.OnCommandsReady(object)`          | CallSite    | remplace l'appel à `EffectCommand::Register` ; après avoir restauré l'appel d'origine, il déclenche `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)`                | CallSite ×6 | événements joueur, une méthode par point de hook ; après avoir restauré l'appel d'origine, elle publie l'événement |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | événements de chunk, hookés aux sites d'assignation des trois propriétés de callback de `ServerChunkCache` ; un délégué wrapper est superposé avant de rendre le contrôle au noyau |
| `Internal.BlockProbe.OnXxx(...)`                         | CallSite ×3 | événements de bloc ; la casse et les drops hookent `ServerBlockUpdates`, les changements d'état hookent la méthode d'interface sur `IBlockUpdateSink` |

La signature de `SignalProbe` ne prend que `string`, et les paramètres de `CommandProbe`, `PlayerProbe` et `LevelProbe` sont déclarés en `object` — c'est délibéré : pendant l'assemblage, `Lead.Hook` résout la signature de la méthode de remplacement, et dès qu'un type du noyau y apparaît, sa résolution remonte l'assembly du noyau prématurément et l'injection manque sa fenêtre. Les types du noyau n'apparaissent qu'à l'intérieur des corps de méthode, moment où le code est déjà en cours d'exécution.

Les seules choses qui ne peuvent pas être `object` dans une signature sont les paramètres et valeurs de retour de types valeur : `object` est une référence sur la pile tandis que `float`/`bool` sont des valeurs, et une incohérence donne un IL invalide. Ainsi `PlayerProbe.OnHurtPlayer` conserve `float` pour le montant des dégâts, et `OnRemovePlayer` et `OnHurtPlayer` conservent des valeurs de retour `bool`.

`BlockProbe` est une extension de cette contrainte : la position et l'état de bloc sont les deux types valeur `BlockPos`/`BlockState`, qui ne peuvent être écrits dans la signature que tels quels. Ces deux types viennent de `NetCraft.Primitives` et `NetCraft.Registry`, dont aucun n'est dans la liste d'injection, donc les résoudre pendant l'assemblage ne remonte pas prématurément les assemblies à réécrire.

### 5.2 Liste des points de hook

Le `ncmod.json` de ModApi contient vingt-quatre règles, correspondant une à une au tableau de la section 2.1. Pour changer un point de hook ou ajouter une règle, modifiez ce fichier ; après modification, recompilez (c'est une ressource embarquée) et replacez la dll résultante dans `mods/` — cette dernière étape est déjà faite automatiquement par `DeployModToHosts` dans `NetCraft.ModApi.csproj`, et son absence se manifeste par des règles qui ne prennent pas effet du tout.

`CommandManager::Execute` a deux surcharges qui partagent une seule règle. Le CallSite de `Lead.Hook` fait correspondre les sites d'appel par « type + nom de méthode », pas par liste de paramètres, et les deux surcharges prennent deux paramètres, donc la sonde peut prendre `object` pour le premier paramètre et distribuer selon le type réel.

Les deux règles `PacketProcessor` sont complémentaires : les paquets de la phase play passent par `ScheduleIfPossible` dans la file du thread principal, tandis que les phases handshake et status passent par `HandleNow` pour un traitement immédiat ; un paquet donné ne passe que par l'un des deux. Ne hooker que le premier manque les phases handshake et status — qui se trouvent être les plus faciles à sonder avec des scripts, donc en débogage cela se lit facilement à tort comme « la règle n'a pas pris effet ».

Les trois règles de chunk hookent les **sites d'assignation** des trois propriétés de callback de `ServerChunkCache`, pas les sites de lecture. La raison est que ces trois propriétés sont unicast et déjà occupées par le noyau lui-même lors de la construction de `PersistentServerLevel` (elles injectent la logique de sauvegarde et de nettoyage des block entities) ; un mod qui assignerait directement écraserait la copie du noyau — déchargements non persistés, block entities non nettoyées, et sans aucune erreur. Hooker le site d'assignation permet d'enchaîner à ce moment le callback du noyau et la sonde ; l'assignation n'a lieu qu'une fois, et chaque déclenchement ultérieur ajoute une couche de délégation.

`PlayerList::RespawnPlayer` est privée, donc la sonde ne peut pas restaurer l'appel d'origine ; celle-là passe par réflexion (appelée une fois par mort, donc le surcoût est négligeable). Cela laisse aussi une porte ouverte pour l'alignement avec le noyau : si `InternalsVisibleTo` lui est ajouté à l'avenir, elle pourra être remplacée par un appel direct.

### 5.3 Journal

`Internal.NcCommandRegistry` enregistre les commandes enregistrées via `args.Register`. Ce n'est qu'un journal et il ne participe pas à l'exécution des commandes ; les commandes elles-mêmes sont installées sur le dispatcher du noyau, donc même si le journal a des problèmes, les commandes fonctionnent toujours.

***

## 6. À ajouter

Voici des points de hook dont l'emplacement est confirmé mais qui ne sont pas encore devenus des événements (la liste a été produite par `__scan_mod_api.py` à la racine du dépôt) :

| Direction         | Points de hook candidats                                                        |
| ----------------- | ---------------------------------------------------------------------------- |
| Entités          | `ClientLevel::AddEntity`, `Entity::Die`                                      |
| Monde             | chargement et déchargement de niveau, traitement par lots des chunks de `ServerChunkCache`                     |
| Génération de terrain | étapes de `ChunkGenerator::Generate` par `ChunkStatus`, `WorldGenRegion::SetBlockState` |
| Exécution de commandes | `CommandSourceStack::SendSuccess` / `SendFailure` (la moitié réponse, avec de nombreux sites d'appel) |
| Réseau           | `ServerGamePacketListenerImpl::HandleXxx` par type de paquet (actuellement seulement un point d'entrée unifié) |

Directions déjà traitées : les ticks de niveau sont devenus `ServerEvents.LevelTick`, la persistance des données sauvegardées est devenue `ServerEvents.SavedDataSaving`, l'exécution de commandes est devenue `ServerEvents.CommandExecuted`, et les blocs sont devenus `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`.

Hooker la cellule block sur `SetBlock` ne fonctionne pas : elle a deux paramètres par défaut, `notifyNeighbors` et `strict`, donc les sites d'appel compilés prennent de 4 à 6 paramètres, et comme CallSite fait correspondre par « type + nom de méthode » sans regarder la liste des paramètres, une seule méthode de remplacement ne peut pas gérer les trois formes de pile. À la place, il hooke deux endroits : la synchronisation d'état hooke `IBlockUpdateSink::BlockChanged` (le seul appel d'interface dans `ServerLevel.SetBlock`, couvrant chaque changement avec synchronisation client), et la casse et les drops hookent les méthodes propres à `ServerBlockUpdates`.

La cellule entities est problématique car `Entity` est défini dans `NetCraft.Registry`, qui n'est pas dans la liste d'injection, donc les sites d'appel qui le ciblent ne peuvent pas être réécrits. `ClientLevel::AddEntity` est dans l'assembly client et est faisable ; un événement de mort exige d'abord de résoudre si `Registry` peut être réécrit.

Pour la cellule network, `HandleChat` existe depuis longtemps et `NetworkEvents.PacketReceived` fournit un point d'entrée unifié, donc les hooks par `HandleXxx` ont bien moins de valeur ; seuls les scénarios nécessitant un filtrage fin par type de paquet valent la peine d'être ajoutés.

Le chargement et le déchargement de niveau n'ont pas de point de convergence côté NC : `DedicatedServer::CreateLevel` est privée, donc l'appel d'origine ne peut être restauré que par réflexion, comme `PlayerList::RespawnPlayer` ; le chemin de déchargement est encore plus dispersé. Pour le faire, il faut d'abord décider ce que devraient être les arguments de l'événement.

Tous les autres ont besoin de références d'objets (instances d'entités, etc.), donc ils doivent utiliser `CallSite` plutôt que `Mark` ; si un type valeur du noyau apparaît parmi les paramètres, il ne peut être écrit dans la signature de la méthode de remplacement que tel quel.

La liste de la surface d'interface (`__modapi_api.txt`, produite par `__scan_mod_api.py --api`) a aussi été passée en revue : les entrées qualifiées de points d'entrée de capacité ont été rassemblées dans les façades du chapitre 4 par domaine, et le reste qui n'est pas exposé se répartit en trois catégories — protocole et gestion des paquets (`Network.Protocol.*`), rendu et modèles (`Client.Render.*`), et génération de terrain et fonctions de densité (`LevelGen.*`). Ce sont des internes du noyau ; les utiliser directement lierait les mods aux détails d'implémentation, donc une interface stable devrait d'abord être ouverte dans le noyau.

Côté serveur, il y a deux choses de plus non emballées en façades : l'instance `ReloadableServerResources` est rattachée à `DedicatedServer`, et comme ModApi ne référence pas `NetCraft.Server`, l'emballer exige d'abord d'ouvrir une propriété sur la classe de base du noyau ; `ChunkSender` et `ServerWorldBorderListener` sont des flux internes sans cas d'usage pour les mods.
