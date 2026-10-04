# NetCraft-ModApi リファレンス

`NetCraft.ModApi` は NetCraft が Mod に公開する API サーフェスである。これには2つの顔がある: あなたにとっては API ライブラリであり、それ自身にとってはごく普通の Mod である（`id` は `netcraft-modapi` で、独自の `ncmod.json` と注入プローブを同梱する）。

このファイルは API の成長に合わせて拡充される。アーキテクチャの背景、Fabric との違い、Mod の書き方については [modding-guide-ja.md](modding-guide-ja.md) を参照。

- アセンブリ: `NetCraft.ModApi.dll`

- 依存関係: `NetCraft`（メインライブラリ）、`NetCraft.Game`

公開サーフェスは3つの名前空間に分かれている:

| 名前空間 | 内容 | 備考 |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | イベントと購読の基底クラス `NcEvent<T>`、`Nc*` ファサード、`Nc*` オブジェクトハンドル | ラッパー層。公開サーフェスにカーネル型はない |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` アノテーション | 拡張ポイント。ルールはカーネルのクラス名とメソッド名に束縛される |
| `NetCraft.ModApi.Internal` | 注入プローブ | 直接参照しないこと |

ルート名前空間 `NetCraft.ModApi` にはエントリクラス `ModApiEntry` のみが含まれる。`Wrapper` と `Extension` は並列する2つのルートであり、選び方については [modding-guide-ja.md 2.9](modding-guide-ja.md#29-2つのルートラッパー層と拡張ポイント) を参照。

***

## 1. クイックスタート

```csharp
using NetCraft.ModApi.Wrapper;

public sealed class MyModEntry
{
    public Task Init()
    {
        //購読はハンドルを返す。Dispose すると登録解除される
        var handle = ServerEvents.Tick.Subscribe(args =>
            Log.Info($"tick {args.TickCount}"));

        ServerEvents.CommandRegister.Subscribe(args =>
            args.Register("mymod", "my mod command", builder =>
                builder.Executes(context =>
                {
                    context.GetSource().SendSuccess("hi");
                    handle.Dispose();          //ハンドルで別のことをしてもよい
                    return 1;
                })));

        return Task.CompletedTask;
    }
}
```

`ncmod.json` では `entry` をこのクラスに向け、`hooks` は空のままにする — 以下のイベントはすべて ModApi 自身のプローブが提供する。

***

## 2. イベント

すべてのイベントは `NetCraft.ModApi.Wrapper` の下にある。`using NetCraft.ModApi.Wrapper;` の後で利用可能になる。

### 2.1 一覧表

| イベント                        | Args型                 | トリガー                                                    | サイド   | ModApi フックポイント                                          |
| ------------------------------- | ---------------------- | --------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `ServerEvents.Tick`             | `ServerTickArgs`       | サーバメインループの毎tick                                  | サーバ | `DedicatedServer::Tick` (Mark)                           |
| `ServerEvents.Started`          | `ServerPhaseArgs`      | メインループ開始後、`Done (x.xxxs)!` が出力された後          | サーバ | `MinecraftServer::Run` (Mark)                            |
| `ServerEvents.Stopping`         | `ServerPhaseArgs`      | サーバがシャットダウンを開始し、プレイヤーがまもなく切断される | サーバ | `DedicatedServer::Stop` (Mark)                    |
| `ServerEvents.CommandRegister`  | `CommandRegisterArgs`  | すべての組み込みコマンドが登録された                        | サーバ | `EffectCommand::Register` 呼び出し箇所（CallSite）           |
| `ServerEvents.PlayerJoin`       | `PlayerJoinArgs`       | 参加パケット列が送信された                                  | サーバ | `PlayerList::PlaceNewPlayer` 呼び出し箇所（CallSite）        |
| `ServerEvents.PlayerLeave`      | `PlayerLeaveArgs`      | プレイヤーがオンラインリストから削除された                   | サーバ | `PlayerList::RemovePlayer` 呼び出し箇所（CallSite）          |
| `ServerEvents.PlayerDisconnect` | `PlayerDisconnectArgs` | 切断パケットが送信され接続が閉じられた                       | サーバ | `ServerPlayer::Disconnect` 呼び出し箇所（CallSite）          |
| `ServerEvents.PlayerHurt`       | `PlayerHurtArgs`       | ダメージが実際に与えられた                                  | サーバ | `PlayerList::HurtPlayer` 呼び出し箇所（CallSite）            |
| `ServerEvents.PlayerDeath`      | `PlayerDeathArgs`      | 体力がゼロにリセットされた直後                              | サーバ | `PlayerList::RespawnPlayer` 呼び出し箇所（CallSite）         |
| `ServerEvents.PlayerChat`       | `PlayerChatArgs`       | チャットのブロードキャスト後                                | サーバ | `ServerGamePacketListenerImpl::HandleChat` 呼び出し箇所（CallSite） |
| `ServerEvents.ChunkLoaded`      | `ChunkLoadedArgs`      | チャンクが初めてメモリに載る                                | サーバ | `ServerChunkCache::set_ChunkLoaded` 代入箇所（CallSite） |
| `ServerEvents.ChunkUnloaded`    | `ChunkUnloadedArgs`    | チャンクがメモリから外れる                                  | サーバ | `ServerChunkCache::set_ChunkUnloaded` 代入箇所（CallSite） |
| `ServerEvents.ChunkSaved`       | `ChunkSavedArgs`       | チャンクがディスクに書き込まれる前にスナップショットが取られる   | サーバ | `ServerChunkCache::set_ChunkSaveSink` 代入箇所（CallSite） |
| `ServerEvents.CommandExecuted`  | `CommandExecutedArgs`  | コマンドの実行が完了した。構文エラーと権限拒否も含む           | サーバ | `CommandManager::Execute` (CallSite) |
| `ServerEvents.LevelTick`        | `LevelTickArgs`        | レベルtick。ロード済みレベルごとに1tick1回                   | サーバ | `PersistentServerLevel::Tick` (CallSite)                 |
| `ServerEvents.SavedDataSaving`  | `SavedDataSavingArgs`  | 保存データがディスクに書き込まれた。チャンク保存より1ステップ後 | サーバ | `SavedDataStorage::ScheduleSave` (CallSite)          |
| `ServerEvents.BlockChanged`     | `BlockChangedArgs`     | ブロック状態が変化し、まもなくクライアントへ同期される         | サーバ | `IBlockUpdateSink::BlockChanged` (CallSite)              |
| `ServerEvents.BlockBroken`      | `BlockBrokenArgs`      | ブロックが破壊された。プレイヤーの採掘とレッドストーンの自壊の両方を含む | サーバ | `ServerBlockUpdates::BreakBlock` (CallSite) |
| `ServerEvents.ItemDropped`      | `ItemDroppedArgs`      | ドロップアイテムエンティティが生成された。ブロック破壊のドロップや調理の産物を含む | サーバ | `ServerBlockUpdates::SpawnDrop` (CallSite) |
| `NetworkEvents.PacketReceived`  | `PacketReceivedArgs`   | ハンドラにキューされたすべての受信パケット。ハンドシェイク段階とステータス段階を含む | 両方   | `PacketProcessor::ScheduleIfPossible` と `HandleNow` (CallSite) |
| `ClientEvents.Tick`             | `ClientTickArgs`       | クライアントメインループの毎tick                            | クライアント | `MinecraftClient::Tick` (Mark)                           |

プレイヤーイベントの順序: 死亡はダメージの流れの中に入れ子になっているため、`PlayerDeath` は対応する `PlayerHurt` より先に発生する。`PlayerLeave` と `PlayerDisconnect` は別物である — 前者はオンラインリストからの削除を意味し（`/kick` の後でも接続が切れた時点で1回だけ発火する）、後者は接続断そのものを意味し、両者がペアで現れる保証はない。

### 2.2 購読と登録解除

```csharp
IDisposable Subscribe(Action<T> handler)
```

- 同じイベントを複数回購読すると、通知は購読順に届く。

- ディスパッチはコールバックリストのスナップショットを取るため、コールバック内から購読・登録解除しても現在のディスパッチには影響しない。

- 登録解除しなければ永遠に有効である。Modにはアンロード機構がないため、手動での登録解除は通常不要である。

### 2.3 Args型

**`ServerTickArgs`**

| プロパティ    | 型   | 備考                                |
| ----------- | ------ | ------------------------------------ |
| `TickCount` | `long` | 今回の起動以降のtick数。1から始まる |

これはカーネルの `TickCount` ではなく ModApi 自身が数える点に注意。

**`ClientTickArgs`**

| プロパティ    | 型   | 備考                           |
| ----------- | ------ | ------------------------------- |
| `TickCount` | `long` | 上と同じだが、クライアント側で独立に数えられる |

**`ServerPhaseArgs`**

| プロパティ | 型     | 備考                        |
| -------- | -------- | ---------------------------- |
| `Phase`  | `string` | 段階名。`started` または `stopping` |

このフィールドはイベント自体と重複するが、ログ出力を単一の統一フォーマットにできるよう残されている。

**`CommandRegisterArgs`**

| メンバー                               | 型                                    | 備考            |
| ------------------------------------ | --------------------------------------- | ---------------- |
| `Dispatcher`                         | `CommandDispatcher<CommandSourceStack>` | カーネルのコマンドディスパッチャ |
| `Register(name, description, build)` | メソッド                                  | コマンドを登録し台帳に記録する。3.1 を参照 |

**`PlayerJoinArgs`**

| プロパティ      | 型           | 備考                                          |
| ------------- | -------------- | ---------------------------------------------- |
| `Player`      | `ServerPlayer` | 参加したばかりのプレイヤー。参加パケットは送信済みで、状態の読み取りは安全 |
| `ProfileName` | `string`       | プレイヤー名                                    |

**`PlayerLeaveArgs`**

| プロパティ  | 型           | 備考                                 |
| --------- | -------------- | ------------------------------------- |
| `Player`  | `ServerPlayer` | 離脱するプレイヤー。この時点でオンラインリストにはもういない |
| `Removed` | `bool`         | 実際に削除されたか。重複削除時は `false` |

**`PlayerDisconnectArgs`**

| プロパティ | 型           | 備考                                 |
| -------- | -------------- | ------------------------------------- |
| `Player` | `ServerPlayer` | 切断されたプレイヤー               |
| `Reason` | `string`       | 切断理由。コンポーネントとして渡された場合はプレーンテキスト |

切断パケットは送信済みで接続は閉じられている。このプレイヤーに今パケットを送っても効果はない。

**`PlayerHurtArgs`**

| プロパティ   | 型            | 備考                                        |
| ---------- | --------------- | -------------------------------------------- |
| `Player`   | `ServerPlayer`  | ダメージを受けたプレイヤー                      |
| `Attacker` | `ServerPlayer?` | ダメージを与えたプレイヤー。環境ダメージやコマンドによるダメージでは `null` |
| `Amount`   | `float`         | 今回のダメージ量                      |

無敵時間中や死亡後には発火しない（カーネルの `Hurt` が `false` を返す）。

**`PlayerDeathArgs`**

| プロパティ   | 型            | 備考                          |
| ---------- | --------------- | ------------------------------ |
| `Player`   | `ServerPlayer`  | 死亡したプレイヤー            |
| `Attacker` | `ServerPlayer?` | キルした者。いない場合は `null` |

カーネルは体力がゼロになった直後にリセットするため、イベント発火時にはプレイヤーはすでにリスポーン地点で全快している。死亡した瞬間の座標やドロップは取得できない。

**`PlayerChatArgs`**

| プロパティ     | 型     | 備考                |
| ------------ | -------- | -------------------- |
| `SenderName` | `string` | 送信者名          |
| `Message`    | `string` | プレーンテキストのメッセージ   |

このイベントは**読み取り専用の通知**である。元のメソッドがすでにメッセージをブロードキャストしているため、ここで `Message` を変更しても効果はない。

**`ChunkLoadedArgs`** **/** **`ChunkUnloadedArgs`** **/** **`ChunkSavedArgs`**

3つのイベントは同じ Args 形状を共有する:

| プロパティ | 型  | 備考              |
| -------- | ----- | ------------------ |
| `X`      | `int` | チャンク座標 X |
| `Z`      | `int` | チャンク座標 Z |

最も間違えやすいのは `ChunkSaved` である: カーネルはこのコールバックが**同期的にスナップショットを取る**ことを要求する一方、シリアライズとディスク書き込みはカーネル自身が非同期で行う。したがってこのコールバックで時間のかかる処理をすると、チャンクのアンロードを直接遅くする。破壊的な操作（ブロックの削除、インベントリの変更）もここに置くべきではない — ここが約束するのはスナップショットの瞬間だけである。

`ChunkUnloaded` が発火する時点で、ブロックエンティティはすでにチャンクとともにクリーンアップされている。ブロックを読みたい場合は `ChunkSaved`（これもブロックにはアクセスできない）か、さらに早い時点を使うこと。

**`CommandExecutedArgs`**

| プロパティ  | 型                   | 備考                                 |
| --------- | ---------------------- | ------------------------------------- |
| `Command` | `string`               | 生のコマンドテキスト。チャットコマンドに先頭のスラッシュは付かない |
| `Result`  | `int`                  | コマンドの戻り値。0は失敗または拒否を意味する |
| `Source`  | `CommandSourceStack?`  | コマンドソース。プレイヤーオーバーロード経路では `null` |
| `Player`  | `ServerPlayer?`        | コマンドを発行したプレイヤー。コンソールから発行された場合は `null` |

このイベントはコマンドの完了**後**に発火し、実行を変更することはできない。構文エラーや権限拒否もここを通る。区別するには `Result` を使う。プレイヤーがチャットバーから送るコマンドは `Execute(ServerPlayer, string)` オーバーロードを通り、カーネルが内部的にコマンドソースを構築するため、その場合は `Source` が `null` で `Player` のみが設定される。

**`LevelTickArgs`**

| プロパティ       | 型                    | 備考                                        |
| -------------- | ----------------------- | -------------------------------------------- |
| `Level`        | `NcLevel`               | このtickで進んだレベル                 |
| `RunsNormally` | `bool`                  | 正常に進んだか。`/tick freeze` 中は `false` |

ロード済みレベルごとに1tick1回発火するため、マルチレベルのワールドでは1tickに複数回届く。トリガー地点はレベルtickが**完了した**後であり、監視点であって割り込み点ではない。

**`SavedDataSavingArgs`**

| プロパティ  | 型                | 備考                       |
| --------- | ------------------- | --------------------------- |
| `Storage` | `SavedDataStorage`  | 永続化される保存データテーブル |

これは `ServerEvents.ChunkSaved` とは別の経路である: チャンク側はスナップショットを取って非同期に書き込むだけだが、こちらは同期書き込みが完了した後に発火する。ワールドクロック、ゲームルール、ワールドボーダーのデータがこれを通る。

**`BlockChangedArgs`**

| プロパティ | 型         | 備考                       |
| -------- | ------------ | --------------------------- |
| `Pos`    | `BlockPos`   | 変化したブロックの位置 |
| `State`  | `BlockState` | 変化後のブロック状態  |

状態はすでにチャンクに書き込まれ、まもなくクライアントへ同期されるため、ここで変化自体を変更することはできない。レッドストーン部品が動作コールバック内で自身の状態を変える場合もこの経路から出る。高頻度なので、コールバックで時間のかかる処理をしてはならない。

**`BlockBrokenArgs`**

| プロパティ | 型            | 備考                                      |
| -------- | --------------- | ------------------------------------------ |
| `Pos`    | `BlockPos`      | 破壊されたブロックの位置                |
| `Player` | `ServerPlayer?` | 破壊した者。レッドストーンなどプレイヤー以外の原因では `null` |

ブロックが実際に置き換えられたときのみ発火する。空の位置や拒否された破壊ではトリガーされない。破壊の効果とドロップはすでに処理済みなので、イベントで読めるのは結果である。

**`ItemDroppedArgs`**

| プロパティ | 型        | 備考                   |
| -------- | ----------- | ----------------------- |
| `Pos`    | `BlockPos`  | ドロップアイテムが出現した場所 |
| `Stack`  | `ItemStack` | ドロップしたアイテムスタック   |

ブロック破壊のドロップと焚き火の調理産物はどちらもここを通る。空のアイテムスタックはエンティティを生成しないため、イベントは発生しない。

**`PacketReceivedArgs`**

| プロパティ        | 型     | 備考                    |
| --------------- | -------- | ------------------------ |
| `Listener`      | `object` | このパケットを受け取るリスナー |
| `Packet`        | `object` | パケットオブジェクトそのもの  |
| `IsServerbound` | `bool`   | サーババウンドのパケットかどうか |

すべての受信パケットで発火し、ハンドシェイク・ステータス・コンフィギュレーション・プレイの4段階すべてをカバーする。移動パケットは1tickに複数回到着するため、コールバックで時間のかかる処理をしてはならない。パケットはすでにオブジェクトへデコードされているがビジネス層には入っていない。型を区別するには自分で `Packet` を調べる。送信パケットはこのイベントの対象外である。

***

## 3. 拡張ポイント

### 3.1 コマンド登録

タイミングは `ServerEvents.CommandRegister` である。このイベントの args をキャッシュしてはならない。内部のコマンドツリーは起動時に一度だけ構築される。

```csharp
public void Register(
    string name,                                          //コマンドリテラル。スラッシュは付けない
    string description,                                   //1行の説明。/ncmapi の台帳に表示される
    Action<LiteralArgumentBuilder<CommandSourceStack>> build);   //引数と実行者を接続する
```

受け取る `build` はカーネルの brigadier ビルダーである。引数、サブコマンド、権限述語はカーネルの流儀で書く:

```csharp
ServerEvents.CommandRegister.Subscribe(args =>
    args.Register("tpall", "teleport all players to the executor", builder =>
        builder
            .Requires(s => s.HasPermission(2))            //権限述語
            .Executes(context => { /* ... */ return 1; })));
```

引数付きの場合:

```csharp
args.Register("heal", "heal the target players", builder =>
    builder.Requires(s => s.HasPermission(2))
        .Then(RequiredArgumentBuilder<CommandSourceStack, EntitySelector>
            .Argument("targets", EntityArgument.Players())
            .Executes(context =>
            {
                foreach (var player in EntityArgument.GetPlayers(context, "targets"))
                    player.Heal(20f);                     //例示用
                return 1;
            })));
```

要点:

- `args.Dispatcher.Register(...)` を直接呼んでもコマンドは設置されるが、台帳には入らず `/ncmapi` には表示されない。一覧に載せたい場合は `args.Register` を使う。

- コマンドにはデフォルトで権限制限がない。必要なら自分で `.Requires(...)` を追加する。

- 実行時の挙動は完全に自分次第であり、ModApi は割り込まない。

### 3.2 登録済みコマンドの表示

組み込みの `/ncmapi` があり、権限レベル2を要求する:

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

使用方法の行はコマンドツリーのノード構造からオンザフライで計算される: リテラルは名前で書かれ、引数は山括弧で囲まれ、それ自体が実行可能な中間ノードも独自の行を持つ。

***

## 4. サーバファサード

この章のファサードはすべて `NetCraft.ModApi.Wrapper` の下にある。`using NetCraft.ModApi.Wrapper;` の後で利用可能になる。

ファサードは `Nc*` の静的クラスで、カーネル全体に散らばる機能を少数のエントリポイントに集約する。カーネルインスタンスはメインループ開始時にプローブが捕捉する。`NcServer.IsAvailable` が false のとき、以下はすべて例外を投げる — これらはイベントコールバック内でのみ使い、`Init` からは使わないこと。

| ファサード | 用途 |
| --- | --- |
| `NcServer` | サーバインスタンス、tickレート、コマンド、エンティティ追跡、プレイヤーデータ、ゲームルール、ブロードキャスト、コマンド実行 |
| `NcPlayers` | オンラインプレイヤーの照会と操作（キック、テレポート、体力、ゲームモード、権限） |
| `NcWorld` | オーバーワールドのブロック読み書きと破壊、天気、時間、ボーダー、クロック、サウンド、レベルイベント。他のディメンションへは `NcLevel` ハンドルを取って到達し、座標は単なる `x y z` の int |
| `NcRegistries` | 名前で引く組み込みレジストリ（ブロック、アイテム、流体、エフェクト、バイオーム、パーティクル、エンティティ、ブロックエンティティ） |
| `NcRecipes` | レシピの照会（グリッドクラフト、石切り、調理。idでレシピを取得） |
| `NcLists` | リストと設定（ホワイトリスト、ops、バン、`server.properties`） |
| `NcStartup` | 起動引数（カーネルが認識しないトークンと名前ベースの購読） |

`NcPlayer` は静的ファサードではなく**オブジェクトハンドル**である: `NcPlayers.All` / `Find` がこれを返し、プレイヤーイベントの `Player` / `Attacker` もこれである。ハンドルは読み取り専用でプローブが構築する。Modはカーネルの `ServerPlayer` を取得できない — これが「公開サーフェスにカーネル型を出さない」の第一のアンカーである。同じカーネルのプレイヤーは常に同じハンドルに対応し、内部的に弱参照でキャッシュされ、プレイヤーがログオフすると自動的に無効化される。

`NcLevel` はレベルについて同じ形をとる。`NcWorld.Overworld` / `Nether` / `End` と `NcWorld.Get("minecraft:the_nether")` がこれを返し、`LevelTickArgs.Level` もこれである。ディメンションid、時間、天気、建築高さ、tick数、チャンクの強制ロードを保持する。ブロック操作は `NcWorld` に残り、ハンドルと `x y z` を取る。`BlockPos` は決して現れないため、Modのdllはカーネルのレベル型への参照を持たない。

### 4.1 レジストリ

`NcRegistries` はテーブル全体と名前による検索の両方を提供する。テーブル全体は反復とタグベースの検索用、名前による検索は単一要素の取得用である:

```csharp
var stone = NcRegistries.FindState("minecraft:stone");     //ブロックのデフォルト状態
var diamond = NcRegistries.FindItem("minecraft:diamond");
var over = NcRegistries.FindBiome("minecraft:plains");

//テーブル全体を反復する
foreach (var id in NcRegistries.Blocks.KeySet)
    Log.Info(id.ToString());
```

レジストリは起動中に徐々に組み立てられ、Modは組み立て完了前にロードされる。そのため `Init` で引いたものをキャッシュしてはならない — 組み立てはまだ進行中で、キャッシュした値は null 参照か古い値になる。現在 `BuiltInRegistries.BootStrap` はまだ空の実装である。各レジストリはそれぞれの Bootstrap が個別に埋め、データ駆動型のもの（バイオーム、レシピなど）はデータパックの読み込みが配線されるまでエントリがごく少ない。

### 4.2 レシピ

`NcRecipes` はデータパックから読み込まれたレシピテーブルに支えられている。`/reload` はテーブル全体を差し替えるため、リロードをまたいで `RecipeHolder` を保持してはならない。

```csharp
if (NcRecipes.IsAvailable)
{
    var result = NcRecipes.Craft(input);                    //クラフトグリッドの出力を計算する
    var recipes = NcRecipes.StonecuttingFor(stack);         //この入力で利用可能な石切りレシピ
    var smelting = NcRecipes.CookingFor("smelting", stack); //調理タイプで検索する
    var byId = NcRecipes.Find("minecraft:oak_planks");      //idでレシピを取得する
}
```

***

## 5. 内部実装

Modを書くのにこの節は必要ないが、デバッグ時に役立つことがある。

### 5.1 プローブ

| クラス                                                    | 形式        | 役割                                              |
| -------------------------------------------------------- | ----------- | ----------------------------------------------------------- |
| `Internal.SignalProbe.OnSignal(string)`                  | Mark ×4     | 「何かが起きた」というシグナルはすべて1つのメソッドに集まり、`label` によって対応するイベントへディスパッチされる |
| `Internal.CommandProbe.OnCommandsReady(object)`          | CallSite    | `EffectCommand::Register` への呼び出しを置換する。元の呼び出しを復元した後で `CommandRegister` を発火する |
| `Internal.PlayerProbe.OnXxx(object, ...)`                | CallSite ×6 | プレイヤーイベント。フックポイントごとに1メソッド。元の呼び出しを復元した後でイベントを発行する |
| `Internal.LevelProbe.OnChunkXxxAssigned(object, object)` | CallSite ×3 | チャンクイベント。`ServerChunkCache` の3つのコールバックプロパティの代入箇所でフックする。カーネルに制御を戻す前にラッパーデリゲートを1層重ねる |
| `Internal.BlockProbe.OnXxx(...)`                         | CallSite ×3 | ブロックイベント。破壊とドロップは `ServerBlockUpdates` を、状態変化は `IBlockUpdateSink` のインターフェースメソッドをフックする |

`SignalProbe` のシグネチャは `string` のみを取り、`CommandProbe`、`PlayerProbe`、`LevelProbe` のパラメータは `object` として宣言されている — これは意図的である: アセンブリ時に `Lead.Hook` は置換メソッドのシグネチャを解決し、そこにカーネル型が現れると、その解決がカーネルアセンブリを早期に引き上げて注入がウィンドウを逃す。カーネル型はメソッド本体の内部にのみ現れ、その時点でコードはすでに実行されている。

シグネチャで `object` にできないのは値型のパラメータと戻り値だけである: `object` はスタック上の参照だが `float`/`bool` は値であり、不一致は不正な IL になる。そのため `PlayerProbe.OnHurtPlayer` はダメージ量に `float` を保ち、`OnRemovePlayer` と `OnHurtPlayer` は `bool` の戻り値を保つ。

`BlockProbe` はこの制約の延長である: ブロックの位置と状態は `BlockPos`/`BlockState` という2つの値型で、シグネチャにはそのままの型でしか書けない。これら2つの型は `NetCraft.Primitives` と `NetCraft.Registry` に由来し、どちらも注入リストに載っていないため、アセンブリ時にこれらを解決しても書き換え対象のアセンブリを早期に引き上げることはない。

### 5.2 フックポイント一覧

ModApiの `ncmod.json` には24個のルールが含まれ、2.1 の表と1対1で対応する。フックポイントを変更したりルールを追加したりするにはこのファイルを編集する。編集後は再ビルドし（これは埋め込みリソースである）、生成された dll を `mods/` に戻す — 後者は `NetCraft.ModApi.csproj` の `DeployModToHosts` がすでに自動で行う。これを怠るとルールが全く効かない形で現れる。

`CommandManager::Execute` には2つのオーバーロードがあり、1つのルールを共有する。`Lead.Hook` の CallSite は呼び出し箇所を「型 + メソッド名」で照合し、パラメータリストでは照合しない。両オーバーロードは2つのパラメータを取るため、プローブは最初のパラメータに `object` を取り、実際の型でディスパッチできる。

2つの `PacketProcessor` ルールは相補的である: プレイ段階のパケットは `ScheduleIfPossible` を通ってメインスレッドのキューに入り、ハンドシェイク段階とステータス段階は `HandleNow` を通って即時処理される。どのパケットもそのうち片方しか通らない。前者だけをフックするとハンドシェイク段階とステータス段階を取り逃す — これらはたまたまスクリプトでのプローブが最も容易な段階であるため、デバッグ中には「ルールが効かなかった」と誤読しやすい。

3つのチャンクルールは `ServerChunkCache` の3つのコールバックプロパティの**代入箇所**をフックし、読み取り箇所ではない。理由は、これら3つのプロパティがユニキャストであり、`PersistentServerLevel` の構築時にすでにカーネル自身に占有されているためである（カーネルは保存とブロックエンティティのクリーンアップのロジックを注入している）。Modが直接代入するとカーネルのコピーを上書きしてしまい — アンロードが永続化されず、ブロックエンティティがクリーンアップされず、しかもエラーが全く出ない。代入箇所をフックすれば、その時点でカーネルのコールバックとプローブを連鎖させられる。代入は一度しか起きず、以降の各トリガーでデリゲート転送が1層ずつ増える。

`PlayerList::RespawnPlayer` は private であるため、プローブは元の呼び出しを復元できない。これはリフレクションを通す（死亡ごとに1回呼ばれるだけなのでオーバーヘッドは無視できる）。これはまたカーネル側の対応への余地も残す: 将来 `InternalsVisibleTo` が追加されれば、直接呼び出しに切り替えられる。

### 5.3 台帳

`Internal.NcCommandRegistry` は `args.Register` を通じて登録されたコマンドを記録する。これは台帳にすぎず、コマンド実行には関与しない。コマンド自体はカーネルのディスパッチャに設置されるため、台帳に問題があってもコマンドは機能する。

***

## 6. 今後追加予定

以下は場所は確認済みだがまだイベント化されていないフックポイントである（このリストはリポジトリルートの `__scan_mod_api.py` により生成された）:

| 方向         | 候補フックポイント                                                        |
| ----------------- | ---------------------------------------------------------------------------- |
| エンティティ          | `ClientLevel::AddEntity`、`Entity::Die`                                      |
| ワールド             | レベルのロードとアンロード、`ServerChunkCache` のチャンクバッチ処理                     |
| 地形生成 | `ChunkStatus` ごとの `ChunkGenerator::Generate` 段階、`WorldGenRegion::SetBlockState` |
| コマンド実行 | `CommandSourceStack::SendSuccess` / `SendFailure`（応答側で呼び出し箇所が多い） |
| ネットワーク           | パケット種別ごとの `ServerGamePacketListenerImpl::HandleXxx`（現在は統合エントリポイントのみ） |

すでに完了した方向: レベルtickは `ServerEvents.LevelTick` になり、保存データの永続化は `ServerEvents.SavedDataSaving` になり、コマンド実行は `ServerEvents.CommandExecuted` になり、ブロックは `ServerEvents.BlockChanged` / `BlockBroken` / `ItemDropped` になった。

ブロックのセルを `SetBlock` でフックするのはうまくいかない: `notifyNeighbors` と `strict` という2つのデフォルトパラメータがあるため、コンパイル済みの呼び出し箇所は4から6個のパラメータを取り、CallSite はパラメータリストを見ずに「型 + メソッド名」で照合するため、1つの置換メソッドでは3つのスタック形状すべてを扱えない。代わりに2箇所をフックする: 状態同期は `IBlockUpdateSink::BlockChanged`（`ServerLevel.SetBlock` 内で唯一のインターフェース呼び出しで、クライアント同期を伴うすべての変化をカバーする）をフックし、破壊とドロップは `ServerBlockUpdates` 自身のメソッドをフックする。

エンティティのセルは厄介である。`Entity` は `NetCraft.Registry` で定義されており、これは注入リストに載っていないため、これを対象とする呼び出し箇所は書き換えられない。`ClientLevel::AddEntity` はクライアントアセンブリにあり可能である。死亡イベントにはまず `Registry` を書き換えられるかの決着が必要である。

ネットワークのセルについては、`HandleChat` は以前から存在し、`NetworkEvents.PacketReceived` が統合エントリポイントを提供しているため、`HandleXxx` ごとのフックの価値はかなり低い。パケット種別による細かいフィルタリングが必要なシナリオのみ追加する価値がある。

レベルのロードとアンロードはNC側に収束点がない: `DedicatedServer::CreateLevel` は private であるため、元の呼び出しは `PlayerList::RespawnPlayer` と同様にリフレクションでしか復元できない。アンロード経路はさらに散らばっている。やるならまずイベントの args をどうするか決める必要がある。

残りはすべてオブジェクト参照（エンティティインスタンスなど）を必要とするため、`Mark` ではなく `CallSite` を使う必要がある。パラメータにカーネルの値型が現れる場合は、置換メソッドのシグネチャにそのままの型でしか書けない。

インターフェースサーフェスのリスト（`__scan_mod_api.py --api` が生成する `__modapi_api.txt`）も検討した: 機能エントリポイントとして適格な項目はドメインごとに第4章のファサードへまとめ、公開されていない残りは3つのカテゴリに分かれる — プロトコルとパケット処理（`Network.Protocol.*`）、レンダリングとモデル（`Client.Render.*`）、地形生成と密度関数（`LevelGen.*`）である。これらはカーネルの内部であり、直接使うとModを実装詳細に縛り付けるため、まずカーネル側で安定したインターフェースを開放すべきである。

サーバ側にはファサード化されていないものがさらに2つある: `ReloadableServerResources` インスタンスは `DedicatedServer` にぶら下がっており、ModApi は `NetCraft.Server` を参照していないため、これをラップするにはまずカーネルの基底クラスにプロパティを開く必要がある。`ChunkSender` と `ServerWorldBorderListener` は内部フローであり、Modの用途はない。
