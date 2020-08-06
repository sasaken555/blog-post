# IBM CloudのコンソールログインにMFAを適用する

IBM Cloudのコンソールへのログインに多要素認証(MFA)を適用する話。

AWSやAzureだとIAMでMFAの適用設定できるし、GCPだとGoogleアカウント自体のセキュリティ設定で同様のことができます。
ではIBM Cloudは？と一緒に仕事をしているエンジニアに聞かれたので調べたことを覚え書きします。

結論を先に言えば、コンソールのIAMの設定で有効化できます。

※ 厳密に言えばAzureはGitHubアカウントでサインインできるので、GitHubの設定でもMFAを有効化できたりします。


## MFA適用ガイド

IBM Cloudのドキュメントにやり方は書いてあります。こちらにのっとって説明します。

https://cloud.ibm.com/docs/iam?topic=iam-enablemfa&locale=ja


## やり方

(1) 上のバーの「管理」→「アクセス(IAM)」を選択

(2) アクセス(IAM)画面の左メニューから「設定」を選択

(3) 「アカウント・ログイン」のパネルの「編集」ボタンを選択

<figure class="figure-image figure-image-fotolife" title="アクセス(IAM)画面">[f:id:accelerk:20200416005317p:plain]<figcaption>アクセス(IAM)画面</figcaption></figure>

(4) 「すべてのユーザー」を選択し、「更新」ボタンを押して有効化

<figure class="figure-image figure-image-fotolife" title="MFA設定ダイアログ">[f:id:accelerk:20200416005401p:plain]<figcaption>MFA設定ダイアログ</figcaption></figure>

(5) 1回ログアウト (MFAが有効になるまで5分程度かかる)

(6) IBMidで再度ログイン

(7) MFAの入力画面が出るので、「オーセンティケーター・アプリケーションをセットアップします」のリンクを押して設定用のQRコードを表示する

<figure class="figure-image figure-image-fotolife" title="検証コード入力・設定画面">[f:id:accelerk:20200414075337p:plain]<figcaption>検証コード入力・設定画面</figcaption></figure>

(8) Google AuthenticatorやIBM Verifyなどのオーセンティケーター・アプリケーションを用意して、上記(7)のQRコードを読み込む。1つ罠があるのでのちほど説明。

(9) もう一度検証コード入力画面に戻り、(8)で設定した検証コードを入力する。これでログイン時にMFAできました🎉🎉


## MFA設定リンクの罠

上記手順で注意すべきは、 <b>手順(7)(8)はアカウントにつき１回しか設定できない点です。</b>

「オーセンティケーター・アプリケーションをセットアップします」のリンクを２回目以上踏むと、
以下のスクショのように設定済みのエラーメッセージが出てきます。
１回リンクを踏んで「後で設定しよう」ができないので、これをやってしまうとコンソールにログイン不可能になります。

また、オーセンティケーター・アプリケーションから設定情報を削除しても同様です。何と再設定できません。

もしやらかしたら[Help Desk](https://www.ibm.com/ibmid/myibm/help/jp/helpdesk.html)に問い合わせてMFAを無効化してもらいましょう。
(私は過去２回やらかして問い合わせました)

<figure class="figure-image figure-image-fotolife" title="オーセンティケーター・アプリの設定は１回しかできない">[f:id:accelerk:20200416010130p:plain]<figcaption>検証コードアプリの設定は１回しかできない</figcaption></figure>

---

MFAはコンソールへのログイン確認と同時に設定したい(してもらいたい)ので、手順が簡単なのはありがたいですね。
以上。
