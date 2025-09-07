# AWS監視設計

## 1. 目的と適用範囲

- **目的:** 障害の早期検知と影響最小化、SLO達成、運用コスト最適化を同時に成立させる。
- **対象環境:** dev / stg / prod（環境別にしきい値・通知経路・可用性要件を差別化）
- **対象領域:** 可用性・性能・エラー・容量・セキュリティ・コストの6領域
- **非対象:** バッチ結果の業務検証（別途ジョブ監視設計で管理）

> 定義観点: 監視＝メトリクス/ログ/イベント/合成監視の収集→相関→検知→通知→初動→エスカレーション→事後分析→改善の一連フロー。

---

## 2. 監視原則とSLO/SLI

- **原則1（顧客影響優先）:** ユーザー体験直結のSLI（可用性・レイテンシ・エラー率）を最優先に設計。
- **原則2（アラーム健全性）:** ノイズゼロ志向。アクション不可の通知は排除、しきい値は実測分布＋誤検知率で調整。
- **原則3（自動化）:** 収集・相関・通知・初動（ローテーション/スケール/再実行）は自動実行を前提。
- **原則4（可観測性）:** メトリクス・ログ・トレースをタグで一意結合し、原因特定の平均時間を短縮。

### 環境別SLOと通知レベル

| 環境 | 可用性SLO | p95レイテンシ | エラー率 | 通知レベル | エスカレーション |
|------|------------|----------------|-----------|--------------|------------------|
| prod | 99.9%      | 300 ms         | 1%未満     | Sev1–3       | 24/365オンコール |
| stg  | 99.0%      | 500 ms         | 2%未満     | Sev2–4       | 営業時間         |
| dev  | 任意        | 任意            | 任意        | Sev4のみ      | なし              |

> Sev定義例: Sev1=顧客影響大/即時、Sev2=限定的影響/1h、Sev3=軽微/当日、Sev4=情報/定期確認。

---

## 3. SLOとSevレベルのマトリクス図

| 項目       | SLO目標           | Sev1                             | Sev2                        | Sev3                        | Sev4           |
|------------|-------------------|----------------------------------|-----------------------------|-----------------------------|----------------|
| 可用性     | 99.9%（prod）     | ステータスチェック失敗、Failover発生 | 一部AZ障害、ASG不足           | 一時的な接続失敗             | 情報通知のみ     |
| レイテンシ | p95 < 300ms（prod） | API Gateway遅延、ALB遅延         | Lambda Duration上昇         | RDS Read/WriteLatency上昇   | 定期確認         |
| エラー率   | < 1%（prod）      | Lambda Errors率 > 1%、ALB 5xx急増 | API 4xx/5xx増加             | GuardDuty Medium            | CloudTrail異常   |
| 容量       | FreeStorage > 15% | RDS逼迫、S3急増                  | EC2 Disk使用率上昇           | S3オブジェクト数増加         | 情報通知         |
| セキュリティ | 重大変更なし      | IAM Root使用、Public ACL変更     | Security Hub High           | GuardDuty Low               | 定期確認         |
| コスト     | 予算内            | Budgets逸脱 > 120%              | NAT GW急増                  | Synthetics過剰              | 情報通知         |

---

## 4. アーキテクチャとデータフロー

![監視アーキテクチャ図](https://raw.githubusercontent.com/fgzokn/aws-monitoring-assets/main/monitoring-architecture-v2.png)

> データ保管: メトリクス 高解像度短期(1–14日)→低解像度長期(15カ月)、ログはS3へExport/ライフサイクル管理。

---

## 5. 監視項目としきい値

（※この章はページの内容をそのまま貼り付けてください）

---

## 6. ダッシュボードと報告

![ダッシュボード構成図](https://raw.githubusercontent.com/fgzokn/aws-monitoring-assets/main/dashboard-structure.png)

> prod / stg / dev × 可用性 / レイテンシ / エラー / 容量 / セキュリティ / コスト × EC2 / RDS / Lambda / API Gateway / ALB / S3  
> ↓ 表示項目：現在値 / しきい値 / Sevレベル / 過去推移（7日/30日）  
> ↓ 可視化ツール：CloudWatch Dashboards / ServiceLens / Cost Explorer

---

## 7. 通知・初動・運用統制

![通知フロー図](https://raw.githubusercontent.com/fgzokn/aws-monitoring-assets/main/notification-flow.png)

> Sev1 → Incident Manager（即時）  
> Sev2 → ChatOps（1時間以内）  
> Sev3 → Email（当日）  
> Sev4 → Email（定期）  
> ↓ 初動対応：Lambda / SSM Automation / OpsCenter  
> ↓ 運用統制：OpsCenter記録 / エスカレーション / 月次レビュー

---

## 8. IaC・命名・コスト最適化

### IaC化
- ツール：Terraform / CloudFormation
- 対象：CloudWatch Alarms / Dashboards / SNS / EventBridge / SSM Automation
- 管理：GitHub + CI/CD（CodePipeline）

### 命名ルール
- 形式：

\[環境\]

-

\[サービス\]

-

\[項目\]

-

\[Sev\]

-

\[通知先\]


- 例：prod-ec2-cpu-sev2-chatops

### コスト最適化
- dev環境は主要項目のみ監視
- S3ライフサイクルでログ自動削除
- メトリクスは高解像度14日間 → 低解像度へ移行
