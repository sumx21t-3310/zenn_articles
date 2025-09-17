---
title: "【Unity3D】Quaternionを活用してシンプルでスマートな汎用移動スクリプトを実装する【決定版】"
emoji: "👋"
topics: []
published: false
---

## はじめに
### 今回の目標
- 移動とカメラの回転を責務を分けてコンポーネントを実装する
- 特定のオブジェクトや方向に沿って移動できる
- Quake方式の移動を実現してValorantやCS:GOのようにストレイフジャンプできるようにする
- CharacterControllerの位置をしゃがんでも一定にする
- InputSystemを使った入力

## Quaternionとは

Quaternionについて[こちら](https://www.f-sp.com/entry/2017/08/30/171353)の記事がわかりやすくまとまっています。



## 移動スクリプトの実装

### とりあえずシンプルに動かしてみよう
まずはシンプルにワールド方向に向かって動かせるMoverを書いてみます。


### Quaternionの適用を使って移動方向を特定のオブジェクトに添わせる

Quaternionの適用を使うことで特定の回転に沿ったベクトルを取得することができます。
例えば
```csharp: Quaternionの適用のサンプル
var v1 = transform.forward;

var v2 = transform.rotation * Vector3.forward;
            
Debug.Log(Vector3.Dot(v1, v2) < float.Epsilon); // 内積が0に近ければ同じ方向を向いている
```
とすることで`transform.forward`と同じベクトルを取得することができます。
[なぜ同じかはこちらを見てください](https://qiita.com/OKsaiyowa/items/6b93fe41b33061abf5aa)

つまり、方向の基準になる`Transform`の`rotation`を使ってQuaternionの適用をすれば移動方向を特定のオブジェクトに添わせることができそうです。

では実装してみましょう。

### ジャンプの実装

### ジャンプの判定を安定させる

### しゃがみの実装
しゃがみの実装はとても簡単です`CharacterController`の`height`の数値を上げ下げするだけです。

### 足元に中心を合わせてガタガタを治す
実際に動かしてみるとがたがたしていると思います。これは


### Quake方式の移動計算に変更する
このコードには欠点があります。それはジャンプ中に移動キーを離すと摩擦がないにも関わらず減速してしまうことです。空中にいる間は移動を制限してもいいのですが、やっぱり微調整できたほうが遊びやすくなります。

#### Quakeとは
Quakeの概要はこの動画を見るとわかると思います。
https://www.youtube.com/watch?v=uWKbKh9dCwY
本動画ではストレイフジャンプできるのはQuakeのエンジンのバグとされていますが、アルゴリズムのバグなのでUnityでも、Unreal Engineでも計算式さえ知っておけば再現可能です。

ValorantはUnreal Engineで作られています。ですがQuake方式の移動計算にすることで以下のような挙動を再現しています。
https://www.youtube.com/watch?v=K87KVl4oOUA
https://www.youtube.com/watch?v=CoP3pwFnjKU


[Quake本家のコード](https://github.com/id-Software/Quake/blob/master/WinQuake/sv_user.c#L190)をUnity向けに移植して気持ちのいい移動を目指してみます。
Quakeの移動スクリプトについては[こちらの記事](https://qiita.com/kossuuu/items/9e8e3ab2126d91ac1f59)が詳しいです。

## カメラ操作スクリプトの実装
### Quaternion.AngleAxisの合成を使ってyawとpitchを別々に管理する
Quaternion.AngleAxisの挙動は串をつまんでくるくる一定方向に回転させているイメージが近いと思います。
このとき、axisは串の先が向いている方向だと考えてもらうと直感的に理解できると思います

### Angleクラスを作って回転の責務を移譲する

Rotorに回転の管理をまかせてもいいのですがどうせならAngleクラスを作って回転と制限の管理を任せてしまいましょう。

### マウスカーソルを非表示にする
回転させるとマウスカーソルが表示されたままになり、いろいろなところをクリックしてしまいます。なので、マウスカーソルを非表示にするCursorVisibleを作って、マウスカーソルを非表示にします。

## コンポーネントの組み合わせで様々な移動方式を作ってみる
### FPS
### TPS
TPSの実装にはCinemachineを使うと便利です。RotorにCinemachineCameraをTirdParsonFollow追従させることで、簡単にTPSのカメラを作ることができます。また、

### トップダウン(地面に追従)
レベルのルートのTransformを指定します。こうすることで、レベルを回転させるようなゲームにも対応することができます。
https://www.youtube.com/watch?v=BapVf3fVN6Y
レベルを回転させるゲームの例

### トップダウン(マウスに追従)
マウスとキャラクターでベクトルを作り、そちらを向くコンポーネントを作ります。


## まとめ
いかがだったでしょうか？
3Dアクションゲームであれば、方向の基準さえ決めてしまえばどのようなユースケースでも同じような移動スクリプトを使うことができることがわかると思います。
### 実装のポイント

**責務の分離**: 移動、回転、入力処理を別々のコンポーネントに分けることで、再利用しやすいコードになります。
**Quaternionの活用**: `Quaternion.AngleAxis`と四元数の合成を使うことで、直感的で安定した回転制御が可能になります。
**柔軟な方向基準**: `directionReference`を設定することで、カメラ基準、ワールド基準、特定オブジェクト基準など様々な移動方式に対応できます。
**物理ベースの移動**: Quake方式の移動を採用することで、慣性のある自然な移動感と高度なテクニックが可能になります。
**安定した地面判定**: `CharacterController.isGrounded`だけでなく、Physics.CheckSphereを併用することで、より安定した地面判定を実現できます。

応用例

FPS: カメラを直接制御し、カメラの向きを移動基準とする
TPS: Cinemachineと組み合わせて第三者視点カメラを実現
トップダウン: レベルルートやマウス位置を基準とした移動制御
プラットフォーマー: 重力とジャンプを重視した2.5D的な移動

このフレームワークを基に、あなたのゲームに最適な移動システムを構築してください！