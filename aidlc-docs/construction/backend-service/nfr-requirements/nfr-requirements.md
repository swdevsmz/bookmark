# Backend Service - NFR Requirements

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01  
**ステージ**: Construction - NFR Requirements

---

## Executive Summary

Backend Service は、小規模なスタートアップ環境を想定した AWS Serverless アーキテクチャで動作します。初期段階では少数ユーザー（10人、1年後20人）で、スケーラビリティよりもコスト効率性を優先します。

---

## 1. Scalability Requirements

### 1.1 User Scale
- **初期ユーザー数**: 10人
- **1年後予測**: 20人
- **ピーク時同時アクティブユーザー**: 5人程度
- **平均ブックマーク数/ユーザー**: 100個

### 1.2 Data Growth
- **初期総ブックマーク数**: 約 1,000個（10人 × 100個）
- **1年後予測**: 約 2,000個（20人 × 100個）
- **SQLite データベースサイズ**: 初期段階で数 MB 程度（ユーザースケールが小さい）

### 1.3 Scalability Strategy
- **水平スケーリング**: Lambda のデフォルト同時実行数 1,000 で十分対応可能
- **ボトルネック**: SQLite /tmp 保存（Lambda インスタンス単位）は、同時実行 5 人程度では問題なし
- **今後の拡張**: ユーザー数が 100 人を超える場合、RDS MySQL への移行を検討

---

## 2. Performance Requirements

### 2.1 API Response Time Targets

| 操作 | 目標レスポンスタイム | 許容 Cold Start |
|-----|------------------|-------------|
| SEARCH（検索） | **1秒以内** | 含まない |
| LIST（一覧取得） | 1秒以上可 | 許容 |
| GET（詳細取得） | 1秒以上可 | 許容 |
| CREATE（新規作成） | 1秒以上可 | 許容 |
| UPDATE（更新） | 1秒以上可 | 許容 |
| DELETE（削除） | 1秒以上可 | 許容 |

### 2.2 Cold Start Tolerance
- Lambda Cold Start による遅延は許容
- Provisioned Concurrency は不要
- 初期起動時は数秒かかることを想定（ユーザーに通知）

### 2.3 Performance Optimization
- **SEARCH 最適化**: 部分一致検索（LIKE）に対する簡易インデックス戦略
- **キャッシング**: クライアント側キャッシング推奨（ブックマーク変更頻度が低い）
- **ページング**: LIST 操作は `limit=50` デフォルト、最大 200 で制限

---

## 3. Availability Requirements

### 3.1 Uptime Target
- **目標アップタイム**: 特に指定なし（小規模 PoC 的運用）
- **許容ダウンタイム**: あり（運用コスト最小化優先）

### 3.2 Data Persistence
- **SQLite 保存位置**: Lambda `/tmp` ディレクトリ
- **永続化戦略**: Lambda デプロイ時に初期データベースをバンドル、実行時は `/tmp` で読み書き
- **バックアップ戦略**: 初期段階では不要

### 3.3 Disaster Recovery
- **RTO/RPO**: 特に要件なし
- **復旧戦略**: Lambda 再デプロイで初期状態を復旧可能

---

## 4. Security Requirements

### 4.1 Authentication & Authorization
- **認証**: JWT + AWS Cognito
- **ユーザー識別**: Cognito `sub` → 内部 `userId` マッピング
- **認可**: `ownerId = userId` による データ隔離

### 4.2 Data Protection
- **転送時暗号化**: TLS 1.2 以上（AWS デフォルト）
- **保存時暗号化**: 不要（コスト最適化）
- **監査ログ**: 不要

### 4.3 API Security
- **レート制限**: 実装必須
  - 1ユーザーあたりの上限設定予定
  - API Gateway Throttling で実装予定
- **JWT ローテーション**: AWS Cognito 自動管理

---

## 5. Reliability Requirements

### 5.1 Error Handling
- **Upsert 失敗時**: 自動リトライ（最大3回）
- **競合エラー（409）**: クライアント側でユーザーに通知
- **データベース接続エラー**: 標準的なエラーレスポンス（特別な対応なし）

### 5.2 Timeout Configuration
- **API リクエストタイムアウト**: 30秒
- **データベースクエリタイムアウト**: 5秒
- **SQLite ロック許容時間**: 5秒

### 5.3 Retry Strategy
- **リトライ対象**: Upsert（重複実行の可能性）
- **最大リトライ回数**: 3回
- **リトライ間隔**: Exponential backoff（初期値 100ms、最大値 1秒）

---

## 6. Operational Requirements

### 6.1 Logging Strategy
- **CloudWatch ログレベル**: INFO 以上
- **ログ保持期間**: 30日間
- **監査ログ**: 不要

### 6.2 Monitoring & Alerting
- **メトリクス収集対象**:
  - API レスポンスタイム
  - エラー率
  - Lambda 実行時間
- **アラート閾値**:
  - エラー率が 5% を超えた場合
  - レスポンスタイムが 1 秒を超えた場合（SEARCH 操作）

### 6.3 Dashboard
- CloudWatch ダッシュボードで以下を可視化：
  - API ごとのレスポンスタイム分布
  - エラー率推移
  - Lambda 同時実行数

---

## 7. Maintainability Requirements

### 7.1 Code Quality
- TypeScript による型安全性確保
- 業務ロジックと API ロジックの分離
- ユニットテストカバレッジ: 70% 以上

### 7.2 Documentation
- API エンドポイント仕様書（OpenAPI）
- ビジネスロジック仕様書（ドメインモデル中心）
- 運用マニュアル（デプロイ、ロギング、アラート対応）

---

## Summary: NFR Trade-offs

| 項目 | 選択 | 理由 |
|-----|-----|------|
| Scalability | 小規模優先 | 初期ユーザー数が少ない |
| Performance | 検索系に注力 | ユースケース上、検索の QoE が重要 |
| Availability | 可用性よりコスト | PoC 段階の為、複雑な HA 構成は不要 |
| Security | 基本的な対応 | Cognito による認証で十分 |
| Cost | 最優先 | AWS 無料枠活用、サーバーレス選択 |

---
