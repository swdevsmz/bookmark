# Backend Service - Domain Entities

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01

---

## 1. ドメインモデル概要

Backend の主要エンティティは次の3系統で構成する。

1. ユーザー識別系: `UserIdentityMap`
2. 業務データ系: `Bookmark`
3. 監査系: `AuditLog`

---

## 2. エンティティ定義

## 2.1 UserIdentityMap

### 目的
Cognito `sub` と内部 `userId` の対応付けを管理する。

### 属性
- `userId: string`（PK、UUID）
- `cognitoSub: string`（UNIQUE、必須）
- `createdAt: datetime`（必須）
- `updatedAt: datetime`（必須）

### 制約
- `cognitoSub` は一意
- `userId` は不変

---

## 2.2 Bookmark

### 目的
ユーザーが管理するブックマークを保持する。

### 属性
- `id: string`（PK、UUID）
- `ownerId: string`（FK -> UserIdentityMap.userId）
- `title: string`（必須、1〜200文字）
- `url: string`（必須、完全一致判定対象）
- `description: string | null`
- `tags: string | null`（内部表現はカンマ区切り）
- `version: integer`（必須、初期値1）
- `createdAt: datetime`（必須）
- `updatedAt: datetime`（必須）

### 制約
- UNIQUE(`ownerId`, `url`)  ※ upsertキー
- `version >= 1`

---

## 2.3 AuditLog

### 目的
全API呼び出しの追跡と監査を行う。

### 属性
- `id: string`（PK）
- `timestamp: datetime`（必須, UTC）
- `requestId: string`（必須）
- `userId: string | null`
- `action: string`（LIST/GET/CREATE/UPDATE/DELETE）
- `resourceId: string | null`
- `statusCode: integer`（必須）
- `errorCode: string | null`
- `meta: string | null`（JSON文字列）

---

## 3. API DTO

## 3.1 BookmarkInput

```json
{
  "title": "string",
  "url": "string",
  "description": "string (optional)",
  "tags": ["string"]
}
```

## 3.2 BookmarkResponse

```json
{
  "id": "uuid",
  "ownerId": "uuid",
  "title": "string",
  "url": "string",
  "description": "string|null",
  "tags": ["string"],
  "version": 3,
  "createdAt": "2026-03-01T00:00:00Z",
  "updatedAt": "2026-03-01T00:00:00Z"
}
```

## 3.3 ErrorResponse

```json
{
  "error": "CONFLICT",
  "message": "バージョンが競合しています。",
  "details": {
    "currentVersion": 4,
    "requestedVersion": 3
  },
  "timestamp": "2026-03-01T00:00:00Z"
}
```

---

## 4. 関係モデル

```text
UserIdentityMap (1) ---- (N) Bookmark
UserIdentityMap (1) ---- (N) AuditLog
Bookmark        (1) ---- (N) AuditLog [resourceId参照]
```

---

## 5. 状態遷移

## 5.1 Bookmark.version

- CREATE: `version = 1`
- UPDATE成功: `version = version + 1`
- UPDATE競合: 変更なし（409）
- DELETE: レコード物理削除

## 5.2 Bookmarkライフサイクル

```text
[Created] -> [Updated]* -> [Deleted]
```

---

## 6. SQLite テーブル案

```sql
CREATE TABLE user_identity_map (
  user_id TEXT PRIMARY KEY,
  cognito_sub TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE bookmarks (
  id TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  description TEXT,
  tags TEXT,
  version INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (owner_id) REFERENCES user_identity_map(user_id),
  UNIQUE (owner_id, url)
);

CREATE TABLE audit_logs (
  id TEXT PRIMARY KEY,
  timestamp TEXT NOT NULL,
  request_id TEXT NOT NULL,
  user_id TEXT,
  action TEXT NOT NULL,
  resource_id TEXT,
  status_code INTEGER NOT NULL,
  error_code TEXT,
  meta TEXT
);
```

---
