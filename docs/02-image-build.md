# 02. イメージビルド

> 所要時間: 20分（座学 5分 + ハンズオン 15分）

## 座学: コンテナイメージとビルド

### コンテナとコンテナイメージ

- **コンテナ**: アプリケーションとその実行に必要なライブラリ・設定を一つにパッケージし、ホスト OS 上で隔離された環境として実行する仕組みです。「どの環境でも同じように動く」ことが最大のメリットです
- **コンテナイメージ**: コンテナの実行に必要なファイルシステムの**テンプレート（スナップショット）**です。イメージからコンテナを起動します。一度作成したイメージは変更されない（イミュータブル）ため、dev で検証したイメージをそのまま prod で使えます

### コンテナレジストリ

コンテナレジストリは、ビルドしたイメージを保存・配布するためのリポジトリです。

| レジストリ | 説明 |
|-----------|------|
| **OpenShift 内部レジストリ** | クラスタに組み込まれたレジストリ。ビルドしたイメージが自動で push される（本ワークショップで使用） |
| **Docker Hub** | パブリックな汎用レジストリ |
| **Quay.io** | Red Hat が提供するエンタープライズ向けレジストリ |
| **Amazon ECR / Google GCR** | 各クラウドプロバイダのマネージドレジストリ |

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
| **Build** | BuildConfig に基づく実際のビルド実行。Pod として起動され、完了後に自動削除される |
| **ImageStream** | OpenShift 内部でイメージを管理するリソース。タグ管理やトリガーに使用 |

`oc new-build` コマンドは、BuildConfig と ImageStream を自動作成します。

### OpenShift でのビルドフロー

以下は、ソースコードからコンテナイメージが作成され、デプロイされるまでの全体像です。

```
┌──────────────────────────────────────────────────────────────────────────┐
│  OpenShift クラスタ                                                      │
│                                                                          │
│  ① oc start-build          ② Build Pod が起動           ③ イメージを     │
│     (ソース送信)               (Dockerfile でビルド)        レジストリに    │
│                                                             push         │
│  ┌────────┐   ソース    ┌──────────────┐   イメージ   ┌──────────────┐  │
│  │ 開発者  │──────────▶│  Build Pod   │────────────▶│ 内部レジストリ │  │
│  └────────┘           │ (BuildConfig) │             └───────┬──────┘  │
│                       └──────────────┘                     │          │
│                                                            │          │
│                       ④ ImageStream が      ⑤ Deployment   │          │
│                          タグを記録            が Pod を起動  │          │
│                       ┌──────────────┐     ┌──────────┐    │          │
│                       │ ImageStream  │◀────│Deployment │◀───┘          │
│                       │ :latest      │     │          │               │
│                       └──────────────┘     └──────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

### OpenShift 内部レジストリ

OpenShift には**内部コンテナレジストリ**が組み込まれています。

- ビルドしたイメージはこの内部レジストリに自動で push される
- クラスタ内部からは `image-registry.openshift-image-registry.svc:5000/<project>/<image>:<tag>` でアクセス
- 外部のレジストリ（Docker Hub, Quay.io 等）も利用可能だが、内部レジストリを使うことでネットワーク遅延を回避できる

### ImageStream の役割

ImageStream は単なるイメージの参照先リストですが、OpenShift のデプロイと連携する点で重要です。

- イメージの**タグ変更を検知**して、Deployment の自動更新トリガーにできる
- `oc tag` コマンドでタグを操作し、プロモーション（dev → prod）を実現する
- 外部レジストリのイメージも ImageStream として取り込める

## ハンズオン

### 1. Dockerfile の確認

```bash
cat Dockerfile
```

ポイント:
- **ベースイメージ**: `registry.access.redhat.com/ubi8/openjdk-17` (Red Hat Universal Base Image)
  - ベースイメージとは、Dockerfile の `FROM` で指定する土台となるイメージです。OS やランタイム（ここでは RHEL8 + OpenJDK 17）が含まれており、その上にアプリケーションを追加します
  - UBI (Universal Base Image) は Red Hat が提供する商用サポート付きのベースイメージで、OpenShift 環境での利用に最適化されています
- ステージ 1 で `mvn package` を実行してビルド
- ステージ 2 で Quarkus の実行に必要なファイルのみをコピー

### 2. BuildConfig の作成

```bash
oc new-build --name=workshop-app \
  --binary \
  --strategy=docker \
  -l app=workshop-app
```

各オプションの意味:

| オプション | 説明 |
|-----------|------|
| `--name=workshop-app` | BuildConfig と出力先 ImageStream の名前を指定 |
| `--binary` | ソースコードをローカルからアップロードするバイナリビルドモードを使用（Git URL の代わり） |
| `--strategy=docker` | Dockerfile を使用してビルドする戦略を指定（他に `source` 戦略もある） |
| `-l app=workshop-app` | 作成されるリソースに `app=workshop-app` ラベルを付与（後のリソース管理に使用） |

> **補足: `--strategy=source`（S2I）について**
>
> `--strategy=docker` の代わりに `--strategy=source`（**Source-to-Image = S2I**）を使うと、**Dockerfile なしで**コンテナイメージをビルドできます。
>
> | 項目 | Docker 戦略 | Source（S2I）戦略 |
> |------|------------|------------------|
> | Dockerfile | **必要** | **不要** |
> | 仕組み | Dockerfile の手順に従ってビルド | ビルダーイメージがソースコードを自動検出してビルド |
> | 柔軟性 | 高い（ビルド手順を自由に記述） | 規約ベース（言語・フレームワークに応じた標準ビルド） |
> | 用途 | カスタマイズが必要な場合 | 標準的なアプリを素早くビルドしたい場合 |
>
> S2I の使用例:
> ```bash
> oc new-build --name=workshop-app \
>   --binary \
>   --strategy=source \
>   --image-stream=java:17 \
>   -l app=workshop-app
> ```
> `--image-stream=java:17` で OpenShift が提供する Java 17 用ビルダーイメージを指定します。ビルダーイメージがソースコードの検出・ビルド・ランタイム配置を自動で行うため、Dockerfile を書く必要がありません。
>
> 本ワークショップではビルドの仕組みを理解するために `docker` 戦略を使用しています。

### 3. ソースコードをアップロードしてビルド開始

```bash
oc start-build workshop-app --from-dir=. --follow
```

`--follow` オプションにより、ビルドログがリアルタイムで表示されます。

### 4. ビルドの進行状況を確認

> **注**: 手順 3 で `--follow` を使用した場合、同じターミナルではビルドログが表示されています。この手順は**ビルド中**に別のターミナルを開いて、ビルドのステータスを確認する方法です。

別のターミナルで:

```bash
# ビルド一覧
oc get builds

# ビルドログの確認
oc logs build/workshop-app-1
```

ビルドのステータスは以下のように遷移します:

| ステータス | 説明 |
|-----------|------|
| **New** | ビルドが作成された直後の状態 |
| **Pending** | ビルド Pod の起動待ち |
| **Running** | ビルドが実行中（Dockerfile の各ステップを処理中） |
| **Complete** | ビルドが正常に完了し、イメージが内部レジストリに push された |
| **Failed** | ビルドが失敗（Dockerfile のエラー、リソース不足等） |

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
