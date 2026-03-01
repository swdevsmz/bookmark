# Backend Service - NFR Requirements Plan

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01  
**ステージ**: Construction - NFR Requirements (Planning Phase)

---

## Purpose

このプランは Backend Service の非機能要件（NFR）を定義し、パフォーマンス・スケーラビリティ・セキュリティ・信頼性に関わる重要な設計判断を確定するためのものです。

---

## NFR Assessment Checklist

- [ ] **スケーラビリティ要件** - ユーザー数・ブックマーク数の成長予測を確認
- [ ] **パフォーマンス要件** - API レスポンスタイム・スループットを定義
- [ ] **可用性要件** - アップタイム目標・ディザスタリカバリ戦略を確認
- [ ] **セキュリティ要件** - 暗号化・アクセス制御・監査要件を確認
- [ ] **信頼性要件** - エラーハンドリング・リトライ戦略を定義
- [ ] **運用要件** - ロギング・モニタリング・アラート設定を確認
- [ ] **テック決定** - Lambda・SQLite・API Gateway の制約と最適化を確認

---

## Context-Specific Questions for NFR Requirements

### Q1: Expected User Scale and Growth

現在設計では、Lambda（Cold Start考慮）+ SQLite（ローカルストレージ）という構成です。

- **想定初期ユーザー数**: 何人を想定していますか？
- **1年後の予測ユーザー数**: ?
- **1ユーザーあたりの平均ブックマーク数**: 予想値は？
- **ピーク時の同時アクティブユーザー数**: ?

[Answer]: 10人、１年後20人、1ユーザーあたり平均100ブックマーク、ピーク時同時アクティブユーザー数5人程度を想定しています。

---

### Q2: Performance Requirements for API Operations

Backend API は REST で複数のエンドポイント（LIST/GET/CREATE/UPDATE/DELETE/SEARCH）を提供します。

- **List 取得時の目標レスポンスタイム**: 例：200ms、500ms、1秒？
- **Create/Update 操作の目標レスポンスタイム**: ?
- **Search 操作の目標レスポンスタイム**: キーワード検索が複数ユーザーから同時実行される場合の許容時間は？
- **1回のリクエストあたりの許容 Cold Start 時間**: Lambda の Cold Start を許容できますか？それとも Provisioned Concurrency が必要ですか？

[Answer]: ColdStartは許容、レスポンスタイムは１秒

---

### Q3: Availability and Disaster Recovery Strategy

Lambda + SQLite という構成でのデータ永続性とバックアップについて確認します。

- **目標アップタイム**: 99.9%（月間ダウンタイム ~43分）、99.99%（月間ダウンタイム ~4分）、またはそれ以上？
- **SQLite データベースファイルの配置**: Lambda 内ローカルストレージ、Lambda Layers、AWS EFS、S3 のいずれを想定していますか？
- **バックアップ頻度**: 毎日、毎時間、リアルタイムレプリケーション？
- **ディザスタリカバリ RTO/RPO**: 復旧目標時間（RTO）と許容可能なデータ喪失量（RPO）は？

[Answer]: ダウンタイムはいくらあってもいい。SQLiteはLambdaデプロイ時に消えない場所、かつ無料枠。バックアップはいらないDR対策もいらない

---

### Q4: Security and Data Protection

JWT + Cognito という認証構成でのセキュリティ要件を確認します。

- **データ暗号化**: 通信時（TLS）は前提として、保存時（At Rest）の暗号化は必要ですか？
- **JWT キーローテーション戦略**: Cognito のキーローテーション対応は自動でよいか、手動管理が必要か？
- **レート制限**: API呼び出しのレート制限（例：1ユーザーあたり 100req/分）は必要ですか？
- **監査ログの保護**: 監査ログ自体を暗号化・改ざん防止が必要ですか？

[Answer]: 保存時の暗号化はいらない。ローテーションは自動。レート制限はいる。監査ログはいらない

---

### Q5: Error Handling and Reliability

Database 競合制御（楽観的ロック）とエラーハンドリングの戦略を確認します。

- **Upsert 失敗時の再試行**: 重複実行による失敗時に自動リトライが必要ですか？最大リトライ回数は？
- **競合 (409) 時の クライアント対応**: クライアント側で自動リトライするか、ユーザーに通知するか？
- **データベース接続エラー**: SQLite へのアクセス失敗時の処理（サーキットブレーカー、フォールバック等）は必要ですか？
- **タイムアウト設定**: API リクエストタイムアウト、データベースクエリタイムアウトの設定値は？

[Answer]: リトライは必要、最大3回。クライアントには通知。接続エラー時の特別な処理はいらない。タイムアウトはAPIリクエストが30秒、データベースクエリが5秒程度を想定しています。

---

### Q6: Monitoring, Logging and Operations

運用時のロギング・モニタリング・アラート要件を確認します。

- **CloudWatch ログレベル**: DEBUG、INFO、WARNING のいずれを保存対象にしますか？
- **メトリクス収集**: API レスポンスタイム、エラー率、Lambda 実行時間等をメトリクスとして集約が必要ですか？
- **アラート閾値**: 何%のエラー率でアラート、何秒以上のレスポンスタイムでアラートを実行しますか？
- **ログ保持期間**: CloudWatch ログス、監査ログの保持期間は？

[Answer]: CloudWatch ログは30日間保持、監査ログは不要。メトリクスはAPIレスポンスタイムとエラー率を収集。アラートはエラー率が5%を超えた場合、レスポンスタイムが1秒を超えた場合に実行します。

---

### Q7: Lambda Cold Start and Concurrency Management

Lambda の冷起動とコンカレンシー管理について確認します。

- **Provisioned Concurrency**: コールドスタート回避のために、事前にウォームアップ状態を維持する必要がありますか？
- **Lambda 同時実行数上限**: AWS デフォルト 1000 で十分ですか、それとも増加が必要ですか？
- **Lambda タイムアウト設定**: デフォルト 3秒で十分ですか、それとも 15秒、30秒が必要ですか？
- **Lambda メモリ割り当て**: 128MB、256MB、512MB のいずれを標準構成にしますか？

[Answer]: Cold Startは許容、同時実行数上限はデフォルトで十分、タイムアウトは10秒、メモリ割り当ては128MBを標準構成とします。

---

### Q8: Database Optimization and SQLite Configuration

SQLite をデータストアとして使用する際の最適化について確認します。

- **Connection Pooling**: Lambda 内で複数リクエストを処理する場合、SQLite コネクション プールを実装が必要ですか？
- **Query Optimization**: LIKE クエリ（検索機能）に対するインデックス戦略（Full Text Search vs Simple LIKE）は？
- **Concurrent Write Handling**: 複数の Lambda インスタンスから同時に SQLite に書き込むことが予想されますか？その場合の競合対策は？
- **Database Locks**: SQLite の排他的ロック（EXCLUSIVE）の許容時間は何秒が限度ですか？

[Answer]: コネクションプールは不要、インデックスはSimple LIKEで十分、同時書き込みは想定されないため特別な対策は不要、ロックの許容時間は5秒程度を想定しています。

---

## Artifact Generation (Once Answers Received)

回答後、以下2つのアーティファクトを生成します：

1. **nfr-requirements.md** - NFR 要件の詳細文書（スケーラビリティ、パフォーマンス等）
2. **tech-stack-decisions.md** - テック決定と設定値（Lambda メモリ、タイムアウト、インデックス等）

---

## Next Steps

1. すべての [Answer]: タグに回答を入力してください
2. 回答が完了したら、AI が検証・質問を追加する可能性があります
3. 最終承認後、NFR アーティファクトを生成します
4. その後、NFR Design ステージに進みます

