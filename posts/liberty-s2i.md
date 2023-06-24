# s2i で Open Liberty アプリケーションのコンテナイメージをビルドする

Open Liberty ではコンテナイメージを s2i (Source to Image) でビルドするためのBuilder Imageが用意されています。

本記事では Open Liberty アプリケーションのコンテナイメージを、Red Hat OpenShift 上で s2i ビルドする方法を紹介します。

## 本記事で使用するコンポーネント

本記事では以下のコンポーネント・バージョンで動作確認を行っています。
各コンポーネントの別バージョンでは動作しない可能性がありますので、本番投入する際には事前に手元で動作確認することをお勧めします。

|コンポーネント名|バージョン|
|:--|:--|
|JDK|OpenJDK 17.0.7 (IBM Semeru Runtime Open Edition)|
|Open Liberty|23.0.0.5|
|Red Hat OpenShift|4.13.1|

```bash
$ java -version
openjdk version "17.0.7" 2023-04-18
IBM Semeru Runtime Open Edition 17.0.7.0 (build 17.0.7+7)
Eclipse OpenJ9 VM 17.0.7.0 (build openj9-0.38.0, JRE 17 Mac OS X amd64-64-Bit Compressed References 20230418_450 (JIT enabled, AOT enabled)
OpenJ9   - d57d05932
OMR      - 855813495
JCL      - 9d7a231edbc based on jdk-17.0.7+7)
```

## Builder Imageの選択

### Builder Image 概要

Open Liberty Builder Imageは、 Open Liberty 本体同様にOSSとして開発されています。Open Liberty の新バージョンがリリースされるとBuilder Imageもリリースされているようです。

https://github.com/OpenLiberty/open-liberty-s2i

Builder Imageは以下のタスクを実行します。なお、ベースイメージにすべてのフィーチャーが含まれています。そのためビルドの都度フィーチャーのインストールは実行されません。

- Apache Maven を使用したWARファイルのビルド
- Open Liberty サーバにWARファイルをデプロイ


### Builder Imageの種類

Builder Imageは記事執筆時点で以下のJavaバージョンに対応したイメージが提供されています。
Java 17のBuilder Imageは、Open Liberty バージョン23.0.0.5から提供されています。

- Java 8
- Java 11
- Java 17

それぞれDocker HubとIBM Container Registry (ICR) から取得可能です。

- Docker Hub: https://hub.docker.com/r/openliberty/open-liberty-s2i
- ICR: icr.io/appcafe/open-liberty-s2i

### 本記事におけるイメージの選択

Open Liberty の公式ドキュメントでは、Open Libertyのベースイメージの取得にICRを使用するようにガイドされています。本記事でもBuilder ImageをICRから取得します。

また、Java 17のイメージタグ `23.0.0.5-runtime-java17` を利用します。


## アプリケーションの準備

アプリケーションの作成は本記事の主題ではないので説明しません。Open Liberty のガイドで紹介されているREST APIサンプルをそのまま使います。

完成版は以下のGitHubリポジトリの `finish` ディレクトリに格納されています。

https://github.com/OpenLiberty/guide-rest-intro


## OpenShiftにBuilder Imageを登録

Builder Imageを利用するため、Builder Imageの参照先をImageStreamリソースで登録します。

Builder ImageのGitHubリポジトリにサンプルのImageStreamのマニフェストが提供されています。
このマニフェストを `oc` コマンドで登録します。

```bash
$ wget -q -O- https://raw.githubusercontent.com/OpenLiberty/open-liberty-s2i/main/imagestreams/openliberty-ubi-min.json \
| sed 's|docker.io/openliberty|icr.io/appcafe|g' \
| oc create -f -
imagestream.image.openshift.io/openliberty created
```

ImageStreamリソースを確認すると、最新バージョンのBuilder Imageが登録されています。

```bash
$ oc get is -o name
imagestream.image.openshift.io/openliberty

$ oc get istag -o name
imagestreamtag.image.openshift.io/openliberty:23.0.0.5-java11
imagestreamtag.image.openshift.io/openliberty:23.0.0.5-java17
imagestreamtag.image.openshift.io/openliberty:23.0.0.5-java8
```

> サンプルのImageStreamマニフェストでは、Docker HubのBuilder Imageを参照しています。`sed` コマンドなどでICRのイメージに書き換えてください。

## コンテナイメージのビルド

### s2iビルドの開始

Builder Imageを指定して`oc new-app` コマンドでコンテナイメージのビルドおよびデプロイを実行します。

引数に `openliberty:23.0.0.5-java17~{GitリポジトリURL}` を指定して実行すればOKです。`oc new-app` のログには Open Liberty S2I Builder が選択されます。

```bash
$ oc new-app --name liberty-s2i-demo --context-dir finish openliberty:23.0.0.5-java17~https://github.com/OpenLiberty/guide-rest-intro.git -e DEFAULT_HTTP_PORT=9080 -e DEFAULT_HTTPS_PORT=9443 -e APP_CONTEXT_ROOT=/guide-rest-intro

--> Found image 9314d9c (8 days old) in image stream "s2i-demo-project/openliberty" under tag "23.0.0.5-java17" for "openliberty:23.0.0.5-java17"

    Open Liberty S2I Builder 
    ------------------------ 
    Open Liberty S2I Image

    Tags: runner, builder, openliberty, javaee

    * A source build using source code from https://github.com/OpenLiberty/guide-rest-intro.git will be created
      * The resulting image will be pushed to image stream tag "liberty-s2i-demo:latest"
      * Use 'oc start-build' to trigger a new build

--> Creating resources ...
    imagestream.image.openshift.io "liberty-s2i-demo" created
    buildconfig.build.openshift.io "liberty-s2i-demo" created
    deployment.apps "liberty-s2i-demo" created
    service "liberty-s2i-demo" created
--> Success
    Build scheduled, use 'oc logs -f buildconfig/liberty-s2i-demo' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/liberty-s2i-demo' 
    Run 'oc status' to view your app.
```


### ビルドログの確認

`oc new-app` コマンドは自動的にビルドを開始します。コンテナイメージのビルド状態を最初に確認します。STATUS列が `Complete` になればビルド成功です。

```bash
$ oc get build     
NAME                                          TYPE     FROM          STATUS     STARTED             DURATION
build.build.openshift.io/liberty-s2i-demo-1   Source   Git@0187347   Complete   About an hour ago   2m11s
```

続いてビルドログを確認します。s2iスクリプトに記述されたコマンドが順次実行され、コンテナイメージがプッシュされたというメッセージが表示されているはずです。

```bash
$ oc logs build/liberty-s2i-demo-1
Cloning "https://github.com/OpenLiberty/guide-rest-intro.git" ...
...
STEP 1/9: FROM image-registry.openshift-image-registry.svc:5000/s2i-demo-project/openliberty@sha256:c9b5eb02fcc5d812fae9f99f9d577824da26315b3a0250438446671b5874002a
...
STEP 8/9: RUN /usr/local/s2i/assemble
Running s2i assemble with user  home /home/default
Artifacts Directory: target
Found pom.xml... attempting to build with 'mvn package -Popenshift -DskipTests'
...
Copying provided server.xml to /opt/ol/wlp/usr/servers/defaultServer
Configuring Server
Application deployment finished!
...
Push successful
```

## 稼働確認

s2iでビルドされたコンテナイメージはPodとしてデプロイされています。OpenShift外部からアクセスして稼働確認しましょう。

`oc new-app` コマンドでServiceリソースが作成済みです。Serviceを参照するRouteリソースを次のコマンドで作成します。

```bash
$ oc expose svc/liberty-s2i-demo
route.route.openshift.io/liberty-s2i-demo exposed
```

作成されたRouteリソースからホスト名を取得してアクセスします。

```bash
$ DEMO_ROUTE_HOST=$(oc get route/liberty-s2i-demo -o jsonpath='{.spec.host}')

$ curl http://$DEMO_ROUTE_HOST/guide-rest-intro/system/properties
{
  "java.runtime.name": "IBM Semeru Runtime Open Edition",
  "java.version": "17.0.7",
  "java.vm.vendor": "Eclipse OpenJ9",
  "java.vm.version": "openj9-0.38.0",
  ...
}
```

Podのログにもビルドしたアプリケーションの稼働ログが出力されています。

```bash
$ oc logs pod/liberty-s2i-demo-aaaaaaa

Launching defaultServer (Open Liberty 23.0.0.5/wlp-1.0.77.cl230520230514-1901) on Eclipse OpenJ9 VM, version 17.0.7+7 (en_US)
[AUDIT ] CWWKE0001I: The server defaultServer has been launched.
[AUDIT ] CWWKG0093A: Processing configuration drop-ins resource: /opt/ol/wlp/usr/servers/defaultServer/configDropins/defaults/keystore.xml
[AUDIT ] CWWKG0093A: Processing configuration drop-ins resource: /opt/ol/wlp/usr/servers/defaultServer/configDropins/defaults/open-default-port.xml
[AUDIT ] CWWKZ0058I: Monitoring dropins for applications.
[ERROR ] CWWKZ0013E: It is not possible to start two applications called guide-rest-intro.
[AUDIT ] CWWKT0016I: Web application available (default_host): http://liberty-s2i-demo-67c7798d88-c8ghg:9080/guide-rest-intro/
[AUDIT ] CWWKZ0001I: Application guide-rest-intro started in 0.836 seconds.
[AUDIT ] CWWKF0012I: The server installed the following features: [cdi-4.0, jndi-1.0, jsonb-3.0, jsonp-2.1, restfulWS-3.1, restfulWSClient-3.1].
[AUDIT ] CWWKF0011I: The defaultServer server is ready to run a smarter planet. The defaultServer server started in 2.714 seconds.
```

> `[ERROR ] CWWKZ0013E ...` というエラーログが出ていますが、本記事で扱ったアプリケーションは問題なく稼働していました。

---

以上。