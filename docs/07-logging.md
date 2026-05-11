# 07. ログの確認

> 所要時間: 8分（ハンズオン）

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

別のターミナルからリクエストを送ると、ログにアクセス記録が表示されます:

```bash
curl -s https://${APP_URL}/hello
curl -s https://${APP_URL}/notes
```

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
