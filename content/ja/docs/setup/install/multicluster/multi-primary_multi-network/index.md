---
title: Install Multi-Primary on different networks
description: Install an Istio mesh across multiple primary clusters on different networks.
weight: 30
keywords: [kubernetes,multicluster]
test: yes
owner: istio/wg-environments-maintainers
---
このガイドでは、Istioコントロールプレーンを `cluster1` と`cluster2` 両方にインストールして、
それぞれを{{< gloss >}}primary cluster{{< /gloss >}}にします。
`cluster1` クラスタは `network1`ネットワーク上に
`cluster2` クラスタは `network2`ネットワーク上に置きます。
つまり両方のクラスタ内のポッド間は直接接続できないことを意味します。

進める前に、[before you begin](/ja/docs/setup/install/multicluster/before-you-begin)のステップを確実に完了してください。

{{< boilerplate multi-cluster-with-metallb >}}

この設定では、`cluster1` と `cluster2`両方が、エンドポイントのためにそれぞれのクラスタを監視します

クラスタ境界を跨ぐサービスワークロードは、[east-west](https://en.wikipedia.org/wiki/East-west_traffic)通信用のゲートウェイ経由で間接的に通信します。
それぞれのクラスタのゲートウェイは、他のクラスタから到達可能でなければなりません。

{{< image width="75%"
    link="arch.svg"
    caption="Multiple primary clusters on separate networks"
    >}}

## Set the default network for `cluster1`

istio-system ネームスペースが作成済みの場合、クラスタネットワークのラベルをセットする必要があります。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
{{< /text >}}

## Configure `cluster1` as a primary

`cluster1` 用のIstio設定を作成します

{{< text bash >}}
$ cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
{{< /text >}}

Istio設定を`cluster1`に適用します

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
{{< /text >}}

## Install the east-west gateway in `cluster1`

[east-west](https://en.wikipedia.org/wiki/East-west_traffic)通信用に、`cluster1` にゲートウェイをインストールします。標準ではこのゲートウェイはインターネットに公開されます。商用システムなら追加でアクセス制限（例えばファイヤーウォールルールで）をおこない外部からの攻撃を防ぐ必要があるかもしれません。ご利用のクラウドベンダーでどんなオプションが使えるかチェックしてください。

{{< text bash >}}
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
    --mesh mesh1 --cluster cluster1 --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -
{{< /text >}}

{{< warning >}}
コントロールプレーンがリビジョン付きでインストールされている場合は、`gen-eastwest-gateway.sh` コマンドに  `--revision rev` フラグを追加してください。
{{< /warning >}}

east-westゲートウェイに external IP addressが割り当てられるまで待ちます。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
istio-eastwestgateway   LoadBalancer   10.80.6.124   34.75.71.237   ...       51s
{{< /text >}}

## Expose services in `cluster1`

それぞれのクラスタは別のネットワークにいるので、両方のクラスタで全てのサービス(*.local) を east-westゲートウェイ上に公開する必要があります。このゲートウェイはインターネットに公開されているので、ゲートウェイ配下のサービスは、信頼できるmTLS証明書とワークロードIDによってのみアクセス可能です。ちょうど同じネットワーク上にいる時と同様にです。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
    @samples/multicluster/expose-services.yaml@
{{< /text >}}

## Set the default network for `cluster2`

istio-system ネームスペースが作成済みの場合、クラスタネットワークのラベルをセットする必要があります。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2
{{< /text >}}

## Configure cluster2 as a primary

`cluster2` 用のIstio設定を作成します

{{< text bash >}}
$ cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
EOF
{{< /text >}}

Istio設定を`cluster2`に適用します

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
{{< /text >}}

## Install the east-west gateway in `cluster2`

上で`cluster1` にやったように、`cluster2` のeast-west通信用にゲートウェイをインストールします。

{{< text bash >}}
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
    --mesh mesh1 --cluster cluster2 --network network2 | \
    istioctl --context="${CTX_CLUSTER2}" install -y -f -
{{< /text >}}

east-westゲートウェイに external IP addressが割り当てられるまで待ちます。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
istio-eastwestgateway   LoadBalancer   10.0.12.121   34.122.91.98   ...       51s
{{< /text >}}

## Expose services in `cluster2`

上で`cluster1` にやったように、サービスをeast-westゲートウェイに公開します。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
    @samples/multicluster/expose-services.yaml@
{{< /text >}}

## Enable Endpoint Discovery

`cluster1`の API serverに接続する用に、`cluster2`　にリモートシークレットをインストールします

{{< text bash >}}
$ istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"
{{< /text >}}

`cluster2`の API serverに接続する用に、`cluster1`　にリモートシークレットをインストールします

{{< text bash >}}
$ istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"
{{< /text >}}

**おめでとう!** 異なるネットワークでマルチプライマリクラスタのIstioメッシュのインストールに成功しました。

## Next Steps

You can now [verify the installation](/ja/docs/setup/install/multicluster/verify).
