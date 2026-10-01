---
title: "SteamVRのオーバーレイをC#だけで書く ― FloatSoda v0.4.0でいま作れるもの"
emoji: "🥤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "steamvr", "vrchat", "flutter"]
published: false
---

FloatSoda(フロートソーダ)は、SteamVR のオーバーレイを C# のコードだけで作れる UI フレームワークです。画面の形をそのままコードに書く宣言的 UI で、書き方は Flutter に合わせています。Unity は使わず、.NET のコンソールアプリとして動きます。

![VRChat のワールドに重ねて表示した FloatSoda のオーバーレイ。白い板の中央に青い四角がある](/images/floatsoda-overlay-v040/getting-started.jpg)
*VRChat の上に重ねて表示した FloatSoda のオーバーレイ*

SteamVR を起動した状態で次のコードを実行すると、ダッシュボードに 400×200 の青い箱が表示されます。

```csharp:Program.cs
using FloatSoda;
using FloatSoda.Widgets;
using FloatSoda.Widgets.Layout;
using FloatSoda.Widgets.Paint;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using SkiaSharp;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddFloatSoda();

using var host = builder.Build();
var app = host.Services.GetRequiredService<FloatSodaApp>();

Widget root = new Center
{
    Child = new ColoredBox
    {
        Color = SKColors.CornflowerBlue,
        Child = new SizedBox { Width = 400, Height = 200 }
    }
};

app.CreateWindow(new DashboardWindow { Title = "HelloWorld", Child = root });

await host.RunAsync();
```

「中央に(`Center`)、青く塗った(`ColoredBox`)、400×200 の箱(`SizedBox`)」のように、コードの入れ子がそのまま画面の構成になります。この記事では、2026 年 10 月時点の最新版 v0.4.0 で作れるものと、まだ作れないものを紹介します。あわせて、Unity なしで動く理由になった設計判断と、始め方を説明します。

:::message
FloatSoda は Alpha 段階で、API は予告なく変わります。記事のコードは、NuGet の v0.4.0(2026-09-22 公開)でビルドが通ることを確かめています。
:::

- リポジトリ: https://github.com/sumx21t-3310/FloatSoda (MIT ライセンス)
- ドキュメント: https://floatsoda.sumx21t.com/
- NuGet: https://www.nuget.org/packages/FloatSoda/

## FloatSoda が向いている人

FloatSoda は、3 種類の作り手を想定しています。それぞれにとって変わることは、次のとおりです。

| 作り手 | FloatSoda で変わること |
|---|---|
| AI と一緒に、自分用のツールを作りたい VRChat ユーザー | UI がすべて C# のコードなので、AI がそのまま書いて直せます。自分でコードを書けなくても始められます |
| Booth でオーバーレイ作品を売りたいクリエイター | Unity でワールドやギミックを作れる力があれば足ります。新しく覚えるのは、宣言的 UI の考え方だけです |
| uGUI を使いたくないエンジニア | シーンやプレハブの YAML がないため、diff、PR のレビュー、grep をそのまま使えます |

Unity でオーバーレイを作った経験がある人に向けて、違いを 3 つ挙げます。

- シーン、プレハブ、`.meta` ファイルがない。Hierarchy に当たるのは、コードに書いた Widget の入れ子
- `text.text = ...` のようにオブジェクトを書き換える代わりに、状態を変えて `SetState()` を呼ぶ。すると UI が作り直される
- 配布するときは、普通の .NET アプリと同じ手順で exe をビルドする

## v0.4.0 で作れるもの

v0.4.0 では、表示するだけのオーバーレイ、時計のように変わり続ける表示、レーザーで押すパネルを作れます。ここでは、置き場所、状態、入力、表示部品の順に紹介します。

### オーバーレイを置く場所は 3 種類から選ぶ

SteamVR のオーバーレイには 3 種類の置き方があり、FloatSoda ではウィンドウの型で選びます。中に入れる Widget は、どの型でも同じものを使えます。

| ウィンドウの型 | 置き方 | ポインタ入力 |
|---|---|---|
| `DashboardWindow` | SteamVR のダッシュボードにタブとして出す | 届く |
| `WorldSpaceWindow` | 部屋の中の決まった位置に固定する | 届かない |
| `DeviceTrackedWindow` | HMD やコントローラーに追従させる | 届かない |

```csharp
using FloatSoda.OVR.Overlay; // TrackedDevice

// ダッシュボード
app.CreateWindow(new DashboardWindow { Title = "MyDashboard", Child = root });

// 部屋の中に固定する。Position を省くと、前方 1m・高さ 1m に出る
app.CreateWindow(new WorldSpaceWindow { Title = "MyWorld", Child = root });

// 左コントローラーに追従させる
app.CreateWindow(new DeviceTrackedWindow { Title = "MyHand", Child = root, Target = TrackedDevice.LeftController });
```

同じ時計の Widget でも、ダッシュボードに置けば置き時計になり、左コントローラーに追従させれば腕時計になります。1 つのアプリから、複数のオーバーレイを同時に出せます。

### 状態を変えると画面が作り直される

状態を持つ Widget は、Flutter と同じく `StatefulWidget` と `State` の組で書きます。`SetState()` の中で状態を変えると、FloatSoda が変わった部分だけを作り直して描き直します。

```csharp:WatchWidget.cs
using FloatSoda.Elements;
using FloatSoda.Widgets;

public record WatchWidget : StatefulWidget<WatchWidget>
{
    public override State<WatchWidget> CreateState() => new WatchState();
}

public class WatchState : State<WatchWidget>
{
    private Timer? _timer;
    private string _time = "00:00:00";

    public override void InitState()
    {
        // 1 秒ごとに時刻を更新する
        _timer = new Timer(_ => SetState(() => _time = DateTime.Now.ToString("HH:mm:ss")),
            null, dueTime: 0, period: 1000);
    }

    public override Widget Build(IBuildContext context) => new Text(_time);
}
```

この Widget を `DeviceTrackedWindow` で左コントローラーに追従させると、手首に時計が出ます。

![左手の甲の上に浮かぶ白い丸い板に、時刻が表示されている](/images/floatsoda-overlay-v040/watch.gif)
*左コントローラーに追従させた時計。1 秒ごとに表示が進む*

### タップ・ドラッグ・ホバーはダッシュボードで受け取れる

`GestureDetector` で囲んだ部分は、タップとドラッグを受け取れます。ポインタが乗ったときと離れたときは、v0.4.0 で追加された `PointerRegion` で受け取れます。

```csharp:CounterWidget.cs
using FloatSoda.Elements;
using FloatSoda.Widgets;
using FloatSoda.Widgets.Gesture;
using FloatSoda.Widgets.Layout;
using FloatSoda.Widgets.Paint;
using SkiaSharp;

public record CounterWidget : StatefulWidget<CounterWidget>
{
    public override State<CounterWidget> CreateState() => new CounterState();
}

public class CounterState : State<CounterWidget>
{
    private int _count;

    // 押すたびに数が 1 増える青いボタン
    public override Widget Build(IBuildContext context) => new GestureDetector
    {
        OnTap = () => SetState(() => _count++),
        Child = new ColoredBox
        {
            Color = SKColors.CornflowerBlue,
            Child = new SizedBox
            {
                Width = 200,
                Height = 100,
                Child = new Center { Child = new Text($"{_count}") }
            }
        }
    };
}
```

![SteamVR のダッシュボードに出したカウンターのパネルを、コントローラーのレーザーで押して数を増減させている](/images/floatsoda-overlay-v040/counter.gif)
*リポジトリのサンプルにあるカウンター。レーザーでボタンを押すと数が変わる*

入力が届くのは、v0.4.0 ではダッシュボードのオーバーレイだけです。SteamVR はダッシュボード上のレーザーポインターをマウスイベントとして送ります。FloatSoda はこれを当たり判定に使っています。部屋に固定したオーバーレイとデバイスに追従するオーバーレイに `GestureDetector` を置いた場合、コンパイルは通りますがコールバックは呼ばれません。

### v0.4.0 で画像・アイコン・文字の書式が増えた

v0.4.0 では、表示に使う部品がまとめて増えました。主なものは次のとおりです。

- レイアウト: `Padding`、`Container`、`Stack`、`Wrap`、`Expanded` など 19 種
- 描画: `DecoratedBox`、`Opacity`、`Transform`、`RotatedBox`、`Visibility`、`Offstage`、`RepaintBoundary`
- 画像: `Image` の読み込みが非同期になった。読み込み中と失敗したときに出す Widget を、`LoadingBuilder` と `ErrorBuilder` で指定できる
- アイコンとフォント: アイコンフォントの文字を出す `Icon` と、フォントを指定する `FontProvider`
- 文字の書式: `TextStyle` と、子孫の `Text` へ書式を引き継ぐ `DefaultTextStyle`
- 非同期の結果: `Task<T>` の状態(読み込み中、完了、失敗)で表示を切り替える `TaskBuilder<T>`

アニメーション(`AnimationController`、`FadeTransition`)も使えます。実装済みの Widget の一覧は、ドキュメントの [WidgetSystem](https://floatsoda.sumx21t.com/widgetsystem/) にあります。

## まだ作れないもの

次の機能は、v0.4.0 では使えません。予定の Phase は、GitHub のマイルストーンに合わせています。

| 機能 | v0.4.0 での状態 | 予定 |
|---|---|---|
| スクロール(`ListView`、`GridView`、`SingleChildScrollView`) | 公開 API にない | Phase 3 |
| `Button` などの既製の部品 | ボタンは `GestureDetector` で組み立てる | Phase 5 |
| 部屋に固定・デバイスに追従するオーバーレイへのポインタ入力 | 表示だけに使える | Phase 3.5 |
| VR キーボードの呼び出し | ない | Phase 3.5 |
| テキスト入力 | ない | Phase 4 |

既知の問題も 1 つあります([#285](https://github.com/sumx21t-3310/FloatSoda/issues/285))。`Size` を指定しないウィンドウは、起動後に中身が大きくなっても、SteamVR 上の表示が最初の大きさのまま変わりません。中身の大きさが変わるウィンドウには、`Size` を指定してください。

```csharp
using SkiaSharp; // SKSize

app.CreateWindow(new DashboardWindow { Title = "Album", Size = new SKSize(1000, 600), Child = root });
```

まとめると、写真フォルダを眺めるアルバムのように、スクロールが前提のアプリはまだ作れません。時刻や配信コメントを出す表示用の HUD と、ダッシュボードに置くボタンのパネルは、v0.4.0 の機能で作れます。

## Unity なしで動く理由: オーバーレイにはテクスチャを 1 枚渡せばよい

FloatSoda を作り始める前に立てた仮説は、「テクスチャを転送できれば、描画には何を使ってもよい」というものでした。この仮説が、Unity を使わない構成の土台です。

仮説のもとになったのは、kurohuku さんの本『Unity でつくる SteamVR オーバーレイアプリケーション』です。この本の「カメラ映像の表示」の章では、Unity の RenderTexture から `GetNativeTexturePtr()` でテクスチャのポインタを取ります。それを `Texture_t` に入れて、OpenVR の `SetOverlayTexture()` へ渡します。オーバーレイに絵を出すときに OpenVR が受け取るのは、この 1 枚のテクスチャです。つまり、Unity はテクスチャを作る手段の 1 つにすぎません。

FloatSoda は、Unity の代わりに SkiaSharp で描き、OpenGL のテクスチャとして渡します。`Texture_t` の形は Unity の場合と同じです。違うのは、`eType` が `DirectX` から `OpenGL` に変わる点だけです。

```csharp:OverlayWindow.cs(抜粋)
var texture = new Texture_t
{
    handle = renderer.GetTextureHandle(),
    eType = ETextureType.OpenGL,
    eColorSpace = EColorSpace.Auto,
};

// 内部で OpenVR の SetOverlayTexture() を呼ぶ
Overlay.Texture.FromTexture_t(texture);
```

仮説を確かめられたのは、最初のコミット(2026-04-17)の 9 日後です。2026-04-26 に、OpenTK と SkiaSharp で描いた板が SteamVR の中に表示されました。

![SteamVR の VR ビューの中央に、左上に小さな四角を描いた白い板が浮かんでいる](/images/floatsoda-overlay-v040/first-overlay-2026-04-26.png)
*2026-04-26、初めて SteamVR に表示できたオーバーレイ*

この設計のおかげで、FloatSoda のアプリはゲームエンジンを持たない .NET アプリのまま動きます。内部では、メインスレッドでレイアウトと描画命令の記録をすませ、その結果をレンダースレッドへ渡します。レンダースレッドは OpenGL のテクスチャに描いて、OpenVR へ渡します。詳しくはドキュメントの [Architecture](https://floatsoda.sumx21t.com/architecture/) にあります。

### UI の組み立て方は Flutter から写した

テクスチャを作る前の、UI を組み立てる部分は Flutter の設計に合わせています。Flutter と同じく、UI を Widget、Element、RenderObject の 3 つのツリーで管理します。

- Widget: 開発者が書く UI の設計図。状態が変わるたびに作り直される
- Element: 前回と今回の Widget を比べ、変わった部分だけを更新する
- RenderObject: レイアウトと描画を受け持つ。変更があった部分だけを計算し直す

Widget の名前と役割も Flutter に合わせています。そのため、`Center`、`SizedBox`、`GestureDetector` は Flutter と同じ意味で使えます。

Flutter の内部構造を学ぶには、fastriver さんの本『再実装 Flutter』が役に立ちます。この本は、Skia での描画、Layer のツリー、RenderObject、Element と Widget のツリーを順に実装しながら、Flutter が内部で何をしているかを説明しています。

## AI に書かせるときは llms.txt を渡す

ドキュメントサイトでは、AI が読むためのテキストも配布しています。コーディングエージェントに FloatSoda のコードを書かせる場合は、次のどちらかの URL を渡してください。

- 索引: https://floatsoda.sumx21t.com/llms.txt
- 全文: https://floatsoda.sumx21t.com/llms-full.txt

ドキュメントは、AI が読んで正しいコードを書けることを基準に手入れしています。新しい public API を追加する PR は、マージ前に「ジュニアコーダーテスト」を通します。このテストでは、ソースを見せずにドキュメントだけを AI に渡し、その API を使うお題を解かせます。AI が API を見つけられなかったり、使い方を誤ったりしたら、ドキュメントか API の形を直します。

## 始め方

必要なものは、.NET 10 SDK と SteamVR です。SteamVR は、アプリを実行する前に起動しておいてください。

```bash
dotnet new console -n MyOverlayApp
```

```bash
cd MyOverlayApp
```

```bash
dotnet add package FloatSoda
```

`Program.cs` を冒頭のコードに書き換えて `dotnet run` を実行すると、ダッシュボードに青い箱のタブが追加されます。SteamVR を終了すると、アプリも終了します。

### リポジトリのサンプルを動かす

リポジトリの `samples/` フォルダには、目的別のサンプルプロジェクトがあります。使い方を手早く知りたいときは、記事のコードより先にサンプルを動かすと分かりやすくなります。

| サンプル | 内容 | SteamVR |
|---|---|---|
| `FloatSoda.Samples.OverlayApp` | レイアウト、時計、アニメーション、カウンター、ドラッグをまとめたデモ。3 種類のオーバーレイを同時に出す | 必要 |
| `FloatSoda.Samples.GettingStarted` | 冒頭のコードとほぼ同じ最小のアプリ | 必要 |
| `FloatSoda.Samples.PointerRegion` | ホバー、押下、取り消しの状態を画面に出す入力のデモ | 必要 |
| `FloatSoda.Samples.PrimitiveOverlay` | Widget を使わず、低レベルの API だけでオーバーレイを出す | 必要 |
| `FloatSoda.Samples.PaintingSample` | Widget のツリーを PNG に書き出す | 不要 |
| `FloatSoda.Samples.Align`、`Image`、`GestureDetector` など | Widget ごとの使い方。各フォルダの `README.md` がチュートリアルになっている | 必要 |

総合デモは、次の手順で起動できます。

```bash
git clone https://github.com/sumx21t-3310/FloatSoda.git
```

```bash
cd FloatSoda
```

```bash
dotnet run --project samples/FloatSoda.Samples.OverlayApp
```

HMD をかぶらずにレイアウトの結果だけを確かめたいときは、`PaintingSample` を使ってください。SteamVR も HMD もない状態で、デスクトップに PNG が保存されます。

## これからの予定とフィードバックの送り先

開発は Phase という単位で進めています。Phase は「何が作れる段階か」を表す区切りであり、NuGet のバージョン番号とは対応しません。対応するのは、Phase 7 と 1.0.0 だけです。

| Phase | 内容 | 作れるようになるアプリ | Issue(未完了 / 完了) |
|---|---|---|---|
| Phase 1 | 入力の基盤(当たり判定、ポインタ、ジェスチャー) | 操作できるパネル | 0 / 7(完了) |
| Phase 2 | Flutter の基本 Widget に当たる表示部品をそろえる | 表示の多い HUD、字幕オーバーレイ | 6 / 23 |
| Phase 3 | スクロールとアニメーションの充実 | チャットビューア、画像ギャラリー、フレンド一覧 | 16 / 1 |
| Phase 3.5 | SteamVR まわりの API を確定させる | 通知、VR キーボード、触覚フィードバックを使うツール | 17 / 0 |
| Phase 4 | Hooks、テキスト入力、API の安定化 | VR の中のメモ帳など、文字を入力するアプリ | 10 / 1 |
| Phase 5 | デザインシステム(Cream / FizzyPop) | テーマを選べる UI のアプリ | 32 / 0 |
| Phase 6 | 開発体験の改善(Storybook など) | デスクトップに常駐し、VR でも使うツール | 6 / 0 |
| Phase 7 | 安定版(1.0) | 実用的なオーバーレイ全般 | 2 / 0 |

Issue の数は 2026-10-02 時点のものです。各 Phase の範囲は、GitHub の[マイルストーン](https://github.com/sumx21t-3310/FloatSoda/milestones)にあります。

Alpha の間は API の形がまだ固まっていないので、使った人の声で API を変えられます。使ってみてつまずいた点、分かりにくい API、欲しい Widget があれば、[GitHub の Issue](https://github.com/sumx21t-3310/FloatSoda/issues) へ送ってください。

## 参考資料

- [Unity でつくる SteamVR オーバーレイアプリケーション](https://zenn.dev/kurohuku/books/a082c5728cc1f6)(kurohuku さん): Unity と OpenVR でオーバーレイを作るチュートリアルです。腕時計アプリを作りながら、オーバーレイの作成、デバイスへの追従、ダッシュボード、コントローラーの入力までを説明しています。FloatSoda のテクスチャ転送の仮説は、この本の「カメラ映像の表示」の章から生まれました。
- [再実装 Flutter](https://zenn.dev/fastriver/books/reimpl-flutter)(fastriver さん): Flutter を 1 から作り直しながら、Flutter が内部で何をしているかを説明する本です。FloatSoda の 3 つのツリーを理解するときの手がかりになります。
- [OVRSharp](https://github.com/OVRTools/OVRSharp): OpenVR の .NET ラッパーです。FloatSoda の OpenVR まわりのクラス設計は、OVRSharp を参考にしています。
