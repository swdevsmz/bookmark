# Unit of Work 定義

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Units Generation  
**作成日**: 2026-03-01

---

## ユニット概要

このプロジェクトは **3つの独立したユニット** に分解されます。各ユニットは独立してデプロイ可能で、Monorepo 構造（packages/配下）で管理されます。

---

## Unit 1: Frontend Service

### 概要
ユーザーインターフェース、ユーザーインタラクション、クライアント側状態管理に責任を持ちます。

### 責務
- ユーザーアウトフローのUI実装（ログイン、MFAセットアップ、ブックマーク管理）
- クライアント側の検索・フィルタリング
- クライアント側の状態管理（React Context + useReducer）
- ネットワークエラーハンドリングと再試行ロジック
- ローカル入力バリデーション

### 含まれるコンポーネント（Application Design より）
**ページ層**:
- LoginPage
- MFASetupPage
- HomePage
- BookmarkDetailPage
- CreateBookmarkPage
- EditBookmarkPage

**機能層**:
- BookmarkManagementFeature
- SearchFeature

**サービス層**:
- State Management Service（useBookmarkState, useSearchState, useUIState hooks）
- Validation Service

**インターフェース/ツール**:
- React Context API
- useReducer pattern
- React Router (SPA navigation)

### 外部依存関係
- **Backend Service** API（REST via API Gateway）- ブックマーク CRUD, ログイン検証
- **Authentication Service** API（REST via API Gateway）- MFA検証, セッション管理

### デプロイ
- **環境**: S3 + CloudFront + Lambda@Edge (SPA)
- **ビルド**: React TypeScript → static HTML/JS/CSS
- **配布**: CDN

### チーム
フロントエンドエンジニア（React/TypeScript専門）

---

## Unit 2: Backend Service

### 概要
ビジネスロジック、CRUD操作、データ永続化に責任を持ちます。

### 責務
- ブックマークの Create, Read, Update, Delete（CRUD）操作
- 競合制御（optimistic locking with version fields）
- キャッシング戦略と最適化
- バックエンド検証ロジック
- エラー処理とログ記録
- データベース操作
- S3 による バックアップ・アーカイブ

### 含まれるコンポーネント（Application Design より）
**バックエンド処理**:
- Lambda Handler（single handler で全CRUD操作をルーティング）

**サービス層**:
- API Service（AWS SDK v3 で HTTP communication）
- Data Persistence Service（CRUD 抽象化, caching, retry logic）
- Error Handling Service（エラー変換, ユーザーフレンドリーメッセージ）

**データストレージ**:
- SQLite Database（Lambda Layers 内）
- S3 Storage（バックアップ・アーカイブ）

### 外部依存関係
- **Frontend Service**（クライアント）via REST API
- **Authentication Service**（トークン検証）via システム間呼び出し
- AWS Service: Lambda, API Gateway, RDS/SQLite, S3 IAM

### デプロイ
- **環境**: AWS Lambda + API Gateway
- **コード**: TypeScript/Node.js
- **トリガー**: API Gateway events
- **依存**: Lambda Layers (SQLite driver)

### チーム
バックエンドエンジニア（Node.js/AWS専門）

---

## Unit 3: Authentication Service

### 概要
AWS Cognito との統合、MFA管理、セッション管理に責任を持ちます。

### 責務
- ユーザー認証（ログイン / ログアウト）
- MFA チャレンジの処理（SMS/TOTP/Email）
- MFA設定の管理
- セッショントークンの発行・検証・リフレッシュ
- パスワードリセット
- トークンの有効期限管理
- JWT ベースのセッション管理

### 含まれるコンポーネント（Application Design より）
**フロントエンド機能**:
- AuthFeature（AWS Cognito SDK 呼び出し）

**サービス層**:
- Authentication Service（セッション管理）
- API Service（HTTP for Cognito calls）

**外部サービス**:
- AWS Cognito（ユーザー認証, MFA）

### 外部依存関係
- **Frontend Service**（認証UI）via REST API
- **Backend Service**（トークン検証）via システム間呼び出し
- AWS Service: Cognito, Lambda, API Gateway

### デプロイ
- **環境**: AWS Lambda + API Gateway
- **コード**: TypeScript/Node.js
- **トリガー**: API Gateway events
- **Cognito設定**: 外部管理（ユーザープール, アプリクライアント）

### チーム
認証・セキュリティエンジニア（AWS Cognito専門）

---

## クロスユニット考慮事項

### API契約
- **Frontend ↔ Backend / Auth**: REST API（JSON）
- **Backend ↔ Cognito**: SDK via Lambda
- **版管理**: Semantic Versioning for API contracts

### 共有関心事
- **Error Response Format**: 統一フォーマット
- **Logging**: 構造化ログ（CloudWatch）
- **Monitoring**: CloudWatch Metrics / X-Ray tracing
- **Security**: JWT validation, CORS, rate limiting

### 環境設定
- **開発環境**: ローカル + mocked services
- **ステージング環境**: AWS アカウント（本番準備環境）
- **本番環境**: 厳密なゲートキーパー + CloudTrail

---

## コード組織戦略（Monorepo）

```
bookmark/  (workspace root)
├── packages/
│   ├── frontend/
│   │   ├── src/
│   │   │   ├── pages/
│   │   │   ├── features/
│   │   │   ├── services/
│   │   │   └── shared/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── backend/
│   │   ├── src/
│   │   │   ├── handlers/
│   │   │   ├── services/
│   │   │   ├── db/
│   │   │   └── utils/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── auth/
│       ├── src/
│       │   ├── handlers/
│       │   ├── cognito/
│       │   └── utils/
│       ├── package.json
│       └── tsconfig.json
├── aidlc-docs/
├── .github/
├── package.json  (workspace root)
├── tsconfig.json (base config)
└── .gitignore
```

---

## ユニット優先度と開発順序

1. **Unit 1: Backend Service** （基盤）
   - ビジネスロジックが他のユニットに依存
   - API コントラクト定義手順

2. **Unit 2: Authentication Service** （平行開発可能）
   - Backend Service への依存なし
   - スタンドアロン実装可能

3. **Unit 3: Frontend Service** （最後）
   - Backend + Auth サービスに依存
   - 統合テストが必要

---

## まとめ

| ユニット | 責務 | チーム | 依存関係 | デプロイ頻度 |
|---------|------|--------|---------|------------|
| Frontend | UI/状態管理 | FE エンジニア | Backend, Auth | 週1-2回 |
| Backend | ビジネスロジック | BE エンジニア | (独立) | 週1-2回 |
| Auth | 認証/MFA | セキュリティ | Cognito | 月1回 |

---
