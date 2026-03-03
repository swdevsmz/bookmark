# Backend Service - Infrastructure Design

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-03  
**ステージ**: Construction - Infrastructure Design

---

## 1. Infrastructure Overview

Backend Service は AWS Serverless アーキテクチャ上で稼働する Lambda ベースの API 層です。
主要なインフラコンポーネントは以下の通り:

- **API Gateway (REST API)**
- **AWS Lambda**（1つのハンドラー `handler.js`）
- **EBS ボリューム**（Lambda にアタッチして SQLite データを永続化）
- **IAM ロールとポリシー**（Lambda 実行、EBS アクセス、Cognito 認証）
- **CloudWatch Logs & Metrics**（30日保持、エラーベースアラート）
- **Terraform** によるインフラコード管理（ローカル state）

API Gateway は 3 つのリソース (`/bookmarks`、`/bookmarks/{id}`) と各種 HTTP メソッドを定義し、各メソッドを同じ Lambda に統合するリソースベース構成を採用する。Lambda 内で受け取ったリクエストをルーターが振り分ける。

Lambda は 128 MB メモリ・10秒タイムアウトで展開され、/?proxy+ のような単一リソースではなく個別リソースから呼び出される。

環境変数は、

- `DB_PATH` = `/tmp/bookmark.db`
- `COGNITO_REGION` = `ap-northeast-1`
- `COGNITO_USER_POOL_ID` = フロントエンドまたは Auth Service から設定
- `LOG_LEVEL` = `<実装時に設定>`
- `METRICS_ENABLED` = `true`
- `TRACE_ENABLED` = `false`

としており、監視構成を有効化している。

コードは複数の Lambda Layer に分割し、ビジネスロジック層とデータ層を分離（Multi-Layer Strategy）。

ネットワークは VPC 外部で実行し、Cognito へのトークン検証以外の外部通信は発生しない。

## 2. Compute & Deployment

### 2.1 Lambda Configuration

- **Handler**: `index.handler`（パッケージ内で `handler.js` をエクスポート）
- **Memory**: 128 MB
- **Timeout**: 10 s
- **Concurrency**: デフォルト
- **Layers**:
  - `common-deps-layer`（npm 依存関係、sqlite3 など）
  - `business-logic-layer`（サービス/リポジトリ/ユーティリティコード）
- **Environment Variables**: 上記参照
- **IAM Role**:
  - `AWSLambdaBasicExecutionRole`（CloudWatch Logs）
  - `AmazonEC2FullAccess` のうち最低限 EBS への `AttachVolume` / `DetachVolume` 権限
  - Cognito ID トークン検証用 `cognito-idp:DescribeUserPool` など

### 2.2 Storage

- **EBS Volume** (gp3, 10 GB) を Lambda 実行時にアタッチし、永続的な SQLite データを格納する。
- **DB Initialization**: デプロイパッケージに含む `schema.sql` を Lambda 起動時に `/tmp/bookmark.db` に読み込み（Bundled SQL File）。
- バックアップ/保持は EBS のスナップショット機能に任せる。

### 2.3 State Management

Terraform を使用し、ローカル `terraform.tfstate` で管理。開発用ワークフローでは Git に同梱せず、個別に管理する。

## 3. Networking & Security

- **VPC**: 未使用（Public Execution）。
- **Security Groups**: Lambda のデフォルト。
- **API Gateway**: Cognito オーソライザーを設定し、JWT を検証。
- **Rate Limiting**: API Gateway でリクエストレート制限を設定（100 RPS など）。

## 4. Monitoring & Observability

- **CloudWatch Logs**: 30 日ログ保持、エラーメッセージのみ記録（`LOG_LEVEL=ERROR`）、メトリクスはカスタムでリクエスト時間とエラー率。
- **CloudWatch Metrics**: レスポンスタイム、エラー率、スロットル率。
- **No X-Ray or Sentry**: CloudWatch Only 選択。

## 5. Additional Notes

- データ永続化の観点から EBS 方式を使用するが、これは PoC/デモ向けであり本番では RDS/DynamoDB への移行を検討。
- インフラコードは Terraform で Lambda, API Gateway, IAM, EBS Volume, CloudWatch Alarm を定義。

---

*このドキュメントは Infrastructure Design の一部であり、Deployment Architecture と併せてレビューされるべきです。*
