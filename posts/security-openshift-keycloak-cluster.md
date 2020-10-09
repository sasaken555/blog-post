# k8sやOpenShiftでKeycloakのクラスタを組むにはHeadless Serviceがいる

最近OpenShiftを触る機会があり、認証・認可サーバのOSSであるKeycloakをOpenShiftで動かした時に、クラスタを組むのにService周りでハマりました。
解決のために調べたことをメモ書きします。


## TL; DR

* 外部公開用のServiceを使ってDNS_PINGでクラスタを作成しても、Podごとにクラスタができてしまう。
* 外部公開用のServiceとは別に、クラスタ内のPod間で疎通するためのHeadless Serviceを作成しましょう。


## 動作確認

CodeReady Containerで動作確認を行いました。
これ以外の環境では別途確認をお勧めします。

* crc: 1.16.0+bf72d3a
* OpenShift: 4.5.9


## サンプルを流用してクラスタを作成する...しかしできない

まずはKeycloakの公式ドキュメント ["Keycloak on Openshift"](https://www.keycloak.org/getting-started/getting-started-openshift) で紹介されているTemplateを使用してクラスタを作成してみます。

### Pod1つで動かす

サンプル通りまずはPodを1つだけ立てるパターンを確認します。
上記のTemplateを `oc process` コマンドで適用して各種リソースを作成します。

```
$ oc get po -w
NAME                     READY   STATUS      RESTARTS   AGE
keycloak-demo-1-deploy   0/1     Completed   0          5m26s
keycloak-demo-1-n8gpd    1/1     Running     0          5m23s
```

ここで作成されたPodのログを確認するとクラスタを作成していることがわかります。
最初に立てるPodなのでほかのクラスタのメンバーが見つからないというメッセージが出力されます。

```
$ oc logs keycloak-demo-1-n8gpd -f
(...中略...)
07:26:27,828 INFO  [org.jgroups.protocols.pbcast.GMS] (ServerService Thread Pool -- 60) keycloak-demo-1-n8gpd: no members discovered after 3321 ms: creating cluster as coordinator
```

### Podを2つ以上で動かす

上記のTemplateで同じPodを追加すると、同じクラスタに2つのPodが含まれると予想できます。
実際に`oc scale` コマンドでDeploymentConfigを2つ目のPodを立ち上げます。

```
$ oc scale dc keycloak-demo --replicas=2

$ oc get po -w
NAME                     READY   STATUS      RESTARTS   AGE
keycloak-demo-1-deploy   0/1     Completed   0          5m26s
keycloak-demo-1-j6p5h    0/1     Running     0          8s
keycloak-demo-1-n8gpd    1/1     Running     0          5m23s
keycloak-demo-1-j6p5h    1/1     Running     0          44s

$ oc logs keycloak-demo-1-j6p5h -f
(...中略...)
07:30:35,764 INFO  [org.jgroups.protocols.pbcast.GMS] (ServerService Thread Pool -- 60) keycloak-demo-1-j6p5h: no members discovered after 3322 ms: creating cluster as coordinator
```

Podは正常に立ち上がりましたが、上記のログを見ると1つ目のPodと同じように新しくクラスタを作成しています。
2つのPodが1つのクラスタのメンバーになることを期待していますので、期待通りの結果とはなりませんでした。


## 解決方法

### Headless Serviceを作成

結論としてはHeadless Serviceを作成し、以下のようにKeycloakのPodではHeadless Serviceに対してDNSクエリを投げるように設定すれば解決できます。

```yaml
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      description: Cluster discovery http port.
    labels:
      application: "${APPLICATION_NAME}"
    name: "${APPLICATION_NAME}-discovery"
  spec:
    ports:
      - port: 8080
        targetPort: 8080
    selector:
      deploymentConfig: "${APPLICATION_NAME}"
    clusterIP: None  # Headless Service
  - apiVersion: v1
    kind: DeploymentConfig
    metadata: ...
    spec:
      replicas: 1
      selector:
        deploymentConfig: "${APPLICATION_NAME}"
      template:
        metadata: ...
        spec:
          containers:
            - env:
                - name: KEYCLOAK_USER
                  value: "${KEYCLOAK_USER}"
                - name: KEYCLOAK_PASSWORD
                  value: "${KEYCLOAK_PASSWORD}"
                - name: DB_VENDOR
                  value: "${DB_VENDOR}"
                - name: JGROUPS_DISCOVERY_PROTOCOL
                  value: dns.DNS_PING
                - name: JGROUPS_DISCOVERY_PROPERTIES  # xxx-discoveryにクラスタ疎通のPINGを投げる
                  value: "dns_query=${APPLICATION_NAME}-discovery.${NAMESPACE}.svc.cluster.local"
                # - name: JGROUPS_DISCOVERY_PROPERTIES
                #   value: "dns_query=${APPLICATION_NAME}.${NAMESPACE}.svc.cluster.local"
              image: quay.io/keycloak/keycloak:11.0.2
```

この設定はKeycloakのクラスタを組むしくみである JGroup のドキュメントに説明がありました。  
JGroupのドキュメントには、KubernetesやOpenShiftでクラスタを構築するにはHeadless Serviceが必要との記載があります。

なお、上記の設定では環境変数 `JGROUPS_DISCOVERY_PROTOCOL` に何も変更を加えていません。
[JGroupのDNS_PINGで使用されるプロトコルがUDPと記載されている記事](https://qiita.com/t-mogi/items/ba38a614c1637a8aef93)もあるのですが、現在はTCPがデフォルトになっているためここであらためて設定する必要はありません。

> JGROUPS_TRANSPORT_STACK - an optional name of the transport stack to use udp or tcp are possible values. Default: tcp
> 
> 出典: [Reliable group communication with JGroups : 6.4.15. DNS_PING](http://www.jgroups.org/manual4/index.html#_dns_ping)


### Podを2つ以上で動かしてクラスタを組む

Headless Serviceを作成してあらためてクラスタを組んでみます。
上記の通り追記・修正したTemplateを再度適用し、Podを2つ立てみます。

```
# 外部疎通用とクラスタ疎通用の2つのServiceを作成する
$ oc get svc
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
keycloak-demo             ClusterIP   172.25.250.178   <none>        8443/TCP   4s
keycloak-demo-discovery   ClusterIP   None             <none>        8080/TCP   4s

# 1つ目のPod
$ oc logs -f keycloak-demo-1-mvrfg
07:42:50,796 INFO  [org.jgroups.protocols.pbcast.GMS] (ServerService Thread Pool -- 60) keycloak-demo-1-mvrfg: no members discovered after 3037 ms: creating cluster as coordinator
07:42:51,889 INFO  [org.infinispan.CLUSTER] (MSC service thread 1-1) ISPN000078: Starting JGroups channel ejb
07:42:51,898 INFO  [org.infinispan.CLUSTER] (MSC service thread 1-1) ISPN000094: Received new cluster view for channel ejb: [keycloak-demo-1-mvrfg|0] (1) [keycloak-demo-1-mvrfg]

# 2つ目のPod
$ oc logs -f
07:45:02,173 INFO  [org.infinispan.CLUSTER] (MSC service thread 1-1) ISPN000078: Starting JGroups channel ejb
07:45:02,191 INFO  [org.infinispan.CLUSTER] (MSC service thread 1-1) ISPN000094: Received new cluster view for channel ejb: [keycloak-demo-1-mvrfg|1] (2) [keycloak-demo-1-mvrfg, keycloak-demo-1-vv5jn]
```

1つ目のPodはクラスタを新規で作成していますが、2つ目のPodは無事に既存クラスタに参加した旨のメッセージがログに出力されています。
これにて一件落着です。

---

以上。
