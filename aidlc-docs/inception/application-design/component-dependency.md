# アプリケーション設計: コンポーネント依存関係

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Application Design  
**作成日**: 2026-03-01

---

## 依存関係マトリックス

下記の依存関係マトリックスは、どのコンポーネントが他のコンポーネントおよびサービスに依存しているかを示しています。"X" は依存関係を示します。

```
                              サービス層（横軸）
                 認証 │ 状態 │ データ│ エラー│ バリ │ API
                サービス│管理  │永続化│処理  │デ   │ 通信
──────────────────────────────────────────────────────────────
機能層:
  AuthFeature            │       │       │       │       │ X
  BookmarkMgmt Feature   │ X     │ X     │ X     │ X     │
  SearchFeature          │ X     │       │       │       │
──────────────────────────────────────────────────────────────
Pages:
  LoginPage              │ X     │ X     │ X     │ X     │
  MFASetupPage           │ X     │ X     │ X     │ X     │
  HomePage               │ X     │ X     │ X     │ X     │ X
  BookmarkDetailPage     │ X     │ X     │ X     │ X     │
  CreateBookmarkPage     │ X     │ X     │ X     │ X     │ X
  EditBookmarkPage       │ X     │ X     │ X     │ X     │ X
──────────────────────────────────────────────────────────────
```

---

## 依存関係の詳細説明

### 機能層

#### AuthFeature
- **直接依存**: `API Service`
- **推移的依存**: なし
- **説明**: API Serviceを使用してAWS Cognitoコールを実行

**Dependency Flow**:
```
AuthFeature
    └─→ API Service (AWS SDK v3 -> Cognito)
```

---

#### BookmarkManagementFeature
- **直接的な依存**: `State Management Service`、`Data Persistence Service`、`Error Handling Service`、`Validation Service`
- **説明**: CRUD操作の中核ビジネスロジック

**Dependency Flow**:
```
BookmarkManagementFeature
    ├─→ Validation Service (validate inputs)
    ├─→ Data Persistence Service (fetch/update data)
    ├─→ State Management Service (update app state)
    └─→ Error Handling Service (handle failures)
```

**Detailed Interactions**:
1. Validates input with Validation Service
2. Calls Data Persistence Service to perform backend operations
3. Updates State Management Service with results
4. Handles errors with Error Handling Service
5. Returns result to calling component

---

#### SearchFeature
- **直接的な依存**: `State Management Service`
- **説明**: クライアント側の検索・フィルタリング

**Dependency Flow**:
```
SearchFeature
    └─→ State Management Service (get/update search state)
```

**Detailed Behavior**:
- Reads bookmarks from state
- Performs in-memory filtering
- Updates search state in context
- No backend dependencies (MVP)

---

### Pages

#### LoginPage
- **Invokes**: `AuthFeature` (via Authentication Service)
- **Dependencies**: `Authentication Service`, `State Management Service`, `Error Handling Service`, `Validation Service`

**Dependency Flow**:
```
LoginPage
    ├─→ Validation Service (validate login inputs)
    ├─→ Authentication Service (login flow)
    │     └─→ AuthFeature
    │         └─→ API Service
    ├─→ State Management Service (store tokens, user info)
    └─→ Error Handling Service (show login errors)
```

**Execution Sequence**:
1. User fills login form
2. Validate inputs with Validation Service
3. Call Authentication Service.authenticate()
4. Auth Service invokes AuthFeature which calls AWS SDK
5. On success: Store tokens in State Management Service
6. Navigate to MFA page or Home depending on MFA status

---

#### MFASetupPage
- **Invokes**: `AuthFeature` (via Authentication Service)
- **Dependencies**: `Authentication Service`, `State Management Service`, `Error Handling Service`, `Validation Service`

**Dependency Flow**:
```
MFASetupPage
    ├─→ Validation Service (validate MFA codes)
    ├─→ Authentication Service (MFA setup)
    │     └─→ AuthFeature
    │         └─→ API Service
    ├─→ State Management Service (update MFA status)
    └─→ Error Handling Service (show setup errors)
```

---

#### HomePage
- **Invokes**: `BookmarkManagementFeature`, `SearchFeature`
- **Dependencies**: `State Management Service`, `Data Persistence Service`, `Error Handling Service`, `Validation Service`, `API Service`

**Dependency Flow**:
```
HomePage
    ├─→ BookmarkManagementFeature.getBookmarks()
    │     ├─→ Data Persistence Service
    │     │     └─→ API Service
    │     ├─→ State Management Service
    │     └─→ Error Handling Service
    ├─→ SearchFeature (search/filter)
    │     └─→ State Management Service
    └─→ User interactions (edit, delete, create)
```

**Page Lifecycle**:
1. On mount: Call getBookmarks()
2. BookmarkManagementFeature calls Data Persistence Service
3. Data Service makes API call via API Service
4. Results stored in State Management Service
5. Page renders bookmarks from context
6. User can search using SearchFeature
7. Click actions route to detail/edit/create pages

---

#### BookmarkDetailPage
- **Invokes**: `BookmarkManagementFeature`
- **Dependencies**: `State Management Service`, `Data Persistence Service`, `Error Handling Service`, `Validation Service`

**Dependency Flow**:
```
BookmarkDetailPage
    ├─→ BookmarkManagementFeature.getBookmark(id)
    │     ├─→ Data Persistence Service
    │     │     └─→ API Service
    │     ├─→ State Management Service
    │     └─→ Error Handling Service
    └─→ User actions → navigate to edit/delete
```

---

#### CreateBookmarkPage
- **Invokes**: `BookmarkManagementFeature`
- **Dependencies**: `State Management Service`, `Data Persistence Service`, `Error Handling Service`, `Validation Service`

**Dependency Flow**:
```
CreateBookmarkPage
    ├─→ Validation Service (validate form)
    ├─→ BookmarkManagementFeature.createBookmark()
    │     ├─→ Validation Service (re-validate)
    │     ├─→ Data Persistence Service
    │     │     └─→ API Service
    │     ├─→ State Management Service
    │     └─→ Error Handling Service
    └─→ Navigate to detail page on success
```

---

#### EditBookmarkPage
- **Invokes**: `BookmarkManagementFeature`
- **Dependencies**: `State Management Service`, `Data Persistence Service`, `Error Handling Service`, `Validation Service`

**Dependency Flow**:
```
EditBookmarkPage
    ├─→ BookmarkManagementFeature.getBookmark(id) [load current]
    │     ├─→ Data Persistence Service
    │     │     └─→ API Service
    │     └─→ State Management Service
    ├─→ Validation Service (validate updates)
    ├─→ BookmarkManagementFeature.updateBookmark()
    │     ├─→ Validation Service (re-validate)
    │     ├─→ Data Persistence Service [optimistic lock]
    │     │     └─→ API Service
    │     ├─→ State Management Service
    │     └─→ Error Handling Service [handle conflicts]
    └─→ Navigate on success or show conflict dialog
```

---

## 一方向依存ルール（非サイクル）

すべての依存はサイクルなしで一方向に流れます：

```
サービス層（下層 - 最安定）
    ↑ ↑ ↑ ↑ ↑ ↑ ↑
    │ │ │ │ │ │ │
機能層（中層）
    ↑ ↑ ↑ ↑ ↑
    │ │ │ │ │
ページ層（上層 - 最不安定）
```

**依存ルール**:
- ✅ ページ層は機能層に依存可能
- ✅ ページ層はサービス層に依存可能
- ✅ 機能層はサービス層に依存可能
- ❌ 機能層はページ層に依存不可
- ❌ サービス層はページ層または機能層に依存不可
- ❌ 循環依存なし

---

## コミュニケーションパターン

### パターン 1: シンプルなクエリ
```
HomePage
  ├─ State からブックマークを読み取り (useBookmarkState hook 経由)
  └─ 描画 / 様子をローカル更新
```

### パターン 2: データが誁需な場合
```
HomePage
  ├─ BookmarkManagementFeature.getBookmarks() を呼び出し
  ├─ Feature が検証 (必要な場合)
  ├─ Feature が Data Persistence Service を呼び出し
  ├─ Service が API Service を呼び出し
  ├─ API が Lambda を呼び出し
  ├─ レスポンスが消費元に戻る
  ├─ Feature が State Management Service を更新
  └─ HomePage が新しいデータで再描画
```

### パターン 3: 作成/更新でエラーハンドリングを处理
```
CreateBookmarkPage
  ├─ Validation Service でフォームを検証
  ├─ BookmarkManagementFeature.createBookmark() を呼び出し
  ├─ Feature が再検証
  ├─ Feature が Data Persistence Service を呼び出し
  ├─ Service が API Service を呼び出し
  ├─ エラー発生時: Feature が Error Handling Service を呼び出し
  ├─ Error Service が State Management Service を呼び出し (速報を表示)
  ├─ 成功時: Feature が State + UI を更新
  └─ ページが成功先にナビゲート
```

---

## サービス的達成

各サービスを独立して以下の出来を可能にします:
- **独立的にテスト** やいよいよ芳估依存关䯸
- **简単に交換** 例えば API Service を Mock に互換
- **拡大** 他のサービスに影響なし

### サービスインターフェース

```typescript
// API Service - すべての HTTP を処理
interface APIService {
  request<T>(method: string, path: string, body?: any): Promise<T>;
}

// State Management - App state を読み取り/更新
interface StateService {
  getBookmarks(): Bookmark[];
  updateBookmarks(bookmarks: Bookmark[]): void;
}

// Data Persistence - CRUD 抽象
interface DataService {
  create<T>(endpoint: string, data: any): Promise<T>;
  read<T>(endpoint: string): Promise<T>;
  update<T>(endpoint: string, data: any): Promise<T>;
  delete(endpoint: string): Promise<void>;
}

// Error Handling - エラーを変換
interface ErrorService {
  handleError(error: any): UserFriendlyError;
  isRetryable(error: any): boolean;
}

// Validation - 入力を検証
interface ValidationService {
  validate(data: any, rules: ValidationRules): ValidationError[];
}
```

---

## 依存サービス初期化顺序

アプリが起動したとき、サービスはこの順序で初期化する必要があります:

```
1. API Service (基礎 - 依存なし)
2. Validation Service (サービス依存なし)
3. Error Handling Service (API Service を使用)
4. Authentication Service (API Service を使用)
5. Data Persistence Service (API と Error サービスを使用)
6. State Management Service (UI にデータを提供)
7. すべての機能 (サービスを使用)
8. すべてのページ (機能 + サービスを使用)
```

---

## コンポーネント一覧

| コンポーネント | 種類 | 直接依存 | 推移的依存 | 責務 |
|-----------|------|-------------|-----------------|-----------------|
| AuthFeature | Feature | API Svc | - | AWS Cognito integration |
| BookmarkMgmt | Feature | State, Data, Error, Valid | API | CRUD operations |
| SearchFeature | Feature | State | - | Client-side search |
| LoginPage | Page | Auth, State, Error, Valid | API | Login UI |
| MFASetupPage | Page | Auth, State, Error, Valid | API | MFA setup UI |
| HomePage | Page | Feature(2), services(4) | API | Bookmark list view |
| DetailPage | Page | Feature, services | API | Detail view |
| CreatePage | Page | Feature, services | API | Create form |
| EditPage | Page | Feature, services | API | Edit form |

**主要指標**:
- **総コンポーネント数**: 11
- **総サービス数**: 6
- **循環依存**: 0
- **設計パターン**: サービス層による抽象化を含むレイヤーアーキテクチャ

