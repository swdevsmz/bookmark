# アプリケーション設計: コンポーネントメソッド

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Application Design  
**作成日**: 2026-03-01

---

## コンポーネント別メソッドシグネチャー

### AuthFeature

#### `signUp(email: string, password: string): Promise<SignUpResult>`
- **目的**: AWS Cognitoに新規ユーザーを登録
- **入力**: メールアドレスとパスワード
- **出力**: ユーザー確認状態と次のステップを含むSignUpResult
- **備考**: メール確認が必要な場合がある

#### `login(username: string, password: string): Promise<LoginResult>`
- **目的**: ユーザーを認証しMFAが有効な場合は開始
- **入力**: ユーザー名/メールとパスワード
- **出力**: セッショントークンまたはMFAチャレンジを含むLoginResult
- **備考**: ユーザーがMFA有効の場合はMFAチャレンジを返す

#### `respondToMFAChallenge(challengeToken: string, mfaCode: string): Promise<SessionTokens>`
- **目的**: 提供されたコードを使ってMFA検証を完了
- **入力**: ログインからのチャレンジトークン、MFAコード（SMS/TOTP/メール）
- **出力**: セッショントークン（IDトークン、アクセストークン、リフレッシュトークン）
- **備考**: コード形式とタイムリネスを検証

#### `setupMFA(mfaOption: 'SMS' | 'TOTP' | 'EMAIL'): Promise<MFASetupInfo>`
- **目的**: MFAデバイス設定を開始
- **入力**: ユーザーが選択したMFA方式
- **出力**: 設定情報（電話番号、QRコード付きTOTPシークレット、メール確認）
- **備考**: コード生成の詳細ビジネスロジックは機能設計フェーズで定義

#### `confirmMFASetup(setupToken: string, verificationCode: string): Promise<void>`
- **目的**: ユーザーがコード検証後、MFAデバイスを確認
- **入力**: セットアップトークン、ユーザー提供の検証コード
- **出力**: MFAデバイス関連付けの確認
- **備考**: もう一度コードを検証する必要あり

#### `logout(): Promise<void>`
- **目的**: セッションをクリアしてトークンを無効化
- **入力**: 現在のセッショントークン
- **出力**: なし（成功確認）
- **備考**: リフレッシュトークンを無効化

#### `resetPassword(email: string): Promise<ResetTokenInfo>`
- **目的**: ユーザーのパスワードリセットフローを開始
- **入力**: ユーザーのメールアドレス
- **出力**: リセット手順を含む確認
- **備考**: メールにリセットコードを送信

#### `confirmPasswordReset(email: string, resetCode: string, newPassword: string): Promise<void>`
- **目的**: 新しいパスワードでパスワードリセットを完了
- **入力**: メール、メールからのリセットコード、新しいパスワード
- **出力**: リセット成功の確認
- **備考**: パスワードバリデーションは機能設計フェーズで定義

---

### BookmarkManagementFeature

#### `createBookmark(bookmarkInput: BookmarkInput): Promise<Bookmark>`
- **目的**: ユーザーのコレクションに新しいブックマークを追加
- **入力**: ブックマークデータ（URL、タイトル、説明、タグ、カテゴリ、メモ、サムネイル）
- **出力**: システムフィールド付きのブックマーク（id、userId、createdAt、updatedAt）
- **備考**: URLとタイトルのバリデーションは機能設計フェーズで定義；Lambdaバックエンドを呼び出し

#### `getBookmarks(): Promise<Bookmark[]>`
- **目的**: 認証ユーザーのすべてのブックマークを取得
- **入力**: ユーザーセッション（暗黙的）
- **出力**: ブックマークオブジェクトの配列
- **備考**: ページネーション/無限スクロール処理は機能設計フェーズで定義

#### `getBookmark(bookmarkId: string): Promise<Bookmark>`
- **目的**: 単一ブックマークの詳細を取得
- **入力**: ブックマークID
- **出力**: すべてのフィールドを含む完全なブックマークオブジェクト
- **備考**: Lambdaバックエンドを呼び出し；見つからないエラーを処理

#### `updateBookmark(bookmarkId: string, updates: BookmarkInput): Promise<Bookmark>`
- **目的**: 既存ブックマークを修正
- **入力**: ブックマークIDと更新フィールド値
- **出力**: インクリメントされたバージョンと新しいupdatedAtを含む更新済みブックマーク
- **備考**: 楽観的ロック（バージョン確認）を実装；詳細な競合解決は機能設計フェーズで定義

#### `deleteBookmark(bookmarkId: string): Promise<void>`
- **目的**: ブックマークをコレクションから削除
- **入力**: ブックマークID
- **出力**: 削除の確認
- **備考**: クライアント側確認UI；ハード削除（MVPではソフト削除なし）

#### `validateBookmarkData(data: BookmarkInput): ValidationResult`
- **目的**: 送信前にブックマーク入力を検証
- **入力**: 検証するブックマークデータ
- **出力**: エラーがある場合の検証結果
- **備考**: 詳細なバリデーションルールは機能設計フェーズで定義

#### `getCachedBookmarks(): Bookmark[]`
- **目的**: ローカルキャッシュされたブックマークを取得
- **入力**: なし
- **出力**: React Contextからのキャッシュされたブックマーク
- **備考**: 楽観的UI更新に使用

#### `updateLocalCache(bookmarks: Bookmark[] | Bookmark): void`
- **目的**: 新しいブックマークデータでReact Contextを更新
- **入力**: 単一ブックマークまたはブックマークの配列
- **出力**: なし（Reduxコンテキストを更新）
- **備考**: UI再レンダリングをトリガー

---

### SearchFeature

#### `search(query: string, bookmarks: Bookmark[]): Bookmark[]`
- **目的**: クライアント側でブックマークコレクションに検索を実行
- **入力**: 検索クエリ文字列と検索対象ブックマーク
- **出力**: クエリと一致したブックマーク配列
- **備考**: クライアント側実装；タイトルとURLを検索

#### `filter(bookmarks: Bookmark[], criteria: FilterCriteria): Bookmark[]`
- **目的**: ブックマークリストにフィルター条件を適用
- **入力**: ブックマークとフィルター条件
- **出力**: フィルター済みブックマーク配列
- **備考**: 現在サポート穋度なし；機能設計フェーズを目指したタグ/カテゴリフィルター

#### `clearSearch(): void`
- **目的**: 検索クエリをリセットしてすべてのブックマークを表示
- **入力**: なし
- **出力**: 検索状態をリセット
- **備考**: UIを更新して完全リストを表示

#### `isSearchActive(): boolean`
- **目的**: 検索が現在アクティブかどうかをチェック
- **入力**: なし
- **出力**: ブーリアンフラグ
- **備考**: 検索インジケーターを表示/非表示するために使用

#### `getSearchResults(): Bookmark[]`
- **目的**: 現在の検索結果を取得
- **入力**: なし
- **出力**: 現在表示されている検索結果
- **備考**: React Contextから

---

### ページコンポーネント

#### LoginPage

##### `handleLogin(username: string, password: string): Promise<void>`
- **目的**: ログインフォーム送信を処理
- **入力**: ユーザーフォームからの認証情報
- **出力**: MFAページまたはホームページへのナビゲーション
- **備考**: エラー処理は機能設計フェーズで定義

##### `handleForgotPassword(): void`
- **目的**: パスワードリセットフローにナビゲート
- **入力**: なし
- **出力**: パスワードリセットダイアログ/ページを開く
- **備考**: 機能設計フェーズで定義予定

---

#### HomePage

##### `loadBookmarks(): Promise<void>`
- **目的**: ページ読み込み時にブックマークを取得
- **入力**: ユーザーセッション
- **出力**: コンテキストを更新
- **備考**: ローディング状態管理は機能設計フェーズで定義

##### `handleSearchInput(query: string): void`
- **目的**: 検索フィールドを更新し結果をフィルタ
- **入力**: 検索入力からのクエリ文字列
- **出力**: フィルター済みブックマークリストを更新
- **備考**: ユーザー入力時のリアルタイム検索

##### `handleBookmarkClick(bookmarkId: string): void`
- **目的**: ブックマーク詳細ページにナビゲート
- **入力**: クリックされたブックマークID
- **出力**: ナビゲーションアクション
- **備考**: 機能設計フェーズで定義予定

##### `handleEditBookmark(bookmarkId: string): void`
- **目的**: 編集ページにナビゲート
- **入力**: ブックマークID
- **出力**: 編集ページへのナビゲーション
- **備考**: 機能設計フェーズで定義予定

##### `handleDeleteBookmark(bookmarkId: string): Promise<void>`
- **目的**: 確認付きでブックマークを削除
- **入力**: ブックマークID
- **出力**: リストからブックマークを削除
- **備考**: 削除前に確認ダイアログを表示

##### `handleCreateNew(): void`
- **目的**: 新規ブックマーク作成ページにナビゲート
- **入力**: なし
- **出力**: 作成ページへのナビゲーション
- **備考**: 機能設計フェーズで定義予定

---

#### BookmarkDetailPage

##### `loadBookmark(bookmarkId: string): Promise<void>`
- **目的**: 単一ブックマークエクスウギを読み込み
- **入力**: ルートパラメータからのブックマークID
- **出力**: 詳細視でブックマークを表示
- **備考**: 機能設計フェーズで定義予定

##### `handleEdit(): void`
- **目的**: 編集ページにナビゲート
- **入力**: なし（ブックマーク作成済み）
- **出力**: 編集ページへのナビゲーション
- **備考**: 機能設計フェーズで定義予定

##### `handleDelete(bookmarkId: string): Promise<void>`
- **目的**: 確認二名後にブックマークを削除
- **入力**: ブックマークID
- **出力**: ブックマーク削除コンプリート、ホームへのナビゲーション
- **備考**: 事前確認ダイアログ表示

---

#### CreateBookmarkPage

##### `handleCreateSubmit(formData: BookmarkInput): Promise<void>`
- **目的**: 新規ブックマークフォームを送信
- **入力**: ユーザーからのフォームデータ
- **出力**: 新規ブックマーク作成、詳細視/ホームへのナビゲーション
- **備考**: 送信前のバリデーション

##### `validateForm(formData: BookmarkInput): ValidationError[]`
- **目的**: 送信前にフォームを検証
- **入力**: フォーム入力データ
- **出力**: バリデーションエラーの配列（有効の場合は空）
- **備考**: 詳細なルールは機能設計フェーズで定義

##### `handleCancel(): void`
- **目的**: フォームを粗梨して戻る
- **入力**: なし
- **出力**: ホームへのナビゲーション
- **備考**: フォーム変更がある場合提供

---

#### EditBookmarkPage

##### `loadBookmark(bookmarkId: string): Promise<void>`
- **目的**: 編集用ブックマークデータを読み込み
- **入力**: ルートパラメータからのブックマークID
- **出力**: ブックマークデータで事前入力フォームを埋める
- **備考**: 機能設計フェーズで定義予定

##### `handleUpdateSubmit(formData: BookmarkInput): Promise<void>`
- **目的**: 更新されたブックマークを送信
- **入力**: 更新されたフォームデータ
- **出力**: ブックマーク更新、詳細視へのナビゲーション
- **備考**: 楽観的ロック並行制御の競合を処理

##### `detectConcurrentEdit(): boolean`
- **目的**: ブックマークが他のセッションで編集されたかどうかを検查
- **入力**: 現在のバージョン的サーバーバージョン
- **出力**: ブーリアン（競合検出場合、真）
- **備考**: 楽観的ロックを使ったバージョン比較

##### `handleConflict(serverVersion: Bookmark): void`
- **目的**: 並行編集競合を処理
- **入力**: サーバーの現在ブックマークバージョン
- **出力**: 競合解決UIを表示
- **備考**: 詳細な競合解決戦略は機能設計フェーズで定義

---

### Lambda Handler（バックエンド処理）

#### `handleRequest(event: APIGatewayEvent): Promise<APIGatewayResponse>`
- **目的**: API Gatewayリクエストのメインエントリーポイント
- **入力**: HTTPリクエストイベント
- **出力**: JSONレスポンス
- **備考**: メソッド + パスに基づき適入なハンドラーにルート事後

#### `handleCreateBookmark(userId: string, input: BookmarkInput): Promise<Bookmark>`
- **目的**: データベース内ブックマークを作成
- **入力**: ユーザー ID、ブックマークデータ
- **出力**: システムフィールド付きブックマーク作成されたもの
- **備考**: データベーストランザクション；詳細ビジネスロジックは機能設計フェーズで定義

#### `handleGetBookmarks(userId: string): Promise<Bookmark[]>`
- **目的**: データベースからすべてのユーザーブックマークを取得
- **入力**: ユーザー ID
- **出力**: ブックマーク配列
- **備考**: ページネーションパラメータ処理は機能設計フェーズで定義

#### `handleGetBookmark(userId: string, bookmarkId: string): Promise<Bookmark>`
- **目的**: データベースから単一ブックマークを取得
- **入力**: ユーザー ID、ブックマーク ID
- **出力**: ブックマークオブジェクト
- **備考**: ユーザーがブックマークを醨有していることを検証

#### `handleUpdateBookmark(userId: string, bookmarkId: string, updates: BookmarkInput, version: number): Promise<Bookmark>`
- **目的**: 楽観的ロック付きブックマーク更新
- **入力**: ユーザー ID、ブックマーク ID、更新内容、現在バージョン
- **出力**: 更新されたブックマークまたは競合エラー
- **備考**: 並行編集検出のためのバージョン確認

#### `handleDeleteBookmark(userId: string, bookmarkId: string): Promise<void>`
- **目的**: データベースを削除
- **入力**: ユーザー ID、ブックマーク ID
- **出力**: 削除確認
- **備考**: ユーザー所有権を検証

#### `verifyUserToken(token: string): Promise<TokenPayload>`
- **目的**: JWTトークンを検証しデコード
- **入力**: Authorizationヘッダーからの JWT
- **出力**: userIdを含むトークンペイロード
- **備考**: トークン検証と有効期限確認

#### `openDatabaseConnection(): Promise<DatabaseConnection>`
- **目的**: SQLiteデータベースへの接続を設立
- **入力**: なし
- **出力**: データベース接続オブジェクト（Lambda Layer内）
- **備考**: Lambdaコンテナを使ったリクエスト間で再利用

#### `closeDatabaseConnection(connection: DatabaseConnection): void`
- **目的**: データベース接続を閉じる
- **入力**: 接続オブジェクト
- **出力**: なし
- **備考**: リクエスト終了時のクリーンアップ

---

## 統計サマリー

| コンポーネント種別 | 個数 | メソッド数 |
|---|---|---|
| 機能層 | 3 | 15 |
| ページ層 | 5 | 20 |
| バックエンド | 1 | 10 |
| データ層 | 2 | 0 |
| **合計** | **11** | **45** |

**備考**: 詳細な実装ロジック（バリデーション、エラーハンドリング、データベース操作）は機能設計ステージ（ユニット単位、CONSTRUCTION フェーズ）で指定されます。
