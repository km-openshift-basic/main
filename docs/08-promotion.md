# 08. プロモーション

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
dev 設定                            prod 設定
┌──────────────┐                 ┌──────────────┐
│ workshop-app │  ── oc tag ──▶  │ workshop-app │
│   :latest    │                 │  :promoted   │
└──────────────┘                 └──────────────┘
```

- dev でビルド・検証したイメージをそのまま prod に渡す（再ビルドしない）
- `oc tag` でイメージにタグ付け（検証済みであることを明示）

> **本ワークショップでは**: 通常は dev / prod で別プロジェクトを使いますが、今回は `<user>-devspaces` プロジェクト 1 つで実施します。同一プロジェクト内でイメージタグとKustomize overlay を切り替えることで、プロモーションの流れを体験します。

### Kustomize overlays

base のマニフェストを共通定義とし、overlays で環境差分のみを管理:

```
base/           ← 共通のリソース定義
overlays/dev/   ← dev 環境の差分（replicas=1, dev設定）
overlays/prod/  ← prod 環境の差分（replicas=2, prod設定）
```

## ハンズオン

### 1. イメージのプロモーション

dev 設定で検証済みのイメージに `promoted` タグを付与します。

```bash
oc tag workshop-app:latest workshop-app:promoted
```

確認:
```bash
oc get is workshop-app
```

`latest` と `promoted` の 2 つのタグが表示されます。

### 2. prod overlay の確認

```bash
cat overlays/prod/kustomization.yaml
cat overlays/prod/patch-config.yaml
```

dev との差分:
- `replicas: 2` （アプリ Pod が 2 つ起動）
- `APP_ENVIRONMENT: "prod"`
- `APP_GREETING: "Hello from Production!"`

### 3. prod overlay のイメージ設定を更新

`overlays/prod/kustomization.yaml` の `images` セクションで `NAMESPACE` をプロジェクト名に更新します。

```bash
vim overlays/prod/kustomization.yaml
```

```yaml
images:
  - name: workshop-app
    newName: image-registry.openshift-image-registry.svc:5000/<user>-devspaces/workshop-app
    newTag: promoted
```

### 4. prod 設定でデプロイ

prod overlay を適用して、設定とレプリカ数を prod 仕様に切り替えます。

```bash
oc apply -k overlays/prod/
```

### 5. Pod の状態確認

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

### 6. prod 設定の動作確認

```bash
export APP_URL=$(oc get route app -o jsonpath='{.spec.host}')

# /info で環境が prod であることを確認
curl -s https://${APP_URL}/info | python3 -m json.tool

# /hello で prod 用メッセージを確認
curl -s https://${APP_URL}/hello | python3 -m json.tool
```

期待される出力:
```json
{
    "environment": "prod",
    ...
}
```

### 7. dev 設定に戻す

確認が終わったら、dev overlay を再適用して元の状態に戻します。

```bash
oc apply -k overlays/dev/
```

```bash
curl -s https://${APP_URL}/info | python3 -m json.tool
```

環境が `dev` に戻っていることを確認してください。

---

> **ポイント**: ここで手動実行した「イメージタグ付け → prod 設定でのデプロイ」は、次回の CD 編 (ArgoCD) で **自動化** されます。Git リポジトリにマニフェストを push するだけで、ArgoCD が自動的に検知・Sync します。

---

**次のセクション**: [09. まとめ](09-summary.md)
