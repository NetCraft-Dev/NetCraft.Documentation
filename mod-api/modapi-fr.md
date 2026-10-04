# Référence NetCraft-ModApi

`NetCraft.ModApi` est la surface API que NetCraft expose aux mods. Elle a deux identités : pour vous c'est une bibliothèque API ; en soi, il s'agit d'un mod ordinaire (`id` est `netcraft-modapi`, livrant son propre `ncmod.json` et ses sondes d'injection).

Ce fichier grandit à mesure que l'API grandit. Pour connaître le contexte architectural, les différences par rapport à Fabric et comment écrire un mod, voir [modding-guide.md](modding-guide.md).

- Assemblage : `NetCraft.ModApi.dll`

- Dépendances : `NetCraft` (la bibliothèque principale), `NetCraft.Game`

La surface publique est divisée en trois espaces de noms :

| Espace de noms | Contenu | Remarques |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | Classe de base d'événements et d'abonnements `NcEvent<T>`, façades `Nc*`, poignées d'objets `Nc*` | Couche d'emballage ; aucun type de noyau sur la surface publique |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` annotations | Points de rallonge ; règles liées aux noms de classes et de méthodes du noyau |
| `NetCraft.ModApi.Internal` | Sondes d'injection | Ne faites pas référence directement |

L'espace de noms racine `NetCraft.ModApi` contient uniquement la classe d'entrée `ModApiEntry`. `Wrapper` et `Extension` sont deux routes parallèles ; pour savoir comment choisir, voir [modding-guide.md 2.9](modding-guide.md#29-two-routes-wrapper-layer-and-extension-points).

***

## 1. Démarrage rapide

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

Dans `ncmod.json`, pointez `entry` sur cette classe et laissez `hooks` vide — les événements ci-dessous sont tous fournis par les propres sondes de ModApi.

***

## 2. Événements

Tous les événements sont en direct sous `NetCraft.ModApi.Wrapper` ; après `using NetCraft.ModApi.Wrapper;` ils sont disponibles.

### 2.1 Tableau récapitulatif

| Événement | Type d'arguments | Déclencheur | Côté | Point d'accroche ModApi |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick` | `ServerTickArgs` | chaque tick de la boucle principale du serveur | serveur | `DedicatedServer::Tick` (Marque) |
| `ServerEvents.Started` | `ServerPhaseArgs` | boucle principale démarrée, après l'impression de `Done (x.xxxs)!` | serveur | `MinecraftServer::Run` (Marque) |
| `ServerEvents.Stopping` | `ServerPhaseArgs` | le serveur commence à s'arrêter ; joueurs sont sur le point d'être déconnectés | serveur | `DedicatedServer::Stop` (Marque) |
| `ServerEvents.CommandRegister` | `CommandRegisterArgs` | toutes les commandes intégrées ont été enregistrées | serveur | `EffectCommand::Register` site d'appel (CallSite) |
| `ServerEvents.PlayerJoin` | `PlayerJoinArgs` | la séquence de paquets de jointure a été envoyée | serveur | `PlayerList::PlaceNewPlayer` site d'appel (CallSite) |
| `ServerEvents.PlayerLeave` | `PlayerLeaveArgs` | joueur retiré de la liste en ligne | serveur | `PlayerList::RemovePlayer` site d'appel (CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | déconnecter le paquet envoyé et la connexion fermée | serveur | `ServerPlayer::Disconnect` site d'appel (CallSite) |
| `ServerEvents.PlayerHurt` | `PlayerHurtArgs` | dégâts réellement infligés | serveur | `PlayerList::HurtPlayer` site d'appel (CallSite) |
| `ServerEvents.PlayerDeath` | `PlayerDeathArgs` | immédiatement après la réinitialisation de la santé à zéro | serveur | `PlayerList::RespawnPlayer` site d'appel (CallSite) |
| `ServerEvents.PlayerChat` | `PlayerChatArgs` | après la diffusion du chat | serveur | `ServerGamePacketListenerImpl::HandleChat` site d'appel (CallSite) |
| `ServerEvents.ChunkLoaded` | `ChunkLoadedArgs` | morceau entre en mémoire pour la première fois | serveur | Site d'affectation `ServerChunkCache::set_ChunkLoaded` (CallSite) |
| `ServerEvents.ChunkUnloaded` | `ChunkUnloadedArgs` | morceau laisse la mémoire | serveur | Site d'affectation `ServerChunkCache::set_ChunkUnloaded` (CallSite) |
| `ServerEvents.ChunkSaved` | `ChunkSavedArgs` | instantané pris avant que le fragment ne soit écrit sur le disque | serveur | Site d'affectation `ServerChunkCache::set_ChunkSaveSink` (CallSite) |
| `ServerEvents.CommandExecuted` | `CommandExecutedArgs` | une commande a fini de s'exécuter ; les erreurs de syntaxe et les refus d'autorisation comptent également | serveur | `CommandManager::Execute` (AppelSite) |
| `ServerEvents.LevelTick` | `LevelTickArgs` | tick de niveau, une fois par niveau chargé et par tick | serveur | `PersistentServerLevel::Tick` (AppelSite) |
| `ServerEvents.SavedDataSaving` | `SavedDataSavingArgs` | données enregistrées écrites sur le disque, une étape après la sauvegarde en bloc | serveur | `SavedDataStorage::ScheduleSave` (AppelSite) |
| `ServerEvents.BlockChanged` | `BlockChangedArgs` | état du bloc modifié, sur le point d'être synchronisé avec les clients | serveur | `IBlockUpdateSink::BlockChanged` (AppelSite) |
| `ServerEvents.BlockBroken` | `BlockBrokenArgs` | bloc cassé ; l'exploitation minière des joueurs et l'autodestruction de Redstone comptent toutes deux | serveur | `ServerBlockUpdates::BreakBlock` (AppelSite) |
| `ServerEvents.ItemDropped` | `ItemDroppedArgs` | entité d'objet abandonnée générée, y compris les chutes de rupture de bloc et les produits de cuisine | serveur | `ServerBlockUpdates::SpawnDrop` (AppelSite) |
| `NetworkEvents.PacketReceived` | `PacketReceivedArgs` | chaque paquet entrant mis en file d'attente vers un gestionnaire, y compris les phases de prise de contact et d'état | les deux | `PacketProcessor::ScheduleIfPossible` et `HandleNow` (CallSite) |
| `ClientEvents.Tick` | `ClientTickArgs` | chaque tick de la boucle principale du client | cliente | `MinecraftClient::Tick` (Marque) |

Ordre des événements des joueurs : la mort est imbriquée dans le flux des blessures, donc `PlayerDeath` précède le `PlayerHurt` correspondant ; `PlayerLeave` et `PlayerDisconnect` sont deux choses différentes — le premier signifie la suppression de la liste en ligne (même après `/kick`, il ne se déclenche qu'une fois la connexion interrompue), le second signifie que la connexion elle-même est interrompue, et les deux ne sont pas garantis d'apparaître par paires.

### 2.2 Abonnement et désinscription

```csharp
IDisposable Subscribe(Action<T> handler)
```

- S'abonner plusieurs fois au même événement envoie des notifications dans l'ordre d'abonnement.

- Dispatch prend un instantané de la liste de rappel, donc l'abonnement ou la désinscription depuis un rappel n'affecte pas la répartition actuelle.

- Sans désinscription, il reste effectif pour toujours ; les mods ne fournissent aucun mécanisme de déchargement, donc la désinscription manuelle est généralement inutile.

### 2.3 Types d'arguments

**`ServerTickArgs`**

| Propriété | Tapez | Remarques |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | nombre de ticks depuis ce lancement, à partir de 1 |

Notez que cela est compté par ModApi lui-même, et non par le `TickCount` du noyau.

**`ClientTickArgs`**

| Propriété | Tapez | Remarques |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | comme ci-dessus, compté indépendamment du côté client |

**`ServerPhaseArgs`**

| Propriété | Tapez | Remarques |
| -------- | -------- | ---------------------------- |
| `Phase` | `string` | nom de phase, `started` ou `stopping` |

Le champ duplique l'événement lui-même ; il est conservé afin que la journalisation puisse utiliser un format uniforme.

**`CommandRegisterArgs`**

| Membre | Tapez | Remarques |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher` | `CommandDispatcher<CommandSourceStack>` | le répartiteur de commandes du noyau |
| `Register(name, description, build)` | méthode | enregistrer une commande et l'enregistrer dans le grand livre, voir 3.1 |

**`PlayerJoinArgs`**

| Propriété | Tapez | Remarques |
| ------------- | -------------- | ---------------------------------------------- |
| `Player` | `ServerPlayer` | le joueur qui vient de rejoindre ; rejoindre les paquets déjà envoyés, l'état peut être lu en toute sécurité |
| `ProfileName` | `string` | nom du joueur |

**`PlayerLeaveArgs`**

| Propriété | Tapez | Remarques |
| --------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | le joueur sortant, qui n'est plus dans la liste en ligne à ce stade |
| `Removed` | `bool` | si effectivement supprimé ; `false` lors de suppressions répétées |

**`PlayerDisconnectArgs`**

| Propriété | Tapez | Remarques |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | le joueur déconnecté |
| `Reason` | `string` | raison de la déconnexion ; texte brut lorsqu'il est donné en tant que composant |

Le paquet de déconnexion a été envoyé et la connexion fermée ; l'envoi de paquets à ce joueur n'a désormais aucun effet.

**`PlayerHurtArgs`**

| Propriété | Tapez | Remarques |
| ---------- | --------------- | -------------------------------------------- |
| `Player` | `ServerPlayer` | le joueur blessé |
| `Attacker` | `ServerPlayer?` | le joueur qui a infligé les dégâts ; `null` pour les dommages environnementaux et de commandement |
| `Amount` | `float` | montant des dégâts cette fois |

Ne se déclenche pas pendant les frames d'invulnérabilité ou après la mort (le `Hurt` du noyau renvoie `false`).

**`PlayerDeathArgs`**

| Propriété | Tapez | Remarques |
| ---------- | --------------- | ------------------------------ |
| `Player` | `ServerPlayer` | le joueur décédé |
| `Attacker` | `ServerPlayer?` | le tueur ; `null` quand il n'y en a pas |

Le noyau se réinitialise immédiatement après que la santé atteint zéro, donc lorsque l'événement se déclenche, le joueur est déjà en pleine santé au point de réapparition ; les coordonnées et les largages au moment du décès ne sont pas disponibles.

**`PlayerChatArgs`**

| Propriété | Tapez | Remarques |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | nom de l'expéditeur |
| `Message` | `string` | message en texte brut |

Cet événement est une **notification en lecture seule** : la méthode d'origine a déjà diffusé le message, donc modifier `Message` ici n'a aucun effet.

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

Les trois événements partagent la même forme d'arguments :

| Propriété | Tapez | Remarques |
| -------- | ----- | ------------------ |
| `X` | `int` | coordonnée du morceau X |
| `Z` | `int` | coordonnée du morceau Z |

Le plus simple à se tromper est `ChunkSaved` : le noyau nécessite ce rappel pour **prendre l'instantané de manière synchrone**, tandis que la sérialisation et les écritures sur le disque sont effectuées de manière asynchrone par le noyau lui-même. Ainsi, le travail fastidieux de ce rappel ralentit directement le déchargement des morceaux, et les opérations destructrices (suppression de blocs, modification des inventaires) ne devraient pas non plus avoir lieu ici - cela ne promet qu'un moment instantané.

Lorsque `ChunkUnloaded` se déclenche, les entités de bloc ont déjà été nettoyées avec le morceau ; si vous souhaitez lire des blocs, utilisez `ChunkSaved` (qui ne peut pas non plus accéder aux blocs) ou un point antérieur.

**`CommandExecutedArgs`**

| Propriété | Tapez | Remarques |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string` | texte de commande brut ; les commandes de chat ne comportent pas de barre oblique |
| `Result` | `int` | valeur de retour de commande ; 0 signifie échec ou refus |
| `Source` | `CommandSourceStack?` | source de commande ; `null` sur le chemin de la surcharge du joueur |
| `Player` | `ServerPlayer?` | le joueur qui a donné l'ordre ; `null` lorsqu'il est émis depuis la console |

L'événement se déclenche **après** la fin de la commande ; il ne peut pas changer l'exécution. Des erreurs de syntaxe et des refus d'autorisation apparaissent également ici ; utilisez `Result` pour les distinguer. Les commandes qu'un joueur envoie depuis la barre de discussion passent par la surcharge `Execute(ServerPlayer, string)`, où le noyau construit la source de commande en interne, donc dans ce cas, `Source` est `null` et seul `Player` est défini.

**`LevelTickArgs`**

| Propriété | Tapez | Remarques |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level` | `NcLevel` | le niveau a avancé ce tick |
| `RunsNormally` | `bool` | s'il a progressé normalement ; `false` pendant `/tick freeze` |

Se déclenche une fois par niveau chargé et par tick, donc un monde à plusieurs niveaux en reçoit plusieurs par tick. Le point de déclenchement se situe après que le tick de niveau **est terminé** ; c'est un point d'observation, pas un point d'interception.

**`SavedDataSavingArgs`**

| Propriété | Tapez | Remarques |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage` | la table de données enregistrées étant conservée |

Il s'agit d'un chemin différent de `ServerEvents.ChunkSaved` : le morceau prend uniquement un instantané et écrit de manière asynchrone, alors que celui-ci se déclenche une fois une écriture synchrone terminée. L'horloge mondiale, les règles du jeu et les données sur les frontières mondiales y passent.

**`BlockChangedArgs`**

| Propriété | Tapez | Remarques |
| -------- | ------------ | --------------------------- |
| `Pos` | `BlockPos` | position du bloc modifié |
| `State` | `BlockState` | état de blocage après le changement |

L'état a déjà été écrit dans le bloc et est sur le point d'être synchronisé avec les clients, la modification elle-même ne peut donc pas être modifiée ici. Les composants Redstone qui changent leur propre état lors des rappels de comportement sortent également par ce chemin ; il s'agit d'une haute fréquence, vous ne devez donc pas effectuer de travail fastidieux lors du rappel.

**`BlockBrokenArgs`**

| Propriété | Tapez | Remarques |
| -------- | --------------- | ------------------------------------------ |
| `Pos` | `BlockPos` | position du bloc cassé |
| `Player` | `ServerPlayer?` | le disjoncteur ; `null` pour les causes non-joueurs telles que Redstone |

Se déclenche uniquement lorsque le bloc est réellement remplacé ; les positions vides et les breaks rejetés ne le déclenchent pas. Les effets de rupture et les drops ont déjà été gérés, donc ce que vous lisez dans l'événement est le résultat.

**`ItemDroppedArgs`**

| Propriété | Tapez | Remarques |
| -------- | ----------- | ----------------------- |
| `Pos` | `BlockPos` | où l'élément déposé est apparu |
| `Stack` | `ItemStack` | la pile d'éléments déposés |

Les chutes de blocs et les produits de cuisson au feu de camp passent tous deux par ici. Une pile d'éléments vide ne génère aucune entité, donc il n'y a pas d'événement.

**`PacketReceivedArgs`**

| Propriété | Tapez | Remarques |
| --------------- | -------- | ------------------------ |
| `Listener` | `object` | l'auditeur recevant ce paquet |
| `Packet` | `object` | l'objet paquet lui-même |
| `IsServerbound` | `bool` | s'il s'agit d'un paquet lié au serveur |

Se déclenche pour chaque paquet entrant, couvrant les quatre phases : prise de contact, état, configuration et lecture. Les paquets de mouvement arrivent plusieurs fois par tick, vous n'avez donc pas besoin de faire un travail fastidieux lors du rappel. Le paquet est déjà décodé en objet mais n'est pas entré dans la couche métier ; pour distinguer les types, inspectez vous-même `Packet`. Les paquets sortants sortent du champ d'application de cet événement.

***

## 3. Points d'extension

### 3.1 Enregistrement des commandes

Le timing est `ServerEvents.CommandRegister`. Ne mettez pas en cache les arguments de cet événement ; l'arborescence de commandes interne n'est construite qu'une seule fois au démarrage.

```csharp
public void Register(
    string name,                                          //command literal, without the slash
    string description,                                   //one-line description, shown in the /ncmapi ledger
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attach arguments and the executor
```

Le `build` que vous recevez est le brigadier builder du noyau ; écrire des arguments, des sous-commandes et des autorisations dépend de la manière du noyau :

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //permission predicate
            .Executes(context => { /* ... */ return 1; })));
```

Avec arguments :

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

Points clés :

- L'appel direct de `args.Dispatcher.Register(...)` installe également une commande, mais elle n'entre pas dans le grand livre et `/ncmapi` ne l'affichera pas. Utilisez `args.Register` si vous souhaitez qu'il soit répertorié.

- Les commandes n'ont aucune restriction d'autorisation par défaut ; ajoutez vous-même `.Requires(...)` si nécessaire.

- Le comportement au moment de l'exécution dépend entièrement de vous ; ModApi ne l'intercepte pas.

### 3.2 Affichage des commandes enregistrées

Il existe un `/ncmapi` intégré, nécessitant le niveau d'autorisation 2 :

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

Les lignes d'utilisation sont calculées à la volée à partir de la structure des nœuds de l'arborescence de commandes : les littéraux sont écrits par leur nom, les arguments sont entourés de crochets angulaires et les nœuds intermédiaires qui sont eux-mêmes exécutables reçoivent également leur propre ligne.

***

## 4. Façades de serveurs

Les façades de ce chapitre vivent toutes sous `NetCraft.ModApi.Wrapper` ; après `using NetCraft.ModApi.Wrapper;` ils sont disponibles.

Les façades sont des classes statiques `Nc*` qui rassemblent des capacités dispersées dans le noyau en quelques points d'entrée. L'instance du noyau est capturée par une sonde lorsque la boucle principale démarre ; lorsque `NcServer.IsAvailable` est faux, tout ce qui suit est lancé — utilisez-les uniquement dans les rappels d'événements, pas à partir de `Init`.

| Façade | Objectif |
| --- | --- |
| `NcServer` | instance de serveur, taux de ticks, commandes, suivi des entités, données des joueurs, règles du jeu, diffusion, exécution des commandes |
| `NcPlayers` | requêtes et opérations des joueurs en ligne (coup de pied, téléportation, santé, mode de jeu, autorisations) |
| `NcWorld` | lecture/écriture et coupure des blocs du monde entier, météo, heure, bordure, horloge, sons, événements de niveau ; prend des poignées `NcLevel` pour atteindre d'autres dimensions, les coordonnées sont simples `x y z` entiers |
| `NcRegistries` | registres intégrés recherchés par nom (blocs, éléments, fluides, effets, biomes, particules, entités, entités de bloc) |
| `NcRecipes` | requêtes de recettes (création de grilles, taille de pierre, cuisine ; récupérer des recettes par identifiant) |
| `NcLists` | listes et configuration (liste blanche, opérations, bans, `server.properties`) |
| `NcStartup` | arguments de démarrage (jetons non reconnus par le noyau et abonnement basé sur le nom) |

`NcPlayer` n'est pas une façade statique mais un **handle d'objet** : `NcPlayers.All` / `Find` le renvoie, et `Player` / `Attacker` dans les événements du joueur le sont également. Les handles sont en lecture seule et construits par des sondes ; les mods ne peuvent pas obtenir le `ServerPlayer` du noyau — la première ancre de "aucun type de noyau sur la surface publique". Le même lecteur du noyau correspond toujours au même handle, mis en cache en interne par une référence faible et automatiquement invalidé une fois que le lecteur se déconnecte.

`NcLevel` suit la même forme pour les niveaux. `NcWorld.Overworld` / `Nether` / `End` et `NcWorld.Get("minecraft:the_nether")` le renvoient, et `LevelTickArgs.Level` en est un également. Il contient l'identifiant de dimension, l'heure, la météo, la hauteur de construction, le nombre de ticks et le chargement forcé des morceaux ; les opérations de bloc restent sur `NcWorld` et prennent le handle plus `x y z`. `BlockPos` n'apparaît jamais, donc la DLL d'un mod ne contient aucune référence au type au niveau du noyau.

### 4.1 Registres

`NcRegistries` fournit à la fois des tables entières et des recherches par nom. Des tableaux entiers sont destinés à l'itération et à la recherche basée sur des balises ; les recherches par nom servent à obtenir un seul élément :

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //block default state
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//iterate the whole table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

Les registres sont assemblés progressivement au démarrage et les mods se chargent avant la fin de l'assemblage, donc ne mettez rien en cache dans `Init` — l'assemblage est toujours en cours et une valeur mise en cache sera une référence nulle ou une valeur obsolète. Actuellement, `BuiltInRegistries.BootStrap` est encore une implémentation vide ; chaque registre est rempli séparément par son propre Bootstrap, et ceux basés sur les données (biomes, recettes, etc.) ont très peu d'entrées avant que le chargement du pack de données ne soit câblé.

### 4.2 Recettes

`NcRecipes` est soutenu par une table de recettes chargée à partir de packs de données ; `/reload` remplace la table entière, donc ne tenez pas un `RecipeHolder` lors des rechargements.

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

## 5. Internes

Vous n'avez pas besoin de cette section pour écrire des mods, mais elle peut être utile lors du débogage.

### 5.1 Sondes

| Classe | Formulaire | Responsabilité |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)` | Marque ×4 | tous les signaux « quelque chose s'est produit » sont regroupés dans une seule méthode, envoyée à l'événement correspondant par `label` |
| `Internal.CommandProbe.OnCommandsReady(object)` | AppelSite | remplace l'appel à `EffectCommand::Register` ; après avoir restauré l'appel d'origine, il se déclenche `CommandRegister` |
| `Internal.PlayerProbe.OnXxx(object, ...)` | AppelSite ×6 | événements de joueurs, une méthode par point d'accroche ; après avoir restauré l'appel d'origine, il publie l'événement |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | AppelSite ×3 | événements de bloc, accrochés aux sites d'affectation des trois propriétés de rappel de `ServerChunkCache` ; un délégué wrapper est superposé avant de rendre le contrôle au noyau |
| `Internal.BlockProbe.OnXxx(...)` | AppelSite ×3 | bloquer les événements ; rupture et chute du hook `ServerBlockUpdates`, les changements d'état accrochent la méthode d'interface sur `IBlockUpdateSink` |

La signature de `SignalProbe` ne prend que `string`, et les paramètres de `CommandProbe`, `PlayerProbe` et `LevelProbe` sont déclarés comme `object` — c'est délibéré : lors de l'assemblage, `Lead.Hook` résout la signature de la méthode de remplacement, et une fois qu'un type de noyau y apparaît, sa résolution entraîne l'assemblage du noyau plus tôt et l'injection manque sa fenêtre. Les types de noyau apparaissent uniquement dans les corps de méthode, moment auquel le code est déjà en cours d'exécution.

Les seules choses qui ne peuvent pas être `object` dans une signature sont les paramètres de type valeur et les valeurs de retour : `object` est une référence sur la pile tandis que `float`/`bool` sont des valeurs, et une incompatibilité est un IL invalide. Ainsi, `PlayerProbe.OnHurtPlayer` conserve `float` pour le montant des dégâts, et `OnRemovePlayer` et `OnHurtPlayer` conservent `bool` valeurs de retour.

`BlockProbe` est une extension de cette contrainte : la position et l'état du bloc sont les deux types de valeur `BlockPos`/`BlockState`, qui ne peuvent être écrits que tels quels dans la signature. Ces deux types proviennent de `NetCraft.Primitives` et `NetCraft.Registry`, dont aucun n'est sur la liste d'injection, donc les résoudre lors de l'assemblage ne récupère pas les assemblages à réécrire plus tôt.

### 5.2 Liste des points d'accroche

Le `ncmod.json` de ModApi contient vingt-quatre règles, correspondant au tableau de la version 2.1 un à un. Pour modifier un point d'accroche ou ajouter une règle, éditez ce fichier ; après l'édition, reconstruisez (c'est une ressource intégrée) et remettez la DLL résultante dans `mods/` — cette dernière est déjà effectuée automatiquement par `DeployModToHosts` dans `NetCraft.ModApi.csproj`, et son absence se manifeste par le fait que les règles ne prennent pas effet du tout.

`CommandManager::Execute` a deux surcharges qui partagent une règle. Le CallSite de `Lead.Hook` correspond aux sites d'appel par "type + nom de méthode", et non par liste de paramètres, et les deux surcharges prennent deux paramètres, de sorte que la sonde peut prendre `object` pour le premier paramètre et l'envoyer par le type réel.

Les deux règles `PacketProcessor` sont complémentaires : les paquets de phase de lecture passent par `ScheduleIfPossible` dans la file d'attente du thread principal, tandis que les phases de prise de contact et d'état passent par `HandleNow` pour un traitement immédiat ; un paquet donné ne traverse qu'un seul d'entre eux. L'accrochage uniquement du premier manque les phases de prise de contact et d'état - qui s'avèrent être les plus faciles à sonder avec des scripts, donc lors du débogage, cela est facilement interprété à tort comme "la règle n'a pas pris effet".

Les trois règles de bloc accrochent les **sites d'affectation** des trois propriétés de rappel de `ServerChunkCache`, pas les sites de lecture. La raison en est que ces trois propriétés sont monodiffusion et déjà occupées par le noyau lui-même lorsque `PersistentServerLevel` est construit (elles injectent une logique de sauvegarde et de nettoyage d'entité de bloc) ; un mod attribuant directement remplacerait la copie du noyau - les déchargements ne seraient pas conservés, les entités de bloc ne seraient pas nettoyées et sans aucune erreur. L'accrochage du site d'affectation permet d'enchaîner le rappel du noyau et la sonde à ce moment-là ; l'affectation n'a lieu qu'une seule fois et chaque déclencheur suivant ajoute une couche de transfert de délégué.

`PlayerList::RespawnPlayer` est privé, la sonde ne peut donc pas restaurer l'appel d'origine ; celui-ci passe par réflexion (appelée une fois par décès, donc les frais généraux sont négligeables). Cela laisse également une porte ouverte pour l'alignement du noyau : si `InternalsVisibleTo` est ajouté à l'avenir, il peut être remplacé par un appel direct.

### 5.3 Grand livre

`Internal.NcCommandRegistry` enregistre les commandes enregistrées via `args.Register`. Il ne s'agit que d'un grand livre et ne participe pas à l'exécution des commandes ; les commandes elles-mêmes sont installées sur le répartiteur du noyau, donc même si le grand livre rencontre des problèmes, les commandes fonctionnent toujours.

***

## 6. À ajouter

Voici les points d'accroche dont les emplacements sont confirmés mais qui ne sont pas encore devenus des événements (la liste a été produite par `__scan_mod_api.py` à la racine du référentiel) :

| Itinéraire | Points d'accroche candidats |
| ----------------- | ---------------------------------------------------------------------------- |
| Entités | `ClientLevel::AddEntity`, `Entity::Die` |
| Monde | chargement et déchargement de niveau, `ServerChunkCache` batch batch |
| Génération de terrain | `ChunkGenerator::Generate` étapes par `ChunkStatus`, `WorldGenRegion::SetBlockState` |
| Exécution des commandes | `CommandSourceStack::SendSuccess` / `SendFailure` (la moitié de réponse, avec de nombreux sites d'appel) |
| Réseau | par type de paquet `ServerGamePacketListenerImpl::HandleXxx` (actuellement uniquement un point d'entrée unifié) |

Instructions déjà effectuées : les graduations de niveau sont devenues `ServerEvents.LevelTick`, la persistance des données enregistrées est devenue `ServerEvents.SavedDataSaving`, l'exécution des commandes est devenue `ServerEvents.CommandExecuted` et les blocs sont devenus `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped`.

Accrocher la cellule de bloc sur `SetBlock` ne fonctionne pas : il a deux paramètres par défaut, `notifyNeighbors` et `strict`, donc les sites d'appel compilés prennent entre 4 et 6 paramètres, et puisque CallSite correspond par "type + nom de méthode" sans regarder la liste des paramètres, une méthode de remplacement ne peut pas gérer les trois formes de pile. Au lieu de cela, il s'accroche à deux endroits : les hooks de synchronisation d'état `IBlockUpdateSink::BlockChanged` (le seul appel d'interface dans `ServerLevel.SetBlock`, couvrant chaque changement avec la synchronisation client), et la rupture et l'abandon des propres méthodes du hook `ServerBlockUpdates`.

La cellule des entités est gênante car `Entity` est défini dans `NetCraft.Registry`, qui n'est pas sur la liste d'injection, donc les sites d'appels qui la ciblent ne peuvent pas être réécrits. `ClientLevel::AddEntity` est dans l'assembly client et est faisable ; un événement de décès nécessite d'abord de déterminer si `Registry` peut être réécrit.

Pour la cellule réseau, `HandleChat` existe depuis longtemps et `NetworkEvents.PacketReceived` fournit un point d'entrée unifié, donc les hooks per-`HandleXxx` sont beaucoup moins précieux ; seuls les scénarios nécessitant un filtrage fin par type de paquet méritent d'être ajoutés.

Le chargement et le déchargement de niveau n'ont pas de point de convergence du côté NC : `DedicatedServer::CreateLevel` est privé, donc l'appel d'origine ne peut être restauré que par réflexion comme `PlayerList::RespawnPlayer` ; le chemin de déchargement est encore plus dispersé. Pour ce faire, déterminez d’abord quels devraient être les arguments de l’événement.

Tous les autres ont besoin de références d'objet (instances d'entité, etc.), ils doivent donc utiliser `CallSite` plutôt que `Mark` ; si un type de valeur du noyau apparaît parmi les paramètres, il ne peut être écrit que dans la signature de la méthode de remplacement en tant que tel.

La liste des surfaces d'interface (`__modapi_api.txt`, produite par `__scan_mod_api.py --api`) a également été revue : les entrées qualifiées de points d'entrée de capacités ont été regroupées dans les façades du chapitre 4 par domaine, et le reste qui n'est pas exposé tombe dans trois catégories : protocole et gestion des paquets (`Network.Protocol.*`), rendu et modèles (`Client.Render.*`), et fonctions de génération et de densité de terrain (`LevelGen.*`). Ce sont des éléments internes du noyau ; les utiliser directement lierait les mods aux détails d'implémentation, donc une interface stable doit d'abord être ouverte dans le noyau.

Du côté du serveur, il y a deux autres choses qui ne sont pas enveloppées comme des façades : l'instance de `ReloadableServerResources` est suspendue à `DedicatedServer`, et comme ModApi ne fait pas référence à `NetCraft.Server`, l'envelopper nécessite d'abord d'ouvrir une propriété sur la classe de base du noyau ; `ChunkSender` et `ServerWorldBorderListener` sont des flux internes sans cas d'utilisation pour les mods.
