# Backend Service - NFR Design Patterns

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-02  
**ステージ**: Construction - NFR Design

---

## Executive Summary

このドキュメントは、Backend ServiceのNFR要件を満たすための具体的な設計パターンを定義します。小規模ユーザーベース（10→20人）とコスト最適化を優先しながら、<1秒の検索SLA、レジリエンス、セキュリティを達成します。

---

## 1. Performance Design Patterns

### 1.1 Search Performance Optimization

**パターン**: SQLite LIKE with Indexes

**戦略**:
- SQLite `LIKE` クエリと適切なインデックス戦略を組み合わせて検索パフォーマンスを最適化
- メモリ内キャッシュの複雑さを回避し、シンプルなアーキテクチャを維持

**実装詳細**:

```sql
-- インデックス戦略
CREATE INDEX idx_bookmarks_owner_id ON bookmarks (owner_id);
CREATE INDEX idx_bookmarks_title ON bookmarks (title);
CREATE INDEX idx_bookmarks_url ON bookmarks (url);
CREATE INDEX idx_bookmarks_updated_at ON bookmarks (updated_at);

-- 検索クエリ最適化
SELECT * FROM bookmarks 
WHERE owner_id = ? 
  AND (title LIKE ? OR url LIKE ? OR description LIKE ? OR tags LIKE ?)
ORDER BY updated_at DESC
LIMIT 50;
```

**パフォーマンス特性**:
- **初回クエリ**: SQLiteのクエリプランナーがインデックスを使用、100件のブックマークで < 50ms 想定
- **後続クエリ**: Lambda インスタンス再利用時、SQLite キャッシュが有効、< 20ms 想定
- **最悪ケース**: Cold Start + 初回クエリでも < 1s SLA を満たす

**トレードオフ**:
- ✅ シンプルな実装、メンテナンスコスト低
- ✅ 小規模データ（2,000件未満）で十分なパフォーマンス
- ✅ インデックス管理が容易
- ⚠️ ユーザー数が100人を超えると性能低下の可能性（将来的にはRDS移行を検討）

---

### 1.2 Query Optimization Strategy

**パターン**: Index-Driven Query Planning

**最適化手法**:

1. **WHERE句の最適化**:
   - `owner_id` を最初のフィルター条件として使用（カーディナリティが高い）
   - 複数フィールドの LIKE 検索を OR で結合

2. **SELECT句の最適化**:
   - 必要なカラムのみを取得（`SELECT *` の使用は開発段階のみ）
   - プロダクション環境では明示的なカラムリストを使用

3. **ページネーション**:
   - `LIMIT` と `OFFSET` を使用
   - デフォルト: `LIMIT 50`、最大: `LIMIT 200`
   - `updated_at DESC` でソート（タイムスタンプインデックス利用）

**例**:
```sql
-- 効率的なページネーション
SELECT id, owner_id, title, url, description, tags, version, created_at, updated_at
FROM bookmarks
WHERE owner_id = ?
ORDER BY updated_at DESC
LIMIT 50 OFFSET ?;
```

---

### 1.3 Cold Start Acceptance Pattern

**パターン**: No Optimization / Client-Side UX

**戦略**:
- Lambda Cold Start を許容し、インフラストラクチャコスト（Provisioned Concurrency）を削減
- クライアント側で「読み込み中...」UIを表示してユーザー体験を管理

**実装ガイドライン**:
- Lambda 初期化コードは最小限に（SQLite ドライバーのロードのみ）
- Provisioned Concurrency は使用しない
- クライアント側で API 呼び出し前にローディングスピナーを表示

**パフォーマンス期待値**:
- **Cold Start**: 2-4秒（Lambda 初期化 + SQLite ロード）
- **Warm Start**: < 100ms（Lambda 再利用時）

---

## 2. Resilience Design Patterns

### 2.1 Retry Pattern with Exponential Backoff

**パターン**: Exponential Backoff with Jitter

**目的**:
- 一時的なエラー（SQLite ロックタイムアウト、ネットワーク障害）からの自動回復
- リトライストームを防ぐためのジッター導入

**実装詳細**:

```javascript
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
      const exponentialDelay = baseDelay * Math.pow(3, attempt); // 100ms, 300ms, 900ms
      const jitter = Math.random() * 50; // 0-50ms のランダムジッター
      const totalDelay = exponentialDelay + jitter;
      
      console.log(`Retry attempt ${attempt + 1}/${maxRetries} after ${totalDelay}ms`);
      await sleep(totalDelay);
    }
  }
}

function isRetryableError(error) {
  // SQLite ロックタイムアウト、一時的なネットワークエラー
  return error.code === 'SQLITE_BUSY' 
      || error.code === 'SQLITE_LOCKED'
      || error.code === 'ECONNRESET'
      || error.statusCode >= 500; // サーバーエラー
}
```

**リトライスケジュール**:
- **試行1**: 即座に実行
- **試行2**: 100ms + ジッター後
- **試行3**: 300ms + ジッター後
- **試行4**: 900ms + ジッター後

**適用範囲**:
- ✅ SQLite クエリ実行
- ✅ データベース書き込み操作
- ❌ バリデーションエラー（即座に失敗）
- ❌ 認証エラー（即座に失敗）

---

### 2.2 Fail-Fast Pattern

**パターン**: Early Validation with Fail-Fast

**目的**:
- 永続的なエラー（バリデーションエラー、認証エラー）を早期に検出
- 無駄なリトライを避け、レスポンスタイムを改善

**実装戦略**:
- リクエスト受信直後にすべての入力を検証
- バリデーション失敗時は即座に 400 Bad Request を返す
- JWT 検証失敗時は即座に 401 Unauthorized を返す

---

### 2.3 Timeout Pattern

**パターン**: Cascading Timeout Strategy

**タイムアウト階層**:

| レイヤー | タイムアウト | 目的 |
|---------|------------|------|
| **API Gateway** | 30秒 | クライアントへの最大応答時間 |
| **Lambda 関数** | 10秒 | 処理全体のタイムアウト |
| **SQLite クエリ** | 5秒 | データベースロック待機時間 |
| **個別操作** | 2秒 | 単一クエリの最大実行時間 |

**実装例**:
```javascript
// SQLite クエリタイムアウト
db.configure('busyTimeout', 5000); // 5秒

// Lambda 関数レベルのタイムアウト監視
const timeoutPromise = new Promise((_, reject) => 
  setTimeout(() => reject(new Error('Lambda timeout')), 9000) // 9秒（余裕を持たせる）
);

const result = await Promise.race([
  executeQuery(sql, params),
  timeoutPromise
]);
```

---

## 3. Security Design Patterns

### 3.1 Request-Level Throttling Pattern

**パターン**: In-Memory Rate Limiting (Per Lambda Instance)

**戦略**:
- Lambda インスタンスごとにメモリ内マップでレート制限を追跡
- DynamoDB を使用しないことでコストを削減（小規模ユーザーベース向け）

**実装詳細**:

```javascript
// Lambda インスタンスレベルのレート制限マップ
const rateLimitMap = new Map(); // userId -> { count, resetTime }

function checkRateLimit(userId, limit = 100, windowMs = 60000) {
  const now = Date.now();
  const userLimit = rateLimitMap.get(userId);
  
  if (!userLimit || now > userLimit.resetTime) {
    // 新しいウィンドウ
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

// ハンドラーでの使用
exports.handler = async (event) => {
  const userId = event.requestContext.authorizer.claims.sub;
  
  if (!checkRateLimit(userId, 100, 60000)) {
    return {
      statusCode: 429,
      body: JSON.stringify({ error: 'Too Many Requests' }),
      headers: {
        'Retry-After': '60',
        'X-RateLimit-Limit': '100',
        'X-RateLimit-Remaining': '0'
      }
    };
  }
  
  // 通常の処理
  // ...
};
```

**レート制限設定**:
- **デフォルト**: 100リクエスト/分/ユーザー
- **検索系**: 20リクエスト/分/ユーザー（より厳格）
- **書き込み系**: 30リクエスト/分/ユーザー

**トレードオフ**:
- ✅ コスト効率的（DynamoDB 不要）
- ✅ 低レイテンシ（メモリアクセス）
- ⚠️ Lambda インスタンス間で一貫性なし（小規模では許容）
- ⚠️ Lambda インスタンス終了時にリセット

---

### 3.2 Input Validation Pattern

**パターン**: Early Validation / Fail-Fast

**戦略**:
- HTTP リクエストパース直後、ルーティング前に全入力を検証
- バリデーションエラーは即座に返し、ビジネスロジックに到達させない

**バリデーション階層**:

```javascript
// レイヤー1: スキーマバリデーション（JSONスキーマ）
function validateSchema(body, schema) {
  const ajv = new Ajv();
  const valid = ajv.validate(schema, body);
  if (!valid) {
    throw new ValidationError(ajv.errors);
  }
}

// レイヤー2: ビジネスルールバリデーション
function validateBusinessRules(bookmark) {
  // タイトル長さ
  if (bookmark.title.length < 1 || bookmark.title.length > 200) {
    throw new ValidationError('Title must be 1-200 characters');
  }
  
  // URL形式
  const urlPattern = /^https?:\/\/.+/;
  if (!urlPattern.test(bookmark.url)) {
    throw new ValidationError('Invalid URL format');
  }
  
  // 説明文長さ
  if (bookmark.description && bookmark.description.length > 1000) {
    throw new ValidationError('Description must be <= 1000 characters');
  }
  
  // タグ形式
  if (bookmark.tags) {
    const tags = bookmark.tags.split(',');
    if (tags.length > 10) {
      throw new ValidationError('Maximum 10 tags allowed');
    }
  }
}

// レイヤー3: セキュリティバリデーション（XSS、SQLインジェクション対策）
function sanitizeInput(input) {
  // HTMLタグ除去
  const cleaned = input.replace(/<[^>]*>/g, '');
  // SQLキーワードエスケープ（Parameterized Queriesで対応）
  return cleaned;
}
```

**バリデーションタイミング**:
1. **HTTP Layer**: パース直後にスキーマバリデーション
2. **Service Layer**: ビジネスルールバリデーション
3. **Repository Layer**: SQLインジェクション対策（Parameterized Queries）

---

### 3.3 Optimistic Locking Pattern

**パターン**: Strict Version Check with 409 Conflict

**戦略**:
- バージョンフィールドを使用して競合を検出
- バージョン不一致時は 409 Conflict を返し、クライアントに再フェッチ + 再試行を要求

**実装詳細**:

```sql
-- 更新クエリ（楽観的ロック）
UPDATE bookmarks
SET 
  title = ?,
  url = ?,
  description = ?,
  tags = ?,
  version = version + 1,
  updated_at = ?
WHERE id = ? 
  AND owner_id = ?
  AND version = ?; -- 楽観的ロックチェック
```

```javascript
async function updateBookmark(bookmarkId, userId, updates, expectedVersion) {
  const result = await db.run(
    `UPDATE bookmarks 
     SET title = ?, url = ?, description = ?, tags = ?, 
         version = version + 1, updated_at = ? 
     WHERE id = ? AND owner_id = ? AND version = ?`,
    [updates.title, updates.url, updates.description, updates.tags, 
     new Date().toISOString(), bookmarkId, userId, expectedVersion]
  );
  
  if (result.changes === 0) {
    // バージョン不一致 or レコード不存在
    const existing = await db.get(
      'SELECT version FROM bookmarks WHERE id = ? AND owner_id = ?',
      [bookmarkId, userId]
    );
    
    if (!existing) {
      throw new NotFoundError('Bookmark not found');
    } else {
      throw new ConflictError('Version mismatch. Please re-fetch and retry.');
    }
  }
  
  return result;
}
```

**エラーレスポンス**:
```json
{
  "statusCode": 409,
  "error": "Conflict",
  "message": "The bookmark has been modified. Please re-fetch and retry.",
  "currentVersion": 5
}
```

**クライアント側の対応**:
1. 409 Conflict を受信
2. 最新のブックマークを再フェッチ（GET /bookmarks/:id）
3. ユーザーの変更をマージ
4. 新しいバージョン番号で再試行

---

## 4. Observability Design Patterns

### 4.1 Minimal Logging Pattern

**パターン**: Error-Only Logging with CloudWatch Metrics

**戦略**:
- ログ出力を最小限に抑え、CloudWatch のストレージコストを削減
- エラーのみをログに記録
- CloudWatch メトリクスで Lambda の Duration、Invocations、Errors を追跡

**ログ実装**:

```javascript
// エラーのみをログ
function logError(error, context) {
  console.error(JSON.stringify({
    timestamp: new Date().toISOString(),
    requestId: context.requestId,
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack,
      code: error.code
    }
  }));
}

// 正常系はログ出力なし（CloudWatch メトリクスで追跡）
exports.handler = async (event, context) => {
  try {
    const result = await processRequest(event);
    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    logError(error, context); // エラーのみログ
    return {
      statusCode: error.statusCode || 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

**CloudWatch メトリクス設定**:
- **Duration**: Lambda 実行時間（レスポンスタイム追跡）
- **Invocations**: 呼び出し回数
- **Errors**: エラー発生回数
- **Throttles**: スロットリング発生回数

**アラート設定**:
```yaml
# CloudWatch Alarms
ResponseTimeAlarm:
  MetricName: Duration
  Threshold: 1000 # 1秒
  EvaluationPeriods: 2
  ComparisonOperator: GreaterThanThreshold

ErrorRateAlarm:
  MetricName: Errors
  Statistic: Average
  Threshold: 0.05 # 5%
  EvaluationPeriods: 5
  ComparisonOperator: GreaterThanThreshold
```

**トレードオフ**:
- ✅ コスト効率的（ログストレージ最小限）
- ✅ CloudWatch 無料枠内で運用可能
- ⚠️ デバッグ時に詳細なログが不足（環境変数で切り替え可能にする）

---

### 4.2 Metrics-Driven Monitoring

**パターン**: CloudWatch Native Metrics

**メトリクス戦略**:
- Lambda のネイティブメトリクスを最大限活用
- カスタムメトリクスは最小限に（コスト削減）

**追跡メトリクス**:

| メトリクス | ソース | 目的 |
|----------|--------|------|
| **Duration** | Lambda | レスポンスタイム追跡（1秒 SLA） |
| **Errors** | Lambda | エラー率追跡（5% アラート） |
| **Throttles** | Lambda | レート制限監視 |
| **ConcurrentExecutions** | Lambda | 同時実行数監視 |
| **Invocations** | Lambda | トラフィック量追跡 |

---

## 5. Architecture Design Patterns

### 5.1 Layered Architecture Pattern

**パターン**: 4-Layer Clean Architecture

**レイヤー構成**:

```
┌─────────────────────────────────────┐
│      HTTP Layer (API Gateway)       │  ← リクエスト受信、レスポンス返却
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│       Service Layer (Business)      │  ← ビジネスロジック、オーケストレーション
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│    Repository Layer (Data Access)   │  ← データベースクエリ、永続化
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│      Data Layer (SQLite /tmp)       │  ← データストレージ
└─────────────────────────────────────┘
```

**各レイヤーの責務**:

#### HTTP Layer
- リクエストパース
- JWT 検証
- 入力バリデーション
- レスポンスフォーマット
- エラーハンドリング

```javascript
// http-layer/handler.js
exports.handler = async (event, context) => {
  try {
    // 1. JWT 検証
    const userId = await verifyJwt(event.headers.Authorization);
    
    // 2. リクエストパース
    const body = JSON.parse(event.body);
    
    // 3. 早期バリデーション
    validateSchema(body, bookmarkSchema);
    
    // 4. ルーティング
    const result = await routeRequest(event.httpMethod, event.path, body, userId);
    
    // 5. レスポンスフォーマット
    return {
      statusCode: 200,
      body: JSON.stringify(result),
      headers: { 'Content-Type': 'application/json' }
    };
  } catch (error) {
    return handleError(error);
  }
};
```

#### Service Layer
- ビジネスロジック実行
- トランザクション管理
- 楽観的ロック制御
- リトライロジック

```javascript
// service-layer/bookmark-service.js
class BookmarkService {
  constructor(bookmarkRepository) {
    this.repo = bookmarkRepository;
  }
  
  async updateBookmark(id, userId, updates, expectedVersion) {
    // ビジネスルールバリデーション
    validateBusinessRules(updates);
    
    // 楽観的ロックで更新（リトライ付き）
    return await executeWithRetry(async () => {
      return await this.repo.update(id, userId, updates, expectedVersion);
    });
  }
  
  async searchBookmarks(userId, query) {
    // 検索ロジック
    return await this.repo.search(userId, query);
  }
}
```

#### Repository Layer
- SQLクエリ実行
- Parameterized Queries（SQLインジェクション対策）
- データマッピング

```javascript
// repository-layer/bookmark-repository.js
class BookmarkRepository {
  constructor(db) {
    this.db = db;
  }
  
  async update(id, userId, updates, expectedVersion) {
    const result = await this.db.run(
      `UPDATE bookmarks 
       SET title = ?, url = ?, description = ?, tags = ?, 
           version = version + 1, updated_at = ? 
       WHERE id = ? AND owner_id = ? AND version = ?`,
      [updates.title, updates.url, updates.description, updates.tags,
       new Date().toISOString(), id, userId, expectedVersion]
    );
    
    if (result.changes === 0) {
      throw new ConflictError('Version mismatch');
    }
    
    return await this.findById(id, userId);
  }
  
  async search(userId, query) {
    const pattern = `%${query}%`;
    return await this.db.all(
      `SELECT * FROM bookmarks 
       WHERE owner_id = ? 
         AND (title LIKE ? OR url LIKE ? OR description LIKE ? OR tags LIKE ?)
       ORDER BY updated_at DESC
       LIMIT 50`,
      [userId, pattern, pattern, pattern, pattern]
    );
  }
}
```

#### Data Layer
- SQLite データベース接続管理
- Lambda /tmp ディレクトリ管理

```javascript
// data-layer/sqlite-connection.js
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');

let dbConnection = null;

async function getDatabase() {
  if (!dbConnection) {
    dbConnection = await open({
      filename: '/tmp/bookmark.db',
      driver: sqlite3.Database
    });
    
    // タイムアウト設定
    await dbConnection.run('PRAGMA busy_timeout = 5000;');
  }
  
  return dbConnection;
}
```

---

## 6. Design Pattern Summary

| パターン | 目的 | 実装戦略 |
|---------|------|---------|
| **SQLite LIKE with Indexes** | 検索パフォーマンス | インデックス最適化、シンプルな実装 |
| **Exponential Backoff** | レジリエンス | 3回リトライ、指数遅延、ジッター |
| **Request-Level Throttling** | セキュリティ | メモリ内レート制限、コスト削減 |
| **Layered Architecture** | 保守性 | HTTP → Service → Repository → Data |
| **Cold Start Acceptance** | コスト削減 | 最適化なし、クライアント側UX対応 |
| **Early Validation** | セキュリティ | Fail-Fast、早期エラー検出 |
| **Minimal Logging** | コスト削減 | エラーのみログ、CloudWatch メトリクス活用 |
| **Strict Version Check** | データ整合性 | 409 Conflict、クライアント再試行 |

---

## 7. Quality Attributes Achievement

| NFR要件 | 達成手段 | 期待値 |
|---------|---------|--------|
| **検索 < 1秒** | SQLite LIKE + インデックス | < 50ms (warm), < 1s (cold) |
| **3回リトライ** | Exponential backoff | 自動回復率 > 95% |
| **レート制限** | Request-level throttling | 100 req/min/user |
| **Cold Start許容** | No optimization | 2-4秒（許容） |
| **楽観的ロック** | Strict version check | 409 Conflict で競合解決 |
| **30日ログ保持** | CloudWatch Logs | 自動管理 |
| **5%エラー警告** | CloudWatch Alarms | 自動アラート |

---

## 8. Next Steps

1. **Infrastructure Design**: これらのパターンをTerraformで実装
2. **Code Generation**: レイヤードアーキテクチャでコード生成
3. **Testing**: パフォーマンステスト、レジリエンステスト実施

---
