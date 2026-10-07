# AWS Multi-Region Portfolio Platform

AWS上に構築した、Web/API・コンテナ・プライベートネットワーク・監視・イベント駆動処理・マルチリージョン通信を組み合わせたクラウドインフラ構成です。

## Architecture

```text
Internet
   |
   v
Application Load Balancer
   |
   v
Web EC2 (Private Subnet)
Docker / Node.js
   |
   v
API EC2 (Private Subnet)
Docker / Node.js
   |
   +------------------> Amazon S3
   |
   +---- VPC Peering ----> Tokyo VPC
                           |
                           v
                     Minutes Analytics API