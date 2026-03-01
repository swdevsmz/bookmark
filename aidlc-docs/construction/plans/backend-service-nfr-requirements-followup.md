# Backend Service - NFR Requirements Follow-up Questions

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-01  
**目的**: NFR 要件の曖昧な部分を明確化

---

## Answer Validation Summary

全体的に回答ありがとうございます。8つの質問のうち、以下の2点について、設計を確定するために追加の明確化が必要です：

---

## Follow-up Question 1: Performance Requirement Detail (Q2 関連)

**ユーザー回答**: 「ColdStartは許容、レスポンスタイムは１秒」

**明確にしたい点**:

レスポンスタイムの「1秒」について、以下のどちらペースで理解すればよいでしょうか？

**A)** 「1秒」は**全操作**（LIST/GET/CREATE/UPDATE/DELETE/SEARCH）に適用される統一的な目標値

**B)** 「1秒」は**平均的な目安**で、以下のように操作によって異なる:
- LIST（一覧取得）: 500ms
- GET（詳細取得）: 300ms
- CREATE/UPDATE: 1秒
- SEARCH（検索）: 1.5秒
- DELETE: 500ms

**C)** 「1秒」は**Cold Start を含めた許容値**で、実際の処理は Cold Start 時間を除いて計測した場合、どれくらい許容していますか？

[Answer]: 検索系は1秒以内。他の操作（LIST/GET/CREATE/UPDATE/DELETE）は1秒以上かかってもいい。Cold Startで遅くなってもいい

---

## Follow-up Question 2: SQLite Database Storage Location (Q3 関連)

**ユーザー回答**: 「SQLiteはLambdaデプロイ時に消えない場所、かつ無料枠」

**明確にしたい点**:

データベースの配置についてです。以下のどの構成を想定していますか？

**A)** **Lambda Layers にバンドル**: SQLite ファイルを Lambda Layers として含める
- メリット: 無料、デプロイ時に更新可能
- デメリット: 読み取り専用、書き込みには別途 /tmp 使用が必要

**B)** **Lambda /tmp ディレクトリ**: Lambda コンテナの /tmp にファイルを保存
- メリット: 読み書き可能、無料
- デメリット: Lambda 実行終了後に削除される（次回起動時は新規作成）

**C)** **AWS EFS**: Elastic File System を使用
- メリット: 読み写き可能、persistent
- デメリット: 有料フレンティア外（月額費用10ドル程度）

**D)** **Amazon S3**: S3 にデータベースファイルを保存、各 Lambda 実行時に /tmp にコピー
- メリット: persistent、無料枠あり
- デメリット: 各リクエストで読み込みが必要（レイテンシ増加）

[Answer]: **B) Lambda /tmp ディレクトリ** - Lambda コンテナの /tmp にデータベースファイルを保存（読み書き可能、無料）

---

## Next Steps

1. 上記2つの Follow-up Questions に回答してください
2. 回答後、NFR アーティファクトを生成します
3. その後、PR を作成します

