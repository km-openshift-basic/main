# 03. デプロイメント

> 所要時間: 10分（座学 3分 + ハンズオン 7分）

## 座学: Deployment リソース

### Deployment の役割

Deployment は Pod のデプロイと管理を担うリソースです。

- **宣言的な管理**: 「望ましい状態」をマニフェストに記述し、OpenShift が現在の状態を自動的に一致させる
- **レプリカ管理**: 指定した数の Pod を常に維持
- **ローリングアップデート**: ダウンタイムなしでアプリケーションを更新
- **自動復旧**: Pod が異常終了すると、自動的に新しい Pod を起動して指定数を維持

### Deployment が管理するリソースの階層

```
Deployment（望ましい状態を宣言）
  └─ ReplicaSet（指定数の Pod を維持）
       ├─ Pod (1)
       ├─ Pod (2)
       └─ Pod (...)
```

- **ReplicaSet** は Deployment が内部的に作成・管理するリソースで、直接操作することは通常ありません
- アプリ更新時、Deployment は新しい ReplicaSet を作成し、古い Pod を徐々に新しい Pod に置き換えます（ローリングアップデート）

### 複数コンポーネントのデプロイ

本アプリケーションは 2 つのコンポーネントで構成されています。

| コンポーネント | Deployment | 役割 |
|---------------|------------|------|
| Quarkus App | `app` | REST API アプリケーション |
| PostgreSQL | `db` | データベース |

### Kustomize とは

Kustomize は Kubernetes マニフェストの管理ツールで、`oc apply -k` で利用できます。

- **base**: 全環境で共通のリソース定義
- **overlays**: 環境ごとの差分（レプリカ数、設定値、ラベル等）のみを管理
- マニフェストの**コピー**ではなく**パッチ**で差分管理するため、base を修正すれば全環境に反映される

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
vim overlays/dev/kustomization.yaml
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

### 8. Flyway マイグレーションの動作確認

アプリ Pod が起動すると、Flyway が自動で PostgreSQL にマイグレーションを適用します。実際に DB の中身を確認してみましょう。

#### PostgreSQL Pod に接続

```bash
oc rsh deployment/db
```

#### psql でデータベースに接続

```bash
psql -U workshop -d workshop
```

#### マイグレーション履歴の確認

Flyway は `flyway_schema_history` テーブルにマイグレーションの適用履歴を記録します。

```sql
SELECT version, description, installed_on, success FROM flyway_schema_history ORDER BY version;
```

期待される出力:
```
 version |     description      |      installed_on       | success
---------+----------------------+-------------------------+---------
 1       | create notes         | 2026-05-26 01:30:00.000 | t
 2       | create notes seq     | 2026-05-26 01:30:00.000 | t
 3       | add status to notes  | 2026-05-26 01:30:00.000 | t
 4       | insert seed data     | 2026-05-26 01:30:00.000 | t
```

4つのマイグレーションがすべて `success = t` で適用されていることを確認してください。

#### テーブル構造の確認

```sql
\d notes
```

V1 で作成されたテーブルに、V3 で追加された `status` 列と `updated_at` 列が反映されていることを確認します。

#### シードデータの確認

V4 で投入された初期データを確認します。

```sql
SELECT id, title, status, created_at FROM notes;
```

期待される出力:
```
 id |      title       | status |       created_at
----+------------------+--------+-------------------------
  1 | Welcome          | OPEN   | 2026-05-26 01:30:00.000
  2 | Setup Complete   | DONE   | 2026-05-26 01:30:00.000
  3 | Next Steps       | OPEN   | 2026-05-26 01:30:00.000
```

#### psql を終了

```sql
\q
```

```bash
exit
```

> **ポイント**: Flyway によって、アプリ起動時にテーブル作成・スキーマ変更・初期データ投入が**自動的に**行われました。手動で `CREATE TABLE` や `INSERT` を実行する必要はありません。マイグレーションファイルを Git で管理しているため、どの環境でも同じ DB 状態を再現できます。

> **現場での話**: 手動デプロイで障害が起きた場合、「どのイメージに戻せばいいか」「前の設定はどうだったか」をチーム内で確認するだけで時間がかかります。CD (ArgoCD) で Git 管理していれば、`git revert` → 自動 Sync で**数分以内にロールバック**できます。

---

**次のセクション**: [04. Service と Route](04-service-route.md)
