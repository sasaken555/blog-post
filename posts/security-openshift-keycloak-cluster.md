# k8sやOpenShiftでKeycloakのクラスタを組むにはHeadless Serviceがいる

OpenID Connectのプロトコルで認証させるためにKeycloakをKubernetes / OpenShiftで動かした時に、クラスタを組むのにService周りでハマった。
解決のために調べたことをメモ書きする。


## TL; DR

* 外部公開用のServiceを使ってDNS_PINGでクラスタを作成しても、Podごとにクラスタができてしまう。
* 外部公開用のServiceとは別に、クラスタ内のPod間で疎通するためのHeadless Serviceを作成しよう。


## 動作確認

CodeReady Container (OpenShift 4.5.9)で動作確認を行った。
これ以外の環境では別途確認をお勧めします。


## サンプルを流用してクラスタを作成する...しかしできない

まずはKeycloakの公式ドキュメント ["Keycloak on Openshift"](https://www.keycloak.org/getting-started/getting-started-openshift) で紹介されているTemplateを使用してクラスタを作成してみます。

### Pod1つで動かす

* サンプル通りまずはPodを1つだけ立てるパターンを確認します。
* ログを確認すると、クラスタを作成していました。
* Podが1つしかないので、クラスタである意味はない。

### Podを2つ以上で動かす

* それぞれPodは立ち上がる。
* ログを見ると、クラスタができていない。「クラスタが見つからないから新しく作成する」と出力されている。


## 解決方法

### Headless Serviceを作成

* Headless Serviceを作成して、このServiceに対してDNSクエリを投げるように設定すれば良い。
* Keycloakではなく、クラスタを動かすしくみである JGroup にHeadless Serviceが必要との記載がある。
* [参考記事](https://qiita.com/t-mogi/items/ba38a614c1637a8aef93)あり。この記事ではJGroupがクラスタを組むために使用しているプロトコルがUDPと記載されているが、現在はTCPがデフォルトになっているためここであらためて設定する必要はない。**(要出典)**

### Podを2つ以上で動かしてクラスタを組む

* 実際に設定してみると、ログ上でクラスタが見つかった旨のメッセージが出力されている。
* これにて一件落着。

---

以上。
