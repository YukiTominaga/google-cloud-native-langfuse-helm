# Google Cloud native Langfuse

## Overview

Langfuseが用意しているv3用のHelm chartはGoogle Cloudネイティブではないため、様々なリソースをGoogle Cloudのものに変更したものがこちらです。

## Prerequisites

- [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine?hl=en)
- [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity?hl=en)
- [External Secrets Operator](https://external-secrets.io/)
- [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy?hl=en)
- [Google Cloud Storage](https://cloud.google.com/storage?hl=en)

## Additional Google Cloud components

- Global external Application Load Balancer (Gateway)
- Cloud SQL for PostgreSQL
- Secret Manager
- Certificate Manager
- Google Cloud Storage

### Global external Application Load Balancer (Gateway)

`langfuse-web` へのトラフィックを受け取るLoadBalancerを作成します。
クラスタ内の別のアプリケーションへLBを流用できるように、`allowedRoutes` のnamespacesは`from: All` を設定しています。

### Cloud SQL for PostgreSQL

langfuseのデータベースにはCloud SQL for PostgreSQLを使用するので、事前に用意してください。
Podにcloud sql proxyコンテナを追加しています。
Podに設定するKSAに、Cloud SQL Clientのロールを付与したGSAに対してWorkload Identityを設定しています。

### Secret Manager

各機密情報はSecret Managerに保存しています。
Secret Managerに保存した値をKubernetesクラスタで利用するために、External Secrets Operatorを使用しています。
External Secrets Operatorも事前にクラスタにインストールしてください。

### Google Cloud Storage

`minio`の代わりにGoogle Cloud Storageを使用します。
バケットを作成し、[こちら](https://langfuse.com/self-hosting/infrastructure/blobstorage#google-cloud-storage)を参考にAccess KeyとSecret Keyを取得してSecret Managerに保存してください。
