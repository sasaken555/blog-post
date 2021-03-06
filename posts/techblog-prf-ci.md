# 技術文章の文章校正をtextlintとGitHub Actionsで快適にする

8月前半に社内で『テックブログの文章校正をCIで快適にする』というタイトルでLTをしました。
もともとは技術同人誌を執筆する時に校正作業を自動化していたこともあり、ブログに応用した例の発表です。
発表から２週間ほど経ってしまったので、忘れないようにセルフ文字起こしです。


## 文章校正で解決したいこと

まず自分なりに「なぜ文章校正をCIで実行させたかったか」をまとめます。
ブログや技術同人誌のような外向けに文章の校正作業は、軽微な修正が繰り返して面倒です。
たとえば以下の作業が面倒な校正作業です。

* 読みやすさのためにバラついた文体を整える。
  * 「〜です/ます。」「〜である/なのだ。」「〜やねん。」

* 固有名詞のタイポや表記ゆれを直す。
  * 「サーバ」なのか、「サーバ」なのか、、、(好みによる)

* 修正リクエストを反映して上記を繰り返す。
  * 他人の文章と自分の文章の書き方を統一するのは骨が折れる。

上記のように軽微で繰り返し発生する作業は人間がやることじゃないですね。
なるべく執筆に専念したいところです。これが文章校正CIを作りたかったモチベーションです。


## textlintとは

上記の文章校正をCIに組み込める最適なツールが[textlint](https://github.com/textlint/textlint)です。

textlintはテキストファイルとMarkdown文章に対して、事前定義したルールに違反する文章を検出する校正ツールです。
Node.jsのパッケージで提供されているので、プログラムから起動するのに適しています。
また、公開されている校正ルールを複数取り込むことで細かい校正ルールを自分で作れます。

今回はtextlintと、私が技術同人誌2冊とブログ執筆で使っている以下の校正ルールを組み合わせます。

* textlint-ja/textlint-rule-preset-ja-technical-writing
  * 日本語の技術文書を書くためのtextlintのrule-preset
  * e.g. 一文は100文字以下、二重否定は使用しない、半角カナを使わせない、など。

* textlint-rule/textlint-rule-prh
  * 表記ゆれに強い文章校正ツールprh (proofreading helper)を組み合わせたルールです。
  * textlintと組み合わせて細かい校正ルールを適用できます。


## textlintを実行してみる

textlintをCIへ組み込む前に、ローカルマシン上でtextlintを実行して文章校正してみます。
事前にNode.jsをインストールします。

textlint実行の下準備は以下3ステップになります。

* npmでtextlintとルールをインストールします。

* 技術評論社の書籍の校正ルールをダウンロードして配置します。
  * prh/rulesのリポジトリにある。
  * https://github.com/prh/rules/blob/master/media/WEB%2BDB_PRESS.yml

* `.textlintrc` に使用する校正ルールを定義します。
  * 先にインストールしたルールをオプションとともにファイルに記述します。

これで準備は整いました。npmスクリプトからtextlintを実行してみます。
実際に過去のブログをtextlintにかけてみた例は以下の通りです。ルール違反が一眼で分かりますね。

※ここに実行結果の画像を貼り付けます。


## GitHub Actionsにtextlintを組み込む

textlintも準備できたので、校正CIを構築します。
CIツールは実プロジェクトですとクラウドのCIサービスを使うケースが多いですが、今回はブログ記事をGitHubに集約していたのでGitHub Actionsを採用します。

GitHub Actionsを使用して文章校正CIは以下の流れで実行させます。

* Markdownで文章を書きます。
* 編集が終わったらGitリポジトリにcommit&pushします。
* リポジトリへpushされると、GitHub Actionsのワークフローが起動してtextlintを実行します。
* 文章にルール違反があればエラーで落ちます。

GitHub ActionsのワークフローはYAMLで定義します。
書き方はドキュメントや書籍が出ていますのでそちらを参考にしてください。一例として[私が作成したCIワークフローの定義](https://github.com/sasaken555/blog-post/blob/master/.github/workflows/lint_markdowns.yml)がGitHubに上がっています。

ワークフローの定義をコミットした状態でリポジトリにソースコードをプッシュすると、ワークフローが実行されます。

試しに校正ルール違反の文章をプッシュした時のワークフロー実行結果は以下の通りです。

※ここに実行結果の画像を貼り付けます。

無事検出されました。コミットのページを見ると成功はチェック、失敗はバツがついているので分かりやすいですね。


## まとめ

文章校正はCIで快適にできる。


## 参考

* 文章校正で使えるツール
  * https://github.com/textlint/textlint
  * https://github.com/textlint-ja/textlint-rule-preset-ja-technical-writing
  * https://github.com/prh/prh
  * https://github.com/textlint-rule/textlint-rule-prh

* textlintとprhで文章を校正する方法
  * https://qiita.com/munieru_jp/items/83c2c44fcadb177d2806

* 校正CIのサンプルリポジトリ
  * https://github.com/sasaken555/blog-post


---

9月から技術書典9が始まりますね。私も既刊のみですが参加します。
この記事で取り上げた校正CIが、技術同人誌の校正作業につまずいている方の助けになればうれしいです。

[技術書典 : 技術書オンリーイベント](https://techbookfest.org/)

以上。
