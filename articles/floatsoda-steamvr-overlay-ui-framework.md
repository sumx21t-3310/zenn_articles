---
title: "SteamVRのオーバーレイをFlutterのように書けるC#製UIフレームワーク「FloatSoda」を作っている"
emoji: "🥤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "steamvr", "vrchat", "flutter"]
published: false
---

## はじめに

FloatSoda(フロートソーダ)は、C# / .NET 製の UI フレームワークです。SteamVR のオーバーレイを Flutter のような宣言的 UI で書けます。UI はシーンファイルもプレハブも持たず、すべて C# コードで完結します。私が 2026 年 4 月から個人で開発しており、NuGet に公開済みです(執筆時点で v0.3.1)。

次のコードを実行すると、SteamVR のダッシュボードに 400×200 の青い箱が浮かびます。ゲームエンジンは起動していません。ただの .NET コンソールアプリです。

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

この記事では、FloatSoda で今日動くもの、内部の仕組み、なぜこの形にしたのか、そして現在地とこれからを紹介します。

:::message
FloatSoda は Alpha 段階です。簡単なアプリは動きますが、API は予告なく変わります。この記事のコードは執筆時点(2026 年 9 月、v0.3.1)のものです。最新の情報は[ドキュメントサイト](https://floatsoda.sumx21t.com/)と [GitHub](https://github.com/sumx21t-3310/FloatSoda) を参照してください。
:::

- リポジトリ: https://github.com/sumx21t-3310/FloatSoda
- ドキュメント: https://floatsoda.sumx21t.com/
- NuGet: https://www.nuget.org/packages/FloatSoda/

## なぜ作ったか

### きっかけは、フレンドの C++ フルスクラッチ

FloatSoda が生まれたきっかけは、技術トレンドでも仕事の要件でもなく、VRChat のフレンドとの会話でした。

そのフレンドは元グラフィックエンジニアで、これから Booth で商品を販売しようとしていたところでした。私は「VRChat で撮った写真を、オーバーレイでアルバムのように見られるツールがあれば売れるのではないか」と提案しました。VR で撮った写真を VR の中で見返したいというのは、自然な欲求です。

しばらくして、フレンドは実際に作りはじめました。ところがある日、「Claude のトークンをめちゃくちゃ消費してしまった」と言いました。何をそんなに書かせているのか聞いたところ、C++ でフルスクラッチしていました。私は当然 Unity で作っていると思い込んでいたため、驚きました。

でも、笑ったあとで気づきました。フレンドが間違っていたのではなく、まともな選択肢が無かったのです。

### 自作オーバーレイの選択肢には、空白がある

SteamVR には「オーバーレイ」という仕組みがあります。どの VR アプリの上にも重ねて表示できる、VR 空間に浮かぶ 2D のパネルです。時計を表示したり、デスクトップを映したり、配信のコメントを流したりできます。XSOverlay や OVR Toolkit といった既製ツールがあり、多くの人が使っています。

既製ツールは素晴らしいのですが、「自分だけの」オーバーレイを作ろうとした瞬間、話が変わります。

| 選択肢 | 現実 |
|---|---|
| Unity | パネル 1 枚のためにゲームエンジンをまるごと持ち出す。UI は uGUI か UI Toolkit で組む |
| C++ でフルスクラッチ | SteamVR の API(OpenVR)を直接叩き、描画も自前で書く |
| Qt などのデスクトップ UI | 自力で OpenVR に繋ぎ込む |

Web には React があり、モバイルには Flutter や SwiftUI があり、デスクトップには WPF や Electron があります。UI フレームワークがこれだけ飽和している時代に、VR オーバーレイという領域だけ、ぽっかり空白なのです。「時計を 1 個浮かべたい」と「ゲームエンジンを起動する」の間に、何もありません。

そこで、自分で作ろうと考えました。

### 仮説:テクスチャさえ転送できれば、何でも描ける

UI フレームワークの自作は、普通に考えると無謀です。作れると確信できる根拠が必要でした。

決め手になったのは、Unity で SteamVR オーバーレイを作る方法を解説した [kurohuku さんの Zenn 本](https://zenn.dev/kurohuku/books/a082c5728cc1f6)です。この本では、Unity の RenderTexture を OpenVR に渡す実装が丁寧に解説されています。これを読んで気づきました。OpenVR のオーバーレイ API は、突き詰めると「GPU テクスチャのハンドルを 1 個受け取る」だけなのです。`SetOverlayTexture` という関数に描き終わったテクスチャを渡すと、OpenVR がそれを VR 空間に貼って表示します。Unity は「テクスチャを作る手段」の 1 つにすぎませんでした。

つまり、テクスチャさえ転送できれば、描画には何を使ってもよいのです。この仮説が FloatSoda の出発点です。仮説は first commit から 11 日後に、自分で描いた四角形が VR 空間に浮かんだことで現実になりました。

名称は、オーバーレイが VR 空間に「浮かんでいる」ため Float としました。また、設計の元ネタである Flutter と同じ FL で始まる言葉を探し、FloatSoda にしました。

## 今日動くもの

### 宣言的 UI で書く

FloatSoda の書き味は「宣言的 UI」です。言葉で説明する前に、まず従来の一般的な UI コードを見てください。

```csharp
// 命令型(擬似コード)
var window = CreateWindow();
var box = new Box();
box.Width = 400;
box.Height = 200;
box.Color = Colors.Blue;
window.Add(box);
box.CenterIn(window);
```

箱を作り、サイズを設定し、色を塗り、ウィンドウに追加して、中央に寄せます。これは手順の羅列です。どのような画面になるかは、頭の中でコードを実行してみないと分かりません。

宣言的 UI は、これを逆転させます。手順ではなく「中央に、青い箱がある」という画面の完成形をそのまま書きます。冒頭のコードをもう一度見てください。

```csharp
Widget root = new Center
{
    Child = new ColoredBox
    {
        Color = SKColors.CornflowerBlue,
        Child = new SizedBox { Width = 400, Height = 200 }
    }
};
```

「中央に(Center)、青く塗った(ColoredBox)、400×200 の箱(SizedBox)」。コードがそのまま画面の説明文になっています。Web 開発で素の JavaScript による DOM 操作から React に移ったときの気持ちよさ、と言えば伝わるでしょうか。見た目とコードの構造が一致するので、読みやすく、変更に強くなります。

Flutter を書いたことがある方なら、初見で読めると思います。`Center`、`ColoredBox`、`SizedBox` は Flutter と同じ名前で、同じ役割です。

### オーバーレイの置き方は 3 種類

SteamVR のオーバーレイには置き方が 3 種類あり、FloatSoda ではそれぞれをウィンドウの型として区別します。同じ Widget ツリーを、どの置き方でも表示できます。

| ウィンドウの型 | 置き方 | ポインタ入力 |
|---|---|---|
| `DashboardWindow` | SteamVR ダッシュボードのタブとして表示する | 届く |
| `WorldSpaceWindow` | ワールド座標に固定する(部屋の壁のように、そこに在り続ける) | 届かない |
| `DeviceTrackedWindow` | HMD やコントローラーなどのデバイスに追従する | 届かない |

```csharp
// ダッシュボード
app.CreateWindow(new DashboardWindow { Title = "MyDashboard", Child = root });

// ワールド座標固定。Position を省略すると前方 1m・高さ 1m
app.CreateWindow(new WorldSpaceWindow { Title = "MyWorld", Child = root });

// 左コントローラーに追従
app.CreateWindow(new DeviceTrackedWindow { Title = "MyHand", Child = root, Target = TrackedDevice.LeftController });
```

同じ時計ウィジェットを、ダッシュボードに置けば置き時計になり、左手に追従させれば腕時計になります。UI の記述と、VR 空間への置き方が分かれています。1 つのアプリから複数のオーバーレイを同時に管理できます。

ポインタ入力が届くのは、執筆時点ではダッシュボードだけです。SteamVR がダッシュボード上のレーザーポインターをマウスイベントとして送るため、FloatSoda はそれをそのままヒットテストへ渡せます。ワールド座標固定とデバイス追従のオーバーレイには、コントローラーのレイからポインタ座標を作る仕組みがまだありません。

### 腕時計:状態が変わると画面が追従する

静止した箱を表示するだけでは、スクリーンショットと変わりません。動くものを見てみましょう。

Flutter と同じく、状態を持つウィジェットは `StatefulWidget` と `State` の組で書きます。`SetState()` で「状態が変わった」と伝えると、フレームワークが画面の差分だけを更新します。

```csharp
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
        _timer = new Timer(_ => SetState(() => _time = DateTime.Now.ToString("HH:mm:ss")),
            null, dueTime: 0, period: 1000);
    }

    public override Widget Build(IBuildContext context) => new Text(_time);
}
```

これを `DeviceTrackedWindow` で左コントローラーに追従させると、腕時計になります。VR の中で手首を返すと時計があり、秒が進みます。開発の最初の 3 ヶ月は、この腕時計が動くことをマイルストーンのゴールにしていました。

### 押せるボタン:GestureDetector で組む

v0.3.0 から、オーバーレイが「押せる」ようになりました。Flutter の設計に沿ってポインタ入力、ヒットテスト、ジェスチャー認識を実装しました。そのため、`GestureDetector` でタップとパン(ドラッグ)を受け取れます。

執筆時点では `Button` のような既製の部品はまだ無いため、ボタンは `GestureDetector` で組み立てます。サンプルのカウンターアプリから、ボタンを作るメソッドを抜粋します。

```csharp:CounterWidget.cs(抜粋)
private Widget BuildButton(string label, Action onPressed)
{
    return new GestureDetector
    {
        OnTap = () => SetState(onPressed),
        Child = new ClipRoundRect
        {
            BorderRadius = BorderRadius.All(Radius.Circular(15)),
            Child = new SizedBox
            {
                Width = 120,
                Height = 80,
                Child = new ColoredBox
                {
                    Color = SKColors.CornflowerBlue,
                    Child = new Center
                    {
                        Child = new RichText
                        {
                            Text = new TextSpan(label)
                            {
                                Style = new TextStyle
                                {
                                    Color = SKColors.WhiteSmoke,
                                    FontSize = 50,
                                    FontWeight = 700
                                }
                            }
                        }
                    }
                }
            }
        }
    };
}
```

コントローラーのレーザーでボタンを狙ってトリガーを引くと、`OnTap` が呼ばれ、`SetState()` でカウントが増えます。表示するだけだったオーバーレイが、これで「操作できるアプリ」になりました。

### まだできないこと

正直に書いておきます。執筆時点で、次のものはまだ使えません。

- スクロール系のウィジェット(`ListView` / `GridView` / `SingleChildScrollView`)。Phase 3 で実装します
- `Button` などの既製 UI 部品。3 層構成のデザインシステム(`FloatSoda.UI` / `Cream` / `FizzyPop`)は Phase 5 の予定です
- ワールド座標固定とデバイス追従のオーバーレイへのポインタ入力。Phase 3.5 で接続します
- テキスト入力(VR キーボード連携)。Phase 4 の予定です

一方で、レイアウト系(`Row` / `Column` / `Stack` / `Padding` / `Container` / `Wrap` / `Expanded` など)、描画系(`DecoratedBox` / `Opacity` / `Transform` / `Clip*`)、テキスト、画像、アイコン、アニメーション(`AnimationController` / `FadeTransition`)は使えます。実装済みのウィジェット一覧は[ドキュメントの WidgetSystem](https://floatsoda.sumx21t.com/widgetsystem/) にあります。

## 中身:Flutter の三つのツリーを C# に写す

### Widget / Element / RenderObject

FloatSoda は、Flutter の「三つのツリー」というモデルをそのまま踏襲しています。

```text
Widget ──────▶ Element ──────▶ RenderObject ──────▶ Layer
不変の設計図    生きている仲介者    レイアウトと描画      合成結果
(record)       (差分を判断)       (制約↓ サイズ↑)      (SKPicture)
```

- **Widget** は、みなさんが書く UI の設計図です。「中央に配置して、400×200 で、青く塗る」という宣言そのもの。C# の `abstract record` として実装していて、イミュータブルで、フレームごとに使い捨てられる軽い存在です
- **Element** は、画面の裏でずっと生き続ける仲介者です。Widget は毎回作り直されますが、Element は「前のフレームの Widget と今の Widget、どこが変わった?」を判断して、必要な箇所だけを更新します
- **RenderObject** は、レイアウト計算と描画を実際に担う層です。「親から制約が降りてきて、子がサイズを返す」という Flutter と同じレイアウトプロトコルで動きます

なぜ三層に分けるのでしょうか。宣言的 UI は「毎フレーム UI を全部作り直す」という顔をしたプログラミングモデルです。でも本当に全部作り直したら重くて話になりません。だから、作り直される軽い層(Widget)と、生き残って差分だけ更新する重い層(Element、RenderObject)に分けます。これが三つのツリーの本質です。

差分更新は徹底しています。`SetState()` で変更のあった Element は dirty リストに登録され、`BuildOwner` というスケジューラが親から順に再ビルドします。RenderObject 側も同じで、プロパティが変わると `MarkNeedsLayout` や `MarkNeedsPaint` が呼ばれ、影響範囲の境界までしか波及しません。何も変わらないフレームでは、レイアウトもペイントも一切走りません。VR オーバーレイは常駐アプリなので、アイドル時に CPU を食わないことは快適さに直結します。

### 出口は、テクスチャ 1 枚の転送

RenderObject が描いた結果は、即座に画面に出るのではなく、いったん「描画命令の記録」としてレイヤーツリーに積まれます。実体は SkiaSharp の `SKPicture` であり、録画されたドローコールの束です。

```text
 メインスレッド                              レンダースレッド
 ┌────────────────────────┐                ┌──────────────────────────────┐
 │ Build → Layout → Paint │    Clone()     │ Skia on OpenGL で描画          │
 │         ↓              │ ─────────────▶ │         ↓                    │
 │   レイヤーツリー          │  (不変の        │ SetOverlayTexture(GL ハンドル) │
 └────────────────────────┘   スナップショット) └──────────────────────────────┘
```

このような構造にする理由は、スレッドを分けるためです。レイアウトやビルドをするメインスレッドと、OpenGL を呼び出して OpenVR にテクスチャを渡すレンダースレッドを分離しています。OpenGL のコンテキストは 1 つのスレッドに縛られるため、GL を操作するコードを 1 つのスレッドに閉じ込め、タスクはキュー経由で渡します。このとき、メインスレッド側のレイヤーツリーをそのまま渡すとデータ競合が起きるため、`Clone()` で不変のスナップショットを作成して渡します。

レンダースレッドは受け取ったレイヤーツリーを GL 上の Skia サーフェスに描き、最後に GL のテクスチャハンドルを OpenVR の `SetOverlayTexture` に渡します。どれだけ立派な三つのツリーを組んでも、最後の出口は「テクスチャを 1 枚転送する」だけです。最初の仮説の上に、全部が乗っています。

### なぜ Flutter の設計を写したのか

理由は 2 つあります。

1 つめは、教科書があったことです。Zenn に [fastriver さんの「再実装Flutter」](https://zenn.dev/fastriver/books/reimpl-flutter)という本があります。Flutter のウィジェットシステムを自分の手で再実装しながら内部構造を理解する本です。この本の存在が教えてくれたのは、Flutter のアーキテクチャはブラックボックスではなく、個人が理解して再構築できる程度にきれいに整理された設計だということでした。三つのツリーは各層が疎結合なので、下の層から順番に、動くものを確かめながら積み上げられます。実際の実装順も、Flutter の解説とは逆に、レイヤーツリー(5 月)→ RenderObject(6 月前半)→ Widget / Element(6 月後半〜)と下から登りました。

2 つめは、開発を進めてから気づいたことです。Flutter のアーキテクチャを踏襲していると、LLM に実装を手伝わせたときにズレた実装が返ってきません。世界中の Flutter のコードとドキュメントを学習しているモデルにとって、「これは Flutter の RenderObject に相当します」という一言は、分厚い仕様書に匹敵します。RenderObject を実装していた頃、「Flutter の RenderPadding に相当するものを」と頼むだけで、ほとんど直さなくていいコードが返ってくることに気づきました。UI フレームワークには、`Padding` や `Align` のように構造がほぼ同じウィジェットを何十個も実装するフェーズが必ず来ます。ここを AI に任せられる安心感は、独自アーキテクチャでは得られなかったものです。

### C# だから足せたもの:オブジェクト初期化子

Flutter の UI コードが読みやすいのは、Dart に名前付き引数があるからです。C# で同じことをコンストラクタでやろうとすると、どうしても野暮ったくなります。悩んだ末に再発見したのが、C# のオブジェクト初期化子でした。

```dart
// Flutter(Dart の名前付き引数)
SizedBox(
  width: 100,
  height: 100,
  child: ColoredBox(color: Colors.red),
)
```

```csharp
// FloatSoda(C# のオブジェクト初期化子)
new SizedBox
{
    Width = 100,
    Height = 100,
    Child = new ColoredBox { Color = SKColors.Red }
}
```

プロパティを `init` アクセサで宣言し、コンストラクタ引数を使わずに `new SizedBox { Width = 100, Child = ... }` と書くと、C# のコードが Dart の名前付き引数とほぼ同じ見た目になります。必須のプロパティには `required` を付ければ、渡し忘れはコンパイルエラーになります。

これを「オブジェクト初期化子ファースト」という設計規約に昇格させました。すべてのウィジェットはコンストラクタ引数を持たず、子が 1 つなら `Child`、複数なら `Children` です。規約は [docs/APIDesign](https://floatsoda.sumx21t.com/apidesign/) に文書化してあります。

もう 1 つの主役が `record` です。Widget を `record` として宣言すると、等値比較がただで手に入ります。「前の Widget と今の Widget が等しいなら更新不要」という差分更新の判定が、言語機能だけで書けます。`Offset` や `BoxConstraints` のような値型は `readonly record struct` にして、値セマンティクスとイミュータビリティとゼロアロケーションを一度に確保しました。

`init` は生成後に書き換えられないこと、`required` は必須であること、`On` プレフィックスの `Action?` はイベントであることを示します。このように、API の使い方がドキュメントを読まなくても型シグネチャから読み取れます。個人開発では詳細なマニュアルを維持する余力がないため、型によって使い方を示すのが最も誠実な方法だと考えています。

### C# だから苦労したこと:Mixin と、透明なオーバーレイ

Flutter の内部実装は、Dart の Mixin という言語機能に大きく依存しています。これは、「子を 1 つ持つ振る舞い」や「子を複数持つ振る舞い」をクラスに組み込める機能です。C# は単一継承であり、Mixin がありません。

FloatSoda では最終的に、この問題を継承ではなくコンポジションで解決しました。`SingleChildContainer<T>` や `MultiChildrenCollection<T>` という「子を保持する部品」を作り、各 RenderObject がそれをフィールドに持って委譲します。この形に落ち着くまでは、継承階層と型パラメーターの板挟みになり、バグの温床になっていました。

もう 1 つ、開発を通してずっと苦しめられたのが「オーバーレイが透明になる」現象です。普通の GUI 開発なら、バグっても「何かおかしな絵」が出ます。ところが VR オーバーレイの描画バグは、かなりの確率で「何も出ない」に化けます。テクスチャが真っ透明のまま転送されて、例外も出ない、ログも正常、ただ無。原因は GL コンテキストのカレント化忘れ、レイヤーの Layout 呼び忘れ、dirty フラグの伝播漏れと毎回違うのに、症状は全部同じ顔でやってきます。差分更新は「再描画をサボる」仕組みなので、サボってはいけないところをサボると、絵が消えるのです。

## AI に書かせやすい、を設計目標にする

### 想定している作り手

FloatSoda は、次の 3 タイプの作り手を想定して設計しています。

1. **AI と一緒に「自分用のツール」を作りたい VRChatter**。コードが書けなくても構いません。AI にコードを生成させるバイブコーディングで完結することを設計目標にしています
2. **Booth でオーバーレイ作品を売りたいクリエイター**。Unity でワールドやギミックを作れるなら、必要な知識は十分あります。新しく覚えるのは宣言的 UI の概念だけです
3. **uGUI を使いたくないエンジニア**。シーンとプレハブの YAML、GUID 参照、読めない diff、レビューできない UI 変更にうんざりしている人です

3 タイプに共通する土台が、「UI が 100% C# コードである」ことです。diff が読めて、PR でレビューできて、grep で探せて、コピー&ペーストで共有できて、AI がそのまま生成・修正できます。この物語の始まりは、フレンドが AI に C++ を書かせてトークンを溶かした事件でした。バイブコーディングは VRChat の文化圏でもう普通に起きています。だったらフレームワークの側が、AI に書かせやすい形をしているべきだと考えました。

### ジュニアコーダーテスト:ドキュメントだけで AI に作らせる

この思想を検証する仕組みが「ジュニアコーダーテスト」です。あえて性能を抑えた AI モデルに FloatSoda の公開ドキュメントだけを渡し、VRChat ユーザーからよくある依頼を提示してツールを作成させます。ソースコードは見せず、NuGet から導入した利用者と同じ条件にします。想定ユーザーが AI にコードを書かせる人であれば、ドキュメントの本当の読者は AI になります。そのため、AI がつまずいた箇所はすべてドキュメントや API の欠陥だと判断できます。

v0.0.3 の時点で初回を回したところ、想像以上に効きました。中量級のモデルに「友達がオンラインになったら出るトースト通知」を作らせたところ、使った API 約 20 個はすべて実在し、名前も形も正確でした。一方で、ドキュメントの不備どころか本物のバグが 2 件見つかりました。当時のドキュメントが推奨していたフレームレート制御の API が、オーバーレイアプリでは必ずクラッシュするものだったこと。そして、子ウィジェットを動的に取り除くと無限再帰で落ちること。人間の私なら踏まなかった地雷を、何も知らない AI が正直に踏み抜いてくれました。両方とも v0.1.0 で修正済みです。

このテストはリリース手順と、新しい public API を追加する PR の受け入れ条件に組み込んであります。最も価値が高いのは、AI がより自然な別の形の API を捏造したときに、マージ前に API をその直感へ寄せることです。「AI が誤用できない API」が、このフレームワークの品質基準です。

### ドキュメントサイトと llms.txt

ドキュメントは https://floatsoda.sumx21t.com/ で公開しています。AI が読み取ることを前提として、[llms.txt](https://floatsoda.sumx21t.com/llms.txt)(索引)と [llms-full.txt](https://floatsoda.sumx21t.com/llms-full.txt)(全文)も用意しています。コーディングエージェントに FloatSoda を利用させる際は、この URL を渡してください。

## 現在地とこれから

### Phase で進めている

開発は Phase 単位で進めています。Phase は「フレームワークとして何ができる段階か」を表す到達点であり、NuGet のバージョン番号とは連動しません。バージョンはリリースの通し番号として独立して上がり、同じ Phase 中に複数のバージョンが公開されることがあります。

| Phase | 内容 | 作れるようになるアプリ | 状況 |
|---|---|---|---|
| Phase 1 | 入力基盤(HitTest / Pointer / Gesture) | 操作できるパネル(`GestureDetector` で自作したボタン) | 完了(非ダッシュボードへのポインタ接続は Phase 3.5 へ) |
| Phase 2 | basic.dart 相当の表示系ウィジェット網羅 | リッチな HUD / 字幕オーバーレイ | 進行中 |
| Phase 3 | スクロールとアニメーションの充実 | チャットビューア、画像ギャラリー | 未着手 |
| Phase 3.5 | SteamVR API の確定(FloatSoda.OVR の公開 API を固める) | 通知・VR キーボード・触覚フィードバックまで使い切るユーティリティ | 未着手 |
| Phase 4 | Hooks・テキスト入力・API 安定化 | VR 内メモ帳などの入力を伴うアプリ | 未着手 |
| Phase 5 | Cream / FizzyPop デザインシステム完成 | テーマを選べる実用 UI アプリ | 未着手 |
| Phase 6 | DX 向上(Storybook・manifest 自動生成・ライフサイクル) | デスクトップ常駐 + VR のハイブリッドツール | 未着手 |
| Phase 7 | 安定版リリース(1.0) | 実用オーバーレイ全般 | 未着手 |

各 Phase の詳細は [GitHub のマイルストーン](https://github.com/sumx21t-3310/FloatSoda/milestones)にあります。

マイルストーンの立て方には、1 つこだわりがあります。機能のリストではなく、「このバージョンで、どんなアプリが作れるようになるか」で区切ることです。「SetState が動く」ではなく「腕時計が作れる」。進捗が、そのまま誰かの作りたいものに繋がるようにしています。

Phase 2 では、Flutter の基本ウィジェットに相当する表示系 30 個あまりを、コーディングエージェントで量産しています。「Flutter のアナロジーがあると AI が迷わない」という仮説の、いちばん大きな検証実験でもあります。

### 目指す世界

日本の VRChat コミュニティには、独特の文化があります。アバター改変を支えるために、NDMF というフレームワークの上で Unity のエディタ拡張を書き、Booth で配布・販売する人たちがいます。プロのソフトウェアエンジニアではない人も含めて、C# を書き、読み、共有する文化圏がすでにあります。

VR オーバーレイのフレームワークを作るのであれば、その文化圏で使われている言語を採用するべきだと考えました。Rust や TypeScript ではなく C# を選んだのは、技術的な理由以上に、コミュニティを意識した設計判断です。

私が見たいのは、この文化がオーバーレイに広がった世界です。プログラマーでなくても、AI と一緒に自分だけのオーバーレイツールを作れて、便利なものができたら Booth で売れる。睡眠ログを取る人、イベント会場のマップを浮かべる人、推しの誕生日カウントダウンを腕に付ける人。ニーズが一人ぶんしかないアプリ、つまりオーダーメイドのアプリケーションが、あたりまえに生まれる世界です。最初にフレンドに提案した写真アルバムが、誰にでも作れるものになる。そういう世界のための土台になりたいと思っています。

## 使ってみてください

### 最小構成を動かす

前提は .NET 10 SDK と、起動済みの SteamVR です。

```bash
dotnet new console -n MyOverlayApp
```

```bash
cd MyOverlayApp
```

```bash
dotnet add package FloatSoda
```

`Program.cs` を冒頭のコードに書き換えて、SteamVR を起動した状態で `dotnet run` すると、ダッシュボードに青い箱が浮かびます。オーバーレイのサイズは root ウィジェットのレイアウト結果に自動で追従します。

サンプルアプリも用意しています。リポジトリをクローンして `samples/FloatSoda.Samples.OverlayApp` を実行すると、レイアウト、時計、アニメーション、カウンター、ドラッグのデモがダッシュボードに追加されます。HMD を被らずにレイアウト結果だけ確かめたいときは、`samples/FloatSoda.Samples.PaintingSample` を実行すると、Widget ツリーが PNG に書き出されます。

### 最初のお題:VR を被ったまま AI を承認するボタン

私がいま一番作りたいものを 1 つ挙げます。VR を被ったまま、Claude Code の「実行していいですか?」を承認できるボタンです。AI にコードを書かせているとき、承認のたびにヘッドセットを外すのは面倒です。ツール名とコマンドがオーバーレイに表示され、レーザーで「許可」を押します。部品は、この記事で紹介したテキスト表示とボタンでもう揃っています。

もしよかったら、作ってみてください。作ってみて足りない機能があったら、私が追加します。ロードマップは「どんなアプリが作れるようになるか」で区切っていると書きました。みなさんが作りたいものこそがロードマップです。

### フィードバックをください

FloatSoda はまだ v0.3 のひよっこですが、青い箱と腕時計とボタンは今日から動きます。詰まったところ、変だと思った API、欲しいウィジェットなど、なんでも [Issue](https://github.com/sumx21t-3310/FloatSoda/issues) へお寄せください。v1.0 で API を凍結する前のいまが、フレームワークの形をいちばん変えられる時期です。個人開発のフレームワークは、使う人の声だけが栄養だからです。

コントリビュートも歓迎です。ウィジェットの実装には Flutter という最高の参考実装があるので、実は入門しやすい OSS だと思います。

## まとめ

FloatSoda は、SteamVR のオーバーレイを Flutter のような宣言的 UI で書ける C# 製の UI フレームワークです。

- 「テクスチャさえ転送できれば何でも描ける」という仮説から始まり、5 ヶ月で箱・腕時計・押せるボタンが動くところまで来ました
- 内部は Flutter の三つのツリーの移植で、C# のオブジェクト初期化子と `record` で Flutter の書き味を再現しています
- AI に書かせやすいことを設計目標にし、ドキュメントだけで AI に作らせるテストを品質基準にしています
- Alpha 段階で API は変わります。スクロールと既製の UI 部品はこれからです

VR オーバーレイの UI フレームワークは、空白地帯でした。でも空白地帯は「誰も必要としていない場所」ではなく、「まだ誰も埋めていない場所」です。あなたのアイデアが、VR 空間に浮かびますように。

## 参考と謝辞

- [kurohuku さんの Zenn 本](https://zenn.dev/kurohuku/books/a082c5728cc1f6):Unity で SteamVR オーバーレイを作る解説。「テクスチャ転送仮説」の種になりました
- [fastriver さんの「再実装Flutter」](https://zenn.dev/fastriver/books/reimpl-flutter):Flutter を個人で登れる山にしてくれた本です
- OVRSharp:OpenVR の .NET ラッパー。一時期フォークして多くを学ばせてもらいました。現在の FloatSoda.OVR は自前実装ですが、リポジトリにサードパーティ表記を残しています
- そして、C++ フルスクラッチという衝撃ですべてを始めてくれた、あのフレンド
