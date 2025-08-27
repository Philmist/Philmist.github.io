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

以下の説明用のFMOD Stuioの画像は全てVersion 2.01.05のものです。

### STEP 0: テンプレートプロジェクトを解凍する

まず必要な[テンプレートプロジェクト][project-templ]をダウンロードして
適当な作業ディレクトリに解凍してください。
以下のような順の階層で解凍されるはずです。

![テンプレートを解凍したルートディレクトリ](/assets/images/ets2-nav-voicemod/template-extract-1.png)

これは作業ディレクトリで解凍した時の状態です。
`fmod_template_sound_project_143_01`というディレクトリが見えると思います。

![テンプレートを解凍したディレクトリの中の1層目](/assets/images/ets2-nav-voicemod/template-extract-2.png)

解凍した中には`template`というディレクトリが入っています。
解凍に使用したソフトによってはこのディレクトリが直に作業ディレクトリに表われるかもしれません。

![テンプレートを解凍したディレクトリの中の2層目](/assets/images/ets2-nav-voicemod/template-extract-3.png)

その中の構成はこのようになっています。
実際は`template.fspro`というファイルが解凍されますが、
作業をするには別のファイル名(ここでは`anneli.fspro`)に変えておいたほうが混乱しなくなるので無難でしょう。

### STEP 1: 使用する音声をインポートする

使用する音声をFMOD Studioの左側、Assetsタブの中へドラッグ&ドロップしてください。

![WAVファイルをインポートしたあとの状態](/assets/images/ets2-nav-voicemod/import-wav.png)

インポートしたあとで整理用のフォルダを作ることも出来ますが、
FMOD Studio側でフォルダを作ったほうが無難です(音声ファイルを右クリックして"Move to New Folder")。

Assetsにインポートした音声は画面左下で再生することが出来ます。
"Autoplay"をONにすれば選択した時点で再生されます。

![アセットのプレビュー画面](/assets/images/ets2-nav-voicemod/wav-playing.png)

なおWAV以外の形式もFMOD Studioは認識できます(mp3など)。

### STEP 2: MOD用のbankを新規作成する

今度は画面左側を"Banks"タブに切り替えて、
右クリックから"New Bank"を選んでMOD用のbankを作成します。
このbank名はそのままMODで使用するファイルのファイル名の一部になります。
必ず**英数字記号で長さ12文字以内**にしてください。

bankを作った直後はこのような画面になっているはずです。

![bankを作った直後の画像](/assets/images/ets2-nav-voicemod/new-bank.png)

### STEP 3: ナビ用のeventを追加してbankに割り当てる

Eventsタブに切り替えて右クリックの"New Event"から
MODで要求されている全てのeventを作ります。
"New 2D Action"か"New 2D Timeline"のどちらかを選んで作成します。
必ず**2Dがついているほう**を選んでください。

![必要なeventを作成した直後の画像](/assets/images/ets2-nav-voicemod/new-events-unassigned.png)

"#unassigned"とついている状態になっていると思います。
ここからeventを右クリックして自分の作ったbankに割り当てます。
**masterには割り当てない**でください。
なお複数指定することが出来るのでまとめて割り当てたほうが楽です。

![bankにeventを割り当てようとしている画像](/assets/images/ets2-nav-voicemod/assign-to-bank.png)

実際に割り当てると"#unassigned"という表示が消えます。

eventはフォルダを作成して階層構造にすることも出来ますが、
**ETS2から呼びだされるeventはフォルダの下には作らない**ようにしてください。
フォルダの下に作ってしまうと呼ぶことが出来ません。

### STEP 4: 作ったeventを発声するbusに割り当てる

画面一番上あたりにあるメニュー一覧から"Window"の下の"Mixer"か"Mixer Routing"のどちらかを選んでください。
今回は"Mixer"を選んでみます。

![Mixerウィンドウを出した直後の画像](/assets/images/ets2-nav-voicemod/mixer-window-1.png)

画面左側のRoutingタブに注目してください。
上にmaster bankで定義されているbusが並び、下に自分が作ったbankに入れたeventが並んでいます。
"Grp"となっている場所のうち"game"を開いてその下にある"navigation"の下に自分の作ったeventをドラッグ&ドロップで移動します。
そのようにすると次の画像の通りになっているはずです。

![Mixerウィンドウでeventをbusに割り当てた画像](/assets/images/ets2-nav-voicemod/mixer-window-2.png)

なお自分で独自に定義したeventについてはこの作業をしなくてもかまいません。
自分で独自に定義したeventはETS2用のeventで入れ子にして使われるはずで、
何も指定しない場合はETS2のeventを発声するbusを使用するからです。

### STEP 5: ナビ音声をeventで発声するようにする

ここがFMOD Studioで一番楽しい部分です。

まずEventsタブでどのeventに音声を割り当てたいのかを選択して中央に表示させます。
その次にAssetsタブに切り替えて中央の何もない部分にドラッグ&ドロップします。

eventの種類がActionだった場合はこのような画面になるでしょう。

![ActionにSingle Instrumentを作成した場合](/assets/images/ets2-nav-voicemod/action-single-instrument.png)

eventの種類がTimelineだった場合はこれと似たような感じになるはずです。
ここでは複数並べていますが1回だけドラッグ&ドロップした場合、実際にはSingle Instrumentが1つだけ並んでいると思います。

![TimelineにSingle Instrumentを作成した場合](/assets/images/ets2-nav-voicemod/timeline-single-instrument.png)

他に[どのような手法でeventをいじれるか](#event-howto)については別項目にしてあります。

なおevent一覧では自分用のeventを整理するためのフォルダを作成することが出来ます(画面一番下の"New Folder"、名前変更は選択した状態でF2)。
繰り返しになりますが**ETS2から呼ばれるeventはフォルダの中に入れない**でください。

### STEP 6: ビルドして必要なファイルを作る

FMOD Studioでの作業はこれが最後になります。
modに必要なファイルをビルドして作成します。

画面左上の"File"メニューから"Build..."を選んでください。

![FMOD StudioのメニューでBuildを選ぶ](/assets/images/ets2-nav-voicemod/fmod-build.png)

何もなければプロジェクトファイルのあるディレクトリ(ここでは`anneli.fspro`があった場所)の下にある
`build`ディレクトリのさらに下の`-`ディレクトリにbankファイルが出来ています。

![ビルドされたbankファイル](/assets/images/ets2-nav-voicemod/bank-files.png)

重要なのは自分の作ったMOD用のbankファイル(ここでは`anneli.bank`)があるかどうかです。
このファイルはMODで使いますのでどこかにコピーしておきましょう。

もう1つ必要なファイルがあってこちらは"Export GUIDs..."から作成します。

![FMOD StudioのメニューでExport GUIDsを選ぶ](/assets/images/ets2-nav-voicemod/fmod-guid.png)

こちらは先ほどの`build`ディレクトリの中に`GUIDs.txt`というファイルが出来ています。

![エクスポートされたGUIDファイル](/assets/images/ets2-nav-voicemod/GUID-files.png)

こちらもMODで使うのでどこかにコピーしておきましょう。

### STEP 7: 開発中のナビMODをETS2で使えるようにする

開発中のMODをETS2で使用するにはフォルダを作成してその中にファイルを入れるのが一般的です。
マイドキュメントの中のEuro Truck Simulator 2フォルダの中に`mod`フォルダがあります。
もしなければ`mod`フォルダを作成してください。

その下に自分の作ったMODを入れるためのフォルダを作成します。
さらにその下に`sound`フォルダを作成し、もうひとつ下に`navigation`フォルダを作成します。
実際に作ると以下のような構造になるはずです。

![MODフォルダの構造図](/assets/images/ets2-nav-voicemod/mod-folder-structure.png)

これはETS2が内部で使用しているディレクトリ構成と一緒になっています。
自分で調べてみたい方はETS2のゲームフォルダにある`base.scs`ファイルを専用のツールを使って解凍してみましょう(ETS2のゲームフォルダ内には解凍しないでください)。

MODを使うためにはまずMOD自体の説明を書いたファイルである`manifest.sii`が必要になります。
自分の作ったMODフォルダの直下(ここでは`navigation-anneli`の中)に以下のような感じのテキストファイル(文字コードはBOMなしUTF-8)を置いてください。
このファイルがどのような物なのかは[SCS SoftwareのMOD Managerページ](https://eurotrucksimulator2.com/mod_manager.php#using_the_manifest)で解説されています。

```
SiiNunit
{
# ".package_name" does not matter as the dot at the beginning of the file means that this unit is anonymous.
# Please keep this form to not make any conflicts with other mod packages (name collisions).
mod_package : .package_name
{

  # Package version can be any string with any length.
  package_version: "0.1"

  # Display name can be any string with any length.
  display_name: "音声ナビ AivisProject:Anneli"

  # Author can be any string with any length.
  author: "Philmist"

  # Categories is an array of strings.
  category[]: "sound"

  # Icon inside the root directory of the mod.
  #icon: "AnneliBanner.jpg"

  # Description file inside the root directory of the mod.
  #description_file: "mod_description.txt"

  # compatible_versions[]: "1.19.*" # Mod is compatible with 1.19.X..
}
}
```

行の先頭に`#`があるとその行はコメント行であるとみなされます。
ファイルのコメントには色々書いてありますが**`.package_name`の部分は12文字以内の他のMODと重ならない英数字**にしてください。

必ず変えなければならないのは`package_version`と`display_name`です。
`display_name`はMODマネージャーで表示される名前で`package_version`はMODのバージョン番号/名です。
どちらも文字列で指定してください。`package_version`を日本語で指定するのはやめたほうが良いでしょう。

もしアイコンを作る場合、**横262ピクセル×縦162ピクセルのjpgファイル**を作って`icon`のところで指定し先頭の`#`を消してください。
このファイルも自分の作ったMODフォルダの直下にある必要があります。

次に`sound`の下の`navigation`フォルダに自分の作ったbankファイルとGUIDsファイルをコピーする必要があります。

自分の作ったbankファイル(ここでは`anneli.bank`)を`言語名_バンク名.bank`という形の名前(ここでは`japanese_anneli.bank`)に変更してコピーし、
`GUIDs.txt`ファイルを`言語名_バンク名.bank.guids`という形の名前(ここでは`japanese_anneli.bank.guids`)に変更してコピーしてください。

そして同じフォルダ(`navigation`フォルダ)に`言語名_バンク名.sii`(ここでは`japanese_anneli.sii`)というプレーンテキストファイル(文字コードはBOMなしUTF-8)を置いて次のような内容を書いてください。

```
SiiNunit
{
voice_navigation_config : .japanese.anneli.bank
{
	pack_name: "日本語 - Anneli (from AIVIS Project)"
}
}
```

`voice_navigation_config`の部分は`.言語名.バンク名.bank`という形の名前を書いてください)。
再確認になりますがバンク名の部分は英数字で12文字以下にしてください。

`pack_name`の部分はナビゲーションボイスを選択する時に表示されます。
ゲームの表記に合わせて`日本語 - 話者名 (バリエーション等)`という形にしたほうが良いでしょう。

### ENJOY!

ここまですれば実際に自分で作った音声ナビMODを使えるようになっているはずです。
遊んでみて確認しましょう！

なおzipファイルにしておくと直にmodフォルダに解凍せずに入れて遊べるようになっています。
その場合、`manifest.sii`がある場所の中身を全て選択して圧縮してください。

実際にzipファイルにしたものが以下になります。
参考にしてください。

+ [実際に使用できるMODのzipファイル](/assets/navigation-anneli.zip)

### ありがちなトラブル

とりあえず何をするにもまずはETS2で開発者用コンソールを出せるように設定してください。
たとえばマイドキュメントの中にある`Euro Truck Simulator 2/steam_profiles/XXX/controls.sii`(XXXはランダムな文字列)の
`mix console`が含まれる部分を以下のように変更するとかです。

```
mix console `modifier(no_modifier, keyboard.equal?0)`
```

コンソールを使用すると原因を早期に特定することが出来ます。

#### MODマネージャーで有効化できない/表示されない

表示されない場合は`mod`フォルダの下に自分用のMODフォルダーを作成しているか確認してください。
有効化できない場合は`manifest.sii`が含まれていないか正しくないことが原因の場合が多いです。

zipファイルにまとめて使っている場合はzipファイルの構造が間違っています。
自分用のMODフォルダーを圧縮するのではなく、その下の`manifest.sii`などをまとめて圧縮してください。

#### 音声ナビの一覧に表示されない

MOD内のファイル名が間違っているかsiiファイルでの指定が正しくないです。

MODのファイル名は`言語名_バンク名.bank`などですが、
siiファイル内での指定は`.言語名.バンク名.bank`で`_`と`.`が入れかわっている場所があります。

また別の場合としてバンク名が12文字より長い場合があります。
バンク名は英数字で12文字以下にしてください。

#### プレビュー(や使用)しても別なナビの音声が再生される

bankの中に必要なeventが足りていなかったり、
フォルダの中にETS2から呼ばれるeventを入れている場合に発生します。

必要なイベントがちゃんと必要な場所に名前を間違えず配置されているか確認してください。
`speed_signal`event(速度警告音)は忘れられがちです。

# FMOD Studioでの音の出し方いろいろ {#event-howto}

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

単体のEvent Instrumentは画面左側のEvents一覧から中央にドラッグ&ドロップすれば簡単に作成できます。

この方法で参照されているeventは一覧で"#referenced"と表示されます。

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

# 使いやすいボイスにするためには

[Wikiのページ][mod-navi-voice]にも解説されていますがいくつか挙げてみます。

+ 嘘をつかない
    + "左かな？いや右ー！"みたいな奴です。事故ります。
+ 一貫した言い方にする
    + 瞬時に判断しないといけないナビ音声では複数パターンの言い方をすると混乱することがあります。
    + 重要な部分はあまりバリエーションを多くしないほうがいいでしょう。
+ 忙しくなる部分は長さをおさえる/重要な部分を先に言う
    + 特に"compound_XXX"は後に別のボイスがつながる関係で長いボイスだと使いづらいです。
    + "roundabout_X"あたりはロータリーに進入する直前のタイミングなので何番目かを先に知りたいです。

ぜひあなたもお好みの声のナビボイスを作成してみてください。

[project-templ]: https://modding.scssoft.com/wiki/Documentation/Engine/Sound/Downloads
[mod-navi-voice]: https://modding.scssoft.com/wiki/Documentation/Engine/Units/sound_data_voice_navigation
[fmod-studio-instrument]: https://www.fmod.com/docs/2.01/studio/instrument-reference.html
