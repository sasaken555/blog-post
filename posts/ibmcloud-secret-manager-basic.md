IBM Cloudにシークレットを一元管理するためのサービスとしてIBM Cloud Secret Managerがベータリリースされました。
この記事ではSecret Managerのサービスの概要を理解するために、GUIでのシークレット保管・取得を通じて探ります。

[https://www.ibm.com/cloud/blog/announcements/introducing-ibm-cloud-secrets-manager-beta:embed:cite]

※ 2020/11時点でベータリリースの状態ですので、機能やUIが大幅に変更になる可能性があります。詳細は[最新のドキュメント](https://cloud.ibm.com/docs/secrets-manager?topic=secrets-manager-getting-started)を参照してください。


## IBM Cloud Secret Managerとは

IBM Cloud Secret Manager (以降、Secret Managerと記載) とは、HashiCorp Vaultをベースに構築されたシークレットを一元管理するためのサービスです。
シークレットとして、IBM CloudなどのプラットフォームのAPIキー、SaaSのAPIキー、DB接続情報、ユーザーIDとパスワードの組み合わせといった機微な資格情報を格納できます。

自分の理解では、このサービスを使うことで2つのメリットがあります。

1つ目はアプリケーション、ツール、ファイルストレージといったさまざまな場所に散らばりがちなシークレットを1箇所で管理できること。
あるアプリケーションでは環境変数、あるツールではハードコードといったシークレットが散らばった状態だと、シークレットを1つ更新するだけでも影響範囲の特定が難しくなります。1箇所にシークレットを管理することで、安全にシークレットを更新・削除できるので運用が簡単になります。

2つ目はシークレットをSecret Managerから取得するように構成することで、ソースコード内にシークレットのハードコードがなくなります。結果的に外部の人が閲覧できるリポジトリ経由でシークレットが漏洩するリスクを低減できます。GitHubにうっかりAWSの認証情報をコミットしてPublicリポジトリにさらすなんて事態を回避できます。


## Secret Managerのアーキテクチャ

冒頭のAnnouncementの図にも表記されていますが、サービスのアーキテクチャはVault on Kubernetesのようです。
シークレットはCloud Object Storageに暗号化されて管理されています。

Vaultのバックエンドの構成やKubernetes上にどうやってStatefulなサービスを動かしているのかとアーキテクチャ的に気になりますね。ただし、GUIを触る限りユーザーは内部アーキテクチャを意識することはありません。


## Secret Managerでシークレットを操作してみる

### やってみること

本題です。今回はSecret Managerから操作可能なシークレットのうち、ユーザー資格情報のシークレットをGUIから保管・取得してみます。

ちなみにSecret Managerでは以下の3種類のシークレットを操作可能です。デフォルトではIAM資格情報以外を設定なしで保管できます。
IAM資格情報の管理は別の記事で書きます。

* ユーザー資格情報 (ユーザーIDとパスワードの組み合わせ)
* IAM資格情報 (IBM CloudのService IDに紐づいたAPIキー)
* 任意の資格情報 (上記以外のテキストで表現できる値)

### シークレットの保管

まずはSecret Managerのインスタンスを作成します。
カタログからIBM Cloud Secret Managerのライトプランをオーダーすると、Secret Managerのトップページが表示されます。

ここから左メニューの「秘密」から「追加」→「User Credentials」(ユーザー資格情報)を選択すると、
以下のスクショの通りユーザー資格情報の入力画面が表示されます。
軽く触るだけにとどめたいので、必須項目のName、Username、Passwordのみ入力してシークレットを保管します。

シークレットを保管したら、一覧に先ほど入力したユーザー資格情報が1行追加されています。

また、Edit detailsを押すとNameとは別にID (UUID) が採番されています。
IDはシークレット取得時に使用するので、クリップボードにコピーしておきます。

### シークレットの取得

シークレットを保管できたので、続いてシークレットをSecret Managerから取得します。
シークレットの取得はREST APIから呼び出します。
なんとSecret ManagerはSwagger UIが事前構成されているので、REST APIを試すのも簡単です。

ではSwagger UIでREST APIを呼び出しましょう。
APIを呼び出すにはアクセストークンが必要です。
`ibmcloud`コマンドでログインとアクセストークンの取得を事前に済ませておきます。
アクセストークンはSwagger UIの「Authorize」ボタンを押すと出現するポップアップに入力すればOKです。

```:bash
# ログインユーザーの権限をもったアクセストークンをクリップボードにコピー
$ ibmcloud iam oauth-tokens --output json \
    | jq -r '.iam_token' \
    | awk '{print $2}' \
    | pbcopy
```

準備は整ったのでREST APIを呼び出します。
シークレットの取得は、 `/api/v1/secrets/{secret_type}/{id}` エンドポイントです。
`id`はシークレット保管時に採番されたIDを指定します。

REST APIを呼び出すと、GUIから登録したUsernameとPasswordを取得できました。
これでシークレットを使用するクライアントは、シークレットをハードコードする代わりにIDからシークレットを取得して使えそうです。

---

次の記事では、Vaultの特徴を活かしたIAM資格情報の動的生成を実施してみます。

以上。
