AWS監視設計

1. 目的と適用範囲

目的: 障害の早期検知と影響最小化、SLO達成、運用コスト最適化を同時に成立させる。

対象環境: dev / stg / prod（環境別にしきい値・通知経路・可用性要件を差別化）

対象領域: 可用性・性能・エラー・容量・セキュリティ・コストの6領域

非対象: バッチ結果の業務検証（別途ジョブ監視設計で管理）

定義観点: 監視＝メトリクス/ログ/イベント/合成監視の収集→相関→検知→通知→初動→エスカレーション→事後分析→改善の一連フロー。

2. 監視原則とSLO/SLI

原則1（顧客影響優先）: ユーザー体験直結のSLI（可用性・レイテンシ・エラー率）を最優先に設計。

原則2（アラーム健全性）: ノイズゼロ志向。アクション不可の通知は排除、しきい値は実測分布＋誤検知率で調整。

原則3（自動化）: 収集・相関・通知・初動（ローテーション/スケール/再実行）は自動実行を前提。

原則4（可観測性）: メトリクス・ログ・トレースをタグで一意結合し、原因特定の平均時間を短縮。

環境別SLOと通知レベル

環境

可用性SLO

p95レイテンシ

エラー率

通知レベル

エスカレーション

prod

99.9%

300 ms

1%未満

Sev1–3

24/365オンコール

stg

99.0%

500 ms

2%未満

Sev2–4

営業時間

dev

任意

任意

任意

Sev4のみ

なし

Sev定義例: Sev1=顧客影響大/即時、Sev2=限定的影響/1h、Sev3=軽微/当日、Sev4=情報/定期確認。

→ 詳細なSev判定条件は第3章マトリクス図を参照

3. SLOとSevレベルのマトリクス図

このマトリクスは、上記SLOを基準に各監視項目の重大度（Sev1〜Sev4）を定義したものです。 アラーム発生時の優先度判断と通知経路の選定に直結します。

項目

SLO目標

Sev1

Sev2

Sev3

Sev4

可用性

99.9%（prod）

ステータスチェック失敗、Failover発生

一部AZ障害、ASG不足

一時的な接続失敗

情報通知のみ

レイテンシ

p95 < 300ms（prod）

API Gateway遅延、ALB遅延

Lambda Duration上昇

RDS Read/WriteLatency上昇

定期確認

エラー率

< 1%（prod）

Lambda Errors率 > 1%、ALB 5xx急増

API 4xx/5xx増加

GuardDuty Medium

CloudTrail異常

容量

FreeStorage > 15%

RDS逼迫、S3急増

EC2 Disk使用率上昇

S3オブジェクト数増加

情報通知

セキュリティ

重大変更なし

IAM Root使用、Public ACL変更

Security Hub High

GuardDuty Low

定期確認

コスト

予算内

Budgets逸脱 > 120%

NAT GW急増

Synthetics過剰

情報通知

※stg/dev環境ではしきい値を緩和し、通知レベルを1段階下げる

→ 環境別SLOは第2章を参照

4. アーキテクチャとデータフロー



データ保管: メトリクス 高解像度短期(1–14日)→低解像度長期(15カ月)、ログはS3へExport/ライフサイクル管理。

5. 監視項目としきい値

以下は主要AWSサービスごとの監視項目、しきい値、Sevレベル、通知経路の例です。

EC2

項目

しきい値

Sev

通知

CPU使用率

85%（5分継続）

Sev2

SNS → ChatOps

メモリ使用率

90%

Sev3

SNS → Email

Disk使用率

80%

Sev3

SNS → Email

RDS

項目

しきい値

Sev

通知

FreeStorageSpace

< 15%

Sev1

SNS → Incident Mgmt

ReadLatency

100ms

Sev2

SNS → ChatOps

CPU使用率

80%

Sev3

SNS → Email

S3

項目

しきい値

Sev

通知

バケットサイズ急増

前日比+50%

Sev2

SNS → ChatOps

オブジェクト数増加

前日比+30%

Sev3

SNS → Email

Lambda

項目

しきい値

Sev

通知

Error率

1%

Sev1

SNS → Incident Mgmt

Duration

p95 > 1000ms

Sev2

SNS → ChatOps

API Gateway

項目

しきい値

Sev

通知

5xxエラー率

0.5%

Sev1

SNS → Incident Mgmt

レイテンシ

p95 > 500ms

Sev2

SNS → ChatOps

ALB

項目

しきい値

Sev

通知

Target Response Time

p95 > 400ms

Sev2

SNS → ChatOps

5xxエラー率

1%

Sev1

SNS → Incident Mgmt

6. ダッシュボードと報告



[環境別] prod / stg / dev    × [カテゴリ別] 可用性 / レイテンシ / エラー / 容量 / セキュリティ / コスト    × [サービス別] EC2 / RDS / Lambda / API Gateway / ALB / S3    ↓ [表示項目] 現在値 / しきい値 / Sevレベル / 過去推移（7日/30日）    ↓ [可視化ツール] CloudWatch Dashboards / ServiceLens / Cost Explorer

報告運用

定期報告: 週次（運用チーム）・月次（マネジメント）

報告内容: アラーム件数・Sev分布・対応状況・改善提案

報告形式: PDFレポート + ダッシュボードスクリーンショット

7. 通知・初動・運用統制



[Sev1] SNS → Incident Manager（即時） [Sev2] SNS → ChatOps（1時間以内対応） [Sev3] SNS → Email（当日対応） [Sev4] SNS → Email（定期確認）    ↓ [初動対応] 自動化（Lambda / SSM Automation）または手動（OpsCenter）    ↓ [運用統制] OpsCenter記録 / エスカレーション / 月次レビュー

8. IaC・命名・コスト最適化

以下は監視構成のIaC化、命名ルール、コスト最適化の方針です。

IaC化

ツール: Terraform / CloudFormation

対象: CloudWatch Alarms / Dashboards / SNS / EventBridge / SSM Automation

管理: GitHub + CI/CD（CodePipeline）

命名ルール

形式: [環境]-[サービス]-[項目]-[Sev]-[通知先]

例: prod-ec2-cpu-sev2-chatops

コスト最適化

監視対象の絞り込み: dev環境は主要項目のみ

ログ保管: S3ライフサイクルで自動削除

メトリクス: 高解像度は14日間、以降は低解像度に移行