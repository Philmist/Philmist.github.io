---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "FBXからVRMを作成する方法"
---

この文書では3DモデルフォーマットのFBXファイルから
VR用人型モデルフォーマットのVRMファイルを作成する方法を解説します。

## VRMとは

VRMとは主にVRで用いられる3Dアバターを横断的に扱うためのフォーマットです。

詳細については[公式サイトの解説](https://vrm.dev/vrm_about/)に譲りますが、
一度作成すれば多くのソフトで同一ファイルを扱うことが出来るのが特徴です。

## FBXからVRMを作成する方法

それではFBXファイルからVRMファイルを作成する方法を解説します。

ここでは既に各種セットアップ済みのFBXファイルを用います。
VRChatを想定して作られたモデルが一番使いやすいでしょう。
今回はCC0で使用可能な[シャペル](https://lowteq.booth.pm/items/1349366)さんを使います。

### UniVRMをダウンロードしてインポートする

公式サイトから飛べる[githubのリリース一覧](https://github.com/vrm-c/UniVRM/releases)から
UniVRMの最新版をダウンロードします。

次にUnityHubなどで新規プロジェクトを作成しましょう。
Unityのバージョンは最新リリースで提示されているバージョン(大抵の場合はLTSの最新)を用いるのが確実です。

次に公式サイトの説明通り、
[UniVRMをインポート](https://vrm.dev/docs/univrm/install/univrm_install/)します。
メニューの位置はこのあたりです。

![unitypackageのimportの位置](/assets/images/shapell/unity_vrm_import_menu.png)

### シャペルさんをプロジェクトに追加する

次にシャペルさんを解凍したフォルダをプロジェクトに追加します。

具体的にはzipファイルを解凍してアセットのあるフォルダに追加します。
画面下のAssetsで右クリックをして"Show in Explorer"でエクスプローラーを開いたら
その中に解凍したフォルダを移動します。

最終的にUnityの画面で次のようになればOKです。

![フォルダをコピーしたUnityの画面](/assets/images/shapell/unity_shapell_directory_imported.png)

### 正規化したVRMを出力する


