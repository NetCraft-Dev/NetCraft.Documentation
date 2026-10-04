# NetCraft-ModApi リファレンス

`NetCraft.ModApi` は、NetCraft が MOD に公開する API サーフェスです。これには 2 つのアイデンティティがあります。あなたにとっては API ライブラリです。それ自体は通常の MOD です (`id` は `netcraft-modapi` であり、独自の `ncmod.json` と注入プローブを同梱しています)。

API が成長するにつれて、このファイルも成長します。アーキテクチャの背景、Fabric との違い、MOD の書き方については、[modding-guide.md](modding-guide.md) を参照してください。

- アセンブリ: `NetCraft.ModApi.dll`

- 依存関係: `NetCraft` (メイン ライブラリ)、`NetCraft.Game`

パブリック サーフェスは 3 つの名前空間に分割されます。

|ネームスペース |目次 |メモ |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` |イベントおよびサブスクリプションの基本クラス `NcEvent<T>`、`Nc*` ファサード、`Nc*` オブジェクト ハンドル |ラッパー層。カーネルタイプは公開されていません。
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` 注釈 |拡張ポイント。ルールはカーネルのクラス名とメソッド名にバインドされます。
| `NetCraft.ModApi.Internal` |インジェクションプローブ |直接参照しないでください |

ルート名前空間 `NetCraft.ModApi` には、エントリ クラス `ModApiEntry` のみが含まれます。 `Wrapper` と `Extension` は 2 つの並列ルートです。選択方法については、[modding-guide.md 2.9](modding-guide.md#29-two-routes-wrapper-layer-and-extension-points)を参照してください。

***

## 1. クイックスタート

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

`ncmod.json` では、このクラスの `entry` をポイントし、`hooks` は空のままにします。以下のイベントはすべて ModApi 独自のプローブによって提供されます。

***

## 2. イベント

すべてのイベントは `NetCraft.ModApi.Wrapper` の下に存在します。 `using NetCraft.ModApi.Wrapper;` 以降は利用可能です。

### 2.1 概要表

|イベント |引数の型 |トリガー |側面 | ModApi フックポイント |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick` | `ServerTickArgs` |サーバーのメインループの各ティック |サーバー | `DedicatedServer::Tick` (マーク) |
| `ServerEvents.Started` | `ServerPhaseArgs` | `Done (x.xxxs)!` が出力された後、メインループが開始されました。サーバー | `MinecraftServer::Run` (マーク) |
| `ServerEvents.Stopping` | `ServerPhaseArgs` |サーバーがシャットダウンを開始します。プレーヤーが接続解除されようとしています |サーバー | `DedicatedServer::Stop` (マーク) |
| `ServerEvents.CommandRegister` | `CommandRegisterArgs` |すべての組み込みコマンドが登録されました。サーバー | `EffectCommand::Register` コール サイト (CallSite) |
| `ServerEvents.PlayerJoin` | `PlayerJoinArgs` |参加パケット シーケンスが送信されました。サーバー | `PlayerList::PlaceNewPlayer` コール サイト (CallSite) |
| `ServerEvents.PlayerLeave` | `PlayerLeaveArgs` |プレーヤーがオンライン リストから削除されました |サーバー | `PlayerList::RemovePlayer` コール サイト (CallSite) |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` |切断パケットが送信され、接続が閉じられました。サーバー | `ServerPlayer::Disconnect` コール サイト (CallSite) |
| `ServerEvents.PlayerHurt` | `PlayerHurtArgs` |実際に与えられたダメージ |サーバー | `PlayerList::HurtPlayer` コール サイト (CallSite) |
| `ServerEvents.PlayerDeath` | `PlayerDeathArgs` |健康状態がゼロにリセットされた直後 |サーバー | `PlayerList::RespawnPlayer` コール サイト (CallSite) |
| `ServerEvents.PlayerChat` | `PlayerChatArgs` |チャットブロードキャスト後 |サーバー | `ServerGamePacketListenerImpl::HandleChat` コール サイト (CallSite) |
| `ServerEvents.ChunkLoaded` | `ChunkLoadedArgs` |チャンクが初めてメモリに入ります。サーバー | `ServerChunkCache::set_ChunkLoaded` 割り当てサイト (CallSite) |
| `ServerEvents.ChunkUnloaded` | `ChunkUnloadedArgs` |チャンクはメモリを残します |サーバー | `ServerChunkCache::set_ChunkUnloaded` 割り当てサイト (CallSite) |
| `ServerEvents.ChunkSaved` | `ChunkSavedArgs` |チャンクがディスクに書き込まれる前に作成されたスナップショット |サーバー | `ServerChunkCache::set_ChunkSaveSink` 割り当てサイト (CallSite) |
| `ServerEvents.CommandExecuted` | `CommandExecutedArgs` |コマンドの実行が終了しました。構文エラーとアクセス許可の拒否もカウントされます。サーバー | `CommandManager::Execute` (コールサイト) |
| `ServerEvents.LevelTick` | `LevelTickArgs` |レベル ティック、ロードされたレベルごとに 1 ティックごとに 1 回 |サーバー | `PersistentServerLevel::Tick` (コールサイト) |
| `ServerEvents.SavedDataSaving` | `SavedDataSavingArgs` |チャンク保存の 1 ステップ後に、保存データがディスクに書き込まれます。サーバー | `SavedDataStorage::ScheduleSave` (コールサイト) |
| `ServerEvents.BlockChanged` | `BlockChangedArgs` |ブロックの状態が変更されました。クライアントと同期しようとしています。サーバー | `IBlockUpdateSink::BlockChanged` (コールサイト) |
| `ServerEvents.BlockBroken` | `BlockBrokenArgs` |ブロックが壊れた。プレイヤーの採掘とレッドストーンの自己破壊は両方ともカウントされます |サーバー | `ServerBlockUpdates::BreakBlock` (コールサイト) |
| `ServerEvents.ItemDropped` | `ItemDroppedArgs` |生成されたドロップアイテムエンティティ（ブロック破壊ドロップや調理製品を含む） |サーバー | `ServerBlockUpdates::SpawnDrop` (コールサイト) |
| `NetworkEvents.PacketReceived` | `PacketReceivedArgs` |ハンドシェイクおよびステータスフェーズを含む、ハンドラーのキューに入れられたすべての受信パケット。両方 | `PacketProcessor::ScheduleIfPossible` および `HandleNow` (CallSite) |
| `ClientEvents.Tick` | `ClientTickArgs` |クライアントのメインループの各ティック |クライアント | `MinecraftClient::Tick` (マーク) |

プレイヤー イベントの順序: 死はダメージ フロー内にネストされているため、`PlayerDeath` は対応する `PlayerHurt` の前にあります。 `PlayerLeave` と `PlayerDisconnect` は 2 つの異なるものです。前者はオンライン リストからの削除を意味し (`/kick` の後でも、接続が切断された場合にのみ起動されます)、後者は接続そのものが切断されることを意味し、この 2 つがペアで表示されることは保証されていません。

### 2.2 購読と登録解除

```csharp
IDisposable Subscribe(Action<T> handler)
```

- 同じイベントを複数回サブスクライブすると、サブスクリプションの順序で通知が配信されます。

- ディスパッチはコールバック リストのスナップショットを取得するため、コールバック内からのサブスクライブまたは登録解除は現在のディスパッチには影響しません。

- 登録を解除しなくても、永久に有効です。 MOD にはアンロード メカニズムがないため、通常は手動で登録を解除する必要はありません。

### 2.3 引数の型

**`ServerTickArgs`**

|プロパティ |タイプ |メモ |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` |この起動以降のティック数 (1 から始まります) |

これはカーネルの `TickCount` ではなく、ModApi 自体によってカウントされることに注意してください。

**`ClientTickArgs`**

|プロパティ |タイプ |メモ |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` |上記と同様、クライアント側で独立してカウントされます。

**`ServerPhaseArgs`**

|プロパティ |タイプ |メモ |
| -------- | -------- | ---------------------------- |
| `Phase` | `string` |フェーズ名、`started` または `stopping` |

フィールドはイベント自体を複製します。これは、ロギングで 1 つの統一フォーマットを使用できるように保持されます。

**`CommandRegisterArgs`**

|メンバー |タイプ |メモ |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher` | `CommandDispatcher<CommandSourceStack>` |カーネルのコマンドディスパッチャ |
| `Register(name, description, build)` |方法 |コマンドを登録し、台帳に記録します。3.1 | を参照してください。

**`PlayerJoinArgs`**

|プロパティ |タイプ |メモ |
| ------------- | -------------- | ---------------------------------------------- |
| `Player` | `ServerPlayer` |参加したばかりのプレーヤー。結合パケットはすでに送信されており、状態は安全に読み取れます。
| `ProfileName` | `string` |プレイヤー名 |

**`PlayerLeaveArgs`**

|プロパティ |タイプ |メモ |
| --------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` |退団プレイヤー、現時点ではオンラインリストには存在しない |
| `Removed` | `bool` |実際に削除されたかどうか。繰り返し取り外した場合の `false` |

**`PlayerDisconnectArgs`**

|プロパティ |タイプ |メモ |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` |切断されたプレーヤー |
| `Reason` | `string` |切断理由。コンポーネントとして与えられる場合はプレーンテキスト |

切断パケットが送信され、接続が閉じられました。このプレーヤーにパケットを送信しても効果がなくなりました。

**`PlayerHurtArgs`**

|プロパティ |タイプ |メモ |
| ---------- | --------------- | -------------------------------------------- |
| `Player` | `ServerPlayer` |負傷した選手 |
| `Attacker` | `ServerPlayer?` |ダメージを与えたプレイヤー。 `null` 環境およびコマンドによる損傷 |
| `Amount` | `float` |今回の被害額 |

無敵フレーム中または死亡後は起動しません (カーネルの `Hurt` は `false` を返します)。

**`PlayerDeathArgs`**

|プロパティ |タイプ |メモ |
| ---------- | --------------- | ------------------------------ |
| `Player` | `ServerPlayer` |死亡したプレイヤー |
| `Attacker` | `ServerPlayer?` |殺人者。 `null` がない場合 |

カーネルはヘルスがゼロになるとすぐにリセットされるため、イベントが発生すると、プレーヤーはリスポーンポイントですでに完全なヘルスになっています。死亡時の座標とドロップは取得できません。

**`PlayerChatArgs`**

|プロパティ |タイプ |メモ |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` |送信者名 |
| `Message` | `string` |プレーンテキストメッセージ |

このイベントは **読み取り専用の通知**です。元のメソッドはすでにメッセージをブロードキャストしているため、ここで `Message` を変更しても効果はありません。

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

3 つのイベントは同じ引数の形状を共有します。

|プロパティ |タイプ |メモ |
| -------- | ----- | ------------------ |
| `X` | `int` |チャンク座標 X |
| `Z` | `int` |チャンク座標 Z |

最も間違いやすいのは `ChunkSaved` です。カーネルは**スナップショットを同期的に取得する**ためにこのコールバックを必要としますが、シリアル化とディスク書き込みはカーネル自体によって非同期的に行われます。そのため、このコールバックでの時間のかかる作業はチャンクのアンロードの速度を直接低下させます。また、破壊的な操作 (ブロックの削除、インベントリの変更) もここで行うべきではありません。これはスナップショットの瞬間を約束するだけです。

`ChunkUnloaded` が起動すると、ブロック エンティティはチャンクとともにすでにクリーンアップされています。ブロックを読み取りたい場合は、`ChunkSaved` (ブロックにもアクセスできません) またはそれ以前のポイントを使用してください。

**`CommandExecutedArgs`**

|プロパティ |タイプ |メモ |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string` |生のコマンドテキスト。チャットコマンドには先頭にスラッシュがありません |
| `Result` | `int` |コマンドの戻り値。 0 は失敗または拒否を意味します。
| `Source` | `CommandSourceStack?` |コマンドソース。 player-overload パス上の `null` |
| `Player` | `ServerPlayer?` |コマンドを発行したプレイヤー。 `null` コンソールから発行された場合 |

イベントはコマンドの完了後**に発生します。実行を変更することはできません。構文エラーや権限の拒否もここで発生します。それらを区別するには `Result` を使用してください。プレイヤーがチャット バーから送信するコマンドは、カーネルが内部でコマンド ソースを構築する `Execute(ServerPlayer, string)` オーバーロードを通過します。その場合、`Source` は `null` となり、`Player` のみが設定されます。

**`LevelTickArgs`**

|プロパティ |タイプ |メモ |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level` | `NcLevel` |このティックで進んだレベル |
| `RunsNormally` | `bool` |正常に進んだかどうか。 `/tick freeze` 中の `false` |

ロードされたレベルごとに 1 ティックごとに 1 回発火するため、マルチレベルのワールドはティックごとに複数のパケットを受け取ります。トリガーポイントはレベルティックが**完了**した後です。それは観測点であり、傍受点ではありません。

**`SavedDataSavingArgs`**

|プロパティ |タイプ |メモ |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage` |保存されたデータ テーブルが永続化される |

これは `ServerEvents.ChunkSaved` とは異なるパスです。チャンクはスナップショットを取得して非同期に書き込むだけですが、このチャンクは同期書き込みが完了した後に起動されます。世界時計、ゲームのルール、世界の国境データが通過します。

**`BlockChangedArgs`**

|プロパティ |タイプ |メモ |
| -------- | ------------ | --------------------------- |
| `Pos` | `BlockPos` |変更されたブロックの位置 |
| `State` | `BlockState` |変更後のブロック状態 |

状態はすでにチャンクに書き込まれており、クライアントに同期しようとしているため、変更自体をここで変更することはできません。動作コールバックで自身の状態を変更するレッドストーン コンポーネントも、このパスを通じて終了します。これは高頻度であるため、コールバックで時間のかかる作業を行わないでください。

**`BlockBrokenArgs`**

|プロパティ |タイプ |メモ |
| -------- | --------------- | ------------------------------------------ |
| `Pos` | `BlockPos` |壊れたブロックの位置 |
| `Player` | `ServerPlayer?` |ブレーカー。 `null` レッドストーンなどのプレイヤー以外の原因によるもの |

ブロックが実際に置き換えられた場合にのみ起動します。空のポジションや拒否されたブレークはトリガーしません。ブレイク効果やドロップは既に対応済みなのでイベントで読む内容になります。

**`ItemDroppedArgs`**

|プロパティ |タイプ |メモ |
| -------- | ----------- | ----------------------- |
| `Pos` | `BlockPos` |ドロップしたアイテムが出現した場所 |
| `Stack` | `ItemStack` |ドロップされたアイテムスタック |

ブロック崩しのドロップとキャンプファイヤー調理用製品の両方がここを通過します。空のアイテムスタックはエンティティを生成しないため、イベントは発生しません。

**`PacketReceivedArgs`**

|プロパティ |タイプ |メモ |
| --------------- | -------- | ------------------------ |
| `Listener` | `object` |このパケットを受信するリスナー |
| `Packet` | `object` |パケットオブジェクト自体 |
| `IsServerbound` | `bool` |サーバーバウンドパケットかどうか |

すべての受信パケットに対して起動され、ハンドシェイク、ステータス、構成、再生の 4 つのフェーズすべてをカバーします。移動パケットはティックごとに数回到着するため、コールバックで時間のかかる作業を行わないでください。パケットはすでにオブジェクトにデコードされていますが、ビジネス層にはまだ入っていません。タイプを区別するには、`Packet` を自分で調べてください。送信パケットはこのイベントの範囲外です。

***

## 3. 拡張ポイント

### 3.1 コマンドの登録

タイミングは`ServerEvents.CommandRegister`です。このイベントの引数をキャッシュしないでください。内部コマンド ツリーは起動時に 1 回だけ構築されます。

```csharp
public void Register(
    string name,                                          //command literal, without the slash
    string description,                                   //one-line description, shown in the /ncmapi ledger
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //attach arguments and the executor
```

受け取る `build` はカーネルの准ビルダーです。引数、サブコマンド、およびパーミッションを記述して、カーネルの方法を規定します。

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //permission predicate
            .Executes(context => { /* ... */ return 1; })));
```

引数あり:

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

重要なポイント:

- `args.Dispatcher.Register(...)` を直接呼び出してもコマンドはインストールされますが、台帳には入力されず、`/ncmapi` では表示されません。リストしたい場合は `args.Register` を使用してください。

- デフォルトではコマンドに権限制限はありません。必要に応じて `.Requires(...)` を自分で追加してください。

- 実行時の動作は完全にあなた次第です。 ModApi はそれをインターセプトしません。

### 3.2 登録されたコマンドの表示

組み込みの `/ncmapi` があり、アクセス許可レベル 2 が必要です。

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

使用行はコマンド ツリーのノード構造からオンザフライで計算されます。リテラルは名前で書かれ、引数は山かっこで囲まれ、それ自体が実行可能な中間ノードも独自の行を取得します。

***

## 4. サーバーのファサード

この章のファサードはすべて `NetCraft.ModApi.Wrapper` の下に存在します。 `using NetCraft.ModApi.Wrapper;` 以降は利用可能です。

ファサードは、カーネル全体に散在する機能をいくつかのエントリ ポイントに収集する `Nc*` 静的クラスです。カーネル インスタンスは、メイン ループの開始時にプローブによってキャプチャされます。 `NcServer.IsAvailable` が false の場合、以下のすべてがスローされます。これらは `Init` からではなく、イベント コールバック内でのみ使用されます。

|ファサード |目的 |
| --- | --- |
| `NcServer` |サーバー インスタンス、ティック レート、コマンド、エンティティ トラッキング、プレイヤー データ、ゲーム ルール、ブロードキャスト、コマンド実行 |
| `NcPlayers` |オンライン プレーヤーのクエリと操作 (キック、テレポート、ヘルス、ゲーム モード、権限) |
| `NcWorld` |オーバーワールドブロックの読み取り/書き込みと破壊、天気、時間、国境、時計、サウンド、レベルイベント。他の次元に到達するには `NcLevel` ハンドルを使用します。座標はプレーンな `x y z` ints |
| `NcRegistries` |名前で検索される組み込みレジストリ (ブロック、アイテム、流体、エフェクト、バイオーム、パーティクル、エンティティ、ブロック エンティティ) |
| `NcRecipes` |レシピ クエリ (グリッド クラフト、石切り、料理、ID によるレシピの取得) |
| `NcLists` |リストと設定 (ホワイトリスト、運用、禁止、`server.properties`) |
| `NcStartup` |起動引数 (カーネルが認識できないトークンと名前ベースのサブスクリプション) |

`NcPlayer` は静的ファサードではなく **オブジェクト ハンドル**: `NcPlayers.All` / `Find` はそれを返し、プレーヤー イベントの `Player` / `Attacker` もそれです。ハンドルは読み取り専用であり、プローブによって構築されます。 MOD はカーネルの `ServerPlayer`、つまり「公開面にカーネル タイプがない」という最初のアンカーを取得できません。同じカーネル プレーヤーは常に同じハンドルにマップされ、弱い参照によって内部的にキャッシュされ、プレーヤーがログオフすると自動的に無効になります。

`NcLevel` はレベルに対して同じ形状に従います。 `NcWorld.Overworld` / `Nether` / `End` および `NcWorld.Get("minecraft:the_nether")` はそれを返します。`LevelTickArgs.Level` も同様です。これには、ディメンション ID、時間、天気、ビルドの高さ、ティック数、およびチャンクの強制読み込みが含まれます。ブロック操作は `NcWorld` に留まり、ハンドルに `x y z` を加えたものを取ります。 `BlockPos` は決して表示されないため、MOD の DLL にはカーネル レベルのタイプへの参照がありません。

### 4.1 レジストリ

`NcRegistries` は、テーブル全体と名前による検索の両方を提供します。テーブル全体は反復とタグベースの検索用です。名前による検索は、単一の要素を取得するためのものです。

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //block default state
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//iterate the whole table
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

レジストリは起動中に段階的にアセンブルされ、MOD はアセンブリが完了する前にロードされるため、`Init` で検索したものはキャッシュしないでください。アセンブリはまだ進行中であり、キャッシュされた値は null 参照または古い値になります。現在、`BuiltInRegistries.BootStrap` はまだ空の実装です。各レジストリは独自のブートストラップによって個別に設定され、データ駆動型のレジストリ (バイオーム、レシピなど) には、データ パックの読み込みが完了するまでのエントリがほとんどありません。

### 4.2 レシピ

`NcRecipes` は、データ パックからロードされたレシピ テーブルによってサポートされています。 `/reload` はテーブル全体を置き換えるので、リロードするたびに `RecipeHolder` を保持しないでください。

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

## 5. 内部構造

MOD を記述するにはこのセクションは必要ありませんが、デバッグするときに役立つ場合があります。

### 5.1 プローブ

|クラス |フォーム |責任 |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)` |マーク×4 |すべての「何かが起こった」シグナルは 1 つのメソッドに集められ、`label` | によって一致するイベントにディスパッチされます。
| `Internal.CommandProbe.OnCommandsReady(object)` |コールサイト | `EffectCommand::Register` への呼び出しを置き換えます。元の呼び出しを復元した後、`CommandRegister` | が実行されます。
| `Internal.PlayerProbe.OnXxx(object, ...)` |コールサイト ×6 |プレーヤー イベント。フック ポイントごとに 1 つのメソッド。元の呼び出しを復元した後、イベントを発行します。
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` |コールサイト ×3 | `ServerChunkCache` の 3 つのコールバック プロパティの割り当てサイトにフックされるチャンク イベント。制御をカーネルに戻す前に、ラッパー デリゲートが重ねられます。
| `Internal.BlockProbe.OnXxx(...)` |コールサイト ×3 |イベントをブロックします。フック `ServerBlockUpdates` を切断してドロップし、状態変更はインターフェイス メソッドを `IBlockUpdateSink` にフックします。

`SignalProbe` の署名は `string` のみを受け取り、`CommandProbe`、`PlayerProbe`、`LevelProbe` のパラメータは `object` として宣言されます。これは意図的なものです。アセンブリ中に `Lead.Hook` が置換メソッドの署名を解決し、その中にカーネル型が現れると、解決によってカーネル アセンブリが早期にプルアップされ、注入がそのウィンドウをミスします。カーネル型はメソッド本体内にのみ表示され、その時点でコードはすでに実行されています。

シグネチャ内で `object` にすることができないのは、値型パラメータと戻り値だけです。`object` はスタック上の参照ですが、`float`/`bool` は値であり、不一致は無効な IL です。したがって、`PlayerProbe.OnHurtPlayer` はダメージ量として `float` を保持し、`OnRemovePlayer` と `OnHurtPlayer` は戻り値 `bool` を保持します。

`BlockProbe` はこの制約の拡張です。ブロックの位置と状態は 2 つの値タイプ `BlockPos`/`BlockState` であり、それ自体として署名にのみ書き込むことができます。これら 2 つの型は `NetCraft.Primitives` と `NetCraft.Registry` に由来しており、どちらも注入リストには載っていないため、アセンブリ中にこれらを解決しても、早期に書き換えられるアセンブリが取得されません。

### 5.2 フックポイントリスト

ModApi の `ncmod.json` には 24 のルールが含まれており、2.1 のテーブルと 1 対 1 で一致します。フック ポイントを変更するか、ルールを追加するには、このファイルを編集します。編集後、再構築し (これは埋め込みリソースです)、結果の dll を `mods/` に戻します。後者は `NetCraft.ModApi.csproj` の `DeployModToHosts` によってすでに自動的に行われており、これが欠けていると、ルールがまったく有効になっていないことがわかります。

`CommandManager::Execute` には、1 つのルールを共有する 2 つのオーバーロードがあります。 `Lead.Hook` の CallSite は、パラメーター リストではなく、「型 + メソッド名」で呼び出しサイトを照合します。両方のオーバーロードは 2 つのパラメーターを取るため、プローブは最初のパラメーターとして `object` を受け取り、実際の型でディスパッチできます。

2 つの `PacketProcessor` ルールは補完的です。再生フェーズのパケットは `ScheduleIfPossible` を経由してメインスレッド キューに入りますが、ハンドシェイクとステータス フェーズは即時に処理するために `HandleNow` を経由します。特定のパケットはそのうちの 1 つだけを通過します。前者のみをフックすると、ハンドシェイク フェーズとステータス フェーズが見逃されます。これはスクリプトで調べるのが最も簡単なため、デバッグ中に「ルールが有効ではなかった」と誤解されやすくなります。

3 つのチャンク ルールは、読み取りサイトではなく、`ServerChunkCache` の 3 つのコールバック プロパティの **割り当てサイト** をフックします。その理由は、これら 3 つのプロパティはユニキャストであり、`PersistentServerLevel` の構築時にカーネル自体によってすでに占有されているためです (保存およびブロック エンティティのクリーンアップ ロジックが挿入されます)。 MOD を直接割り当てると、カーネルのコピーがオーバーライドされます。アンロードは保持されず、ブロック エンティティはクリーンアップされず、エラーはまったく発生しません。割り当てサイトをフックすると、その時点でカーネルのコールバックとプローブがチェーンされるようになります。割り当ては 1 回だけ発生し、後続のトリガーごとにデリゲート転送のレイヤーが 1 つ追加されます。

`PlayerList::RespawnPlayer` はプライベートであるため、プローブは元の呼び出しを復元できません。つまり、リフレクションが実行されます (死亡ごとに 1 回呼び出されるため、オーバーヘッドは無視できます)。これにより、カーネル調整のための扉も開いたままになります。将来 `InternalsVisibleTo` が追加された場合、直接呼び出しに置き換えることができます。

### 5.3 元帳

`Internal.NcCommandRegistry` は、`args.Register` を通じて登録されたコマンドを記録します。これは単なる台帳であり、コマンドの実行には関与しません。コマンド自体はカーネル ディスパッチャーにインストールされるため、台帳に問題がある場合でもコマンドは引き続き機能します。

***

## 6. 追加予定

以下は、場所は確認されているがまだイベントになっていないフック ポイントです (リストはリポジトリ ルートの `__scan_mod_api.py` によって生成されました)。

|方向 |フックポイントの候補 |
| ----------------- | ---------------------------------------------------------------------------- |
|エンティティ | `ClientLevel::AddEntity`、`Entity::Die` |
|世界 |レベルのロードとアンロード、`ServerChunkCache` チャンクのバッチ処理 |
|地形生成 | `ChunkStatus`、`WorldGenRegion::SetBlockState` ごとの `ChunkGenerator::Generate` ステージ |
|コマンド実行 | `CommandSourceStack::SendSuccess` / `SendFailure` (応答の半分、多くのコール サイト) |
|ネットワーク | per-packet-type `ServerGamePacketListenerImpl::HandleXxx` (現在は統合エントリ ポイントのみ) |

すでに完了した指示: レベル ティックは `ServerEvents.LevelTick`、保存データの永続性は `ServerEvents.SavedDataSaving`、コマンドの実行は `ServerEvents.CommandExecuted`、ブロックは `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped` になりました。

`SetBlock` にブロック セルをフックすることは機能しません。これには 2 つのデフォルト パラメータ `notifyNeighbors` と `strict` があるため、コンパイルされた呼び出しサイトは 4 ～ 6 個のパラメータを受け取ります。また、CallSite はパラメータ リストを参照せずに「タイプ + メソッド名」で一致するため、1 つの置換メソッドで 3 つのスタック形状すべてを処理できません。代わりに、状態同期フック `IBlockUpdateSink::BlockChanged` (クライアント同期ですべての変更をカバーする `ServerLevel.SetBlock` の唯一のインターフェイス呼び出し) と、フック `ServerBlockUpdates` 独自のメソッドの中断とドロップの 2 か所をフックします。

エンティティセルが厄介なのは、`Entity`がインジェクションリストにない`NetCraft.Registry`に定義されているため、それを対象とした呼び出しサイトを書き換えることができないためです。 `ClientLevel::AddEntity` はクライアント アセンブリ内にあり、実行可能です。死亡イベントでは、まず `Registry` を書き換えられるかどうかを解決する必要があります。

ネットワーク セルの場合、`HandleChat` は以前から存在しており、`NetworkEvents.PacketReceived` は統合されたエントリ ポイントを提供するため、`HandleXxx` ごとのフックの価値ははるかに低くなります。追加する価値があるのは、パケット タイプによるきめ細かいフィルタリングが必要なシナリオのみです。

レベルのロードとアンロードには NC 側に収束点がありません。`DedicatedServer::CreateLevel` はプライベートであるため、元の呼び出しは `PlayerList::RespawnPlayer` のようなリフレクションによってのみ復元できます。アンロードパスはさらに分散しています。これを行うには、まずイベントの引数を何にするかを決定します。

残りのものはすべてオブジェクト参照 (エンティティ インスタンスなど) を必要とするため、`Mark` ではなく `CallSite` を使用する必要があります。カーネル値の型がパラメータの中に現れる場合、それ自体として置換メソッドのシグネチャにのみ書き込むことができます。

インターフェイス サーフェス リスト (`__scan_mod_api.py --api` によって作成された `__modapi_api.txt`) もレビューされました。機能エントリ ポイントとして認定されたエントリは、ドメインごとに第 4 章のファサードに集められ、公開されていない残りのカテゴリは、プロトコルとパケット処理 (`Network.Protocol.*`)、レンダリングとモデル (`Client.Render.*`)、地形生成と密度関数 (`LevelGen.*`) の 3 つのカテゴリに分類されます。これらはカーネルの内部構造です。それらを直接使用すると、MOD が実装の詳細に関連付けられるため、最初に安定したインターフェイスをカーネルで開く必要があります。

サーバー側には、ファサードとしてラップされていないものがさらに 2 つあります。`ReloadableServerResources` インスタンスは `DedicatedServer` からハングしており、ModApi は `NetCraft.Server` を参照していないため、ラップするには最初にカーネル基本クラスのプロパティを開く必要があります。 `ChunkSender` と `ServerWorldBorderListener` は内部フローであり、MOD の使用例はありません。
