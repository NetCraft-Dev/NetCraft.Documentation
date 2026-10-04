# NetCraft 改造ガイド


個々のAPIの詳細はここにはない。[modapi-ja.md](modapi-ja.md) を参照。

---

## 1. まずは実行時の構造

### 1.1 3つのプロセスエントリポイント

NCには3つのエントリポイントがあり、Mod注入パイプラインは3つとも同じである:

| エントリ | 用途 |
| --- | --- |
| `NetCraft.Loader` | 両サイド兼用の1つのexe。`--server` でサーバを起動し、`--client` かモードフラグなしでクライアントを起動する |
| `NetCraft.Server.Exe` | 単体のサーバ実行ファイル |
| `NetCraft.Client.Exe` | 単体のクライアント実行ファイル |

`Main` 自体は薄い殻で、コールバックを登録し、次のメソッドに処理を渡すだけである。`NetCraft.Server.Exe` を例に取ると:

```csharp
public static int Main(string[] args)
{
    EmbeddedAssemblyLoader.Initialize();   // カーネルアセンブリ解決コールバックを登録する
    BootMods(args);                        // Modのブートストラップを実行する
    return Launch(args);                   // ここで初めて業務実装に入る
}
```

この分離はスタイルの問題ではない。JITがメソッドをコンパイルするとき、そのメソッド本体に現れる**すべての型**を解決し、これはメソッドの実行前に起こる。もし `Main` が `ServerMain.Run(args)` を直接呼んでいたら、`NetCraft.Server.dll` は `Main` がJITコンパイルされた瞬間、Modブートストラップが走る前に引き上げられ、書き換えウィンドウは失われていた。そのため `BootMods` と `Launch` の両方に `MethodImplOptions.NoInlining` を付けなければならない — これがないとJITがそれらを `Main` にインライン展開してしまい、分割が無意味になる。

`NetCraft.Loader` も同じ構造だが、モード判定とModブートストラップがどちらも `Launch` にあり、`Main` は `Initialize` と `Launch` の2ステップのみを保つ。

### 1.2 kernel/ サブディレクトリにあるカーネルアセンブリ

ビルド後の出力ディレクトリは次のようになる:

```
NetCraft.Server.Exe.exe
NetCraft.dll              <- メインライブラリ。下位のサブライブラリをすべて埋め込む
NetCraft.ModLoader.dll    <- ローダー自身
NetCraft.Server.Exe.dll   <- エントリアセンブリ
kernel/
  NetCraft.Game.dll
  NetCraft.Server.dll
  ... 残りのカーネルアセンブリ
mods/
  your-mod.dll
```

なぜルートに置かず `kernel/` に移すのか？

.NETホストは `deps.json` に登録されたアセンブリをTPA（Trusted Platform Assemblies）として扱う。TPA内のアセンブリについて、ランタイムは**パスで**解決する — `AssemblyLoadContext.LoadFromStream` に渡されたバイト列は単に無視される。つまり、事前に書き換えたバイト列を渡しても、ランタイムはディスクから未書き換えのコピーを読んでしまう。カーネルアセンブリを `deps.json` から外し、ファイルを他所へ移して初めて、ランタイムは解決失敗時に `AssemblyLoadContext.Resolving` へコールバックし、書き換え済みのバイト列を渡す機会が得られる。

ルートに残る3種は移せない: メインライブラリ（埋め込みホストであり最初に起動する必要がある）、ローダー自身（ブートストラップコードがその中にある）、エントリアセンブリ（apphostがそこから起動する）。

**コスト**: エントリアセンブリ自体には注入できない。フック対象がたまたま `NetCraft.Server.Exe.dll` アセンブリにある場合、それは効かない。カーネルの業務コードはすべて `kernel/` の下にあるため、通常これは問題にならない。

### 1.3 Modの読み込み順序

```
EmbeddedAssemblyLoader.Initialize()
  └─ Resolving コールバックをインストールする
BootMods → ModBootstrap.Run(current side)
  ├─ mods/*.dll を静的にスキャンする（MetadataReader が埋め込み ncmod.json を読み、アセンブリはロードしない）
  ├─ environment が現在のサイドと一致しないModを除外する
  ├─ 注入ルールを組み立て、リライタをメインライブラリに渡す
  ├─ 対象アセンブリをプリロードする: バイト列を読む → リライタを通す → LoadFromStream
  └─ 各Modエントリの Init() を呼ぶ
Launch → ServerMain/ClientMain.Run(args)
  └─ カーネルの業務が動き始める。プローブはすでに内側にある
```

順序に注意: **宣言がスキャンされ、書き換え済みのバイト列がロードされ、エントリコードが最後に走る**。Modの `Init()` が実行される時点で、カーネルアセンブリはすでに置き換えられている。

---

## 2. Fabricとの主な違い

| 観点 | Fabric | NetCraft |
| --- | --- | --- |
| 言語 / ランタイム | Java / JVM | C# / .NET 10 (CoreCLR) |
| Modの入れ物 | `fabric.mod.json` を含む jar | `ncmod.json` を埋め込んだ dll |
| 宣言の読み取り | jar内のファイルを読む | `MetadataReader` がアセンブリをロードせずに埋め込みリソースを静的に読む |
| コード注入 | Mixin（ソース内のアノテーション。クラスロード時にメンバーが対象クラスに混ぜ込まれる） | `Lead.Hook`（マニフェストかアノテーションでルールを宣言。アセンブリ解決時にバイト列がその場で書き換えられる） |
| 注入の粒度 | メソッド本体内の任意の行（ローカルや中間の式の値を含む） | 13の形式（呼び出し箇所、フィールド読み書き、コンストラクタ、型チェック、ボックス化、ローカル変数、定数、メソッド本体全体の置換、プローブなど）で、前に挿入または後に挿入が可能 |
| ロードモデル | Fabric Loader + Knot クラスローダ | 単一のデフォルトALC + `AssemblyLoadContext.Resolving` |
| 公式APIの範囲 | Fabric API は非常に多くのモジュールを持つ | NetCraft-ModApi は現在、イベントとコマンドの拡張ポイントしかない |

### 2.1 注入スタイル：アノテーションかマニフェストか、どちらかを選ぶ

FabricのMixinは**ソース内**にアノテーションを書く:

```java
@Inject(method = "tick", at = @At("HEAD"))
private void onTick(CallbackInfo ci) { ... }
```

NCは両スタイルをサポートするが、前提条件が異なる: アノテーション形式は `NetCraft.ModApi.Extension` の `InjectAttribute` に依存するため、これを参照しないModはアノテーションを使えない。マニフェスト形式は `ncmod.json` に書く純粋なデータで、注入ルールに参照は不要である。

**アノテーション**は、自分の置換メソッドに付ける:

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick),
        HookType = "Mark", Label = "server_tick", Environment = "server")]
public static void OnTick(object self) { ... }
```

**マニフェスト**は、`ncmod.json` の `hooks` に書く:

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

アセンブリ時、2つのルートは1つのルールテーブルに統合され、**同じ注入ポイントを両方に書いた場合はアノテーションが優先される**。統合後は両者を区別できない。違いは前提条件と使い勝手にある:

| | アノテーション | マニフェスト |
| --- | --- | --- |
| 書く場所 | 置換メソッド上 | `ncmod.json` の `hooks` 内 |
| 前提条件 | `NetCraft.ModApi` を参照する必要がある | なし。純粋なデータ |
| 型名 | `typeof` / `nameof` でコンパイラがチェックする | 手書きの文字列。タイポはアセンブリ時にしか見つからない |
| 持てる内容 | 注入ルールのみ | Modの識別情報（id、entry、environment、表示情報）と注入ルール |

したがって、アノテーションを使うかどうかに関わらず `ncmod.json` を書かなければならない。これがModの識別情報の唯一の源である。アノテーションはルールを書き間違えにくくするだけである。マニフェストに依存関係フィールドはない。依存関係はアセンブリ参照から推論され（6.1 参照）、宣言は不要である。

逆に、**`NetCraft.ModApi` を参照しないModはマニフェストしか使えない** — これは注入ルール以上に影響する: `ServerEvents` のようなイベントや `Nc*` ファサードもModApiの中にあり（`NetCraft.ModApi.Wrapper` の下、[3.3](#33-並べて比較する例) 参照）、アノテーションを使えないModはこれらも使えない。

Fabricとの違いは次のとおり:

- **変更の仕方とタイミング**: Mixin はトランスフォーマが**クラスロード時**に**mixinクラスのメンバーを**対象クラスに**混ぜ込み**、ロードされるのは合成された新しいクラスで元のものはもう存在しない。NCは**アセンブリがメモリに入る前**に対象メソッドの命令をその場で書き換えるため、クラスは同じクラスのままでメソッド本体だけが変わる。どちらもロード時に書き換え、どちらもコンパイル時にバイトコードを変更しない — Mixinのアノテーションプロセッサはrefmap（難読化マッピング）を生成してビルド時に検証を行うだけであり、NCは難読化されておらず、その層自体が存在しない。
- Mixinは**メソッド本体の途中の任意の位置**に注入できる。NCは指定したホストメソッド内の特定の呼び出し箇所、フィールドアクセス、構築、ローカル変数の読み書き、定数を対象にでき、その前後に挿入できる（`InType`/`InMethod` がスコープを絞り、`Placement` が挿入か置換を決める）が、**任意の行番号には到達できず**、ジャンプ先やスタック上の中間式の値は変更できない。
- Mixinの対象はメソッド名の文字列とディスクリプタを使う。NCは「完全な型名 + メソッド名」を使うため、同名のオーバーロードはすべてマッチし、1つに絞るには `InType`/`InMethod` が必要である。

**どの層がアノテーションを扱うか**: アノテーション型（`InjectAttribute`）は `NetCraft.ModApi.Extension` が提供し、それを解決するのは `NetCraft.ModLoader` である — Modのスキャン時に `MetadataReader` で `CustomAttribute` テーブルを静的に読み、アセンブリはロードしない。**`Lead.Hook` はアノテーションを認識しない**。見るのは統合後のルールテーブルだけであり、ネイティブの注入層はマネージド側でコンパイルされた記述バイト列しか認識せず、`ncmod.json` すら読まない。

これはアノテーションで表現できるものを決める: 何を書けるかは `InjectAttribute` がどのフィールドを持つかに完全に依存する。現在7つある — 対象型、メソッド名、`HookType`、`Label`、`Environment`、`PatchMode`、`Ordinal` — そして [2.4](#24-1箇所への絞り込みホストのスコープと配置) の `InType`/`InMethod`/`Placement` と [2.5](#25-メソッド本体内のアンカーローカル変数と定数) の `LocalIndex`/`ConstantValue` は**アノテーションには書けない**。C# APIを使うか、マニフェストが追いつくのを待つこと。マニフェスト側もこれらが欠けており、アノテーションより多く受け付けるのは `ordinal` だけである。

13の注入形式については、[改造ガイドの付録](#付録hooktype-の概要) と [modapi-ja.md](modapi-ja.md) を参照。

### 2.2 重要な制約：プローブクラスはシグネチャにカーネル型を含めてはならない

NCの書き換えはカーネルアセンブリがロードされる**前に**起こる。ルールを組み立てるとき、`Lead.Hook` はリフレクションを使って置換メソッドを見つけメソッド参照を構築し、この過程でシグネチャ内のすべてのパラメータ型と戻り値型を解決する。

したがって: **置換メソッドのシグネチャにはBCL型と `object` しか使えない**。シグネチャに `NetCraft.*` 型が現れると、その解決がカーネルアセンブリを早期に引き上げ、注入は即座に失敗する。

カーネルのオブジェクトが必要なときは、パラメータを `object` として宣言し、メソッド本体内でキャストする:

```csharp
//アセンブリは object しか見ない。メソッド本体はカーネル起動後にJITコンパイルされる
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    ...
}
```

### 2.3 CallSite は挿入ではなく置換

`CallSite` ルールの置換メソッドは元の呼び出しを**置換する**ため、元のメソッドは実行されない。元の挙動を保つには、置換メソッド内で自分で復元しなければならない:

```csharp
public static void OnCommandsReady(object dispatcher)
{
    var typed = (CommandDispatcher<CommandSourceStack>)dispatcher;
    EffectCommand.Register(typed);              //置換した呼び出しを復元する
    ServerEvents.CommandRegister.Publish(...);  //その後でMod自身のロジックを追加する
}
```

このステップを怠ると、元の機能は完全に消える。

これを実施する上での細かい点:

- **インスタンスメソッドの `this` もパラメータとして数える**。対象メソッドがインスタンスメソッドかどうかが、置換メソッドに先頭の追加パラメータが必要かを決める。ILでは `call` も `callvirt` もインスタンス呼び出しとして数える — `sealed` 型の非仮想メソッドではコンパイラは `call` を出力する。
- **1つのルールキーはそのクラス名の下のすべてのオーバーロードをカバーする**。同じパラメータ数のオーバーロードは1つの置換メソッドを共有する。`Disconnect(string)` と `Disconnect(Component)` はこのように共有し、置換メソッドは実際の引数型でディスパッチする。
- **private メソッドは外部から呼べない**ため、置換メソッドは元の呼び出しを復元できない。このフックポイントを諦めるか、リフレクションで1回呼ぶ（低頻度なら許容できる）。
- **値型のパラメータと戻り値は `object` として宣言できない**。`object` はスタック上の参照だが `float`/`bool` は値であり、不一致は不正な IL になる。これら2つの位置は本来の型のままにする。
- **ユニキャストのコールバックは直接代入できない**。一部のカーネルコールバックプロパティ（例: `ServerChunkCache` の3つのチャンクコールバック）は `event` ではなく `Action<T>` で、カーネルがすでに占有している。Modが直接代入するとカーネルのコピーを上書きし、エラーは全く出ない。正しい方法は、そのプロパティの setter をフックし、代入の瞬間に自分のロジックとカーネルのコールバックを1つのラッパーデリゲートに連鎖させることである。

### 2.4 1箇所への絞り込み：ホストのスコープと配置

命令レベルのルールのデフォルトスコープは**アセンブリ全体**である — 対象メソッドを呼ぶ、または対象フィールドを読み書きするすべての箇所がマッチする。1箇所に絞るには2つの省略可能なパラメータを使う:

| パラメータ | 効果 |
| --- | --- |
| `InType` / `InMethod` | 指定したホストメソッド本体の内部でのみアンカーを照合する。両方空なら無制限 |
| `Placement` | `Replace` はアンカーを置換する（デフォルト）。`Before` / `After` はアンカーを保ち、その前後に呼び出しを1つ挿入する |
| `Ordinal` | 同じアンカーがホストメソッド内の複数箇所にマッチするとき、どれを選ぶか（0始まり）。省略するとすべての箇所が変更される |

```csharp
//例: LevelChunk がブロック状態を読むときのみ計測する。他所の PalettedContainer::Get はそのまま
new HookRule("NetCraft.Storage.PalettedContainer", "Get", typeof(MyProbe), nameof(MyProbe.OnGet),
    HookType.CallSite, PatchMode.ILRewrite,
    inType: "NetCraft.Storage.LevelChunk", inMethod: "GetBlockState",
    placement: HookPlacement.Before)
```

2つのモードはコールバックシグネチャに異なる要件を課す:

- **置換モード**は置換する呼び出しの引数（インスタンス呼び出しでは `this` を含む）に合わせる。コールバックが元の呼び出しを復元するかは自分次第である。
- **挿入モード**は**ホストメソッドのパラメータ**（`this` を含む）を渡す。`MethodBody` の慣例と一致する。挿入はアンカーがすでに構築したスタックを乱さない。元の呼び出しは通常どおり実行され、その前後にコールバックが1つ増えるだけである。

いくつかの境界:

- `InType` と `InMethod` は独立しており、片方だけ指定してもよい。両方空はスコープなしと等価である。
- 複数のルールが同じアンカーをフックでき、それぞれ別のホストにスコープされる。**最初にホストへマッチしたものが勝つ**。
- `Placement` は命令レベルの形式にのみ適用される（`CallSite`、`NewObj`、フィールド読み書き、`TypeCheck`、`Box`、`FunctionPointer`、および [2.5](#25-メソッド本体内のアンカーローカル変数と定数) の3種）。`MethodBody` は常に全体を置換する。
- `Ordinal` は**マッチの順序**を数え、その箇所が最終的に変更されるかは問わない。ルールがホストメソッド内で十分な回数出現しない場合、そのルールは適用されない。Mixinの `@At(ordinal)` と同じ考え方である。
- `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` は現在C# APIでのみ利用可能である。`ncmod.json` も `[Inject]` もこれらをサポートしない（マニフェストは `ordinal` を受け付ける）。そのためマニフェストベースのModは最初のいくつかを使えない。

### 2.5 メソッド本体内のアンカー：ローカル変数と定数

前述の各種は**参照される実体**（メソッド、フィールド、コンストラクタ）にアンカーするのに対し、`LocalRead` / `LocalWrite` / `Constant` は**ホストメソッド本体内部の位置**にアンカーし、Mixinの `@ModifyVariable` と `@ModifyConstant` に対応する。この3つでは、`OriginalType` / `OriginalMethod` は参照される実体ではなく**ホストメソッド**を指す。

| 形式 | 追加パラメータ | 選択される位置 |
| --- | --- | --- |
| `LocalRead` | `LocalIndex` | そのスロットのすべての読み取り（0始まり） |
| `LocalWrite` | `LocalIndex` | そのスロットへのすべての書き込み |
| `Constant` | `ConstantValue` | その定数のすべてのロード。ボックス化された型で比較される |

```csharp
//例: G のスロット0への書き込み前にコールバックを1つ挿入する
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnWrite),
    HookType.LocalWrite, PatchMode.ILRewrite,
    localIndex: 0, placement: HookPlacement.Before)

//例: G の定数5を OnConst() の戻り値で置き換える
new HookRule("TargetLib.Host", "G", typeof(MyProbe), nameof(MyProbe.OnConst),
    HookType.Constant, PatchMode.ILRewrite, constantValue: 5)
```

置換モードでは、コールバックシグネチャはホストのパラメータではなく**命令のスタック効果**に合わせる:

| アンカー | スタック効果 | 置換メソッドのシグネチャ |
| --- | --- | --- |
| `LocalRead` | 値を1つプッシュする | パラメータなし、その値を返す |
| `LocalWrite` | 値を1つポップする | パラメータ1つ |
| `Constant` | 値を1つプッシュする | パラメータなし、その値を返す |

`ConstantValue` はボックス化された型で比較されるため、`5`（int）と `5L`（long）は2つの異なるアンカーである。`ldc.i8` にマッチさせるには `long` を渡す必要がある。

スロットはコンパイル後のローカル変数インデックスである。同じソースでもコンパイラのバージョンが違えば変わることがあるため、バージョン間の移植では安定した識別子として扱ってはならない。

### 2.6 実行時注入：すでに実行中のコードを変更する

ここまでに述べた注入はすべて**アセンブリのロード前**に起こる — まずバイト列を書き換え、それからランタイムに渡す。前提は対象アセンブリがまだロードされていないことである。

`Lead.Hook` にはもう1つのルートがある: CLRのProfilerインターフェース（ReJIT）を使い、**すでにロードされている、またはメソッドがすでに実行された**コードを変更する。どちらも同じ `HookRule` を共有し、[2.4](#24-1箇所への絞り込みホストのスコープと配置) と [2.5](#25-メソッド本体内のアンカーローカル変数と定数) のパラメータは引き続き利用できる:

```csharp
var engine = new HookEngine();
engine.AddRule(new HookRule("TargetLib.Host", "Callee", typeof(Hooks), nameof(Hooks.Double),
    hookType: HookType.CallSite, patchMode: PatchMode.RuntimeInject, inMethod: "A"));

//1回の呼び出しで注入が完了する。再起動もファイル変更も不要
RuntimeInjector.Inject(typeof(Host).Assembly, engine);
```

| | ロード時書き換え | 実行時注入 |
| --- | --- | --- |
| マニフェストの `patchMode` | `ILRewrite`（デフォルト） | `RuntimeInject` |
| タイミング | アセンブリがメモリに入る前 | プロセス開始後のいつでも |
| 基盤 | Mono.Cecil によるバイト書き換え | CLR Profiler ReJIT |
| 前提 | 対象がまだロードされていない | 対象がすでにプロセス内にある |
| JITコンパイル済みコードの変更 | 不可 | 可能 |

**なぜルールを共有するのか**: ここでも書き換えはCecilが行うが、結果はディスクに書かれない。代わりに記述へとコンパイルされネイティブ層へ渡され、ネイティブ層が実行時に新しいメソッド本体をCLRへ提出し、残りのバージョン管理はCLRに委ねられる。

**Mod側でのやり方**: `ncmod.json` のルール項目に `patchMode` を追加する。`[Inject]` アノテーションにも同名のパラメータがある。

```json
{ "target": "NetCraft.Game.Server.DedicatedServer", "method": "Tick",
  "type": "CallSite", "patchMode": "RuntimeInject",
  "replaceType": "MyMod.Probe", "replaceMethod": "OnTick" }
```

```csharp
[Inject(typeof(DedicatedServer), nameof(DedicatedServer.Tick), PatchMode = "RuntimeInject")]
public static void OnTick(object self) { }
```

**アセンブリ側の動作**: これらのルールはロード時書き換えの経路を通らない（`ModHooks.Rewrite` は `ILRewrite` のみを適用する）。アセンブリ時に別の実行時ターゲットテーブルが記録される。カーネルアセンブリのプリロード後、Modの `Init()` の前に、ローダーは対象の**すでにロード済み**のインスタンスをそれぞれ取り、書き換え済みのメソッド本体をCLRへ提出する。その時点で対象がロードされていなければ警告とともにスキップされ、そのために早期ロードされることもない — [2.2](#22-重要な制約プローブクラスはシグネチャにカーネル型を含めてはならない) の「カーネルを早期に引き上げるとウィンドウを逃す」という制約はここでは向きが逆になり、結論は同じである: 存在しなければできない。

**ネイティブ注入層へのフック**: ReJITのスイッチはプロセス開始時の環境変数（`CORECLR_ENABLE_PROFILING` / `CORECLR_PROFILER` / `CORECLR_PROFILER_PATH`）でしか設定できない。起動後に設定しても効果はない。起動時にローダーが `RuntimeInject` ルールを早期に検出し、プロセスがまだフックされていない場合、これら3つの変数を伴って**同じコマンドラインでプロセスを再起動する**（`NC_PROFILER_ATTACHED=1` は「フックしたが効かない」ときに再起動が繰り返されるのを防ぐ）。ネイティブライブラリ `lead_hook_native` はプログラムルートに置くか、`NC_PROFILER_PATH` で別の場所を指す必要がある。どちらも存在しない場合、そのバッチのルール全体が警告1つに降格され、起動は妨げられない。

**同じ対象でロード時書き換えと混在させないこと**: 実行時注入で提出されるメソッド本体は**元のバイト列**から作られ、同じメソッドに対してロード時書き換えが加えた変更を含まない — 同じメソッドが両方のルール種別に当たると、ロード時のものが完全に上書きされる。アセンブリは2つのルールが同じメソッドに当たるかを判別できないため、対象アセンブリによる粗い判断しかできず、警告を記録する。

**アノテーションはこの経路に直接関与しない**: ネイティブ注入層は `InjectAttribute` を認識せず `ncmod.json` も読まない — 認識するのは記述バイト列だけである。アノテーションとマニフェストはどちらも**アセンブリ時**のもので（[2.1](#21-注入スタイルアノテーションかマニフェストかどちらかを選ぶ) 参照）、`NetCraft.ModLoader` が `HookRule` に解析してから `RuntimeInjector` に渡す。Mod側で異なる扱いをする必要はない。

**制限**（ロード時書き換えより狭い）: 例外処理テーブルを持つメソッドは非対応、ローカル変数テーブルは変更不可、ジェネリック型とジェネリックメソッドは非対応、オペランドはメソッド参照のみを認識する（フィールド参照、文字列定数、型トークンは `NotSupportedException` を投げる）。

**パフォーマンス**: 注入は登録時にのみ起こる。その後メソッドは通常のJITコードとなり、注入なしと同じ呼び出しオーバーヘッドである。プロファイラのアタッチには一度きりのコストがある — ReJITを有効にすると同時にReadyToRunイメージを無効化する必要があり、実測でプロセス起動が約80〜110ms遅くなる。定常状態の計算には差が見られない。起動がすでに秒単位で測られるNCのようなサーバでは、これは無視できる。

### 2.7 同じクラスを変更する2つのModはMixinのように競合するか？

まず、Mixin側でなぜ競合するのか。Mixinは**メンバーを対象クラスに混ぜ込み**、クラスロード時に適用する: 複数のmixinが同じクラスに混ざる場合、同じ場所への繰り返し注入や同じクラスへの同名メンバーの追加などで `MixinApplyError` が投げられ、デフォルトのfail-hardは**ゲームを問答無用で落とす**。しかもこの検出はクラスロードの瞬間に起こり、そのときゲームはすでに中途半端に動いているかもしれない。

NCのモデルは異なり、競合が起こり得る面ははるかに小さい:

| | Mixin | NetCraft |
| --- | --- | --- |
| 適用方法 | メンバーを対象クラスに混ぜ込む + バイトコード書き換え | IL命令の書き換えのみ。型合成なし、メンバー追加なし |
| 構造的競合（同名メンバー、継承の競合） | あり | なし |
| ルールの検証時期 | クラスロード時 | アセンブリ時にメタデータを静的に読む |
| 2つのルールが同じ場所に当たる | 例外を投げる | 先着順。後からのは黙って失敗する |
| 1つのModが失敗した場合 | ロード全体を巻き込むことがある | それ自身にのみ影響する |

**静的検証**: ルールはアセンブリをロードして型をリフレクションするのではなく、PEメタデータテーブルを読んで構築される。そのため「対象型がどの既知のアセンブリにもない」「注入形式の綴りが間違っている」といった問題は、クラスがロードされるまで爆発を待たず、**起動の早い段階**で記録されスキップされる。

**障害の分離**: Modのルールの解析に失敗した場合、置換クラスのロードに失敗した場合、またはエントリの `Init()` が例外を投げた場合、**その1つのModだけ**が失敗としてマークされ（状態 `Error`、MODSページに「ロード失敗」と表示される）、他のModは通常どおりロードされる。ここで1つ表現を訂正する必要がある: NCには**実行時アンロードがない** — Modは一度ロードされ、`ModManager` は動的なロード/アンロードを明示的に提供しない。いわゆる「失敗時の自動アンロード」は実際には**ロード時の分離**である: 失敗したModは初期化されないが、「アンロード」されるわけでもない。

**動的注入**: [2.6](#26-実行時注入すでに実行中のコードを変更する) のReJITルートでは、複数のModが同じメソッドを奪い合うときの意味論は静的の場合と同じである — 最初に登録したものが勝ち、後からの要求は送られても確保できない（`GetReJITParameters` は「モジュール + メソッド」で確保し、最初のものを取る）。

このルートで一度落とし穴に遭ったので記録しておく: 初期の実装では、`FindTypeRef` の**解決スコープパラメータに `mdTokenNil` を渡していた**。その意味論は「解決スコープを持たないTypeRefのみにマッチする」であり、我々の参照はすべて `AssemblyRef` にぶら下がっているため、構築したものが決して見つからなかった。現れ方は: 最初の注入は成功したが、2回目の注入で参照解決に失敗し、新しいメソッド本体を構築できず、CLRは元のILにフォールバックし、**最初の注入もそれとともに失われた**（対象メソッドは未注入の挙動に戻った）。修正後は再現しなくなったが、制限は残る: **参照のメタデータ注入は対象モジュールのロード直後のウィンドウ内で完了しなければならず、遅くなるほど失敗しやすい**。

**コストは明示すべきである**: NCがクラッシュしない挙動は、競合を見逃しやすいことと引き換えである — Mixinは少なくともロードを中断するが、NCは後からのが黙って失敗するままにする。これに対処するため、アセンブリは**同一アンカーの競合チェック**を行う: 同じ注入ポイントが複数のModから宣言された場合、後に組み立てられたものを `ModHooks.Warnings` に記録し、起動ログに警告として報告する（ロードは妨げない）:

```
Mod injection conflict mod my-mod-b's injection NetCraft.Game.Server.DedicatedServer::Tick[CallSite/ILRewrite] is already taken by mod my-mod-a; this rule will not take effect
```

検出キーは「対象型 + メソッド + 注入形式 + パッチモード」である。**ホストのスコープは区別されない** — マニフェストもアノテーションも `InType`/`InMethod` を書けないため、Modから来るルールは自然とアセンブリ全体スコープになり、同じキーは衝突を意味する。C# APIで直接追加されたルールはこのチェックを迂回する。その場合、2つのルールがそれぞれ別のホストに当たり得て、本質的に競合しないからである。

**実行時注入もこのチェックを通る**: 同じアセンブリのエントリポイントに入り、検出キーのパッチモードがロード時書き換えと区別する。両方の種別が同じ対象アセンブリに落ちる場合は、別途上書きの通知がある（[6.4](#64-2つのmodが同じターゲットに注入する) 参照）。

### 2.8 Mixin：ターゲット型にメンバーを追加する

これまでの節はすべて既存コードの命令を変更するもので、新しいものは作れない。対象型に**フィールド、メソッド、インターフェースを追加する**にはmixinを使う。

Mixinの構文との対応は次のとおり:

| Mixin | NC |
| --- | --- |
| mixinクラスに `@Mixin(X.class)` | ソースクラスに `[Mixin(typeof(X))]` |
| mixinクラスのメンバーが対象クラスに混ぜ込まれる | ソースクラスのフィールドとメソッドが対象型へ移動される |
| `@Unique` がprivateフィールドを追加する | ソースクラスに普通のフィールドを書く。同じように移動される |
| `@Shadow` が対象クラスの既存メンバーを参照する | 不要。`X` のメンバーを直接書き、その後でフックする |
| `@Implements` / `implements` | `Interfaces` |

マニフェストにも書ける:

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
    //移動された後、これは対象型のインスタンスフィールドになる
    public int MyCounter = 5;

    //混ぜ込まれたメソッド。一緒に移動したフィールドを読み書きする
    public int Bump() => MyCounter + 1;

    //インターフェースが要求する実装。移動された後、対象型は ITagged を実装する
    public string Describe() => $"tagged:{MyCounter}";
}
```

**移動でありコピーではない**。これらのメンバーはソースクラスから取り除かれ、空の殻だけが残る — Mixinのmixinクラスと同様、**Modのコードはそのクラスをもう使うべきではない**（`new SomeEntityMixin()` やそのメソッドの呼び出しはメンバーを見つけられず失敗する）。

いくつかの適用ルール:

- **ロード時書き換えにのみ適用される**。対象型はまだメモリに入っていないカーネルアセンブリかModアセンブリにある必要がある。実行時注入はメソッド本体を提出するだけで型レイアウトを変更できないため、この形式は実行時注入には存在し得ない。
- **コンストラクタの初期化子も付いてくる**。ソースクラスのフィールド初期化子に書かれた値は、対象型のすべてのインスタンスコンストラクタにマージされる。静的フィールド初期化子は静的コンストラクタにマージされる（対象に存在しない場合は作成される）。ソースクラスのコンストラクタ内の基底クラスへの連鎖呼び出しは取り除かれ、基底コンストラクタが二度実行されることはない。
- **インターフェースを伴うルールは、移動したpublicインスタンスメソッドをvirtualにする**。インターフェースのディスパッチはvtableしか認識せず、このマークがないとCLRはインターフェースが実装されていないと判断し、ロードに失敗する。したがってインターフェースを混ぜ込むとき、それらのメソッドが非virtualのままだとは期待しないこと。
- **同名メンバーはスキップされる**。対象型にすでに同名のフィールドやメソッドがある場合、その1項目は移動されず、残りは通常どおり進む。2つのModが同じ対象型に混ぜ込む場合、どちらも適用され、後者の名前が衝突する部分だけがスキップされる — [2.7](#27-同じクラスを変更する2つのmodはmixinのように競合するか) のフックの「後者が完全に失敗する」挙動より穏やかである。
- **ネスト型は移動されない**。ソースクラス内のネスト型やジェネリックメソッドも現在この経路の対象外である。

ソース型は**Mod自身のアセンブリ**にある必要があるため、マニフェストもアノテーションもアセンブリ名を書かない。

### 2.9 2つのルート：ラッパー層と拡張ポイント

`NetCraft.ModApi` の公開サーフェスは2つの名前空間に分かれ、2つの用途に対応する:

| 名前空間 | 内容 | 得られるもの |
| --- | --- | --- |
| `NetCraft.ModApi.Wrapper` | `ServerEvents` / `ClientEvents` / `NetworkEvents`、`NcServer` / `NcWorld` / `NcPlayers` / `NcLists` / `NcRegistries` / `NcRecipes` / `NcStartup`、および `NcPlayer` / `NcLevel` のようなオブジェクトハンドル | ラッパー型。公開サーフェスにカーネル型はない |
| `NetCraft.ModApi.Extension` | `[Inject]` / `[Mixin]` アノテーション | ルールがカーネルのクラス名とメソッド名に束縛される |

両者は**並列**のルートであり、一方が他方の上に重なるものではない:

- **安定性を求めるなら `Wrapper` を使う**。ファサードがカーネルの面倒な呼び出し順序を処理してくれる（1つのブロックを書き込みながらレベル、プレイヤーリスト、同期チェーンに触れるのが一例）。イベントの args もすべてラッパー型である。代償は、ファサードが公開していない機能は使えないことである。
- **網羅性を求めるなら `Extension` を使う**。注入ルールはカーネルのクラスとメソッドを直接変更するが、書く対象名はカーネルの名前であるため、カーネルが変わればルールもそれに合わせて変えなければならない。

両方を参照できる。`Wrapper` の路線は「公開サーフェスにカーネル型を出さない」方向へ統合されつつある。プレイヤーとレベルの部分は完了している — プレイヤーイベントの `Player` / `Attacker` と `NcPlayers` の入出力パラメータは `NcPlayer` ハンドル、`NcWorld.Overworld` / `Nether` / `End` / `Get` と `LevelTickArgs.Level` は `NcLevel` ハンドル、ブロック座標は単なる `x y z` の int である。エンティティと残りの値型（`BlockPos` / `BlockState` / `Vec3`）はまだラップされていない。

もう1つ挙げておく境界: **ラッパー層は注入を遮蔽しない**。書く `hooks` ルールや `[Inject]` アノテーションは依然としてカーネルのクラス名とメソッド名に束縛され、カーネルが変われば同じように壊れる。

---

## 3. Fabricからの移行

### 3.1 概念の対応

| Fabric | NetCraft |
| --- | --- |
| `fabric.mod.json` | dllに埋め込まれた `ncmod.json` |
| `ModInitializer.onInitialize()` | エントリクラスの `public Task Init()` |
| `@Inject` / `@Redirect` | `[Inject]` アノテーション、または `hooks` 内の `Mark` / `Probe` / `CallSite` のようなルール |
| `@ModifyVariable` | `LocalRead` / `LocalWrite`。[2.5](#25-メソッド本体内のアンカーローカル変数と定数) 参照。アノテーションとしては書けない |
| `@ModifyConstant` | `Constant`。[2.5](#25-メソッド本体内のアンカーローカル変数と定数) 参照。アノテーションとしては書けない |
| `@Accessor` | まだ相当するものはない（`private` メンバーは可視性を広げる必要がなく、ルールを書けばよい） |
| `Registry.register(...)` | カーネルのレジストリ（`BuiltInRegistries`） |
| `ServerLifecycleEvents.SERVER_STARTED` | `ServerEvents.Started` |
| `ServerTickEvents.END_SERVER_TICK` | `ServerEvents.Tick` |
| `CommandRegistrationCallback` | `ServerEvents.CommandRegister` |
| `ClientTickEvents.END_CLIENT_TICK` | `ClientEvents.Tick` |
| `FabricLoader.getInstance().getModContainer(id)` | まだ相当するものはない（`ModManager` はModに公開されていない） |
| `@Mixin` / `@Unique` / `@Implements` | `[Mixin]` アノテーション、またはマニフェストの `mixins`。[2.8](#28-mixinターゲット型にメンバーを追加する) 参照 |

### 3.2 引き継げないもの

- **Mixinのアノテーションシステム**: NCには `[Inject]` と `[Mixin]` の2つのアノテーションがあり、どちらも**宣言スタイル**にすぎず、`ncmod.json` の `hooks` / `mixins` と等価で、アセンブリ時にマージされる（解決するのはローダーであり `Lead.Hook` ではない。[2.1](#21-注入スタイルアノテーションかマニフェストかどちらかを選ぶ) 参照）。命令の書き換えはデフォルトでロード時に適用され、[2.6](#26-実行時注入すでに実行中のコードを変更する) に従って実行時提出に変更できる。メンバーとインターフェースの追加は [2.8](#28-mixinターゲット型にメンバーを追加する) のmixinを通る。対象指定に `@At` 風の文字列構文はないが、`CallSite`/`FieldRead`/`LocalWrite`/`Constant` といった形式と、`Ordinal`、`InType`/`InMethod`、`Placement` を組み合わせれば `HEAD`/`RETURN`/`INVOKE`/`FIELD`/`NEW`/`CONSTANT`/`LOAD`/`STORE` や `shift` の用法をカバーできる。欠けているのは `JUMP` である。
- **AccessWidener**: なし。NCでは可視性はIL書き換えの障害にならない。`private` メソッドも同様にフックできる（リライタはバイトレベルで動作する）。
- **Yarn / Mojang マッピング**: 不要。NCはバニラから直接翻訳されたC#ソースで、型名とメソッド名はバニラに対応し、命名スタイルだけがC#に従う。
- **Fabric APIのモジュールの大半**: 利用できるのは `NetCraft-ModApi` がカバーする機能だけである。それ以外は自分で注入ルールを書くか、APIが追いつくのを待つ。

### 3.3 並べて比較する例

Fabric: サーバ起動時に1行ログを出し、コマンドを1つ登録する。

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

NC:

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

マニフェスト（`ncmod.json`、埋め込みリソースとして）:

```json
{
  "id": "my-mod",
  "version": "0.1.0",
  "environment": "server",
  "entry": "MyMod.MyModEntry",
  "hooks": []
}
```

注: ここでは空の `hooks` でも機能する — `ServerEvents.Started` のようなイベントは `NetCraft-ModApi` 自身のプローブが提供し、あなたのModは購読するだけでよい（`ServerEvents` は `NetCraft.ModApi.Wrapper` の下にある。[2.9](#29-2つのルートラッパー層と拡張ポイント) 参照）。自分でフックルールを書く必要があるのは、ModApiがまだイベントを提供していないカーネルの場所をフックしたい場合だけである。

---

## 4. ncmの基本要件

ncmはNetCraft modの略である。ncmは `ncmod.json` を埋め込んだ .NET クラスライブラリのdllで、`mods/` ディレクトリに置かれる。

手っ取り早い始め方はテンプレートを使うことである:

```
dotnet new install NetCraft.ModsProjectType
dotnet new ncm -n MyMod -e server
```

ローカルパッケージがまだ公開されていない場合は `dotnet new install <nupkg path>` を使うか、リポジトリで `dotnet pack` を実行してから出力をインストールする。

`-e` は `both`（デフォルト）/ `server` / `client` を取り、マニフェストの `environment` と、エントリクラスにどちらのサイドの購読コードを生成するかを決める。テンプレートはNCの参照アセンブリを同梱するためプロジェクト参照は不要で、`ncmod.json` の id / entry はプロジェクト名から埋められる。

4.1 以降では、手書きのプロジェクトが満たすべき事項を述べる。

### 4.1 プロジェクトファイル

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Private をオフにしないと、NCのアセンブリが 4.6 の EmbedDependencies によってModのdllに埋め込まれてしまう -->
    <ProjectReference Include="xxx\NetCraft\NetCraft.csproj" Private="false" />
    <ProjectReference Include="xxx\NetCraft.ModApi\NetCraft.ModApi.csproj" Private="false" />
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Include="ncmod.json" LogicalName="ncmod.json" />
  </ItemGroup>
</Project>
```

`LogicalName` は `ncmod.json` でなければならない。スキャナはその名前しか認識しない。

テンプレートは別のルートを取る: `libs/` 配下のアセンブリ参照（`Reference Include="libs\*.dll" Private="false"`）である。NCのソースがない場合はこちらに従う。両方式に共通するのは、**NC自身のdllを絶対に出力ディレクトリに入れてはならない**ことである — [4.6](#46-サードパーティ依存関係) の `EmbedDependencies` は出力ディレクトリのサードパーティdllをModに埋め込むため、NCのアセンブリまで埋め込まれると型の同一性が二重になる。

### 4.2 ncmod.json のフィールド

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

| フィールド | 必須 | 備考 |
| --- | --- | --- |
| `id` | はい | Mod識別子。依存関係と検索に使われる。空文字列はスキャナにスキップされる |
| `version` | 推奨 | バージョン番号。他のModから依存されるとき、バージョン制約はこれで判定される。後述 |
| `name` | いいえ | 表示名。Modページに表示されるもので、デフォルトでは `id` に戻る |
| `description` | いいえ | 1行の説明 |
| `authors` / `contributors` | いいえ | クレジット。文字列配列 |
| `license` | いいえ | ライセンス識別子 |
| `contact` | いいえ | 外部リンク。`homepage` / `sources` / `issues` を取れる |
| `icon` | いいえ | アイコンの埋め込みリソース名。4.7 参照 |
| `environment` | いいえ | `both` / `client` / `server`、デフォルトは `both`。現在のサイドと一致しない場合、Mod全体がロードされない |
| `entry` | はい | エントリクラスの完全名。クラスは `public Task Init()` を持たなければならない |
| `depends` | いいえ | 依存する他のModと要求バージョン。後述 |
| `hooks` | いいえ | 注入ルールのリスト。空配列は、ModApiがすでに持つイベントを購読するだけを意味する |
| `mixins` | いいえ | mixinルールのリスト。このModのクラスのメンバーを対象型へ移動する。[2.8](#28-mixinターゲット型にメンバーを追加する) 参照 |

`depends` は依存する他のModと要求バージョンを宣言する。キーはModのidで、値はバージョン制約である:

```json
{
  "depends": {
    "othermod": "^1.0.0",
    "another-mod": ">=2.1"
  }
}
```

**`depends` に書かれていない依存関係はバージョン制約を持たない**。Mod間の依存関係はコンパイル時の参照からすでに自動推論され（[6.1](#61-mod依存関係と依存先modへの注入) 参照）、`depends` はその上にバージョン制約を追加するだけである。依存先のModが現在のサイドにない場合はチェックがスキップされる — その場合はアセンブリ解決が報告するに委ねられる。

| 制約構文 | 意味 |
| --- | --- |
| `*` | 任意のバージョン。項目を省略するのと等価 |
| `1.2.3` | セグメントプレフィックス。`1.2` は `1.2` と `1.2.9` にマッチするが `1.3` にはマッチしない |
| `^1.2.3` | 同じメジャーバージョンで、基準以上。メジャーバージョンが0の場合は代わりにマイナーバージョンを使うため、`0.1` と `0.2` は非互換とみなされる |
| `>=1.2.3` | 基準以上 |

バージョン番号は各セグメントの先頭の数字のみを取るため、`26.2-netcraft` は `26.2` として扱われる。バージョンが一致しない場合、**宣言したModだけがスキップされ**、残りは通常どおりロードされる。起動ログには「依存関係 X はバージョン … を要求、実際のバージョン …」と記される。

判定の根拠は依存先Modのマニフェストの `version` フィールドである。したがって**依存されることを意図したModは `version` を正しく設定しなければならない** — 空のバージョン番号はどの特定の制約も満たさない。

### 4.3 hookルールのフィールド

| フィールド | 備考 |
| --- | --- |
| `target` | 対象型の完全名。カーネルアセンブリか `mods/` 配下のModアセンブリのいずれかにある必要がある |
| `method` | 対象メソッド名。同名のオーバーロードはすべてマッチする |
| `type` | 注入形式。付録参照 |
| `patchMode` | 適用方法。`ILRewrite`（デフォルト）または `RuntimeInject`。[2.6](#26-実行時注入すでに実行中のコードを変更する) 参照 |
| `ordinal` | 同じアンカーがホストメソッド内の複数箇所にマッチするとき、どれを選ぶか（0始まり）。[2.4](#24-1箇所への絞り込みホストのスコープと配置) 参照 |
| `replaceType` | 置換メソッドを含むクラスの完全名 |
| `replaceMethod` | 置換メソッド名 |
| `label` | プローブのラベル。`Mark` と `Probe` のみが使う |
| `environment` | ルールが適用されるサイド。デフォルトは `both`。サーバ型を対象とするルールをクライアントで実行すると対象が全く存在せず、`environment` によって除外される |

同じルールは置換メソッド上のアノテーションとしても書ける。対応は次のとおり:

```csharp
[Inject(typeof(SomeType), nameof(SomeType.SomeMethod),
        HookType = "CallSite", Label = "some_label", Environment = "server")]
public static void OnSomeMethod(object self) { }
```

| マニフェストのフィールド | アノテーションの形式 |
| --- | --- |
| `target` | 最初のコンストラクタパラメータ。`typeof(...)` として書く |
| `method` | 2番目のコンストラクタパラメータ。できれば `nameof(...)` |
| `type` | 名前付きパラメータ `HookType`、デフォルトは `CallSite` |
| `patchMode` | 名前付きパラメータ `PatchMode`、デフォルトは `ILRewrite` |
| `ordinal` | 名前付きパラメータ `Ordinal` |
| `label` | 名前付きパラメータ `Label` |
| `environment` | 名前付きパラメータ `Environment`、デフォルトは `both` |
| `replaceType` | 書かない。注釈を付けたクラスから自動的に取られる |
| `replaceMethod` | 書かない。注釈を付けたメソッドから自動的に取られる |

アノテーションは `NetCraft.ModApi.Extension` の `InjectAttribute` に由来するため、アノテーションを使うModはこれを参照する必要がある。アノテーションとマニフェストが両方存在する場合はマージされ、同じ注入ポイントが両方で宣言されている場合は**アノテーションが優先される**。

Mixinルールは別のフィールド群を使う: `target`（対象型の完全名）、`source`（ソース型の完全名。Mod自身のアセンブリにある必要がある）、`interfaces`（省略可能。インターフェースの完全名の配列）。意味論は [2.8](#28-mixinターゲット型にメンバーを追加する) 参照。ソースクラスの `[Mixin(typeof(target))]` はマニフェストの項目と等価である。

### 4.4 フックできるものとできないもの

フックできるもの: `kernel/` 配下のカーネルアセンブリ、および `mods/` 配下の他のMod。

フックできないもの:

- メインライブラリ `NetCraft.dll`
- ローダー `NetCraft.ModLoader.dll`
- エントリアセンブリ（`NetCraft.Server.Exe.dll` など）

すべてのフックの `target` はカーネルかいずれかのModアセンブリで見つけられなければならない。そうでないとアセンブリは「注入対象がどの既知のアセンブリにもない」と報告する。**名前空間はアセンブリを意味しない**点に注意 — `NetCraft.Game.Server.DedicatedServer` は実際には `NetCraft.Server.dll` にある。ローダーはメタデータテーブルから構築したインデックスでこれを探すため、完全名を書けばよい。

ModによるModへの注入も同じ仕組みに従う: 対象Modは**それ自体がロードされた瞬間**に書き換えられ、`mods/` 内の順序にはよらない。循環ルール（AがBを、BがAを注入する）はロード時に「循環ロード」と報告される。そのようなルールはIL書き換えのレベルでは解決策がないため、一方を削除するだけでよい。

注入されるModは**自分のModであってはならない**点に注意 — 自己注入も循環とみなされる。

### 4.5 デプロイ

コンパイルしたdllを出力ディレクトリの `mods/`（最上位。サブディレクトリの再帰はない）へコピーし、プロセスを再起動する。

**このステップはすでにビルドで自動化されている**: `NetCraft.ModApi.csproj` の `DeployModToHosts` がビルド後にdllを各ホストプロジェクトの出力ディレクトリの `mods/` ディレクトリへコピーする。ホストのリストは `ModHostProjects` プロパティである。自分のホストプロジェクトを作るときは、その名前を追加するだけでよい。ルールが全く効かずエラーも全く出ない場合は、まずホストプロジェクトの `mods/` 内のdllが古くないか確認する。

起動ログは次のように出力する:

```
Rewrote and preloaded assembly NetCraft.Server
Rewrote and preloaded assembly NetCraft.Game
Mod scan finished: 1 found, injection targets NetCraft.Server,NetCraft.Game runtime
Mod init finished: 1 loaded, 0 skipped
```

トラブルシューティングの順序:

- どのルールも効かない: まず `mods/` 内のdllが古くないか確認する（最もよくある）。
- `Rewrote and preloaded assembly` が見当たらない場合、対象アセンブリはブートストラップ前にすでにロードされており、ルールの記述が遅すぎた。
- `injection targets` は見えるが対象が間違っている場合、通常は `target` の打ち間違いである。[4.4](#44-フックできるものとできないもの) に従い、その型がどのアセンブリに属するか確認する。
- `environment` が `client` に設定されたルールはサーバ上で黙ってスキップされる（Modレベルでの不一致はMod全体がロードされないことを意味する）。これは想定された挙動であり、不具合ではない。

### 4.6 サードパーティ依存関係

FabricのJar-in-Jarに対応する。

テンプレートから作成したプロジェクトでは**これを気にする必要はない**: `dotnet add package` で追加したライブラリはビルド時に自動でModのdllに埋め込まれ、ローダーがアセンブリを解決できないときはModの埋め込み `.dll` リソースを探す。

```
dotnet add package Newtonsoft.Json
```

これだけである。`ncmod.json` の変更は不要。

手書きのプロジェクトはテンプレートのcsprojから `EmbedDependencies` ターゲットをコピーするか、自分で埋め込む必要がある:

```xml
<ItemGroup>
  <EmbeddedResource Include="deps\MyLib.dll" LogicalName="MyLib.dll" />
</ItemGroup>
```

依存関係はマニフェストには書かれない。解決はModの埋め込み `.dll` リソースだけを見て、リソース名がアセンブリ名と一致していればよい。

3つの注意点:

- **照合はアセンブリ名で行う**。ローダーは要求されたアセンブリ名を比較する: `.dll` を除いた後、リソース名がアセンブリ名と等しいか、`.` + アセンブリ名で終わればよい。したがって `MyLib.dll` とデフォルトの `MyProject.deps.MyLib.dll` のどちらも機能する。
- **認識されるのはMod自身の埋め込みリソースだけである**。埋め込まれていないライブラリは解決できず、他所を探されることもない。
- **解決順序はカーネルが先である**。`EmbeddedAssemblyLoader` はまずカーネルアセンブリと埋め込みサブライブラリの中を探し、何も見つからない場合のみModにフォールバックする。そのためModはカーネルと同じ名前のアセンブリを埋め込むべきではない。

テンプレートのターゲットは `Condition` を書かずに `WithMetadataValue` で `.dll` をフィルタする。テンプレートエンジンはテンプレート時に `.csproj` の `Condition` を評価し、そのとき `%(...)` には値がないため、行全体が削除されてしまうからである。

### 4.7 アイコンと表示情報

名前、説明、作者、リンク、アイコンはすべて `ncmod.json` に書かれ、バニラの `fabric.mod.json` の `name` / `description` / `authors` / `contact` / `icon` に対応する。

**アイコンは埋め込みリソース**であり、外部ファイルではない。埋め込み依存関係と同じリソース命名規則を使う:

```xml
<ItemGroup>
  <EmbeddedResource Include="icon.png" LogicalName="icon.png" />
</ItemGroup>
```

```json
{ "icon": "icon.png" }
```

テンプレートはすでに `icon.png` を同梱し、両方の箇所が設定済みである。その画像を置き換えるだけでよい。64×64 または 128×128 のPNGを推奨する。

アイコンの検索順序は: マニフェストの `icon` が指すもの → `icon.png` という名前の埋め込みリソース → どちらも存在しない場合はUIのデフォルト画像（灰色の疑問符）。

この情報はサーバGUIの**MODS**ページで見られる。左側がModリスト（小さいアイコン + 表示名 + バージョン + 状態）で、右側が選択中のものの説明、クレジット、リンク、依存関係、初期化時間、注入ルールである。依存関係の列のMod名はクリックでき、その項目へ直接ジャンプする。

ロードに失敗したModやスキップされたModもリストにあり、状態列に表示される。「なぜ自分のModが効かないのか」を診断するときは、まずこの列を確認する。

---

## 5. 既知の制限

- **プローブのシグネチャにはBCL型と `object` しか使えない**。理由は 2.2 にある。値型のパラメータと戻り値は例外で、本来の型を保たなければならない。
- **`CallSite` は置換である**。置換メソッドは元の呼び出しを自分で復元しなければならない。2.3 参照。private メソッドは復元できず、リフレクションが必要である。
- **`mods/` 配下のdllは `DeployModToHosts` が自動で配置する**。自分のホストプロジェクトを作るときは、`ModHostProjects` にその名前を追加するのを忘れないこと。
- **エントリアセンブリには注入できない**: 対象とフックがエントリアセンブリにある場合、それは効かない。
- **メインライブラリとローダーはフックできない**。これはModがロードプロセス自体を変更するのを防ぐ設計上の制約である。
- **`environment` の不一致はMod全体がロードされないことを意味する**。「一部のルールが失敗する」ではない。
- **実行時注入にはネイティブライブラリが必要である**: `RuntimeInject` を使うルールはプロセス開始時に `lead_hook_native` がアタッチされている必要があり、そのためにローダーは自身を再起動する。ライブラリが見つからないか再起動に失敗した場合、このバッチのルールは警告に降格され、起動は妨げられない。能力の境界とコストについては [2.6](#26-実行時注入すでに実行中のコードを変更する) を参照。
- **アノテーションはC# APIよりフィールドが少ない**: `InType` / `InMethod` / `Placement` / `LocalIndex` / `ConstantValue` は `[Inject]` に書けない（`PatchMode` と `Ordinal` はサポートされる）。[2.1](#21-注入スタイルアノテーションかマニフェストかどちらかを選ぶ) 参照。
- **Mixinはロード時書き換えにのみ適用される**。ソースクラスのメンバーはコピーではなく移動される。ソースクラスのネスト型とジェネリックメソッドは対象外で、インターフェースを伴って混ぜ込まれたメソッドはvirtualにされる。[2.8](#28-mixinターゲット型にメンバーを追加する) 参照。
- テストやデバッグの際、言語テーブルやモデルリソースを使う場合は `assets` ディレクトリ（バニラのjarから抽出したもの）が必要である。そうでないと関連機能が翻訳キーやプレースホルダテクスチャに降格する。

---

## 6. 既知の挙動

この節は観察された挙動の記録であり、仕様ではない。

### 6.1 Mod依存関係と依存先Modへの注入

**シナリオ**: b が a に依存し、かつ a を注入する。

**結論**: 機能し、循環にはならない。

連鎖は3つのステップからなる:

1. ルールテーブルはPEメタデータを読むだけで構築され、アセンブリを一切ロードしない。「bがaを注入する」というルールはaもbも存在することを要求しない。
2. `PreloadReplacers` は `ModManager` より前にすべての置換クラス（bを含む）をロードし、この時点でaはまだロードされていない。`LoadFromStream` はメタデータを読むだけでメソッド本体をJITしないため、この瞬間のbからaへの参照は遅延であり、ロードは失敗しない。
3. 次に `ModManager` が `AssemblyRef` でトポロジカルソートし、aがbより前になる。aを書き換えるとき、置換クラスbはすでに `Default` にあるため、型は直接取得され、bを指す `AssemblyRef` が1つaのメタデータに追加される。**aはbの存在を全く知る必要がない。**

**厳守事項**: csprojで注入されるModを参照するときは `Private="false"` を書かなければならない。デフォルトでは `a.dll` を出力ディレクトリにコピーし、それが `EmbedDependencies` によって埋め込み依存関係として `b.dll` に埋め込まれ、実行時に `ModLibs` が `a` を解決する際に2つ目のコピーを拾い、両者間の型同一性チェックが失敗する原因になる。

**検証ケース**: 2つのテンプレートプロジェクト `NetCraft.Test1`（a）と `NetCraft.Test2`（b）。aは `Test1Api.Greet` を提供し自身の `ModEntry.Server` でこれを呼び、bはその呼び出し箇所を `Test2Probe.OnGreet` に置換し、対照としてbの `ModEntry.Server` でも `Greet` を1回呼ぶ。実際の実行ログ:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]
Mod scan finished: 2 found, injection targets NetCraft.Server,NetCraft.Game,NetCraft.Storage,NetCraft.Test1 runtime
Test1 internal greeting: Test2-rewritten greeting self    ← injection took effect
Test2 internal greeting: Test1 original greeting test2    ← control group: the rule only rewrites the target assembly, so b's own internal call site is untouched
```

依存関係は `AssemblyRef` からのみ順序を導き、バージョン制約を持たない。バージョンを制約するにはマニフェストの `depends` で宣言する。[4.2](#42-ncmodjson-のフィールド) 参照。

### 6.2 アノテーションによる注入

**シナリオ**: 置換メソッドに `[Inject(typeof(X), nameof(X.M))]` を付け、`ncmod.json` にはルールがない。

**結論**: 機能し、マニフェストへの宣言は不要である。アノテーションとマニフェストは1つの源を共有し、アセンブリ時にマージされ、同じ注入ポイントではアノテーションが優先される。

**なぜ宣言が不要か**: アノテーションはメタデータ内の `CustomAttribute` テーブルにすぎず、ローダーはModのスキャン時にすでに同じメタデータ（マニフェスト、埋め込みリソース、AssemblyRef の3項目）を読んでいる。テーブルを1つ多く読んでも新たなロードやタイミングの制約は生じない。唯一の前提は、Modが `NetCraft.ModApi`（アノテーションのホスト）を参照していることである。

**前提は静的読み取りである**: アノテーションの読み取りは `MetadataReader` を通さなければならず、**`Assembly.Load` + `GetCustomAttributes` を使ってはならない** — 後者はルールを読むためだけにModアセンブリを引き上げてしまい、その場で書き換えウィンドウが失われる。

**`typeof` は型参照を構成しない**: `typeof(X)` がパラメータへコンパイルするのは型のシリアル化された名前（`full name, assembly, Version=…`）であり、これは単なる文字列に解決され、`X` の存在を要求しない。したがってアノテーションに `typeof(injected-mod)` と書いても、2.2 の「置換クラスは注入されるModの型を参照してはならない」という制約に**違反しない** — 名前がメタデータに現れることと、実行時に型を解決することは別物である。

**検証ケース**: `NetCraft.Test2` を `NetCraft.Test1` の2つのメソッドに対して。`Greet` はアノテーションを通り、`Farewell` はマニフェストを通る。両方のルールが設置され、両方の呼び出し箇所が置換される:

```
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Greet replaced by NetCraft.Test2.Test2Probe::OnGreet [CallSite/ILRewrite]        ← annotation
Mod netcraft.test2 injects NetCraft.Test1!NetCraft.Test1.Test1Api::Farewell replaced by NetCraft.Test2.Test2Probe::OnFarewell [CallSite/ILRewrite]  ← manifest
Test1 internal greeting: Test2-rewritten greeting self
Test1 internal farewell: Test2-rewritten farewell self
Test2 internal greeting: Test1 original greeting test2     ← control group: the rule only rewrites the target assembly
Test2 internal farewell: Test1 original farewell test2     ← control group
```

同じケースで、アノテーションの名前付きパラメータ（`Environment = "server"`）も解決されることが副次的に確認された。

### 6.3 プロファイラをアタッチする性能コスト

**シナリオ**: 同じ純粋計算プログラム（1億回の剰余と加算の反復）を、プロファイラなしで1回、アタッチあり（3つの `CORECLR_ENABLE_PROFILING` 環境変数）で1回実行する。各5回。

**結論**: 定常状態に差はなく、コストは完全に起動時にある。

| | プロセス総時間（5回、ms） | プログラム内の計算時間 |
| --- | --- | --- |
| なし | 306 / 260 / 278 / 253 / 300 | 225 ms |
| あり | 406 / 370 / 340 / 325 / 354 | 193 ms |

総時間は約80〜110ms長い。原因は `COR_PRF_DISABLE_ALL_NGEN_IMAGES` である — ReJITを有効にすると同時にReadyToRunイメージを無効化する必要があるため、フレームワークのコードはJITしか通れない。プログラム内の計測ループはJITコンパイル後は同一で、差は見られない（プロファイラありの実行の方が実際にはわずかに速く、これはノイズである）。

**ついでに直した無駄**: 初期の実装は `COR_PRF_MONITOR_JIT_COMPILATION` を購読していた。これはメソッドがコンパイルされるたびに発火するネイティブコールバックで、我々は一切使っていなかった。これを削除した後、イベントマスクは `0x80040024` から `0x80040004` に変わった。上の表は削除後のデータである。

**検証ケース**: `__hookverify/BenchProbe`。

### 6.4 2つのModが同じターゲットに注入する

**シナリオ**: 2つのModがそれぞれ同じ対象メソッドに当たるルールを宣言する（`MethodBody` 形式、置換メソッドは異なる）。

**結論**: エラーもクラッシュもない。**先に組み立てられたルールが勝ち、後からのものは黙って失敗する**。

両方のルールがルールテーブルに入る — Mod間の重複排除は行われない。書き換え時には、`OriginalType::OriginalMethod` の同一キーリストの**最初の**項目が取られる。ホストのスコープも同じように振る舞い、`InType`/`InMethod` は「最初にマッチしたものが勝つ」。どちらが最初かはアセンブリ順序に依存し、アセンブリ順序はmodsディレクトリの列挙順に由来する。**優先度フィールドはなく、依存関係を宣言しても制御できない**（依存関係が影響するのは `Init()` の順序だけで、注入ルールの組み立てではない）。

**実行時注入も先着順である**: 後からの登録要求は通常どおり送られるが、`GetReJITParameters` は「モジュール + メソッド」で確保し、常に最初の要求にマッチするため、後からの登録は適用されない。2回目の注入後に計測しても、対象メソッドの挙動は最初の結果のままである。

**1つの壊れたルールは他に影響しない**: 対象型がどの既知のアセンブリにもない、注入形式の綴りが間違っているといった問題はアセンブリ時に `ModHooks.Errors` に記録され、その項目はスキップされ、他のModのルールは通常どおり組み立てられる。

**競合は記録される**: アセンブリが同じ注入ポイントを複数のModが宣言しているのを検出すると、後に組み立てられたものが `ModHooks.Warnings` に書かれ、起動ログに `Mod injection conflict ...` として出力され、どの2つのModが衝突したか、どちらが効果を発揮しないかが示される。これは警告であってエラーではなく、ロードに影響せず、ルール自体はテーブルに残る（単に到達できないだけである）。

**実行時注入もこのチェックを通る**: 競合検出キーは `patchMode` を含むため、あるModが `ILRewrite` を、別のModが `RuntimeInject` を書いても衝突とはみなされない（2つの独立した経路がそれぞれ動く）。同じモードの2つだけが競合と判定され警告される。その実際の適用も同様に先着順である — `GetReJITParameters` は「モジュール + メソッド」で確保し最初の要求にマッチするため、後からのものは送られても適用されない。

**唯一のハードクラッシュ点**: 2つのModが同じメソッドを `RuntimePatch` モードでパッチすると、2つ目が `RuntimeHookEngine` の重複登録チェックに引っかかり `InvalidOperationException` を投げ、この経路は捕捉されないため起動が完全に失敗する。`RuntimePatch` は廃止方向にある（[2.6](#26-実行時注入すでに実行中のコードを変更する) 参照）。新しいルールでは使わないこと。

**検証ケース**: `NetCraft.Test` の `modinjection` モジュール、項目 `same anchor first mod wins quietly` と `one bad rule does not sink the rest`。

### 6.5 未検証

- **実サーバでの実行時注入の動作**: `RuntimeInject` モードは `__hookverify/RuntimeProbe` でエンドツーエンドに検証済みである（登録後、対象メソッドの挙動が置換先のものに差し替わる）。Modアセンブリ側にもルーティングと降格のテストカバレッジがある。しかし現在、`lead_hook_native` をNCの実行ディレクトリに置くビルドステップがないため、この連鎖を実サーバで動かすにはまずライブラリをプログラムルートに置く（または `NC_PROFILER_PATH` で指す）必要がある。このステップは未実施である。
- **置換クラスが注入されるModの型を参照する場合**: 推論では、aを書き換える瞬間にaを解決しようとするが、aはロード完了の直前に留まっている（まだ `Default` におらず、`ModLibs` もカーネルの解決コールバックもModアセンブリを認識しない）。そのため `PrepareMod` は例外を投げ `result.Errors` に記録されると予想される。まだ実際には実行していない。6.2 は**アノテーション内の `typeof`** が参照でないことを証明するだけである。**メソッドシグネチャに現れる型**は別の話である。

---

## 付録：HookType の概要

| 形式 | 効果 | 置換メソッドのシグネチャへの要件 |
| --- | --- | --- |
| `CallSite` | 対象メソッドへの呼び出し箇所を自分のメソッドに置き換える | パラメータ数が呼び出されるメソッドと一致する（インスタンス呼び出しは +1） |
| `MethodBody` | 対象メソッド本体全体を置き換える | 置換されるメソッドと一致する |
| `NewObj` | `new X(...)` を置き換える | パラメータ数がコンストラクタと一致する |
| `FieldRead` | フィールドの読み取りを計測する | 読み取る型による |
| `FieldWrite` | フィールドの書き込みを計測する | 書き込む型による |
| `TypeCheck` | `isinst` / `castclass` を計測する | チェックされる型による |
| `Box` | ボックス化/ボックス解除を計測する | 要素の型による |
| `FunctionPointer` | 関数ポインタのロードを計測する | デリゲートの型による |
| `LocalRead` | ローカル変数の読み取りを計測する | パラメータなし、その変数の値を返す |
| `LocalWrite` | ローカル変数の書き込みを計測する | パラメータ1つ、書き込まれる値を受け取る |
| `Constant` | 定数のロードを計測する | パラメータなし、その定数の値を返す |
| `Probe` | 元のメソッド本体を保ち、入口とすべての出口を計測する。`LabelArgumentIndex` を使うと、引数を1つラベルに畳み込める | `Begin()` は long を返し、`End(string, long)` |
| `Mark` | メソッド入口でのみ1回報告し、計時はしない | `void method(string label)` |

`Probe` と `Mark` はラベルテキストのみを渡す（`Probe` は引数1つの `ToString()` も含められる）。オブジェクト参照は取得できない。実際の引数を得るには `CallSite` を使う。

`InType`/`InMethod`、`Placement`、`Ordinal`（[2.4](#24-1箇所への絞り込みホストのスコープと配置) 参照）は命令レベルの形式にのみ意味がある: 上の表のうち `MethodBody`、`Probe`、`Mark` 以外の10項目は置換か前/後への挿入を選べ、`Ordinal` で1回の出現を選べる。`MethodBody` は常に全体を置換し、`Probe`/`Mark` はこれらのパラメータを無視する。

`LocalRead` / `LocalWrite` / `Constant` の3種では、ホストメソッドは参照される実体ではなく `target` に書かれ、加えて `localIndex` または `constantValue` が必要である。[2.5](#25-メソッド本体内のアンカーローカル変数と定数) 参照。
