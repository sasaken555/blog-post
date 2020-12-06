# IBM Cloud Secret ManagerでAPIキーを有効期限つきで発行する

この投稿は[IBM Cloud Advent Calendar 2020](https://qiita.com/advent-calendar/2020/ibmcloud)の8日目の記事です。
IBM Cloud Secret Managerが提供されています。

## この記事で説明しないこと

* Secret Managerの概要説明
  * 概要と基本的な使い方は別記事で紹介していますので、そちらを参照ください。
  * [IBM Cloud Secret Manager (Beta)でシークレットを保管・取得する](https://ponzmild.hatenablog.com/entry/2020/11/28/002322)


## この記事で説明すること

* Secret Managerのインスタンスを立てる
* Secret ManagerのIAMシークレットエンジンを有効化する
* 有効期限つきAPIキーを発行する


## Secret Managerのインスタンスを立てる

* 公式ドキュメントを提示する
* 面倒な人向けにTerraformモジュールのサンプルを提示する


## IAMシークレットエンジンを有効化する

* シークレットエンジン用のアクセスグループ作成
* シークレットエンジン用のAPIキー発行
* Secret Manager管理画面でIAMシークレットエンジンを有効化


## 有効期限つきAPIキーを発行する

* 発行するAPIキーのアクセスグループ作成
* REST APIでAPIキーを発行
* APIキーを使ってリソースを操作する
  * [APIキーからIBM Cloud IAMトークンを発行](https://cloud.ibm.com/apidocs/secrets-manager#authentication)
  * リソース操作APIを実行 (e.g. [今月の課金取得API](https://cloud.ibm.com/apidocs/metering-reporting))
