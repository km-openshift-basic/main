# 06. 永続化 - PV と PVC

> 所要時間: 15分（座学 5分 + ハンズオン 10分）

## 座学: PersistentVolume と PersistentVolumeClaim

### なぜ永続化が必要か

- Pod のコンテナファイルシステムは**一時的**（Pod が削除されるとデータも消失）
- データベースのデータは Pod のライフサイクルに依存せず保持する必要がある

### PV / PVC / StorageClass

| 概念 | 説明 |
|------|------|
| **PersistentVolume (PV)** | クラスタ管理者が用意した物理ストレージ |
| **PersistentVolumeClaim (PVC)** | ユーザーがストレージを要求するリソース |
| **StorageClass** | 動的プロビジョニングのテンプレート |

```
PVC (ユーザーの要求)  ───▶  StorageClass  ───▶  PV (自動作成)
 "1Gi ください"            "gp3で作ります"       "EBS 1Gi"
```

### ROSA での StorageClass

ROSA (Red Hat OpenShift Service on AWS) では、AWS EBS (Elastic Block Store) がデフォルトの StorageClass として設定されています。

- PVC を作成すると、EBS ボリュームが**動的に**プロビジョニングされる
- `ReadWriteOnce` (単一ノードからの読み書き) が基本

## ハンズオン

### 1. PVC マニフェストの確認

```bash
cat base/db-pvc.yaml
```

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### 2. PostgreSQL での PVC マウント確認

```bash
cat base/db-deployment.yaml
```

以下の箇所で PVC をマウントしています:

```yaml
volumeMounts:
  - name: db-data
    mountPath: /var/lib/pgsql/data
volumes:
  - name: db-data
    persistentVolumeClaim:
      claimName: db-pvc
```

### 3. PVC の状態確認

```bash
oc get pvc
```

期待される出力:
```
NAME     STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
db-pvc   Bound    pvc-xxx   1Gi        RWO            gp3-csi        5m
```

`STATUS` が `Bound` であれば、PV が正常にプロビジョニングされています。

### 4. データの保存

```bash
# ノートを作成
curl -s -X POST https://${APP_URL}/notes \
  -H "Content-Type: application/json" \
  -d '{"title": "PVC テスト", "content": "このデータは永続化されます"}' | python3 -m json.tool

# ノート一覧を確認
curl -s https://${APP_URL}/notes | python3 -m json.tool
```

### 5. Pod 削除によるデータ永続化の検証

```bash
# PostgreSQL Pod を削除
oc delete pod -l component=db

# Pod が再作成されるのを待つ
oc get pods -w -l component=db
```

Pod が `Running` に戻ったら:

```bash
# アプリ Pod も再起動（DB 接続の再確立のため）
oc rollout restart deployment/app
oc rollout status deployment/app

# データが残っていることを確認
curl -s https://${APP_URL}/notes | python3 -m json.tool
```

先ほど保存したノートが表示されれば、PVC によるデータ永続化が正常に動作しています。

> **ポイント**: PVC がなければ、Pod の再起動でデータベースのデータはすべて消失します。

---

**次のセクション**: [07. ログの確認](07-logging.md)
