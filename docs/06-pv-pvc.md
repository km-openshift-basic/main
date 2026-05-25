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

### アクセスモード

| モード | 略称 | 説明 |
|--------|------|------|
| **ReadWriteOnce** | RWO | 単一ノードから読み書き可能（本ワークショップで使用） |
| **ReadOnlyMany** | ROX | 複数ノードから読み取り専用 |
| **ReadWriteMany** | RWX | 複数ノードから読み書き可能（EFS 等が必要） |

> **補足**: AWS EBS は RWO のみ対応です。複数の Pod から同時に書き込みたい場合は EFS (ReadWriteMany) が必要になります。

### PVC のライフサイクル

- PVC を削除しても、デフォルトでは PV（実際のデータ）は**即座に削除される**（`reclaimPolicy: Delete`）
- データを保護したい場合は `reclaimPolicy: Retain` を設定できる
- PVC は特定の Pod に紐づかず、**複数の Deployment から再利用可能**

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
  -d '{"title": "PVC テスト", "content": "このデータは永続化されます"}' | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"

# ノート一覧を確認
curl -s https://${APP_URL}/notes | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"
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
curl -s https://${APP_URL}/notes | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin), indent=4, ensure_ascii=False))"
```

先ほど保存したノートが表示されれば、PVC によるデータ永続化が正常に動作しています。

> **ポイント**: PVC がなければ、Pod の再起動でデータベースのデータはすべて消失します。

---

### (Optional) PV を使った Maven キャッシュの高速化

`oc start-build` でイメージをビルドするたびに Maven の依存関係が毎回ダウンロードされ、時間がかかります。PVC にキャッシュを永続化した Pod でビルドすることで、2 回目以降のビルドを大幅に短縮できます。

#### 1. キャッシュ用 PVC の作成

```bash
oc create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-cache
  labels:
    app: workshop-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF
```

#### 2. キャッシュ PVC をマウントしたビルド Pod を起動

```bash
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: build-pod
  labels:
    app: workshop-app
spec:
  securityContext:
    fsGroup: $(oc get namespace $(oc project -q) -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}' | cut -d/ -f1)
  containers:
    - name: builder
      image: registry.access.redhat.com/ubi8/openjdk-17:1.20
      command: ["sleep", "3600"]
      volumeMounts:
        - name: cache
          mountPath: /tmp/m2-cache
  volumes:
    - name: cache
      persistentVolumeClaim:
        claimName: maven-cache
EOF
oc wait --for=condition=Ready pod/build-pod
```

> **補足**: `fsGroup` を設定すると、PVC マウントポイントの所有グループが変更され、コンテナユーザーから書き込み可能になります。ここではヒアドキュメントの変数展開（`<<EOF`）を使って namespace から自動取得しています。

#### 3. ソースコードをコピーしてビルド（1 回目 -- 遅い）

```bash
oc cp /projects/app build-pod:/tmp/build

# 1回目: 依存関係がすべてダウンロードされ PVC にキャッシュされる
time oc exec build-pod -- bash -c "cd /tmp/build && mvn package -DskipTests -B -Dmaven.repo.local=/tmp/m2-cache"
```

#### 4. 再ビルド（2 回目 -- 高速）

```bash
# ビルド出力をクリアして再ビルド
time oc exec build-pod -- bash -c "cd /tmp/build && rm -rf target && mvn package -DskipTests -B -Dmaven.repo.local=/tmp/m2-cache"
```

2 回目は PVC にキャッシュされた依存関係が使われるため、`Downloading from` のログが大幅に減り、ビルド時間が短縮されることを確認してください。

#### 5. クリーンアップ

```bash
oc delete pod build-pod
```

> **補足**: PVC `maven-cache` を削除せずに残しておけば、別の Pod からマウントして同じキャッシュを再利用できます。これが PVC によるデータ永続化のメリットです。

---

**次のセクション**: [07. ログの確認](07-logging.md)
