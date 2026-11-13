# S02 IAM比較：専用callerのFunction URL権限

この手順はS02で作成したFunction URLだけを対象にします。先に[演習README](../README.md)の固定版・account・Region・stack owner確認を終えてください。ログイン中のrole、permission set、SSO session、Function URL設定、resource policy、CloudTrail、Organizationsには変更を加えません。

## caller roleを作る

始める前に、実行者にCloudFormationのstack確認、Lambda/CloudWatch Logs参照、IAM role/policyの演習対象範囲の操作、`sts:GetCallerIdentity`と`sts:AssumeRole`が認められていることを確認します。固定アプリのdeploy/cleanupにはCloudFormationが作成する実行roleに対する`iam:PassRole`も必要です。権限不足ならAccessDeniedの操作で止め、対象・resource・エラーを組織のAWS権限管理者へ確認します。login roleやSSO基盤の権限を自分で拡張しません。

CloudShellはbashです。前の演習READMEと同じterminalで作業してください。role名は他の人と重ならない値に変え、3〜64文字のIAM role名にします。

```bash
export LAB_ROLE_NAME=s02-lab-caller-learner1
CALLER_ARN="$(aws sts get-caller-identity --query Arn --output text)"
case "$CALLER_ARN" in
  *:assumed-role/*)
    CURRENT_ROLE_NAME="${CALLER_ARN#*:assumed-role/}"
    CURRENT_ROLE_NAME="${CURRENT_ROLE_NAME%%/*}"
    CALLER_PRINCIPAL_ARN="$(aws iam get-role --role-name "$CURRENT_ROLE_NAME" --query Role.Arn --output text)"
    ;;
  *:user/*)
    CALLER_PRINCIPAL_ARN="$(aws iam get-user --query User.Arn --output text)"
    ;;
  *) echo "rootや判別できないprincipalでは続けず、管理者に確認してください。"; exit 1 ;;
esac
printf '信頼するprincipal: %s\n' "$CALLER_PRINCIPAL_ARN"
```

表示したprincipalが自分の利用中roleまたはuserであることを確認してください。空または不一致なら止めます。roleの同名チェックでは、`NoSuchEntity`以外のエラーも停止として扱います。

```bash
ROLE_CHECK="$(aws iam get-role --role-name "$LAB_ROLE_NAME" 2>&1)" && { echo "同名roleがあります。別名を選んでください。"; exit 1; }
[[ "$ROLE_CHECK" == *"NoSuchEntity"* ]] || { printf '%s\n' "$ROLE_CHECK"; exit 1; }
cat > lab-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "$CALLER_PRINCIPAL_ARN"},
    "Action": "sts:AssumeRole"
  }]
}
EOF
aws iam create-role --role-name "$LAB_ROLE_NAME" --assume-role-policy-document file://lab-trust.json --description "S02 dedicated Function URL invocation exercise"
export LAB_ROLE_ARN="$(aws iam get-role --role-name "$LAB_ROLE_NAME" --query Role.Arn --output text)"
```

この信頼policyは、自分が確認したprincipal一つだけに`sts:AssumeRole`を許可します。`*`や別アカウントを指定しません。

## 対象Functionだけにinvoke policyを設定する

`FUNCTION_ARN`と`FUNCTION_URL`は、演習READMEでstack出力から設定したものを使います。policyは対象Function URLのIAM認証と、Function URL経由の呼び出しだけを許します。

```bash
cat > invoke-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunctionUrl",
      "Resource": "$FUNCTION_ARN",
      "Condition": {"StringEquals": {"lambda:FunctionUrlAuthType": "AWS_IAM"}}
    },
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "$FUNCTION_ARN",
      "Condition": {"Bool": {"lambda:InvokedViaFunctionUrl": "true"}}
    }
  ]
}
EOF
aws iam put-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab --policy-document file://invoke-policy.json
aws iam get-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab --query PolicyDocument --output json
```

出力したpolicyのResourceが`$FUNCTION_ARN`と完全一致し、wildcardがなく、上の2 actionとconditionだけであることを確認します。別Functionや任意のLambda呼び出しを許す変更はしません。

## 一時sessionで同じrequestを3回試す

このroleにはログ参照権限を付けません。固定版の`invoke.py`はFunction URL送信前にCloudFormation `DescribeStacks`を実行するため、専用callerの最小権限ではその事前確認が拒否されます。そこで同じboto3/botocoreのSigV4署名APIを使い、stack owner・status・URLの事前照合とログ参照を元のCloudShell identityで、HTTPS POSTだけを一時caller identityで行います。以下の関数はSTSのsecret key/session tokenを端末やfileへ表示・保存しません。一時credentialは各サブシェルの環境変数にだけ置き、終了時に消します。role sessionの有効期限だけを記録します。

```bash
verify_base_target() {
  local base_arn stack_json owner actual_url status
  base_arn="$(aws sts get-caller-identity --query Arn --output text)" || return 1
  case "$base_arn" in *:assumed-role/"$LAB_ROLE_NAME"/*) echo "base identityではありません。" >&2; return 1;; esac
  stack_json="$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" --region "$AWS_REGION" --output json)" || return 1
  owner="$(jq -r '.Stacks[0].Tags[]? | select(.Key=="SampleOwner") | .Value' <<<"$stack_json")"
  actual_url="$(jq -r '.Stacks[0].Outputs[]? | select(.OutputKey=="FunctionUrl") | .OutputValue' <<<"$stack_json")"
  status="$(jq -r '.Stacks[0].StackStatus' <<<"$stack_json")"
  [[ "$owner" == aws-observability-sample-app && "$actual_url" == "$FUNCTION_URL" && "$status" == CREATE_COMPLETE ]] || { echo "stack owner/status/URLが一致しません。停止してください。" >&2; return 1; }
  printf 'Base identity: %s\nStack %s (%s) owner and Function URL verified.\n' "$base_arn" "$STACK_NAME" "$status"
}

assume_lab_and_invoke() {
  verify_base_target || return 1
  (
    set -euo pipefail
    local assumed
    assumed="$(aws sts assume-role --role-arn "$LAB_ROLE_ARN" --role-session-name s02-lab-call --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken,Expiration]' --output text)"
    read -r AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN SESSION_EXPIRATION <<<"$assumed"
    [[ -n "$AWS_ACCESS_KEY_ID" && -n "$AWS_SECRET_ACCESS_KEY" && -n "$AWS_SESSION_TOKEN" && "$AWS_SESSION_TOKEN" != None ]] || { echo "STS credentialsがありません。送信せず停止します。" >&2; exit 1; }
    export AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
    unset AWS_PROFILE AWS_DEFAULT_PROFILE
    export AWS_REGION AWS_DEFAULT_REGION="$AWS_REGION" AWS_EC2_METADATA_DISABLED=true
    printf 'Temporary caller session expires: %s\n' "$SESSION_EXPIRATION"
    python3 - "$FUNCTION_URL" "$LAB_ROLE_NAME" "$ACCOUNT_ID" "$AWS_REGION" '{"sku":"SKU-RED-01","quantity":3}' <<'PY'
import os, re, sys, urllib.error, urllib.parse, urllib.request
import boto3
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest
url, role_name, account_id, region, payload = sys.argv[1:]
if not all(os.environ.get(k) for k in ("AWS_ACCESS_KEY_ID", "AWS_SECRET_ACCESS_KEY", "AWS_SESSION_TOKEN")):
    raise SystemExit("一時STS credentialsがありません。送信しません")
session = boto3.Session()
identity = session.client("sts", region_name=region).get_caller_identity()
expected = f"arn:aws:sts::{account_id}:assumed-role/{role_name}/s02-lab-call"
if identity.get("Arn") != expected:
    raise SystemExit("想定した専用caller ARNではありません。送信しません")
parsed = urllib.parse.urlsplit(url)
try:
    port = parsed.port
except ValueError:
    raise SystemExit("Function URLのportが不正です。送信しません")
if parsed.scheme != "https" or parsed.username or parsed.password or port or parsed.query or parsed.fragment or parsed.path not in ("", "/") or not re.fullmatch(r"[a-z0-9-]+\.lambda-url\.([a-z0-9-]+)\.on\.aws", parsed.hostname or "") or re.fullmatch(r"[a-z0-9-]+\.lambda-url\.([a-z0-9-]+)\.on\.aws", parsed.hostname or "").group(1) != region:
    raise SystemExit("Function URL/Region形式が一致しません。送信しません")
credentials = session.get_credentials()
if credentials is None:
    raise SystemExit("一時STS credentialsを取得できません。送信しません")
request = AWSRequest(method="POST", url=url, data=payload.encode(), headers={"content-type": "application/json"})
SigV4Auth(credentials.get_frozen_credentials(), "lambda", region).add_auth(request)
req = urllib.request.Request(url, data=payload.encode(), headers=dict(request.headers), method="POST")
class NoRedirect(urllib.request.HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        return None
try:
    with urllib.request.build_opener(NoRedirect()).open(req, timeout=15) as response:
        print(f"HTTP {response.status}")
        print(f"AWS request ID: {response.headers.get('x-amzn-requestid', 'not returned')}")
        print(response.read().decode())
except urllib.error.HTTPError as exc:
    print(f"HTTP {exc.code}")
    print(f"AWS request ID: {exc.headers.get('x-amzn-requestid', 'not returned')}")
    print(exc.read().decode())
PY
  )
}
```

### Allow

```bash
sleep 60
if ! POLICY_CHECK="$(aws iam get-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab --query PolicyDocument --output json 2>&1)"; then
  printf 'policyを読み取れません。送信せず停止してください。\n%s\n' "$POLICY_CHECK"
  exit 1
fi
EXPECTED_POLICY="$(jq -S . invoke-policy.json)" || exit 1
ACTUAL_POLICY="$(jq -S . <<<"$POLICY_CHECK")" || exit 1
[[ "$ACTUAL_POLICY" == "$EXPECTED_POLICY" ]] || { echo "policyが対象Functionの指定2 action/conditionと完全一致しません。送信せず停止してください。"; exit 1; }
VERIFY_START_MS="$(date +%s%3N)"
assume_lab_and_invoke
VERIFY_END_MS="$(date +%s%3N)"
aws logs filter-log-events --log-group-name "/aws/lambda/$STACK_NAME" --start-time "$VERIFY_START_MS" --end-time "$VERIFY_END_MS" --output json
```

HTTP 200と応答JSONの`requestId`を記録します。上記の時間帯に出力されるapp JSONから同じIDの`request_complete` logを探します。ログが未着なら、同じ`VERIFY_START_MS` / `VERIFY_END_MS`でread-only検索を最大10分まで繰り返し、HTTP requestは再送しません。

```bash
aws logs filter-log-events --log-group-name "/aws/lambda/$STACK_NAME" --start-time "$VERIFY_START_MS" --end-time "$VERIFY_END_MS" --output json
```

応答のrequestIdと一致する`request_complete` app JSON logがあることを確認します。LambdaのSTART/END/REPORT行とは区別してください。

### Deny

ログイン中のCloudShell identityへ戻ったことを`aws sts get-caller-identity`で確認してから、caller roleのinline invoke policyだけを一時的に削除します。

```bash
aws sts get-caller-identity --query Arn --output text
aws iam delete-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab
sleep 60
POLICY_CHECK="$(aws iam get-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab 2>&1)" && { echo "policyがまだ存在します。送信せず停止してください。"; exit 1; }
[[ "$POLICY_CHECK" == *"NoSuchEntity"* ]] || { printf '%s\n' "$POLICY_CHECK"; exit 1; }
VERIFY_START_MS="$(date +%s%3N)"
assume_lab_and_invoke
VERIFY_END_MS="$(date +%s%3N)"
aws logs filter-log-events --log-group-name "/aws/lambda/$STACK_NAME" --start-time "$VERIFY_START_MS" --end-time "$VERIFY_END_MS" --output json
```

IAM反映を最大60秒待ち、`get-role-policy`が`NoSuchEntity`であることを確認してから一度だけ送信します。出力やエラーが異なる場合は送信せず停止します。

実際に返ったHTTP status/bodyと表示されたAWS request IDを記録します。AWS request IDが応答headerにない場合は「返されず」と記録します。403等でresponseにapp `requestId`がない場合は「なし」と記録します。専用Lambdaを他から呼んでいない時間帯で、同じ時刻範囲にLambda START/アプリ完了logがないかCloudWatchで確認します。拒否が予想と異なれば追加requestを止めてidentity・Function URL・policy差分を調べます。

### Re-allow

同じinline policyを戻し、IAM反映をread-only確認しながら最大60秒待ちます。`get-role-policy`の内容が対象Function ARNと指定条件に合う2 actionだけだと確認してから送信します。

```bash
aws iam put-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab --policy-document file://invoke-policy.json
sleep 60
if ! POLICY_CHECK="$(aws iam get-role-policy --role-name "$LAB_ROLE_NAME" --policy-name S02InvokeFunctionUrlLab --query PolicyDocument --output json 2>&1)"; then
  printf 'policyを読み取れません。送信せず停止してください。\n%s\n' "$POLICY_CHECK"
  exit 1
fi
EXPECTED_POLICY="$(jq -S . invoke-policy.json)" || exit 1
ACTUAL_POLICY="$(jq -S . <<<"$POLICY_CHECK")" || exit 1
[[ "$ACTUAL_POLICY" == "$EXPECTED_POLICY" ]] || { echo "policyが対象Functionの指定2 action/conditionと完全一致しません。送信せず停止してください。"; exit 1; }
VERIFY_START_MS="$(date +%s%3N)"
assume_lab_and_invoke
VERIFY_END_MS="$(date +%s%3N)"
aws logs filter-log-events --log-group-name "/aws/lambda/$STACK_NAME" --start-time "$VERIFY_START_MS" --end-time "$VERIFY_END_MS" --output json
```

allow、deny、re-allowのすべてで同じ入力`{"sku":"SKU-RED-01","quantity":3}`を1回ずつ使います。denyではpolicy削除後に`get-role-policy`が`NoSuchEntity`であること、allow/re-allowではpolicy内容が対象Function ARNと指定条件に完全一致することを確認してから、各状態1回だけ送信します。`sleep 60`は待機時間であり伝播の証明ではありません。再試行は演習全体で最大1回です。すでに再試行を使った場合や、log遅延だけの場合は再送しません。

## 復元と削除

演習がdeny状態で止まったら、まずpolicyを戻してallow結果を確認してから削除します。必要な記録を許可された場所へ保存し、secret/session token/完全なURLを公開しません。

演習READMEの[cleanup節](../README.md#7-後片付けと残存確認)へ戻り、そこにあるcaller policy/role削除とstack cleanupを一度だけ実行します。role削除をこの節で重ねて実行しません。`get-role`が`NoSuchEntity`になり、stack、Lambda、Function URL、実行role、log groupの不在が確認できることを見届けます。失敗を完了扱いにせず、対象外resourceへ削除を広げません。

必要な証拠を安全な場所へ保存した後、検証用の`lab-trust.json`、`invoke-policy.json`、`run-evidence-v1.0.3-all.txt`を削除し、`test -e`で不在を確認します。保持する共有アプリsource cloneは削除対象ではありません。`unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_SECURITY_TOKEN`を実行し、shell historyに秘密値を含む入力がないことを確認します。STS role sessionは個別に削除できるとは扱わず、最後にassumeしたsessionのExpirationを記録します。file整理が終わってから自分が開始したCloudShell terminalの終了を確認し、その後に自分で開いたbrowser tabを閉じます。
