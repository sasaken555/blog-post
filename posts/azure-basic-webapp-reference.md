GWに[Microsoft Learn](https://docs.microsoft.com/ja-jp/learn/)を使ってMicrosoft Azureを触っていました。
Azureおもしろいですね。
今回はリファレンス・アーキテクチャの [Azure で基本的な Web アプリケーションを実行する](https://docs.microsoft.com/ja-jp/azure/architecture/reference-architectures/app-service-web-app/basic-web-app) を自分で推奨・考慮事項を実装したときに参考にしたリンク集です。
後で何回もググりそうだったので覚え書きです。

https://docs.microsoft.com/ja-jp/azure/architecture/reference-architectures/app-service-web-app/basic-web-app


## リファレンス・アーキテクチャの推奨・考慮事項

上記のリファレンス・アーキテクチャに記載されている推奨事項と考慮事項を抜き出すと要素は以下の通りです。

* スケーリング
    * 自動スケールを有効にする
* 可用性
    * DBのバックアップ
* 管理容易性
    * リソースのデプロイのプロビジョニングの自動化する
    * アプリケーションのビルド・デプロイの自動化 (CI CD)
    * デプロイスロットを使って安全なデプロイ
    * 構成情報はアプリケーション構成に記述してアプリケーションソースに記述しない
    * 診断・監視設定を有効にする
* セキュリティ
    * ネットワークレベルのアクセス制御 (IPアドレス)
    * HTTPSを矯正する (HTTPからHTTPSへのリダイレクト)
    * 認証と承認の構築
    * DBの監査


## 実装のためのリンク集

※DB周りの部分が整理できていないので、今後追記します。

### スケーリング

* 自動スケール
    * https://docs.microsoft.com/ja-jp/azure/azure-monitor/platform/autoscale-get-started

### 管理容易性

* ビルド自動化
    * [Azure App Service のカスタム Linux コンテナーを構成する](https://docs.microsoft.com/ja-jp/azure/app-service/containers/configure-custom-container)
    * [Azure App Service への継続的デプロイ -(*1)](https://docs.microsoft.com/ja-jp/azure/app-service/deploy-continuous-deployment)
    * [チュートリアル:カスタム イメージを作成し、プライベート レジストリから App Service 内で実行する](https://docs.microsoft.com/ja-jp/azure/app-service/containers/tutorial-custom-docker-image)

* デプロイ自動化
    * [Web App for Containers での継続的デプロイ](https://docs.microsoft.com/ja-jp/azure/app-service/containers/app-service-linux-ci-cd?toc=/azure/app-service/containers/toc.json)

* スロットを使って安全にデプロイ
    * [Azure App Service でステージング環境を設定する](https://docs.microsoft.com/ja-jp/azure/app-service/deploy-staging-slots)

* 構成情報
    * [Azure portal で App Service アプリを構成する](https://docs.microsoft.com/ja-jp/azure/app-service/configure-common)

* 診断・監視
    * [診断ログの有効化 (デバッグログを保存する)](https://docs.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs)

### セキュリティ

* IPアドレスによる接続元の制限
    * [Azure App Service のアクセス制限](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-ip-restrictions)

* HTTPSの強制
    * [Azure App Service で TLS/SSL バインドを使用してカスタム DNS 名をセキュリティで保護する - HTTPSの適用](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-bindings#enforce-https)

* 認証・認可を追加
    * [チュートリアル:Azure App Service でユーザーをエンド ツー エンドで認証および承認する](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-web-tutorial-auth-aad)
    * [Azure AD ログインを使用するように App Service または Azure Functions アプリを構成する](https://docs.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad)

---

リファレンス・アーキテクチャを1つみるだけでもおもしろいですね。
今後もほかのクラウドと比較しながら深掘りしてみたいです。

以上。
