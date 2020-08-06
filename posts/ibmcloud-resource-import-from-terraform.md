AWSの既存リソースをTerraformの管理下に入れる記事を見たので、IBM Cloudでも同じことできるだろうと手を動かした時のメモです。

**偉大なる元記事**
https://dev.classmethod.jp/articles/aws-with-terraform


## TL;DR

* terraform importコマンドを使えば、IBM Cloud上のリソース作成状況を `terraform.tfstate` ファイルに落とせる。
* tfstateをもとにリソース定義のtfファイルを書けばOK。


## 検証環境

* macOS Catalina : 10.15.5
* Terraform : v0.12.26
* IBM Cloud CLI : 1.1.0+cc908fe-2020-04-29T09:33:25+00:00
* IBM Cloud Provider plug-in : 1.17.1

※ Terraform as a ServiceのIBM Cloud Schematicsでは未検証です。  
※ IBM Cloud Schematicsを使う例は[Qiitaの記事](https://qiita.com/khayama/items/07266c98b32769cf84e0)が詳しい。


## やること

シンプルな例として、作成済のNoSQL DBであるCloudantをTerraformの管理下に置きたい場合を考えます。

### (1) ダミーのtfファイルを作成する

管理下におく前に、ダミーのリソース定義として中身が空の定義を作成します。

```
# Provider定義
variable "ibmcloud_api_key" {}
variable "region" {}

provider ibm {
  ibmcloud_api_key = var.ibmcloud_api_key
  region = var.region
}

# Cloudantのインスタンスを作成するリソース定義 (中身空のダミー)
resource "ibm_resource_instance" "db_cloudant" {
}
```

### (2) terraform importでCloud上のリソースの状態をterraform.tfstateに取り込む

ドキュメントの[import コマンドの説明](https://cloud.ibm.com/docs/terraform?topic=terraform-resource-mgmt-resources#rg-import)に沿ってIBM Cloud上のリソースの作成状態をローカルファイル (terraform.tfstateファイル) に取り込みます。
インポートする対象リソースのID(CRNも可)をする必要があるので、事前にIBM Cloud CLIでIDを取得しておきましょう。

```:bash
# カレントディレクトリのTerraformを初期化
$ terraform init

# IBM Cloud CLIでCloudantのインスタンスのIDを取得する。
$ ibmcloud resource service-instance ponz-cloudant --output json | jq -r '.[] | .id'
crn:v1:bluemix:public:cloudantnosqldb:jp-tok:a/123abc:7890abc::

# CloudantのインスタンスのIDを指定してImport
$ terraform import ibm_resource_instance.db_cloudant crn:v1:bluemix:public:cloudantnosqldb:jp-tok:a/123abc:7890abc::
```


### (3) terraform planで必要パラメータと差分を出力

(2)でIBM Cloud上のCloudantの状態を取り込むと、terraform.tfstateファイルの中に以下のようなJSONが出力されます。

```:json
{
  "mode": "managed",
  "type": "ibm_resource_instance",
  "name": "db_cloudant",
  "provider": "provider.ibm",
  "instances": [
    {
      "schema_version": 0,
      "attributes": {
        "crn": "crn:v1:bluemix:public:cloudantnosqldb:jp-tok:a/123abc:7890abc::",
        "id": "crn:v1:bluemix:public:cloudantnosqldb:jp-tok:a/123abc:7890abc::",
        "location": "jp-tok",
        "name": "demo-cloudant",
        "parameters": null,
        "plan": "lite",
        "resource_controller_url": "https://cloud.ibm.com/services/",
        "resource_crn": "crn:v1:bluemix:public:cloudantnosqldb:jp-tok:a/123abc:7890abc::",
        "resource_group_id": "abc...",
        "resource_group_name": "",
        "resource_name": "demo-cloudant",
        "resource_status": "active",
        "service": "cloudantnosqldb",
        "service_endpoints": null,
        "status": "active",
        "tags": [
          "env:demo"
        ],
        "timeouts": null
      }
    }
  ]
}
```

この状態でまずは `terraform plan` を打ちます。
`Error: Missing required argument` と必須項目チェックエラーが出力されますね？
チェックエラーとなったすべての値を tfstate を参考にしながら埋めていきます。
`terraform plan` で必須チェックエラーが出なくなるまで定義を埋めると以下のようなtfファイルになるはず。

```
resource "ibm_resource_instance" "db_cloudant" {
  name = "ponz-cloudant"
  service = "cloudantnosqldb"
  location = var.region
  plan = "lite"
}
```

これで完成です。まだ差分が出る場合はresourceブロック内をtfstateを参考にさらに埋めればOKです。


## IBM Cloud Provider plug-inで作成できるリソース

IBM Cloudのリソースを管理するときに作成できるリソースは2種類に大別されます。
個別設定可能なリソースと、Resource Management resourceという区分で作成できるリソースの2種類です。

前者はAPI GatewayやCloudFoundryアプリケーション、Classic Infrastructureは専用のリソース定義が用意されているのでこれらを使用すればリソースを簡単に作成できます。  
後者は個別定義されていない(ほぼ)すべてのリソースを作成するためのリソース定義です。
NoSQLのCloudantや認証・認可のSaasのAppIDなどは、こちらに分類されます。 ドキュメント上の `ibm_resource_instance` で作成できないか確認してみましょう。細かい設定は `parameters` パラメータで指定します。

1点注意すべき点として、Terraform未対応のリソースがあります。
たとえば、IBM Cloud FunctionsはIAMベースの名前空間に存在する関数やトリガを管理できません。(ただし、CloudFoundryベースはサポートされている)  
Terraform採用前にどのリソースが管理可能なのかドキュメントの確認をお勧めします。  
--> [Index of Terraform resources and data sources](https://cloud.ibm.com/docs/terraform?topic=terraform-index-of-terraform-resources-and-data-sources)


## terraform importでカバーできないこと

また、Terraformでリソース管理できても `terraform import` だけでは完全に管理下に置けない場合があります。
最初からTerraformでIBM Cloudのリソースを作成した場合と、既存リソースを `terraform import` でTerraform管理下に置いた場合の2つのtfファイルを比較すると以下のような注意点がありました。

* リソースが所属するリソースグループの指定が必要
  * terraform importした状態だと、 `resource_group_id` が空になります。
  * この状態でも `terraform plan` は差分なしと判定される。
  * デフォルトのリソースグループの場合は未指定でも良さそうだが、リソースの権限管理上は明示的に指定した方が良い。

* 認証情報 (Service Key) が紐づくリソースのID指定が必要
  * terraform importした状態だと、 `resource_instance_id` が空になる。
  * この状態でも `terraform plan` は差分なしと判定される。
  * 「どのリソースの認証情報なのか？」という紐付けがないため、このままではサービス再作成時に元に戻せないことになる。
  * 回避策として、tfstateファイルに resource_instance_idを追加し、以下のようにtfファイルにも `resource_instance_id` を指定する。

```
resource "ibm_resource_key" "key_db_cloudant_tf" {
  name = "cloudant-credential-tf"
  role = "Writer"
  resource_instance_id = ibm_resource_instance.db_cloudant.id
}
```

---

既存リソースをすべて自動でとはいかないものの、Terraform管理下におくことができました。
ちなみにAnsibleにもIBM Cloudのリソースを管理するCollectionがあるようです。
次はAnsibleの学習がてらそちらを触ってみようかな。

以上。

