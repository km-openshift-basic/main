# 02. イメージビルド

> 所要時間: 20分（座学 5分 + ハンズオン 15分）

## 座学: コンテナイメージとビルド

### Dockerfile とは

Dockerfile はコンテナイメージを作成するための手順書です。本アプリケーションでは **マルチステージビルド** を採用しています。

```
ステージ1 (builder): Maven でアプリケーションをビルド
    ↓
ステージ2 (runtime): ビルド成果物のみを軽量なランタイムイメージにコピー
```

メリット:
- ビルドツール（Maven, JDK）がランタイムイメージに含まれない
- イメージサイズの削減
- セキュリティの向上

### OpenShift のビルド機構

| 概念 | 説明 |
|------|------|
| **BuildConfig** | ビルドの設定を定義するリソース（ソース、Dockerfile、出力先等） |
| **Build** | BuildConfig に基づく実際のビルド実行 |
| **ImageStream** | OpenShift 内部でイメージを管理するリソース。タグ管理やトリガーに使用 |

`oc new-build` コマンドは、BuildConfig と ImageStream を自動作成します。

## ハンズオン

### 1. Dockerfile の確認

```bash
cat Dockerfile
```

ポイント:
- **ベースイメージ**: `registry.access.redhat.com/ubi8/openjdk-17` (Red Hat Universal Base Image)
- ステージ 1 で `mvn package` を実行してビルド
- ステージ 2 で Quarkus の実行に必要なファイルのみをコピー

### 2. BuildConfig の作成

```bash
oc new-build --name=workshop-app \
  --binary \
  --strategy=docker \
  -l app=workshop-app
```

### 3. ソースコードをアップロードしてビルド開始

```bash
oc start-build workshop-app --from-dir=. --follow
```

`--follow` オプションにより、ビルドログがリアルタイムで表示されます。

### 4. ビルドの進行状況を確認

別のターミナルで:

```bash
# ビルド一覧
oc get builds

# ビルドログの確認
oc logs build/workshop-app-1
```

### 5. ImageStream の確認

ビルドが完了したら、作成されたイメージを確認します。

```bash
# ImageStream 一覧
oc get is

# ImageStream の詳細
oc describe is/workshop-app
```

出力例:
```
Name:         workshop-app
...
Tags:         latest
  ...
  Image Name: sha256:xxxxxx
```

---

### ここまでの手作業の積み上がり

前セクションから続けて、**1回のコード変更**に対して手動で行った作業を振り返ります:

```
01 で実施済み:
  ソースコード確認 → Flyway 確認 (4ファイル) → テスト実行 (11件)
  → テストレポート確認 (3ファイル)
  → SpotBugs 実行 → Checkstyle 実行 → PMD 実行 → 各結果確認

02 で追加:
  → Dockerfile 確認 → BuildConfig 作成 → ビルド開始
  → ビルドログ確認 → ImageStream 確認
```

> **ポイント**: ここまでで既に **13 ステップ以上** の手作業を行っています。テスト通過 → 静的解析 3 種 OK → イメージビルドという一連の流れは、CI パイプラインの基本です。次回の CI 編 (Tekton) では、これらすべてが **`git push` だけで自動実行** されます。

---

**次のセクション**: [03. デプロイメント](03-deployment.md)
