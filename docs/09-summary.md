# 09. まとめ

> 所要時間: 7分（振り返り + QA）

## 本日のハンズオン振り返り

本日は、以下の作業を**すべて手作業で**実施しました。

| # | 作業 | 使用したコマンド・リソース |
|---|------|--------------------------|
| 01 | テスト実行 | `mvn test` |
| 02 | イメージビルド | `oc new-build`, `oc start-build` |
| 03 | デプロイ | `oc apply -k overlays/dev/` |
| 04 | サービス公開 | Service + Route |
| 05 | 設定・機密情報の管理 | ConfigMap + Secret |
| 06 | データ永続化 | PVC |
| 07 | ログ確認 | `oc logs`, Console |
| 08 | プロモーション | `oc tag`, `oc apply -k overlays/prod/` |

## CI/CD パイプラインとの対比

今回手作業で行った各ステップは、次回以降のワークショップで**自動化**されます。

### テスト & ビルド → CI 編 (Tekton) で自動化

| 本日の手作業 | CI 編での自動化 |
|-------------|----------------|
| `mvn test` を手動実行 | Tekton Task がコード push 時に自動テスト実行 |
| テスト結果を目視確認 | テスト失敗時にパイプラインが自動停止 |
| `oc start-build` で手動ビルド | Tekton Task がテスト通過後に自動ビルド |

### デプロイ & プロモーション → CD 編 (ArgoCD) で自動化

| 本日の手作業 | CD 編での自動化 |
|-------------|----------------|
| `oc apply -k` で手動デプロイ | ArgoCD が Git の manifests リポジトリから自動 Sync |
| ConfigMap/Secret を手動編集・再適用 | Git push だけで ArgoCD が差分検知・自動 Sync |
| `oc tag` で手動イメージプロモーション | パイプライン完了後に自動でイメージタグ更新 |
| `oc apply -k overlays/prod/` で手動 prod デプロイ | ArgoCD が prod overlay を監視し自動反映 |

### 全体の流れ

```
  本日 (手作業)                CI 編 (自動化)           CD 編 (自動化)
┌─────────────────┐      ┌──────────────────┐    ┌──────────────────┐
│                 │      │                  │    │                  │
│  mvn test       │─────▶│  Tekton Task     │    │                  │
│  (手動テスト)    │      │  (自動テスト)     │    │                  │
│        │        │      │        │         │    │                  │
│        ▼        │      │        ▼         │    │                  │
│  oc start-build │─────▶│  Tekton Task     │    │                  │
│  (手動ビルド)    │      │  (自動ビルド)     │    │                  │
│        │        │      │        │         │    │                  │
│        ▼        │      │        ▼         │    │                  │
│  oc apply -k    │      │  Tekton Task     │───▶│  ArgoCD Sync     │
│  dev/ (手動)    │      │  (タグ更新)       │    │  (dev 自動反映)   │
│        │        │      │                  │    │        │         │
│        ▼        │      │                  │    │        ▼         │
│  oc tag +       │      │                  │───▶│  ArgoCD Sync     │
│  oc apply -k    │      │                  │    │  (prod 自動反映)  │
│  prod/ (手動)   │      │                  │    │                  │
│                 │      │                  │    │                  │
└─────────────────┘      └──────────────────┘    └──────────────────┘
```

## 次回予告

- **第2回: CI 編 (Tekton)** -- テストとビルドを Tekton Pipeline で自動化
- **第3回: CD 編 (ArgoCD)** -- Git push をトリガーにデプロイを ArgoCD で自動化

## クリーンアップ

ワークショップで作成したリソースを削除する場合:

```bash
oc delete all -l app=workshop-app
oc delete configmap,secret,pvc -l app=workshop-app
```

> **注**: DevSpaces のプロジェクト (`<user>-devspaces`) 自体は削除しないでください。

## Q&A

質問がありましたらお気軽にどうぞ。
