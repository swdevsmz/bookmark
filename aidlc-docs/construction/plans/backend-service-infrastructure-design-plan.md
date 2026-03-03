# Backend Service - Infrastructure Design Plan

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-03  
**ステージ**: Construction - Infrastructure Design (Planning)

---

## Overview

このプランは、Backend Service の論理的コンポーネント（HTTP Layer、Service Layer、Repository Layer、Data Layer）を実際の AWS インフラストラクチャサービスにマッピングすることを目的としています。

**既決定**: Lambda 128MB、API Gateway、SQLite /tmp、CloudWatch、Terraform  
**明確化必要**: API Gateway 設定、Lambda デプロイメント構造、データベース初期化、IAM 権限

---

## Infrastructure Design Execution Checklist

- [x] Step 1: Domain Questions — User Input Collected
- [x] Step 2: Ambiguity Check — Responses Validated
- [ ] Step 3: Artifacts Generated — infrastructure-design.md and deployment-architecture.md
- [ ] Step 4: Final Approval — User Review Complete

---

## Step 1: Clarifying Questions

このセクションに答えを記入してください。各質問に対して、[Answer]: の後に選択肢またはカスタムコメントを追加してください。

---

### Q1: API Gateway Resource Structure

Lambda ハンドラーと API Gateway の統合方法を確認します。

**背景**: Lambda ハンドラーは単一エントリーポイント（`handler.js`）で 5 つの HTTP メソッド（GET list、GET detail、POST create、PUT update、DELETE）を処理します。

**選択肢**:
- **A) Single Lambda Integration** - API Gateway は 1 つのリソース `/{proxy+}` で全リクエストを Lambda に転送。Lambda 内部でルーティング。（推奨）
- **B) Resource-Based Integration** - API Gateway で 3 リソース（`/bookmarks`、`/bookmarks/{id}`）を定義し、各リソース × メソッドで Lambda 統合。
- **C) Custom Configuration** - 上記以外の統合方法を使用。

[Answer]: B

---

### Q2: Lambda Environment Variables

デプロイメント時に Lambda に設定する環境変数を確認します。

**既知の環境変数**:
- `DB_PATH` = `/tmp/bookmark.db`
- `COGNITO_REGION` = `ap-northeast-1` (日本)
- `COGNITO_USER_POOL_ID` = （Frontend/Auth Service から提供）

**追加設定が必要か?** 以下から選択してください:

- **A) Existing Variables Only** - 上記 3 つの環境変数のみ使用。ロギングレベルなどは hard-coded。
- **B) Add Logging Configuration** - `LOG_LEVEL=ERROR` (エラーのみ記録) を追加。
- **C) Add Monitoring Configuration** - `LOG_LEVEL`、`METRICS_ENABLED=true`、`TRACE_ENABLED=false` を追加。
- **D) Custom Configuration** - 上記以外の環境変数を追加。

[Answer]: C

---

### Q3: SQLite Schema Initialization Strategy

Lambda デプロイメント時に SQLite スキーマを初期化する方法を確認します。

**背景**: SQLite `/tmp/bookmark.db` は Lambda デプロイ時に初期化され、実行時に CRUD が行われます。

**選択肢**:
- **A) Bundled SQL File** - `schema.sql` をコードベースに含め、Lambda 起動時に一回だけ実行（初回起動のみ）。
- **B) Database Initialization Layer** - 専用 `database/` ディレクトリに初期化ロジックをモジュール化し、startup 時に実行。
- **C) Inline SQL Strings** - ハンドラーで SQL を hard-code。
- **D) Event-Driven Init** - 初回リクエスト時に遅延初期化。

[Answer]: A

---

### Q4: Lambda Layer Usage

コード共通化するため Lambda Layers を使用するか確認します。

**背景**: 
- HTTP 層: JWT 検証、入力バリデーション、エラーハンドリング
- Service 層: BookmarkService、UserIdentityService
- Repository 層: BookmarkRepository、UserIdentityRepository
- Utility 層: Retry ロジック、SQLite Connection Manager

**選択肢**:
- **A) Single Code Package** - すべてのコードを単一 Lambda Zip に含める。Layer 不使用。
- **B) Shared Layer** - npm dependencies（sqlite3 など）と共通コードを層化。
- **C) Multi-Layer Strategy** - Business Logic Layer と Database Layer を分離。
- **D) Dependencies Only** - Layer は npm dependencies のみ。アプリケーションコードは Zip に含める。

[Answer]: C

---

### Q5: Networking Configuration

Lambda と他サービス間のネットワーク設定を確認します。

**背景**: Lambda は SQLite（/tmp）とローカル認証情報を使用し、外部呼び出しは最小限（Cognito トークン検証のみ）。

**選択肢**:
- **A) Public Execution** - Lambda は VPC 外で実行。VPC 設定なし。最短デプロイ。
- **B) VPC Integration** - Lambda を VPC 内に配置。NAT Gateway で外部通信。
- **C) VPC with Private Subnets** - Lambda を Private Subnet に配置。詳細な ネットワーク分離。

[Answer]: A

---

### Q6: Database Backup & Persistence Strategy

SQLite `/tmp` ディレクトリは ephemeral なため、データ永続化戦略を確認します。

**背景**: 
- Lambda 実行終了後 `/tmp` は削除される（エフェメラル）
- 次の Lambda 起動時に新しい `/tmp` + 新規 `bookmark.db` が初期化される
- このため、ブックマークデータは各 Lambda コールドスタート時に失われる

**選択肢**:
- **A) Lambda /tmp Only (Current)** - Ephemeral storage のみ。データ永続化なし。デモ/PoC 用。
- **B) S3 Synchronization** - 各リクエスト後に S3 に DB をアップロード。起動時は S3 から復元。
- **C) EBS Volume** - Lambda を EBS Volume にアタッチ。永続化。
- **D) Managed Database** - RDS/DynamoDB 等に移行。

[Answer]: C

---

### Q7: Monitoring & Observability Infrastructure

CloudWatch 以外の監視ツールを使用するか確認します。

**背景**: 既決定 = CloudWatch 30日ログ保持、エラーベースのアラート、カスタムメトリクス対応。

**選択肢**:
- **A) CloudWatch Only** - CloudWatch Logs、CloudWatch Metrics のみ。追加ツール不使用。
- **B) X-Ray Integration** - AWS X-Ray で分散トレーシング有効化。
- **C) CloudWatch + Sentry** - Sentry で例外追跡。
- **D) Custom Monitoring** - 上記以外のツール。

[Answer]: A

---

### Q8: Terraform Backend & State Management

Terraform の state ファイルを管理する方法を確認します。

**背景**: Infrastructure as Code（Terraform）で Lambda、API Gateway、IAM ロール等を管理。

**選択肢**:
- **A) Local State** - ローカル `terraform.tfstate`。開発環境のみ。
- **B) S3 + DynamoDB** - S3 で state 保管、DynamoDB でロック管理。本番向け。
- **C) Terraform Cloud** - Terraform Cloud でリモート state 管理。
- **D) GitHub Actions Integration** - GitHub Actions で自動デプロイ。Terraform Cloud と統合。

[Answer]: A

---

## Next Steps

すべての質問に回答したら、以下のコマンドを実行してください:

1. **本ドキュメントの [Answer]: フィールドをすべて埋める**
2. **回答完了を AI に通知: `回答完了。PRを作成して`**

---

**作成者**: AI-DLC  
**最終更新**: 2026-03-03
