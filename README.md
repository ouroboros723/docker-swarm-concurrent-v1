# Concurrent V1 Swarm 構成ファイル
このプロジェクトはdocker SwarmにConcurrent v1をデプロイするのに必要なファイルをまとめたものです。
内容物は下記の通りです。
```
/
├── README.md
├── .env
├── etc
│   ├── config
│   │   ├── activitypub.yaml
│   │   ├── config.yaml
│   │   └── gateway.yaml
│   └── static
│       ├── code-of-conduct.txt
│       ├── register-template.json
│       └── tos.txt
├── swarm-stack.yml
└── docker-compose-portainer.yml
```

### README.md
本ファイルです。

### etc
concurrentの設定ファイル群です。  
SwarmでworkerかつActiveになっている全ノードの同じパスに同じ内容で配置する必要があります。(nfsなどで各ノードに共有して配置することを推奨します。)  
`/etc/` フォルダごとコピーして配置してください。

### .env
swarm-stack.ymlファイル用の設定ファイルです。  
現時点ではは下記項目が含まれます。  
- CONCRNT_CONF_DIR  
  configフォルダを配置したパスをフルパスで記述します。  
  ex. `/opt/concurrent-v1/etc/` となるように配置した場合は `/opt/concurrent-v1/`をパラメーターとして指定します。

### swarm-stack.yml  
  Swarm のstackを構成するファイルです。
  基本的に`docker-compose.yml`と書き方は同様ですが、Swarmモードの場合は一部使えないプロパティがある為注意してください。  
  ex. `depends_on:` など

### docker-compose-portainer.yml
  Swarm スタック管理 Webダッシュボード(portainer.io)用の構成ファイルです。(任意)  
  `docker compose -f "docker-compose-portainer.yml" up -d --build` で構築・起動します。  
  起動後、`http://<ホストIP>:9000`へアクセスするとダッシュボードが開きます。なお、初回起動時にログインユーザー情報の登録が求められるので、設定を行ってください。

# Concurrentドメイン管理手引(本家より引用)

## Overview

Concurrentはモジュラリティの思想の元、マイクロサービスのような構成を取っています。

例えば、画像ストレージサービスはストレージやそのコンテンツの配信のため、自宅サーバーで運用するにはやや重いサービスです。
Concurrentはより気軽にドメインをホストできるようにするため、これらの「重い」オプショナルな要素をモジュールとして切り出しています。

Concurrentドメインを動かすために最低限必要なサービスは`gateway`サービスと`api`サービスです。

続いてほぼmustなのが`webui`, `url-summary`サービスで、完全にオプションなのが`activitypub`サービスとなっています。

これらのサービスはgatewayサービスを経由してユーザーに提供されます。

<TODO: なんかわかりやすい図>

## サービス概略
- `gateway` 文字通り外部とのゲートウェイとなるサービスです。認証等と各サービスへのアクセスの分配をおこなってくれます。gatewayサービスだけ外部公開することを想定しています。
- `api` Concurrentの基本機能を提供するサービスです。
- `webui` ユーザー登録ページを表示したり、admin panelを提供するサービスです
- `url-summary` ユーザーの代わりにurlで指定されたページのogタグ等をフェッチし、プレビューを表示するための情報を集めるサービスです。(これがないとconcurrent.worldではURLのプレビューが表示されません)
- `activitypub` concurrent-worldでの投稿をactivitypubとして外に公開し、また外部のユーザーのフォロー・相互通信を行うサービスです。(外部のノートをconcurrentオブジェクトにコピーして保存するのでDBを消費します)

## 構築

dockerを使う方法と、k8sを使う方法があります。

#### with docker
このディレクトリ内のcompose.ymlがそのまま使える形になっています(ルートのcompose.ymlは開発用です)。

現在利用可能なすべてのサービスが入っているので、適宜使わないサービスをコメントアウトしてください。
サービスを省略した場合、config内のgateway.yamlに記述してあるルーティング設定も取り除くのを忘れないように。

config/config.yamlのxxxの箇所を適宜埋めてください。サーバーのprivatekey等は、concurrent.worldで設定から開発者モードを有効にした際に現れるDevToolページのIdentityGeneratorを使うと便利です。

#### with k8s
helmchartがあります: https://helmcharts.gammalab.net
チャート本体のレポジトリ: https://github.com/totegamma/helmcharts/tree/master/charts/concurrent

valuesに入れる値はdocker composeを使う場合のconfigの生成方法を参考にしてください。

モニタリングを有効にする場合はValues.observabilityにgrafanacloudの各種認証情報を追加すると利用できます。

### アカウントの作成
通常通りconcurrent.worldのアカウント作成画面を進み、ドメイン選択で構築したドメインのアドレスを入力して作成します。
(最初から登録をcloseにしたい場合、後述するコマンドで手動で作成し、アカウントインポートから読み込むという方法もあります。)

<TODO: わかりやすいスクショ>

### 管理者アカウントとして設定
(k8sを使える人はよしなに解釈できると思うので以降はdockerを用いた場合を想定してコマンドを掲載します。)

```
# FYI: アカウントの作成(webから行わない場合)
docker compose exec api ccadmin -H db entity add <CCID>

# アカウントに`_admin`ロールを付与する
docker compose exec api ccadmin -H db entity role <CCID> _admin
```

### 管理者画面にアクセス
concurrent-worldの設定画面にある`go to domain home`からジャンプできます

<TODO: わかりやすいスクショ>

### インスタンスを連合する
まずは、hub.concurrent.worldと連合してみましょう。
hostタブで`hub.concurrent.world`と入力し、goボタンを押すだけで連合が完了します。

<TODO: わかりやすいスクショ>

