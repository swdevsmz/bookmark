# アプリケーション設計: サービス層

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Application Design  
**作成日**: 2026-03-01

---

## サービス層概要

サービスはコンポーネント間の高度なオーケストレーションと調整を提供するもので、操作の実行方法の詳細を抽象化します。特定のドメインに焦点を当てた機能とは異なり、サービスはコンポーネント間で調整を行います。

---

## サービス定義

### 1. **API Service**（API通信サービス）

**責務**: フロントエンドとLambdaバックエンド間のすべてのHTTP通信を処理

**サービス境界**:
- APIエンドポイントURL管理（環境ベース）
- HTTPリクエスト/レスポンスのシリアライゼーション処理
- 指数バックオフを用いたリクエストリトライロジック実装
- エラーレスポンスとステータスコード管理
- リクエストに認証ヘッダー（JWTトークン）追加
- ネットワークタイムアウトとオフラインシナリオ処理

**相互作用**:
```
フロントエンド機能
    ↓
API Service（HTTP通信をオーケストレート）
    ↓
Lambda Handler API
```

**メソッド**（概要）:
- `request<T>(method: 'GET'|'POST'|'PUT'|'DELETE', path: string, body?: any): Promise<T>`
- `addAuthHeader(token: string): void`
- `setBaseUrl(url: string): void`

---

### 2. **Authentication Service**（認証サービス）

**責務**: 機能とページ全体の認証フローとセッション管理をオーケストレート

**サービス境界**:
- AuthFeature経由のAWS Cognito SDKコール調整
- JWTトークン保存管理（安全な保存、リフレッシュトークン）
- セッショントークンのリフレッシュ/有効期限処理
- グローバルな認証状態の維持
- 保護されたルートの強制

**相互作用**:
```
ページ（LoginPage、HomePageなど）
    ↓
Authentication Service（ログインフローをオーケストレート）
    ↓
AuthFeature（SDKコール実行）
    ↓
AWS Cognito
```

**管理される状態**:
- 現在のユーザー情報
- セッショントークン（ID、アクセス、リフレッシュ）
- MFAデバイス状態
- 認証状態（ログイン/ログアウト）

**メソッド**（概要）:
- `authenticate(username: string, password: string): Promise<AuthResult>`
- `handleMFAChallenge(challenge: MFAChallenge): Promise<SessionTokens>`
- `refreshSession(): Promise<SessionTokens>`
- `isAuthenticated(): boolean`
- `getCurrentUser(): User | null`

---

### 3. **State Management Service**（状態管理サービス）

**責務**: React Context + useReducerパターンを使用してアプリケーション状態を一中化

**サービス的達成**:
- ブックマークデータ状態を管理
- 検索/フィルター状態を管理
- UI状態（読み込み、モーダル、速報）を管理
- コンポーネント用の context hooks を提供
- 予測可能な状態更新のために reducer パターンを実装

**Interactions**:
```
Components (via hooks)
    ↓
State Management Service (React Context Providers)
    ↓
useBookmarkState, useSearchState, useUIState hooks
```

**State Domains**:

#### Bookmark State
- `bookmarks: Bookmark[]` - All loaded bookmarks
- `selectedBookmark: Bookmark | null` - Currently viewed bookmark
- `bookmarkLoading: boolean` - Loading flag
- `bookmarkError: string | null` - Error message

#### Search State
- `searchQuery: string` - Current search query
- `filteredResults: Bookmark[]` - Search results
- `isSearchActive: boolean` - Whether search is active

#### UI State
- `isModalOpen: boolean` - Modal open/close
- `notification: Notification | null` - Toast notifications
- `loading: boolean` - Global loading indicator

**メソッド**（概要）:
- `updateBookmarks(bookmarks: Bookmark[]): void`
- `addBookmark(bookmark: Bookmark): void`
- `updateBookmark(id: string, updates: any): void`
- `deleteBookmark(id: string): void`
- `setSearchQuery(query: string): void`
- `showNotification(message: string, type: 'success'|'error'|'info'): void`

---

### 4. **Data Persistence Service**（データ永続化サービス）

**責務**: データ取得操作を抽象化してバックエンドへの一貫したインターフェースを提供

**サービス的達成**:
- フロントエンドモデルと API ペイロード間の変換
- 失敗したリクエストのリトライロジックを実装
- ローカルキャッシュ戦略を管理
- ページネーションを処理
- API コールを最小化するためにレスポンスをキャッシュ

**Interactions**:
```
BookmarkManagementFeature
    ↓
Data Persistence Service (abstract storage)
    ↓
API Service (HTTP layer)
    ↓
Lambda Handler (database operations)
```

**メソッド**（概要）:
- `create<T>(endpoint: string, data: any): Promise<T>`
- `read<T>(endpoint: string): Promise<T>`
- `update<T>(endpoint: string, data: any, version?: number): Promise<T>`
- `delete(endpoint: string): Promise<void>`
- `getFromCache<T>(key: string): T | null`
- `setCache<T>(key: string, value: T, ttl?: number): void`

---

### 5. **Error Handling Service**（エラーハンドリングサービス）

**責務**: エラーハンドリング、ロギング、ユーザーフィードバックを一中化

**サービス的達成**:
- バックエンドエラーをユーザーフレンドリーなメッセージに変換
- エラーをコンソール/監視サービスに記録
- エラーの再試行履箇を判定
- React コンポーネントのエラー境界を管理
- UI 表示用をエラーメッセージをフォーマット

**Interactions**:
```
Any Component/Feature encountering error
    ↓
Error Handling Service
    ↓
User Notification (via UI State Service)
    ↓
Logging/Monitoring (optional)
```

**エラー種別を処理**:
- ネットワークエラー（タイムアウト、オフライン）
- 誤認認証エラー（有效失效トークン、無効認証情報）
- 認可エラー（アクセスを拒征）
- 検証エラー（不正な入力）
- サーバーエラー (500番並、データベースエラー)
- 競合エラー（乐観的ロック）

**メソッド**（概要）:
- `handleError(error: Error): HandledError`
- `getUserMessage(error: Error): string`
- `isRetryable(error: Error): boolean`
- `logError(error: Error, context: any): void`

---

### 6. **Validation Service**（バリデーションサービス）

**責務**: ブックマークデータとフォームのバリデーションロジックを一中化

**サービス的達成**:
- ブックマーク入力（URL、タイトルなど）を検証
- ユーザー入力（パスワード、メール）を検証
- 検証エラーメッセージを提供
- フィールドレベルとフォームレベルの検証を実装

**Interactions**:
```
Form Components (LoginPage, CreateBookmarkPage, etc.)
    ↓
Validation Service
    ↓
Returns validation errors
```

**メソッド**（概要）:
- `validateUrl(url: string): ValidationError | null`
- `validateTitle(title: string): ValidationError | null`
- `validateEmail(email: string): ValidationError | null`
- `validatePassword(password: string): ValidationError | null`
- `validateBookmarkInput(input: BookmarkInput): ValidationError[]`

---

## サービス相互作用図

```
┌─────────────────────────────────────────────────────────────────┐
│                      Frontend Application                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Pages       │  │  Features    │  │  Shared/UI   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│          │                │                │                     │
│          └────────────────┼────────────────┘                     │
│                           │                                       │
│                    ┌──────▼───────────┐                          │
│                    │  SERVICES LAYER  │                          │
│                    └──────┬───────────┘                          │
│         ┌──────────┬──────┼──────────┬──────────┬────────────┐   │
│         │          │      │          │          │            │   │
│    ┌────▼──┐  ┌────▼──┐ ┌─▼──────┐┌─▼──────┐┌─▼──────┐┌────▼──┐│
│    │ Auth  │  │ State │ │ Data   ││ Error  ││ Valid  ││ API   ││
│    │ Svc   │  │ Svc   │ │ Persist││ Svc    ││ Svc    ││ Svc   ││
│    └────┬──┘  └────┬──┘ │ Svc    │└────┬───┘└────┬───┘└────┬──┘│
│         │          │    └────┬───┘     │          │         │    │
│         │          │         │         │          │         │    │
│         └──────────┼─────────┼─────────┼──────────┼─────────┘    │
│                    │         │         │          │              │
├────────────────────┼─────────┼─────────┼──────────┼──────────────┤
│                    │         │         │          │              │
│              ┌─────▼─────────▼─────────▼──────────▼───────┐      │
│              │    AWS SDK v3 / HTTP Client                │      │
│              └────────────────────┬──────────────────────┘      │
│                                   │                             │
├───────────────────────────────────┼─────────────────────────────┤
│                                   │                             │
│                      ┌────────────▼────────────┐                │
│                      │  Lambda Handler (API)   │                │
│                      └────────────┬────────────┘                │
│                                   │                             │
│                      ┌────────────▼────────────┐                │
│                      │   Database Service      │                │
│                      │ (SQLite + S3)           │                │
│                      └─────────────────────────┘                │
│                                                                  │
│                           Backend                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## サービスライフサイクルと初期化

```javascript
// App.tsx初期化シーケンス

// 1. API Serviceを環境設定で初期化
APIService.setBaseUrl(env.API_ENDPOINT);

// 2. Authentication Serviceを初期化
const authService = new AuthenticationService();

// 3. State Managementを初期化
<StateProvider>
  <App />
</StateProvider>

// 4. 認証ガード付きでアプリをラップ
<ProtectedRoutes authService={authService}>
  <Pages />
</ProtectedRoutes>
```

---

## 通信パターン

### 同期パターン（UI フィードバック）
```
コンポーネント
  ↓
Validation Service（同期）
  ↓
Error Handling Service（無効な場合）
  ↓
UI State Service（エラー表示）
```

### 非同期パターン（データ操作）
```
コンポーネント
  ↓
Data Persistence Service（非同期）
  ↓
API Service（HTTPコール）
  ↓
Lambda Handler
  ↓
State Management Service（状態更新）
  ↓
コンポーネント再レンダリング
```

### 認証フロー
```
LoginPage
  ↓
Authentication Service
  ↓
AuthFeature（AWS SDKコール）
  ↓
AWS Cognito
  ↓
State Management Service（トークン保存）
  ↓
HomePageへのナビゲーション
```

---

## サマリー

| サービス | 目的 | 主要な責務 |
| API Service | HTTP通信 | リクエスト処理、認証ヘッダー、リトライ |
| Authentication Service | セッション管理 | トークン保存、ログイン/ログアウト、MFA |
| State Management Service | アプリケーション状態 | Context + Reducer による一中化された状態 |
| Data Persistence Service | データ操作 | CRUD抽象化、キャッシング |
| Error Handling Service | エラー管理 | エラー変換、ユーザーメッセージ |
| Validation Service | 入力検証 | フィールドおよびフォーム検証 |

**設計パターン**: 各サービスは**シングルトン風のインスタンス**であり、コンポーネントがReact hooksまたは直接インポート経由でアクセスするメソッドを提供します。サービスは**疎結合**で、状態更新または直接メソッド呼び出しを通じて相互調整します。
