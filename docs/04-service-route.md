# 04. サービス公開 - Service と Route

> 所要時間: 15分（座学 5分 + ハンズオン 10分）

## 座学: Service と Route

### Service の役割

Service は Pod へのネットワークアクセスを提供する抽象レイヤーです。

- Pod の IP アドレスは起動のたびに変わるが、Service は固定のアクセスポイントを提供
- **ClusterIP** (デフォルト): クラスタ内部からのみアクセス可能
- ラベルセレクタで対象 Pod を決定

### Pod 間通信と外部公開の違い

```
[外部]  ──Route──▶ [App Service] ──▶ [App Pod]
                                        │
                                        ▼ (ClusterIP)
                                   [DB Service] ──▶ [DB Pod]
```

| 通信経路 | 使用リソース | 説明 |
|---------|-------------|------|
| 外部 → App | Route + Service | ブラウザ等からアクセス |
| App → DB | Service のみ | クラスタ内部通信。DB は外部公開しない |

### Route の役割

Route は OpenShift 固有のリソースで、Service を外部に公開します。

- DNS ホスト名を自動割り当て
- TLS 終端（HTTPS 対応）
- Kubernetes の Ingress に相当する機能

## ハンズオン

### 1. Service と Route マニフェストの確認

```bash
# アプリの Service
cat base/app-service.yaml

# アプリの Route
cat base/app-route.yaml

# DB の Service（ClusterIP のみ、Route なし）
cat base/db-service.yaml
```

ポイント:
- `app-service.yaml`: ポート 8080 でアプリ Pod に接続
- `app-route.yaml`: TLS edge termination で HTTPS 公開
- `db-service.yaml`: ポート 5432 で DB Pod に接続。**Route は作成しない**（外部公開不要）

### 2. デプロイ済みの Service と Route を確認

Kustomize で既にデプロイ済みのリソースを確認します。

```bash
# Service 一覧
oc get svc

# Route 一覧
oc get route
```

期待される出力:
```
NAME   HOST/PORT                                    ...
app    app-<user>-devspaces.apps.<cluster-domain>     ...
```

### 3. アプリの URL を取得

```bash
export APP_URL=$(oc get route app -o jsonpath='{.spec.host}')
echo "https://${APP_URL}"
```

### 4. エンドポイントにアクセス

```bash
# /hello エンドポイント
curl -s https://${APP_URL}/hello | python3 -m json.tool

# /info エンドポイント
curl -s https://${APP_URL}/info | python3 -m json.tool
```

期待される出力 (`/info`):
```json
{
    "application": "openshift-workshop-app",
    "version": "1.0.0",
    "environment": "dev",
    "javaVersion": "17.x.x",
    "hostname": "app-xxxxxxxxxx-xxxxx"
}
```

### 5. ブラウザからのアクセス

ブラウザで以下の URL にアクセスしてみましょう。

- `https://<APP_URL>/hello`
- `https://<APP_URL>/info`

### 6. OpenShift Console での確認

OpenShift Web Console にログインし、以下を確認します。

- **Networking > Services**: `app` と `db` の 2 つの Service
- **Networking > Routes**: `app` の Route（DB には Route がないことを確認）

---

**次のセクション**: 休憩（10分）の後、[05. ConfigMap と Secret](05-configmap-secret.md)
