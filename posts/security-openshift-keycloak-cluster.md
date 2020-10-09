# k8sやOpenShiftでKeycloakのクラスタを組むにはHeadless Serviceがいる

最近OpenShiftを触る機会があり、認証・認可サーバのOSSであるKeycloakをOpenShiftで動かした時に、クラスタを組むのにService周りでハマりました。
解決のために調べたことをメモ書きします。


## TL; DR

* 外部公開用のServiceを使ってDNS_PINGでクラスタを作成しても、Podごとにクラスタができてしまう。
* 外部公開用のServiceとは別に、クラスタ内のPod間で疎通するためのHeadless Serviceを作成しよう。


## 動作確認

CodeReady Container (crc: 1.16.0+bf72d3a, OpenShift: 4.5.9) で動作確認を行いました。
これ以外の環境では別途確認をお勧めします。


## サンプルを流用してクラスタを作成する...しかしできない

まずはKeycloakの公式ドキュメント ["Keycloak on Openshift"](https://www.keycloak.org/getting-started/getting-started-openshift) で紹介されているTemplateを使用してクラスタを作成してみます。

### Pod1つで動かす

* サンプル通りまずはPodを1つだけ立てるパターンを確認します。
* ログを確認するとクラスタを作成しています。最初に立てるPodなのでほかのクラスタのメンバーが見つからないというメッセージが出力されています。

```
$ oc get po -w
NAME                     READY   STATUS      RESTARTS   AGE
keycloak-demo-1-deploy   0/1     Completed   0          5m26s
keycloak-demo-1-n8gpd    1/1     Running     0          5m23s

$ oc logs keycloak-demo-1-n8gpd -f
(...中略...)
07:26:27,828 INFO  [org.jgroups.protocols.pbcast.GMS] (ServerService Thread Pool -- 60) keycloak-demo-1-n8gpd: no members discovered after 3321 ms: creating cluster as coordinator
```

### Podを2つ以上で動かす

* 上記のTemplateで作成したDeploymentConfigをスケールさせてPod2つでクラスタは作成できるでしょうか。
* `oc scale` コマンドで追加のPodを立ち上げます。
* Podは正常に立ち上がりましたが、ログを見ると1つ目のPodと同じように新しくクラスタを作成しています。

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

* 2つのPodが1つのクラスタのメンバーになることを期待していますので、これは期待を裏切っています。


## 解決方法

### Headless Serviceを作成

* Headless Serviceを作成し、以下のようにKeycloakのPodではHeadless Serviceに対してDNSクエリを投げるように設定すれば解決できます。
* この設定はKeycloakではなく、クラスタを動かすしくみである JGroup に説明がありました。
* JGroupのドキュメントには、KubernetesやOpenShiftでクラスタを構築するにはHeadless Serviceが必要との記載がある。

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

* なお、PROTOCOLの欄には何も変更を加えていません。
* [JGroupのDNS_PINGで使用されるプロトコルがUDPと記載されている記事](https://qiita.com/t-mogi/items/ba38a614c1637a8aef93)もあるのですが、現在はTCPがデフォルトになっているためここであらためて設定する必要はありません。**(要出典)**


```
$ oc get svc
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
keycloak-demo             ClusterIP   172.25.250.178   <none>        8443/TCP   4s
keycloak-demo-discovery   ClusterIP   None             <none>        8080/TCP   4s
```

### Podを2つ以上で動かしてクラスタを組む

* 実際に設定してみると、ログ上でクラスタが見つかった旨のメッセージが出力されている。
* これにて一件落着。

```
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

---

以上。
