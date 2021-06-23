---
title: Install Istio with an External Control Plane
description: Install Istio with an external control plane and a remote cluster data plane.
weight: 50
aliases:
    - /docs/setup/additional-setup/external-controlplane/
    - /latest/docs/setup/additional-setup/external-controlplane/
keywords: [external,control,istiod,remote]
owner: istio/wg-environments-maintainers
test: yes
---

このガイドは、{{< gloss >}}external control plane{{< /gloss >}}をインストールして、一つまたはそれ以上の
{{< gloss "remote cluster" >}}remote clusters{{< /gloss >}}を接続するプロセスを案内します。  
外部コントロールプレーン[deployment model](/docs/ops/deployment/deployment-models/#control-plane-models)では
メッシュオペレータは、メッシュを構成するデータプレーンクラスタ（または複数クラスタ群）とは別の外部のクラスタにコントロールプレーンをインストールして管理することができます。  
このデプロイメントモデルでは、メッシュオペレータとメッシュ管理者間の明確な分離ができます。
メッシュオペレータはIstioコントロールプレーンをインストール・管理しますが、メッシュ管理者はメッシュを設定するだけでよいです。

{{< image width="75%"
    link="external-controlplane.svg"
    caption="External control plane cluster and remote cluster"
    >}}

リモートクラスタで稼働しているEnvoyプロキシ(サイドカーとゲートウェイ)は、ingress gateway経由で外部のistiodにアクセスします。
このingress gatewayは、ディスカバリ、CA、インジェクション、検証のために必要なエンドポイントを公開しています。

外部コントロールプレーンの設定と管理は外部クラスタのメッシュオペレータによっておこなわれますが、
（外部コントロールプレーンに接続した）一つ目のリモートクラスタは、メッシュ自身の設定クラスタとして動作します。
メッシュ管理者は設定クラスタを使って、メッシュサービス自身を設定するのに加えて、メッシュリソース（ゲートウェイ、仮想サービス、他）を設定します。  
外部コントロールプレーンは上図のようにリモートからKubernetes APIサーバ経由でこの設定にアクセスします。


## Before you begin

### Clusters

このガイドでは、2つのKubernetesクラスタが必要です。
Kuberentesクラスタは次のサポートバージョンのいずれか：{{< supported_kubernetes_versions >}}

最初のクラスタには、`external-istiod`ネームスペースに{{< gloss >}}external control plane{{< /gloss >}}がインストールされます。
また`istio-system`ネームスペースに、ingress gatewayもインストールされ外部コントロールプレーンへのクラスタを跨いだ接続を可能にします。

2つ目のクラスタは、{{< gloss >}}remote cluster{{< /gloss >}}です。メッシュアプリケーションのワークロードが動きます。
このKubernetes APIサーバは、外部コントロールプレーン(istiod)がワークロードのプロキシ設定をするのに使うメッシュ設定を提供します。

### API server access

リモートクラスタのKubernetes APIサーバは、外部コントロールプレーンのクラスタから接続できないといけません。
多くのクラウドプロバイダーはネットワークロードバランサー(NLB)経由で、APIサーバにパブリックな接続ができるようにしています。
APIサーバに直接接続できない場合、接続できるようにインストール手順を修正する必要があります。
例えば、[multicluster configuration](#adding-clusters)で使われる[east-west](https://en.wikipedia.org/wiki/East-west_traffic) gatewayを使って
APIサーバへの接続をできるようにもできます。

### Environment Variables

手順をシンプルにするために、全体を通じて次の環境変数を使います。

Variable | Description
-------- | -----------
`CTX_EXTERNAL_CLUSTER` | The context name in the default [Kubernetes configuration file](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) used for accessing the external control plane cluster.
`CTX_REMOTE_CLUSTER` | The context name in the default [Kubernetes configuration file](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) used for accessing the remote cluster.
`REMOTE_CLUSTER_NAME` | The name of the remote cluster.
`EXTERNAL_ISTIOD_ADDR` | The hostname for the ingress gateway on the external control plane cluster. This is used by the remote cluster to access the external control plane.
`SSL_SECRET_NAME` | The name of the secret that holds the TLS certs for the ingress gateway on the external control plane cluster.

ここで`CTX_EXTERNAL_CLUSTER`, `CTX_REMOTE_CLUSTER`, `REMOTE_CLUSTER_NAME` を設定しましょう。それ以外は後で設定します。

{{< text syntax=bash snip_id=none >}}
$ export CTX_EXTERNAL_CLUSTER=<your external cluster context>
$ export CTX_REMOTE_CLUSTER=<your remote cluster context>
$ export REMOTE_CLUSTER_NAME=<your remote cluster name>
{{< /text >}}

## Cluster configuration

### Mesh operator steps

メッシュオペレータは、外部クラスタに、外部Istioコントロールプレーンのインストールと管理に責任があります。
これは外部クラスタのingress gatewayを設定することも含みます。リモートクラスタに、コントロールプレーンへのアクセスを許可し、
外部コントロールプレーンを使うように、リモートクラスタのサイドカーインジェクターwebhook設定をインストールできるようにします。

#### Set up a gateway in the external cluster

1. Istioインストール設定を作成します。この設定は外部コントロールプレーンのポートを他のクラスターに公開します。

    {{< text bash >}}
    $ cat <<EOF > controlplane-gateway.yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    metadata:
      namespace: istio-system
    spec:
      components:
        ingressGateways:
          - name: istio-ingressgateway
            enabled: true
            k8s:
              service:
                ports:
                  - port: 15021
                    targetPort: 15021
                    name: status-port
                  - port: 15012
                    targetPort: 15012
                    name: tls-xds
                  - port: 15017
                    targetPort: 15017
                    name: tls-webhook
    EOF
    {{< /text >}}

    そして、外部クラスタの`istio-system`ネームスペースにゲートウェイをインストールします。

    {{< text bash >}}
    $ istioctl install -f controlplane-gateway.yaml --context="${CTX_EXTERNAL_CLUSTER}"
    {{< /text >}}

1. 次のコマンドを実行して、ingress gatewayが起動してRunning状態であることを確認する

    {{< text bash >}}
    $ kubectl get po -n istio-system --context="${CTX_EXTERNAL_CLUSTER}"
    NAME                                   READY   STATUS    RESTARTS   AGE
    istio-ingressgateway-9d4c7f5c7-7qpzz   1/1     Running   0          29s
    istiod-68488cd797-mq8dn                1/1     Running   0          38s
    {{< /text >}}

    `istio-system` ネームスペースにistiod デプロイメントも作成されていることに気づくでしょう。これはingress gatewayを設定するのに使われます。リモートクラスタから使われるコントロールプレーンではありません。

    {{< tip >}}
    この例では、 `external-istiod` ネームスペースに一つの外部istiodをデプロイするだけですが、
    外部クラスタの違うネームスペースに複数の外部コントロールプレーンをホストするように、
    ingress gatewayを設定することもできます。
    {{< /tip >}}

1. Istio ingress gatewayサービスを公開ホスト名とTLSで公開するために環境変数を設定する
  `EXTERNAL_ISTIOD_ADDR`環境変数にホスト名を、`SSL_SECRET_NAME`環境変数にTLS証明書を保持するシークレット名を設定する。

    {{< text syntax=bash snip_id=none >}}
    $ export EXTERNAL_ISTIOD_ADDR=<your external istiod host>
    $ export SSL_SECRET_NAME=<your external istiod secret>
    {{< /text >}}

#### Set up the remote config cluster

1. リモートクラスタのIstioインストール設定を作成します。この設定は、ローカルでなく外部コントロールプレーンのインジェクターを使うインジェクションフックをインストールします。
  このクラスタは設定クラスタとしても動作するので、 
  `base.enabled` と `pilot.configMap`を`true`に設定することで、Istio CRDと `istio`コンフィグマップ（例：グローバルメッシュ設定）もインストールされます。

    {{< text syntax=bash snip_id=get_remote_config_cluster_iop >}}
    $ cat <<EOF > remote-config-cluster.yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    metadata:
      namespace: external-istiod
    spec:
      profile: external
      components:
        base:
          enabled: true
      values:
        global:
          istioNamespace: external-istiod
        pilot:
          configMap: true
        istiodRemote:
          injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/inject/:ENV:cluster=${REMOTE_CLUSTER_NAME}:ENV:net=network1
        base:
          validationURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/validate
    EOF
    {{< /text >}}

    そして、リモートクラスタにその設定をインストールする。

    {{< text bash >}}
    $ kubectl create namespace external-istiod --context="${CTX_REMOTE_CLUSTER}"
    $ istioctl manifest generate -f remote-config-cluster.yaml | kubectl apply --context="${CTX_REMOTE_CLUSTER}" -f -
    {{< /text >}}

1. リモートクラスタのwebhook設定がインストールされたのを確認する

    {{< text bash >}}
    $ kubectl get mutatingwebhookconfiguration --context="${CTX_REMOTE_CLUSTER}"
    NAME                                     WEBHOOKS   AGE
    istio-sidecar-injector-external-istiod   4          6m24s
    {{< /text >}}

#### Set up the control plane in the external cluster

1. 外部コントロールプレーンをホストする`external-istiod`ネームスペースを作成する

    {{< text bash >}}
    $ kubectl create namespace external-istiod --context="${CTX_EXTERNAL_CLUSTER}"
    {{< /text >}}

1. 外部クラスタのコントロールプレーンが、サービス、エンドポイント、ポッド属性を発見できるようにリモートクラスタへのアクセスが必要です。
リモートクラスタの`kube-apiserver`にアクセスするクレデンシャルが入ったシークレットを作成し、外部クラスタにインストールする。

    {{< text bash >}}
    $ kubectl create sa istiod-service-account -n external-istiod --context="${CTX_EXTERNAL_CLUSTER}"
    $ istioctl x create-remote-secret \
      --context="${CTX_REMOTE_CLUSTER}" \
      --type=config \
      --namespace=external-istiod | \
      kubectl apply -f - --context="${CTX_EXTERNAL_CLUSTER}"
    {{< /text >}}

1. Istio設定を作成します。この設定は、外部クラスタの`external-istiod` ネームスペースにコントロールプレーンをインストールします。
   istiodは、ローカルでマウントされた `istio`コンフィグマップと、`SHARED_MESH_CONFIG`環境変数（値は`istio`と設定されている）を使うように設定されていることに注意してください。
   この指定はistiodに設定クラスタのコンフィグマップにメッシュ管理者によって設定された設定値と、メッシュオペレータによって設定されたローカルコンフィグマップの設定値をマージするよう、何か不整合があった場合にはローカルコンフィグマップの設定値を優先するよう指示します。   

    {{< text syntax=bash snip_id=get_external_istiod_iop >}}
    $ cat <<EOF > external-istiod.yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    metadata:
      namespace: external-istiod
    spec:
      profile: empty
      meshConfig:
        rootNamespace: external-istiod
        defaultConfig:
          discoveryAddress: $EXTERNAL_ISTIOD_ADDR:15012
          proxyMetadata:
            XDS_ROOT_CA: /etc/ssl/certs/ca-certificates.crt
            CA_ROOT_CA: /etc/ssl/certs/ca-certificates.crt
      components:
        pilot:
          enabled: true
          k8s:
            overlays:
            - kind: Deployment
              name: istiod
              patches:
              - path: spec.template.spec.volumes[100]
                value: |-
                  name: config-volume
                  configMap:
                    name: istio
              - path: spec.template.spec.volumes[100]
                value: |-
                  name: inject-volume
                  configMap:
                    name: istio-sidecar-injector
              - path: spec.template.spec.containers[0].volumeMounts[100]
                value: |-
                  name: config-volume
                  mountPath: /etc/istio/config
              - path: spec.template.spec.containers[0].volumeMounts[100]
                value: |-
                  name: inject-volume
                  mountPath: /var/lib/istio/inject
            env:
            - name: INJECTION_WEBHOOK_CONFIG_NAME
              value: ""
            - name: VALIDATION_WEBHOOK_CONFIG_NAME
              value: ""
            - name: EXTERNAL_ISTIOD
              value: "true"
            - name: CLUSTER_ID
              value: ${REMOTE_CLUSTER_NAME}
            - name: SHARED_MESH_CONFIG
              value: istio
      values:
        global:
          caAddress: $EXTERNAL_ISTIOD_ADDR:15012
          istioNamespace: external-istiod
          operatorManageWebhooks: true
          meshID: mesh1
    EOF
    {{< /text >}}

    そして、外部クラスタにIstio設定を適用する

    {{< text bash >}}
    $ istioctl install -f external-istiod.yaml --context="${CTX_EXTERNAL_CLUSTER}"
    {{< /text >}}

1. 外部istiodが正常にデプロイされたのを確認する

    {{< text bash >}}
    $ kubectl get po -n external-istiod --context="${CTX_EXTERNAL_CLUSTER}"
    NAME                      READY   STATUS    RESTARTS   AGE
    istiod-779bd6fdcf-bd6rg   1/1     Running   0          70s
    {{< /text >}}

1. Istio `Gateway`, `VirtualService`, と `DestinationRule`設定を作成し、通信をingress gatewayから外部コントロールプレーンにルーティングします。

    {{< text syntax=bash snip_id=get_external_istiod_gateway_config >}}
    $ cat <<EOF > external-istiod-gw.yaml
    apiVersion: networking.istio.io/v1beta1
    kind: Gateway
    metadata:
      name: external-istiod-gw
      namespace: external-istiod
    spec:
      selector:
        istio: ingressgateway
      servers:
        - port:
            number: 15012
            protocol: https
            name: https-XDS
          tls:
            mode: SIMPLE
            credentialName: $SSL_SECRET_NAME
          hosts:
          - $EXTERNAL_ISTIOD_ADDR
        - port:
            number: 15017
            protocol: https
            name: https-WEBHOOK
          tls:
            mode: SIMPLE
            credentialName: $SSL_SECRET_NAME
          hosts:
          - $EXTERNAL_ISTIOD_ADDR
    ---
    apiVersion: networking.istio.io/v1beta1
    kind: VirtualService
    metadata:
       name: external-istiod-vs
       namespace: external-istiod
    spec:
        hosts:
        - $EXTERNAL_ISTIOD_ADDR
        gateways:
        - external-istiod-gw
        http:
        - match:
          - port: 15012
          route:
          - destination:
              host: istiod.external-istiod.svc.cluster.local
              port:
                number: 15012
        - match:
          - port: 15017
          route:
          - destination:
              host: istiod.external-istiod.svc.cluster.local
              port:
                number: 443
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: external-istiod-dr
      namespace: external-istiod
    spec:
      host: istiod.external-istiod.svc.cluster.local
      trafficPolicy:
        portLevelSettings:
        - port:
            number: 15012
          tls:
            mode: SIMPLE
          connectionPool:
            http:
              h2UpgradePolicy: UPGRADE
        - port:
            number: 443
          tls:
            mode: SIMPLE
    EOF
    {{< /text >}}

    そして、外部クラスタにこの設定を適用する

    {{< text bash >}}
    $ kubectl apply -f external-istiod-gw.yaml --context="${CTX_EXTERNAL_CLUSTER}"
    {{< /text >}}

### Mesh admin steps

さて、Istioは起動して稼働しています。
メッシュ管理者は、あとメッシュ（必要ならゲートウェイも含め）にサービスをデプロイして設定すればよいだけです。

#### Deploy a sample application

1. リモートクラスタに`sample`ネームスペースを作成し、インジェクション用のラベルを付けます。

    {{< text bash >}}
    $ kubectl create --context="${CTX_REMOTE_CLUSTER}" namespace sample
    $ kubectl label --context="${CTX_REMOTE_CLUSTER}" namespace sample istio-injection=enabled
    {{< /text >}}

1. `helloworld` (`v1`) と `sleep` のサンプルをデプロイする

    {{< text bash >}}
    $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l service=helloworld -n sample --context="${CTX_REMOTE_CLUSTER}"
    $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l version=v1 -n sample --context="${CTX_REMOTE_CLUSTER}"
    $ kubectl apply -f @samples/sleep/sleep.yaml@ -n sample --context="${CTX_REMOTE_CLUSTER}"
    {{< /text >}}

1. `helloworld` と `sleep`のポッドが、サイドカーと共に稼働になるまで数秒待ちます。

    {{< text bash >}}
    $ kubectl get pod -n sample --context="${CTX_REMOTE_CLUSTER}"
    NAME                             READY   STATUS    RESTARTS   AGE
    helloworld-v1-776f57d5f6-s7zfc   2/2     Running   0          10s
    sleep-64d7d56698-wqjnm           2/2     Running   0          9s
    {{< /text >}}

1. `sleep`ポッドから`helloworld`サービスにリクエストを送ります。

    {{< text bash >}}
    $ kubectl exec --context="${CTX_REMOTE_CLUSTER}" -n sample -c sleep \
        "$(kubectl get pod --context="${CTX_REMOTE_CLUSTER}" -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')" \
        -- curl -sS helloworld.sample:5000/hello
    Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
    {{< /text >}}

#### Enable gateways

1. リモートクラスタのingress gatewayを有効にする

    {{< text bash >}}
    $ cat <<EOF > istio-ingressgateway.yaml
    apiVersion: operator.istio.io/v1alpha1
    kind: IstioOperator
    spec:
      profile: empty
      components:
        ingressGateways:
        - namespace: external-istiod
          name: istio-ingressgateway
          enabled: true
      values:
        gateways:
          istio-ingressgateway:
            injectionTemplate: gateway
    EOF
    $ istioctl install -f istio-ingressgateway.yaml --context="${CTX_REMOTE_CLUSTER}"
    {{< /text >}}

1. リモートクラスタのegress gatewayまたは、その他のゲートウェイを有効にする（オプション）

    {{< text bash >}}
    $ cat <<EOF > istio-egressgateway.yaml
    apiVersion: operator.istio.io/v1alpha1
    kind: IstioOperator
    spec:
      profile: empty
      components:
        egressGateways:
        - namespace: external-istiod
          name: istio-egressgateway
          enabled: true
      values:
        gateways:
          istio-egressgateway:
            injectionTemplate: gateway
    EOF
    $ istioctl install -f istio-egressgateway.yaml --context="${CTX_REMOTE_CLUSTER}"
    {{< /text >}}

1. Istio ingress gatewayが稼働しているのを確認する

    {{< text bash >}}
    $ kubectl get pod -l app=istio-ingressgateway -n external-istiod --context="${CTX_REMOTE_CLUSTER}"
    NAME                                    READY   STATUS    RESTARTS   AGE
    istio-ingressgateway-7bcd5c6bbd-kmtl4   1/1     Running   0          8m4s
    {{< /text >}}

1. `helloworld`アプリケーションをingress gatewayに公開する

    {{< text bash >}}
    $ kubectl apply -f @samples/helloworld/helloworld-gateway.yaml@ -n sample --context="${CTX_REMOTE_CLUSTER}"
    {{< /text >}}

1. `GATEWAY_URL`環境変数を設定する（詳細は [determining the ingress IP and ports](/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports) 参照）

    {{< text bash >}}
    $ export INGRESS_HOST=$(kubectl -n external-istiod --context="${CTX_REMOTE_CLUSTER}" get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    $ export INGRESS_PORT=$(kubectl -n external-istiod --context="${CTX_REMOTE_CLUSTER}" get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    $ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
    {{< /text >}}

1. `helloworld`アプリケーションがingress gateway経由でアクセスできるのを確認する

    {{< text bash >}}
    $ curl -s "http://${GATEWAY_URL}/hello"
    Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
    {{< /text >}}

## Adding clusters to the mesh (optional) {#adding-clusters}

{{< boilerplate experimental >}}

このセクションでは、既存の外部コントロールプレーンメッシュに、別のリモートクラスタを追加してマルチクラスタに拡張する方法を説明します。
これにより簡単に複数のサービスを分散して、[location-aware routing and fail over](/docs/tasks/traffic-management/locality-load-balancing/) を使って、アプリケーションの高可用性をサポートすることができます。

{{< image width="75%"
    link="external-multicluster.svg"
    caption="External control plane with multiple remote clusters"
    >}}

最初のリモートクラスタと違って、同じ外部コントロールプレーンに追加される2つ目以降のクラスタはメッシュ設定を提供しません。
代わりに、Istio複数クラスタ設定[primary-remote](/docs/setup/install/multicluster/primary-remote_multi-network/)のリモートクラスタのように
エンドポイント設定の情報源とだけなります。

先に進めるために、メッシュの2つ目のリモートクラスタとして別のKubernetesクラスタが必要です。
次の環境変数に、コンテキスト名とクラスタ名を設定します。

{{< text syntax=bash snip_id=none >}}
$ export CTX_SECOND_CLUSTER=<your second remote cluster context>
$ export SECOND_CLUSTER_NAME=<your second remote cluster name>
{{< /text >}}

### Register the new cluster

1. コントロールプレーンが2つ目のリモートクラスタのエンドポイントにアクセスできるクレデンシャルが入ったシークレットを作成し、インストールする

    {{< text bash >}}
    $ istioctl x create-remote-secret \
      --context="${CTX_SECOND_CLUSTER}" \
      --name="${SECOND_CLUSTER_NAME}" \
      --type=remote \
      --namespace=external-istiod | \
      kubectl apply -f - --context="${CTX_REMOTE_CLUSTER}" #TODO use --context="{CTX_EXTERNAL_CLUSTER}" when #31946 is fixed.
    {{< /text >}}

    設定クラスタとしても動作するメッシュの1つ目のリモートクラスタと違い、`--type`引数に、`config`ではなく今回は`remote`を設定することに注意してください。

    {{< tip >}}
    外部istiodが両方のクラスタの追加を監視しているため
    新しいシークレットはリモート(設定)クラスタまたは外部クラスタいずれかに適用できることに注意してください。
    {{< /tip >}}

1. ローカルのインジェクターの代わりに、外部コントロールプレーンのインジェクターを使うインジェクションwebhookをインストールする
  リモートistioインストール設定を作成する

    {{< text syntax=bash snip_id=get_second_config_cluster_iop >}}
    $ cat <<EOF > second-config-cluster.yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    metadata:
      namespace: external-istiod
    spec:
      profile: external
      values:
        global:
          istioNamespace: external-istiod
        istiodRemote:
          injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/inject/:ENV:cluster=${SECOND_CLUSTER_NAME}:ENV:net=network2
    EOF
    {{< /text >}}

    そして、リモートクラスタに設定をインストールする

    {{< text bash >}}
    $ istioctl manifest generate -f second-config-cluster.yaml | kubectl apply --context="${CTX_SECOND_CLUSTER}" -f -
    {{< /text >}}

1. リモートクラスタのwebhook設定がインストールされているのを確認する

    {{< text bash >}}
    $ kubectl get mutatingwebhookconfiguration --context="${CTX_SECOND_CLUSTER}"
    NAME                                     WEBHOOKS   AGE
    istio-sidecar-injector-external-istiod   4          4m13s
    {{< /text >}}

### Setup east-west gateways

1. 両方のリモートクラスタにeast-westゲートウェイをデプロイする

    {{< text bash >}}
    $ @samples/multicluster/gen-eastwest-gateway.sh@ \
        --mesh mesh1 --cluster "${REMOTE_CLUSTER_NAME}" --network network1 > eastwest-gateway-1.yaml
    $ istioctl manifest generate -f eastwest-gateway-1.yaml \
        --set values.gateways.istio-ingressgateway.injectionTemplate=gateway \
        --set values.global.istioNamespace=external-istiod | \
        kubectl apply --context="${CTX_REMOTE_CLUSTER}" -f -
    {{< /text >}}

    {{< text bash >}}
    $ @samples/multicluster/gen-eastwest-gateway.sh@ \
        --mesh mesh1 --cluster "${SECOND_CLUSTER_NAME}" --network network2 > eastwest-gateway-2.yaml
    $ istioctl manifest generate -f eastwest-gateway-2.yaml \
        --set values.gateways.istio-ingressgateway.injectionTemplate=gateway \
        --set values.global.istioNamespace=external-istiod | \
        kubectl apply --context="${CTX_SECOND_CLUSTER}" -f -
    {{< /text >}}

1. east-westゲートウェイに外部IPが割り当てされるのを待つ

    {{< text bash >}}
    $ kubectl --context="${CTX_REMOTE_CLUSTER}" get svc istio-eastwestgateway -n external-istiod
    NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
    istio-eastwestgateway   LoadBalancer   10.0.12.121   34.122.91.98   ...       51s
    {{< /text >}}

    {{< text bash >}}
    $ kubectl --context="${CTX_SECOND_CLUSTER}" get svc istio-eastwestgateway -n external-istiod
    NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
    istio-eastwestgateway   LoadBalancer   10.0.12.121   34.122.91.99   ...       51s
    {{< /text >}}

1. サービスをeast-westゲートウェイに公開する

    {{< text bash >}}
    $ kubectl --context="${CTX_REMOTE_CLUSTER}" apply -n external-istiod -f \
        @samples/multicluster/expose-services.yaml@
    {{< /text >}}

    {{< text bash >}}
    $ kubectl --context="${CTX_SECOND_CLUSTER}" apply -n external-istiod -f \
        @samples/multicluster/expose-services.yaml@
    {{< /text >}}

### Validate the installation

1. リモートクラスタの`sample`ネームスペースを作成し、インジェクション用のラベルを付与する

    {{< text bash >}}
    $ kubectl create --context="${CTX_SECOND_CLUSTER}" namespace sample
    $ kubectl label --context="${CTX_SECOND_CLUSTER}" namespace sample istio-injection=enabled
    {{< /text >}}

1. `helloworld` (`v2`) と `sleep`サンプルをデプロイする

    {{< text bash >}}
    $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l service=helloworld -n sample --context="${CTX_SECOND_CLUSTER}"
    $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l version=v2 -n sample --context="${CTX_SECOND_CLUSTER}"
    $ kubectl apply -f @samples/sleep/sleep.yaml@ -n sample --context="${CTX_SECOND_CLUSTER}"
    {{< /text >}}

1. `helloworld` と `sleep`ポッドがサイドカー付きで稼働するのを数秒待つ

    {{< text bash >}}
    $ kubectl get pod -n sample --context="${CTX_SECOND_CLUSTER}"
    NAME                            READY   STATUS    RESTARTS   AGE
    helloworld-v2-54df5f84b-9hxgw   2/2     Running   0          10s
    sleep-557747455f-wtdbr          2/2     Running   0          9s
    {{< /text >}}

1. `sleep` ポッドから `helloworld`サービスにリクエストを送る

    {{< text bash >}}
    $ kubectl exec --context="${CTX_SECOND_CLUSTER}" -n sample -c sleep \
        "$(kubectl get pod --context="${CTX_SECOND_CLUSTER}" -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')" \
        -- curl -sS helloworld.sample:5000/hello
    Hello version: v2, instance: helloworld-v2-54df5f84b-9hxgw
    {{< /text >}}

1. `helloworld` アプリケーションにingressゲートウェイ経由で数回アクセスすると、`v1`と`v2`両方のバージョンから応答があることを確認する

    {{< text bash >}}
    $ for i in {1..10}; do curl -s "http://${GATEWAY_URL}/hello"; done
    Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
    Hello version: v2, instance: helloworld-v2-54df5f84b-9hxgw
    Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
    Hello version: v2, instance: helloworld-v2-54df5f84b-9hxgw
    ...
    {{< /text >}}
