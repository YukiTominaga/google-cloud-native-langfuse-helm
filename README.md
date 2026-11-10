# Google Cloud native Langfuse

## Overview

Langfuseが用意しているHelm chartはGoogle Cloudネイティブではないため、様々なリソースをGoogle Cloudのものに変更したものがこちらです。

Langfuse v4に対応しています(デフォルトのイメージタグは`4.77.0`)。
新規構築を前提としており、既存のv3環境をアップグレードする場合は[公式のアップグレードガイド](https://langfuse.com/self-hosting/upgrade/upgrade-guides/upgrade-v3-to-v4)に沿った移行作業が別途必要です。

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
  --machine-type=e2-standard-4 --spot \
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

### Spot VMを使用する場合の注意点

上記のコマンドはコストを抑えるためにSpot VM(`--spot`)を指定しています。
Spot VMはGoogle Cloud側の都合でいつでも回収され、回収の予告は約30秒前です。回収されたノード上のPodは別のノードで再作成されますが、その間は以下の影響があります。

- **ClickHouse / ClickHouse Keeper**: それぞれ3レプリカを別々のノードに配置しているため、ノード1台の回収であれば動作を継続します。2台以上が同時に回収されるとKeeperが過半数を失い、復旧するまでClickHouseへの書き込みが停止します。データはPersistent Diskに保存しているため、回収されても失われません。
- **Valkey**: 1レプリカで、データを永続化していません(`emptyDir`)。回収されるとキューに残っていた未処理のジョブが失われます。
- **langfuse-web / langfuse-worker**: デフォルトでは各1レプリカのため、再作成が完了するまでUI・APIへのアクセスやバックグラウンド処理が停止します。

これらの影響を許容できない場合は、`--spot`を外して通常のVMでノードプールを作成してください。

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
Langfuse v4はPostgreSQL 15以上が必要です(16推奨)。
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

以下のシークレットをSecret Managerに作成してください。`values.yaml`の`externalSecret.data`で、各シークレットをPodの環境変数に対応付けています。

| Secret Managerのシークレット名 | 環境変数 | 内容 |
| --- | --- | --- |
| `LANGFUSE_DATABASE_PROXY_URL` | `DATABASE_URL` | PostgreSQLの接続文字列。cloud sql proxy経由のため、ホストは`localhost`を指定します(例: `postgresql://<USER>:<PASSWORD>@localhost:5432/langfuse`) |
| `LANGFUSE_NEXTAUTH_SECRET` | `NEXTAUTH_SECRET` | セッションCookieの検証に使うシークレット(生成例: `openssl rand -base64 32`) |
| `LANGFUSE_SALT` | `SALT` | APIキーのハッシュ化に使うソルト(生成例: `openssl rand -base64 32`) |
| `LANGFUSE_ENCRYPTION_KEY` | `ENCRYPTION_KEY` | 機密データの暗号化キー。256ビット(16進数64文字)である必要があります(生成例: `openssl rand -hex 32`) |
| `LANGFUSE_AUTH_GOOGLE_CLIENT_ID` | `AUTH_GOOGLE_CLIENT_ID` | GoogleログインのOAuthクライアントID |
| `LANGFUSE_AUTH_GOOGLE_CLIENT_SECRET` | `AUTH_GOOGLE_CLIENT_SECRET` | GoogleログインのOAuthクライアントシークレット |

### Google Cloud Storage

`minio`の代わりにGoogle Cloud Storageを使用します。
[ネイティブ統合](https://langfuse.com/self-hosting/deployment/infrastructure/blobstorage#google-cloud-storage)で接続するため、HMACキー(Access Key / Secret Key)は不要です。認証にはWorkload Identity(KSAの認証情報)を使用します。

バケットを作成し、`values.yaml`の`LANGFUSE_S3_EVENT_UPLOAD_BUCKET`にバケット名を設定してください。
そのうえで、以下のコマンドを例にKSAへバケットのIAMロールを付与してください。

```
gcloud storage buckets add-iam-policy-binding gs://<YOUR_LANGFUSE_BUCKET_NAME> \
  --role=roles/storage.objectUser \
  --member=principal://iam.googleapis.com/projects/<YOUR_PROJECT_NUMBER>/locations/global/workloadIdentityPools/<YOUR_PROJECT_ID>.svc.id.goog/subject/ns/langfuse/sa/langfuse
```
