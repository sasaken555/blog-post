# おうちクラウドのUbuntu VMをautoinstallで作る

今年11月からおうちクラウドと称して自宅サーバにKubernetesクラスタを構築して使っています。
仕事や興味のあるトピックを自宅で検証できる環境があるのは良いものです。

[https://ponzmild.hatenablog.com/entry/2021/11/16/205936:embed:cite]

使い始めるとやりたいことは増えてくるもので、実はここ最近は週1でクラスタを再構築しています。
多くの原因はストレージ(Rook/Ceph, Longhorn)やネットワーク(NGINX Ingress, MetalLB)に関する設定なので、下手にいじるより作り直しで対応しています。

しかし、構築当初からクラスタのNode VMだけは手作業で構築していました。そうなると毎回手作業の構築だと隙間時間ではできない場面が出てきます。もっとカジュアルに作って壊すために、Ubuntuのautoinstallを使ってVM構築を自動化するできないか検証しました。


## Ubuntu autoinstallとは

もともとUbuntu ServerにはSubiquityというインストーラがあります。
ISOから起動した時に出てくるオレンジ帯のTUIでポチポチしながらセットアップする画面を提供しています。

autoinstallはUbuntu Serverに設定ファイルを渡すことでセットアップを自動化するしくみです。
設定ファイルは[cloud-init](https://cloudinit.readthedocs.io/en/latest/index.html)を拡張した形式で記述します。

[https://ubuntu.com/server/docs/install/autoinstall:embed:cite]


## autoinstallの設定ファイルを書いてみる

ここから実際にautoinstallでVMを自動設定します。
UbuntuはUbuntu Server 20.04.3 LTSを使用します。

設定ファイルはcloud-init同様のディレクトリ構成で設定ファイルを配置します。
meta-dataファイルは中身は空のファイルです。
user-dataファイルに設定値を書き込みます。

```bash
$ tree template/
k8s-master-0
├── meta-data
└── user-data

0 directories, 2 files
```

VM構築時に実現したいことを列挙します。

* 日本語ロケール/タイムゾーン/キーボードを使用する。
* ホスト名を固定で設定する。
* SSHで接続するが、公開鍵認証のみ許可する。
* 特定のパッケージを入れる。
* インストール済みパッケージを最新化する。
* 固定のIPアドレスを振る。

これらの設定を実現する場合は、以下の内容になります。
IPアドレスの設定だけはリファレンスの `network` セクション通りに記載しても設定できませんでした。なので苦肉の策として `late-commands` セクションで直接netplanの設定ファイルを書き込むようにしています。


[https://gist.github.com/ff9398c58643833341b41afb0851a1fc:embed#Ubuntu autoinstall config]

[Ubuntu autoinstall config](https://gist.github.com/ff9398c58643833341b41afb0851a1fc)




## autoinstallで自動設定してみる

設定ファイルを書いたら、Ubuntu Serverに適用して自動インストールしてみます。
設定ファイルはISOファイルに固めた上で、CD/DVDドライブにマウントして適用すればOKです。

ISOファイルへ固める方法はワンライナーで実行可能です。以下はmacOSの場合の例です。

```bash
$ hdiutil makehybrid -o $VM_HOST_NAME.iso \
  -hfs \
  -joliet \
  -iso \
  -default-volume-name cidata \
  template
```


## 結局効果はあったのか

ありました。
今まで再構築に1時間以上かかっていましたが、30分程度になったので作業時間を半減できたことになります。
おおよその作業時間の比較を並べます。

|作業項目|Before (時間: 分)|After (時間: 分)|
|:--|--:|--:|
|VM設定値の保管|5|1|
|VM作成|4VM * 10|4VM/2並列 * 6|
|VM疎通確認|2|2|
|パッケージインストール|5|0|
|Kubernetesクラスタ構築 by Kubespray|15|15|
|疎通確認|3|3|
|合計|70|33|

一番影響が大きかったのはVMごとに手を動かす必要がないので、並列で構築できるようになったことです。VMごとの構築作業短縮と合わせて効果は絶大でした。

また、設定値を全てファイルに記載しているので、設定値を変更するときは直接SSHするのではなく、ファイルの中身を変えてあげれば良いという安心感があります。


## 参考記事

* [Ubuntu Weekly Recipe: サーバー版インストーラーに導入された自動インストール機能](https://gihyo.jp/admin/serial/01/ubuntu-recipe/0615)
* [Autoinstall Quick Start](https://ubuntu.com/server/docs/install/autoinstall-quickstart)
