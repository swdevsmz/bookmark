# Unit of Work 依存関係

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Units Generation  
**作成日**: 2026-03-01

---

## 依存関係マトリックス

各ユニットが他のユニットに依存しているかを示しています。"→" は依存関係を表します。

```
                    Frontend    Backend    Auth
──────────────────────────────────────────────
Frontend              -          →         →
Backend               -          -         ←
Auth                  -          ←         -
```

**説明**:
- **Frontend → Backend**: API呼び出し（CRUD操作）
- **Frontend → Auth**: API呼び出し（ログイン、MFA検証）
- **Backend ← Auth**: トークン検証（内部呼び出し）
- **Backend ↔ Frontend**: REST API (JSON)
- **Auth ↔ Frontend**: REST API (JSON)

---

## 詳細な依存関係フロー

### 1. ユーザー認証フロー

```
Frontend (AuthFeature)
    ↓
Auth Service API
    ↓ (cognito.initiateAuth)
AWS Cognito
    ↓
Frontend (MFA challenge handler)
    ↓
Auth Service API
    ↓ (cognito.respondToAuthChallenge)
AWS Cognito
    ↓
Frontend (store tokens + navigate)
```

**依存関係**: Frontend → Auth Service

---

### 2. ブックマーク取得フロー

```
Frontend (HomePage)
    ↓ (BookmarkManagementFeature.getBookmarks())
Backend API
    ↓ (verify JWT token)
Auth Service (internal call)
    ↓
AWS Cognito (validate token)
    ↓
Backend Service
    ↓ (query database)
SQLite / RDS
    ↓
Frontend (render list)
```

**依存関係**: Frontend → Backend → Auth Service

---

### 3. ブックマーク作成フロー

```
Frontend (CreateBookmarkPage)
    ↓ (validate input)
Validation Service
    ↓
Frontend (BookmarkManagementFeature.createBookmark())
    ↓
Backend API
    ↓ (verify JWT)
Auth Service
    ↓
Backend (validate + insert)
    ↓
Database
    ↓
Frontend (update state + navigate)
```

**依存関係**: Frontend → Backend → Auth Service

---

### 4. ブックマーク更新フロー（並行制御）

```
Frontend (EditBookmarkPage)
    ↓ (load current version)
Backend API
    ↓ (return with version field)
Frontend (optimistic update)
    ↓ (submit updated bookmark with version)
Backend API
    ↓ (check version match)
Backend Service
    ├─ if version matches: update + increment version
    └─ if version mismatch: return conflict error
    ↓
Frontend (show success or conflict dialog)
```

**依存関係**: Frontend → Backend (version-aware update)

---

## API コントラクト

### Backend Service API

**Base URL**: `/api` (API Gateway)

#### 認証ヘッダ (全エンドポイント)
```
Authorization: Bearer <JWT_TOKEN>
```

#### ブックマーク CRUD

```
GET /api/bookmarks
  Query: ?search=<query>&limit=20&offset=0
  Response: { bookmarks: Bookmark[], total: number }

GET /api/bookmarks/{id}
  Response: Bookmark

POST /api/bookmarks
  Body: BookmarkInput
  Response: Bookmark (with id, version, createdAt, updatedAt)

PUT /api/bookmarks/{id}
  Body: { ...BookmarkInput, version: number }
  Response: Bookmark (updated with new version)
  Error 409: { error: "CONFLICT", message: "Version mismatch" }

DELETE /api/bookmarks/{id}
  Query: ?version=<current_version>
  Response: { success: true }
  Error 409: { error: "CONFLICT" }
```

### Authentication Service API

**Base URL**: `/auth` (API Gateway)

#### ログイン
```
POST /auth/login
  Body: { username: string, password: string }
  Response: AuthResult {
    challengeName?: 'SOFTWARE_TOKEN_MFA' | 'SMS_MFA' | 'EMAIL_MFA',
    session?: string,
    challengeParameters?: {...}
  }
```

#### MFA チャレンジ応答
```
POST /auth/mfa-challenge
  Body: { 
    session: string,
    challengeResponse: string  // MFA code
  }
  Response: SessionTokens {
    idToken: string,
    accessToken: string,
    refreshToken: string,
    expiresIn: number
  }
```

#### セッションリフレッシュ
```
POST /auth/refresh-session
  Body: { refreshToken: string }
  Response: SessionTokens (新しいトークン)
```

#### ログアウト
```
POST /auth/logout
  Body: { refreshToken: string }
  Response: { success: true }
```

#### パスワードリセット
```
POST /auth/forgot-password
  Body: { email: string }
  Response: { success: true }

POST /auth/confirm-password-reset
  Body: { email: string, confirmationCode: string, newPassword: string }
  Response: { success: true }
```

---

## デプロイ依存関係

### デプロイ順序（推奨）

1. **Auth Service** (独立)
   - AWS Cognito との連携
   - スタンドアロンデプロイ可能

2. **Backend Service** (Auth に依存)
   - Auth Service の API エンドポイント情報が必要
   - API Gateway エンドポイントを Frontend に提供

3. **Frontend Service** (Backend + Auth に依存)
   - Backend API エンドポイント
   - Auth Service エンドポイント
   - CloudFront ディストリビューション

### 各環境での初期化順序

**開発環境**:
```bash
1. npm install (root)
2. cd packages/auth && npm install && npm run dev
3. cd packages/backend && npm install && npm run dev
4. cd packages/frontend && npm install && npm run dev
```

**本番環境**:
```
1. Deploy Auth Service (Lambda)
2. Deploy Backend Service (Lambda + API Gateway)
3. Deploy Frontend Service (S3 + CloudFront)
```

---

## 通信パターン

### パターン 1: フロントエンド → バックエンド

```typescript
// Frontend (service layer)
async createBookmark(input: BookmarkInput): Promise<Bookmark> {
  const token = getAuthToken();
  const response = await fetch('/api/bookmarks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(input)
  });
  
  if (!response.ok) {
    // Error handling via Error Handling Service
    throw new APIError(response.status, await response.json());
  }
  
  return response.json();
}
```

### パターン 2: バックエンド → Auth 検証

```typescript
// Backend (Lambda handler)
async handleGetBookmarks(event, context) {
  const token = extractToken(event.headers);
  
  // Auth Service に検証依頼
  const validationResult = await authService.validateToken(token);
  
  if (!validationResult.valid) {
    return {
      statusCode: 401,
      body: JSON.stringify({ error: 'UNAUTHORIZED' })
    };
  }
  
  const userId = validationResult.userId;
  // ... CRUD operation
}
```

### パターン 3: 共有エラーハンドリング

```typescript
// Error response format (all services)
interface ErrorResponse {
  error: string;        // ERROR_CODE
  message: string;      // 日本語/英語メッセージ
  details?: any;        // 詳細情報
  timestamp: string;    // ISO 8601
}

// 例
{
  "error": "CONFLICT",
  "message": "バージョンが競合しています。最新のデータで再度試してください。",
  "details": {
    "current_version": 5,
    "provided_version": 3
  },
  "timestamp": "2026-03-01T10:30:00Z"
}
```

---

## まとめ

```
┌─────────────────────────────────────────────────┐
│         Frontend Service (SPA)                  │
│      S3 + CloudFront + Lambda@Edge              │
└──────────┬──────────────────────┬───────────────┘
           │                      │
           │ REST API             │ REST API
           ↓                      ↓
┌──────────────────────┐  ┌──────────────────────┐
│  Backend Service     │  │  Auth Service        │
│  (Lambda + APIGW)    │  │  (Lambda + APIGW)    │
│  SQLite + S3         │  │  AWS Cognito         │
└──────────────────────┘  └──────────────────────┘
           │
           └──→ Token Validation (internal)
```

**重要**: 各ユニットは REST API 経由で通信し、共有データベースはありません。各サービスのデータは独立しています。

---
