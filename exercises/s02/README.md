# S02 実践：注文受付アプリの応答・ログ・IAM権限を調べる

## この演習でできること

固定版の注文受付アプリをAWSへ用意し、5種類のリクエストと応答、CloudWatch Logsの記録をrequest IDで結び付けます。その後、新しく作る演習専用caller roleで同じ正常リクエストを許可・拒否・再許可し、アプリへ届く前のIAM認可とアプリ内の400/500を区別します。最後に、caller roleとアプリstackを削除して残存を確認します。

AWSを使わずにCourseを視聴できます。自分で実践する場合は、組織から利用を認められた専用または隔離済みの非本番AWSアカウントと、受講者自身が取得した短期sessionを使ってください。本番環境や顧客データを使わず、長期credentialやsecretを教材・記録に含めたり公開したりしません。資源とログには料金が発生します。無料利用枠や厳密な料金上限は保証されません。

## 固定版と前提

- 共有アプリ: [aws-observability-sample-app v1.0.3](https://github.com/toma1110/aws-observability-sample-app/tree/v1.0.3)、commit `4c47b860e4c373629f93bed87eb1effa481565fe`
- Region: `ap-northeast-1`（東京）
- 実行環境: AWS CloudShell（AWS CLI v2、Python 3、boto3/botocore、git）
- 作成するもの: CloudFormation stack、Python 3.12 / 128 MB / 8秒のLambda、AWS_IAM認証Function URL、Lambda実行role、3日保持のCloudWatch Logs log group、演習専用caller roleとinline invoke policy

アプリはデータベースへ注文を保存しません。固定runnerはsuccess、validation、意図的な例外、上限付きdelay、固定の無害なuntrusted inputを各1回送ります。意図的なHTTP 500はアプリ内の例外で、Lambdaサービス障害ではありません。untrusted inputは固定例文を記録するだけで、攻撃検知やAI防御の実証ではありません。

## 1. 対象と料金を確認する

1. AWSコンソールで対象アカウントと東京Regionを確認します。組織管理者から利用を認められていないアカウントでは実行しません。
2. アプリを新規取得して使います。以前の演習stackを更新・再利用しません。
3. 見積もりにはLambda呼び出し回数と実行時間、CloudWatch Logs取り込み・保存、その他の転送を含めます。[Lambda料金](https://aws.amazon.com/lambda/pricing/)、[CloudWatch料金](https://aws.amazon.com/cloudwatch/pricing/)、[共有アプリの料金説明](https://github.com/toma1110/aws-observability-sample-app/blob/4c47b860e4c373629f93bed87eb1effa481565fe/README.md#1-%E6%A7%8B%E6%88%90%E3%81%A8%E6%96%99%E9%87%91%E3%82%92%E7%A2%BA%E8%AA%8D%E3%81%99%E3%82%8B)を確認し、必要ならAWS Pricing Calculatorで利用量を仮定します。通常8 request（固定5ケースとIAM比較3状態）、原因確認後に許可された再試行を1回加えた最大9 requestを見積もりの前提にできます。東京Regionのx86 Lambdaを128 MB、各requestが上限の8秒動くと仮定すると、Lambda計算量は最大9 GB秒です。2026-11-11時点の公開単価（GB秒あたり$0.0000166667、request 100万回あたり$0.20）ではLambda分は約$0.00016です。CloudWatch Logsの取り込みが1 MiB以下（$0.76/GB）なら約$0.0008、保存が1か月1 MiB以下（$0.033/GB-month）なら約$0.00004です。転送などを含まないこの仮定では合計$0.01未満ですが、上限や請求額の保証ではありません。実際の単価・利用量は[Lambda料金](https://aws.amazon.com/lambda/pricing/)と[CloudWatch料金](https://aws.amazon.com/cloudwatch/pricing/)で東京を選び、実行前に確認してください。無料とは見なさず、連続実行しません。
4. 必要な権限を事前に確認します。固定アプリのCloudFormation stack作成・参照・削除（CloudFormation deployが使うchange setを含む）、Lambdaの作成・参照・削除、CloudWatch Logs log groupの作成・参照・削除とログ検索が必要です。CloudFormationがLambda実行roleを作成して関数へ渡すためのIAM role作成と`iam:PassRole`も必要です。IAM比較では`sts:GetCallerIdentity`、`iam:GetRole`（必要に応じて`iam:GetUser`）、専用roleとinline policyの作成・参照・削除、`sts:AssumeRole`を使います。組織の権限管理者へ必要範囲を示して確認してください。`AccessDenied`が出たらその操作を止め、対象操作・resource・エラーを管理者へ伝えます。自分でlogin role、permission set、SSO sessionや既存policyを広げたり、回避して続けたりしません。

## 2. 固定アプリを取得する

新しいCloudShell terminalを開き、次を入力します。tag名だけではなく、取得後のcommitも照合してください。

```bash
export AWS_REGION=ap-northeast-1
git clone --branch v1.0.3 --depth 1 https://github.com/toma1110/aws-observability-sample-app.git aws-observability-sample-app-v1.0.3
cd aws-observability-sample-app-v1.0.3
git rev-parse HEAD
aws sts get-caller-identity
printf '%s\n' "$AWS_REGION"
```

`git rev-parse HEAD`が`4c47b860e4c373629f93bed87eb1effa481565fe`と一致すること、`get-caller-identity`のaccountとArnが利用対象であること、Regionが`ap-northeast-1`であることを確認します。不一致なら止めてください。

## 3. 専用stackを作成する

以下はstack名の例です。同じアカウントで他の演習と重ならない名前へ置き換えてください。名前は英字で始まる3〜31文字で、英数字とハイフンだけを使います。使い回し・既存stackの更新を避け、作成前に同名stackがないことを確認します。

```bash
export STACK_NAME=obs-s02-learner1
STACK_CHECK="$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" --region "$AWS_REGION" 2>&1)" && { echo "同名stackがあります。別名に変更してください。"; exit 1; }
[[ "$STACK_CHECK" == *"does not exist"* ]] || { printf '%s\n' "$STACK_CHECK"; exit 1; }
./deploy.sh
export FUNCTION_URL="$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" --region "$AWS_REGION" --query "Stacks[0].Outputs[?OutputKey=='FunctionUrl'].OutputValue" --output text)"
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export FUNCTION_ARN="arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:${STACK_NAME}"
aws cloudformation describe-stacks --stack-name "$STACK_NAME" --region "$AWS_REGION" --query 'Stacks[0].[StackStatus,Tags,Outputs]' --output json
printf 'Function ARN: %s\n' "$FUNCTION_ARN"
```

出力でstack名、`SampleOwner=aws-observability-sample-app`、Function URLを確認します。Function URLの認証方式は`AWS_IAM`のままです。URLは秘密ではありませんが、公開記録へ貼らないでください。対象が別のstackやownerなら続けません。

## 4. 固定5ケースを各1回実行する

同じterminalで実行します。runnerは各caseを1回だけ送信し、responseのrequestIdと同じアプリJSON logを探します。

```bash
python3 run_scenarios.py "$FUNCTION_URL" | tee run-evidence-v1.0.3-all.txt
```

| ケース | 期待status | 読むポイント |
| --- | ---: | --- |
| `success` | 200 | 応答と同じrequestIdの`request_complete` log |
| `validation` | 400 | 入力検証エラーでありIAM拒否ではない |
| `injected_failure` | 500 | 意図的なアプリ例外でありLambdaサービス障害ではない |
| `bounded_delay` | 200 | 1.5秒待機と、アプリ内`durationMs`と課金時間の違い |
| `untrusted_input` | 200 | 固定fixtureの文字列がlogへ記録されたこと。攻撃検知ではない |

runnerの各HTTP試行は15秒でtimeoutし、log照合は40秒で終了します。logが見つからない場合は同じrequestを再送せず、`aws logs filter-log-events`などで同じrequestIdを最大10分までread-onlyで照合します。それでも見つからなければ未確認として記録し、追加送信を止めます。原因を特定してから許される再試行は、5ケースとIAM比較を合わせた演習全体で最大1回です。遅延したlogだけを理由に再送しません。想定外の繰返しや費用増加を認めたら追加実行を止めます。

## 5. 専用callerで許可・拒否・復元を比較する

[IAM比較の実行手順](iam/README.md)に従い、新規caller roleを作成します。演習用roleだけに、対象Functionへの`lambda:InvokeFunctionUrl`と、Function URL経由に限定した`lambda:InvokeFunction`を設定します。既存ログインrole、permission set、session、resource policyは変更しません。

固定入力`{"sku":"SKU-RED-01","quantity":3}`を使い、同じFunction URLへ次の順で各1回送信します。

1. allow: 200、responseのrequestId、同一IDのアプリlogを記録。
2. 専用roleからinvoke inline policyだけを削除して最大60秒待ち、deny: 実際のHTTP status/body/AWS request identifier（返る場合）を記録。原因を特定してからの再試行は、5ケースとIAM比較を合わせ演習全体で1回を超えません。
3. 同じinline policyを戻して最大60秒待ち、re-allow: 同じ入力が成功し、requestIdと対応logが戻るか記録。

Function URLの認可で拒否されたrequestはLambda handlerへ届かず、app requestIdや`request_complete` logがない場合があります。返らなかったIDやlogを補わないでください。アプリJSON log、Lambda START/END/REPORT行、AWSの拒否応答は別の記録です。log配信が遅れているときは同じ時間帯を最大10分まで再確認できますが、requestは再送しません。

## 6. 結果を読み、別の演習と区別する

- `200` / `400` / `500`はアプリが返したHTTP結果です。400は入力検証、意図的な500はサンプル内の例外です。
- IAM拒否はアプリ処理前の認可結果です。app logがないときはそのまま記録します。
- `durationMs`はアプリ内計測値で、Lambdaの課金時間やREPORT行の時間ではありません。
- Northstar Retailは架空のtabletopです。実AWS実測をscenarioへ持ち込んだり、実測結果を架空の事件の証拠にしたりしません。

記録には[worksheet](worksheet.md)、架空状況には[scenario](scenario.md)、判断例には[answer guide](answer-guide.md)を使います。実測値は実行時刻・環境・固定app commitを併記し、account番号、credential、完全なFunction URLは公開しません。

## 7. 後片付けと残存確認

先に必要な実行根拠を許可された安全な場所へ保存します。公開repositoryへcredential、account固有のURL、内部logを保存しません。次に専用caller roleのinline policyとroleを削除し、同名roleが`NoSuchEntity`になることを確認します。IAM READMEの比較手順から戻って、ここにあるcleanupコマンドは一度だけ実行します。

```bash
printf 'Region=%s Stack=%s CallerRole=%s\n' "$AWS_REGION" "$STACK_NAME" "$LAB_ROLE_NAME"
ROLE_POLICY_CHECK="$(aws iam get-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab 2>&1)"
if [[ "$ROLE_POLICY_CHECK" == *"NoSuchEntity"* ]]; then
  echo "caller inline policy is already absent"
else
  [[ "$ROLE_POLICY_CHECK" == *"PolicyDocument"* ]] || { printf '%s\n' "$ROLE_POLICY_CHECK"; exit 1; }
  aws iam delete-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab
fi
aws iam delete-role --role-name "$LAB_ROLE_NAME"
aws iam get-role --role-name "$LAB_ROLE_NAME"
./cleanup.sh
```

`get-role`が`NoSuchEntity`で失敗し、`cleanup.sh`が固定の削除完了メッセージを表示することを確認します。cleanupはstack、Lambda、Function URL、実行role、log groupの完全一致する対象の不在を確認します。削除エラーや確認失敗を完了と扱わず、別stackやresourceへ削除を広げません。

実行根拠を許可された安全な場所へ必要最小限だけ保存したら、専用IAM role/policy、stackなどAWS資源を上記の順で削除し、残存がないことを確認します。アプリsource cloneは再現用に保持できます。検証専用の`lab-trust.json`、`invoke-policy.json`、`run-evidence-v1.0.3-all.txt`と、IAM手順で作る一時JSONは不要になった時点で削除し、`test -e`で不在を確認します。使ったshellで`unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_SECURITY_TOKEN`を実行し、shell historyに秘密値を含む入力がないことも確認してください。ファイル整理とCloudShell terminalの終了は別作業です。処理・保存・一時物整理が済んだ後、自分で開始したterminalの終了を確認し、その後自分で開いたtabを閉じます。STS sessionは個別に失効できるとは扱わず、残存する有効期限を記録します。

### 関連資料

- [IAM比較の詳細なコマンド](iam/README.md)
- [実行シナリオ](scenario.md)
- [記録ワークシート](worksheet.md)
- [判断例](answer-guide.md)
