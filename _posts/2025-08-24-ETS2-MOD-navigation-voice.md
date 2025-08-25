---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "ETS2用のナビゲーションボイス作成方法まとめ"
---

**まだ書きかけです**

過去の先人達が残した資料はあったんですがまとまってなかったので自分用に。
先人の資料で調べたものは以下。

* <https://haineons.com/fmod-create-mod/>
* <https://plaza.rakuten.co.jp/mrgamer500/diary/202010160034/>

# ETS2のナビボイスMODを自作するその前に

## 用意するもの

* FMOD Studio (バージョン指定あり)
* FMOD Studio用のテンプレート
* 自作したナビ用音声

FMOD StudioはETS2のバージョンによって使用するバージョンが違います。

[ETS2で使用するFMOD Studioのプロジェクトテンプレート][project-templ]で必要なバージョンを確認してください。
ちなみに記事執筆現在(2025-08-24)、FMOD Studioは[**Ver 2.01.05**をダウンロード](https://www.fmod.com/download#fmodstudio)する必要があります(**Older**の中にあります)。

FMOD Studio自体は無料で使えますがダウンロードにはユーザー登録が必要です。

## 必要な音声一覧

必要な音声は[SCSのMOD製作者用Wikiのナビ音声ページ][mod-navi-voice]に一覧が載っています。
現在の製作方法には対応していませんが必要な音声の種類は一緒です。
こちらにも一応転記しておきます。

|トークン名|英語での音声例|注記|
|---|---|---|
|`turn_left`|"Turn left."|単純な左折|
|`turn_right`|"Turn right."|単純な右折|
|`keep_left`|"Keep left."|左側の車線へ移動もしくは維持|
|`keep_right`|"Keep right."|右側の車線へ移動もしくは維持|
|`exit_left`|"Exit left."|左側出口|
|`exit_right`|"Exit right."|右側出口|
|`go_straight`|"Go straight on."|直進|
|`roundabout_1`|"At the roundabout take the first exit."|ロータリー進入時1番目案内|
|`roundabout_2`|"At the roundabout take the second exit."|ロータリー進入時2番目案内|
|`roundabout_3`|"At the roundabout take the third exit."|ロータリー進入時3番目案内|
|`roundabout_4`|"At the roundabout take the fourth exit."|ロータリー進入時4番目案内|
|`roundabout_5`|"At the roundabout take the fifth exit."|ロータリー進入時5番目案内|
|`roundabout_6`|"At the roundabout take the sixth exit."|ロータリー進入時6番目案内|
|`exit_now`|"Exit now."|ラウンドアバウト/ロータリーの出口で使用|
|`prepare_turn_left`|"Get ready to turn left."|この次左折|
|`prepare_turn_right`|"Get ready to turn right."|この次右折|
|`prepare_exit_left`|"Get ready to exit left."|この次左の出口|
|`prepare_exit_right`|"Get ready to exit right."|この次右の出口|
|`compound_turn_left`|"Turn left ..."|まず左折してから...|
|`compound_turn_right`|"Turn right ..."|まず右折してから...|
|`compound_keep_left`|"Keep left ..."|左の車線を維持してから...|
|`compound_keep_right`|"Keep right ..."|右の車線を維持してから...|
|`compound_exit_left`|"Exit left ..."|左の出口を出てから...|
|`compound_exit_right`|"Exit right ..."|右の出口を出てから...|
|`compound_go_straight`|"Go straight on ..."|まっすぐ行ってから...|
|`and_then_turn_left`|"... and then turn left."|...そして左折|
|`and_then_turn_right`|"... and then turn right."|...そして右折|
|`and_then_keep_left`|"... and then keep left."|...そしてそのまま左を維持|
|`and_then_keep_right`|"... and then keep right."|...そしてそのまま右を維持|
|`and_then_exit_left`|"... and then exit left."|...そして左出口|
|`and_then_exit_right`|"... and then exit right."|...そして右出口|
|`and_then_go_straight`|"... and then continue straight on."|...そしてそのまま真っ直ぐ|
|`start`|"OK, here we go!"|ナビゲーション開始|
|`finish`|"Here we are..."|目的地到着|
|`recomputing`|"Recomputing..."|ユーザーが経路から外れた際の経路再探索|
|`u_turn`|"Make a u-turn."|Uターン|
|`speed_signal`|(Only acoustic signal, no voice command.)|_音声ではなく単なる音_|
|`speed_warning`|"Caution, please mind the speed limit."|制限速度超過|

`compound_XXX`は`and_then_XXX`と組みあわせて使用します。

全ての音声はFMOD側の機能をフルに使って作成することが出来ますので、
様々な工夫をすることが可能です。
それについては後述します。

# ナビ音声MODを作ろう

だいたいは[MOD製作者用WikiのFMODのページ](https://modding.scssoft.com/wiki/Documentation/Engine/Sound/Modding)を見ればわかります。

## FMODで必要な概念を覚えよう

FMODではだいたいの物にGUIDという名前とは関係ない一意の識別子が振られます。
つまり何かを新規作成するたびに他の物とは全く重ならないユニークなIDが振られるということです。
ちなみにFMODでのGUIDは手動では簡単に修正できないようです。
逆に言うと名前を変えただけではGUIDは変わりません。ちゃんと新規に作成する必要があります。

ゲーム側で音を出すのがどこかということについてはテンプレート内で定義されています。
MOD作者はテンプレート内のmainバンク(bank)で定義されているbus(音を出す場所)に自分の音を出せばいい(ルーティング)わけです。
ただしMOD作者はmainバンクではなく別に自分のバンクを作ってその中に自分の物を格納しておく必要があります。

MODの音がゲームでどのように呼びだされるかということですが、
FMODのイベント(event)を使用して呼びだされます。
先のトークンと同名のイベントを作成してあげればゲームとサウンドエンジンが全てよしなにしてくれます。

長かったので必要な部分を箇条書きで書くと以下の通り。

- 特定の名称のイベントを作っておくとゲームがそれを呼んで音を出してくれる
- MOD用の物は全部main以外の**新規に作った**バンクに入れなければならない
- 作ったイベントは特定のbus(具体的には`master/game/navigation`)へ流さなければならない

## MOD作成ステップバイステップ

今回のMOD作成で使用したボイスを用意しておきました。

* [ボイスのZIPファイル](/assets/ETS2_MOD_AnneliVoices.zip)

### STEP 0: テンプレートプロジェクトを解凍する

### STEP 1: 使用する音声をインポートする

### STEP 2: MOD用のbankを新規作成する

### STEP 3: ナビ用のeventを追加してbankに割り当てる

### STEP 4: ビルドして必要なファイルを作る

### STEP 5: 開発中のナビMODを入れる環境を整備する

### ENJOY!

# FMOD Studioでの音の出し方いろいろ

ここではFMOD Studioでの音の出し方にどういう手法があるかを記していきます。

## 手法解説のその前にあらためて

FMOD Studioではeventにactionとtimelineの2つの種類のどちらかを割り当てることができます。
そしてeventそれ自体も入れ子にすることが出来ます。

actionは細かい時間の制御が効かないかわりに
どのような種類のどのような長さの音も順々に(もしくは並列に)流すことが出来ます。
たいていのMODはこちらだけを使ったほうが簡単です。

timelineは音声などを切ったりするなどの加工が出来て柔軟な音出しが可能です。
ただしこちらは時間が不定であるようなeventは作れません。

## 複数音声の単純なつなぎあわせ

## Multi Instrumentを使っての複数パターン作成

### 補: Multi Instrumentでの発声確率調整

## Event Instrumentを使って部分ごとに組み合わせて発声

## 長さ0msのSilece Instrumentを使った気まぐれ発声

[project-templ]: https://modding.scssoft.com/wiki/Documentation/Engine/Sound/Downloads
[mod-navi-voice]: https://modding.scssoft.com/wiki/Documentation/Engine/Units/sound_data_voice_navigation
