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

** なんかすごい図 **

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


