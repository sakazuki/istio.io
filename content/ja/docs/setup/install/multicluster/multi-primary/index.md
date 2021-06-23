---
title: Install Multi-Primary
description: Install an Istio mesh across multiple primary clusters.
weight: 10
keywords: [kubernetes,multicluster]
test: yes
owner: istio/wg-environments-maintainers
---
このガイドでは、Istioコントロールプレーンを `cluster1` と`cluster2` 両方にインストールして、
それぞれを{{< gloss >}}primary cluster{{< /gloss >}}にします。
両方のクラスタを`network1`ネットワーク上に並べます。つまり両方のクラスタ内のポッド間は直接接続できます。

進める前に、[before you begin](/ja/docs/setup/install/multicluster/before-you-begin)のステップを確実に完了してください。

この設定では、それぞれのコントロールプレーンが、エンドポイントのために両方のクラスタのAPI Serverを監視します。

サービスワークロードは、クラスタを跨いで直接ポッド間で通信します。

{{< image width="75%"
    link="arch.svg"
    caption="Multiple primary clusters on the same network"
    >}}

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

## Configure `cluster2` as a primary

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
      network: network1
EOF
{{< /text >}}


Istio設定を`cluster2`に適用します

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
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

**おめでとう!** マルチプライマリクラスタのIstioメッシュのインストールに成功しました。

## Next Steps

You can now [verify the installation](/ja/docs/setup/install/multicluster/verify).
