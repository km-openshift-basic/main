# OpenShift コンテナアプリケーション ハンズオンワークショップ

## 概要

本ワークショップでは、コンテナ（Pod）化されたアプリケーションにおけるデプロイメントの基礎を OpenShift (ROSA) 上で実践します。

Quarkus で作成された REST API アプリケーションと PostgreSQL データベースを題材に、イメージビルドからデプロイ、サービス公開、設定管理、データ永続化、環境プロモーションまでを一通り手作業で体験します。

## 所要時間

2.5 時間（座学・ハンズオン・QA・休憩含む）

## 対象者

- Red Hat トレーニング DO180 / DO188 受講済みの方
- OpenShift / Kubernetes の実践経験を積みたい方

## ワークショップ構成

| # | セクション | 時間 | 形式 |
|---|-----------|------|------|
| 00 | [イントロダクション](docs/00-introduction.md) | 10分 | 座学 |
| 01 | [ソースコード確認 & テスト](docs/01-source-and-test.md) | 20分 | 座学 + ハンズオン |
| 02 | [イメージビルド](docs/02-image-build.md) | 20分 | 座学 + ハンズオン |
| 03 | [デプロイメント](docs/03-deployment.md) | 10分 | 座学 + ハンズオン |
| 04 | [Service と Route](docs/04-service-route.md) | 15分 | 座学 + ハンズオン |
| -- | 休憩 | 10分 | -- |
| 05 | [ConfigMap と Secret](docs/05-configmap-secret.md) | 20分 | 座学 + ハンズオン |
| 06 | [PV と PVC](docs/06-pv-pvc.md) | 15分 | 座学 + ハンズオン |
| 07 | [ログの確認](docs/07-logging.md) | 8分 | ハンズオン |
| 08 | [プロモーション (dev → prod)](docs/08-promotion.md) | 15分 | 座学 + ハンズオン |
| 09 | [まとめ](docs/09-summary.md) | 7分 | 座学 + QA |

## 関連リポジトリ

| リポジトリ | 説明 |
|-----------|------|
| [app](https://github.com/km-openshift-basic/app) | Quarkus REST API アプリケーション |
| [manifests](https://github.com/km-openshift-basic/manifests) | OpenShift マニフェスト (Kustomize) |

## 事前準備

### 必要なツール

- `oc` コマンド（OpenShift CLI）
- `git`
- `mvn`（Apache Maven 3.9+）
- Java 17（JDK）
- Web ブラウザ

### 環境情報

| 項目 | 値 |
|------|---|
| OpenShift クラスタ URL | （当日案内） |
| OpenShift Console URL | （当日案内） |
| ユーザー名 | （当日案内） |
| パスワード | （当日案内） |

### 事前確認

```bash
# oc コマンドの確認
oc version

# Java の確認
java -version

# Maven の確認
mvn -version

# Git の確認
git --version
```
