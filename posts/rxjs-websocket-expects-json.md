# RxJSのWebSocket通信ではメッセージにJSONを期待している

## TL; DR

* RxJSの `rxjs/webSocket` はメッセージ送受信時にデフォルトで `JSON.parse` / `JSON.stringify` の変換を行う。
* 単純なテキストを受信する場合は、オプションの `serializer`または`deserializer`を指定すれば良い。

## 遭遇した事象

AngularからWebSocketサーバと通信する場合は、[RxJSの`rxjs/webSocket`](https://rxjs-dev.firebaseapp.com/api/webSocket/webSocket)を使用すれば簡単に実現できます。たとえばWebSocketでメッセージを受信するServiceクラスは以下のように数行で書けます。便利！

**サンプルコード**
```typescript
import { Injectable } from '@angular/core';
import { Subject } from "rxjs";
import { webSocket } from "rxjs/webSocket";

@Injectable({
  providedIn: 'root'
})
export class MessageService {
  connect(): Subject<string> {
    return webSocket({
      url: "ws://127.0.0.1:8000/",
    });
  }
}
```

しかし、サーバからシンプルなテキスト(たとえば "first message"とか)を受け取るとパースできない...これが今回解決したいことです。

`SyntaxError: Unexpected token i in JSON at position 1`

## 解決策

`rxjs/webSocket`のオプションを定義する [WebSocketSubjectConfig](https://rxjs-dev.firebaseapp.com/api/webSocket/WebSocketSubjectConfig)を参照すると、ちゃんと書いてありました。たとえば `deserializer` には以下のような説明があります。(サンプルもあった)

> deserializer: A deserializer used for messages arriving on the socket from the server. Defaults to JSON.parse.
> サーバーから受信するメッセージをデシリアライズする時に使われる関数。デフォルトは JSON.parse です。

渡ってきたテキストをそのまま呼び出し元に返すようにオプション serializer / deserializer にカスタム関数を定義すればOKです。

**サンプルコード**
```typescript
import { Injectable } from '@angular/core';
import { Subject } from "rxjs";
import { webSocket } from "rxjs/webSocket";

@Injectable({
  providedIn: 'root'
})
export class MessageService {
  connect(): Subject<string> {
    return webSocket({
      url: "ws://127.0.0.1:8000/",
      deserializer: ({ data }) => data  // ここでカスタムのdeserializer関数を渡す
    });
  }
}
```

## 参考

サンプル実装をGitHubに置きました。
https://github.com/sasaken555/angular-Websocket-sample


---
困ったら公式ドキュメントを読もうな。

以上。
