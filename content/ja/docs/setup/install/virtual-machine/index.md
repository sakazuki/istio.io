---
title: Virtual Machine Installation
description: Deploy Istio and connect a workload running within a virtual machine to it.
weight: 60
keywords:
- kubernetes
- virtual-machine
- gateways
- vms
owner: istio/wg-environments-maintainers
test: yes
---

このガイドでは、Istioをデプロイし仮想マシンをそれに接続します。

## Prerequisites

1. [Download the Istio release](/docs/setup/getting-started/#download)
1. Perform any necessary [platform-specific setup](/docs/setup/platform-setup/)
1. Check the requirements [for Pods and Services](/docs/ops/deployment/requirements/)
1. 仮想マシンは、接続するメッシュの入力ゲートウェイへのIP接続が必要です。パフォーマンスの向上が必要な場合は、オプションでL3ネットワークを介してメッシュ内のすべてのポッドに接続する必要があります。
1. Istioの仮想マシン統合のハイレベルなアーキテクチャについて理解したい場合は、[Virtual Machine Architecture](/docs/ops/deployment/vm-architecture/) を学習してください。

## Prepare the guide environment

1. 仮想マシンを作成する
1. クラスタをセットアップするのに使う（作業用）マシンに環境変数（`VM_APP`, `WORK_DIR` , `VM_NAMESPACE`,　`SERVICE_ACCOUNT`）をセットする
    (e.g., `WORK_DIR="${HOME}/vmintegration"`):

    {{< tabset category-name="network-mode" >}}

    {{< tab name="Single-Network" category-value="single" >}}

    {{< text bash >}}
    $ VM_APP="<the name of the application this VM will run>"
    $ VM_NAMESPACE="<the name of your service namespace>"
    $ WORK_DIR="<a certificate working directory>"
    $ SERVICE_ACCOUNT="<name of the Kubernetes service account you want to use for your VM>"
    $ CLUSTER_NETWORK=""
    $ VM_NETWORK=""
    $ CLUSTER="Kubernetes"
    {{< /text >}}

    {{< /tab >}}

    {{< tab name="Multi-Network" category-value="multiple" >}}

    {{< text bash >}}
    $ VM_APP="<the name of the application this VM will run>"
    $ VM_NAMESPACE="<the name of your service namespace>"
    $ WORK_DIR="<a certificate working directory>"
    $ SERVICE_ACCOUNT="<name of the Kubernetes service account you want to use for your VM>"
    $ # Customize values for multi-cluster/multi-network as needed
    $ CLUSTER_NETWORK="kube-network"
    $ VM_NETWORK="vm-network"
    $ CLUSTER="cluster1"
    {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

1. クラスタをセットアップするのに使う（作業用）マシンに作業ディレクトリを作成する

    {{< text syntax=bash snip_id=setup_wd >}}
    $ mkdir -p "${WORK_DIR}"
    {{< /text >}}

## Install the Istio control plane

Istioコントロールプレーンが既にある場合、このインストール工程はスキップできます。ただし、仮想マシンがアクセスするためにコントロールプレーンを公開する必要はあるでしょう。

Istioをインストールし、仮想マシンがアクセスできるようにするため、コントロールプレーンを公開します。

1. インストールのため、`IstioOperator`設定を作成する。

    {{< text syntax="bash yaml" snip_id=setup_iop >}}
    $ cat <<EOF > ./vm-cluster.yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    metadata:
      name: istio
    spec:
      values:
        global:
          meshID: mesh1
          multiCluster:
            clusterName: "${CLUSTER}"
          network: "${CLUSTER_NETWORK}"
    EOF
    {{< /text >}}

1. Istioをインストールする

    {{< tabset category-name="registration-mode" >}}

    {{< tab name="Default" category-value="default" >}}

    {{< text bash >}}
    $ istioctl install -f vm-cluster.yaml
    {{< /text >}}

    {{< /tab >}}

    {{< tab name="Automated WorkloadEntry Creation" category-value="autoreg" >}}

    {{< boilerplate experimental >}}

    {{< text syntax=bash snip_id=install_istio >}}
    $ istioctl install -f vm-cluster.yaml --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION=true --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_HEALTHCHECKS=true
    {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

1. east-westゲートウェイをデプロイする

    {{< warning >}}
    コントロールプレーンがリビジョン付きでインストールされている場合は、`gen-eastwest-gateway.sh` コマンドに  `--revision rev` フラグを追加してください。
    {{< /warning >}}

    {{< tabset category-name="network-mode" >}}

    {{< tab name="Single-Network" category-value="single" >}}

    {{< text syntax=bash snip_id=install_eastwest >}}
    $ @samples/multicluster/gen-eastwest-gateway.sh@ --single-cluster | istioctl install -y -f -
    {{< /text >}}

    {{< /tab >}}

    {{< tab name="Multi-Network" category-value="multiple" >}}

    {{< text bash >}}
    $ @samples/multicluster/gen-eastwest-gateway.sh@ \
        --mesh mesh1 --cluster "${CLUSTER}" --network "${CLUSTER_NETWORK}" | \
        istioctl install -y -f -
    {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

1. クラスタ内のサービスを、east-westゲートウェイに経由で公開する

    {{< tabset category-name="network-mode" >}}

    {{< tab name="Single-Network" category-value="single" >}}

    コントロールプレーンを公開する

    {{< text syntax=bash snip_id=expose_istio >}}
    $ kubectl apply -n istio-system -f @samples/multicluster/expose-istiod.yaml@
    {{< /text >}}

    {{< /tab >}}

    {{< tab name="Multi-Network" category-value="multiple" >}}

    コントロールプレーンを公開する

    {{< text bash >}}
    $ kubectl apply -n istio-system -f @samples/multicluster/expose-istiod.yaml@
    {{< /text >}}

    サービスを公開する

    {{< text bash >}}
    $ kubectl apply -n istio-system -f @samples/multicluster/expose-services.yaml@
    {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

## Configure the VM namespace

1. 仮想マシンをホストするネームスペースを作成する

    {{< text syntax=bash snip_id=install_namespace >}}
    $ kubectl create namespace "${VM_NAMESPACE}"
    {{< /text >}}

1. 仮想マシン用のサービスアカウントを作成する

    {{< text syntax=bash snip_id=install_sa >}}
    $ kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
    {{< /text >}}

## Create files to transfer to the virtual machine

{{< tabset category-name="registration-mode" >}}

{{< tab name="Default" category-value="default" >}}

最初に、VM用の`WorkloadGroup`テンプレートを作成する

{{< text bash >}}
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
{{< /text >}}

{{< /tab >}}

{{< tab name="Automated WorkloadEntry Creation" category-value="autoreg" >}}

最初に、VM用の`WorkloadGroup`テンプレートを作成する

{{< boilerplate experimental >}}

{{< text syntax=bash snip_id=create_wg >}}
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
{{< /text >}}

それから、`WorkloadEntry`の自動作成を許可するために、クラスタに`WorkloadGroup`を適用する

{{< text syntax=bash snip_id=apply_wg >}}
$ kubectl --namespace "${VM_NAMESPACE}" apply -f workloadgroup.yaml
{{< /text >}}

`WorkloadEntry`の自動作成機能を使って、アプリケーションのヘルスチェックも使えるようになる。これらのヘルスチェックは[Kubernetes Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)のAPIと振る舞いを共有します

For example, to configure a probe on the `/ready` endpoint of your application:
例えば、アプリケーションの`/ready` エンドポイントのprobe設定は、

{{< text bash >}}
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${NETWORK}"
  probe:
    periodSeconds: 5
    initialDelaySeconds: 1
    httpGet:
      port: 8080
      path: /ready
EOF
{{< /text >}}

この設定で、自動作成された`WorkloadEntry` は、probeが成功するまで、"Ready"とマークされません。

{{< /tab >}}

{{< /tabset >}}

{{< warning >}}
`istioctl x workload entry` の個所で、`istio-token`作成をする前に、クラスタでサードパーティートークンが有効になっているか確認すべきです。
確認は　[here](/docs/ops/best-practices/security/#configure-third-party-service-account-tokens)　に記載の方法でやります。
サードパーティートークンが有効になってない場合、Istioインストールコマンドに、`--set values.global.jwtPolicy=first-party-jwt` オプションを追加すべきです。
{{< /warning >}}

続いて、`istioctl x workload entry` コマンドを使って、次のものを作成する

* `cluster.env`: Contains metadata that identifies what namespace, service account, network CIDR and (optionally) what inbound ports to capture.
* `istio-token`: A Kubernetes token used to get certs from the CA.
* `mesh.yaml`: Provides `ProxyConfig` to configure `discoveryAddress`, health-checking probes, and some authentication options.
* `root-cert.pem`: The root certificate used to authenticate.
* `hosts`: An addendum to `/etc/hosts` that the proxy will use to reach istiod for xDS.*

{{< idea >}}
洗練されたオプションでは、仮想マシンのDNSに外部DNSサーバーを参照させる設定ができます。
このオプションはこのガイドの範囲外です。
{{< /idea >}}

{{< tabset category-name="registration-mode" >}}

{{< tab name="Default" category-value="default" >}}

{{< text bash >}}
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
{{< /text >}}

{{< /tab >}}

{{< tab name="Automated WorkloadEntry Creation" category-value="autoreg" >}}

{{< boilerplate experimental >}}

{{< text syntax=bash snip_id=configure_wg >}}
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --autoregister
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## Configure the virtual machine

仮想マシンで次のコマンドを実行して、Istioメッシュに追加します

1. `"${WORK_DIR}"`から仮想マシンにファイルを安全に転送します。安全な転送方法をどのように選ぶかは、あなたの情報セキュリティポリシーを考慮しておこなわれるべきです。このガイドでは便利のため、必須ファイルを全て、仮想マシンの`"${HOME}"`に転送します。

1. ルート証明書を`/etc/certs`にインストールする

    {{< text bash >}}
    $ sudo mkdir -p /etc/certs
    $ sudo cp "${HOME}"/root-cert.pem /etc/certs/root-cert.pem
    {{< /text >}}

1. トークンを`/var/run/secrets/tokens`にインストールする

    {{< text bash >}}
    $ sudo  mkdir -p /var/run/secrets/tokens
    $ sudo cp "${HOME}"/istio-token /var/run/secrets/tokens/istio-token
    {{< /text >}}

1. Istio仮想マシン統合ランタイムを含んだパッケージをインストールする

    {{< tabset category-name="vm-os" >}}

    {{< tab name="Debian" category-value="debian" >}}

    {{< text syntax=bash snip_id=none >}}
    $ curl -LO https://storage.googleapis.com/istio-release/releases/{{< istio_full_version >}}/deb/istio-sidecar.deb
    $ sudo dpkg -i istio-sidecar.deb
    {{< /text >}}

    {{< /tab >}}

    {{< tab name="CentOS" category-value="centos" >}}

    Note: CentOS 8 だけが現在サポートされています

    {{< text syntax=bash snip_id=none >}}
    $ curl -LO https://storage.googleapis.com/istio-release/releases/{{< istio_full_version >}}/rpm/istio-sidecar.rpm
    $ sudo rpm -i istio-sidecar.rpm
    {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

1. `cluster.env`を`/var/lib/istio/envoy/`ディレクトリにインストールする

    {{< text bash >}}
    $ sudo cp "${HOME}"/cluster.env /var/lib/istio/envoy/cluster.env
    {{< /text >}}

1. [Mesh Config](/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig) を `/etc/istio/config/mesh`にインストールする

    {{< text bash >}}
    $ sudo cp "${HOME}"/mesh.yaml /etc/istio/config/mesh
    {{< /text >}}

1. istiod のホスト情報を `/etc/hosts`に追加する

    {{< text bash >}}
    $ sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
    {{< /text >}}

1. `/etc/certs/` と `/var/lib/istio/envoy/`のファイル所有権を、Istio proxyに変更する

    {{< text bash >}}
    $ sudo mkdir -p /etc/istio/proxy
    $ sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
    {{< /text >}}

## Start Istio within the virtual machine

1. Isitoエージェントを開始する

    {{< text bash >}}
    $ sudo systemctl start istio
    {{< /text >}}

## Verify Istio Works Successfully

1. `/var/log/istio/istio.log`のログを確認する。次と同様な出力が確認できるべきです。

    {{< text bash >}}
    $ 2020-08-21T01:32:17.748413Z info sds resource:default pushed key/cert pair to proxy
    $ 2020-08-21T01:32:20.270073Z info sds resource:ROOTCA new connection
    $ 2020-08-21T01:32:20.270142Z info sds Skipping waiting for gateway secret
    $ 2020-08-21T01:32:20.270279Z info cache adding watcher for file ./etc/certs/root-cert.pem
    $ 2020-08-21T01:32:20.270347Z info cache GenerateSecret from file ROOTCA
    $ 2020-08-21T01:32:20.270494Z info sds resource:ROOTCA pushed root cert to proxy
    $ 2020-08-21T01:32:20.270734Z info sds resource:default new connection
    $ 2020-08-21T01:32:20.270763Z info sds Skipping waiting for gateway secret
    $ 2020-08-21T01:32:20.695478Z info cache GenerateSecret default
    $ 2020-08-21T01:32:20.695595Z info sds resource:default pushed key/cert pair to proxy
    {{< /text >}}

1. サービスをデプロイするためにネームスペースを作成する

    {{< text bash >}}
    $ kubectl create namespace sample
    $ kubectl label namespace sample istio-injection=enabled
    {{< /text >}}

1. `HelloWorld`サービスをデプロイする

    {{< text bash >}}
    $ kubectl apply -n sample -f @samples/helloworld/helloworld.yaml@
    {{< /text >}}

1. 仮想マシンからサービスにリクエストを送る

    {{< text bash >}}
    $ curl helloworld.sample.svc:5000/hello
    Hello version: v1, instance: helloworld-v1-578dd69f69-fxwwk
    {{< /text >}}

## Next Steps

For more information about virtual machines:

* [Debugging Virtual Machines](/docs/ops/diagnostic-tools/virtual-machines/) to troubleshoot issues with virtual machines.
* [Bookinfo with a Virtual Machine](/docs/examples/virtual-machines/) to set up an example deployment of virtual machines.

## Uninstall

Stop Istio on the virtual machine:

{{< text bash >}}
$ sudo systemctl stop istio
{{< /text >}}

Then, remove the Istio-sidecar package:

{{< tabset category-name="vm-os" >}}

{{< tab name="Debian" category-value="debian" >}}

{{< text bash >}}
$ sudo dpkg -r istio-sidecar
$ dpkg -s istio-sidecar
{{< /text >}}

{{< /tab >}}

{{< tab name="CentOS" category-value="centos" >}}

{{< text bash >}}
$ sudo rpm -e istio-sidecar
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

To uninstall Istio, run the following command:

{{< text bash >}}
$ kubectl delete -n istio-system -f @samples/multicluster/expose-istiod.yaml@
$ istioctl manifest generate | kubectl delete -f -
{{< /text >}}

The control plane namespace (e.g., `istio-system`) is not removed by default.
If no longer needed, use the following command to remove it:

{{< text bash >}}
$ kubectl delete namespace istio-system
{{< /text >}}
