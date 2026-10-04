# Guide de modding NetCraft


Les détails des API individuelles ne sont pas ici ; voir [modapi-fr.md](modapi-fr.md).

---

## 1. La structure d'exécution avant tout

### 1.1 Trois points d'entrée de processus

NC a trois points d'entrée, et le pipeline d'injection de mods est le même pour les trois :

| Point d'entrée | Rôle |
| --- | --- |
| `NetCraft.Loader` | un seul exe pour les deux côtés : `--server` démarre le serveur ; `--client` ou aucun drapeau de mode démarre le client |
| `NetCraft.Server.Exe` | exécutable serveur autonome |
| `NetCraft.Client.Exe` | exécutable client autonome |

`Main` lui-même n'est qu'une fine enveloppe qui se contente d'enregistrer des callbacks et confie le travail à la méthode suivante. Prenons `NetCraft.Server.Exe` :

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // enregistrer le callback de résolution des assemblies du noyau
    BootMods(args);                        // exécuter le bootstrap des mods
    return Launch(args);                   // c'est seulement maintenant qu'on entre dans l'implémentation métier
}
```

Cette séparation n'est pas une question de style. Quand le JIT compile une méthode, il résout **tous les types** apparaissant dans ce corps de méthode, et cela se produit avant l'exécution de la méthode. Si `Main` appelait directement `ServerMain.Run(args)`, `NetCraft.Server.dll` serait remontée à l'instant où `Main` est compilée par le JIT, avant que le bootstrap des mods ne s'exécute, et la fenêtre de réécriture serait perdue. Donc `BootMods` et `Launch` doivent tous deux être marqués `MethodImplOptions.NoInlining` — sans ce marqueur, le JIT les inline à nouveau dans `Main`, ce qui réduit à néant la séparation.

`NetCraft.Loader` a la même structure, sauf que sa détection de mode et son bootstrap des mods sont tous deux dans `Launch`, et `Main` ne garde que les deux étapes `Initialize` et `Launch`.

### 1.2 Les assemblies du noyau dans le sous-répertoire kernel/

Le répertoire de sortie après une compilation ressemble à ceci :

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- bibliothèque principale, intègre toutes les sous-bibliothèques de niveau inférieur
NetCraft.ModLoader.dll    <- le loader lui-même
NetCraft.Server.Exe.dll   <- assembly d'entrée
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... assemblies du noyau restantes
mods/
  your-mod.dll
```

Pourquoi les déplacer dans `kernel/` au lieu de les laisser à la racine ?

L'hôte .NET traite les assemblies enregistrées dans `deps.json` comme des TPA (Trusted Platform Assemblies). Pour les assemblies du TPA, le runtime résout **par chemin** — les octets passés à `AssemblyLoadContext.LoadFromStream` sont simplement ignorés. Autrement dit, même si nous fournissons à l'avance des octets réécrits, le runtime lit quand même la copie non réécrite depuis le disque. Ce n'est qu'en retirant les assemblies du noyau de `deps.json` et en déplaçant les fichiers que le runtime rappelle `AssemblyLoadContext.Resolving` en cas d'échec de résolution, nous donnant la possibilité de fournir les octets réécrits.

Les trois sortes laissées à la racine ne peuvent pas être déplacées : la bibliothèque principale (c'est l'hôte d'embarquement et elle doit démarrer en premier), le loader lui-même (le code de bootstrap y réside) et l'assembly d'entrée (l'apphost démarre à partir d'elle).

**Coût** : l'assembly d'entrée elle-même ne peut pas être injectée. Si votre cible de hook se trouve dans l'assembly `NetCraft.Server.Exe.dll`, c'est sans effet. Le code métier du noyau est entièrement sous `kernel/`, donc normalement ce n'est pas un problème.

### 1.3 Séquence de chargement des mods

```
EmbeddedAssemblyLoader.Initialize()
  └─ installer le callback Resolving
BootMods → ModBootstrap.Run(côté courant)
  ├─ scanner statiquement mods/*.dll (MetadataReader lit le ncmod.json embarqué, aucune assembly n'est chargée)
  ├─ filtrer les mods dont l'environnement ne correspond pas au côté courant
  ├─ assembler les règles d'injection et confier le rewriter à la bibliothèque principale
  ├─ précharger les assemblies cibles : lire les octets → passer par le rewriter → LoadFromStream
  └─ appeler Init() de chaque entrée de mod
Launch → ServerMain/ClientMain.Run(args)
  └─ le métier du noyau commence à tourner ; les sondes sont déjà à l'intérieur
```

Notez l'ordre : **les déclarations sont scannées, les octets réécrits sont chargés, et le code d'entrée s'exécute en dernier**. Au moment où le `Init()` d'un mod s'exécute, les assemblies du noyau ont déjà été remplacées.

---

## 2. Différences clés avec Fabric

| Dimension | Fabric | NetCraft |
| --- | --- | --- |
| Langage / runtime | Java / JVM | C# / .NET 10 (CoreCLR) |
| Support du mod | jar contenant `fabric.mod.json` | dll embarquant `ncmod.json` |
| Lecture des déclarations | lire un fichier à l'intérieur du jar | `MetadataReader` lit statiquement les ressources embarquées sans charger d'assemblies |
| Injection de code | Mixin (annotations dans le source ; les membres sont mixés dans la classe cible au chargement de la classe) | `Lead.Hook` (règles déclarées dans un manifeste ou des annotations ; les octets sont réécrits sur place lors de la résolution des assemblies) |
| Granularité d'injection | n'importe quelle ligne d'un corps de méthode, y compris les locales et les valeurs intermédiaires d'expressions | treize formes (site d'appel, lecture/écriture de champ, constructeur, vérification de type, boxing, variable locale, constante, remplacement de tout le corps de méthode, sondes, etc.), avec insertion avant ou après |
| Modèle de chargement | Fabric Loader + class loader Knot | ALC par défaut unique + `AssemblyLoadContext.Resolving` |
| Périmètre de l'API officielle | l'API Fabric a un très grand nombre de modules | NetCraft-ModApi ne possède actuellement que des points d'extension pour les événements et les commandes |

### 2.1 Styles d'injection : annotations ou manifeste, choisissez-en un

Le Mixin de Fabric s'annote **dans le source** :

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NC prend en charge les deux styles, mais leurs prérequis diffèrent : la forme par annotation s'appuie sur `InjectAttribute` dans `NetCraft.ModApi.Extension`, donc un mod qui ne le référence pas ne peut pas utiliser les annotations ; la forme par manifeste est de la pure donnée écrite dans `ncmod.json` et n'exige aucune référence pour les règles d'injection.

**Annotation**, placée sur votre propre méthode de remplacement :

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**Manifeste**, écrit dans `hooks` de `ncmod.json` :

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

Pendant l'assemblage, les deux voies fusionnent en une seule table de règles, et **si le même point d'injection est écrit dans les deux, l'annotation gagne**. Après fusion, elles sont indiscernables ; la différence tient aux prérequis et à l'ergonomie :

| | Annotation | Manifeste |
| --- | --- | --- |
| Où elle est écrite | sur la méthode de remplacement | dans `hooks` de `ncmod.json` |
| Prérequis | doit référencer `NetCraft.ModApi` | aucun, pure donnée |
| Noms de types | `typeof` / `nameof`, vérifiés par le compilateur | chaînes écrites à la main ; les fautes ne sont trouvées que pendant l'assemblage |
| Ce qu'elle peut porter | les règles d'injection uniquement | l'identité du mod (id, entry, environment, informations d'affichage) et les règles d'injection |

Donc `ncmod.json` doit être écrit que vous utilisiez ou non les annotations ; c'est la seule source de l'identité du mod. Les annotations ne font que rendre les règles moins sujettes aux erreurs d'écriture. Le manifeste n'a pas de champ de dépendance ; les relations de dépendance sont déduites des références d'assemblies (voir 6.1) et n'ont pas besoin d'être déclarées.

Inversement, **un mod qui ne référence pas `NetCraft.ModApi` ne peut utiliser que le manifeste** — cela affecte plus que les règles d'injection : des événements comme `ServerEvents` et les façades `Nc*` sont aussi dans ModApi (sous `NetCraft.ModApi.Wrapper`, voir [3.3](#33-un-exemple-côte-à-côte)), donc un mod qui ne peut pas utiliser les annotations ne peut pas non plus les utiliser.

Les différences avec Fabric subsistent :

- **Comment et quand les changements ont lieu** : Mixin dispose d'un transformer qui **mixe les membres de la classe mixin dans** la classe cible au **chargement de la classe**, donc ce qui est chargé est une nouvelle classe synthétisée et l'originale n'existe plus ; NC réécrit sur place les instructions de la méthode cible **avant que l'assembly n'entre en mémoire**, donc la classe reste la même classe, seul son corps de méthode change. Les deux réécrivent au chargement, aucun ne modifie le bytecode à la compilation — le processeur d'annotations de Mixin ne génère qu'un refmap (mapping d'obfuscation) et effectue la validation à la compilation, tandis que NC n'est pas obfusqué et n'a pas du tout une telle couche.
- Mixin peut injecter à **n'importe quelle position au milieu d'un corps de méthode** ; NC peut cibler un site d'appel, un accès à un champ, une construction, une lecture/écriture de variable locale ou une constante précis dans une méthode hôte spécifiée, et peut insérer avant ou après (`InType`/`InMethod` restreignent la portée, `Placement` décide insertion ou remplacement), mais il **ne peut pas atteindre un numéro de ligne arbitraire** et ne peut pas changer une cible de saut ni une valeur intermédiaire d'expression sur la pile.
- Les cibles de Mixin utilisent un nom de méthode sous forme de chaîne plus un descripteur ; NC utilise « nom de type complet + nom de méthode », donc les surcharges de même nom correspondent toutes, et la précision sur une seule exige `InType`/`InMethod`.

**Quelle couche traite les annotations** : le type d'annotation (`InjectAttribute`) est fourni par `NetCraft.ModApi.Extension`, et il est résolu par `NetCraft.ModLoader` — lors du scan des mods, il lit statiquement la table `CustomAttribute` avec `MetadataReader`, sans charger d'assemblies. **`Lead.Hook` ne reconnaît pas les annotations** ; il ne voit que la table de règles fusionnée, et la couche d'injection native ne reconnaît que les octets de description compilés côté managé, sans même lire `ncmod.json`.

Cela détermine ce que les annotations peuvent exprimer : ce que vous pouvez écrire dépend entièrement des champs que possède `InjectAttribute`. Actuellement il y en a sept — type cible, nom de méthode, `HookType`, `Label`, `Environment`, `PatchMode`, `Ordinal` — et `InType`/`InMethod`/`Placement` de la section [2.4](#24-se-restreindre-à-un-seul-site--portée-de-lhôte-et-placement) et `LocalIndex`/`ConstantValue` de la section [2.5](#25-ancres-dans-le-corps-de-méthode--variables-locales-et-constantes) **ne peuvent pas être écrits dans les annotations** ; utilisez l'API C# ou attendez que le manifeste rattrape son retard. Le côté manifeste les omet aussi — la seule chose qu'il accepte au-delà des annotations est `ordinal`.

Pour les treize formes d'injection, voir l'[annexe du guide de modding](#annexe--aperçu-des-hooktype) et [modapi-fr.md](modapi-fr.md).

### 2.2 Une contrainte importante : les classes de sonde ne doivent pas porter de types du noyau dans leurs signatures

La réécriture NC a lieu **avant** que les assemblies du noyau soient chargées. Lors de l'assemblage des règles, `Lead.Hook` utilise la réflexion pour trouver votre méthode de remplacement et construire une référence de méthode, et ce processus résout chaque type de paramètre et type de retour de la signature.

Par conséquent : **la signature d'une méthode de remplacement ne peut utiliser que des types BCL et `object`**. Dès qu'un type `NetCraft.*` apparaît dans la signature, sa résolution remonte prématurément les assemblies du noyau et l'injection échoue complètement.

Quand vous avez besoin d'un objet du noyau, déclarez le paramètre en `object` et effectuez un cast à l'intérieur du corps de méthode :

```csharp
//l'assemblage ne voit que object ; le corps de méthode est compilé par le JIT après le démarrage du noyau
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite est un remplacement, pas une insertion

La méthode de remplacement d'une règle `CallSite` **remplace** l'appel d'origine, donc la méthode d'origine n'est pas exécutée. Pour préserver le comportement d'origine, vous devez le restaurer vous-même dans la méthode de remplacement :

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              // restaurer l'appel remplacé
    ServerEvents.CommandRegister.Publish(...);  // puis ajouter la logique propre au mod
}
```

Manquez cette étape et la fonctionnalité d'origine disparaît entièrement.

Quelques détails sur la mise en œuvre :

- **`this` sur une méthode d'instance compte aussi comme un paramètre**. Le fait que la méthode cible soit une méthode d'instance détermine si la méthode de remplacement a besoin d'un paramètre supplémentaire en tête. En IL, `call` et `callvirt` comptent tous deux comme des appels d'instance — pour une méthode non virtuelle sur un type `sealed`, le compilateur émet `call`.
- **Une clé de règle couvre toutes les surcharges sous ce nom de classe** ; les surcharges de même nombre de paramètres partagent une même méthode de remplacement. `Disconnect(string)` et `Disconnect(Component)` partagent ainsi, et la méthode de remplacement distribue selon le type réel de l'argument.
- **Une méthode privée ne peut pas être appelée depuis l'extérieur**, donc la méthode de remplacement ne peut pas restaurer l'appel d'origine. Soit abandonner ce point de hook, soit l'appeler une fois par réflexion (acceptable à basse fréquence).
- **Les paramètres et valeurs de retour de types valeur ne peuvent pas être déclarés en `object`**, `object` est une référence sur la pile tandis que `float`/`bool` sont des valeurs, et une incohérence donne un IL invalide. Conservez ces deux positions avec leurs types réels.
- **Les callbacks unicast ne peuvent pas être assignés directement**. Certaines propriétés de callback du noyau (p. ex. les trois callbacks de chunk sur `ServerChunkCache`) sont des `Action<T>` plutôt que des `event`, et le noyau les occupe déjà. Un mod qui assigne directement écrase la copie du noyau sans aucune erreur. L'approche correcte est de hooker le setter de la propriété et, au moment de l'assignation, d'enchaîner votre logique et le callback du noyau dans un seul délégué wrapper.

### 2.4 Se restreindre à un seul site : portée de l'hôte et placement

La portée par défaut d'une règle au niveau instruction est **l'assembly entière** — chaque endroit qui appelle la méthode cible ou lit/écrit le champ cible correspond. Pour se restreindre à un seul site, utilisez deux paramètres optionnels :

| Paramètre | Effet |
| --- | --- |
| `InType` / `InMethod` | ne faire correspondre les ancres qu'à l'intérieur du corps de la méthode hôte spécifiée ; les deux vides signifie sans restriction |
| `Placement` | `Replace` remplace l'ancre (par défaut) ; `Before` / `After` gardent l'ancre et insèrent un appel avant ou après |
| `Ordinal` | quand la même ancre correspond à plusieurs endroits dans la méthode hôte, choisir lequel, base 0. Omis signifie que tous les endroits sont modifiés |

```csharp
//exemple : n'instrumenter que lorsque LevelChunk lit l'état d'un bloc ; PalettedContainer::Get ailleurs n'est pas touché
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

Les deux modes imposent des exigences différentes sur la signature du callback :

- **Mode remplacement** s'aligne sur les arguments de l'appel remplacé (y compris `this` pour les appels d'instance) ; c'est à vous de décider si le callback restaure l'appel d'origine.
- **Mode insertion** passe les **paramètres de la méthode hôte** (y compris `this`), conformément à la convention `MethodBody`. L'insertion ne perturbe pas la pile déjà construite par l'ancre ; l'appel d'origine s'exécute comme d'habitude, juste avec un callback supplémentaire avant ou après.

Quelques limites :

- `InType` et `InMethod` sont indépendants ; vous pouvez n'en spécifier qu'un. Les deux vides équivaut à aucune portée.
- Plusieurs règles peuvent hooker la même ancre, chacune limitée à un hôte différent ; **la première correspondance d'hôte l'emporte**.
- `Placement` ne s'applique qu'aux formes au niveau instruction (`CallSite`, `NewObj`, lecture/écriture de champ, `TypeCheck`, `Box`, `FunctionPointer`, et les trois sortes de la section [2.5](#25-ancres-dans-le-corps-de-méthode--variables-locales-et-constantes)) ; `MethodBody` remplace toujours l'ensemble.
- `Ordinal` compte l'**ordre des correspondances**, indépendamment du fait que ce site soit finalement modifié ; si la règle n'apparaît pas assez de fois dans la méthode hôte, la règle n'est pas appliquée. Même idée que `@At(ordinal)` de Mixin.
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` ne sont actuellement disponibles que sur l'API C# ; ni `ncmod.json` ni `[Inject]` ne les prennent en charge (le manifeste accepte `ordinal`), donc les mods basés sur le manifeste ne peuvent pas utiliser les premiers.

### 2.5 Ancres dans le corps de méthode : variables locales et constantes

Les sortes précédentes s'ancrent à une **entité référencée** (une méthode, un champ ou un constructeur), tandis que `LocalRead` / `LocalWrite` / `Constant` s'ancrent à **une position à l'intérieur du corps de la méthode hôte**, correspondant à `@ModifyVariable` et `@ModifyConstant` de Mixin. Pour ces trois, `OriginalType` / `OriginalMethod` désignent la **méthode hôte**, pas une entité référencée.

| Forme | Paramètre supplémentaire | Positions sélectionnées |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` | chaque lecture de cet emplacement (base 0) |
| `LocalWrite` | `LocalIndex` | chaque écriture dans cet emplacement |
| `Constant` | `ConstantValue` | chaque chargement de cette constante, comparé par type boxé |

```csharp
//exemple : insérer un callback avant l'écriture dans l'emplacement 0 de G
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//exemple : remplacer la constante 5 de G par la valeur de retour de OnConst()
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

En mode remplacement, la signature du callback s'aligne sur l'**effet sur la pile de l'instruction**, pas sur les paramètres de l'hôte :

| Ancre | Effet sur la pile | Signature de la méthode de remplacement |
| --- | --- | --- |
| `LocalRead` | pousse une valeur | zéro paramètre, renvoie cette valeur |
| `LocalWrite` | dépile une valeur | un paramètre |
| `Constant` | pousse une valeur | zéro paramètre, renvoie cette valeur |

`ConstantValue` est comparé par type boxé, donc `5` (int) et `5L` (long) sont deux ancres différentes ; pour correspondre à `ldc.i8`, vous devez passer un `long`.

Un emplacement est l'index compilé de la variable locale ; le même source peut le changer sous une version différente du compilateur, donc ne le traitez pas comme un identifiant stable lors d'un portage entre versions.

### 2.6 Injection à l'exécution : modifier du code déjà en cours d'exécution

L'injection dont on a parlé jusqu'ici a toujours lieu **avant le chargement de l'assembly** — les octets sont d'abord réécrits, puis remis au runtime. Le postulat est que l'assembly cible n'est pas encore chargée.

`Lead.Hook` a une autre voie : utiliser l'interface Profiler du CLR (ReJIT) pour modifier du code **déjà chargé, ou dont les méthodes ont déjà tourné**. Les deux partagent le même `HookRule`, et les paramètres de la section [2.4](#24-se-restreindre-à-un-seul-site--portée-de-lhôte-et-placement) et de la section [2.5](#25-ancres-dans-le-corps-de-méthode--variables-locales-et-constantes) restent disponibles :

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//un seul appel et l'injection est faite ; pas de redémarrage et pas de modification de fichier
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| | Réécriture au chargement | Injection à l'exécution |
| --- | --- | --- |
| `patchMode` du manifeste | `ILRewrite` (par défaut) | `RuntimeInject` |
| Moment | avant que l'assembly n'entre en mémoire | à tout moment après le démarrage du processus |
| Fondement | réécriture d'octets Mono.Cecil | CLR Profiler ReJIT |
| Prérequis | la cible n'a pas été chargée | la cible est déjà dans le processus |
| Modifier du code déjà compilé par le JIT | impossible | possible |

**Pourquoi les règles sont partagées** : Cecil fait toujours la réécriture ici, mais le résultat n'est pas écrit sur disque ; il est plutôt compilé en une description remise à la couche native, qui soumet le nouveau corps de méthode au CLR à l'exécution, la gestion des versions restante étant laissée au CLR.

**Comment un mod procède** : ajoutez une entrée `patchMode` à l'élément de règle dans `ncmod.json` ; l'annotation `[Inject]` a un paramètre du même nom.

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**Ce que fait l'assemblage** : ces règles ne passent pas par le chemin de réécriture au chargement (`ModHooks.Rewrite` n'applique que `ILRewrite`) ; au moment de l'assemblage, une table de cibles d'exécution distincte est enregistrée. Après le préchargement des assemblies du noyau et avant le `Init()` des mods, le loader prend chacune de leurs instances **déjà chargées** et soumet les corps de méthode réécrits au CLR. Si une cible n'est pas chargée à ce moment, elle est ignorée avec un avertissement, et elle ne sera pas chargée à l'avance pour elle — la contrainte de la section [2.2](#22-une-contrainte-importante--les-classes-de-sonde-ne-doivent-pas-porter-de-types-du-noyau-dans-leurs-signatures) selon laquelle « remonter le noyau prématurément fait manquer la fenêtre » est ici inversée en direction, avec la même conclusion : si elle n'est pas présente, on ne peut pas le faire.

**Pour hooker la couche d'injection native** : l'interrupteur ReJIT ne peut être activé que via des variables d'environnement au démarrage du processus (`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`) ; les définir après le démarrage n'a aucun effet. Quand le loader détecte tôt, au démarrage, des règles `RuntimeInject` et que le processus n'est pas encore hooké, il **redémarre le processus avec la même ligne de commande** en portant ces trois variables (`NC_PROFILER_ATTACHED=1` protège contre les redémarrages répétés quand « hooké mais sans effet »). La bibliothèque native `lead_hook_native` doit être placée à la racine du programme, ou pointée ailleurs via `NC_PROFILER_PATH` ; si ni l'une ni l'autre n'est présente, tout le lot de règles est dégradé en un simple avertissement et le démarrage n'est pas bloqué.

**Ne le mélangez pas avec la réécriture au chargement sur la même cible** : un corps de méthode soumis par injection à l'exécution est dérivé des **octets d'origine** et n'inclut pas les changements apportés par la réécriture au chargement à la même méthode — si la même méthode est touchée par les deux types de règles, la version issue de la réécriture au chargement est entièrement écrasée. L'assemblage ne peut pas dire si deux règles touchent la même méthode, donc il ne peut qu'en juger grossièrement par assembly cible et enregistrer un avertissement.

**Les annotations ne participent pas directement à ce chemin** : la couche d'injection native ne reconnaît ni `InjectAttribute` ni `ncmod.json` — elle ne reconnaît que les octets de description. Les annotations et le manifeste sont tous deux des choses **propres au moment de l'assemblage** (voir [2.1](#21-styles-dinjection--annotations-ou-manifeste-choisissez-en-un)), analysés par `NetCraft.ModLoader` en `HookRule` puis remis à `RuntimeInjector` ; le côté mod n'a pas besoin de les traiter différemment.

**Limitations** (plus étroites que la réécriture au chargement) : les méthodes avec des tables de gestion d'exceptions ne sont pas prises en charge, la table des variables locales ne peut pas être modifiée, les types et méthodes génériques ne sont pas pris en charge, et les opérandes ne reconnaissent que les références de méthode (les références de champ, les constantes de chaîne et les tokens de type lèvent `NotSupportedException`).

**Performance** : l'injection n'a lieu qu'à l'enregistrement ; ensuite la méthode est du code JIT ordinaire avec le même surcoût d'appel qu'avec l'injection. L'attachement du profiler a un coût ponctuel — activer ReJIT exige de désactiver en même temps les images ReadyToRun, et le démarrage du processus mesuré est environ 80–110 ms plus lent ; le calcul en régime stable ne montre aucune différence. Pour un serveur comme NC dont le démarrage se mesure déjà en secondes, c'est négligeable.

### 2.7 Deux mods qui modifient la même classe entrent-ils en conflit comme avec Mixin ?

D'abord, pourquoi les choses entrent en conflit du côté Mixin. Mixin **mixe des membres dans la classe cible** et les applique au chargement de la classe : quand plusieurs mixins se mixent dans la même classe, des cas comme injecter plusieurs fois au même endroit ou ajouter des membres de même nom à la même classe lèvent `MixinApplyError`, et le fail-hard par défaut **tue le jeu carrément** ; de plus, cette détection se produit à l'instant du chargement de la classe, quand le jeu peut déjà être à moitié en cours d'exécution.

Le modèle de NC est différent, et la surface où des conflits peuvent se produire est bien plus petite :

| | Mixin | NetCraft |
| --- | --- | --- |
| Méthode d'application | mixer des membres dans la classe cible + réécrire le bytecode | réécrire uniquement les instructions IL ; aucune synthèse de type, aucun membre ajouté |
| Conflits structurels (membres de même nom, conflits d'héritage) | oui | aucun |
| Moment de validation des règles | au chargement de la classe | au moment de l'assemblage, en lisant statiquement les métadonnées |
| Deux règles touchant le même endroit | lève une exception | premier arrivé, premier servi ; la seconde échoue silencieusement |
| Un mod échoue | peut entraîner tout le chargement | n'affecte que lui-même |

**Validation statique** : les règles ne sont pas construites en chargeant des assemblies et en réfléchissant sur les types, mais en lisant les tables de métadonnées PE. Ainsi, des problèmes comme « le type cible n'est dans aucune assembly connue » ou « la forme d'injection est mal orthographiée » sont enregistrés et ignorés **tôt au démarrage**, sans attendre qu'une classe se charge avant d'exploser.

**Isolation des échecs** : quand les règles d'un mod ne parviennent pas à être analysées, que la classe de remplacement ne parvient pas à se charger, ou que le `Init()` d'entrée lève une exception, **seul ce mod** est marqué comme en échec (statut `Error`, affiché comme « échec de chargement » sur la page MODS) et les autres mods se chargent normalement. Une formulation mérite d'être corrigée ici : NC n'a **aucun déchargement à l'exécution** — les mods sont chargés une fois, et `ModManager` n'offre explicitement pas de chargement/déchargement dynamique. Le soi-disant « déchargement automatique en cas d'échec » est en réalité une **isolation au chargement** : un mod en échec n'est pas initialisé, mais il n'est pas non plus « déchargé ».

**Injection dynamique** : sur la voie ReJIT de la section [2.6](#26-injection-à-lexécution--modifier-du-code-déjà-en-cours-dexécution), la sémantique de plusieurs mods se disputant la même méthode est la même que dans le cas statique — le premier enregistré gagne, et les requêtes ultérieures sont envoyées mais ne peuvent pas la revendiquer (`GetReJITParameters` revendique par « module + méthode », en prenant le premier).

Nous avons rencontré un piège sur cette voie une fois, qui mérite d'être noté : dans une implémentation précoce, le **paramètre de portée de résolution de `FindTypeRef` était passé à `mdTokenNil`**, dont la sémantique est « ne faire correspondre que les TypeRefs sans portée de résolution » — nos références pendent toutes à `AssemblyRef`, donc celle que nous avions construite n'était jamais trouvée. Cela se manifestait ainsi : la première injection réussissait, mais à la seconde injection la résolution des références échouait, aucun nouveau corps de méthode ne pouvait être construit, le CLR revenait à l'IL d'origine, et **la première injection était perdue avec lui** (la méthode cible revenait à son comportement non injecté). Après correction, le problème ne se reproduisait plus, mais la limitation demeure : **l'injection de références dans les métadonnées doit s'achever dans la fenêtre qui suit immédiatement le chargement du module cible ; plus tard c'est, plus l'échec est probable**.

**Le coût doit être dit clairement** : le comportement sans plantage de NC a pour prix des conflits facilement manqués — Mixin au moins interrompt le chargement, tandis que NC laisse la seconde échouer silencieusement. Pour y remédier, l'assemblage effectue une **vérification des conflits de même ancre** : quand le même point d'injection est déclaré par plusieurs mods, le dernier assemblé est consigné dans `ModHooks.Warnings` et signalé comme avertissement dans le journal de démarrage (sans bloquer le chargement) :

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

La clé de détection est « type cible + méthode + forme d'injection + mode de patch ». **La portée d'hôte n'est pas distinguée** — ni le manifeste ni les annotations ne peuvent écrire `InType`/`InMethod`, donc les règles venant des mods ont naturellement une portée au niveau de l'assembly entière, et la même clé signifie une collision. Les règles ajoutées directement via l'API C# contournent cette vérification, puisque dans ce cas deux règles peuvent chacune toucher un hôte différent et sont par nature non conflictuelles.

**L'injection à l'exécution passe aussi par cette vérification** : elle entre par le même point d'entrée d'assemblage, et le mode de patch dans la clé de détection la garde séparée de la réécriture au chargement ; quand les deux types atterrissent sur la même assembly cible, il y a un avis d'écrasement distinct (voir [6.4](#64-deux-mods-injectant-la-même-cible)).

### 2.8 Mixins : ajouter des membres à un type cible

Les sections précédentes modifient toutes des instructions dans du code existant et ne peuvent rien créer de nouveau. Pour **ajouter des champs, des méthodes ou des interfaces** à un type cible, utilisez un mixin.

La relation avec la syntaxe de Mixin est la suivante :

| Mixin | NC |
| --- | --- |
| `@Mixin(X.class)` sur la classe mixin | `[Mixin(typeof(X))]` sur la classe source |
| les membres de la classe mixin sont mixés dans la classe cible | les champs et méthodes de la classe source sont déplacés dans le type cible |
| `@Unique` ajoute un champ privé | écrivez un champ ordinaire dans la classe source ; il est déplacé de la même façon |
| `@Shadow` référence un membre existant de la classe cible | pas nécessaire ; écrivez directement les membres de `X` puis hookez-les |
| `@Implements` / `implements` | `Interfaces` |

Il peut aussi être écrit dans le manifeste :

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
    //après déplacement, c'est un champ d'instance sur le type cible
    public int MyCounter = 5;

    //une méthode mixée ; elle lit/écrit le champ déplacé avec elle
    public int Bump() => MyCounter + 1;

    //l'implémentation requise par l'interface ; après déplacement, le type cible implémente ITagged
    public string Describe() => $"tagged:{MyCounter}";
}
```

**Déplacement, pas copie**. Ces membres sont retirés de la classe source, ne laissant qu'une coquille vide — tout comme la classe mixin de Mixin, **le code du mod ne doit plus utiliser cette classe** (`new SomeEntityMixin()` ou l'appel de ses méthodes ne trouveront pas les membres).

Quelques règles d'application :

- **Ne s'applique qu'à la réécriture au chargement**. Le type cible doit être dans une assembly du noyau ou une assembly de mod qui n'est pas encore entrée en mémoire. L'injection à l'exécution ne soumet que des corps de méthode et ne peut pas changer la disposition du type, donc cette forme ne peut pas exister dessus.
- **Les initialiseurs de constructeur suivent**. Les valeurs écrites dans les initialiseurs de champs de la classe source sont fusionnées dans chaque constructeur d'instance du type cible ; les initialiseurs de champs statiques sont fusionnés dans le constructeur statique (créé si la cible n'en a pas). L'appel d'enchaînement à la classe de base à l'intérieur du constructeur de la classe source est supprimé, donc le constructeur de base n'est pas exécuté deux fois.
- **Les règles avec interfaces marquent comme virtuelles les méthodes d'instance publiques déplacées**. La distribution d'interface ne reconnaît que la vtable, et sans le marqueur le CLR conclurait que l'interface n'est pas implémentée et échouerait complètement au chargement. Donc ne vous attendez pas à ce que ces méthodes restent non virtuelles lors d'un mixage d'interface.
- **Les membres de même nom sont ignorés**. Quand le type cible a déjà un champ ou une méthode du même nom, ce seul élément n'est pas déplacé et le reste se poursuit normalement. Quand deux mods se mixent dans le même type cible, les deux s'appliquent, et seule la partie de même nom du second est ignorée — plus doux que le comportement « le second échoue entièrement » des hooks de la section [2.7](#27-deux-mods-qui-modifient-la-même-classe-entrent-ils-en-conflit-comme-avec-mixin).
- **Les types imbriqués ne sont pas déplacés**, et les types imbriqués ou méthodes génériques dans la classe source sont également hors de la couverture de ce chemin actuellement.

Le type source doit être dans **l'assembly du mod lui-même**, donc ni le manifeste ni l'annotation n'écrivent de nom d'assembly.

### 2.9 Deux voies : couche wrapper et points d'extension

La surface publique de `NetCraft.ModApi` est répartie en deux espaces de noms, correspondant à deux usages :

| Espace de noms | Contenu | Ce que vous obtenez |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`, `NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup`, et des handles d'objets comme `NcPlayer` / `NcLevel` | types wrapper ; aucun type du noyau dans la surface publique |
| `NetCraft.ModApi.Extension` | Annotations `[Inject]` / `[Mixin]` | les règles se lient aux noms de classes et de méthodes du noyau |

Les deux sont des voies **parallèles**, pas l'une superposée à l'autre :

- **Pour la stabilité, utilisez `Wrapper`**. Les façades gèrent pour vous l'ordre d'appel fastidieux du noyau (écrire un seul bloc tout en touchant le niveau, la liste des joueurs et la chaîne de synchronisation en est un exemple), et les arguments d'événements sont tous des types wrapper. Le coût est que les capacités que les façades n'exposent pas vous sont inaccessibles.
- **Pour l'exhaustivité, utilisez `Extension`**. Les règles d'injection modifient directement les classes et méthodes du noyau, mais les noms de cibles que vous écrivez sont ceux du noyau, donc quand le noyau change, les règles doivent changer avec lui.

Vous pouvez référencer les deux. La ligne `Wrapper` est en cours de consolidation vers « aucun type du noyau dans la surface publique » ; les parties joueur et niveau sont faites — `Player` / `Attacker` dans les événements joueur et les paramètres d'entrée/sortie de `NcPlayers` sont des handles `NcPlayer`, `NcWorld.Overworld` / `Nether` / `End` / `Get` et `LevelTickArgs.Level` sont des handles `NcLevel`, et les coordonnées de blocs sont de simples entiers `x y z` ; les entités et les types valeur restants (`BlockPos` / `BlockState` / `Vec3`) ne sont pas encore emballés.

Une limite de plus à signaler : **la couche wrapper ne protège pas de l'injection**. Les règles `hooks` ou les annotations `[Inject]` que vous écrivez se lient toujours aux noms de classes et de méthodes du noyau, et cassent tout autant quand le noyau change.

---

## 3. Migrer depuis Fabric

### 3.1 Correspondance des concepts

| Fabric | NetCraft |
| --- | --- |
| `fabric.mod.json` | le `ncmod.json` embarqué dans la dll |
| `ModInitializer.onInitialize()` | la `public Task Init()` de la classe d'entrée |
| `@Inject` / `@Redirect` | l'annotation `[Inject]`, ou des règles comme `Mark` / `Probe` / `CallSite` dans `hooks` |
| `@ModifyVariable` | `LocalRead` / `LocalWrite`, voir la section [2.5](#25-ancres-dans-le-corps-de-méthode--variables-locales-et-constantes) ; non inscriptible en annotation |
| `@ModifyConstant` | `Constant`, voir la section [2.5](#25-ancres-dans-le-corps-de-méthode--variables-locales-et-constantes) ; non inscriptible en annotation |
| `@Accessor` | pas encore d'équivalent (les membres `private` n'ont pas besoin d'élargissement de visibilité ; écrivez simplement une règle) |
| `Registry.register(...)` | le registre du noyau (`BuiltInRegistries`) |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` | pas encore d'équivalent (`ModManager` n'est pas ouvert aux mods) |
| `@Mixin` / `@Unique` / `@Implements` | l'annotation `[Mixin]` ou les `mixins` du manifeste, voir la section [2.8](#28-mixins--ajouter-des-membres-à-un-type-cible) |

### 3.2 Ce qui ne se transpose pas

- **Le système d'annotations de Mixin** : NC a deux annotations, `[Inject]` et `[Mixin]`, qui ne sont que des **styles de déclaration**, équivalents à `hooks` / `mixins` dans `ncmod.json` et fusionnées à l'assemblage (elles sont résolues par le loader, pas par `Lead.Hook`, voir [2.1](#21-styles-dinjection--annotations-ou-manifeste-choisissez-en-un)). La réécriture d'instructions s'applique au chargement par défaut, et peut être changée en soumission à l'exécution selon la section [2.6](#26-injection-à-lexécution--modifier-du-code-déjà-en-cours-dexécution) ; l'ajout de membres et d'interfaces passe par les mixins de la section [2.8](#28-mixins--ajouter-des-membres-à-un-type-cible). Le ciblage n'a pas de syntaxe de chaîne de type `@At`, mais des formes comme `CallSite`/`FieldRead`/`LocalWrite`/`Constant`, avec `Ordinal`, `InType`/`InMethod` et `Placement`, peuvent couvrir les usages de `HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE` et `shift` ; ce qui manque est `JUMP`.
- **AccessWidener** : aucun. La visibilité n'est pas un obstacle à la réécriture IL dans NC ; les méthodes `private` peuvent être hookées de la même façon (le rewriter travaille au niveau des octets).
- **Mappings Yarn / Mojang** : pas nécessaires. NC est du source C# traduit directement depuis vanilla, avec des noms de types et de méthodes correspondant à vanilla, seul le style de nommage suivant C#.
- **La grande majorité des modules de l'API Fabric** : seules les capacités couvertes par `NetCraft-ModApi` sont disponibles ; pour le reste, écrivez vos propres règles d'injection ou attendez que l'API rattrape son retard.

### 3.3 Un exemple côte à côte

Fabric : journaliser une ligne au démarrage du serveur et enregistrer une commande.

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

NC :

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

Manifeste (`ncmod.json`, comme ressource embarquée) :

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

Remarque : même un `hooks` vide fonctionne ici — des événements comme `ServerEvents.Started` sont fournis par les sondes de `NetCraft-ModApi` lui-même, et votre mod n'a qu'à s'abonner (`ServerEvents` est sous `NetCraft.ModApi.Wrapper`, voir [2.9](#29-deux-voies--couche-wrapper-et-points-dextension)). Vous n'avez besoin d'écrire vos propres règles de hook que lorsque vous voulez hooker un endroit du noyau où ModApi ne fournit pas encore d'événement.

---

## 4. Exigences de base pour un ncm

ncm signifie mod NetCraft. Un ncm est une dll de bibliothèque de classes .NET embarquant un `ncmod.json`, placée dans le répertoire `mods/`.

La façon la plus simple de commencer est le modèle :

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

Si le paquet local n'a pas encore été publié, utilisez `dotnet new install <chemin du nupkg>`, ou exécutez `dotnet pack` dans le dépôt puis installez la sortie.

`-e` prend `both` (par défaut) / `server` / `client`, déterminant l'`environment` du manifeste et quel code d'abonnement côté est généré dans la classe d'entrée. Le modèle fournit les assemblies de référence NC, donc aucune référence de projet n'est nécessaire, et les id / entry de `ncmod.json` sont renseignés à partir du nom du projet.

À partir de la section 4.1, ce qui suit couvre ce qu'un projet écrit à la main doit respecter.

### 4.1 Fichier de projet

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private doit être désactivé, sinon les assemblies NC sont embarquées dans la dll du mod par EmbedDependencies en 4.6 -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName` doit être `ncmod.json` ; le scanner ne reconnaît que ce nom.

Le modèle prend une voie différente : des références d'assemblies sous `libs/` (`Reference Include="libs\*.dll" Private="false"`). Suivez-la lorsque vous n'avez pas le source de NC. Ce que les deux approches ont en commun est que **les dll propres à NC ne doivent jamais entrer dans le répertoire de sortie** — `EmbedDependencies` de la section [4.6](#46-dépendances-tierces) embarque les dll tierces du répertoire de sortie dans le mod, et si les assemblies de NC sont embarquées elles aussi, il y aura deux ensembles d'identité de type.

### 4.2 Champs de ncmod.json

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

| Champ | Requis | Remarques |
| --- | --- | --- |
| `id` | Oui | identifiant du mod, utilisé par les dépendances et les recherches. Une chaîne vide est ignorée par le scanner |
| `version` | Recommandé | numéro de version ; quand d'autres mods en dépendent, les contraintes de version sont jugées par lui, voir ci-dessous |
| `name` | Non | nom d'affichage ; c'est ce que montre la page du mod, par défaut `id` |
| `description` | Non | description sur une ligne |
| `authors` / `contributors` | Non | crédits, un tableau de chaînes |
| `license` | Non | identifiant de licence |
| `contact` | Non | liens externes ; peut prendre `homepage` / `sources` / `issues` |
| `icon` | Non | le nom de la ressource embarquée de l'icône, voir 4.7 |
| `environment` | Non | `both` / `client` / `server`, par défaut `both`. Quand il ne correspond pas au côté courant, tout le mod n'est pas chargé |
| `entry` | Oui | nom complet de la classe d'entrée ; la classe doit avoir `public Task Init()` |
| `depends` | Non | autres mods dont il dépend et versions requises, voir ci-dessous |
| `hooks` | Non | liste de règles d'injection ; un tableau vide signifie ne s'abonner qu'aux événements que ModApi possède déjà |
| `mixins` | Non | liste de règles mixin ; déplace les membres d'une des classes de ce mod dans le type cible, voir la section [2.8](#28-mixins--ajouter-des-membres-à-un-type-cible) |

`depends` déclare les autres mods dont il dépend et les versions requises ; la clé est un id de mod et la valeur une contrainte de version :

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**Les dépendances non écrites dans `depends` ne portent aucune contrainte de version**. Les dépendances entre mods sont déjà déduites automatiquement des références à la compilation (voir la section [6.1](#61-dépendance-entre-mods--injection-du-mod-dont-on-dépend)), et `depends` ne fait qu'ajouter des contraintes de version par-dessus. Quand le mod dont on dépend n'est pas du côté courant, la vérification est ignorée — ce cas est laissé à la résolution d'assemblies pour le signaler.

| Syntaxe de contrainte | Signification |
| --- | --- |
| `*` | n'importe quelle version, équivalent à omettre l'entrée |
| `1.2.3` | préfixe de segment ; `1.2` correspond à `1.2` et `1.2.9` mais pas à `1.3` |
| `^1.2.3` | même version majeure et pas inférieure à la base ; quand la version majeure est 0, la version mineure est utilisée à la place, donc `0.1` et `0.2` sont considérées incompatibles |
| `>=1.2.3` | pas inférieure à la base |

Les numéros de version ne prennent que les chiffres de tête de chaque segment, donc `26.2-netcraft` participe en tant que `26.2`. Quand les versions ne correspondent pas, **seul le mod déclarant est ignoré** et le reste se charge normalement ; le journal de démarrage indique « la dépendance X requiert la version …, version réelle … ».

La base du jugement est le champ `version` du manifeste du mod dont on dépend. Donc **un mod destiné à être une dépendance doit définir correctement `version`** — un numéro de version vide ne satisfait aucune contrainte spécifique.

### 4.3 Champs d'une règle de hook

| Champ | Remarques |
| --- | --- |
| `target` | nom complet du type cible ; doit se trouver dans une assembly du noyau ou l'une des assemblies de mods sous `mods/` |
| `method` | nom de la méthode cible ; les surcharges de même nom correspondent toutes |
| `type` | forme d'injection, voir l'annexe |
| `patchMode` | méthode d'application, `ILRewrite` (par défaut) ou `RuntimeInject`, voir la section [2.6](#26-injection-à-lexécution--modifier-du-code-déjà-en-cours-dexécution) |
| `ordinal` | quand la même ancre correspond à plusieurs endroits dans la méthode hôte, choisir lequel, base 0, voir la section [2.4](#24-se-restreindre-à-un-seul-site--portée-de-lhôte-et-placement) |
| `replaceType` | nom complet de la classe contenant la méthode de remplacement |
| `replaceMethod` | nom de la méthode de remplacement |
| `label` | label de la sonde, utilisé uniquement par `Mark` et `Probe` |
| `environment` | le côté auquel la règle s'applique, par défaut `both` ; une règle ciblant un type serveur exécutée sur le client n'a pas de cible du tout et est filtrée par `environment` |

La même règle peut aussi être écrite comme annotation sur la méthode de remplacement ; la correspondance est :

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

| Champ du manifeste | Forme en annotation |
| --- | --- |
| `target` | le premier paramètre du constructeur, écrit comme `typeof(...)` |
| `method` | le second paramètre du constructeur, de préférence `nameof(...)` |
| `type` | paramètre nommé `HookType`, par défaut `CallSite` |
| `patchMode` | paramètre nommé `PatchMode`, par défaut `ILRewrite` |
| `ordinal` | paramètre nommé `Ordinal` |
| `label` | paramètre nommé `Label` |
| `environment` | paramètre nommé `Environment`, par défaut `both` |
| `replaceType` | non écrit ; pris automatiquement depuis la classe qu'il annote |
| `replaceMethod` | non écrit ; pris automatiquement depuis la méthode qu'il annote |

Les annotations viennent de `InjectAttribute` dans `NetCraft.ModApi.Extension`, donc les mods qui utilisent les annotations doivent le référencer. Quand annotations et manifeste coexistent, ils sont fusionnés, et si le même point d'injection est déclaré dans les deux, **l'annotation gagne**.

Les règles mixin utilisent un ensemble de champs différent : `target` (nom complet du type cible), `source` (nom complet du type source, qui doit être dans l'assembly du mod lui-même), `interfaces` (optionnel, un tableau de noms complets d'interfaces) ; pour la sémantique, voir la section [2.8](#28-mixins--ajouter-des-membres-à-un-type-cible). `[Mixin(typeof(target))]` sur la classe source équivaut à l'entrée du manifeste.

### 4.4 Ce qui peut et ne peut pas être hooké

Peut être hooké : les assemblies du noyau sous `kernel/`, et les autres mods sous `mods/`.

Ne peut pas être hooké :

- la bibliothèque principale `NetCraft.dll`
- le loader `NetCraft.ModLoader.dll`
- les assemblies d'entrée (`NetCraft.Server.Exe.dll`, etc.)

Le `target` de chaque hook doit être trouvable dans le noyau ou dans une assembly de mod, sinon l'assemblage signale « la cible d'injection n'est dans aucune assembly connue ». Notez qu'**un espace de noms n'implique pas une assembly** — `NetCraft.Game.Server.DedicatedServer` vit en réalité dans `NetCraft.Server.dll` ; le loader le recherche via un index construit à partir des tables de métadonnées, donc écrivez simplement le nom complet.

L'injection de mods par des mods suit le même système : le mod cible est réécrit **à l'instant où il est lui-même chargé**, quel que soit l'ordre dans `mods/`. Les règles cycliques (A injecte B et B injecte A) signalent « chargement cyclique » au moment du chargement ; de telles règles n'ont pas de solution au niveau de la réécriture IL, donc retirez simplement l'une d'elles.

Notez que le mod injecté **ne peut pas être votre propre mod** — l'auto-injection compte aussi comme un cycle.

### 4.5 Déploiement

Copiez la dll compilée dans le `mods/` du répertoire de sortie (au premier niveau, sans récursion dans les sous-répertoires) et redémarrez le processus.

**Cette étape est déjà automatisée par la compilation** : `DeployModToHosts` dans `NetCraft.ModApi.csproj` copie la dll dans le répertoire `mods/` du répertoire de sortie de chaque projet hôte après la compilation. La liste des hôtes est la propriété `ModHostProjects` ; lorsque vous créez votre propre projet hôte, ajoutez simplement son nom. Si aucune règle ne prend effet et qu'il n'y a aucune erreur, vérifiez d'abord si la dll dans le `mods/` du projet hôte est obsolète.

Le journal de démarrage affiche :

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

Ordre de dépannage :

- Aucune règle ne prend effet : vérifiez d'abord si la dll dans `mods/` est obsolète (le plus courant).
- Si vous ne voyez pas `Rewrote and preloaded assembly`, l'assembly cible était déjà chargée avant le bootstrap, donc la règle a été écrite trop tard.
- Si vous voyez `injection targets` mais que la cible est erronée, c'est généralement un `target` mal saisi ; suivez la section [4.4](#44-ce-qui-peut-et-ne-peut-pas-être-hooké) pour vérifier à quelle assembly appartient le type.
- Une règle avec `environment` défini à `client` est silencieusement ignorée sur le serveur (une incompatibilité au niveau du mod signifie que tout le mod n'est pas chargé) ; c'est un comportement attendu, pas un défaut.

### 4.6 Dépendances tierces

Correspond au Jar-in-Jar de Fabric.

Les projets créés à partir du modèle **n'ont pas à s'en soucier** : les bibliothèques ajoutées via `dotnet add package` sont automatiquement embarquées dans la dll du mod à la compilation, et quand le loader ne peut pas résoudre une assembly, il parcourt les ressources `.dll` embarquées du mod.

```
dotnet add package Newtonsoft.Json
```

C'est tout ; `ncmod.json` n'a besoin d'aucun changement.

Un projet écrit à la main doit copier la cible `EmbedDependencies` du csproj du modèle, ou l'embarquer lui-même :

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

Les dépendances ne sont pas écrites dans le manifeste ; la résolution ne regarde que les ressources `.dll` embarquées du mod, et il suffit que le nom de la ressource corresponde au nom de l'assembly.

Trois remarques :

- **La correspondance se fait par nom d'assembly**. Le loader compare le nom d'assembly demandé : après suppression de `.dll`, le nom de ressource soit égale au nom d'assembly, soit se termine par `.` + le nom d'assembly. Donc `MyLib.dll` et le `MyProject.deps.MyLib.dll` par défaut fonctionnent tous les deux.
- **Seules les ressources embarquées du mod lui-même sont reconnues**. Les bibliothèques non embarquées ne peuvent pas être résolues et ne seront pas recherchées ailleurs.
- **L'ordre de résolution met le noyau en premier**. `EmbeddedAssemblyLoader` cherche d'abord parmi les assemblies du noyau et les sous-bibliothèques embarquées, et ne se rabat sur les mods que si rien n'est trouvé, donc les mods ne devraient pas embarquer d'assemblies portant le même nom que celles du noyau.

La cible du modèle utilise `WithMetadataValue` pour filtrer les `.dll` plutôt que d'écrire `Condition`, parce que le moteur de modèles évalue `Condition` dans le `.csproj` au moment du modèle, quand `%(...)` n'a pas de valeur, et la ligne entière serait supprimée.

### 4.7 Icône et informations d'affichage

Le nom, la description, les auteurs, les liens et l'icône sont tous écrits dans `ncmod.json`, correspondant à `name` / `description` / `authors` / `contact` / `icon` dans le `fabric.mod.json` de vanilla.

**L'icône est une ressource embarquée**, pas un fichier externe, utilisant la même convention de nommage de ressources que les dépendances embarquées :

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

Le modèle fournit déjà un `icon.png` avec les deux emplacements configurés ; remplacez simplement cette image. Un PNG de 64×64 ou 128×128 est recommandé.

L'ordre de recherche de l'icône est : ce vers quoi pointe l'`icon` du manifeste → une ressource embarquée nommée `icon.png` → si aucune des deux n'existe, l'image par défaut de l'interface (un point d'interrogation gris).

Ces informations sont visibles sur la page **MODS** de l'interface graphique du serveur. À gauche se trouve la liste des mods (petite icône + nom d'affichage + version + statut) ; à droite se trouvent la description, les crédits, les liens, les dépendances, le temps d'initialisation et les règles d'injection de celui qui est sélectionné. Les noms de mods dans la colonne des dépendances sont cliquables et sautent directement à cette entrée.

Les mods qui ont échoué au chargement ou qui ont été ignorés sont aussi dans la liste, marqués dans la colonne des statuts — lorsque vous diagnostiquez « pourquoi mon mod n'a-t-il pas pris effet », consultez d'abord cette colonne.

---

## 5. Limitations connues

- **Les signatures de sonde ne peuvent utiliser que des types BCL et `object`**, pour la raison exposée en 2.2 ; les paramètres et valeurs de retour de types valeur font exception et doivent garder leurs types réels.
- **`CallSite` est un remplacement**, et la méthode de remplacement doit restaurer elle-même l'appel d'origine, voir 2.3. Les méthodes privées ne peuvent pas être restaurées et nécessitent la réflexion.
- **Les dll sous `mods/` sont déployées automatiquement par `DeployModToHosts`** ; lorsque vous créez votre propre projet hôte, pensez à ajouter son nom à `ModHostProjects`.
- **Les assemblies d'entrée ne peuvent pas être injectées** : si votre cible et votre hook tombent dans une assembly d'entrée, c'est sans effet.
- **La bibliothèque principale et le loader ne peuvent pas être hookés** ; c'est une contrainte de conception empêchant les mods de modifier le processus de chargement lui-même.
- **Un `environment` incompatible signifie que tout le mod n'est pas chargé**, pas que « certaines règles échouent ».
- **L'injection à l'exécution exige la bibliothèque native** : les règles utilisant `RuntimeInject` exigent que `lead_hook_native` soit attachée au démarrage du processus, et le loader se redémarre pour ce faire ; si la bibliothèque n'est pas trouvée ou si le redémarrage échoue, ce lot de règles est dégradé en avertissement et le démarrage n'est pas bloqué. Pour les limites de capacité et le coût, voir la section [2.6](#26-injection-à-lexécution--modifier-du-code-déjà-en-cours-dexécution).
- **Les annotations ont moins de champs que l'API C#** : `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` ne peuvent pas être écrits dans `[Inject]` (`PatchMode` et `Ordinal` sont pris en charge), voir la section [2.1](#21-styles-dinjection--annotations-ou-manifeste-choisissez-en-un).
- **Les mixins ne s'appliquent qu'à la réécriture au chargement**, et les membres de la classe source sont déplacés plutôt que copiés ; les types imbriqués et les méthodes génériques de la classe source sont hors couverture, et les méthodes mixées avec une interface sont marquées virtuelles. Voir la section [2.8](#28-mixins--ajouter-des-membres-à-un-type-cible).
- Lors des tests et du débogage, si des tables de langue ou des ressources de modèles sont utilisées, un répertoire `assets` (extrait du jar vanilla) est requis, sinon les fonctionnalités liées se dégradent en clés de traduction ou en textures de substitution.

---

## 6. Comportements connus

Cette section est un compte rendu de comportements observés, pas une spécification.

### 6.1 Dépendance entre mods + injection du mod dont on dépend

**Scénario** : b dépend de a et injecte aussi a.

**Conclusion** : cela fonctionne et ne forme pas de cycle.

La chaîne comporte trois étapes :

1. La table de règles est construite purement en lisant les métadonnées PE, sans charger aucune assembly. La règle « b injecte a » n'exige que ni a ni b soient présents.
2. `PreloadReplacers` charge toutes les classes de remplacement (y compris b) avant `ModManager`, moment où a n'est pas encore chargé. `LoadFromStream` ne lit que les métadonnées et ne compile pas par JIT les corps de méthode, donc la référence de b à a est paresseuse à ce moment et le chargement n'échoue pas.
3. `ModManager` trie ensuite topologiquement par `AssemblyRef`, avec a avant b. Au moment de réécrire a, la classe de remplacement b est déjà dans `Default`, donc le type est récupéré directement et un `AssemblyRef` pointant vers b est ajouté aux métadonnées de a. **a n'a pas du tout besoin de savoir que b existe.**

**Exigence stricte** : lors du référencement du mod injecté dans le csproj, vous devez écrire `Private="false"`. Par défaut, il copie `a.dll` dans le répertoire de sortie, qui est ensuite embarqué dans `b.dll` par `EmbedDependencies` en tant que dépendance embarquée, et à l'exécution `ModLibs` résolvant `a` récupère la seconde copie, ce qui fait échouer les vérifications d'identité de type entre les deux.

**Cas de vérification** : deux projets de modèle `NetCraft.Test1` (a) et `NetCraft.Test2` (b) ; a fournit `Test1Api.Greet` et l'appelle dans son propre `ModEntry.Server`, tandis que b remplace ce site d'appel par `Test2Probe.OnGreet` et appelle aussi `Greet` une fois dans le `ModEntry.Server` de b comme contrôle. Journal d'exécution réel :

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← l'injection a pris effet
Test2 internal greeting: Test1 original greeting test2    ← groupe de contrôle : la règle ne réécrit que l'assembly cible, donc le site d'appel interne de b n'est pas touché
```

Les dépendances ne déduisent l'ordre que de `AssemblyRef` et ne portent aucune contrainte de version. Pour contraindre les versions, déclarez-les dans les `depends` du manifeste, voir la section [4.2](#42-champs-de-ncmodjson).

### 6.2 Injection par annotation

**Scénario** : `[Inject(typeof(X), nameof(X.M))]` sur la méthode de remplacement, sans aucune règle dans `ncmod.json`.

**Conclusion** : cela fonctionne et ne nécessite aucune déclaration dans le manifeste. Les annotations et le manifeste partagent une seule source, sont fusionnés à l'assemblage, et l'annotation gagne pour le même point d'injection.

**Pourquoi aucune déclaration n'est nécessaire** : les annotations ne sont que la table `CustomAttribute` dans les métadonnées, et le loader lit déjà les mêmes métadonnées lors du scan des mods (manifeste, ressources embarquées et AssemblyRef — trois éléments), donc lire une table de plus n'introduit aucune nouvelle contrainte de chargement ou de timing. Le seul prérequis est que le mod référence `NetCraft.ModApi` (l'hôte des annotations).

**Le postulat est la lecture statique** : la lecture des annotations doit passer par `MetadataReader` et **ne doit pas utiliser `Assembly.Load` + `GetCustomAttributes`** — ce dernier remonte l'assembly du mod juste pour lire les règles, et la fenêtre de réécriture est perdue sur-le-champ.

**`typeof` ne constitue pas une référence de type** : ce que `typeof(X)` compile dans le paramètre est le nom sérialisé du type (`nom complet, assembly, Version=…`), qui se résout en une simple chaîne et n'exige pas que `X` soit présent. Donc écrire `typeof(mod-injecté)` dans une annotation **ne viole pas** la contrainte de la section 2.2 selon laquelle « une classe de remplacement ne doit pas référencer les types du mod injecté » — un nom apparaissant dans les métadonnées et la résolution d'un type à l'exécution sont deux choses différentes.

**Cas de vérification** : `NetCraft.Test2` contre deux méthodes de `NetCraft.Test1` ; `Greet` passe par l'annotation et `Farewell` par le manifeste. Les deux règles sont installées et les deux sites d'appel sont remplacés :

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← annotation
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← manifeste
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← groupe de contrôle : la règle ne réécrit que l'assembly cible
Test2 internal farewell: Test1 original farewell test2     ← groupe de contrôle
```

Le même cas a aussi vérifié incidemment que le paramètre nommé de l'annotation (`Environment = "server"`) est résolu lui aussi.

### 6.3 Le coût en performance de l'attachement du profiler

**Scénario** : le même programme de pur calcul (cent millions d'itérations modulo-et-accumulation), exécuté une fois sans le profiler et une fois avec lui attaché (les trois variables d'environnement `CORECLR_ENABLE_PROFILING`). Cinq exécutions chacune.

**Conclusion** : aucune différence en régime stable ; le coût est entièrement au démarrage.

| | Temps total du processus (5 exécutions, ms) | Temps de calcul dans le programme |
| --- | --- | --- |
| sans | 306 / 260 / 278 / 253 / 300 | 225 ms |
| avec | 406 / 370 / 340 / 325 / 354 | 193 ms |

Le temps total est environ 80–110 ms plus long. La source est `COR_PRF_DISABLE_ALL_NGEN_IMAGES` — activer ReJIT exige de désactiver en même temps les images ReadyToRun, donc le code du framework ne peut passer que par le JIT ; la boucle chronométrée à l'intérieur du programme est identique une fois compilée par JIT, ne montrant aucune différence (l'exécution avec le profiler était en fait légèrement plus rapide, ce qui est du bruit).

**Un gaspillage aussi corrigé au passage** : l'implémentation initiale s'abonnait à `COR_PRF_MONITOR_JIT_COMPILATION`, un callback natif déclenché après la compilation de chaque méthode, que nous n'utilisions jamais. Après l'avoir retiré, le masque d'événements est passé de `0x80040024` à `0x80040004` ; le tableau ci-dessus correspond aux données après retrait.

**Cas de vérification** : `__hookverify/BenchProbe`.

### 6.4 Deux mods injectant la même cible

**Scénario** : deux mods déclarent chacun une règle qui touche la même méthode cible (forme `MethodBody`, méthodes de remplacement différentes).

**Conclusion** : aucune erreur, aucun plantage ; **la règle assemblée en premier gagne et la seconde échoue silencieusement**.

Les deux règles entrent dans la table de règles — aucune déduplication inter-mods n'est effectuée. Au moment de la réécriture, la **première** entrée de la liste de même clé pour `OriginalType::OriginalMethod` est prise ; la portée d'hôte se comporte de la même façon, `InType`/`InMethod` étant « la première correspondance l'emporte ». Laquelle est première dépend de l'ordre d'assemblage, et l'ordre d'assemblage vient de l'ordre d'énumération du répertoire des mods ; **il n'y a pas de champ de priorité et cela ne peut pas être contrôlé en déclarant des dépendances** (les dépendances n'affectent que l'ordre de `Init()`, pas l'assemblage des règles d'injection).

**L'injection à l'exécution est aussi premier arrivé, premier servi** : une requête d'enregistrement ultérieure est envoyée normalement, mais `GetReJITParameters` revendique par « module + méthode » et correspond toujours à la première requête, donc l'enregistrement ultérieur ne s'applique pas. Mesuré après une seconde injection, le comportement de la méthode cible reste au premier résultat.

**Une règle défectueuse n'affecte pas les autres** : des problèmes comme le type cible absent de toute assembly connue ou une forme d'injection mal orthographiée sont consignés dans `ModHooks.Errors` au moment de l'assemblage et cette entrée est ignorée, tandis que les règles des autres mods sont assemblées normalement.

**Les conflits sont consignés** : quand l'assemblage détecte le même point d'injection déclaré par plusieurs mods, celui assemblé en dernier est écrit dans `ModHooks.Warnings` et affiché dans le journal de démarrage sous la forme `Mod injection conflict ...`, nommant quels deux mods sont entrés en collision et lequel ne prendra pas effet. C'est un avertissement, pas une erreur, cela n'affecte pas le chargement, et la règle elle-même reste dans la table (simplement inatteignable).

**L'injection à l'exécution passe aussi par cette vérification** : la clé de détection de conflit inclut `patchMode`, donc un mod écrivant `ILRewrite` et un autre écrivant `RuntimeInject` ne compte pas comme une collision (deux chemins indépendants, chacun faisant son propre travail) ; seuls deux du même mode sont jugés en conflit et avertis. Son application effective est de même premier arrivé, premier servi — `GetReJITParameters` revendique par « module + méthode », correspondant à la première requête, donc les suivantes sont envoyées mais ne s'appliquent pas.

**Le seul point de plantage dur** : quand deux mods patchent tous deux la même méthode en mode `RuntimePatch`, le second atteint la vérification de double enregistrement de `RuntimeHookEngine` et lève `InvalidOperationException`, et ce chemin n'est pas intercepté, donc le démarrage échoue complètement. `RuntimePatch` est en voie de disparition (voir la section [2.6](#26-injection-à-lexécution--modifier-du-code-déjà-en-cours-dexécution)) ; ne l'utilisez pas dans de nouvelles règles.

**Cas de vérification** : le module `modinjection` de `NetCraft.Test`, entrées `same anchor first mod wins quietly` et `one bad rule does not sink the rest`.

### 6.5 Pas encore vérifié

- **L'injection à l'exécution fonctionnant sur un vrai serveur** : le mode `RuntimeInject` a été vérifié de bout en bout dans `__hookverify/RuntimeProbe` (après enregistrement, le comportement de la méthode cible est remplacé par celui de la méthode de remplacement), et le côté assembly du mod a aussi une couverture de test pour le routage et la dégradation ; mais aucune étape de build ne place actuellement `lead_hook_native` dans le répertoire d'exécution de NC, donc exécuter cette chaîne sur un vrai serveur exige de placer d'abord la bibliothèque à la racine du programme (ou de la pointer avec `NC_PROFILER_PATH`). Cette étape n'est pas faite.
- **Une classe de remplacement référençant les types du mod injecté** : par raisonnement, au moment de réécrire a, elle résoudrait a, tandis que a est bloqué juste avant la fin du chargement (il n'est pas encore dans `Default`, et ni `ModLibs` ni le callback de résolution du noyau ne reconnaît les assemblies de mods), donc `PrepareMod` devrait lever une exception et consigner dans `result.Errors`. Pas encore réellement exécuté. Notez que la section 6.2 ne prouve que le fait que **`typeof` dans une annotation** n'est pas une référence ; **le type apparaissant dans une signature de méthode** est une autre affaire.

---

## Annexe : aperçu des HookType

| Forme | Effet | Exigence sur la signature de la méthode de remplacement |
| --- | --- | --- |
| `CallSite` | remplacer les sites d'appel de la méthode cible par votre méthode | le nombre de paramètres correspond à la méthode appelée (appel d'instance +1) |
| `MethodBody` | remplacer tout le corps de la méthode cible | correspond à la méthode remplacée |
| `NewObj` | remplacer `new X(...)` | le nombre de paramètres correspond au constructeur |
| `FieldRead` | instrumenter les lectures de champs | selon le type lu |
| `FieldWrite` | instrumenter les écritures de champs | selon le type écrit |
| `TypeCheck` | instrumenter `isinst` / `castclass` | selon le type vérifié |
| `Box` | instrumenter le boxing/unboxing | selon le type d'élément |
| `FunctionPointer` | instrumenter les chargements de pointeurs de fonction | selon le type de délégué |
| `LocalRead` | instrumenter les lectures de variables locales | zéro paramètre, renvoie la valeur de la variable |
| `LocalWrite` | instrumenter les écritures de variables locales | un paramètre, reçoit la valeur écrite |
| `Constant` | instrumenter les chargements de constantes | zéro paramètre, renvoie la valeur de la constante |
| `Probe` | garder le corps de méthode d'origine, en instrumentant l'entrée et chaque sortie ; avec `LabelArgumentIndex`, un argument peut être replié dans le label | `Begin()` renvoie long, `End(string, long)` |
| `Mark` | signaler une fois à l'entrée de la méthode seulement, sans chronométrage | `void method(string label)` |

`Probe` et `Mark` ne passent que le texte du label (`Probe` peut aussi inclure le `ToString()` d'un argument) ; ils ne peuvent pas obtenir de références d'objets. Pour obtenir les arguments réels, utilisez `CallSite`.

`InType`/`InMethod`, `Placement` et `Ordinal` (voir la section [2.4](#24-se-restreindre-à-un-seul-site--portée-de-lhôte-et-placement)) n'ont de sens que pour les formes au niveau instruction : les dix entrées du tableau ci-dessus autres que `MethodBody`, `Probe` et `Mark` peuvent choisir remplacer ou insérer avant/après, et peuvent utiliser `Ordinal` pour choisir une seule occurrence ; `MethodBody` remplace toujours l'ensemble, et `Probe`/`Mark` ignorent ces paramètres.

Pour les trois sortes `LocalRead` / `LocalWrite` / `Constant`, la méthode hôte est écrite dans `target` plutôt qu'une entité référencée, et `localIndex` ou `constantValue` est en plus requis ; voir la section [2.5](#25-ancres-dans-le-corps-de-méthode--variables-locales-et-constantes).
