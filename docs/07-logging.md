# 07. ログの確認

> 所要時間: 8分（座学 2分 + ハンズオン 6分）

## 座学: OpenShift でのログ管理

### コンテナログの基本

- コンテナの**標準出力 (stdout) / 標準エラー出力 (stderr)** が `oc logs` で取得できる
- Pod が再起動すると前回のログは失われる（`oc logs --previous` で直前の1世代分のみ取得可能）
- 永続的なログ保存が必要な場合は、クラスタレベルの集中ログ基盤（OpenShift Logging / EFK スタック等）を利用する

### よく使う `oc logs` オプション

| オプション | 説明 |
|-----------|------|
| `-f` | リアルタイムでログを追跡（`tail -f` 相当） |
| `--previous` | 前回のコンテナのログを表示（クラッシュ原因の調査に有用） |
| `--since=1h` | 直近1時間のログのみ表示 |
| `--tail=100` | 末尾100行のみ表示 |
| `-c <container>` | Pod 内の特定コンテナのログを指定 |

## ハンズオン

### 1. アプリ Pod のログ確認

```bash
# アプリ Pod のログを表示
oc logs deployment/app
```

出力のポイント:
- Quarkus の起動ログ
- **Flyway のマイグレーションログ**（起動時にテーブルが作成された記録）

Flyway のログ例:
```
INFO  [org.fly.cor.int.lic.VersionPrinter] Flyway Community Edition ...
INFO  [org.fly.cor.int.com.DbMigrate] Current version of schema "public": << Empty Schema >>
INFO  [org.fly.cor.int.com.DbMigrate] Migrating schema "public" to version "1 - create notes"
INFO  [org.fly.cor.int.com.DbMigrate] Successfully applied 1 migration ...
```

### 2. リアルタイムログの追跡

```bash
# リアルタイムでログを追跡
oc logs -f deployment/app
```

別のターミナルを開き、リクエストを送ると HTTP アクセスログがリアルタイムで表示されます:

```bash
# 別のターミナルでは APP_URL が未設定のため、再取得する
export APP_URL=$(oc get route app -o jsonpath='{.spec.host}')

curl -s https://${APP_URL}/hello
curl -s https://${APP_URL}/notes
```

アクセスログの出力例:
```
127.0.0.1 - - [26/May/2026:10:15:30 +0000] "GET /hello HTTP/1.1" 200 60
127.0.0.1 - - [26/May/2026:10:15:35 +0000] "GET /notes HTTP/1.1" 200 245
```

> **補足**: 本アプリでは `application.properties` で `quarkus.http.access-log.enabled=true` を設定しているため、すべての HTTP リクエストがログに記録されます。

ログの追跡を終了するには `Ctrl + C` を押します。

### 3. PostgreSQL Pod のログ確認

```bash
oc logs deployment/db
```

PostgreSQL の起動ログ、接続ログが確認できます。

### 4. OpenShift Console でのログ確認

OpenShift Web Console にログインし、以下の手順でログを確認します。

1. **Workloads > Pods** を選択
2. アプリ Pod をクリック
3. **Logs** タブを選択

コンソールでは、フィルタリングやダウンロードも可能です。

---

**次のセクション**: [08. プロモーション (dev → prod)](08-promotion.md)
