# Backend Service - Business Rules

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01

---

## 1. 認証・認可ルール

- **BR-AUTH-001**: すべての API は有効な JWT を要求する。
- **BR-AUTH-002**: JWT から取得した Cognito `sub` は内部 `userId` にマッピングされる。
- **BR-AUTH-003**: リソース操作は `ownerId = userId` のデータのみ許可する。
- **BR-AUTH-004**: 認証失敗時は `401` を返す。
- **BR-AUTH-005**: 認可失敗（他人データ）は `403` を返す。

---

## 2. 入力検証ルール

- **BR-VAL-001**: `title` は必須、前後空白除去後 1〜200 文字。
- **BR-VAL-002**: `url` は必須、URI 形式であること。
- **BR-VAL-003**: `tags` は任意、配列またはカンマ区切り文字列を許可し、内部保存時は正規化された文字列に統一する。
- **BR-VAL-004**: 不正入力時は `400` を返し、`details` にフィールド別エラーを含める。

---

## 3. URL・重複ルール

- **BR-URL-001**: URL は正規化しない（完全一致比較）。
- **BR-URL-002**: 重複判定キーは `(ownerId, url)`。
- **BR-URL-003**: 既存URLがある場合、`POST` は新規作成せず **upsert（更新）** とする。
- **BR-URL-004**: upsert 時は `updatedAt` 更新、`version` を 1 増分する。

---

## 4. CRUD ルール

- **BR-CRUD-001**: `GET /bookmarks` は `updatedAt DESC` で返す。
- **BR-CRUD-002**: `GET /bookmarks/{id}` で対象が無ければ `404`。
- **BR-CRUD-003**: `DELETE /bookmarks/{id}` は **物理削除** を行う。
- **BR-CRUD-004**: 物理削除後の再取得は `404` を返す。

---

## 5. 検索・ページングルール

- **BR-SEARCH-001**: 検索対象は `title`, `url`, `tags`。
- **BR-SEARCH-002**: 一致方式は部分一致（`LIKE %query%`）。
- **BR-SEARCH-003**: `limit` 既定値は `50`。
- **BR-SEARCH-004**: `limit` 上限は `200`。超過時は `200` に丸める。
- **BR-SEARCH-005**: `offset` 既定値は `0`。

---

## 6. 競合制御ルール

- **BR-CONF-001**: `PUT /bookmarks/{id}` は `version` 必須。
- **BR-CONF-002**: 要求 `version` とDB `version` が不一致なら `409 CONFLICT`。
- **BR-CONF-003**: 競合時にサーバー側自動マージは行わない。
- **BR-CONF-004**: 競合レスポンス `details` に `currentVersion` と `requestedVersion` を返す。

---

## 7. エラーレスポンスルール

- **BR-ERR-001**: エラー形式は `{ error, message, details, timestamp }` に統一。
- **BR-ERR-002**: `error` は機械判読可能な固定コード。
- **BR-ERR-003**: `message` は利用者向け文言。
- **BR-ERR-004**: `timestamp` は ISO 8601（UTC）。

推奨エラーコード:
- `UNAUTHORIZED` (401)
- `FORBIDDEN` (403)
- `NOT_FOUND` (404)
- `VALIDATION_ERROR` (400)
- `CONFLICT` (409)
- `INTERNAL_ERROR` (500)

---

## 8. 監査ログルール

- **BR-AUD-001**: 全API（READ/WRITE）を監査対象とする。
- **BR-AUD-002**: 監査ログは少なくとも `timestamp, requestId, userId, action, statusCode` を含む。
- **BR-AUD-003**: エラー時は `error` コードを記録する。
- **BR-AUD-004**: 監査ログ書き込み失敗は本処理結果を変更しない（非同期・ベストエフォート）。

---
