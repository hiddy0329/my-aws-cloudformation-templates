# my-aws-cloudformation-templates

AWS CloudFormationテンプレート集です。学習用に各種AWSリソースを簡単に構築できるテンプレートを提供しています。

## 📁 テンプレート一覧

### 1. ALB + Auto Scaling による占いAPI構成

**ファイル名:** `templates/alb-asg-template.yaml`

#### 概要
占い結果をランダムで取得できるAPIを実装したEC2サーバーに対し、Application Load Balancer（ALB）とAuto Scaling Group（ASG）を使った**高可用性でスケーラブルな設計**を実現するCloudFormationテンプレートです。

#### 主な特徴
- ✅ **高可用性**: 複数のAvailability Zoneにまたがってインスタンスを配置
- ✅ **自動スケーリング**: CPU使用率70%を基準に自動的にインスタンス数を調整（最小2台、最大4台）
- ✅ **セキュアな設計**: EC2インスタンスはプライベートサブネットに配置し、直接インターネットからアクセス不可
- ✅ **負荷分散**: ALBによる複数インスタンスへのトラフィック分散
- ✅ **簡単デプロイ**: ワンクリックで全インフラを構築

#### 構築されるAWSリソース

| リソース | 個数 | 説明 |
|---------|-----|------|
| VPC | 1 | CIDR: 10.0.0.0/16 |
| Public Subnet | 2 | ALB配置用（各AZ） |
| Private Subnet | 2 | EC2インスタンス配置用（各AZ） |
| Internet Gateway | 1 | インターネット接続用 |
| NAT Gateway | 1 | プライベートサブネットからのアウトバウンド通信用 |
| Application Load Balancer | 1 | HTTP(80)でリクエストを受付 |
| Target Group | 1 | ヘルスチェック付き |
| Auto Scaling Group | 1 | 最小2台、最大4台のインスタンスを管理 |
| Launch Template | 1 | EC2インスタンスの起動設定 |
| Security Groups | 2 | ALB用とEC2用 |
| IAM Role & Instance Profile | 1 | SSM管理用 |

#### アーキテクチャ図（概念図）

```
                    Internet
                       |
                   [ALB] ← HTTP (Port 80)
                  /     \
         [Public Subnet] [Public Subnet]
                  |         |
              [NAT Gateway] |
                  |         |
         [Private Subnet] [Private Subnet]
              [EC2]       [EC2]
              ↓ Auto Scaling Group (2-4台)
```

#### APIエンドポイント

デプロイ後、以下のエンドポイントが利用可能になります：

| エンドポイント | 説明 | レスポンス例 |
|--------------|------|------------|
| `/Uranai` | ランダムな占い結果を取得 | `{"result":"大吉","message":"すべてがうまくいく最高の運勢です！"}` |
| `/hostname` | リクエストを処理したインスタンスID/ホスト名を取得 | `{"hostname":"i-0123456789abcdef"}` |
| `/health` | ヘルスチェック用エンドポイント | `{"status":"healthy"}` |
| `/stress` | CPU負荷テスト（60秒間）を開始 | `{"message":"CPU stress test started for 60 seconds"}` |

#### 占い結果の種類
- **大吉**: すべてがうまくいく最高の運勢です！
- **中吉**: 良いことが起こりそうな予感です。
- **小吉**: 小さな幸せが訪れるでしょう。
- **吉**: 穏やかな一日になりそうです。
- **末吉**: 努力が実を結ぶ兆しがあります。
- **凶**: 慎重に行動することをお勧めします。
- **大凶**: 今日は控えめに過ごしましょう。

#### デプロイ方法

##### AWS CLIを使用する場合

```bash
# スタックの作成
aws cloudformation create-stack \
  --stack-name uranai-api-stack \
  --template-body file://templates/alb-asg-template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# スタックの状態確認
aws cloudformation describe-stacks \
  --stack-name uranai-api-stack \
  --query 'Stacks[0].StackStatus'

# ALBのDNS名を取得
aws cloudformation describe-stacks \
  --stack-name uranai-api-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerURL`].OutputValue' \
  --output text
```

##### AWSマネジメントコンソールを使用する場合

1. CloudFormationコンソールを開く
2. 「スタックの作成」をクリック
3. 「テンプレートファイルのアップロード」を選択し、`alb-asg-template.yaml`をアップロード
4. スタック名を入力（例: `uranai-api-stack`）
5. パラメータはデフォルトのまま（最新のAmazon Linux 2023を使用）
6. IAMリソースの作成を承認
7. 「スタックの作成」をクリック

#### 動作確認

デプロイ完了後（約5-10分）、以下のコマンドで動作確認できます：

```bash
# ALBのURLを環境変数に設定（上記で取得したURL）
export ALB_URL="http://uranai-alb-xxxxxxxxxx.ap-northeast-1.elb.amazonaws.com"

# 占い結果を取得
curl $ALB_URL/Uranai

# どのインスタンスが応答したか確認
curl $ALB_URL/hostname

# ヘルスチェック
curl $ALB_URL/health

# スケーリングテスト（CPU負荷をかけて自動スケールアウトを確認）
curl $ALB_URL/stress
```

#### Auto Scalingの動作確認

```bash
# 複数回ストレステストを実行してCPU使用率を上げる
for i in {1..5}; do
  curl $ALB_URL/stress
  sleep 2
done

# Auto Scaling Groupの状態を確認（数分後にインスタンス数が増加）
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names UranaiASG \
  --query 'AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize]'
```

#### 料金について

このテンプレートで作成されるリソースには以下の料金が発生します：
- **EC2インスタンス**: t3.micro × 2-4台（オンデマンド料金）
- **ALB**: 時間料金 + LCU料金
- **NAT Gateway**: 時間料金 + データ転送料金
- **Elastic IP**: 1個（NAT Gateway使用中は無料）

> **注意**: 使用しない場合は必ずスタックを削除してください。

#### スタックの削除

```bash
aws cloudformation delete-stack --stack-name uranai-api-stack
```

または、AWSマネジメントコンソールから該当スタックを選択して削除してください。

#### 技術仕様

- **OS**: Amazon Linux 2023
- **インスタンスタイプ**: t3.micro
- **アプリケーション**: Go言語で実装
- **ポート**: HTTP 80
- **管理方法**: AWS Systems Manager (Session Manager)

---

## 📝 使い方

1. 使用したいテンプレートを選択
2. 上記のデプロイ方法に従ってCloudFormationスタックを作成
3. Outputsセクションからエンドポイント等の情報を取得
4. 動作確認を実施

## 🔧 前提条件

- AWSアカウント
- AWS CLIがインストール・設定済み（CLI使用の場合）
- 適切なIAM権限（CloudFormation、EC2、VPC、IAM等の作成権限）

## 📚 参考情報

- [AWS CloudFormation ドキュメント](https://docs.aws.amazon.com/cloudformation/)
- [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/)

## 📄 ライセンス

このリポジトリのテンプレートは自由に使用できます。

---

**最終更新**: 2026/04/02
