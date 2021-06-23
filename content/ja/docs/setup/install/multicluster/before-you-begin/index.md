---
title: Before you begin
description: Initial steps before installing Istio on multiple clusters.
weight: 1
keywords: [kubernetes,multicluster]
test: n/a
owner: istio/wg-environments-maintainers
---

マルチクラスタのインストールを始める前に、このガイドを通じて適用される基本的なコンセプトを記載した[デプロイメントモデルガイド](/ja/docs/ops/deployment/deployment-models)を読んでください。

さらに、次の要件を読んで、最初のステップを実行してください。

## Requirements

### Cluster

このガイドでは、2つのKubernetesクラスタが必要です。
Kubernetesクラスタは次のバージョンをサポートしています。 {{< supported_kubernetes_versions >}}

### API Server Access

各クラスタのAPI Serverはメッシュ内の他のクラスタからアクセスできなければなりません。
多くのクラウドプロバイダは、API Serverをネットワークロードバランサー(NLB)経由で公開アクセスできるようにしています。
API Serverが直接アクセス出来ない場合、アクセスを実現するためにインストール手順を修正する必要があります。例えば、マルチネットワークとプライマリーリモート設定で使っている[イーストウェスト](https://en.wikipedia.org/wiki/East-west_traffic)ゲートウェイは、API Serverへのアクセスを実現するためにも使えます。

## Environment Variables

このガイドは2つのクラスタ（ `cluster1` と `cluster2` ） を使います。手順をシンプルにするため、次の環境変数を全体を通じて使います

Variable | Description
-------- | -----------
`CTX_CLUSTER1` | The context name in the default [Kubernetes configuration file](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) used for accessing the `cluster1` cluster.
`CTX_CLUSTER2` | The context name in the default [Kubernetes configuration file](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) used for accessing the `cluster2` cluster.

Set the two variables before proceeding:

{{< text syntax=bash snip_id=none >}}
$ export CTX_CLUSTER1=<your cluster1 context>
$ export CTX_CLUSTER2=<your cluster2 context>
{{< /text >}}

## Configure Trust

マルチクラスタのサービスメッシュデプロイには、メッシュ内のすべてのクラスタ間で信頼関係を確立する必要があります。
あなたのシステム要件に応じて、信頼関係を確立する方法は複数のオプションがあります。

[certificate management](/ja/docs/tasks/security/cert-management/) に全ての選択可能なオプションについて、詳細な説明と手順があります。
選択したオプション毎にIsitoのインストール手順が少し変わる可能性があります。

このガイドでは、各クラスタの中間証明書を生成するのに共通のルート証明書を使うことを仮定します。
[instructions](/ja/docs/tasks/security/cert-management/plugin-ca-cert/) にしたがって、CA 証明書のシークレットを生成して、`cluster1` と `cluster2` の両方にプッシュしてください。

{{< tip >}}
もし、現在シングルクラスタで自己CA証明書を使っている場合 ([Getting Started](/ja/docs/setup/getting-started/)に記載されているように)は、CAを[certificate management](/ja/docs/tasks/security/cert-management/)記載のいずれかの方法を使って変更する必要があります。
CAの変更には、一般的にIstioの再インストールが必要となります。次のインストール手順はあなたがどのCAを選択したかで変更しなくてはいけないかもしれません。
{{< /tip >}}

## Next steps

マルチクラスタにIstioをインストール準備ができました。一部のステップは、あなたのネットワークとコントロールプレーンのトポロジーに依存します。

あなたのニーズに一番合致したインストール方法を選択してください。

- [Install Multi-Primary](/ja/docs/setup/install/multicluster/multi-primary)

- [Install Primary-Remote](/ja/docs/setup/install/multicluster/primary-remote)

- [Install Multi-Primary on Different Networks](/ja/docs/setup/install/multicluster/multi-primary_multi-network)

- [Install Primary-Remote on Different Networks](/ja/docs/setup/install/multicluster/primary-remote_multi-network)

{{< tip >}}
２クラスタ以上のメッシュを拡げるには、これらのオプションの1つ以上が必要になるかもしれません。例えば、リージョン毎にプライマリークラスターを置いて（例えばマルチプライマリー）、ぞれぞれのゾーンでリモートクラスタを置いて、各リージョンのプライマリコントロールプレーンを使う（例えば、プライマリーリモート）

詳細は[deployment models](/ja/docs/ops/deployment/deployment-models)を参照
{{< /tip >}}
