# 03. デプロイメント

> 所要時間: 10分（座学 3分 + ハンズオン 7分）

## 座学: Deployment リソース

### Deployment の役割

Deployment は Pod のデプロイと管理を担うリソースです。

- **宣言的な管理**: 「望ましい状態」をマニフェストに記述し、OpenShift が現在の状態を自動的に一致させる
- **レプリカ管理**: 指定した数の Pod を常に維持
- **ローリングアップデート**: ダウンタイムなしでアプリケーションを更新

### 複数コンポーネントのデプロイ

本アプリケーションは 2 つのコンポーネントで構成されています。

| コンポーネント | Deployment | 役割 |
|---------------|------------|------|
| Quarkus App | `app` | REST API アプリケーション |
| PostgreSQL | `db` | データベース |

Kustomize を使って、これらを環境ごとにまとめてデプロイします。

## ハンズオン

### 1. マニフェストディレクトリへの移動

> **注**: DevSpaces を利用しているため、リポジトリは既にワークスペースに展開されています。git clone は不要です。

```bash
cd /projects/manifests
```

### 2. マニフェスト構成の確認

```bash
tree
```

```
manifests/
├── base/
│   ├── kustomization.yaml
│   ├── app-deployment.yaml     ← Quarkus アプリ
│   ├── app-service.yaml
│   ├── app-route.yaml
│   ├── app-configmap.yaml
│   ├── db-deployment.yaml      ← PostgreSQL
│   ├── db-service.yaml
│   ├── db-secret.yaml
│   └── db-pvc.yaml
└── overlays/
    ├── dev/                    ← dev 環境用
    │   ├── kustomization.yaml
    │   └── patch-config.yaml
    └── prod/                   ← prod 環境用
        ├── kustomization.yaml
        └── patch-config.yaml
```

### 3. Deployment マニフェストの確認

```bash
# アプリの Deployment
cat base/app-deployment.yaml

# PostgreSQL の Deployment
cat base/db-deployment.yaml
```

ポイント:
- `app-deployment.yaml`: 環境変数で DB 接続情報を参照（ConfigMap, Secret）
- `db-deployment.yaml`: PVC をマウントしてデータを永続化

### 4. イメージ参照先の設定

`overlays/dev/kustomization.yaml` の `images` セクションで、`NAMESPACE` をご自身のプロジェクト名に更新します。

```bash
# dev overlay の kustomization.yaml を編集
# NAMESPACE を <user>-devspaces に変更
vi overlays/dev/kustomization.yaml
```

```yaml
images:
  - name: workshop-app
    newName: image-registry.openshift-image-registry.svc:5000/<user>-devspaces/workshop-app
    newTag: latest
```

### 5. dev 環境にデプロイ

```bash
oc apply -k overlays/dev/
```

### 6. Pod の状態確認

```bash
# Pod 一覧
oc get pods

# すべての Pod が Running になるまで待つ
oc get pods -w
```

期待される出力:
```
NAME                   READY   STATUS    RESTARTS   AGE
app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
db-xxxxxxxxxx-xxxxx    1/1     Running   0          30s
```

### 7. Pod の詳細確認

```bash
oc describe pod -l component=app
```

環境変数やボリュームマウントの設定が確認できます。

---

**次のセクション**: [04. Service と Route](04-service-route.md)
