# 01. ソースコード確認 & テスト

> 所要時間: 20分（座学 5分 + ハンズオン 15分）

## 座学: テスト自動化の重要性

### なぜテストを自動化するのか

- コード変更の度に手動でテストを繰り返すのは非効率
- テストの抜け漏れを防ぎ、品質を一定に保つ
- 「テストが通ったコードだけをビルド・デプロイする」という CI の基本思想

### テスト戦略: テスト時の DB 選択

本アプリケーションでは、環境によって異なるデータベースを使い分けています。

| 環境 | データベース | 理由 |
|------|-------------|------|
| 本番 / OpenShift | PostgreSQL | 実際の運用環境 |
| テスト | H2 (in-memory) | ローカルに PostgreSQL 不要、高速実行 |

この使い分けは Quarkus の設定ファイル（`application.properties`）で管理しています。

### JUnit / REST Assured とは

- **JUnit 5**: Java の標準テストフレームワーク
- **REST Assured**: REST API のテストを直感的に記述できるライブラリ
- **@QuarkusTest**: Quarkus アプリケーションを起動した状態でテストを実行

## ハンズオン

### 1. アプリケーションディレクトリへの移動

> **注**: DevSpaces を利用しているため、リポジトリは既にワークスペースに展開されています。git clone は不要です。

```bash
cd /projects/app
```

### 2. プロジェクト構成の確認

```bash
tree -L 3 --dirsfirst
```

主要なファイル:

| ファイル | 役割 |
|---------|------|
| `pom.xml` | Maven プロジェクト定義・依存関係 |
| `src/main/java/com/example/GreetingResource.java` | `/hello`, `/info` エンドポイント |
| `src/main/java/com/example/NoteResource.java` | `/notes` エンドポイント (REST API) |
| `src/main/java/com/example/Note.java` | Note エンティティ (JPA) |
| `src/main/resources/db/migration/V1__create_notes.sql` | Flyway マイグレーション |
| `src/main/resources/application.properties` | 本番用設定 (PostgreSQL) |
| `src/test/resources/application.properties` | テスト用設定 (H2) |

### 3. ソースコードの確認

#### エンドポイント (`GreetingResource.java`)

```bash
cat src/main/java/com/example/GreetingResource.java
```

- `GET /hello` -- ConfigMap から注入された挨拶メッセージを返却
- `GET /info` -- アプリ情報、環境名、ホスト名を返却

#### エンティティとAPIリソース (`Note.java`, `NoteResource.java`)

```bash
cat src/main/java/com/example/Note.java
cat src/main/java/com/example/NoteResource.java
```

- Panache を使った JPA エンティティ
- `GET /notes` -- PostgreSQL からノート一覧を取得
- `POST /notes` -- PostgreSQL にノートを保存

### 4. Flyway マイグレーションの確認

```bash
cat src/main/resources/db/migration/V1__create_notes.sql
```

このファイルは、アプリケーション起動時に Flyway が自動で実行し、`notes` テーブルを作成します。

### 5. テストコードの確認

```bash
cat src/test/java/com/example/GreetingResourceTest.java
cat src/test/java/com/example/NoteResourceTest.java
```

テスト用の設定:

```bash
cat src/test/resources/application.properties
```

テストプロファイルでは H2 in-memory データベースが使用されるため、PostgreSQL がなくてもテストを実行できます。

### 6. テストの実行

```bash
mvn test
```

実行結果を確認します。すべてのテストが PASS していることを確認してください。

```
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
```

### 7. テストレポートの確認

```bash
ls target/surefire-reports/
cat target/surefire-reports/com.example.GreetingResourceTest.txt
```

---

> **ポイント**: ここで手動実行したテスト (`mvn test`) は、次回の CI 編 (Tekton) で **自動化** されます。コードが push されるたびに Tekton が自動的にテストを実行し、失敗した場合はパイプラインが停止します。

---

**次のセクション**: [02. イメージビルド](02-image-build.md)
