# NetCraft Modテンプレート

NetCraft Mod開発用のサンプルコード。`NetCraftTemplate.yaml` の各エントリは
`examples/` 配下の1ファイルを指し、`ncm template` がオンデマンドで取得する。

## 使い方

```
ncm template view              すべてのエントリを一覧表示
ncm template view Wrapper.?    idでフィルタ、? と * はワイルドカード
ncm template example <api id>  1つのサンプルファイルをカレントディレクトリに取得
```

## 2つのルート

`NetCraft.ModApi` は2つの名前空間を公開する。用途に合う方を選ぶ:

| 名前空間 | 得られるもの |
| --- | --- |
| `NetCraft.ModApi.Wrapper` | イベント、`Nc*` ファサード、`Nc*` ハンドル。公開サーフェスにカーネル型が現れないため、カーネルのリネームでModの再ビルドを強いられない。 |
| `NetCraft.ModApi.Extension` | `[Inject]` と `[Mixin]` 属性。ルールがカーネルの型とメソッドを直接指すため、より強力で、より壊れやすい。 |

## よく使う呼び出し

Modプロジェクト内からこのパネルを開くと、以下の各呼び出しのうちコードが
実際に使用しているものが `NetCraftTemplate.yaml` と照合され評価される:
メンバーが宣言されていれば緑、メンバーがなければ黄、型自体が宣言されて
いなければ赤。ハイライトされた名前をホバーすると理由が表示される。

| 呼び出し | 動作 |
| --- | --- |
| `NcServer.IsAvailable` | サーバが起動し捕捉されているか |
| `NcServer.Broadcast` | オンライン全員へのシステムメッセージ |
| `NcServer.Execute` | コンソールとしてコマンドを実行 |
| `NcWorld.GetBlock` | ブロックを1つ読み取る。チャンク未ロード時はnull |
| `NcWorld.SetBlock` | ブロックを1つ書き込む。更新チェーン全体を実行 |
| `NcWorld.BreakBlock` | プレイヤーと同様にブロックを破壊 |
| `NcPlayer.Name` | プレイヤー名 |
| `NcPlayer.Health` | 現在の体力 |
| `NcPlayers.Find` | 名前でオンラインプレイヤーを検索 |
| `NcPlayers.Send` | 個人向けシステムメッセージ |
| `NcRegistries.FindState` | 名前空間付きidでブロック状態を取得 |
| `NcRegistries.FindItem` | 名前空間付きidでアイテムを取得 |
| `ServerEvents.Tick` | サーバtickごとに実行 |
| `ServerEvents.PlayerJoin` | プレイヤーの参加が完了 |
| `ServerEvents.BlockBroken` | ブロックが実際に置き換えられた |
