# OpenShift CLIでKustomizeを使う

OpenShift CLIのリファレンスを眺めていたらKustomizeのマニフェストを使えそうだったので、メモ書きです。


## 環境情報

OpenShift CLI : 4.8.40


## Kustomizeマニフェストを適用する

通常のマニフェスト適用と同様に `apply` サブコマンドにKustomize用のフラグ `--kustomize` または `-k`をつければ良いです。この方法ではKustomize形式のマニフェストから素のマニフェストをビルドと適用が同時に実行されます。

以下のように `oc apply` の引数に上記フラグとエントリーポイントの kustomization.yaml が入ったディレクトリを指定します。

```bash
oc apply -k /path/to/kustomization/target
```

参考リンク：[OpenShift CLI developer command reference - "oc apply"](https://docs.openshift.com/container-platform/4.8/cli_reference/openshift_cli/developer-cli-commands.html#oc-apply)


## Kustomizeマニフェストのビルドだけする

CLI組み込みの `kustomize` サブコマンドでKustomize形式のマニフェストから素のマニフェストをビルドして標準出力に吐き出します。CLIドキュメントを見る限り、4.8系から使えるようになったサブコマンドのようです。(4.7系以前のドキュメントには記載がなかった)

この方法ではkustomizeのバイナリがなくても使えます。また、OpenShiftクラスタへの認証も不要でローカルに閉じて利用できます。あくまでKustomization形式のマニフェストのビルドだけ実行するので、別途 `apply` サブコマンド等で適用する必要があります。

生成結果を確認したり、さらにマニフェストを書き換える場合に使えそうです。

```bash
# ローカルのディレクトリ指定
oc kustomize /path/to/kustomization/target

# リモートリポジトリのディレクトリ指定
oc kustomize 'https://github.com/argoproj/argo-cd.git/manifests/core-install?ref=v2.3.4'
```

参考リンク：[OpenShift CLI developer command reference - "oc kustomize"](https://docs.openshift.com/container-platform/4.8/cli_reference/openshift_cli/developer-cli-commands.html#oc-kustomize)

---

以上。
