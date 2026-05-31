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
| 挨拶メッセージ | "こんにちは、OpenShift ワークショップへようこそ！"（05 で変更済み） | "Hello from Production!" |
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

- dev でビルド・検証したイメージをそのまま prod に渡す（**再ビルドしない**）
- `oc tag` でイメージにタグ付け（検証済みであることを明示）

> **なぜ再ビルドしないのか**: 再ビルドすると、依存ライブラリのバージョン差異やビルドタイミングの違いにより、dev で検証したものと異なるイメージが生成されるリスクがあります。同一イメージをタグで管理することで、「dev で動いたものが prod でも動く」ことを保証します。

> **本ワークショップでは**: 通常は dev / prod で別プロジェクトを使いますが、今回は `<user>-devspaces` プロジェクト 1 つで実施します。同一プロジェクト内でイメージタグとKustomize overlay を切り替えることで、プロモーションの流れを体験します。

### Kustomize overlays

base のマニフェストを共通定義とし、overlays で環境差分のみを管理:

```
base/           ← 共通のリソース定義
overlays/dev/   ← dev 環境の差分（replicas=1, dev設定, env=dev ラベル）
overlays/prod/  ← prod 環境の差分（replicas=2, prod設定, env=prod ラベル, prod- プレフィックス）
```

> **同一プロジェクトでの共存**: prod overlay は `namePrefix: prod-` を使い、全リソース名に `prod-` を付与します。また `env: prod` / `env: dev` ラベルをセレクタに含めることで、dev と prod のリソースが同一プロジェクト内で衝突せず共存できます。

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

| 項目 | dev | prod |
|------|-----|------|
| リソース名 | `app`, `db`, `db-pvc` ... | `prod-app`, `prod-db`, `prod-db-pvc` ... |
| レプリカ数 | 1 | 2 |
| 環境ラベル | `env: dev` | `env: prod` |
| 挨拶メッセージ | "こんにちは、OpenShift ワークショップへようこそ！"（05 で変更済み） | "Hello from Production!" |
| DB 接続先 | `db:5432` | `prod-db:5432` |
| イメージタグ | `latest` | `promoted` |

### 3. prod overlay のイメージ設定を更新

`overlays/prod/kustomization.yaml` の `images` セクションで `<user>-devspaces` をご自身のプロジェクト名に更新します。

```bash
# <user>-devspaces を自分のプロジェクト名に置換
sed -i "s/<user>-devspaces/$(oc project -q)/g" overlays/prod/kustomization.yaml
```

置換結果を確認:

```bash
cat overlays/prod/kustomization.yaml
```

`images` セクションが以下のようになっていれば OK です:

```yaml
images:
  - name: workshop-app
    newName: image-registry.openshift-image-registry.svc:5000/userXX-devspaces/workshop-app
    newTag: promoted
```

### 4. prod 設定でデプロイ

prod overlay を適用します。`namePrefix: prod-` により、dev とは別のリソース群として作成されます。

```bash
oc apply -k overlays/prod/
```

### 5. Pod の状態確認

```bash
oc get pods
```

期待される出力（dev と prod の Pod が**共存**）:
```
NAME                        READY   STATUS    RESTARTS   AGE
app-xxxxxxxxxx-xxxxx        1/1     Running   0          5m     ← dev (1台)
db-xxxxxxxxxx-xxxxx         1/1     Running   0          5m     ← dev DB
prod-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s    ← prod (2台)
prod-app-xxxxxxxxxx-yyyyy   1/1     Running   0          30s    ← prod (2台)
prod-db-xxxxxxxxxx-xxxxx    1/1     Running   0          30s    ← prod DB
```

### 6. prod 設定の動作確認

```bash
# prod 用 Route は prod-app という名前
export PROD_URL=$(oc get route prod-app -o jsonpath='{.spec.host}')

# /info で環境が prod であることを確認
curl -s https://${PROD_URL}/info | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"

# /hello で prod 用メッセージを確認
curl -s https://${PROD_URL}/hello | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"
```

期待される出力:
```json
{
    "environment": "prod",
    ...
}
```

### 7. dev と prod の比較確認

ここでは、**同一プロジェクト内で dev と prod が同時に動作していること**、そして**それぞれが独立した設定（環境名、挨拶メッセージ、レプリカ数等）で稼働していること**を確認します。これにより、Kustomize overlays による環境分離が正しく機能していることが実証できます。

```bash
export DEV_URL=$(oc get route app -o jsonpath='{.spec.host}')

# dev 側
curl -s https://${DEV_URL}/info | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"

# prod 側
curl -s https://${PROD_URL}/info | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"
```

### 8. prod リソースの削除（オプション）

prod 環境のみ削除したい場合:

```bash
oc delete all -l env=prod
oc delete configmap,secret,pvc -l env=prod
```

---

> **ポイント**: ここで手動実行した「イメージタグ付け → prod 設定でのデプロイ」は、次回の CD 編 (ArgoCD) で **自動化** されます。Git リポジトリにマニフェストを push するだけで、ArgoCD が自動的に検知・Sync します。

> **現場での話**: 手動プロモーションでは「テスト未実行のイメージを誤って prod にデプロイ」「タグの指定ミスで古いイメージをデプロイ」といった事故が起こり得ます。CI/CD パイプラインでは、テスト・静的解析がすべて通過した場合にのみ `promoted` タグが付与されるため、**品質が担保されていないイメージが本番に到達することはありません**。

---

**次のセクション**: [10. 補足: RBAC と ServiceAccount](10-rbac.md)
