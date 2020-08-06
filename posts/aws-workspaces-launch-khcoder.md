テキスト分析で使用するKH CoderのインストーラーがWindows版しかなかったので、
AWSの仮想デスクトップサービスであるAmazon WorkSpacesでインストール、チュートリアルまでやってみた時の備忘録です。

https://aws.amazon.com/jp/workspaces
https://khcoder.net

## WorkSpaceの準備

Amazon WorkSpacesはドキュメントに記載されている通り、WorkSpaceの構築に以下のサービスやリソースの準備が必要です。

* IAMロールの作成
* VPCの作成
* ADディレクトリのセットアップ
* ADディレクトリにユーザーアカウント追加
* ユーザーに招待メール送信

今回は準備に時間をかけたくないので初回プロビジョニングには高速セットアップを利用します。
高速セットアップでは、上記のセットアップとWorkSpaceの作成を一手に実行してくれます。
自分がやったときはセットアップ開始から約20分でWorkSpaceが起動して招待メールが届きました。

また、Windows仮想マシンでKH Coderを動かすことが目的なので、
OS / CPU / メモリといったスペックを選択するバンドルには `Standard with Windows 10 ` を選択します。

<figure class="figure-image figure-image-fotolife" title="Bundleには Standard with WIndows 10を選択">[f:id:accelerk:20200510170344p:plain:alt=Bundleには Standard with WIndows 10を選択]<figcaption>Bundleには Standard with WIndows 10を選択</figcaption></figure>

その他のWorkSpaceのセットアップ方法は以下の記事が詳しいです。

https://qiita.com/yamachan360/items/454c8ad9d2ce6127ba19


## 仮想デスクトップにログイン

起動済みのWorkSpace Windows仮想マシンにログインするときはWorkSpacesクライアント、またはWebブラウザからアクセスします。
(WebブラウザはWindows仮想マシンのみ、Linux仮想マシンは不可)
私は普段からmacOSを利用していたのでWorkSpacesクライアントを以下のサイトからダウンロードしました。

https://clients.amazonworkspaces.com

ドキュメントにも記述されていますが、macOS Catalinaを利用されている場合はバージョン3.0.2以上をインストールしましょう。
ググるとたまに古いバージョンをダウンロードさせるページがあるのですが、
その場合はクライアント起動後にアップデートを促すプロンプトが表示されるので指示にしたがってアップデートしましょう。

> If you use macOS 10.15 (Catalina), you must use version 2.5.11 or later of the macOS client application. Earlier versions of the client application have issues with keyboard input on Catalina. If you are using Catalina and are working with Linux WorkSpaces, we recommend using version 3.0.2 or later of the macOS client to avoid potential keyboard issues with some applications.

最後に認証コード / ユーザー名 / パスワードの３点をWorkSpacesクライアントに入力すれば無事Windows仮想デスクトップにログインできます。

<figure class="figure-image figure-image-fotolife" title="WorkSpacesクライアントで仮想デスクトップにログイン">[f:id:accelerk:20200510171835p:plain:alt=WorkSpacesクライアントで仮想デスクトップにログインできた]<figcaption>WorkSpacesクライアントで仮想デスクトップにログイン</figcaption></figure>

## KH Coderをインストール

[KH Coderのダウンロードページ](https://khcoder.net/dl3.html)からインストーラーをダウンロードして起動します。
インストーラーの指示に従えばOKです。インストール後はそのまま起動できるかと思いきや...できません。
インストール直後にKH Coderを起動すると必ずDLLが足りないというエラーメッセージが出ます。

`MSVCP100.dllが見つからなかったため、アプリケーションを開始できませんでした`

これは既知の問題なので、FAQに沿って必要なコンポーネントをダウンロードします。
https://khcoder.net/FAQ.html#dll_not_found

コンポーネントダウンロード後はWorkSpaceを再起動する必要があるので、AWS ConsoleからWorkSpaceを再起動します。
再起動中は5分程度クライアントからアクセスできないので注意です。

<figure class="figure-image figure-image-fotolife" title="WorkSpaceの再起動">[f:id:accelerk:20200510173304p:plain:alt=AWS ConsoleからWorkSpaceを再起動する]<figcaption>WorkSpaceの再起動</figcaption></figure>


## KH Coderの起動と操作

上記のインストールが完了したらKH Coderを起動できます。
[KH Coderのチュートリアル](https://www.slideshare.net/khcoder/kh-coder-3-231585670)に沿って夏目漱石の『こころ』のテキストから共起ネットワークを作成できました。パフォーマンスや描画速度、キーボードインプットのラグは全く気にならなかったので使いやすいです。

<figure class="figure-image figure-image-fotolife" title="KH Coderの共起ネットワークを描画する">[f:id:accelerk:20200510173436p:plain:alt=KH Coderの共起ネットワークを描画する]<figcaption>KH Coderの共起ネットワークを描画する</figcaption></figure>

---

Windowsでしか使えないソフトウェアは、Amazon WorkSpacesの仮想デスクトップを使えば自分のPCを汚さず簡単に実行できそうですね。
Amazon WorkSpacesと同じ仮想デスクトップサービスにMicrosoft AzureのWindows Virtual Desktopというサービスがあります。
まだ調べきれていないのですがこちらも面白そうなので、何かの折にそっちと比較してみます。

https://azure.microsoft.com/ja-jp/services/virtual-desktop

---

以上。
