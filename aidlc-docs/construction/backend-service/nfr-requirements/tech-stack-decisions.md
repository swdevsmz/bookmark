# Backend Service - Tech Stack Decisions

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01  
**ステージ**: Construction - NFR Design (準備段階)

---

## Executive Summary

Backend Service の技術スタックは **AWS Serverless ファースト** を基本にしながら、マイクロな初期ユーザー数（10→20人）と運用コストを最優先に決定されています。

---

## 1. Compute - AWS Lambda

### 選択理由
- サーバー管理なし（インフラストラクチャコスト削減）
- AWS 無料枠対応（月 100 万リクエスト無料）
- Cognito との統合が容易

### Configuration

| 項目 | 値 | 根拠 |
|-----|-----|------|
| **メモリ割り当て** | 128 MB | 小規模 API サーバー。JSON 処理のみで十分 |
| **タイムアウト** | 10秒 | API リクエストタイムアウト 30秒、DB クエリ 5秒での余裕 |
| **同時実行数** | デフォルト 1,000 | 同時アクティブユーザー 5 人で十分 |
| **Provisioned Concurrency** | なし | Cold Start 許容 |
| **環境変数** | DB_PATH, COGNITO_REGION | 後述の Infrastructure Design で具体化 |

### Cold Start Consideration
- **許容**: Yes（初期ユーザー層は UX 許容）
- **対策**: 不要（コスト優先）
- **クライアント側**: API 呼び出し前に「読み込み中」表示

---

## 2. Data Storage - SQLite on Lambda /tmp

### 選択理由
- **無料**: AWS 無料枠でコスト 0
- **シンプル**: SQL で ブックマーク CRUD 実装容易
- **永続化戦略**: Lambda `/tmp` ディレクトリに保存
  - Lambda デプロイ時に初期 DB スキーマをバンドル
  - 実行時は `/tmp` で READ/WRITE
  - Lambda 起動終了後は削除（次回起動時に新規作成）

### Database Schema

```sql
CREATE TABLE users_identity_map (
  user_id TEXT PRIMARY KEY,
  cognito_sub TEXT UNIQUE NOT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL
);

CREATE TABLE bookmarks (
  id TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  description TEXT,
  tags TEXT,  -- カンマ区切り文字列
  version INTEGER NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  FOREIGN KEY (owner_id) REFERENCES users_identity_map (user_id),
  UNIQUE (owner_id, url)
);

CREATE INDEX idx_bookmarks_owner_id ON bookmarks (owner_id);
CREATE INDEX idx_bookmarks_updated_at ON bookmarks (updated_at);

CREATE TABLE audit_logs (
  id TEXT PRIMARY KEY,
  timestamp DATETIME NOT NULL,
  request_id TEXT NOT NULL,
  user_id TEXT,
  action TEXT NOT NULL,  -- LIST, GET, CREATE, UPDATE, DELETE
  resource_id TEXT,
  status_code INTEGER NOT NULL,
  error_code TEXT,
  meta TEXT  -- JSON
);

CREATE INDEX idx_audit_logs_timestamp ON audit_logs (timestamp);
CREATE INDEX idx_audit_logs_user_id ON audit_logs (user_id);
```

### Configuration

| 項目 | 値 | 根拠 |
|-----|-----|------|
| **DB Location** | `/tmp/bookmark.db` | Lambda `/tmp` ディレクトリ（読み書き可能） |
| **Connection Pool** | なし | Lambda リクエスト/実行単位で接続作成・破棄 |
| **Query Timeout** | 5秒 | DB ロック許容時間 5秒以内 |
| **Index Strategy** | 簡易（owner_id, updated_at, timestamp） | LIKE クエリは Full Text Search 不要 |
| **Concurrent Write** | 低（同時ユーザー5人） | 排他的ロック対策は不要 |

### Data Persistence Note
⚠️ **重要**: Lambda `/tmp` は Lambda 実行終了後に削除されます。初期データベースは Lambda デプロイ時にバンドルされたテンプレートから毎回新規作成されます。這は MVP/PoC 向け設定です。

---

## 3. API Gateway - REST API

### 選択理由
- AWS 無料枠対応（月 100 万リクエスト）
- Lambda との直接統合
- 認証・レート制限の AWS ネイティブサポート

### Configuration

| 項目 | 値 | 設定内容 |
|-----|-----|--------|
| **API タイプ** | REST API | HTTP/HTTPS エンドポイント |
| **認証** | Lambda Authorizer | Cognito JWT 検証（カスタム） |
| **レート制限** | API Gateway Throttling | 1ユーザーあたり 1,000req/分（見直し予定） |
| **CORS** | 有効 | フロント（S3+CloudFront）からのリクエスト許可 |
| **ロギング** | CloudWatch | API リクエスト/レスポンス ログ |

### Endpoints

```
POST   /api/bookmarks          - Create (UPSERT)
GET    /api/bookmarks          - List
GET    /api/bookmarks/{id}     - Get detail
PUT    /api/bookmarks/{id}     - Update (version 必須)
DELETE /api/bookmarks/{id}     - Delete
```

---

## 4. Authentication - AWS Cognito

### 選択理由
- 2要素認証（MFA）ネイティブサポート
- AWS マネージドサービス（運用負荷なし）
- JWT トークン生成・ローテーション自動

### Configuration

| 項目 | 値 |
|-----|-----|
| **ユーザープール** | [プロジェクト専用ユーザープール] |
| **MFA 認証方式** | TOTP（Google Authenticator/Authy 対応） |
| **JWT キーローテーション** | 自動（AWS Cognito 管理） |
| **セッション有効期限** | 1時間（デフォルト） |
| **リフレッシュトークン有効期限** | 30日（デフォルト） |

### User Identity Mapping
- Cognito `sub` ← → 内部 `userId` (UUID)
- `UserIdentityMap` テーブル管理・初回アクセス時に自動作成

---

## 5. Logging & Monitoring - AWS CloudWatch

### Configuration

| 項目 | 値 | 説明 |
|-----|-----|------|
| **ログレベル** | INFO | エラーと重要な操作を記録 |
| **ログ保持期間** | 30日間 | CloudWatch Logs ストレージコスト管理 |
| **メトリクス** | 3つ | response_time, error_rate, execution_duration |
| **アラート** | 2つ | error_rate > 5%, response_time > 1000ms（SEARCH） |

### Logging Strategy

```json
{
  "timestamp": "2026-03-01T00:00:00Z",
  "requestId": "uuid",
  "level": "INFO",
  "action": "CREATE",
  "statusCode": 201,
  "responseTime": 450,
  "userId": "uuid"
}
```

### Metrics & Alarms
- **Lambda Invocations**: 総リクエスト数
- **Lambda Duration**: 平均実行時間
- **Lambda Error**: エラー数・エラー率
- **API Response Time**: SEARCH 操作の PERCENTILE 90

---

## 6. Infrastructure as Code - Terraform

### 選択理由
- AWS リソースの宣言型定義
- バージョン管理・再現可能なデプロイ
- チーム協業容易

### Managed Resources

```
- AWS Lambda Function
- API Gateway REST API
- Cognito User Pool
- CloudWatch Log Groups
- IAM Roles & Policies
```

### IaC Organization

```
terraform/
├── variables.tf          # 入力変数定義
├── main.tf              # メインリソース
├── lambda.tf            # Lambda 関連
├── api_gateway.tf       # API Gateway
├── cognito.tf           # Cognito 認証
├── cloudwatch.tf        # ロギング・メトリクス
└── outputs.tf           # 出力値（API エンドポイント等）
```

---

## 7. Runtime & Language

### Language
- **Node.js 18.x** （AWS Lambda デフォルト）
- **TypeScript** （型安全性）

### Dependencies

```json
{
  "aws-sdk": "v3",
  "better-sqlite3": "SQLite ドライバ",
  "jsonwebtoken": "JWT 検証",
  "uuid": "ID 生成"
}
```

### Dependency Management
- npm + `package-lock.json`（再現可能性）
- 定期的なセキュリティアップデート（月1回）

---

## 8. Error Handling & Resilience

### Error Response Format

```json
{
  "error": "ERROR_CODE",        // 機械判読可能
  "message": "ユーザー向けメッセージ",
  "details": { /* 詳細情報 */ },
  "timestamp": "ISO 8601"
}
```

### Retry Strategy
- **自動リトライ対象**: Upsert（重複実行の可能性）
- **最大リトライ回数**: 3回
- **リトライ間隔**: Exponential backoff
  - 初回: 100ms
  - 2回目: 200ms
  - 3回目: 400ms

### Circuit Breaker
- **不要**: データベースが Lambda 内蔵（外部依存性なし）

---

## 9. Security Measures

### Data Protection
- **転送時**: TLS 1.2 以上（AWS API Gateway デフォルト）
- **保存時**: 暗号化なし（コスト最適化）

### Access Control
- **認証**: JWT（Cognito）
- **認可**: リソース所有者チェック（`ownerId == userId`）
- **監査**: 実装不要（小規模PoC）

### Rate Limiting
```
API Gateway Throttling: 1,000 req/ユーザー/分
（見直し対象：ユーザー成長に応じて調整）
```

---

## 10. Deployment & CI/CD

### Deployment Strategy
- **Infrastructure**: Terraform で管理
- **Lambda Code**: AWS CLI または Serverless Framework
- **環境**: main branch から自動デプロイ（GitHub Actions）

### Environments
- **dev**: 開発・テスト環境
- **prod**: 本番環境

---

## 11. Cost Estimation (Monthly)

| サービス | 無料枠 | 予想超過 | 月額費用 |
|---------|-------|--------|--------|
| Lambda | 100万リクエスト | 想定内 | ¥0 |
| API Gateway | 100万リクエスト | 想定内 | ¥0 |
| Cognito | 50,000ユーザー | 想定内 | ¥0 |
| CloudWatch | 5GB ログ | 想定内 | ¥0 |
| **合計** | | | **¥0~数百円** |

💰 **結論**: AWS 無料枠内で運用可能な設計

---

## 12. Future Scalability Path

ユーザー数が 100 人を超える場合の拡張戦略：

### Phase 2 (50-100 users)
- [ ] RDS MySQL に移行（SQLite の並行書き込み制限を回避）
- [ ] ElastiCache Redis 導入（キャッシング）
- [ ] API Gateway Caching の実装

### Phase 3 (100+ users)
- [ ] マイクロサービス化
- [ ] メッセージキュー導入（非同期処理）
- [ ] CDN キャッシュ戦略の強化

---
