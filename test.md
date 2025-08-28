# 検証用IAMユーザー設定手順

## 検証目的
3つの機能を検証するため、以下の3つのテストユーザーを作成し、それぞれ異なる状態に設定する：

1. **除外機能の検証**
2. **アクセスキー使用状況の検証**  
3. **パスワード使用状況の検証**

## テストユーザーの設定

### 1. test-excluded-user（除外機能検証用）
**目的**: 除外リストに登録されたユーザーがCOMPLIANTと判定されることを確認

**設定手順**:
```bash
# ユーザー作成
aws iam create-user --user-name test-excluded-user

# アクセスキー作成（45日以上前に作成されたことをシミュレート）
aws iam create-access-key --user-name test-excluded-user

# パスワード設定（45日以上前に設定されたことをシミュレート）
aws iam create-login-profile --user-name test-excluded-user --password TempPassword123!

# 除外リストに追加
aws ssm put-parameter \
  --name "/iam-unused-user-cleanup/excluded-users" \
  --value "test-excluded-user" \
  --type StringList
```

**期待結果**: 
- 未使用期間に関係なくCOMPLIANT判定
- 自動修復の対象外

### 2. test-accesskey-user（アクセスキー使用状況検証用）
**目的**: アクセスキーのみが期間内に使用されているユーザーの動作確認

**設定手順**:
```bash
# ユーザー作成
aws iam create-user --user-name test-accesskey-user

# アクセスキー作成
aws iam create-access-key --user-name test-accesskey-user

# パスワードは設定しない（ログインプロファイルを作成しない）
```

**検証パターン**:
- **パターンA（使用中）**: アクセスキーを実際に使用してAWS APIを呼び出す
- **パターンB（未使用）**: アクセスキーを作成後45日間使用しない

**期待結果**:
- パターンA: COMPLIANT（アクセスキー使用でOR条件を満たす）
- パターンB: NON_COMPLIANT（両方とも未使用でOR条件を満たさない）

### 3. test-password-user（パスワード使用状況検証用）
**目的**: パスワードのみが期間内に使用されているユーザーの動作確認

**設定手順**:
```bash
# ユーザー作成
aws iam create-user --user-name test-password-user

# パスワード設定
aws iam create-login-profile --user-name test-password-user --password TempPassword123!

# アクセスキーは作成しない
```

**検証パターン**:
- **パターンA（使用中）**: AWSマネジメントコンソールにログインする
- **パターンB（未使用）**: パスワード設定後45日間ログインしない

**期待結果**:
- パターンA: COMPLIANT（パスワード使用でOR条件を満たす）
- パターンB: NON_COMPLIANT（両方とも未使用でOR条件を満たさない）

## 現在のユーザー状態（2025-08-27時点）

| ユーザー名 | パスワード状況 | アクセスキー状況 | 30日設定での予想結果 | 90日設定での予想結果 |
|-----------|---------------|-----------------|-------------------|-------------------|
| **test-active-user** | 作成49日前<br/>最終使用48日前 | なし | **NON_COMPLIANT**<br/>（使用48日前 > 30日）| **COMPLIANT**<br/>（使用48日前 < 90日）|
| **test-active-user2** | 作成48日前<br/>未使用 | 作成57分前<br/>未使用 | **NON_COMPLIANT**<br/>（パス作成48日前 > 30日）| **COMPLIANT**<br/>（パス作成48日前 < 90日）|
| **test-active-user3** | 作成47分前<br/>未使用 | なし | **COMPLIANT**<br/>（パス作成47分前 < 30日）| **COMPLIANT**<br/>（パス作成47分前 < 90日）|

## 判定ロジック（AND条件）
パスワードとアクセスキーの両方が有効である場合のみCOMPLIANT：

### パスワード有効性
- **使用履歴がある場合**: 最終使用日が設定期間内で有効、期間超過で無効
- **未使用の場合**: 作成日が設定期間内で有効、期間超過で無効
- **未設定の場合**: 有効とみなす（検知対象外）

### アクセスキー有効性  
- **未設定・全Inactive**: 有効とみなす（検知対象外）
- **アクティブキー存在**: 使用履歴または作成日が設定期間内で有効

## 検証シナリオ

### Phase 1: 現在の状態での確認（30日設定）
```bash
# Config Rule実行
aws configservice start-config-rules-evaluation \
  --config-rule-names custom-unused-iam-user-credentials-check

# 結果確認
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name custom-unused-iam-user-credentials-check
```

**期待結果（30日設定）**:
- `test-active-user`: **NON_COMPLIANT**（パスワード使用48日前 > 30日）
- `test-active-user2`: **NON_COMPLIANT**（パスワード作成48日前 > 30日）
- `test-active-user3`: **COMPLIANT**（パスワード作成47分前 < 30日）

### Phase 2: 除外リスト機能確認
```bash
# 除外リスト設定
aws ssm put-parameter \
  --name "/iam-unused-user-cleanup/excluded-users" \
  --value "test-active-user" \
  --type StringList

# Config Rule実行後の期待結果
```
- `test-active-user`: **COMPLIANT**（除外リスト効果）

### Phase 3: 90日設定での確認
```bash
# 90日設定でスタック更新
aws cloudformation deploy \
  --stack-name IamUnusedUserCleanup \
  --template-file templates/template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides MaxCredentialUsageAge=90
```

**期待結果（90日設定）**:
- 全ユーザー: **COMPLIANT**（全て90日以内）

### Phase 4: 自動修復確認
NON_COMPLIANTユーザーが発生した場合の修復確認：
```bash
# 修復実行確認
aws configservice describe-remediation-execution-status \
  --config-rule-name custom-unused-iam-user-credentials-check
```

## 検証コマンド

### Config Rule手動実行
```bash
aws configservice start-config-rules-evaluation \
  --config-rule-names custom-unused-iam-user-credentials-check
```

### 結果確認
```bash
# 全結果確認
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name custom-unused-iam-user-credentials-check

# NON_COMPLIANTのみ確認
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name custom-unused-iam-user-credentials-check \
  --compliance-types NON_COMPLIANT
```

### ログ確認
```bash
aws logs filter-log-events \
  --log-group-name '/aws/lambda/custom-unused-iam-user-credentials-check-function' \
  --start-time $(date -d '5 minutes ago' +%s)000
```

### 修復実行確認
```bash
aws ssm describe-automation-executions \
  --filters "Key=DocumentNamePrefix,Values=AWSConfigRemediation-RevokeUnusedIAMUserCredentials"
```

## クリーンアップ
```bash
# アクセスキー削除
for user in test-excluded-user test-accesskey-user test-password-user; do
  aws iam list-access-keys --user-name $user --query 'AccessKeyMetadata[].AccessKeyId' --output text | \
  xargs -I {} aws iam delete-access-key --user-name $user --access-key-id {} 2>/dev/null || true
done

# ログインプロファイル削除
for user in test-excluded-user test-accesskey-user test-password-user; do
  aws iam delete-login-profile --user-name $user 2>/dev/null || true
done

# ユーザー削除
for user in test-excluded-user test-accesskey-user test-password-user; do
  aws iam delete-user --user-name $user
done

# SSMパラメータ削除
aws ssm delete-parameter --name "/iam-unused-user-cleanup/excluded-users"
```

## 重要なポイント

1. **OR条件の理解**: パスワードまたはアクセスキーのいずれかが期間内に使用されていればCOMPLIANT
2. **除外リストの優先度**: 除外リストに含まれるユーザーは使用状況に関係なくCOMPLIANT
3. **JST基準**: すべての日時計算は日本標準時（JST）基準
4. **自動修復**: NON_COMPLIANTユーザーは自動的にクレデンシャルが無効化される