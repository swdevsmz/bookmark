# Backend Service - Logical Components

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-02  
**ステージ**: Construction - NFR Design

---

## Executive Summary

このドキュメントは、Backend Service Lambda ハンドラーの論理的なコンポーネント分解を定義します。レイヤードアーキテクチャパターンに基づき、各レイヤー、モジュール、クラスの責任と相互作用を明確にします。

---

## 1. Architecture Overview

### 1.1 4-Layer Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        API Gateway                           │
│              (JWT Authorization, Throttling)                 │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                     HTTP Layer (handler.js)                  │
│  ┌────────────┬────────────┬────────────┬────────────────┐  │
│  │  Request   │  JWT       │  Input     │  Error         │  │
│  │  Parser    │  Verifier  │  Validator │  Handler       │  │
│  └────────────┴────────────┴────────────┴────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                  Service Layer (services/)                   │
│  ┌─────────────────────┬─────────────────────────────────┐  │
│  │  BookmarkService    │  UserIdentityService            │  │
│  │  - CRUD operations  │  - Cognito sub mapping          │  │
│  │  - Search logic     │  - User identity management     │  │
│  │  - Version control  │                                 │  │
│  └─────────────────────┴─────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│               Repository Layer (repositories/)               │
│  ┌──────────────────────┬──────────────────────────────────┐│
│  │  BookmarkRepository  │  UserIdentityRepository          ││
│  │  - SQL queries       │  - Cognito sub → userId mapping  ││
│  │  - Data mapping      │  - User CRUD                     ││
│  │  - Transaction mgmt  │                                  ││
│  └──────────────────────┴──────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                   Data Layer (database/)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SQLite Connection Manager                           │   │
│  │  - /tmp/bookmark.db management                       │   │
│  │  - Connection pooling (single connection per Lambda)│   │
│  │  - Timeout configuration (5s)                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. HTTP Layer Components

### 2.1 Main Handler (handler.js)

**責任**:
- Lambda エントリーポイント
- HTTP リクエスト/レスポンスの処理
- エラーハンドリング
- ルーティング

**インターフェース**:
```javascript
exports.handler = async (event, context) => {
  // AWS Lambda standard handler signature
  // event: API Gateway proxy event
  // context: Lambda context
  // return: API Gateway proxy response
};
```

**実装詳細**:
```javascript
// handler.js
const { verifyJwt } = require('./http/jwt-verifier');
const { validateInput } = require('./http/input-validator');
const { routeRequest } = require('./http/router');
const { handleError } = require('./http/error-handler');

exports.handler = async (event, context) => {
  try {
    // 1. JWT 検証
    const claims = await verifyJwt(event.headers.Authorization);
    const userId = claims.sub; // Cognito sub
    
    // 2. リクエストパース
    const body = event.body ? JSON.parse(event.body) : {};
    const pathParams = event.pathParameters || {};
    const queryParams = event.queryStringParameters || {};
    
    // 3. 早期バリデーション
    await validateInput(event.httpMethod, event.path, body);
    
    // 4. レート制限チェック
    if (!checkRateLimit(userId)) {
      return {
        statusCode: 429,
        body: JSON.stringify({ error: 'Too Many Requests' })
      };
    }
    
    // 5. ルーティング
    const result = await routeRequest(
      event.httpMethod, 
      event.path, 
      { body, pathParams, queryParams, userId }
    );
    
    // 6. レスポンス生成
    return {
      statusCode: 200,
      body: JSON.stringify(result),
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      }
    };
  } catch (error) {
    return handleError(error, context);
  }
};
```

---

### 2.2 JWT Verifier (http/jwt-verifier.js)

**責任**:
- JWT トークンの検証
- Cognito Public Key を使用した署名検証
- トークン有効期限チェック

**インターフェース**:
```javascript
async function verifyJwt(authorizationHeader) {
  // Input: "Bearer <JWT_TOKEN>"
  // Output: { sub, email, ... } (decoded claims)
  // Throws: UnauthorizedError if invalid
}
```

**実装詳細**:
```javascript
// http/jwt-verifier.js
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const client = jwksClient({
  jwksUri: `https://cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}/.well-known/jwks.json`
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    if (err) {
      callback(err);
    } else {
      const signingKey = key.publicKey || key.rsaPublicKey;
      callback(null, signingKey);
    }
  });
}

async function verifyJwt(authorizationHeader) {
  if (!authorizationHeader || !authorizationHeader.startsWith('Bearer ')) {
    throw new UnauthorizedError('Missing or invalid Authorization header');
  }
  
  const token = authorizationHeader.substring(7);
  
  return new Promise((resolve, reject) => {
    jwt.verify(token, getKey, {
      issuer: `https://cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}`,
      algorithms: ['RS256']
    }, (err, decoded) => {
      if (err) {
        reject(new UnauthorizedError('Invalid token'));
      } else {
        resolve(decoded);
      }
    });
  });
}

module.exports = { verifyJwt };
```

---

### 2.3 Input Validator (http/input-validator.js)

**責任**:
- リクエストボディのスキーマバリデーション
- ビジネスルールバリデーション
- XSS対策（入力サニタイゼーション）

**インターフェース**:
```javascript
async function validateInput(method, path, body) {
  // Input: HTTP method, path, request body
  // Output: void (throws ValidationError if invalid)
}
```

**実装詳細**:
```javascript
// http/input-validator.js
const Ajv = require('ajv');
const ajv = new Ajv();

// スキーマ定義
const bookmarkCreateSchema = {
  type: 'object',
  properties: {
    title: { type: 'string', minLength: 1, maxLength: 200 },
    url: { type: 'string', pattern: '^https?://.+' },
    description: { type: 'string', maxLength: 1000 },
    tags: { type: 'string', maxLength: 500 }
  },
  required: ['title', 'url'],
  additionalProperties: false
};

const bookmarkUpdateSchema = {
  type: 'object',
  properties: {
    title: { type: 'string', minLength: 1, maxLength: 200 },
    url: { type: 'string', pattern: '^https?://.+' },
    description: { type: 'string', maxLength: 1000 },
    tags: { type: 'string', maxLength: 500 },
    version: { type: 'integer', minimum: 1 }
  },
  required: ['version'],
  additionalProperties: false
};

const schemas = {
  'POST /bookmarks': bookmarkCreateSchema,
  'PUT /bookmarks/:id': bookmarkUpdateSchema
};

async function validateInput(method, path, body) {
  // ルート正規化
  const routeKey = normalizeRoute(method, path);
  const schema = schemas[routeKey];
  
  if (!schema) {
    // バリデーション不要
    return;
  }
  
  // スキーマバリデーション
  const valid = ajv.validate(schema, body);
  if (!valid) {
    throw new ValidationError(ajv.errors);
  }
  
  // ビジネスルールバリデーション
  if (body.tags) {
    const tags = body.tags.split(',').map(t => t.trim()).filter(t => t);
    if (tags.length > 10) {
      throw new ValidationError('Maximum 10 tags allowed');
    }
  }
  
  // XSS対策（HTMLタグ除去）
  if (body.title) {
    body.title = sanitizeHtml(body.title);
  }
  if (body.description) {
    body.description = sanitizeHtml(body.description);
  }
}

function normalizeRoute(method, path) {
  // /bookmarks/abc123 → /bookmarks/:id
  return path.replace(/\/[a-f0-9-]+$/, '/:id');
}

function sanitizeHtml(input) {
  return input.replace(/<[^>]*>/g, '');
}

module.exports = { validateInput };
```

---

### 2.4 Router (http/router.js)

**責任**:
- HTTP メソッドとパスをサービス層メソッドにマッピング
- パスパラメータの抽出

**インターフェース**:
```javascript
async function routeRequest(method, path, params) {
  // Input: HTTP method, path, { body, pathParams, queryParams, userId }
  // Output: Service layer response
}
```

**実装詳細**:
```javascript
// http/router.js
const { BookmarkService } = require('../services/bookmark-service');
const { UserIdentityService } = require('../services/user-identity-service');

async function routeRequest(method, path, params) {
  const { body, pathParams, queryParams, userId } = params;
  
  // サービスインスタンス取得
  const bookmarkService = await BookmarkService.getInstance();
  const userService = await UserIdentityService.getInstance();
  
  // ユーザーID マッピング（Cognito sub → internal userId）
  const internalUserId = await userService.getOrCreateUserId(userId);
  
  // ルーティング
  if (method === 'GET' && path === '/bookmarks') {
    // 一覧取得
    const query = queryParams.q || '';
    const limit = parseInt(queryParams.limit || '50');
    const offset = parseInt(queryParams.offset || '0');
    return await bookmarkService.listBookmarks(internalUserId, { query, limit, offset });
    
  } else if (method === 'GET' && path.startsWith('/bookmarks/')) {
    // 詳細取得
    const id = pathParams.id;
    return await bookmarkService.getBookmark(id, internalUserId);
    
  } else if (method === 'POST' && path === '/bookmarks') {
    // 新規作成
    return await bookmarkService.createBookmark(internalUserId, body);
    
  } else if (method === 'PUT' && path.startsWith('/bookmarks/')) {
    // 更新
    const id = pathParams.id;
    return await bookmarkService.updateBookmark(id, internalUserId, body, body.version);
    
  } else if (method === 'DELETE' && path.startsWith('/bookmarks/')) {
    // 削除
    const id = pathParams.id;
    return await bookmarkService.deleteBookmark(id, internalUserId);
    
  } else {
    throw new NotFoundError('Route not found');
  }
}

module.exports = { routeRequest };
```

---

### 2.5 Error Handler (http/error-handler.js)

**責任**:
- エラーをHTTPレスポンスに変換
- エラーログ出力
- クライアントにセキュアなエラーメッセージを返す

**インターフェース**:
```javascript
function handleError(error, context) {
  // Input: Error object, Lambda context
  // Output: API Gateway proxy response
}
```

**実装詳細**:
```javascript
// http/error-handler.js
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ValidationError';
    this.statusCode = 400;
  }
}

class UnauthorizedError extends Error {
  constructor(message) {
    super(message);
    this.name = 'UnauthorizedError';
    this.statusCode = 401;
  }
}

class NotFoundError extends Error {
  constructor(message) {
    super(message);
    this.name = 'NotFoundError';
    this.statusCode = 404;
  }
}

class ConflictError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ConflictError';
    this.statusCode = 409;
  }
}

function handleError(error, context) {
  // エラーログ（エラーのみログ出力）
  console.error(JSON.stringify({
    timestamp: new Date().toISOString(),
    requestId: context.requestId,
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack
    }
  }));
  
  // クライアントレスポンス
  const statusCode = error.statusCode || 500;
  const message = statusCode === 500 ? 'Internal Server Error' : error.message;
  
  return {
    statusCode,
    body: JSON.stringify({
      error: error.name,
      message: message
    }),
    headers: {
      'Content-Type': 'application/json'
    }
  };
}

module.exports = {
  handleError,
  ValidationError,
  UnauthorizedError,
  NotFoundError,
  ConflictError
};
```

---

### 2.6 Rate Limiter (http/rate-limiter.js)

**責任**:
- ユーザーごとのリクエストレート制限
- メモリ内マップでレート追跡（Lambda インスタンス単位）

**インターフェース**:
```javascript
function checkRateLimit(userId, limit = 100, windowMs = 60000) {
  // Input: userId, rate limit, time window
  // Output: true (allowed) / false (throttled)
}
```

**実装詳細**:
```javascript
// http/rate-limiter.js
const rateLimitMap = new Map(); // userId -> { count, resetTime }

function checkRateLimit(userId, limit = 100, windowMs = 60000) {
  const now = Date.now();
  const userLimit = rateLimitMap.get(userId);
  
  if (!userLimit || now > userLimit.resetTime) {
    // 新しいウィンドウ開始
    rateLimitMap.set(userId, { count: 1, resetTime: now + windowMs });
    return true;
  }
  
  if (userLimit.count >= limit) {
    // レート制限超過
    return false;
  }
  
  // カウント増加
  userLimit.count++;
  return true;
}

// 定期クリーンアップ（古いエントリ削除）
setInterval(() => {
  const now = Date.now();
  for (const [userId, data] of rateLimitMap.entries()) {
    if (now > data.resetTime) {
      rateLimitMap.delete(userId);
    }
  }
}, 60000); // 1分ごと

module.exports = { checkRateLimit };
```

---

## 3. Service Layer Components

### 3.1 BookmarkService (services/bookmark-service.js)

**責任**:
- ブックマークのCRUD操作
- 検索ロジック
- 楽観的ロック制御
- リトライロジック

**インターフェース**:
```javascript
class BookmarkService {
  async listBookmarks(userId, options)
  async getBookmark(id, userId)
  async createBookmark(userId, data)
  async updateBookmark(id, userId, updates, expectedVersion)
  async deleteBookmark(id, userId)
  async searchBookmarks(userId, query)
}
```

**実装詳細**:
```javascript
// services/bookmark-service.js
const { BookmarkRepository } = require('../repositories/bookmark-repository');
const { executeWithRetry } = require('../utils/retry');

class BookmarkService {
  constructor(bookmarkRepository) {
    this.repo = bookmarkRepository;
  }
  
  static async getInstance() {
    const repo = await BookmarkRepository.getInstance();
    return new BookmarkService(repo);
  }
  
  async listBookmarks(userId, options = {}) {
    const { query, limit = 50, offset = 0 } = options;
    
    if (query) {
      return await this.searchBookmarks(userId, query, limit, offset);
    } else {
      return await this.repo.findByOwnerId(userId, limit, offset);
    }
  }
  
  async getBookmark(id, userId) {
    const bookmark = await this.repo.findById(id, userId);
    if (!bookmark) {
      throw new NotFoundError('Bookmark not found');
    }
    return bookmark;
  }
  
  async createBookmark(userId, data) {
    // ビジネスルールバリデーション
    this.validateBookmarkData(data);
    
    // URL重複チェック（UNIQUE制約でも防止）
    const existing = await this.repo.findByUrl(userId, data.url);
    if (existing) {
      throw new ConflictError('Bookmark with this URL already exists');
    }
    
    // 作成（リトライ付き）
    return await executeWithRetry(async () => {
      return await this.repo.create(userId, data);
    });
  }
  
  async updateBookmark(id, userId, updates, expectedVersion) {
    // ビジネスルールバリデーション
    this.validateBookmarkData(updates);
    
    // 楽観的ロックで更新（リトライ付き）
    return await executeWithRetry(async () => {
      return await this.repo.update(id, userId, updates, expectedVersion);
    });
  }
  
  async deleteBookmark(id, userId) {
    // 削除（リトライ付き）
    return await executeWithRetry(async () => {
      const result = await this.repo.delete(id, userId);
      if (result.changes === 0) {
        throw new NotFoundError('Bookmark not found');
      }
      return { success: true };
    });
  }
  
  async searchBookmarks(userId, query, limit = 50, offset = 0) {
    // 検索（SQLite LIKE）
    return await this.repo.search(userId, query, limit, offset);
  }
  
  validateBookmarkData(data) {
    // タイトル長さ
    if (data.title && (data.title.length < 1 || data.title.length > 200)) {
      throw new ValidationError('Title must be 1-200 characters');
    }
    
    // URL形式
    if (data.url && !data.url.match(/^https?:\/\/.+/)) {
      throw new ValidationError('Invalid URL format');
    }
    
    // 説明文長さ
    if (data.description && data.description.length > 1000) {
      throw new ValidationError('Description must be <= 1000 characters');
    }
    
    // タグ数
    if (data.tags) {
      const tags = data.tags.split(',').map(t => t.trim()).filter(t => t);
      if (tags.length > 10) {
        throw new ValidationError('Maximum 10 tags allowed');
      }
    }
  }
}

module.exports = { BookmarkService };
```

---

### 3.2 UserIdentityService (services/user-identity-service.js)

**責任**:
- Cognito sub → 内部 userId マッピング
- ユーザーアイデンティティ管理

**インターフェース**:
```javascript
class UserIdentityService {
  async getOrCreateUserId(cognitoSub)
  async getUserByCognitoSub(cognitoSub)
}
```

**実装詳細**:
```javascript
// services/user-identity-service.js
const { UserIdentityRepository } = require('../repositories/user-identity-repository');
const { v4: uuidv4 } = require('uuid');

class UserIdentityService {
  constructor(userRepo) {
    this.repo = userRepo;
  }
  
  static async getInstance() {
    const repo = await UserIdentityRepository.getInstance();
    return new UserIdentityService(repo);
  }
  
  async getOrCreateUserId(cognitoSub) {
    // 既存ユーザーを検索
    let user = await this.repo.findByCognitoSub(cognitoSub);
    
    if (!user) {
      // 新規ユーザー作成
      const userId = uuidv4();
      user = await this.repo.create({
        userId,
        cognitoSub,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString()
      });
    }
    
    return user.userId;
  }
  
  async getUserByCognitoSub(cognitoSub) {
    return await this.repo.findByCognitoSub(cognitoSub);
  }
}

module.exports = { UserIdentityService };
```

---

## 4. Repository Layer Components

### 4.1 BookmarkRepository (repositories/bookmark-repository.js)

**責任**:
- ブックマークテーブルのCRUD SQL操作
- Parameterized Queries（SQLインジェクション対策）
- データマッピング

**インターフェース**:
```javascript
class BookmarkRepository {
  async findById(id, userId)
  async findByOwnerId(userId, limit, offset)
  async findByUrl(userId, url)
  async search(userId, query, limit, offset)
  async create(userId, data)
  async update(id, userId, updates, expectedVersion)
  async delete(id, userId)
}
```

**実装詳細**:
```javascript
// repositories/bookmark-repository.js
const { getDatabase } = require('../database/sqlite-connection');
const { v4: uuidv4 } = require('uuid');

class BookmarkRepository {
  constructor(db) {
    this.db = db;
  }
  
  static async getInstance() {
    const db = await getDatabase();
    return new BookmarkRepository(db);
  }
  
  async findById(id, userId) {
    return await this.db.get(
      'SELECT * FROM bookmarks WHERE id = ? AND owner_id = ?',
      [id, userId]
    );
  }
  
  async findByOwnerId(userId, limit = 50, offset = 0) {
    return await this.db.all(
      `SELECT * FROM bookmarks 
       WHERE owner_id = ? 
       ORDER BY updated_at DESC 
       LIMIT ? OFFSET ?`,
      [userId, limit, offset]
    );
  }
  
  async findByUrl(userId, url) {
    return await this.db.get(
      'SELECT * FROM bookmarks WHERE owner_id = ? AND url = ?',
      [userId, url]
    );
  }
  
  async search(userId, query, limit = 50, offset = 0) {
    const pattern = `%${query}%`;
    return await this.db.all(
      `SELECT * FROM bookmarks 
       WHERE owner_id = ? 
         AND (title LIKE ? OR url LIKE ? OR description LIKE ? OR tags LIKE ?)
       ORDER BY updated_at DESC 
       LIMIT ? OFFSET ?`,
      [userId, pattern, pattern, pattern, pattern, limit, offset]
    );
  }
  
  async create(userId, data) {
    const id = uuidv4();
    const now = new Date().toISOString();
    
    await this.db.run(
      `INSERT INTO bookmarks (id, owner_id, title, url, description, tags, version, created_at, updated_at)
       VALUES (?, ?, ?, ?, ?, ?, 1, ?, ?)`,
      [id, userId, data.title, data.url, data.description || null, data.tags || null, now, now]
    );
    
    return await this.findById(id, userId);
  }
  
  async update(id, userId, updates, expectedVersion) {
    const now = new Date().toISOString();
    
    const result = await this.db.run(
      `UPDATE bookmarks 
       SET title = ?, url = ?, description = ?, tags = ?, 
           version = version + 1, updated_at = ? 
       WHERE id = ? AND owner_id = ? AND version = ?`,
      [updates.title, updates.url, updates.description || null, updates.tags || null,
       now, id, userId, expectedVersion]
    );
    
    if (result.changes === 0) {
      // バージョン不一致 or レコード不存在
      const existing = await this.findById(id, userId);
      if (!existing) {
        throw new NotFoundError('Bookmark not found');
      } else {
        throw new ConflictError(`Version mismatch. Expected ${expectedVersion}, current ${existing.version}`);
      }
    }
    
    return await this.findById(id, userId);
  }
  
  async delete(id, userId) {
    return await this.db.run(
      'DELETE FROM bookmarks WHERE id = ? AND owner_id = ?',
      [id, userId]
    );
  }
}

module.exports = { BookmarkRepository };
```

---

### 4.2 UserIdentityRepository (repositories/user-identity-repository.js)

**責任**:
- users_identity_map テーブルのCRUD SQL操作
- Cognito sub マッピング管理

**インターフェース**:
```javascript
class UserIdentityRepository {
  async findByCognitoSub(cognitoSub)
  async create(data)
}
```

**実装詳細**:
```javascript
// repositories/user-identity-repository.js
const { getDatabase } = require('../database/sqlite-connection');

class UserIdentityRepository {
  constructor(db) {
    this.db = db;
  }
  
  static async getInstance() {
    const db = await getDatabase();
    return new UserIdentityRepository(db);
  }
  
  async findByCognitoSub(cognitoSub) {
    return await this.db.get(
      'SELECT * FROM users_identity_map WHERE cognito_sub = ?',
      [cognitoSub]
    );
  }
  
  async create(data) {
    await this.db.run(
      `INSERT INTO users_identity_map (user_id, cognito_sub, created_at, updated_at)
       VALUES (?, ?, ?, ?)`,
      [data.userId, data.cognitoSub, data.createdAt, data.updatedAt]
    );
    
    return await this.findByCognitoSub(data.cognitoSub);
  }
}

module.exports = { UserIdentityRepository };
```

---

## 5. Data Layer Components

### 5.1 SQLite Connection Manager (database/sqlite-connection.js)

**責任**:
- SQLite データベース接続管理
- Lambda /tmp ディレクトリ管理
- タイムアウト設定

**インターフェース**:
```javascript
async function getDatabase() {
  // Output: SQLite database connection
}
```

**実装詳細**:
```javascript
// database/sqlite-connection.js
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');
const fs = require('fs');

let dbConnection = null;

async function getDatabase() {
  if (!dbConnection) {
    const dbPath = '/tmp/bookmark.db';
    
    // データベースファイルが存在しない場合、初期化
    const exists = fs.existsSync(dbPath);
    
    dbConnection = await open({
      filename: dbPath,
      driver: sqlite3.Database
    });
    
    // タイムアウト設定（5秒）
    await dbConnection.run('PRAGMA busy_timeout = 5000;');
    
    if (!exists) {
      // スキーマ初期化
      await initializeSchema(dbConnection);
    }
  }
  
  return dbConnection;
}

async function initializeSchema(db) {
  await db.exec(`
    CREATE TABLE IF NOT EXISTS users_identity_map (
      user_id TEXT PRIMARY KEY,
      cognito_sub TEXT UNIQUE NOT NULL,
      created_at DATETIME NOT NULL,
      updated_at DATETIME NOT NULL
    );
    
    CREATE TABLE IF NOT EXISTS bookmarks (
      id TEXT PRIMARY KEY,
      owner_id TEXT NOT NULL,
      title TEXT NOT NULL,
      url TEXT NOT NULL,
      description TEXT,
      tags TEXT,
      version INTEGER NOT NULL DEFAULT 1,
      created_at DATETIME NOT NULL,
      updated_at DATETIME NOT NULL,
      FOREIGN KEY (owner_id) REFERENCES users_identity_map (user_id),
      UNIQUE (owner_id, url)
    );
    
    CREATE INDEX IF NOT EXISTS idx_bookmarks_owner_id ON bookmarks (owner_id);
    CREATE INDEX IF NOT EXISTS idx_bookmarks_title ON bookmarks (title);
    CREATE INDEX IF NOT EXISTS idx_bookmarks_url ON bookmarks (url);
    CREATE INDEX IF NOT EXISTS idx_bookmarks_updated_at ON bookmarks (updated_at);
    
    CREATE TABLE IF NOT EXISTS audit_logs (
      id TEXT PRIMARY KEY,
      timestamp DATETIME NOT NULL,
      request_id TEXT NOT NULL,
      user_id TEXT,
      action TEXT NOT NULL,
      resource_id TEXT,
      status_code INTEGER NOT NULL,
      error_code TEXT,
      meta TEXT
    );
    
    CREATE INDEX IF NOT EXISTS idx_audit_logs_timestamp ON audit_logs (timestamp);
    CREATE INDEX IF NOT EXISTS idx_audit_logs_user_id ON audit_logs (user_id);
  `);
}

module.exports = { getDatabase };
```

---

## 6. Utility Components

### 6.1 Retry Utility (utils/retry.js)

**責任**:
- 指数バックオフリトライロジック
- 再試行可能エラーの判定

**インターフェース**:
```javascript
async function executeWithRetry(operation, maxRetries = 3)
```

**実装詳細**:
```javascript
// utils/retry.js
async function executeWithRetry(operation, maxRetries = 3) {
  const baseDelay = 100; // ms
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (!isRetryableError(error) || attempt === maxRetries - 1) {
        throw error;
      }
      
      // 指数バックオフ with ジッター
      const exponentialDelay = baseDelay * Math.pow(3, attempt);
      const jitter = Math.random() * 50;
      const totalDelay = exponentialDelay + jitter;
      
      console.log(`Retry attempt ${attempt + 1}/${maxRetries} after ${totalDelay}ms`);
      await sleep(totalDelay);
    }
  }
}

function isRetryableError(error) {
  return error.code === 'SQLITE_BUSY' 
      || error.code === 'SQLITE_LOCKED'
      || error.code === 'ECONNRESET'
      || (error.statusCode && error.statusCode >= 500);
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

module.exports = { executeWithRetry };
```

---

## 7. Component Interaction Flow

### 7.1 CREATE Bookmark Flow

```
Client
  │
  ↓ POST /bookmarks { title, url, description, tags }
┌─────────────────────────────────────┐
│  HTTP Layer (handler.js)            │
│  1. verifyJwt()                     │
│  2. validateInput()                 │
│  3. checkRateLimit()                │
│  4. routeRequest()                  │
└─────────────────────────────────────┘
  │
  ↓ userId, { title, url, ... }
┌─────────────────────────────────────┐
│  Service Layer                      │
│  BookmarkService.createBookmark()   │
│  - validateBookmarkData()           │
│  - checkDuplicate()                 │
│  - executeWithRetry()               │
└─────────────────────────────────────┘
  │
  ↓ userId, data
┌─────────────────────────────────────┐
│  Repository Layer                   │
│  BookmarkRepository.create()        │
│  - INSERT SQL with params           │
│  - Return created bookmark          │
└─────────────────────────────────────┘
  │
  ↓ SQL: INSERT INTO bookmarks ...
┌─────────────────────────────────────┐
│  Data Layer                         │
│  SQLite Connection                  │
│  - Execute parameterized query      │
│  - Return result                    │
└─────────────────────────────────────┘
  │
  ↓ { id, owner_id, title, url, version, ... }
┌─────────────────────────────────────┐
│  HTTP Layer                         │
│  - Format response                  │
│  - return { statusCode: 201, ... }  │
└─────────────────────────────────────┘
  │
  ↓ 201 Created
Client
```

---

### 7.2 UPDATE Bookmark Flow (Optimistic Locking)

```
Client
  │
  ↓ PUT /bookmarks/:id { title, url, version: 3 }
┌─────────────────────────────────────┐
│  HTTP Layer                         │
│  1. verifyJwt()                     │
│  2. validateInput()                 │
│  3. routeRequest()                  │
└─────────────────────────────────────┘
  │
  ↓ id, userId, updates, expectedVersion=3
┌─────────────────────────────────────┐
│  Service Layer                      │
│  BookmarkService.updateBookmark()   │
│  - validateBookmarkData()           │
│  - executeWithRetry() {             │
│      repo.update()                  │
│    }                                │
└─────────────────────────────────────┘
  │
  ↓ id, userId, updates, version=3
┌─────────────────────────────────────┐
│  Repository Layer                   │
│  BookmarkRepository.update()        │
│  - UPDATE ... WHERE version = ?     │
│  - if changes = 0 → check version   │
│  - throw ConflictError if mismatch  │
└─────────────────────────────────────┘
  │
  ↓ SQL: UPDATE bookmarks SET ... WHERE version = 3
┌─────────────────────────────────────┐
│  Data Layer                         │
│  SQLite Connection                  │
│  - Execute UPDATE                   │
│  - return { changes: 0 or 1 }       │
└─────────────────────────────────────┘
  │
  ↓ Case 1: changes = 1 (success)
  │    → return { statusCode: 200, body: updated bookmark }
  │
  ↓ Case 2: changes = 0 (conflict)
  │    → throw ConflictError
  │    → return { statusCode: 409, error: "Version mismatch" }
  │
Client
```

---

## 8. Component Summary

| コンポーネント | レイヤー | 主な責任 |
|-------------|---------|---------|
| **handler.js** | HTTP | エントリーポイント、リクエスト処理 |
| **jwt-verifier.js** | HTTP | JWT検証 |
| **input-validator.js** | HTTP | 入力バリデーション |
| **router.js** | HTTP | ルーティング |
| **error-handler.js** | HTTP | エラー処理 |
| **rate-limiter.js** | HTTP | レート制限 |
| **BookmarkService** | Service | ブックマークCRUD、検索 |
| **UserIdentityService** | Service | Cognito sub マッピング |
| **BookmarkRepository** | Repository | SQL実行、データマッピング |
| **UserIdentityRepository** | Repository | ユーザーSQL実行 |
| **sqlite-connection.js** | Data | DB接続管理、スキーマ初期化 |
| **retry.js** | Utility | リトライロジック |

---

## 9. File Structure

```
lambda-handler/
├── handler.js                      # Main Lambda handler
├── http/
│   ├── jwt-verifier.js
│   ├── input-validator.js
│   ├── router.js
│   ├── error-handler.js
│   └── rate-limiter.js
├── services/
│   ├── bookmark-service.js
│   └── user-identity-service.js
├── repositories/
│   ├── bookmark-repository.js
│   └── user-identity-repository.js
├── database/
│   └── sqlite-connection.js
└── utils/
    └── retry.js
```

---

## 10. Next Steps

1. **Code Generation**: この論理コンポーネント設計に基づきコード生成
2. **Unit Testing**: 各コンポーネントのユニットテスト実装
3. **Integration Testing**: レイヤー間の統合テスト実施

---
