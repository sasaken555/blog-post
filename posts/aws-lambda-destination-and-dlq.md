# AWS Lambdaの処理失敗時に「送信先」と「DLQ」で連携するデータの違い

先日[Serverless Meetup Japan Virtual #0](https://serverless.connpass.com/event/179575/)に参加してきました。その時にAWS Lambdaの処理失敗時はDLQとは違う送信先を指定できると聞いたので、連携データの違いは何か調べました。

## TL; DR

* Lambdaを非同期で呼び出す場合、処理失敗した後にデータを渡す方法は「通知先(Destination)」と「DLQ (Dead Letter Queue)」の2つ。
* 通知先はリクエスト・レスポンス、関数などの詳細なデータが連携される。
* DLQはリクエストの内容のみ連携される。


## 送信先(Destination)とDLQ

AWS Lambdaは非同期呼び出しで処理成功・失敗時にデータを連携するサービスを指定できます。
この連携先のサービスを「送信先(Destination)」と呼んでいます。送信先がサポートされたのは2019/11と割と最近です。

https://aws.amazon.com/jp/about-aws/whats-new/2019/11/aws-lambda-supports-destinations-for-asynchronous-invocations

送信先としてAmazon SQS、Amazon SNS、Amazon EventBridge、別のAWS Lambda関数の4種類のサービスを選択可能です。
また、処理成功と処理失敗で送信先に別のサービスを選択可能というところが柔軟ですね。
送信先に連携されるデータは、リクエスト・レスポンス・関数そのものといった情報が含まれます。

一方でDLQ(Dead Letter Queue)は、送信先と同様に処理失敗時にデータ連携します。連携先のサービスはAmazon SQSとAmazon SNSのみです。
DLQに連携されるデータは、送信先と違ってLambdaのリクエストのみです。
ちなみにDead Letter Queueは失敗時の送信先と同時に設定可能です。


## 試してみる

この記事の本題です。ユースケースを交えて確認します。

Amazon API Gatewayから必ず失敗するAWS Lambdaを非同期でキックするREST APIを用意して呼び出してみます。
ここでは処理失敗時の送信時にAmazon SQS、DLQに別のSQSを割り当てます。

<figure class="figure-image figure-image-fotolife" title="Lambda処理失敗時のデータ連携フロー">[f:id:accelerk:20200707012315p:plain]<figcaption>Lambda処理失敗時のデータ連携フロー</figcaption></figure>

cURLでREST APIを呼びます。

```bash
curl -v https://aaaaaaaaaa.example.com/dev/async -d '{ "name": "hoge" }'
```

この時にDLQのSQSに連携されるメッセージはリクエストボディと同じく `{"name": "hoge"}` のみです。
エラータイプやリクエストIDといったトレースや最低限のエラーハンドリングに必要な情報は追加情報に付加されています。

一方で送信先のSQSに保存されるメッセージは以下のようになります。
`requestContext`にはLambdaの関数の情報、`requestPayload`はAPI Gatewayに渡されたリクエストボディです。`responsePayload`にはレスポンスボディ(今回はエラーの情報)が乗っていますね。DLQに連携されるメッセージよりもかなり詳細です。
送信先を利用するメリットとして、送信先のSQSからさらにLambda関数やエラー種別で細かくエラーハンドリングできそうです。

<figure class="figure-image figure-image-fotolife" title="処理失敗時の送信先にSQS連携されたメッセージ">[f:id:accelerk:20200707012600p:plain]<figcaption>処理失敗時の送信先にSQS連携されたメッセージ</figcaption></figure>


## 参考

[https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/invocation-async.html:embed:cite]

---

以上。

