---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "ffmpegオプション個人的まとめ"
---

毎回探すのがいやになりました。

使用例のいくつかは[私のgist](https://gist.github.com/Philmist)に
置いてあります。
あとたいていのことは[ffmpeg wiki](https://trac.ffmpeg.org/)にのってます。

# 入力の指定とか初めの色々

ほとんどのことは[ニコラボ](https://nico-lab.net/about-ffmpeg/)に載ってますが一応。

ffmpegは指定された順番に入力を割りあてます。ファイルを入力に指定したい場合は以下の通り。
`()`はオプション、`|`はどちらかを指定です。

```
(-ss hh:mm:ss) (-to hh:mm:ss | -t seconds) -i input.mkv
```

[ニコラボの解説](https://nico-lab.net/cutting_ffmpeg/)が詳細に書かれています。

動画だけフィルターにかけたい場合は`-filter:v`でフィルタをそれぞれ`,`で区切ります。
音声だけフィルターにかけたい場合は`-filter:a`で同様にします。
フィルター内のオプションの各値は`:`で区切ります。

動画と音声の両方をフィルターにかけたい場合は`-filter_complex`を使います。
それぞれのフィルターは`;`で区切り、フィルターの出力を`[name]`のように指定します。

入力を`filter_complex`で使う場合は最初の入力を0番目として数えます。
0番目の動画は`[0:v]`、0番目の音声は`[0:a]`。
音声が複数トラックある場合、その最初のトラックの音声は`[0:a:0]`です。

最終的な出力をする場合に`filter_complex`を使っていたなら`-map`を使う必要があります。
1番目のストリームから順番にmapを並べます。
通常は1番目は動画のはずなので動画から並べます。

```
-map [v] -map [a]
```

mapについても[ffmpeg wikiのmapの解説](https://trac.ffmpeg.org/wiki/Map)を見てください。

# n倍速/スローモーション

[ffmpeg wikiの該当ページ](https://trac.ffmpeg.org/wiki/How%20to%20speed%20up%20/%20slow%20down%20a%20video)も
参考にしてください。

該当するフィルタは`setpts`と`atempo`。

`setpts`はフレームごとのタイムスタンプを書きかえることで速度を変えます。
`atempo`は音をn倍速にします。

例えば動画と音声をそれぞれ2倍速にしたいのならば次のようなコマンドになるでしょう。

```ps1
ffmpeg -i test.mkv -filter_complex "[0:v]setpts=PTS/2[v];[0:a]atempo=2[a]" -map [v] -map [a] output.mkv
```

PTSは現在のフレーム数なのでPTSがそれぞれ2分の1になるということは結果として2倍速になります。

逆に2分の1のスローモーションにしたい場合は以下の通り。`atempo`は0.5から2.0までの値しか取れないので注意が必要です。

```ps1
ffmpeg -i test.mkv -filter_complex "[0:v]setpts=PTS*2[v];[0:a]atempo=0.5[a]" -map [v] -map [a] output.mkv
```

# 無音を入力する

`anullsrc`を使います。[ffmpeg wikiの解説](https://trac.ffmpeg.org/wiki/Null)も見てください。
そのままだと無限に無音を出力するので`-shortest`が必要です。

```
ffmpeg -i input.mkv -f lavfi -i anullsrc -c:v copy -c:a aac -map [0:v] -map [1:a] -shortest output.mp4
```

# クロップ(切り抜く)する

[ffmpegのドキュメントの該当部分](https://ffmpeg.org/ffmpeg-filters.html#crop)も見てください。

```
crop=w=(出力幅):h=(出力高さ):x=(切りだすX座標,左が0):y=(切りだすy座標,上が0)
```

`in_w`か`iw`で入力される動画の幅、`in_h`か`ih`で同じく高さを表わし、
`sar`で入力される動画のアスペクト比を表わします。

# 動画をつなぐ

いくつか方法がありますが`concat`フィルタによるものを紹介します。
[ffmpegドキュメントの該当部分](https://ffmpeg.org/ffmpeg-filters.html#concat)も見てください。
なおこのやり方の場合は再エンコードが必要になるのでオプションの`-c:v copy`(再エンコードをせずに動画をコピー)等は使えません。

```
[0:v][0:a][1:v][1:a][2:v][2:a]concat=n=3:v=1:a=1[v][a]
```

他にも[ffmpeg wikiの解説](https://trac.ffmpeg.org/wiki/Concatenate)を見ると様々な例が載っています。

---

以下あとから書きます。

# クロスフェード

# 矩形を重ねる

# 曲名スクロールしながら流す

---

# 音声を聞きとりやすい音量にする

[ニコラボの"ffmpegで聞き取りやすい音量に変えるdynaudnorm"](https://nico-lab.net/normalize_audio_with_ffmpeg/)
が非常にわかりやすいです。

ここでは実際の使用例だけ載せておきます。

```
ffmpeg -f lavfi -i color=s=1280x720:c=black -i .\audio.m4a -map 0 -map 1 -c:v libx264 -filter:a dynaudnorm -c:a aac -shortest .\result.mkv
```

# 音声からノイズを除去する

ローパスフィルタを使うなどの様々な方法が存在しますが、
現状で一番手っ取り早そうなのが機械学習を用いたモデルを使う方法です。

参考記事として[日本語の記事](https://zenn.dev/kometan/articles/bfb058afe97b7c)と
[参照されている英語の記事](https://www.amirsharif.com/using-ffmpeg-to-reduce-background-noise/)、
そして[StackExchangeの回答](https://superuser.com/questions/733061/reduce-background-noise-and-optimize-the-speech-from-an-audio-clip-using-ffmpeg/1393535#1393535)を
挙げておきます。

基本的な使用方法は以下の通りです。
事前に[GitHubでモデルを配布しているページ](https://github.com/GregorR/rnnoise-models)から必要なファイルをダウンロードしておいてください。
エンコードするファイルと同じディレクトリにあるものと仮定します。

```ps1
ffmpeg -i video.mp4 -c:v copy -af 'arnndn=m=./cb.rnnn' -c:a aac -b:a 128k out.mp4
```


# 字幕をつける(ソフトサブ:mkv)

動画の字幕ファイルが別に用意してある場合、
matroska video(.mkv)形式にするとその字幕を1つのファイルにまとめることが出来ます。
このようにすることで対応したプレイヤーでは別に字幕ファイルを読まずにすみます。
ちなみに新しく字幕を自分で作る場合はsrt(SubRip)形式が簡単でおすすめです。

```
ffmpeg -i input.mp4 -i sub.srt -map 0:v -map 0:a -map 1 -c:v copy -c:s srt subbed.mkv
```

字幕は複数埋めこむことも出来ます。
この場合、それぞれの字幕にメタデータをつけておくと再生する際に混乱せずにすみます。

```ps1
ffmpeg -i input.mp4 -i eng.srt -i jpn.srt -map 0:v -map 0:a -map 1 -map 2 `
  -metadata:s:s:0 language=eng -metadata:s:s:0 title="English" `
  -metadata:s:s:1 language=jpn -metadata=s:s:1 title="Japanese" `
  out.mkv
```

ここのメタデータを注入する際の記述について少しだけ解説します。

メタデータはファイル全体、あるいはストリームごと、チャプターごとなどいくつかの単位で埋めこむことが出来ます。
ファイル全体に対してのメタデータの場合は`-metadata:g`を使い、
ストリームに対しては`-metadata:s`を使います。

ストリームに対して使う場合はどのストリームに対してメタデータを埋めこむかを指定しないといけないので、
`-map`で使うようなストリーム指定を後に続けなければなりません。
先の例だと`-metadata:s:s:0`は字幕ストリームで一番最初(0番目)のストリーム(`s:0`)を指定していることになります。

この項は
"[FFmpegで動画に字幕・副音声を追加する](https://dev.classmethod.jp/articles/add-audio-and-subtitle-to-video-with-ffmpeg/)"
を参考にしました。

# dispositionオプションでmkvのデフォルトのトラックを指定する

mkv(Matroska)ファイルは複数の映像/音声/字幕トラックを指定できる都合上、
デフォルトでどのトラックを再生するかという情報が重要になることがあります。
この時、元ファイルからそのまま映像等をコピーすると
そのトラックがデフォルトであるという情報まで同時にコピーされます。

他のトラックをデフォルトにしたい場合はデフォルトであるという情報を消さなければなりません。
それらを操作するために`-disposition`オプションを使用します。
[FFmpegのドキュメントの該当部分](https://ffmpeg.org/ffmpeg.html#Main-options)も参照してください。

```ps1
ffmpeg -i video.mp4 -i audio.wav -map 0:v -map 1 -map 0:a -c:v copy -c:a:0 aac -c:a:1 copy `
    -disposition:a:0 default `
    -disposition:a:1 0 `
    out.mkv
```

なお後からmkvを調整したいだけなら[mkvpropedit](https://mkvtoolnix.download/doc/mkvpropedit.html)を使ったほうが良いです。

---

# ハードウェアエンコード(nvidia)

nvidiaのハードウェアエンコードで多くのビデオカードでサポートされているのは
H.264(`h264_nvenc`)とHEVC(`hevc_nvenc`)の2つです。
サポートされているオプションは次のコマンドで出すことが出来ます。

```powershell
ffmpeg -h encoder=h264_nvenc
ffmpeg -h encoder=hevc_nvenc
```

`-b:v`(映像ビットレート:固定ビットレート)は通常のエンコーダーと同様に使えますが、
libx264で使える`-crf`は使えません。代わりに`-cq`を使う必要があります(0から51までの値:0は自動)。

その他プロファイル等の指定も出来ます。
デフォルトでは画質を抑える設定になっているので、
必要に応じて`-preset`の値を変えたり(`p4`から`p6`にするとか)、
`-profile`の値を変えたり(`main`から`high`にするとか)しましょう。

# AV1エンコード(SVT-AV1)

映像をアーカイブとして保存したい場合、
H.264よりも圧縮効率のよい映像コーデックにしたいことがあります。
この時候補として挙がるのがAV1です。

AV1は効率が良いコーデックですがエンコードが重いという重大な難点があります。
一部のGPUではAV1のエンコードに対応していますが
GPUエンコードは残念ながらCPUエンコードに比べて品質が落ちる場合が多々あります。

ffmpegでAV1エンコードを行う場合、最も有用と思われるのがSVT-AV1(`libsvtav1`)です。
エンコード時間も許容できる範囲におさまることが多く、
その設定もかなり幅広くいじれるようになっています。

[FFmpeg wikiの該当部分](https://trac.ffmpeg.org/wiki/Encode/AV1#SVT-AV1)にほとんど書いてありますが、
基本的なエンコード方法は以下の通りです。

```ps1
ffmpeg -i input.mp4 -c:v libsvtav1 -crf 35 -preset 8 -c:a copy out.mkv
```

CRFは0から63の範囲で指定できて0が一番画質が良く63が一番画質が悪くなります。
同時に低ければ低いほどファイルサイズは大きくなり高ければ高いほど小さくなります。
デフォルト値は30ですが38程度だとバランスが良いようです。

Presetは0から13までの範囲を取り高ければ高いほどエンコード速度が速くなります。
しかし13はデバッグ用なので実用的には8から10程度になるでしょう。

H.264にある`-fastdecode`オプションはSVT-AV1だと`-svtav1-params fast-decode=1`で使用できます。
しかしこのオプションを指定する場合はPresetが5から10までの値に制限されるようです。

なお前述の文書では1080pや720pの映像にはAV1はあまり適していないみたいなことが書いてありますが、
HEVC(H.265)でのエンコードについては記述しないでおきます。

# ビットレート指定メモ

[どういう場合にどのビットレート指定をすればいいかを書いた文書](https://slhck.info/video/2017/03/01/rate-control.html)が
あるのでここにメモしておきます。
雑に要約すると以下の通りです。

- ストリーミング用途にはConstrained Encoding(VBV)を使う
    - `ffmpeg -i input -c:v libx264 -crf 23 -maxrate 1M -bufsize 2M output`(改変引用)
- 保存用途にはConstant Rate Factor(CRF)を使う

