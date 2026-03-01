# Backend Service - Business Logic Model

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01

---

## 1. 目的

Backend Service は、認証済みユーザーに対してブックマークの CRUD・検索・競合制御を提供する。

- 実行基盤: Lambda + API Gateway
- 永続化: SQLite
- 認証: Cognito `sub` ↔ 内部 `userId` マッピング
- 一覧既定: `limit=50`（最大 `200`）
- 削除方式: 物理削除
- 重複URL: Upsert

---

## 2. ユースケース別ロジック

### UC-1: 一覧取得 `GET /api/bookmarks`

1. JWT を検証し、Cognito `sub` を取得
2. `UserIdentityMap` で `sub -> userId` を解決（なければ作成）
3. `limit/offset/search` を検証
4. `search` がある場合は `title/url/tags` を部分一致検索（`LIKE %query%`）
5. `ownerId = userId` でフィルタし一覧を返却
6. 監査ログ（READ）を記録

### UC-2: 詳細取得 `GET /api/bookmarks/{id}`

1. JWT 検証 → `userId` 解決
2. `id` の形式を検証
3. `ownerId = userId` 条件で単一取得
4. 見つからない場合は 404
5. 監査ログ（READ）を記録

### UC-3: 新規作成 `POST /api/bookmarks`

1. JWT 検証 → `userId` 解決
2. 入力バリデーション（title/url/tags）
3. URL は正規化せず完全一致で重複判定
4. 同一 `ownerId + url` が存在する場合は **upsert**（既存更新）
5. 未存在なら新規 INSERT
6. 結果を返却（`id/version/createdAt/updatedAt`）
7. 監査ログ（WRITE）を記録

### UC-4: 更新 `PUT /api/bookmarks/{id}`

1. JWT 検証 → `userId` 解決
2. 入力バリデーション
3. 対象レコードを `ownerId + id` で取得
4. 要求 `version` と DB `version` が不一致なら 409
5. 一致時のみ更新、`version += 1`
6. 更新後レコードを返却
7. 監査ログ（WRITE）を記録

### UC-5: 削除 `DELETE /api/bookmarks/{id}`

1. JWT 検証 → `userId` 解決
2. `ownerId + id` 条件で存在確認
3. 存在時は物理削除
4. 存在しない場合は 404
5. 監査ログ（WRITE）を記録

---

## 3. 共通制御フロー

## 3.1 認証・ユーザー解決

```text
Request
  -> JWT verify
  -> get cognitoSub
  -> resolve internal userId
  -> execute use case
```

### 3.2 エラーレスポンス標準

```json
{
  "error": "CONFLICT",
  "message": "バージョンが競合しています。",
  "details": { "currentVersion": 7, "requestedVersion": 6 },
  "timestamp": "2026-03-01T00:00:00Z"
}
```

---

## 4. 検索ロジック

- 対象列: `title`, `url`, `tags`
- 一致方式: 部分一致
- SQLイメージ:

```sql
SELECT *
FROM bookmarks
WHERE owner_id = :ownerId
  AND (
    title LIKE '%' || :query || '%'
    OR url LIKE '%' || :query || '%'
    OR tags LIKE '%' || :query || '%'
  )
ORDER BY updated_at DESC
LIMIT :limit OFFSET :offset;
```

---

## 5. Upsert ロジック

- キー: `(owner_id, url)`
- URL は完全一致のみ（正規化なし）

擬似コード:

```text
existing = findByOwnerAndUrl(ownerId, urlExact)
if existing:
  update existing(title, tags, description, updatedAt, version+1)
  return updated entity
else:
  insert new bookmark(version=1)
  return created entity
```

---

## 6. 競合制御ロジック

擬似コード:

```text
row = findByOwnerAndId(ownerId, id)
if not row: 404
if request.version != row.version:
  return 409 CONFLICT
update ... set version = version + 1 where id=:id and version=:version
return updated row
```

---

## 7. 監査ログロジック

- 対象: **全API（READ/WRITE）**
- 記録項目:
  - timestamp
  - requestId
  - userId
  - action (LIST/GET/CREATE/UPDATE/DELETE)
  - resourceId (任意)
  - statusCode
  - errorCode (任意)

---

## 8. 非機能整合

- NFR-1: JWT 検証を全APIで実施
- NFR-2: 更新時 version 必須
- NFR-3: ページング・検索最適化（`limit=50, max=200`）

---
