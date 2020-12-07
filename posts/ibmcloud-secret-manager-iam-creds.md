# IBM Cloud Secret ManagerでAPIキーを有効期限つきで発行する

この投稿は[IBM Cloud Advent Calendar 2020](https://qiita.com/advent-calendar/2020/ibmcloud)の8日目の記事です。

IBM CloudではユーザーまたはサービスIDに紐づくAPIキーを発行・利用することでリソース操作が可能になります。
しかしユーザーやサービスIDが増えてくると、発行したどのAPIキーの権限や有効性、使用の有無の管理が面倒になります。管理を怠った結果、強力な権限を持ったAPIキーがGitリポジトリにコミットされて公開されてしまったら、リソースを削除されてサービス停止を招きかねません。

そこでIBM Cloud Secret Managerを使用してAPIキーを有効期限つきで発行・一元管理し、APIキーを悪用されないようにしてみます。


## この記事で説明すること

* Secret ManagerのAPIキー発行のしくみ
* Secret Managerのインスタンスを立てる
* Secret ManagerのIAMシークレットエンジンを有効化する
* 有効期限つきAPIキーを発行する


## この記事で説明しないこと

* Secret Managerの概要説明
  * 概要と基本的な使い方は別記事で紹介していますので、そちらを参照ください。
  * [IBM Cloud Secret Manager (Beta)でシークレットを保管・取得する](https://ponzmild.hatenablog.com/entry/2020/11/28/002322)


## Secret ManagerのAPIキー発行のしくみ

APIキーを発行する前に、Secret Managerがいったい何をしているのか簡単に説明します。

Secret ManagerのIAMシークレットエンジンは、シークレット作成リクエストを受け取るとサービスIDとAPIキーを動的に発行しています。
APIキーはサービスID、サービスIDはアクセスグループに紐付けられます。
サービスIDの権限付与にアクセスグループを利用するので、複数のAPIキー発行時も毎回同じ最小権限を持たせることが可能になっています。

ここでAPIキーに関係するエンティティのリレーションを図解します。


## Secret Managerのインスタンスを立てる

まずは公式ドキュメントに沿ってSecret Managerのインスタンスを立てるところから始めましょう。インスタンス作成方法は公式ドキュメントを参考に、カタログから任意の名前を指定するだけでOKです。

[Secrets Manager サービス・インスタンスの作成](https://cloud.ibm.com/docs/secrets-manager?topic=secrets-manager-create-instance)

コンソールをポチポチするのが面倒な人向けにTerraformでインスタンスを作成を用意しました。Terraformに慣れている方はこちらを参照してください。

[sasaken555
/
terraform-ibmcloud-secret-manager](https://github.com/sasaken555/terraform-ibmcloud-secret-manager)

## IAMシークレットエンジンを有効化する

インスタンスを作成したら、APIキーを発行するためのIAMシークレットエンジンを有効化します。
シークレットエンジンは、Secret Managerに備わったシークレットを発行するコンポーネントです。IAMシークレットエンジンはそのうちの１つです。

インスタンス作成直後はIAMシークレットエンジンが無効化されているため、有効化しましょう。
[『シークレット・エンジンの構成』ガイド](https://cloud.ibm.com/docs/secrets-manager?topic=secrets-manager-secret-engines)を参照して進めます。

ここで作成するものは以下の３つです。
前節でTerraformを使用してインスタンスを作成した場合、サービスIDとアクセス・グループはすでに作成されています。この場合はシークレットエンジン用のAPIキー発行のみ実施してください。

* サービスID
* アクセス・グループ
* シークレットエンジン用のAPIキー

### サービスID作成

最初にSecret Managerを表すサービスIDを発行します。
サービスIDは"アクセス (IAM)"メニューの"サービスID"タブから作成できます。

ここで図解します。

### アクセス・グループ作成

続いてサービスIDに紐付けるシークレットエンジン用のアクセスグループを作成します。
IAM Access Groups Service サービスにEditor役割を、IAM Idエンティティ Service サービスにOperator役割をつけます。

ここにアクセス・ポリシー一覧を図解します。

アクセス・グループを作成したら、先ほど作成したサービスIDをメンバーに加えましょう。

### シークレットエンジン用のAPIキー発行

シークレットエンジン用のAPIキーを発行します。
"アクセス (IAM)"メニューの"APIキー"タブから作成できないので、IBM Cloud CLIからAPIキーを発行しましょう。

以下のコマンドを実行します。
ここで発行されたAPIキーは2度と表示できないので、クリップボードやテキストエディタにコピーします。

```bash
$ ibmcloud iam service-api-key-create \
  <APIキーの名前> <サービスIDのUUID> \
  --description "API key for Secret Manager IAM secret engine" 
```

### IAMシークレットエンジンを有効化

最後に先ほど発行したAPIキーをシークレットエンジンに渡して有効化させます。
Secret Manager管理画面の"設定"メニューから、"IAM Secret Engine"欄を探してAPIキーを埋め込みます。

## 有効期限つきAPIキーを発行する

ここから本題です。
本記事では、発行したAPIキーからアクセストークンを生成し、アカウントの使用量(=課金額)を取得してみます。

まずSecret Managerの管理画面から"秘密"メニューを開き、APIキーを発行します。
IAMシークレットエンジンを有効化しているので、"IAM credentials"を選択できるようになっています。
"IAM credentials"を選択し、APIキーの項目を入力します。

入力欄からアカウントの使用量を取得する権限を持ったアクセスグループを指定します。以下のようにBillingサービスに"Viewer"役割を持ったアクセス・ポリシーを付け足アクセスグループを事前に作成しておきます。

また、有効期限を設定したいので最下部の"Lease Duration"欄に有効期間を設定します。今回は10分だけ有効にします。

"Add"ボタンを押下すれば、これでAPIキー発行は完了です。
シークレット一覧に表示され、詳細表示するとUUIDが割り振られていることがわかります。

## APIキーを使用する

APIキー自体はSecret Managerに保管されているので、使用時はSecret ManagerのREST APIを実行して取り出します。

Secret ManagerのREST APIは組込みSwagger UIから実行可能です。それではSwagger UIからシークレット取得APIを実行してAPIキーを取り出しましょう。
Swagger UIは `https://<インスタンスID>.<リージョン>.secrets-manager.appdomain.cloud/swagger-ui/` からアクセス可能です。

シークレット取得APIは `` エンドポイントです。
このエンドポイントに先ほど確認したUUIDを渡すことで、Secret Managerに保存されたAPIキーを取り出します。
実行すると以下のようなJSONレスポンスにAPIキー `api_key` が含まれています。

```json
{
  "metadata": {
    "collection_type": "application/vnd.ibm.secrets-manager.secret+json",
    "collection_total": 1
  },
  "resources": [
    "access_groups": ["AccessGroupId-**********"],
    "api_key": "z4a-*******************",
    "creation_date": "2020-12-07T11:00:00Z",
    "state_description": "Active",
    "ttl": 600
  ]
}
```

取り出したAPIキーからIBM Cloudをリソース操作するためにアクセストークンを発行します。

```bash
$ IAM_TOKEN=$(curl -k -X POST \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --header "Accept: application/json" \
  --data-urlencode "grant_type=urn:ibm:params:oauth:grant-type:apikey" \
  --data-urlencode "apikey=$IAM_API_KEY" \
 "https://iam.cloud.ibm.com/identity/token" | jq -r '.access_token')
```

最後にアカウントのIDと[IBM Cloud Usage Reports API](https://cloud.ibm.com/apidocs/metering-reporting)を使用して使用量を取得します。

```bash
$ ACCOUNT_ID=$(ic account show --output json | jq -r '.account_id')
$ curl -H "Authorization: $IAM_API_TOKEN" -H "Accept: application/json" "https://billing.cloud.ibm.com/v4/accounts/$ACCOUNT_ID/summary/2020-11" | jq .

{
  "account_id": $ACCOUNT_ID,
  "month": "2020-11-30T23:59:59.999Z",
  "resources": {
    "billable_cost": 0,
    "non_billable_cost": 0
  },
  ...
}
```

無事に発行したAPIキーを使用してリソースを操作できるようになりました。

## 参考

* [APIキーからIBM Cloud IAMトークンを発行](https://cloud.ibm.com/apidocs/secrets-manager#authentication)
* [IBM Cloud Usage Reports API](https://cloud.ibm.com/apidocs/metering-reporting)
