# Google Cloud native Langfuse

## Overview

Langfuseが用意しているv3用のHelm chartはGoogle Cloudネイティブではないため、様々なリソースをGoogle Cloudのものに変更したものがこちらです。

## Prerequisites

- [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine?hl=en)
- [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity?hl=en)
- [External Secrets Operator](https://external-secrets.io/)
- [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy?hl=en)
- [Google Cloud Storage](https://cloud.google.com/storage?hl=en)

## Create Node Pool

このHelmで起動するPodは`langfuse-pool`という名前のノードプールにPodをスケジュールします。
事前にクラスタにこの名前でノードプールを作成してください。

```
gcloud container node-pools create langfuse-pool \
  --cluster=<YOUR_CLUSTER_NAME> \
  --location=asia-northeast1 \
  --machine-type=e2-standard-2 --spot \
  --num-nodes=1 \
  --enable-private-nodes \
  --workload-metadata=GKE_METADATA \
  --disk-size=20 \
  --node-version=latest \
  --enable-surge-upgrade \
  --enable-autoscaling \
  --total-max-nodes=6 \
  --total-min-nodes=1 \
  --service-account=<YOUR_NODE_SERVICE_ACCOUNT>
```

## Additional Google Cloud components

- Global external Application Load Balancer (Gateway)
- Cloud SQL for PostgreSQL
- Secret Manager
- Certificate Manager
- Google Cloud Storage

### Global external Application Load Balancer (Gateway)

`langfuse-web` へのトラフィックを受け取るLoadBalancerを作成します。
クラスタ内の別のアプリケーションへLBを流用できるように、`allowedRoutes` のnamespacesは`from: All` を設定しています。

これの作成は任意であり、クラスタに既にGatewayが作成されている場合はそちらを利用するようにHttpRouteを修正してください。

### Cloud SQL for PostgreSQL

langfuseのデータベースにはCloud SQL for PostgreSQLを使用するので、事前に用意してください。
Podにcloud sql proxyコンテナを追加しています。

Kubernetes Service Account(KSA)がCloud SQLへ接続できるように、以下のコマンドを例にKSAにIAMロールを付与してください。

```
gcloud projects add-iam-policy-binding projects/<YOUR_PROJECT_ID> \
  --role=roles/cloudsql.client \
  --member=principal://iam.googleapis.com/projects/<YOUR_PROJECT_NUMBER>/locations/global/workloadIdentityPools/<YOUR_PROJECT_ID>.svc.id.goog/subject/ns/langfuse/sa/langfuse \
  --condition=None
```

### Secret Manager

各機密情報はSecret Managerに保存しています。
Secret Managerに保存した値をKubernetesクラスタで利用するために、External Secrets Operatorを使用しています。
External Secrets Operatorも事前にクラスタにインストールしてください。

### Google Cloud Storage

`minio`の代わりにGoogle Cloud Storageを使用します。
バケットを作成し、[こちら](https://langfuse.com/self-hosting/infrastructure/blobstorage#google-cloud-storage)を参考にAccess KeyとSecret Keyを取得してSecret Managerに保存してください。
