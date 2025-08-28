# AWS IAM未使用ユーザー自動クリーンアップ（除外リスト対応）

未使用のIAMユーザーを自動検出・無効化し、重要なユーザーを除外リストで保護するAWS Config + Lambda + SSM Automationベースのソリューションです。

単純な未使用ユーザー検出であればAWS Configマネージドルール（`iam-user-unused-credentials-check`）で実現可能ですが、本ソリューションは特定ユーザーの除外機能を提供するためにカスタムLambda関数を使用しています。

## 概要

本ソリューションは、未使用のIAMユーザーのクレデンシャルを自動的に無効化します。重要な管理者アカウントやサービスアカウントは除外リストで保護できるため、安全に運用できます。

### 基本機能

- 検出: カスタムLambda関数で未使用のIAMユーザーを検出
- 除外: 特定ユーザーを除外リストで管理
- 修復: AWS Config Remediationで自動的にクレデンシャルを無効化
- カスタマイズ可能:
  - 無効化対象期間: デフォルト90日（パラメータで変更可能）
  - 定期チェック頻度: デフォルト24時間毎（変更可能）

### 除外ユーザー機能

重要な管理者アカウントやサービスアカウントなど、自動無効化を避けたいIAMユーザーは除外リストで管理できます。除外リストはSSM Parameter StoreのStringListタイプで管理され、デプロイ後にいつでも変更可能です。

- パラメータ名: `/iam-unused-user-cleanup/excluded-users`
- パラメータタイプ: StringList
- 値の形式: カンマ区切りユーザー名リスト（例: admin-user,service-account）
- 除外されたユーザーは未使用期間に関係なくCOMPLIANTと判定されます
- **パラメータが存在しない場合**: 除外リストなしで通常の未使用検知処理を実行します（エラーになりません）

## 管理体系

### CloudFormationテンプレートで管理される項目

`templates/template.yaml`でデプロイ時に設定・変更する項目：

#### 定期チェック頻度の設定

管理場所: `MaximumExecutionFrequency`
```yaml
SourceDetails:
  - MaximumExecutionFrequency: TwentyFour_Hours  # ← ここで変更
```

#### 無効化対象期間の設定

管理場所: CloudFormationパラメータ（共有設定）

```yaml
Parameters:
  MaxCredentialUsageAge:
    Type: String
    Default: "90"  # ← デフォルト値（デプロイ時に変更可能）
```

このパラメータは以下の箇所で自動的に共有されます：
- Lambda検出用の環境変数
- SSM修復用のパラメータ

**注意**: 両方の処理で同じ判定基準を使用するため、パラメータ化により設定の一貫性が保たれます。

### CloudFormation外で管理される項目

デプロイ後に動的に変更可能な項目：

#### 除外ユーザーの設定

管理場所: AWS SSM Parameter Store

パラメータ名: `/iam-unused-user-cleanup/excluded-users`

設定方法:
```bash
aws ssm put-parameter \
  --name "/iam-unused-user-cleanup/excluded-users" \
  --value "admin-user,service-account" \
  --type StringList
```

## Lambda関数の処理ロジック

### 実行環境
- ランタイム: Python 3.13
- 実行タイムアウト: 300秒（5分）
- メモリサイズ: 256MB
- 日時基準: UTC（協定世界時）

### 処理フロー
1. **除外ユーザーリスト取得**: SSM Parameter Store（`/iam-unused-user-cleanup/excluded-users`）から除外対象ユーザーのリストを取得
2. **全IAMユーザー取得**: ページネーション対応でアカウント内の全IAMユーザーを取得
3. **ユーザー評価**:
   - 除外リストに含まれるユーザー → COMPLIANT（処理対象外）
   - その他のユーザー → クレデンシャル使用状況をチェック
4. **クレデンシャルチェック**（UTC基準）:
   - パスワードとアクセスキーの両方を個別に評価
   - **判定ロジック**: パスワードとアクセスキーの両方が有効である場合のみCOMPLIANT（AND条件）
   - どちらか一方でも無効（設定期間を超過）の場合はNON_COMPLIANT
5. **評価結果送信**: AWS Configに各ユーザーの評価結果を送信

### 判定ロジック詳細

#### 除外リスト確認
ユーザー名が除外リストに含まれている場合、使用状況に関係なくCOMPLIANTと判定

#### パスワード有効性判定
- **使用履歴がある場合**: 最終使用日が設定期間内であれば有効、期間超過で無効
- **未使用の場合**: 作成日が設定期間内であれば有効、期間超過で無効
- **パスワード未設定の場合**: 有効とみなす（検知対象外）

#### アクセスキー有効性判定
- **アクセスキーが存在しない場合**: 有効とみなす（検知対象外）
- **全キーがInactiveの場合**: 有効とみなす（検知対象外）
- **アクティブキーが存在する場合**: 一つでも以下の条件を満たせば有効
  - 使用履歴があり最終使用日が設定期間内
  - 未使用で作成日が設定期間内

#### 最終判定（AND条件）
パスワードとアクセスキーの両方が有効である場合のみCOMPLIANT。どちらか一方でも無効の場合はNON_COMPLIANT

**例**: パスワードが有効でもアクセスキーが無効ならNON_COMPLIANT、その逆も同様

### 運用上の注意
- Python 3.13ランタイムの定期的な更新が必要な場合があります
- Lambda関数のタイムアウトやメモリサイズは必要に応じて調整可能です
- 日時計算はすべてUTC（協定世界時）基準で行われ、修復アクションと統一されています
- 修復アクションはSSM Document `AWSConfigRemediation-RevokeUnusedIAMUserCredentials` バージョン5で固定されており、検知ロジックがこのバージョンに完全に一致するように組まれています。バージョンが変わると修復ドキュメントのスクリプトが破壊的変更になる可能性があり検知ロジックとズレる可能性があるのでバージョンを上げる際は注意してください。

## 動作フロー

1. AWS Configが定期的（デフォルト24時間毎）にカスタムLambda関数を実行
2. Lambda関数が上記処理ロジックに従って全IAMユーザーを評価
3. NON_COMPLIANTと判定されたユーザーに対してConfig Remediationが自動実行
4. SSM Documentが実行され、対象ユーザーのクレデンシャルを無効化

## 設定値の対応表

| 設定項目 | デフォルト値 | 設定箇所 | 変更可能な値 |
|---------|------------|----------|-------------|
| **定期チェック頻度** | 24時間毎 | `template.yaml`<br/>`MaximumExecutionFrequency` | `Three_Hours`, `Six_Hours`,<br/>`Twelve_Hours`, `TwentyFour_Hours`,<br/>`Seven_Days` |
| **無効化対象期間** | 90日 | CloudFormationパラメータ<br/>`MaxCredentialUsageAge` | 任意の数値（日数）<br/>デプロイ時パラメータで変更 |
| **除外ユーザー** | なし | SSM Parameter Store<br/>`/iam-unused-user-cleanup/excluded-users` | カンマ区切りユーザー名 |

## デプロイ手順

### 1. 前提条件

- AWS Config が有効化されていること
- 適切なIAM権限があること
- AWS CLI が設定済みであること

### 2. CloudFormationスタックのデプロイ

```bash
aws cloudformation deploy \
  --stack-name IamUnusedUserCleanup \
  --template-file templates/template.yaml \
  --capabilities CAPABILITY_NAMED_IAM 
```

### 3. 除外ユーザーリストの設定

```bash
# 除外したいユーザーをカンマ区切りで指定（StringListタイプで作成）
aws ssm put-parameter \
  --name "/iam-unused-user-cleanup/excluded-users" \
  --value "admin-user,service-account,emergency-access" \
  --type StringList \
```


## トラブルシューティング

### Lambda関数が実行されない場合

1. Configの有効化状態を確認
   - Configuration RecorderとDelivery Channelの設定状態を確認

2. Lambdaの実行権限を確認
   - ConfigRuleLambdaRoleのポリシーを確認

### 除外ユーザーが正しく動作しない場合

1. SSMパラメータの内容を確認
   - パラメータストアの除外ユーザーリストを確認

2. Lambdaの実行ログを確認
   - CloudWatch Logsで除外処理の動作を確認

### 修復が実行されない場合

1. Remediation設定を確認
   - Config RuleとSSM Documentの関連付けを確認

2. 修復ロールの権限を確認
   - ConfigRemediationServiceRoleのポリシーを確認

3. SSM Automationの実行履歴を確認
   - 修復処理の実行状態とエラーを確認


## 設定方法

### 1. 未使用期間の変更

#### 方法1: デフォルト値変更（テンプレート修正）
```yaml
# template.yaml の Parameters セクション
Parameters:
  MaxCredentialUsageAge:
    Type: String
    Default: "90"  # デフォルト値
```

#### 方法2: デプロイ時パラメータ指定（推奨）
```bash
# 90日設定でデプロイ
aws cloudformation deploy \
  --stack-name IamUnusedUserCleanup \
  --template-file templates/template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides MaxCredentialUsageAge=90
```

### 2. 定期チェック頻度の変更（例：24時間毎 → 週1回）

設定箇所: `template.yaml` の `MaximumExecutionFrequency`
```yaml
SourceDetails:
  - MaximumExecutionFrequency: Seven_Days  # TwentyFour_Hours → Seven_Days
```

利用可能な頻度オプション
| 設定値 | 実行頻度 | 用途 |
|--------|---------|------|
| Three_Hours | 3時間毎 | 厳格な管理 |
| Six_Hours | 6時間毎 | 高頻度監視 |
| Twelve_Hours | 12時間毎 | 日2回チェック |
| TwentyFour_Hours | 24時間毎 | 推奨設定 |
| Seven_Days | 週1回 | 軽量運用 |

### 3. 除外ユーザーの変更

設定場所: SSM Parameter Store（デプロイ後にいつでも変更可能）

- パラメータ名: /iam-unused-user-cleanup/excluded-users
- パラメータタイプ: StringList
- 値の形式: カンマ区切りユーザー名リスト

設定例:
```bash
# 新規作成
aws ssm put-parameter \
  --name "/iam-unused-user-cleanup/excluded-users" \
  --value "admin-user,service-account" \
  --type StringList

# 既存パラメータの更新
aws ssm put-parameter \
  --name "/iam-unused-user-cleanup/excluded-users" \
  --value "admin-user,service-account,new-admin" \
  --type StringList \
  --overwrite
```

## 注意事項

- 本ソリューションは未使用ユーザーのクレデンシャルを無効化します
- 実行前に除外ユーザーリストを正確に設定してください
- 本番環境での実行前に十分なテストを行ってください
- AWS Config、Lambda、SSM Automationの料金が発生します

## ライセンス

MIT License