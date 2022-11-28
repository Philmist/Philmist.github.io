---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "Powershellとffmpegを使ってPeerCastStationでmkvの音声のみ配信(ラジオ配信)をする"
---

# 目的

PeerCastStationはmp3配信に対応していません(Peercast-YT版はおそらく対応しています)。
ですから通常の音声のみ配信(SHOUTcastみたいなやつ)をやりたい場合は
PeerCastStationが対応しているコンテナの中から
音声だけ乗せられるコンテナを選ぶ必要があります。

通常であればASF(かFLV)を選べば大丈夫だと思いますが
ここではあえてMKV(Matroska Video)/MKA(Matroska Audio)で配信を行います。

そして今回はPowerShell上から全ての操作を行います。

なお今回の手順はWindows 10上で行っています。

# 手順

## 前提条件

PowerShellを使うので出来るだけ新しいバージョンのPowerShellをインストールしてください。
執筆した時に使用したのはPowerShell 7.3.0です。

もしバージョンを確認したい場合は次のコマンドで確認可能です。

```Powershell
Get-Host | ForEach-Object {$_.Version}
```

音声をキャプチャ/エンコードしてPeerCastStationへ送出するのにffmpegが必要です。
https://ffmpeg.org/download.html からつながっているリンク先で
Windows用の実行ファイルをダウンロードして配置してください。

今回の手順ではffmpegの実行ファイルを
PATHが通っている場所に配置していることが前提になっています。

ffmpegは現状だとループバックキャプチャ、
つまり音声出力をそのままキャプチャできません(出来ることもある)。
疑似的にキャプチャするために[VB-CABLE](https://vb-audio.com/Cable/)を
インストールして出力デバイスをVB-Cable Inputと書いてあるものにしてください。

## PowerShellでJSON RPCと戦う

PeerCastStationはJSON RPCのAPIを公開しています(https://github.com/kumaryu/peercaststation/wiki/JSON-RPC-API-%E3%83%A1%E3%83%A2)。
通常であればこの文書に書いてある通りcurlを使うのが一番賢い方法です。
ですが今回はPowerShellで戦います。

まず配信設定用のhash tableを作ります。以下は実際に使用した例です。

```Powershell
$ch_jsonrpc_data = @{
  "networkType"="IPv6";
  "yellowPageId"=2044241712;
  "sourceUri"='http://0.0.0.0:8080/stream.mka';
  "contentReader"="Matroska (MKV or WebM)";
  "info"=@{
    "name"="ふぃるちゃんねる(テスト)";
    "url"='http://bbs.jpnkn.com/philmist/';
    "genre"="Test";
    "desc"="テスト中:突然終わります";
    "comment"="音だけです。何も反応できません。";
    "contentType"="MKV"
  };
  "track"=@{}
}
```

`yellowPageId`は事前にJSON RPCを使用してPeerCastStationから取得してください(`"method": "getYellowPages"`)。
`contentReader`も同様です(`"method": "getContentReaders"`で得られた`name`のほう)。

`sourceUri`はPeerCastStationが待ち受けるURIとポートを指定します。
同一PCで動かしているのであれば`http://localhost:8080/stream.mka`とかで良いと思いますが、
別PCで動かしているのであれば`0.0.0.0`を指定して全てのインタフェースで待ち受けするのが楽です。

あとはこれを`Invoke-WebRequest`にJSON RPCの形式にしてPOSTすれば良いのですが
以下は **動かなかった例です**。

```Powershell
$invoke_result = @{
  "jsonrpc"="2.0";
  "method"="broadcastChannel";
  "id"=7144;"params"=$ch_jsonrpc_data
} |
ConvertTo-Json |
Invoke-WebRequest -Method Post -Headers @{"X-Requested-With"="XMLHttpRequest"} -ContentType "application/json" http://192.168.0.100:7144/api/1
```

これを実行するとPeerCastStationのほうでは日本語が文字化けした状態で登録されます。

なぜこういうことが起こるのかというと、
あくまで推測になりますがPowershellはBOM付きのUTF-8を送信しようとするからです。
同様に受信する時もBOMつきのUTF-8でないと文字化けします。
この挙動を理解するためには受信の例を見るのが良いです。

受信する時は次のような感じにすれば良いです(`$result`を後で参照してください)。

```Powershell
$result = @{
  "method"="getChannels";
  "id"=19021;
  "jsonrpc"="2.0";
  "params"=@{}
} |
ConvertTo-Json |
Invoke-WebRequest -Method Post -Headers @{
  "Content-type"="application/json";
  "X-Requested-With"="XMLHttpRequest"
} http://192.168.0.100:7144/api/1 |
%{ [Text.Encoding]::UTF8.GetString([Text.Encoding]::UTF8.GetBytes($_.Content)) } |
ConvertFrom-Json
```

ここの肝となる部分は
`[Text.Encoding]::UTF8.GetString([Text.Encoding]::UTF8.GetBytes($_.Content))`
です(`%`は`ForEach-Object`のalias)。

.NETの関数を使用した作業です。`Text.Encoding.UTF8.GetBytes`は
対象を(なぜか)BOMつきであろうがなかろうがUTF-8のバイト列に変換します。
`Text.Encoding.UTF8.GetString`はバイト列をBOMつきの文字列に変換します。

なぜこういうことをして上手くいくのかというと
どうやらPowershell内部の文字列処理がBOM付きを前提にしていることによります。
そしてこの問題が`Invoke-WebRequest`で送信する時に問題を引きおこします。

これを踏まえた次の送信例は **動かなかった例** です。

```Powershell
$encoding = New-Object System.Text.UTF8Encoding $false
$invoke_result = @{
  "jsonrpc"="2.0";
  "method"="broadcastChannel";
  "id"=7144;
  "params"=$ch_jsonrpc_data
} |
ConvertTo-Json |
%{ $encoding.GetString([Text.Encoding]::UTF8.GetBytes($_)) } |
Invoke-WebRequest -Method Post -Headers @{"Content-type"="application/json;coding=utf-8";"X-Requested-With"="XMLHttpRequest"} http://192.168.0.100:7144/api/1
```

これはなんとかがんばってBOM付きに変換しようとしていますが、
結局パイプで渡される過程で死んでいます。
これを回避するにはいったんファイルに書きだす必要があります。

```Powershell
@{
  "jsonrpc"="2.0";
  "method"="broadcastChannel";
  "id"=7144;
  "params"=$ch_jsonrpc_data
} |
ConvertTo-Json |
Out-File -Encoding UTF8 request.txt
```

そしてこれを`Invoke-WebRequest`で読み込んで送信します。

```Powershell
$invoke_result = Invoke-WebRequest -Method Post -Headers @{"X-Requested-With"="XMLHttpRequest"} -ContentType "application/json" -InFile .\request.txt http://192.168.0.100:7144/api/1
```

正直ダメな仕様だと思いますが 現状ではこれが回避策だと思います。

参考文献を挙げておきます。

* https://stackoverflow.com/questions/53033242/powershell-convertfrom-json-encoding-special-characters-issue/53033574#53033574
* https://qiita.com/zaki-lknr/items/1ae3258d7b77c5e2a2ba
* https://qiita.com/todkm/items/fbb139b8c9f3e481054b

## ffmpegと戦う

前節の作業でPeerCastStationは待ち受けを開始しています。
今度はffmpegでデータを送信します。

まず最初にffmpegで認識されているデバイスを確認します。

```Powershell
ffmpeg -list_devices true -f dshow -i dummy
```

この中にVB-CableのOutput(`CABLE Output (VB-Audio Virtual Cable)`みたいなやつ)が
あることを確認します。
これを入力デバイスとしてPeerCastStationに出力します。

ここでは`192.168.0.100`をPeerCastStationが起動されているPCとしています。
別途ファイアウォール等の設定はしてあるものとします。

```Powershell
ffmpeg -f dshow -i 'audio=CABLE Output (VB-Audio Virtual Cable)' -map 0:a:0 -c:a libvorbis -metadata 'language=ja-jp' http://192.168.0.100:8080/stream.mka
```

今回は関係ありませんが
ffmpegは`-i`オプションの順番で入力ソースを管理するのに注意が必要です。

`-map 0:a:0`は1番目の入力でオーディオソースのうちの1番目を
出力先の1番目のストリームとして指定しています(いわゆる0-indexed)。
`-map`は指定した順番に出力先へ格納されます。

`-c:a`でオーディオで使用するコーデック(というよりライブラリ)を指定しています。
今回は`libvorbis`なのでVorbisを音声コーデックに指定しています(良く使われるOgg VorbisのVorbis)。

`-b:a`でABRのビットレート、`-q:a`でVBRの品質を指定できると思いますが、
今回は特に指定していません。指定していない場合は`-q:a 3.0`相当になるはずです。
詳細は https://ffmpeg.org/ffmpeg-codecs.html#libvorbis を見てください。

# 結論

光の戦士時代にオーディオだけ送信をやるのはかなり無理があるよ……

