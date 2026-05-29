# 01. ソースコード確認 & テスト & 静的解析

> 所要時間: 30分（座学 10分 + ハンズオン 20分）

## 座学: テスト・品質チェックの自動化はなぜ必要か

### 本セクションで使用するツール

本セクションでは、以下の 4 つのツールを使用します。いずれも Java プロジェクトでよく利用される品質管理ツールです。

| ツール | 分類 | 概要 |
|--------|------|------|
| **Flyway** | DB マイグレーション | データベースのスキーマ変更（テーブル作成・列追加等）をバージョン管理し、アプリ起動時に自動適用するツール |
| **SpotBugs** | 静的解析（バグ検出） | Java のバイトコードを解析し、NullPointerException やリソースリークなどの潜在的バグを検出するツール |
| **Checkstyle** | 静的解析（規約チェック） | コーディング規約（命名規則、インデント、Javadoc 等）への準拠をチェックするツール |
| **PMD** | 静的解析（コード品質） | 未使用変数、冗長コード、複雑すぎるメソッド等のコード品質問題を検出するツール |

### 手作業の「煩雑さ」を体験する

本セクションでは、テスト実行・静的解析・DBマイグレーション確認といった品質チェックを**すべて手作業で**実施します。実際にやってみると、以下のような課題に気づくはずです。

| 課題 | 具体例 |
|------|--------|
| **繰り返しの手間** | コード変更の度に `mvn test` → 結果確認 → `mvn spotbugs:check` → `mvn checkstyle:check` → `mvn pmd:check` → 結果確認... を毎回手動で繰り返す |
| **テストの抜け漏れ** | テストケースが 10 件超になると、目視確認だけでは漏れが生じる |
| **マイグレーション順序の管理** | Flyway のバージョンファイルが増えるほど、適用順序の正しさを手動で担保するのが困難 |
| **静的解析の忘れ** | SpotBugs / Checkstyle / PMD を1つでも実行し忘れると、品質の問題が本番に混入する |
| **属人化** | 「誰がどのタイミングで何を実行するか」がドキュメント頼みになる |

> **この煩雑さこそが、CI/CD 自動化の原動力です。** 次回以降の Tekton 編では、これらすべてが **コード push だけで自動実行** されるようになります。

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

### Flyway によるデータ管理

Flyway はデータベースのスキーマ変更や初期データ投入をバージョン管理する仕組みです。

- `V1`, `V2`, `V3`... とファイルを**順番に**作成し、アプリ起動時に自動適用
- テーブル作成だけでなく、**初期データ（シードデータ）の投入**にも利用できる
- 本番環境でも開発環境でも同じ手順でデータベースを再現可能

> **ポイント**: Flyway を使うことで、「本番 DB を手動で ALTER TABLE するリスク」を排除し、マイグレーションを Git で管理できます。

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
| `src/main/resources/db/migration/V1__create_notes.sql` | Flyway: テーブル作成 |
| `src/main/resources/db/migration/V2__create_notes_seq.sql` | Flyway: シーケンス作成 |
| `src/main/resources/db/migration/V3__add_status_to_notes.sql` | Flyway: ステータス列追加 |
| `src/main/resources/db/migration/V4__insert_seed_data.sql` | Flyway: 初期データ投入 |
| `src/main/resources/application.properties` | 本番用設定 (PostgreSQL) |
| `src/test/resources/application.properties` | テスト用設定 (H2) |
| `src/test/java/com/example/GreetingResourceTest.java` | Greeting API テスト (2件) |
| `src/test/java/com/example/NoteResourceTest.java` | Note API テスト (6件) |
| `src/test/java/com/example/HealthCheckTest.java` | ヘルスチェック テスト (3件) |

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

> **Panache とは**: Quarkus が提供するデータアクセスライブラリで、JPA (Java Persistence API) の定型的な記述を大幅に簡略化します。`PanacheEntity` を継承するだけで、`listAll()`, `persist()`, `findById()` などの CRUD メソッドが自動的に使えるようになります。

- **`Note.java`（エンティティ）**: データベースの `notes` テーブルに対応する Java クラス。`PanacheEntity` を継承し、テーブルの各列がフィールドにマッピングされます
- **`NoteResource.java`（API リソース）**: エンティティを操作する REST API を定義するクラス。エンティティの CRUD 操作を HTTP エンドポイントとして公開します
- `GET /notes` -- PostgreSQL からノート一覧を取得
- `POST /notes` -- PostgreSQL にノートを保存

### 4. Flyway マイグレーションの確認

マイグレーションファイルを**すべて**確認します。ファイルが増えるほど、手動確認の負荷が増すことを実感してください。

```bash
ls src/main/resources/db/migration/
```

```
V1__create_notes.sql
V2__create_notes_seq.sql
V3__add_status_to_notes.sql
V4__insert_seed_data.sql
```

それぞれの内容を確認します:

```bash
cat src/main/resources/db/migration/V1__create_notes.sql
```

V1 は `notes` テーブルを作成します。

```bash
cat src/main/resources/db/migration/V2__create_notes_seq.sql
```

V2 は Hibernate 用のシーケンスを作成します。

```bash
cat src/main/resources/db/migration/V3__add_status_to_notes.sql
```

V3 は `status` 列と `updated_at` 列を追加します。機能追加に伴うスキーマ変更の例です。

```bash
cat src/main/resources/db/migration/V4__insert_seed_data.sql
```

V4 は**初期データ（シードデータ）**を投入します。開発環境やテスト環境で同じデータを簡単に再現できます。

> **手動の場合**: マイグレーションファイルが追加されるたびに、手動で `V1 → V2 → V3 → V4` の順序が正しいか確認し、適用済みかどうかを `flyway_schema_history` テーブルで確認する必要があります。ファイルが増えるほど、この手間は膨らみます。

### 5. テストコードの確認

テストファイルが**3ファイル・計11テストケース**あることを確認します（1テストケース = 1つの `@Test` メソッド）。

```bash
ls src/test/java/com/example/
```

```
GreetingResourceTest.java
HealthCheckTest.java
NoteResourceTest.java
```

各テストの内容を確認します:

```bash
cat src/test/java/com/example/GreetingResourceTest.java
```

- `testHelloEndpoint` -- `/hello` のレスポンスを検証
- `testInfoEndpoint` -- `/info` のレスポンス項目を検証

```bash
cat src/test/java/com/example/NoteResourceTest.java
```

- `testCreateAndListNotes` -- ノートの作成と一覧取得
- `testCreateNoteWithLongContent` -- 長いコンテンツの保存
- `testCreateNoteWithoutContent` -- コンテンツ省略時の動作
- `testCreateMultipleNotes` -- 複数件の連続作成
- `testNoteTimestampIsSet` -- タイムスタンプの自動設定
- `testListNotesReturnsJson` -- レスポンスの Content-Type 検証

```bash
cat src/test/java/com/example/HealthCheckTest.java
```

- `testLivenessEndpoint` -- `/health/live` の応答
- `testReadinessEndpoint` -- `/health/ready` の応答
- `testHealthEndpoint` -- `/health` 全体の応答

テスト用の設定:

```bash
cat src/test/resources/application.properties
```

テストプロファイルでは H2 in-memory データベースが使用されるため、PostgreSQL がなくてもテストを実行できます。

### 6. テストの実行

```bash
mvn test
```

実行結果を確認します。**すべてのテストが PASS していること**を確認してください。

```
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
```

> **手動の場合**: テストケースが今は 11 件ですが、実際のプロジェクトでは **数百〜数千件** に膨れ上がります。すべてのテスト結果を毎回手動で確認し、失敗を見落とさないようにするのは現実的ではありません。

### 7. テストレポートの確認

```bash
ls target/surefire-reports/
cat target/surefire-reports/com.example.GreetingResourceTest.txt
cat target/surefire-reports/com.example.NoteResourceTest.txt
cat target/surefire-reports/com.example.HealthCheckTest.txt
```

> **手動の場合**: テストクラスごとにレポートファイルを開いて確認する必要があります。クラスが 3 つなら 3 ファイル、10 クラスなら 10 ファイル...。

### 8. 静的解析の手動実行（3ツール）

テストが通っても、コード品質に問題があるかもしれません。`pom.xml` に設定済みの **3つの静的解析ツール** を順番に実行します。サーバーの構築は不要で、Maven プラグインだけで動作します。

#### 8-1. SpotBugs（バグ検出）

```bash
mvn spotbugs:check
```

| チェック項目 | 内容 |
|-------------|------|
| Null参照 | NullPointerException の可能性 |
| リソースリーク | close されない Stream / Connection |
| 並行性の問題 | スレッドセーフでないコードパターン |

出力を確認し、問題がないことを確認してください。

#### 8-2. Checkstyle（コーディング規約）

```bash
mvn checkstyle:check
```

| チェック項目 | 内容 |
|-------------|------|
| 命名規約 | クラス名・変数名の命名パターン |
| フォーマット | インデント、空白、改行のスタイル |
| Javadoc | ドキュメントコメントの有無・形式 |

違反がある場合、該当箇所と規約名が表示されます。

#### 8-3. PMD（コード品質）

```bash
mvn pmd:check
```

| チェック項目 | 内容 |
|-------------|------|
| 未使用変数 | 宣言されているが使われていない変数 |
| 複雑度 | メソッドの循環的複雑度が高すぎないか |
| 冗長コード | 不要な import、空の catch ブロック等 |

> **手動の場合**: 3つのツールを**毎回すべて**実行する必要があります。1つでも忘れると問題を見逃します。さらに、違反が見つかった場合はコードを修正して、再度 `mvn test` → `mvn spotbugs:check` → `mvn checkstyle:check` → `mvn pmd:check` を繰り返します。このサイクルを**コード変更のたびに**手動で行うのは大きな負担です。

---

### ここまでの手作業を振り返る

1回のコード変更に対して、ここまでで以下の作業をすべて手動で実施しました:

```
ソースコード確認  →  Flyway 確認 (4ファイル)  →  テスト実行 (11件)
     →  テストレポート確認 (3ファイル)
     →  SpotBugs 実行  →  Checkstyle 実行  →  PMD 実行  →  各結果確認
```

これが**毎回のコード変更ごとに**必要になります。次回の CI 編 (Tekton) では、これらすべてが `git push` だけで自動実行されるようになります。

> **現場での話**: チーム 5 人が 1 日 3 回コードを push すると、**1 日 15 回**この手順が必要です。「急いでいるからテストだけ」「SpotBugs は省略」が常態化し、本番障害につながるのはよくあるパターンです。CI があれば、省略は不可能になります。

---

**次のセクション**: [02. イメージビルド](02-image-build.md)
