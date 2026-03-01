# Application Design Plan

**プロジェクト**: ブックマーク管理システム  
**フェーズ**: INCEPTION - Application Design  
**作成日**: 2026-03-01

---

## 1) 設計方針（要件から確定済み）

- アーキテクチャ: サーバーレス（S3 + CloudFront + API Gateway + Lambda + Cognito）
- 認証: AWS Cognito + MFA（SMS/TOTP/Email）
- データ: SQLite（Lambda Layers）+ S3永続化
- 機能スコープ: ブックマークCRUD、基本検索（タイトル/URL）
- 期間/コスト: 1-2週間、月額$1-5目標

---

## 2) Application Design 実行チェックリスト

### 2.1 設計計画
- [ ] 要件を再確認し、コンポーネント境界を確定する
- [ ] 各コンポーネントの責務を定義する
- [ ] サービス層のオーケストレーション方針を定義する
- [ ] 依存関係と通信パターンを定義する

### 2.2 必須成果物（MANDATORY）
- [ ] `aidlc-docs/inception/application-design/components.md` を生成する
- [ ] `aidlc-docs/inception/application-design/component-methods.md` を生成する
- [ ] `aidlc-docs/inception/application-design/services.md` を生成する
- [ ] `aidlc-docs/inception/application-design/component-dependency.md` を生成する
- [ ] 設計の整合性（責務重複・循環依存・抜け漏れ）を検証する

### 2.3 完了条件
- [ ] 全コンポーネントに責務と境界がある
- [ ] 全主要ユースケース（CRUD、検索、認証、MFA）に対応メソッドがある
- [ ] サービス層の責務分離が明確
- [ ] 依存関係が一方向で説明可能

---

## 3) 設計判断のための質問（回答必須）

以下の各質問に、`[Answer]:` の後ろへ **選択肢の文字**（A/B/C...）を記入してください。  
当てはまらない場合は **X (Other)** を選んで内容を追記してください。

## Question 1
フロントエンドのコンポーネント構成はどれで進めますか？

A) `pages` + `features` + `shared/ui` の3層（機能単位で整理）
B) `pages` + `components` のシンプル2層（MVP優先）
C) Atomic Design（atoms/molecules/organisms）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 2
フロントエンドの状態管理はどれで進めますか？

A) React標準（Context + useReducer、必要最小限）
B) TanStack Query + ローカルstate（サーバーstate中心）
C) Redux Toolkit（明示的なグローバル管理）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 3
Cognito連携ライブラリはどれで進めますか？

A) AWS Amplify Auth を使用（実装速度優先）
B) AWS SDK v3 を直接利用（依存最小化）
C) ハイブリッド（基本はAmplify、特殊処理のみSDK）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 4
Lambda関数の分割方針はどれで進めますか？

A) 単一Lambda（bookmark-handler）で全CRUDを処理
B) CRUDごとに分割（create/read/update/delete/search）
C) 認証系と業務系で2分割
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 5
SQLite同時更新競合（RISK-001）への初期対策はどれで進めますか？

A) 楽観的ロック（versionカラム/更新時チェック）
B) S3オブジェクトロック的制御（更新前後のETag確認）
C) MVPは最小対策（リトライ + 競合時エラー返却）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 6
インフラ定義ツールはどれで進めますか？

A) AWS SAM（シンプル、MVP向け）
B) AWS CDK（拡張性重視）
C) Terraform（マルチクラウド視野）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## 4) 回答後にこちらで実施する内容

- [x] 回答内容の矛盾/曖昧さを分析（2026-03-01 実施、重大な矛盾なし）
- [x] 必要時のみフォローアップ質問を追加（今回は不要）
- [ ] Application Design 成果物4点を生成
- [ ] レビュー依頼（承認ゲート）
