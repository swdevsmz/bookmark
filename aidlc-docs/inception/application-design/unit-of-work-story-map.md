# Unit of Work - ストーリーマッピング

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Units Generation  
**作成日**: 2026-03-01

---

## 概要

このドキュメントは、Requirements Analysis で定義された機能要件を、Units Generation で分解された3つのユニット（Frontend / Backend / Auth）にマッピングします。

---

## 機能要件 → ユニット マッピング

### FR-1: ブックマイク一覧表示

**要件**: ユーザーはブックマークの一覧を表示できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| UI表示 | **Frontend** | HomePage でブックマーク一覧を表示、ページング対応 |
| フェッチロジック | **Frontend** | BookmarkManagementFeature.getBookmarks() で API呼び出し |
| API実装 | **Backend** | GET /api/bookmarks エンドポイント、SQLite から取得 |
| 認証統合 | **Auth** | トークン検証（Backend 内で実施） |

**ユニット分担**:
- Frontend: 70% (UI + 状態管理)
- Backend: 30% (API + DB操作)

---

### FR-2: ブックマーク詳細表示

**要件**: ユーザーは選択したブックマークの詳細を表示できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| 詳細ページ | **Frontend** | BookmarkDetailPage レイアウト |
| データ取得 | **Frontend** | BookmarkManagementFeature.getBookmark(id) 呼び出し |
| API実装 | **Backend** | GET /api/bookmarks/{id} エンドポイント |
| メタデータ | **Backend** | createdAt, updatedAt, version フィールド返却 |

**ユニット分担**:
- Frontend: 60%
- Backend: 40%

---

### FR-3: ブックマーク新規登録

**要件**: ユーザーは新しいブックマークを作成できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| 入力フォーム | **Frontend** | CreateBookmarkPage, フォーム入力エラー処理 |
| クライアント側検証 | **Frontend** | Validation Service で URL, Title 検証 |
| API呼び出し | **Frontend** | BookmarkManagementFeature.createBookmark() |
| サーバー側検証 | **Backend** | バリデーション ロジック実装 |
| CRUD実装 | **Backend** | POST /api/bookmarks, SQLite INSERT |
| 自動フィールド | **Backend** | id, createdAt, updatedAt, version を自動生成 |

**ユニット分担**:
- Frontend: 50%
- Backend: 50%

---

### FR-4: ブックマーク編集

**要件**: ユーザーは既存のブックマークを編集できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| 編集フォーム | **Frontend** | EditBookmarkPage, 現在値の読み込み |
| 楽観的ロック | **Frontend** | version フィールドを取得して保持 |
| ローカル検証 | **Frontend** | Validation Service の再検証 |
| API呼び出し | **Frontend** | PUT /api/bookmarks/{id} で version 送信 |
| 競合制御 | **Backend** | version チェック、合致しない場合は 409 Conflict |
| DB更新 | **Backend** | UPDATE (version = current_version のみ), version increment |
| エラーハンドリング | **Frontend** | 409 検出時に conflict dialog 表示 |

**ユニット分担**:
- Frontend: 55%
- Backend: 45%

---

### FR-5: ブックマーク削除

**要件**: ユーザーはブックマークを削除できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| 削除ボタン | **Frontend** | HomePage / DetailPage に削除ボタン, 確認ダイアログ |
| API呼び出し | **Frontend** | DELETE /api/bookmarks/{id} |
| 権限確認 | **Backend** | ログインユーザーの所有物か確認 |
| DB削除 | **Backend** | DELETE FROM bookmarks WHERE id = ... |
| 状態同期 | **Frontend** | 削除後に一覧を更新 |

**ユニット分担**:
- Frontend: 50%
- Backend: 50%

---

### FR-6: ブックマーク検索・フィルタ

**要件**: ユーザーはブックマークを検索・フィルタできる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| 検索UI | **Frontend** | SearchBar コンポーネント |
| クライアント側検索 | **Frontend** | SearchFeature でローカル検索実装（フロントエンド利用） |
| 検索状態管理 | **Frontend** | State Management Service で searchQuery, filteredResults 保持 |
| サーバー側フィルタ | **Backend** | GET /api/bookmarks?search=<query> クエリパラメータ対応 |
| データベース検索 | **Backend** | SQLite LIKE クエリ機能 |

**ユニット分担**:
- Frontend: 60% (クライアント側検索)
- Backend: 40% (サーバー側検索オプション)

---

### FR-7: ユーザー登録（AWS Cognito）

**要件**: ユーザーはメールアドレスとパスワードで新規登録できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| サインアップフォーム | **Frontend** | 拡張 LoginPage または SignupPage |
| クライアント検証 | **Frontend** | Validation Service (email, password strength) |
| Cognito呼び出し | **Auth** | API エンドポイント: POST /auth/signup |
| ユーザープール登録 | **Auth** | AWS Cognito SignUp API 呼び出し |
| 確認コード | **Auth** | Email で確認コード送信 |
| 確認処理 | **Frontend** | 確認コード入力フォーム |
| 確認完了 | **Auth** | ConfirmSignUp API 呼び出し |

**ユニット分担**:
- Frontend: 40%
- Auth: 60%

---

### FR-8: ユーザーログイン（2要素認証）

**要件**: ユーザーはメール + パスワード + 2FA で認証できる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| ログインフォーム | **Frontend** | LoginPage (username/password) |
| クライアント検証 | **Frontend** | Validation Service (empty check) |
| ログイン API呼び出し | **Frontend** | AuthFeature.login() → Auth API |
| パスワード検証 | **Auth** | POST /auth/login with Cognito InitiateAuth |
| MFA チャレンジ | **Auth** | SMS/TOTP/Email による MFA 開始 |
| MFAコード入力 | **Frontend** | MFASetupPage または MFAInputDialog |
| MFA応答 API | **Frontend** | AuthFeature.respondToMFAChallenge() |
| MFA検証 | **Auth** | POST /auth/mfa-challenge with code |
| セッショントークン | **Auth** | ID, Access, Refresh トークン返却 |
| トークン分散 | **Frontend** | localStorage または secure cookie に保存 |

**ユニット分担**:
- Frontend: 40%
- Auth: 60%

---

### FR-9: MFA 設定

**要件**: ユーザーは 2FA を設定できる（SMS/TOTP/Email）

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| MFA設定UI | **Frontend** | MFASetupPage, 複数オプション表示 |
| MFAオプション選択 | **Frontend** | Radio buttons for SMS/TOTP/Email |
| 設定 API呼び出し | **Frontend** | AuthFeature.setupMFA(option) |
| 設定開始 | **Auth** | POST /auth/mfa/setup with option |
| QR生成（TOTP場合） | **Auth** | Google Authenticator 用 QR コード |
| QR表示 | **Frontend** | QR コード画像表示（TOTP の場合） |
| 検証コード入力 | **Frontend** | MFA 確認フォーム |
| 設定完了 API | **Frontend** | AuthFeature.confirmMFASetup(code) |
| 設定確定 | **Auth** | POST /auth/mfa/confirm with code |

**ユニット分担**:
- Frontend: 40%
- Auth: 60%

---

### FR-10: ログアウト

**要件**: ユーザーはログアウトできる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| ログアウトボタン | **Frontend** | Header / Navigation メニュー |
| ローカルクリア | **Frontend** | localStorage token 削除, State リセット |
| ログアウト API | **Frontend** | AuthFeature.logout() |
| 初期化 | **Auth** | POST /auth/logout (Cognito GlobalSignOut) |
| 画面遷移 | **Frontend** | リダイレクト to LoginPage |

**ユニット分担**:
- Frontend: 40%
- Auth: 60%

---

### FR-11: パスワードリセット

**要件**: ユーザーはパスワードを忘れた場合リセットできる

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| リセット画面 | **Frontend** | LoginPage に「パスワードを忘れた」リンク → ForgotPasswordPage |
| メール入力 | **Frontend** | Validation Service で email 形式チェック |
| リセット要求 API | **Frontend** | AuthFeature.resetPassword(email) |
| メール送信 | **Auth** | POST /auth/forgot-password, Cognito がメール送信 |
| コード確認画面 | **Frontend** | 確認コード入力 + 新パスワード入力 |
| リセット完了 API | **Frontend** | AuthFeature.confirmPasswordReset(code, newPassword) |
| パスワード更新 | **Auth** | POST /auth/confirm-password-reset |

**ユニット分担**:
- Frontend: 45%
- Auth: 55%

---

### NFR-1: セキュリティ - JWT トークン検証

**要件**: すべての API リクエストは JWT 検証が必須

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| bearer token 送信 | **Frontend** | API Service で Authorization ヘッダに token 設定 |
| トークン検証 | **Backend** | 全エンドポイントで JWT 署名・有効期限チェック |
| 失期トークン処理 | **Frontend** | 401 エラー → /auth/refresh-session 自動呼び出し |
| リフレッシュロジック | **Auth** | POST /auth/refresh-session で新 token 発行 |
| リトライ | **Frontend** | トークン更新後に元のリクエストを再試行 |

**ユニット分担**:
- Frontend: 30%
- Backend: 40%
- Auth: 30%

---

### NFR-2: 並行制御 - Optimistic Locking

**要件**: 複数ユーザーの同時編集時に競合を検出

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| version フィールド | **Backend** | 各ブックマークに version カラム（初期値: 1） |
| 更新時の検証 | **Backend** | PUT /api/bookmarks で version チェック |
| 版の増分 | **Backend** | 成功時に version += 1 |
| 競合エラー | **Backend** | 競合時に 409 Conflict レスポンス |
| クライアント処理 | **Frontend** | 409 検出時に conflict dialog (再取得 or 上書き) |

**ユニット分担**:
- Frontend: 40%
- Backend: 60%

---

### NFR-3: キャッシング

**要件**: ブックマーク一覧のキャッシングで応答性を改善

| 側面 | ユニット | 実装内容 |
|------|---------|---------|
| ローカルキャッシュ | **Frontend** | BookmarkManagementFeature で getCachedBookmarks() |
| キャッシュTTL | **Frontend** | 5分間有効（カスタマイズ可能） |
| キャッシュ無効化 | **Frontend** | 新規作成/編集/削除時に自動クリア |
| サーバー側キャッシング | **Backend** | Redis（オプション） or Database Query Cache |

**ユニット分担**:
- Frontend: 60%
- Backend: 40%

---

## ストーリー集約表

| # | 機能 | ユニット | 優先度 | 複雑度 | 推定工数 |
|---|------|---------|--------|--------|---------|
| 1 | ブックマーク一覧 | FE + BE | 高 | 中 | 8h |
| 2 | ブックマーク詳細 | FE + BE | 高 | 低 | 4h |
| 3 | ブックマーク作成 | FE + BE | 高 | 中 | 8h |
| 4 | ブックマーク編集 | FE + BE | 高 | 高 | 12h |
| 5 | ブックマーク削除 | FE + BE | 高 | 低 | 4h |
| 6 | ブックマーク検索 | FE + BE | 中 | 低 | 6h |
| 7 | ユーザー登録 | FE + Auth | 高 | 中 | 8h |
| 8 | ユーザーログイン | FE + Auth | 高 | 高 | 12h |
| 9 | MFA 設定 | FE + Auth | 高 | 高 | 12h |
| 10 | ログアウト | FE + Auth | 中 | 低 | 2h |
| 11 | パスワードリセット | FE + Auth | 中 | 中 | 8h |
| NFR-1 | JWT 検証 | BE + Auth | 高 | 中 | 8h |
| NFR-2 | 並行制御 | FE + BE | 中 | 高 | 10h |
| NFR-3 | キャッシング | FE + BE | 低 | 低 | 6h |

**合計推定**: 118時間 (3-4週間 with 3-4 エンジニア)

---

## ユニット別作業分担

### Frontend ユニット
- FR-1 (70%), FR-2 (60%), FR-3 (50%), FR-4 (55%), FR-5 (50%), FR-6 (60%)
- FR-7 (40%), FR-8 (40%), FR-9 (40%), FR-10 (40%), FR-11 (45%)
- NFR-1 (30%), NFR-2 (40%), NFR-3 (60%)
- **合計工数**: 約 60 時間

### Backend ユニット
- FR-1 (30%), FR-2 (40%), FR-3 (50%), FR-4 (45%), FR-5 (50%), FR-6 (40%)
- NFR-1 (40%), NFR-2 (60%), NFR-3 (40%)
- **合計工数**: 約 30 時間

### Auth ユニット
- FR-7 (60%), FR-8 (60%), FR-9 (60%), FR-10 (60%), FR-11 (55%)
- NFR-1 (30%)
- **合計工数**: 約 28 時間

---
