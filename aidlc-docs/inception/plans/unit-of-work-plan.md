# Unit of Work 計画

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Units Generation  
**作成日**: 2026-03-01

---

## 分解計画チェックリスト

### パート 1: 推奨ユニット構造の提案
- [ ] ブックマークシステムの論理的ユニット分解を分析
- [ ] 各ユニットの責務と境界を定義
- [ ] ユニット間の依存関係を特定

### パート 2: 必須成果物の生成
- [ ] `unit-of-work.md` - ユニット定義と責務
- [ ] `unit-of-work-dependency.md` - 依存関係マトリックス
- [ ] `unit-of-work-story-map.md` - ストーリー割り当て
- [ ] ユニット境界と依存関係の検証
- [ ] すべてのコンポーネントをユニットに割り当て

---

## 設計ベースのユニット分解

現在の Application Design に基づく推奨分解：

### **提案 A: 3ユニット分解（推奨）**

#### Unit 1: Frontend Service
- **責務**: ユーザーインターフェース、ユーザーインタラクション、クライアント side state management
- **含まれるコンポーネント**: 
  - AuthFeature, BookmarkManagementFeature, SearchFeature
  - LoginPage, MFASetupPage, HomePage, BookmarkDetailPage, CreateBookmarkPage, EditBookmarkPage
  - State Management Service, Validation Service
- **依存関係**: API Service → Backend Service
- **展開**: CloudFront + S3 + Lambda@Edge (静的資産と SPA)

#### Unit 2: Backend Service (API + Business Logic)
- **責務**: CRUD操作、ビジネスロジック、データ永続化、認証統合
- **含まれるコンポーネント**:
  - Lambda Handler
  - API Service, Data Persistence Service, Error Handling Service
  - SQLite DB, S3 Storage
- **依存関係**: Cognito (認証), DynamoDB/RDS (オプション)
- **展開**: Lambda + API Gateway + RDS/SQLite

#### Unit 3: Authentication Service
- **責務**: Cognito統合, MFA管理, セッション管理
- **含まれるコンポーネント**:
  - AuthFeature (AWS Cognito SDK)
  - Authentication Service
- **依存関係**: AWS Cognito
- **展開**: Lambda@Edge (認証ミドルウェア)

---

## 質問フォーム

### Q1: ユニット分解の観点
「提案 A（3ユニット分解）」に対して、ご意見をお聞かせください：

a) 3ユニット分解（Frontend / Backend / Auth）で適切か  
b) 2ユニット分解（Frontend / Backend）に統合したい  
c) 別の分解方法を提案したい

[Answer]: a

---

### Q2: チーム構成と責任分離
ユニット間の責任分離とチーム構成について：

a) フロントエンド・バックエンド・認証で独立したチーム編成  
b) フロントエンド・バックエンド 2チームで、認証はバックエンド内に含める  
c) 単一チームで全ユニットを開発

[Answer]: a

---

### Q3: コード組織戦略（Greenfield）
ユニット間のコード組織について（推奨）:

a) Monorepo（packages/ で各ユニット管理）  
  ```
  packages/
    ├── frontend/
    ├── backend/
    └── auth/
  ```

b) マルチリポジトリ（repo/unit-name で分離）  
  ```
  bookmark-frontend/
  bookmark-backend/
  bookmark-auth/
  ```

c) 単一リポジトリ内のディレクトリ分離  
  ```
  src/
    ├── frontend/
    ├── backend/
    └── auth/
  ```

[Answer]: a

---

### Q4: 依存関係と統合方法
ユニット間の通信と依存関係：

a) REST API のみ（Frontend ↔ API Gateway ↔ Backend）  
b) GraphQL のサポートも検討  
c) Event-driven architecture（SNS/SQS）も検討

[Answer]: a

---

### Q5: デプロイ戦略
各ユニットのデプロイ独立性：

a) 各ユニットを独立してデプロイ可能（推奨 for microservices）  
b) 全ユニットを同時にデプロイ  
c) 段階的なデプロイ（依存関係に基づく順序制御）

[Answer]: a

---

## 次のステップ

すべての [Answer]: タグに記入してください。その後、以下が実行されます：

1. ユーザー回答の分析（曖昧性チェック）
2. 以下の成果物を生成：
   - unit-of-work.md
   - unit-of-work-dependency.md
   - unit-of-work-story-map.md
3. ユニット境界と依存関係の検証
4. Units Generation の実行承認

---
