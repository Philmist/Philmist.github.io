---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "MKV+HEVC+opus配信を視聴する方法"
---

この記事は先日書いた記事のフォローアップというか追記に近いものです。

# MKV配信を安定して視聴できるプレイヤーはどれか

現在のところ、MKV+HEVC+opus配信を視聴できるプレイヤーは以下の通りです。

- Windows Media Player + HEVC拡張
- VLC (最近の版)
- mpv
- Media Player Classic

配信した際、リスナーさんからの指摘がいくつかありました。

- Windows Media Playerはキャッシュを残す関係で、SSDを積んでいるPCと相性が悪い。
- VLCは昔の版だとopusのトラックを再生できない。
- VLCで安定して再生するには「ダミー要素」の設定が必要。
- mpvだと安定して視聴できる(らしい)。

配信する上での追記になりますが、OBSでHEVC+opusをffmpeg経由で配信する場合、
リポジトリで最新のコードを使ってビルドをしたOBSでないと、
オーディオバッファが足りなくなった場合の処理に問題がある可能性が高く、
音声が途切れてしまった際にその後の配信が全てプチプチになります(2021/2/16現在)。

