# Backend Service - Functional Design Plan

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: CONSTRUCTION - Functional Design  
**対象ユニット**: Backend Service  
**作成日**: 2026-03-01

---

## 実行チェックリスト

- [x] Unit of Work から Backend Service の責務・境界を再確認
- [x] Story Map から Backend が担当する FR/NFR を抽出
- [x] ドメインエンティティ（Bookmark, UserContext, Versioning）を定義
- [x] ビジネスロジックモデル（CRUD, 検索, 競合制御）を定義
- [x] バリデーションと業務ルールを定義
- [x] エラーシナリオと戻り値ポリシーを定義
- [x] Functional Design 成果物を生成
  - [x] `aidlc-docs/construction/backend-service/functional-design/business-logic-model.md`
  - [x] `aidlc-docs/construction/backend-service/functional-design/business-rules.md`
  - [x] `aidlc-docs/construction/backend-service/functional-design/domain-entities.md`

---

## 事前確認（既知の前提）

- アーキテクチャ: Frontend / Backend / Auth の 3ユニット
- 通信方式: REST API
- Backend 実行基盤: Lambda + API Gateway
- データ戦略: SQLite（＋S3バックアップ）
- 並行制御: Optimistic Locking（version）

---

## 仕様確認質問（回答必須）

### Q1. データ所有権
Bookmark の所有者をどの単位で管理しますか？
A. Cognito `sub` をそのまま `ownerId` として保存  
B. 独自 `userId` テーブルを持ち、Cognito `sub` とマッピング  
C. いったん所有者管理は簡略化（PoC）

[Answer]: B

### Q2. URL 正規化ルール
URL の重複判定に使う正規化ルールを選択してください。  
A. `https/http` を区別しない + 末尾 `/` 無視 + クエリ保持  
B. `https/http` 区別 + 末尾 `/` 無視 + クエリ保持  
C. 完全一致のみ（正規化なし）

[Answer]: C

### Q3. 重複登録の業務ルール
同一ユーザーが同一URLを登録済みの場合の挙動。  
A. 409 を返して新規作成を拒否  
B. 既存レコードを更新（upsert）  
C. 重複を許可

[Answer]: B

### Q4. 削除モデル
削除方式を選択してください。  
A. 物理削除（DELETE）  
B. 論理削除（`deletedAt`）  
C. 論理削除 + 一定期間後に物理削除

[Answer]: A

### Q5. 検索仕様
検索対象フィールドを選択してください。  
A. title のみ  
B. title + url  
C. title + url + tags

[Answer]: C

### Q6. 検索の一致方式
検索文字列の一致方式を選択してください。  
A. 部分一致（LIKE `%query%`）  
B. 前方一致（LIKE `query%`）  
C. 完全一致

[Answer]: A

### Q7. 競合時の更新ルール
version 不一致（409）時の方針。  
A. 常に 409 を返し、クライアント再取得を必須化  
B. サーバー側で自動マージを試行し、失敗時のみ 409  
C. 最終更新優先（409 なし）

[Answer]: A

### Q8. API エラー形式
エラー応答の標準フォーマット方針。  
A. `{ error, message, details, timestamp }` を必須化  
B. `{ code, message }` の最小構成  
C. エンドポイントごとに任意

[Answer]: A

### Q9. ページング既定値
一覧取得時のページング既定値。  
A. `limit=20` / 最大100  
B. `limit=50` / 最大200  
C. ページングなし（全件）

[Answer]: B

### Q10. 監査ログ範囲
Backend の監査ログで記録する範囲。  
A. 変更系のみ（create/update/delete）  
B. 読み取り含む全API  
C. エラー時のみ

[Answer]: B

---

## 回答後に実施すること

- 回答の曖昧性チェック（不足・矛盾があれば追質問）
- Backend Service の Functional Design 成果物3点を生成
- レビュー依頼（Request Changes / Continue to Next Stage）
