# IBM Cloud AppIDで独自の認証画面を使用する

IBM Cloudマネージドの認証サービス IBM Cloud AppID (以下AppID) には標準で認証画面が作られています。
OpenID Connectのプロトコルで認証を通すだけであれば標準画面で十分なのですが、時として独自の認証画面を使う場面が出てきます。

この記事では、WebアプリケーションとAppIDで認証する時に独自の認証画面を使う方法を記載します。


## 最初にやるべきこと

本題に入る前に、その認証画面は独自実装が必要でしょうか。

最初に「本当に独自実装が必要か」と問を投げたのは、私は標準画面が最も手軽かつセキュアなしくみだと理解しているからです。

標準画面はできることが限定されているものの、最初から用意されていますし、PC/モバイル両方に対応したレスポンシブなデザインです。

最後に独自画面のしくみのところでも説明しますが、標準画面を使用するのはOpenID Connectのプロトコルの中でも一番安全かつ標準的な認可コードフローになるため、OIDCのほかのフローやオレオレ独自実装するよりも安全性は高まります。

やんごとなき理由がない限りは、ここで手をやめて標準画面が使用できないか再検討してみてはいかがでしょうか。


## 独自の認証画面を使用するモチベーション

独自の認証画面を使わないといけない理由としては以下のようなモチベーションから来ると思います。(実際に私も遭遇しています)

* すでに認証UIがある。
  * OpenID Connectのようなプロトコルではなく、ただのユーザー名/パスワードで認証させた時の画面を使いまわしたいケースです。
  * 他のクラウドやオンプレミスで稼働するアプリケーションの認証をAppIDに移行した場合などがこれにあたります。

* ユーザー名の変換が必要な場合。
  * エンドユーザーの知らないIDをユーザー名にセットしてしまっているケースです。
  * AppIDは独自のユーザーディレクトリを構成する時に、「E メール・アドレスとパスワード」または「ユーザー名とパスワード」のどちらかを選択できます。
  * 後者を選んでしまった場合、ユーザー名にUUIDなどのランダムな文字列を指定してユーザー登録できてしまいます。この場合は標準画面が表示されてもエンドユーザーはユーザー名を知らないので、入力されたメールアドレスをユーザー名に変換するといったロジックを実装します。

* 標準画面のデザインが企業ブランドと合わない ~~気に入らない~~
  * 標準画面で変更できること以上に画面をカスタムしたいケースです。
  * 標準画面は、画面上下の帯の色と中央のロゴ画像しか変更できません。
  * 既存システムやブランディングデザインとまったく合わないなら、さらにカスタマイズして統一したいというニーズもあるでしょう。特にモバイルアプリケーションとか。


## 実装方法

この記事では以下の構成のWebアプリケーションを想定します。

* 言語 ... Node.js (JavaScript)
* フレームワーク ... [Express.js](https://expressjs.com/)
* AppIDとの接続 ... [IBM Cloud App ID Node.js SDK](https://github.com/ibm-cloud-security/appid-serversdk-nodejs/)

SDKで提供された `WebAppStrategy` を実装したエンドポイントに対して、画面からPOSTメソッドでユーザー名とパスワードを投げつけるだけです。

以下のサンプルの通り、実は標準画面との違いはほとんどありません。
独自画面からPOSTでパラメータを投げるか、標準画面に遷移するか。たったこれだけです。

```:javascxript
// 認証エンドポイントの定義 サンプル
// Passport.jsやAppIDのSDKの宣言は省略

// 独自画面のログイン処理
app.post(
    '/login/rop',
    bodyParser.urlencoded({ extended: false }),
    passport.authenticate(WebAppStrategy.STRATEGY_NAME, {
        successRedirect: SUCCESS_PAGE_URL,
        failureRedirect: '/login.html',
        failureFlash : true, // 認証失敗時にフラッシュ・メッセージを送り返す
    }),
    (req, res) => {
        console.log('successfully logged in!');
        res.redirect('/success.html');
    });

// 標準画面のログイン処理
app.get('/login/standard', passport.authenticate(WebAppStrategy.STRATEGY_NAME, {
    successRedirect: SUCCESS_PAGE_URL,
    forceLogin: true
}));

// 標準画面のログイン後のコールバック処理
app.get('/login/callback', passport.authenticate(WebAppStrategy.STRATEGY_NAME));
```

```:html
<!-- 独自実装のログイン画面 サンプル -->

<form action="/login/rop" method="post">
  <div>
    <label for="username-form">ユーザー名</label>
    <div>
      <input id="username-form" type="text" name="username" required />
    </div>
  </div>
  <div>
    <label for="password-form">パスワード</label>
    <div>
      <input id="password-form" type="password" name="password" required />
    </div>
  </div>
  <div>
    <button type="submit">ログイン</button>
  </div>
</form>
```


## SDKが裏でやっていること

上記のソースコードでは、独自画面の両方をSDKで処理していましたが、SDKの中ではいったい何をしているのでしょうか。

SDKは独自画面からの認証処理では、OpenID Connectの(正確にはOAuth2.0の) Resource Owner Password Credential Grant Flowを実装・実行しています。

実態としては、AppIDの[トークンエンドポイント](https://jp-tok.appid.cloud.ibm.com/swagger-ui/#/Authorization%20Server%20-%20Authorization%20Server%20V4/oauth-server.token)を呼び出しているだけです。

トークンエンドポイントは仕様が公開されているので、SDKではなくてもサーバサイドで実装可能です。


## 参考

* [IBM Cloud資料 / AppID - アプリのブランド設定](https://cloud.ibm.com/docs/appid?topic=appid-branded#branding-ui-nodejs)
* [AppIDのREST API仕様: v1.0.0](https://jp-tok.appid.cloud.ibm.com/swagger-ui/)
