# Backend Service - Deployment Architecture

**プロジェクト**: ブックマーク管理システム  
**ユニット**: Backend Service  
**作成日**: 2026-03-03  
**ステージ**: Construction - Infrastructure Design

---

## 1. High-Level Diagram

```mermaid
flowchart TB
    subgraph Internet
        client[ユーザー (ブラウザ/SPA)]
    end

    client -->|HTTPS| apigw[API Gateway]
    apigw -->|Invoke| lambda[Backend Lambda]
    lambda --> ebs[EBS Volume (/tmp/bookmark.db)]
    lambda --> cognito[Cognito User Pool]
    lambda --> cloudwatch[CloudWatch Logs/Metrics]
    click apigw "https://docs.aws.amazon.com/apigateway/" "API Gateway Docs"
    click lambda "https://docs.aws.amazon.com/lambda/" "AWS Lambda Docs"
    click ebs "https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/ebs-volumes.html" "EBS Volumes"

    style apigw fill:#f9f,stroke:#333,stroke-width:1px
    style lambda fill:#ff9,stroke:#333,stroke-width:1px
    style ebs fill:#9ff,stroke:#333,stroke-width:1px

```

> **Note**: EBS ボリュームは Lambda にアタッチされ、永続的なストレージを提供します。API Gateway はリソースベースで各メソッドを Lambda に統合。

## 2. Resource Mapping

| Logical Component | AWS Service | Configuration Highlights |
|-------------------|-------------|--------------------------|
| HTTP Layer        | API Gateway (REST) | `/bookmarks`, `/bookmarks/{id}` 、Cognito オーソライザー、レート制限 |
| Compute           | AWS Lambda | 128MB, 10s, Layers, EBS アタッチ |
| Data Persistence  | EBS Volume + SQLite `/tmp` | gp3 10GB, スナップショットでバックアップ |
| Authentication    | AWS Cognito User Pool | JWT 検証、`sub` を Lambda で使用 |
| Monitoring        | CloudWatch Logs & Metrics | 30日ログ、カスタムメトリクス、エラーアラート |
| IaC & State       | Terraform (local state) | ローカル `terraform.tfstate` 管理 |

## 3. Deployment Steps Overview

1. Write Terraform configuration for API Gateway, Lambda, IAM, EBS, CloudWatch Alarms.
2. Initialize SQLite schema SQL file (`schema.sql`) bundled with Lambda package.
3. Package Lambda code into zip with Layers.
4. Terraform apply (local state) to create resources.
5. Lambda startup script mounts EBS volume, initializes DB if missing, then serves requests.

---

*Deployment architecture defines how the logical design is realized in AWS and provides guidance for implementation and operations.*
