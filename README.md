# はじめに
今年の1月に、埼玉県八潮市で大規模な道路陥没事故が起きたことは記憶に新しいかと思います。私は現在の住まいが比較的近いこともあり、よく通る道だったため当時とても怖かった思いをしました。 </br>
まあ、そんな状況を根本的に解決するものでは無いですが、普段クルマで、または徒歩や自転車などで通っている道に異変を感じた場合に、何かしらのアラートを送り、それをリアルタイムで地図に反映できたら少しは役に立つかなと思うわけです。 </br>
既存の地図サービスなどでも、このような機能が搭載し始めていますよね。

今回は、私がOSSとして関わっているNode−REDを使い、HERE APIを使ったWeb版の地図と、Datadogのログ検知を使ってユーザーに知らせるという仕組みを作ってみました。一応、Microsoft MVPらしく、クラウドインフラはAzureを使いました 笑

同じ内容を[こちらのQiitaブログ](https://qiita.com/taiponrock/items/1f9e39b258d2fbd9e648) にも書いています。

## 参考サイト
[MIERUNEさんのZenn - The HERE Maps Technical Book](https://zenn.dev/mierune_inc/books/here-writings/viewer/tutorial2) </br>
[NCMさんのQuickConvert - 地図から座標を取得する](http://asp.ncm-git.co.jp/QuickConvert/GetCoordinate.aspx)

# 概要
Node-REDを使って簡易REST APIを作成します。このAPIは、インシデント（事故など）が発生している位置情報（ジオコード）を返却します。 </br>
また、このREST APIがコールされると同時に、Node-REDからDatadogのログインテークのAPIを呼び出します。これには、インシデント発生の位置情報に加えて事象の説明や地図ページへのURLが含まれます。

静的なWebアプリを用意します。このWebアプリでは、HERE MapのAPIを使って地図を描画します。その際に、前述のNode-REDで用意したインシデント座標取得のAPIを呼び出し、情報があればその地図上にピンを立てます。

上記の処理に連動し呼び出されたDatadog APIにて、対象のDatadogのログが可視化されます。 </br>
今回はログの受信までを実装していますが、必要に応じてダッシュボードへの表示やアラート（Monitor）の設定を行うと良いでしょう。

![Screenshot 2025-12-10 at 23.45.16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/9ccfc505-8193-4519-8ff2-66a9a3ab3b13.png)

# Node-RED の実装
## Node-RED を起動します
ご自身で利用しているクラウド環境などへ Node-RED をインストールしてください。 </br>
今回、このブログ用にAzure上で Node-RED を動かしています。

## Azure へ Node-RED をインストールします
Node-RED は Node.js ランタイムで動くWebアプリケーションですので Azure App Service のリソースを作成し、そこへ Node-RED をデプロイしていきます。

App Service を作成しますが、今回はコンテナで動かすのでランタイムは Node.js ではなく Docker を選びます。 </br>
コンテナイメージは、Docker Hub に nodered/node-red で公開されているのでこれを使います。バージョンは latest でOKです。 </br>
プランはB1で十分かと思います。

![Screenshot 2025-12-12 at 13.26.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/5c8a700a-a6ca-414f-8e52-56efac2ba2b8.png)

## フローを作成します
次のような構成でフローを作成します。

`http in` → `function` → `http response` </br>
上記 `function` から枝分かれで → `debug` </br>
上記 `function` から枝分かれで → `function` → `http request` → `debug` </br>

図だと以下のような形になります。 </br>
![Screenshot 2025-12-10 at 14.45.04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/fcd595bf-8cce-4483-9379-2dc9cdcc9ae3.png)

### 1つ目のFunctionノード
インシデント情報を生成し返却するので *generateIncidentPosition* という名前にしています。 </br>
```javascript
// 危険箇所の座標データを返す
msg.payload = [
    { lat: 35.6779624, lng: 139.76347446 },
    { lat: 35.68508213, lng: 139.76313114 },
    { lat: 35.68110829, lng: 139.75866795 }
];

// ステータスコードを設定（オプション）
msg.statusCode = 200;

return msg;
```

ここではテスト用のスタブとして、3つのスタティックな座標を配列で返却するようにしていますが、実際の運用の場合は、データベースを用意してそこで管理できるようにしたデータを取ってくるようにすると良いと思います。

### 2つ目のFunctionノード
Datadogへ送信するためのインシデント情報をメッセージとして作成するので *createMessage* という名前にしています。 </br>
```javascript
const body_message = {
  message: "🚨 異常検知：道路に陥没がありました",
  ddsource: "node-red",
  service: "here-monitor",
  status: "error",
  custom: {
    lat: msg.payload[0].lat ,
    lng: msg.payload[0].lng,
    alert_level: "high",
    map: "https://hogehogemap.com/index.html"
  }
};
msg.payload = body_message;
return msg;
```

同じくテスト用に受け取った位置情報の配列の0番目のみをメッセージに含ませていますが、こちらも実際の運用に合わせて変更すると良いでしょう。

### HTTPリクエストノード
メソッドはPOSTで。 </br>
エンドポイントURLは `https://http-intake.logs.datadoghq.com/v1/input` を使います。 </br>
Headersには `DD-API-KEY` で自分のDatadogのAPIキーを設定してください。 </br>
※ DatadogのAPIキーの取得方法は後ほどの手順で説明します

![Screenshot 2025-12-10 at 15.38.06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/278a351e-9fee-458d-9c5d-82ad0e6ebc7d.png)

Node-REDはここまでになります。忘れずにデプロイしておいてください。

# HERE Mapの実装（Webアプリ）
## APIキーを作成します

[HERE Platform](https://platform.here.com/)へログインし、[Apps](https://platform.here.com/access)にアプリケーションを作成します。

![Screenshot 2025-12-10 at 10.40.03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/de72f8a1-5fa7-449a-bb15-ae524401a83b.png)
![Screenshot 2025-12-10 at 11.00.04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/242bf3a1-246f-42c1-899f-12a1c46e1abe.png)

Create API KeyボタンをクリックするとAPIキーが生成されます。

![Screenshot 2025-12-10 at 11.01.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/94ab7d5c-233f-43dc-b546-01a79f2407ce.png)

## Webアプリを作成します。
今回は静的な1ページのみのアプリを作成しました。 </br>
実装コードは以下の通りです。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <title>HERE Map Sample</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    html, body { margin: 0; padding: 0; width: 100%; height: 100%; }
    #map { width: 100%; height: 100%; }
    .overlay {
      position: absolute;
      top: 12px;
      left: 12px;
      background: rgba(255, 255, 255, 0.9);
      border-radius: 8px;
      padding: 8px 12px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
      font-family: sans-serif;
      line-height: 1.4;
    }
    .controls {
      position: absolute;
      top: 12px;
      right: 12px;
      background: rgba(255, 255, 255, 0.9);
      border-radius: 8px;
      padding: 8px 12px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    }
    button {
      padding: 8px 16px;
      background: #1b468d;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
    }
    button:hover { background: #2a5aa0; }
  </style>
  <script src="https://js.api.here.com/v3/3.1/mapsjs-core.js"></script>
  <script src="https://js.api.here.com/v3/3.1/mapsjs-service.js"></script>
  <script src="https://js.api.here.com/v3/3.1/mapsjs-mapevents.js"></script>
  <script src="https://js.api.here.com/v3/3.1/mapsjs-ui.js"></script>
  <link rel="stylesheet" href="https://js.api.here.com/v3/3.1/mapsjs-ui.css" />
</head>
<body>
  <div id="map"></div>
  <div class="overlay">
    HERE Map サンプルアプリ<br />
    <small>Advent Calendar向けに使うよ！</small>
  </div>
  <div class="controls">
    <button onclick="loadDangerPoints()">危険箇所を更新</button>
  </div>
  <script>
    const platform = new H.service.Platform({
        apikey: '<自分のHERE APIキーを設定>',
    });

    let omvService = platform.getOMVService({
        path: 'v2/vectortiles/core/mc',
    });
    const baseUrl = 'https://js.api.here.com/v3/3.1/styles/omv/oslo/japan/';
    let style = new H.map.Style(`${baseUrl}normal.day.yaml`, baseUrl);
    let omvProvider = new H.service.omv.Provider(omvService, style);
    let omvlayer = new H.map.layer.TileLayer(omvProvider, { max: 22 });

    let map = new H.Map(
        document.getElementById('map'),
        omvlayer,
        {
            zoom: 15,
            center: { lat: 35.681236, lng: 139.767125 },
        }
    );

    const behavior = new H.mapevents.Behavior(new H.mapevents.MapEvents(map));

    // SVGマーカーアイコン
    const svgMarkup =
        '<svg width="24" height="24" xmlns="http://www.w3.org/2000/svg">' +
        '<rect stroke="white" fill="#1b468d" x="1" y="1" width="22" height="22" />' +
        '<text x="12" y="18" font-size="10pt" font-family="Arial" font-weight="bold" ' +
        'text-anchor="middle" fill="white">危</text></svg>';
    const icon = new H.map.Icon(svgMarkup);

    // マーカーグループ（削除・追加を容易にするため）
    let markerGroup = new H.map.Group();
    map.addObject(markerGroup);

    // 動的にマーカーを追加する関数
    function addMarkers(points) {
      // 既存のマーカーをクリア
      markerGroup.removeAll();
      
      points.forEach(point => {
        const marker = new H.map.Marker(
          { lat: point.lat, lng: point.lng },
          { icon: icon }
        );
        markerGroup.addObject(marker);
      });

      // 最初のポイントに中心を移動
      if (points.length > 0) {
        map.setCenter({ lat: points[0].lat, lng: points[0].lng });
      }
    }

    // APIから危険箇所データを取得
    async function loadDangerPoints() {
      try {
        // APIエンドポイント
        const response = await fetch('http://xxxxxxxx/getincident');
        const data = await response.json();
        
        // data形式: [{ lat: 35.69, lng: 139.76 }, { lat: 35.68, lng: 139.77 }]
        addMarkers(data);
        
        console.log(`${data.length}件の危険箇所を読み込みました`);
      } catch (error) {
        console.error('データ取得エラー:', error);        
      }
    }

    // 初期ロード
    loadDangerPoints();

    // 定期更新（30秒ごと）- 必要に応じてコメント解除
    // setInterval(loadDangerPoints, 30000);
  </script>
</body>
</html>
```

上記のコードを Azure Static Web Application としてデプロイします。 </br>
今回はコード（といってもHTMLファイル一つだけですが）を GitHub リポジトリへ公開し、Azure の Static Web App の展開時にそのリポジトリを指定する形にしてます。

![Screenshot 2025-12-12 at 14.05.56.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/e02d1dd9-14e0-42e0-90d4-b8a610676a9e.png)

注意点は、Azure Static Web App でソースを GitHub リポジトリに指定すると、GitHubの対象リポジトリ側に自動でデプロイのための Actions - Workflow が生成されます。そのまま実行すると動的アプリのデプロイを試みて処理が失敗してしまうので、WorkflowのYamlファイルの中に `skip_app_build: true` プロパティを設定しておきましょう。

Webアプリの実装は以上です。

# Datadogの準備
## Datadogのプラットフォームにアカウントを作成します
[こちら](https://app.datadoghq.com/)へアクセスし、プラットフォームへログインします。アカウントを持っていない場合は作成しましょう。トライアルで2週間ほどフル機能を使えるはずです。

## APIキーを確認します
プラットフォームの左下の自分のアカウントが表示されている部分にマウスオーバーすると、API Keysというメニューが現れますのでクリックします。

![Screenshot 2025-12-10 at 15.52.00.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/b7c83775-004b-4e49-9de0-e2c02e7c8598.png)

![Screenshot 2025-12-10 at 15.53.12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/1b9bf5cd-3c4e-4650-9838-f3e13167833c.png)

New Keyで新しくAPIキーを生成し、その値を先程作成したNode-REDのHttp Requestノードの中のDD-API-KEYの値にセットしてください。

# 動作確認
作成したWebアプリが起点になりますので、対象のWebページへアクセスしてください。 </br>
Node-REDで作成したAPIが呼び出されて、インシデント位置情報を3箇所取得し、地図上に「危」というピンを立てているのが確認できました。

![Screenshot 2025-12-12 at 14.07.57.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/f564b1e1-d597-4324-8f3a-fd9f3cfb3d40.png)

この時、Node-REDのフローが起動し、同時にDatadogへカスタムログを送信しているはずです。 </br>
Datadogのログ画面を見てみましょう。

![Screenshot 2025-12-10 at 14.58.06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/d590e88a-a6ae-4c12-aa26-957427da5106.png)

ログが受信できていますね。ログレベルはErrorにしたので（システム障害検知では無いのでErrorというのも違和感ありますが 笑） </br>
詳細画面を開いてみると、設定したタグや情報が確認できます。

![Screenshot 2025-12-10 at 14.58.29a.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/4d2969a4-ee63-48c9-a07e-766a607a85d1.png)

ログの中に、対象の地図アプリのURLを含めるようにしたので、ログ受信からの検知をトリガーにして地図上に立てられたピンを確認しに行く、という運用の流れを想定しました。

ということで、こんな感じですが、一応道路陥没アラートの仕組みが完成しました。

# おまけ - ダッシュボード表示とアラート（Monitor）
## ダッシュボードへウィジェットを追加します
ログを受信できているので、ダッシュボードへのウィジェット追加は簡単です。 </br>
Log Explorer 画面から、クエリを確定させたら `More`プルダウンから `Save to dashboard`を選ぶだけです。

Timeseries を追加

![Screenshot 2025-12-12 at 12.28.58.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/02b8f80c-5643-4a74-9d91-e5409ed6e3df.png)

List を追加

![Screenshot 2025-12-12 at 12.30.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/3bffad66-d520-4af3-9952-7c210c998478.png)

Dashboard はこんな感じ

![Screenshot 2025-12-12 at 12.32.51.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/0a85bbe0-7e6e-4114-81d6-4f4884674606.png)

## アラート（Monitor）を設定します
同じく Log Explorer 画面で `More`プルダウンから `Create monitor`を選ぶだけです。

![Screenshot 2025-12-12 at 12.33.16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/4ef1b590-5776-4631-9248-9201ce468187.png)

Status を error のみにして、アラートのしきい値を1にしてます。（1件でもレポートが報告されたらインシデントとみなす）

![Screenshot 2025-12-12 at 12.35.45.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81226/b16fc600-2e67-462e-8443-a9cc4ac2d598.png)

通知方法はメールでもSlack（事前にインテグレーションが必要）でも、お好きなもので。

# まとめ
今回は、私が関わっている4つのテクノロジーを組み合わせて作成してみました。もちろん、それぞれの仕組みを別のソリューションで代替することもできるので、用途に合わせて拡張してみるとよいかと思います。
Webアプリの部分は、HEREのSDKを使ってモバイルのネイティブアプリで作っても良いかもしれませんね。（PWAとかでも良いかも）

ではでは！
