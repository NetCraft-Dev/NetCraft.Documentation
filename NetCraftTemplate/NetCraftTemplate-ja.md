# NetCraft mod テンプレート

NetCraft mod 開発用のサンプルコード。`NetCraftTemplate.yaml` の各エントリは `examples/` 配下の 1 ファイルを指し、`ncm template` が必要に応じて取得します。

## 使用方法

```
ncm template view              すべてのエントリを一覧表示
ncm template view Wrapper.?    id でフィルタ、? と * はワイルドカード
ncm template example <api id>  1 つのサンプルファイルを現在のディレクトリに取得
```

## 2 つのルート

`NetCraft.ModApi` は 2 つの名前空間を公開します。合う方を選んでください：

| 名前空間 | 得られるもの |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | イベント、`Nc*` ファサード、`Nc*` ハンドル。公開サーフェスにカーネル型は一切現れないため、カーネルの名前変更で mod の再ビルドが強制されることはありません。 |
| `NetCraft.ModApi.Extension` | `[Inject]` と `[Mixin]` 属性。ルールはカーネル型とメソッドを直接指定するため、より強力で、より壊れやすいです。 |

## ハンドル

`NcPlayer` と `NcLevel` は読み取り専用ハンドルです。それらが返すものはすべて通常の文字列、数値、真偽値（`NcLevel.Dimension`、`NcPlayer.X`）であり、カーネル型ではありません。そのため、カーネルの名前変更で mod の再ビルドが強制されることはありません。ブロック操作はレベルハンドルと `x y z` を受け取ります。

## よく使う呼び出し

mod プロジェクト内からこのパネルを開くと、以下の呼び出しのうちコードが実際に使用しているものが `NetCraftTemplate.yaml` に対して判定されます：メンバーが宣言されていれば緑、メンバーが存在しなければ黄、型がまったく宣言されていなければ赤。ハイライトされた名前をホバーすると理由を確認できます。

| 呼び出し | 動作 |
| --- | --- |
| `NcServer.IsAvailable` | サーバーが起動して捕捉されているか |
| `NcServer.Broadcast` | オンライン全員へのシステムメッセージ |
| `NcServer.Execute` | コンソールとしてコマンドを実行 |
| `NcWorld.GetBlock` | 1 つのブロックを読み取る。チャンクが未読み込みなら null |
| `NcWorld.SetBlock` | 1 つのブロックを書き込む。完全な更新チェーンを実行 |
| `NcWorld.BreakBlock` | プレイヤーがするようにブロックを破壊 |
| `NcWorld.Overworld` | オーバーワールドのレベルハンドル |
| `NcLevel.Dimension` | レベルハンドルの次元 id。例：minecraft:overworld |
| `NcLevel.DayTime` | 1 つの次元の時刻を読み取りまたは設定 |
| `NcPlayer.Name` | プレイヤー名 |
| `NcPlayer.Health` | 現在の体力 |
| `NcPlayers.Find` | 名前でオンラインプレイヤーを検索 |
| `NcPlayers.Send` | プライベートシステムメッセージ |
| `NcRegistries.FindState` | 名前空間 id でブロック状態を取得 |
| `NcRegistries.FindItem` | 名前空間 id でアイテムを取得 |
| `ServerEvents.Tick` | サーバー tick ごとに実行 |
| `ServerEvents.PlayerJoin` | プレイヤーの参加が完了 |
| `ServerEvents.BlockBroken` | ブロックが実際に置き換えられた |