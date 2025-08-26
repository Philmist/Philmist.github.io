---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "ETS2用のナビゲーションボイス作成方法まとめ"
toc: true
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

実際にMOD作成で使用したボイスを用意しておきました。
クレジットせずに必要に応じてご利用ください。

* [ボイスのZIPファイル](/assets/ETS2_MOD_AnneliVoices.zip)

### STEP 0: テンプレートプロジェクトを解凍する

### STEP 1: 使用する音声をインポートする

### STEP 2: MOD用のbankを新規作成する

### STEP 3: ナビ用のeventを追加してbankに割り当てる

### STEP 4: ビルドして必要なファイルを作る

### STEP 5: 開発中のナビMODをETS2で使えるようにする

### ENJOY!

### ありがちなトラブル

とりあえず何をするにもまずはETS2で開発者用コンソールを出せるように設定してください。
たとえばマイドキュメントの中にある`Euro Truck Simulator 2/steam_profiles/XXX/controls.sii`(XXXはランダムな文字列)の
`mix console`が含まれる部分を以下のように変更するとかです。

```
mix console `modifier(no_modifier, keyboard.equal?0)`
```

#### `manifest.sii`が含まれていない/正しくない

#### mod内のファイル名が間違っている/siiファイルでの指定が正しくない

#### bankの中に必要なeventが足りていない/フォルダの中にETS2から呼ばれるeventを入れている

#### (zipファイルにまとめた場合)解凍した時に出てくるものがフォルダだけである

# FMOD Studioでの音の出し方いろいろ

ここではFMOD Studioでの音の出し方にどういう手法があるかを記していきます。
手法がわかればナビボイスの台本にも幅が出るかと思います。

## 手法解説のその前にあらためて

FMOD Studioではeventにactionとtimelineの2つの種類のどちらかを割り当てることができます。
そしてeventそれ自体も入れ子にすることが出来ます。

actionは細かい時間の制御が効かないかわりに
どのような種類のどのような長さの音も順々に(もしくは並列に)流すことが出来ます。
たいていのMODはこちらだけを使ったほうが簡単です。

timelineは音声などを切ったりするなどの加工が出来て柔軟な音出しが可能です。
ただしこちらは時間が不定であるようなeventは作れません。
必ずできうる限りで最長の長さになります。

音の基本的な単位はinstrumentと言いいくつかの種類がありますが、
ここでは以下の種類を使います。

+ Single Instrument: 1つの音を割り当てその音だけを発声する
+ Multi Instrument: 複数の音を割り当てランダムにどれかの音を発声する
+ Event Instrument: 他のeventを参照してそれを発声する
+ Silence Instrument: 単純な無音で長さは0msから指定可能

詳しい説明については[FMOD StudioのInstrument解説ページ][fmod-studio-instrument]を参照してください。

他にゲームで使用した方の記事もありますので少々古い記事ですが参考までに紹介しておきます。

+ [FMODをつかってインタラクティブミュージック(1) オーサリング編](https://qiita.com/dTAT/items/8cf855ada87cae88f6ff)

## 複数音声の単純なつなぎあわせ

やり方は2つあります。

### actionでInstrumentを並べる

![Actionの中でSingle Instrumentを並べている画像](/assets/images/ets2-nav-voicemod/single-instrument-consecutive.png)

これは単純なSingle Instrumentを並べただけのactionです。
右上のドロップダウンリストが**Consecutive**になっているのを確認してください。
これだけで上から順々に音を出してくれます。

並べるInstrumentはSingle Instrument以外にも使えます。
その場合のInstrumentの1つの長さはInstrumentが実際に発声される長さになります。

### timelineでInstrumentを並べる

![Timelineの中でSingle Instrumentを並べている画像](/assets/images/ets2-nav-voicemod/single-instrument-timeline.png)

こちらはTimelineの中でSingle Instrumentを並べています。
一般的なオーディオ編集ソフトに近い構成かと思います。
こちらではInstrumentの分割(split)やクロスフェードなども使用できます。

並べるInstrumentはSingle Instrument以外にも使えますが、
その場合の中身のInstrumentの長さはInstrumentで最長の長さになります。
1秒と2秒の音声を使ってMulti Instrumentにしている場合、最長の2秒が採用されます。

## Multi Instrumentを使っての複数パターン作成

![Actionの中にMulti Instrumentを配置した画像](/assets/images/ets2-nav-voicemod/multi-instrument-equality.png)

Actionの中にMulti Instrumentを配置するとこのような画面になります。
4段に分かれていますが、これは4つの音声/eventを中に配置しているからです。

どこに音声/eventを配置するかというとPlaylistの中に配置します。

![Multi InstrumentのPlaylistの画像](/assets/images/ets2-nav-voicemod/multi-instrument-playlist.png)

画面下にPlaylistという区画があるのでこの中にAssets一覧の音声やEvents一覧のeventをドラッグ&ドロップで配置してください。
単純に並べた場合、それぞれが等確率で発声されます。
画像では4つ並べているのでそれぞれが25%の確率で発声されます。

### 補: Multi Instrumentでの発声確率調整

Multi InstrumentではPlaylistの中の項目に対して違った発声確率を指定することが出来ます。

![Multi InstrumentのPlaylistで発声確率を変えた画像](/assets/images/ets2-nav-voicemod/multi-instrument-probability.png)

この例では`start_hightension`というeventに90%の発声確率を指定し、
残りの`start_calm`には残りの10%を割り当てています。
1つ以上のInstrumentに発声確率を指定すれば残りについては自動で均等になるよう設定してくれます。

確率を指定するにはPlaylistの音声部分で右クリックして**Set Play Percentage**を選び、
その後%の欄に発声確率を入力してください。

![Multi InstrumentでSet Percentageを選んでいる画像](/assets/images/ets2-nav-voicemod/multi-instrument-set-percentage.png)

ちなみに合計で100%に足りなかった場合、足りない部分は発声しなくなるわけではないようです。
計算するのは面倒なので必要な部分だけ指定して残りはFMOD Studioに任せたほうが楽です。

## Event Instrumentを使って部分ごとに組み合わせて発声

次の画像は`roundabout_1`event、
つまり"ラウンドアバウト/ロータリーの1つ目の出口"を案内するeventの画像です。

![roundabout_1の実装画像](/assets/images/ets2-nav-voicemod/roundabout_1.png)

1つ目の`pre_roundabout`Event Instrumentで"_ラウンドアバウト、_"もしくは"_ロータリー、_"と発声し、
2つ目の`010_roundabout_1_01`Single Instrument(これは単純にAssetsからwavをドラッグ&ドロップすれば作成できます)で
"_1つ目の出口です_"と発声しています。
3つ目は"_間違わないでくださいね_"と発声したりしなかったりするeventです(作り方は別項目)。

このような作り方をする場合、1つ目と2つ目の間に発声しない区間を入れたほうが聞きとりやすいです。
私は音声合成ソフト側で読点(、)を入れてボイスのほうに発声しない区間を入れました。
もちろん別項のSilence Instrumentを並べても良いと思います。

ちなみに単体のEvent Instrumentは画面左側のEvents一覧から中央にドラッグ&ドロップすれば簡単に作成できます。

## 長さ0msのSilece Instrumentを使った気まぐれ発声

Silence Instrumentは何も発声しない(=無音を発声する)任意の長さのInstrumentです。

![Actionに配置したSilence Instrumentの図](/assets/images/ets2-nav-voicemod/silence-instrument-action.png)

Silence Instrumentは長さを任意に指定することが出来て、
完全に何もしない0msに指定することすら出来ます。

![0msのSilence Instrumentの画像](/assets/images/ets2-nav-voicemod/silence-instrument-duration.png)

これをMulti InstrumentのPlaylistに追加することで、
"きまぐれに何か追加音声を発声する"という動作を実現できます。

![0msのSilence Instrumentを追加したPlaylistの画像](/assets/images/ets2-nav-voicemod/multi-instrument-0ms-playlist.png)

ただしこの動作を実現するためには**Multi Instrumentを入れたeventの種類がaction**でなければいけないことに注意してください。
なぜならばtimelineだとMulti Instrumentの長さは最長時間が採用されるため、その長さだけ空白時間が出来るからです。

[project-templ]: https://modding.scssoft.com/wiki/Documentation/Engine/Sound/Downloads
[mod-navi-voice]: https://modding.scssoft.com/wiki/Documentation/Engine/Units/sound_data_voice_navigation
[fmod-studio-instrument]: https://www.fmod.com/docs/2.01/studio/instrument-reference.html
