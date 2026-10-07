# AWS Multi-Region Infrastructure Portfolio

AWS上に構築した、Web/API・Docker・プライベートネットワーク・イベント駆動処理・監視・VPN・マルチリージョン通信を組み合わせたクラウドインフラのポートフォリオです。

AWSマネジメントコンソールおよびAWS CLIを使用して実環境を構築・検証し、TerraformによるIaC管理についても検証しています。

---

## Architecture

![AWS Architecture](diagrams/architecture.png)

### High-Level Architecture

```text
Internet
   |
   v
Application Load Balancer
   |
   v
Web EC2
Private Subnet
Docker / Node.js
   |
   v
API EC2
Private Subnet
Docker / Node.js
   |
   +--------------------> Amazon S3
   |
   +---- VPC Peering ----> Tokyo VPC
                           |
                           v
                    Minutes Analytics API


Employee PC
   |
OpenVPN
   |
   v
VPN EC2
   |
   v
Private AWS Resources


Administrator
   |
   v
AWS Systems Manager
   |
   v
VPC Endpoints
   |
   v
Private EC2


Amazon S3
   |
   v
AWS Lambda

EventBridge Scheduler
   |
   v
AWS Lambda

AWS CloudTrail
   |
   v
Amazon CloudWatch
```

---

## Architecture Overview

本環境は大阪リージョンをメイン環境とし、東京リージョンの別システムとInter-Region VPC Peeringで接続しています。

WebサーバーおよびAPIサーバーはPrivate Subnetに配置し、インターネットから直接アクセスできない構成としています。

外部ユーザーからのWebアクセスはApplication Load Balancerを経由します。

```text
Internet
    |
    v
ALB
    |
    v
Web EC2
    |
    v
API EC2
    |
    +----> S3
    |
    +----> Tokyo Minutes Analytics API
```

---

## AWS Services

主に以下のAWSサービスを使用しています。

| Category | AWS Service |
|---|---|
| Network | Amazon VPC |
| Compute | Amazon EC2 |
| Load Balancing | Application Load Balancer |
| Storage | Amazon S3 |
| Container Registry | Amazon ECR |
| Management | AWS Systems Manager |
| Serverless | AWS Lambda |
| Scheduling | Amazon EventBridge Scheduler |
| Monitoring | Amazon CloudWatch |
| Audit | AWS CloudTrail |
| Private Access | VPC Endpoint |
| Multi-Region | VPC Peering |
| Infrastructure as Code | Terraform |

---

## Network Design

### Osaka Region

メイン環境は大阪リージョンに構築しています。

```text
VPC
10.10.0.0/16

├── Public Subnet 1
│   10.10.0.0/20
│
├── Public Subnet 2
│   10.10.16.0/20
│
├── Private Subnet 1
│   10.10.128.0/20
│
└── Private Subnet 2
    10.10.144.0/20
```

Public SubnetとPrivate Subnetを2つのAvailability Zoneに分散しています。

Web EC2およびAPI EC2はPrivate Subnetに配置しています。

---

## Web / API Architecture

### Web Server

Webアプリケーションは以下の構成です。

- Amazon EC2
- Private Subnet
- Docker
- Node.js
- Amazon ECR
- Port 3000

インターネットからWeb EC2へ直接アクセスさせず、Application Load Balancer経由でアクセスします。

```text
Internet
   |
   v
ALB :80
   |
   v
Web EC2 :3000
```

### API Server

APIサーバーもPrivate Subnetに配置しています。

- Amazon EC2
- Docker
- Node.js
- Amazon ECR
- Port 3001

通信経路は以下のように制御しています。

```text
ALB
 |
 v
Web EC2
 |
 v
API EC2
```

Security Groupを使用し、APIへのアクセス元をWebサーバー側に制限しています。

---

## Amazon ECR / Docker

WebおよびAPIアプリケーションをDockerコンテナとして構築し、Amazon ECRでイメージを管理しています。

```text
Docker Image
     |
     v
Amazon ECR
     |
     v
Private EC2
     |
     v
Docker Container
```

Web用とAPI用のECR Repositoryを分離しています。

---

## Private AWS Access

Private Subnet上のEC2からAWSサービスへアクセスするため、VPC Endpointを使用しています。

使用しているEndpoint：

- AWS Systems Manager
- AWS Systems Manager Messages
- Amazon ECR API
- Amazon ECR Docker Registry
- Amazon S3 Gateway Endpoint

これにより、NAT Gatewayへ常時依存せずAWSサービスへプライベートにアクセスできる構成を検証しました。

---

## AWS Systems Manager

EC2の管理にはAWS Systems Manager Session Managerを使用しています。

SSHポート22をインターネットへ公開せず、IAMベースでEC2へアクセスします。

```text
Administrator
      |
      v
AWS Systems Manager
      |
      v
VPC Endpoint
      |
      v
Private EC2
```

---

## Multi-Region VPC Peering

大阪リージョンと東京リージョンをInter-Region VPC Peeringで接続しています。

```text
Osaka Region
10.10.0.0/16
      |
      |
Inter-Region
VPC Peering
      |
      |
Tokyo Region
10.0.0.0/24
```

大阪側APIから、東京リージョンのMinutes Analytics APIへPrivate IPで通信します。

これにより、インターネットを経由しないリージョン間通信を検証しました。

---

## OpenVPN

社員端末からAWS内部システムへアクセスする経路として、OpenVPN Access Serverを構築しました。

```text
Employee PC
     |
     v
OpenVPN
     |
     v
VPN EC2
     |
     v
Private AWS Resources
```

VPN接続後、Private Subnet上のWeb/APIサーバーへのアクセスを検証しています。

---

## Serverless / Event Driven Architecture

### S3 + Lambda

Amazon S3へのJSONファイルアップロードをトリガーとしてAWS Lambdaを実行するイベント駆動処理を構築しました。

```text
JSON Upload
     |
     v
Amazon S3
     |
     v
AWS Lambda
     |
     v
CloudWatch Logs
```

### EventBridge Scheduler + Lambda

Amazon EventBridge SchedulerからLambdaを実行する構成も検証しています。

```text
EventBridge Scheduler
        |
        v
     Lambda
        |
        v
CloudWatch Logs
```

また、大阪リージョンのLambdaからVPC Peering経由で東京リージョンのMinutes Analytics APIへアクセスする構成も検証しています。

---

## Monitoring / Audit

運用監視・監査には以下を使用しています。

- Amazon CloudWatch
- AWS CloudTrail
- CloudWatch Alarm
- CloudWatch Logs

Security Group変更イベントをCloudTrailで記録し、CloudWatchで監視する構成を作成しました。

```text
AWS API Operation
       |
       v
AWS CloudTrail
       |
       v
CloudWatch Logs
       |
       v
Metric Filter
       |
       v
CloudWatch Alarm
```

---

## Security Design

セキュリティ面では以下を意識しています。

- Web/API EC2をPrivate Subnetへ配置
- SSH/RDPをインターネットへ非公開
- AWS Systems Manager Session ManagerによるEC2管理
- Security Groupによる通信経路制御
- IAM RoleによるAWSサービスアクセス
- VPC Endpointによるプライベート通信
- S3 Block Public Access
- EC2 IMDSv2
- AWS CloudTrailによる操作監査
- Terraform専用IAM Role
- rootユーザーをTerraform実行に使用しない

---

## Terraform / Infrastructure as Code

既存AWS環境をTerraformで管理できるよう、主要なインフラリソースをTerraformコードとして定義しています。

対象には以下が含まれます。

- VPC
- Subnet
- Route Table
- Internet Gateway
- Security Group
- VPC Endpoint
- Application Load Balancer
- Target Group
- Amazon S3
- Amazon ECR
- VPC Peering

Terraformの初期化および構文検証も実施しています。

```bash
terraform init
terraform validate
```

Terraform実行時はrootユーザーを使用せず、専用IAMユーザーからTerraform用IAM RoleをAssumeする構成としています。

```text
Terraform
    |
    v
Terraform IAM User
    |
    | AssumeRole
    v
Terraform IAM Role
    |
    v
AWS Resources
```

### Terraform Import

本プロジェクトでは、既存AWS環境をTerraform管理へ移行するためのimport定義も作成しています。

`terraform/imports.tf` に既存リソースとTerraform Resourceの対応を定義しています。

> このリポジトリは既存AWS環境のTerraformコード化およびimport設計を扱っています。Terraformから新規AWS環境を0から構築するプロジェクトは別ポートフォリオとして実施予定です。

---

## Terraform Validation

実施済み：

```text
terraform init
    ↓
Success

terraform validate
    ↓
Success! The configuration is valid.
```

---

## Cost Optimization

学習・検証環境のため、不要なAWSリソースを継続稼働させないことを意識しています。

NAT GatewayはDocker / Node.js等の導入時に一時的に使用し、VPC Endpointによる通信経路を構築した後に削除しました。

検証を行わない期間はEC2を停止するなど、コストを抑えながら構築・検証しています。

---

## Repository Structure

```text
aws-multiregion-portfolio/
│
├── diagrams/
│   └── architecture.png
│
├── terraform/
│   ├── .terraform.lock.hcl
│   ├── alb.tf
│   ├── ec2.tf
│   ├── ecr.tf
│   ├── endpoints.tf
│   ├── eventbridge.tf
│   ├── imports.tf
│   ├── lambda.tf
│   ├── network.tf
│   ├── outputs.tf
│   ├── peering.tf
│   ├── providers.tf
│   ├── s3.tf
│   ├── security_groups.tf
│   ├── variables.tf
│   └── versions.tf
│
├── .gitignore
└── README.md
```

Terraform State、認証情報、秘密鍵、VPN ProfileなどはGit管理対象から除外しています。

---

## What I Learned

このプロジェクトを通じて、以下を実際に構築・検証しました。

- VPC / Subnet / Route Table設計
- Public / Private Subnet分離
- Multi-AZ構成
- Security Groupによる通信制御
- Application Load Balancer
- Docker / Node.js
- Amazon ECR
- Amazon S3
- VPC Endpoint
- AWS Systems Manager
- AWS Lambda
- Amazon EventBridge Scheduler
- AWS CloudTrail / CloudWatch
- OpenVPN
- Inter-Region VPC Peering
- TerraformによるIaCコード化
- Terraform専用IAM Roleによる認証
- Git / GitHub
- Feature Branch
- Pull Request
- Merge

---

## Future Improvements

今後は以下を追加予定です。

- HTTPS / AWS Certificate Manager
- Route 53
- CI/CD
- GitHub Actions
- Terraform StateのRemote Backend化
- IAM権限の最小権限化
- Terraformによる新規AWS環境の0→1構築

---

## Purpose

本プロジェクトは、AWSクラウドインフラの設計・構築・運用・セキュリティ・IaCを実践的に学習することを目的として作成したポートフォリオです。