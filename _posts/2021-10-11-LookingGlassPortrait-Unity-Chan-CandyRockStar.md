---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post
title: "ユニティちゃんライブステージ - Candy Rock Star - をLooking Glass Portraitで動かす"
---

** 書いてる途中です **

まずこちらをご覧ください。

https://youtu.be/BUS1tvsb96U

これはLooking Glass Portraitでユニティちゃんライブステージ、
またはCandy Rock Star(CRS)として知られるアプリを実行した時の映像です。

この投稿ではどうやって移植したのかを解説します。

# 基本的な概念(Holoplay Capture)

先にLooking Glassに写すためのカメラとなるHoloplay Captureについて解説します。

![Unity上のHoloplay Capture表示](/assets/images/crs_lkg/unity_holoplay_capture.png)

Holoplay Captureはピラミッドのてっぺんを切り落としたような見た目をしています。
面が小さいほうから順に*near clip*、*focal plane*、*far clip*と呼びます。
focal planeはzero pallax planeとも呼びます。

Holoplay Captureの中心はfocal planeの中心に存在します。
とても重要なことなのですが**Looking Glassの焦点はfocal planeの面で合う**ようになっています。
Looking Glassを使ってみればわかるとおり、
この装置のホログラムは全ての点でくっきり写るわけではなく、
手前と奥側にある物はぼやけて写るようになっています。
これはfocal plane以外にあるものは見る位置がズレて投影されるからです。

near clipとfar clipは名前の通りその面より外にあるものは描画されなくなる面です。

そして今説明したHoloplay Captureの台形的なやつは
sizeというパラメータによって大きさが変化します。
形は変わりません。
つまり物を大きく写したいなら台形的なやつの大きさを小さくして、
台形の中に占める割合を大きくするのが基本になります。

# Candy Rock Starをいじる

構成についてはCRSのリポジトリについてるwikiを参照してください。
ここではどの部分をいじったのかだけ解説します。

## CameraSwitcherを改造する

```C#
using System.Collections;
using LookingGlass;

public class HoloPlaySwitcher : CameraSwitcher
{
    public float nearClip;
    Transform targetPoint;
    Vector3 followCameraPoint;
    Vector3 origCameraPos;
    Quaternion origCameraRotate;
    public AnimationCurve sizeCurve = AnimationCurve.Linear(0, 0.3f, 85, 2.1f);
    Holoplay holoCamera;

    void Start()
    {
        holoCamera = Holoplay.Instance;
        holoCamera.nearClipFactor = nearClip;
        // Target information.
        targetPoint = GameObject.Find(targetName).transform;
        followCameraPoint = targetPoint.position;

        // Start auto-changer if it's enabled.
        if (autoChange) StartAutoChange();
    }

    void Update()
    {
        // Update the follow point with the exponential easing function.
        var param = Mathf.Exp(-rotationSpeed * Time.deltaTime);
        followCameraPoint = Vector3.Lerp(targetPoint.position, followCameraPoint, param);

        // Look at the follow point.
        //transform.LookAt(followPoint);
        transform.position = origCameraPos;
        transform.LookAt(followCameraPoint);
        transform.position = followCameraPoint;
        //transform.Rotate(new Vector3(0,1,0), 180, Space.World);
        holoCamera.transform.rotation = transform.rotation;
        holoCamera.transform.position = transform.position;
    }

    // Change the camera position.
    public override void ChangePosition(Transform destination, bool forceStable = false)
    {
        if (!holoCamera)
        {
            holoCamera = Holoplay.Instance;
        }
        // Do nothing if disabled.
        if (!enabled) return;

        // Move to the point.
        origCameraPos = destination.position;

        // Snap if stable; Shake if unstable.
        if (Random.value < stability || forceStable)
            followCameraPoint = targetPoint.position;
        else
            followCameraPoint += Random.insideUnitSphere;
        holoCamera.transform.position = transform.position;

        // Update the FOV depending on the distance to the target.
        var dist = Vector3.Distance(targetPoint.position, origCameraPos);
        //GetComponentInChildren<Camera>().fieldOfView = fovCurve.Evaluate(dist);
        holoCamera.size = sizeCurve.Evaluate(fovCurve.Evaluate(dist));
    }

    // Choose a point other than the current.
    Transform ChooseAnotherPoint(Transform current)
    {
        while (true)
        {
            var next = points[Random.Range(0, points.Length)];
            var dist = Vector3.Distance(next.position, targetPoint.position);
            if (next != current && dist > minDistance) return next;
        }
    }

    // Auto-changer.
    IEnumerator AutoChange()
    {
        for (var current = points[0]; true; current = ChooseAnotherPoint(current))
        {
            ChangePosition(current);
            yield return new WaitForSeconds(interval);
        }
    }

    public override void StartAutoChange()
    {
        StartCoroutine("AutoChange");
    }

    public override void StopAutoChange()
    {
        StopCoroutine("AutoChange");
    }
}
```

元々のCameraSwitcherと同じ部分は説明を省きます。
重要なのは`ChangePosition`と`Update`メソッドです。

先にも説明した通り、
Holoplay CaptureはUnityで用いられている一般的なカメラとは違い、
対象物を箱の中に納めるように使います。
ですので元々のCRSで使われていたカメラ位置から見て対象物を納めるように位置などを変更します。

まず`ChangePosition`では元々CRSで指定されてたカメラの位置を`origCameraPos`に保管します。
そして対象物の位置を`targetPoint`と`followCameraPos`に保管します。
最後に`targetPoint`と`origCameraPos`の距離に応じて良い感じにHoloplay Captureのサイズを調整します。

サイズを調整する段階で2段階の変換関数を通していますが、
これは1段階目が元々のCRSでFoVを算出する関数があるためで、
2段階目はそのFoVからサイズを求めるようにしています。
実際のところHoloplay CaptureのFoVから位置を逆算してもよさそうなのですが面倒そうなのでやめました。

次に`Update`です。
元々ユニティちゃんの移動にあわせてゆるやかに位置が変わるようになっています。
やっていることは単純で、
元々のカメラの位置(`origCameraPos`)から追跡したい点の位置(`followCameraPos`)を見た時の方向(回転)を取りだし、
Holoplay Captureの位置を`followCameraPos`に置いて取りだした方向を入れています。
ちゃんと方向を入れてあげないとあらぬ方向を見てしまいますので注意しましょう。

なお雑多なことになりますが、
Holoplay Captureはシーンに1つだけ存在できます。
そしてそのインスタンスは直に参照できるので雑に扱っても大丈夫だと思います。
詳細については公式のドキュメントをご覧ください。

https://docs.lookingglassfactory.com/developer-tools/unity/scripts/holoplay

## 改造したスクリプトをMain Camera Rigにくっつける

次に改造してできた`HoloPlaySwitcher`を`Assets/UnityChanStage/Prefabs/`にある`Main Camera Rig`プリファブにくっつけます。

![HoloplaySwitcherをくっつける位置](/assets/images/crs_lkg/unity_main_camera_rig_position.png)

この位置にくっつけます。上部にある`Camera Switcher`以下のスクリプト等は全て無効化してください。

くっつけたスクリプトの設定は上部にあった`Camera Switcher`の設定をコピペします。

![HoloPlay Switcherの設定](/assets/images/crs_lkg/unity_holoplay_switcher_setting.png)

FoVカーブの設定は何らかのスクリプトを探してコピーするのが手っ取り早いと思います。
sizeカーブのほうは試行錯誤で設定していくことになります。

`Camera Switcher`以下を全て無効にしている場合、オーディオリスナーをくっつけておきましょう。

## ユニティちゃんのカメラターゲットを移動する

元々のCRSではカメラの視点はユニティちゃんの身体がうまいこと写るような位置に設定されていましたが、
Looking Glassの場合、Holoplay Captureの位置に焦点が合う都合上、顔に焦点が当たるようにすると良い感じになります。

`/Assets/UnityChan/Prefabs/CandyRockStar`プリファブを開いてください。

目のちょっと下あたりに`Camera Target`を移動します。
このオブジェクトにLooking Glassの位置が移動するようになります。

![Camera Targetの位置](/assets/images/crs_lkg/unity_camera_target.png)

このあたりは試行錯誤すると良いと思います。

## Holoplay Captureを設置する

最後にHoloplay Captureをメインシーン(`Main`)に設置します。

![Holoplay CaptureをMainに置く](/assets/images/crs_lkg/unity_holoplay_capture_position.png)

設定は初期値でいいと思いますが、
`Far Clip Factor`と`Near Clip Factor`は割と重要なので試行錯誤しながら設定しましょう。

![Holoplay Captureの設定](/assets/images/crs_lkg/unity_holoplay_capture_setting.png)

## 完成！

これで完成です。
`Toggle Preview`してから再生するとLooking Glassに表示されると思います。
Scene画面に切りかえると何をしているのかよくわかると思います。

# まとめ

比較的簡単な改造でLooking Glassにユニティちゃんのステージを写すことが出来ました。

今回はCRS全体の構造をほぼ理解しないで改造しましたが、
本来ならもう少しデバイスに合わせた改造が出来るかと思います。
どなたか挑戦してみてください。

# 参考文献

- https://tsubakit1.hateblo.jp/entry/2014/11/09/235348
- https://github.com/unity3d-jp/unitychan-crs/wiki

# ライセンス

この文書は"ユニティちゃんライセンス"でライセンスされます。 ©UTJ/UCL

https://unity-chan.com/contents/license_jp/


