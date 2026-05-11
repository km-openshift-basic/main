# 08. プロモーション: dev → prod

> 所要時間: 15分（座学 3分 + ハンズオン 12分）

## 座学: 環境分離とプロモーション

### なぜ環境を分けるのか

- **dev**: 開発・検証用。頻繁にデプロイ、設定変更が発生
- **prod**: 本番用。安定性が最優先。変更は慎重に実施

環境ごとに異なる設定:

| 項目 | dev | prod |
|------|-----|------|
| アプリレプリカ数 | 1 | 2 |
| 挨拶メッセージ | "Hello from Dev environment!" | "Hello from Production!" |
| 環境名 | dev | prod |
| DB | 独立した PostgreSQL | 独立した PostgreSQL |

### プロモーション戦略

「dev で検証済みのイメージを prod にデプロイする」という考え方です。

```
dev 環境                          prod 環境
┌──────────────┐                 ┌──────────────┐
│ workshop-app │  ── oc tag ──▶  │ workshop-app │
│   :latest    │                 │  :promoted   │
└──────────────┘                 └──────────────┘
```

- dev でビルド・検証したイメージをそのまま prod に渡す（再ビルドしない）
- `oc tag` でイメージを別プロジェクトにコピー（タグ付け）

### Kustomize overlays

base のマニフェストを共通定義とし、overlays で環境差分のみを管理:

```
base/           ← 共通のリソース定義
overlays/dev/   ← dev 環境の差分（replicas=1, dev設定）
overlays/prod/  ← prod 環境の差分（replicas=2, prod設定）
```

## ハンズオン

### 1. prod プロジェクトの作成

```bash
oc new-project <user>-prod
```

### 2. イメージのプロモーション

dev 環境のイメージを prod にタグ付けします。

```bash
oc tag <user>-dev/workshop-app:latest <user>-prod/workshop-app:promoted
```

確認:
```bash
oc get is -n <user>-prod
```

### 3. prod overlay の確認

```bash
cat overlays/prod/kustomization.yaml
cat overlays/prod/patch-config.yaml
```

dev との差分:
- `replicas: 2` （アプリ Pod が 2 つ起動）
- `APP_ENVIRONMENT: "prod"`
- `APP_GREETING: "Hello from Production!"`

### 4. prod overlay のイメージ設定を更新

`overlays/prod/kustomization.yaml` の `images` セクションで `NAMESPACE` を prod プロジェクト名に更新します。

```bash
vi overlays/prod/kustomization.yaml
```

```yaml
images:
  - name: workshop-app
    newName: image-registry.openshift-image-registry.svc:5000/<user>-prod/workshop-app
    newTag: promoted
```

### 5. prod 環境にデプロイ

```bash
oc project <user>-prod
oc apply -k overlays/prod/
```

### 6. Pod の状態確認

```bash
oc get pods
```

期待される出力（アプリ Pod が **2つ** 起動）:
```
NAME                   READY   STATUS    RESTARTS   AGE
app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
app-xxxxxxxxxx-yyyyy   1/1     Running   0          30s
db-xxxxxxxxxx-xxxxx    1/1     Running   0          30s
```

### 7. prod 環境の動作確認

```bash
export PROD_URL=$(oc get route app -o jsonpath='{.spec.host}')

# /info で環境が prod であることを確認
curl -s https://${PROD_URL}/info | python3 -m json.tool

# /hello で prod 用メッセージを確認
curl -s https://${PROD_URL}/hello | python3 -m json.tool
```

期待される出力:
```json
{
    "environment": "prod",
    ...
}
```

### 8. 環境の独立性を確認

prod の DB は dev とは完全に独立しています。

```bash
# prod でノートを作成
curl -s -X POST https://${PROD_URL}/notes \
  -H "Content-Type: application/json" \
  -d '{"title": "Prod ノート", "content": "これは prod 環境のデータです"}' | python3 -m json.tool

# prod のノート一覧（prod のデータのみ表示される）
curl -s https://${PROD_URL}/notes | python3 -m json.tool

# dev に戻ってノート一覧を確認（dev のデータのみ）
curl -s https://${APP_URL}/notes | python3 -m json.tool
```

---

> **ポイント**: ここで手動実行した「イメージタグ付け → prod へのデプロイ」は、次回の CD 編 (ArgoCD) で **自動化** されます。Git リポジトリにマニフェストを push するだけで、ArgoCD が自動的に検知・Sync します。

---

**次のセクション**: [09. まとめ](09-summary.md)
