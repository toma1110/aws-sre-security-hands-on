# S02判断例

この資料は考え方の一例です。実AWS runの観測値を与えるものではなく、自動採点もしません。手元で得た実際の応答・logと、架空Northstar Retailのtabletopを混ぜずに照らし合わせてください。

## 実アプリの読み方

- successの200はアプリが注文を処理した結果です。responseのrequestIdで同じIDの`request_complete` logを探します。
- validationの400はアプリの入力検証が返した応答です。IAM拒否ではありません。
- injected_failureの500はサンプル内で意図して発生させたアプリ例外です。AWS Lambdaサービス障害を示す証拠ではありません。
- bounded_delayの200では、固定された待ち時間とアプリ内`durationMs`を確認します。これはLambda請求時間と等しい値ではありません。
- untrusted_inputの200では固定例文が`untrustedInput`として記録されることを確認します。検知、実行、AI防御の結果と解釈しません。
- HTTP responseとアプリ完了logの照合には同一requestIdを使います。AWS LambdaのSTART/END/REPORT行とアプリJSON logは別の記録です。

実行前の例示JSONや期待statusは実測値ではありません。空欄や不一致は、実runで確認した観測をそのまま書き、期待値へ書き換えません。

## IAM許可・拒否・復元の判断

Function URLの`AuthType`は`AWS_IAM`のままです。caller roleに必要なのは対象関数への`lambda:InvokeFunctionUrl`と`lambda:InvokeFunction`です。後者は`lambda:InvokedViaFunctionUrl=true`条件でFunction URL経由に限定します。

1. **許可:** 専用caller roleで固定requestを送り、200とLambda requestIdを得ます。対応するアプリJSON logをrequestIdで確認します。
2. **拒否:** 専用callerからそのinvoke policyだけを外し、同じ入力を1回送ります。Function URLが認可段階で拒否した場合、403等のHTTP statusやAccessDenied本文が観測されることがあります。Lambda handlerへ届かなければ、Lambda requestIdやアプリ`request_complete` logはありません。AWS request identifierも実際に返った場合だけ記録します。
3. **復元:** 同じ2 action、同じResource、同じFunction URL条件を専用callerへ戻し、同じrequestを一度送ります。200、requestId、対応logが戻るかを確認します。

ここで示すのは、このFunction URLを呼び出すIAM許可の実験です。全AWS操作の権限、アカウントのsecurity posture、CloudTrailの有効性、攻撃や侵害の有無を結論しません。login role、permission set、アカウントassignment、既存callerやresource policyを変更する必要はありません。

## 拒否時に記録がない場合

拒否responseにLambda requestIdがなく、同時刻に該当アプリ完了logがないなら「Function URLの認可で拒否され、handler処理に到達した証拠はない」と記録できます。存在しないIDやlogを作らないでください。CloudTrailは元から有効でアクセス可能な場合に限り補助証跡として確認し、この演習のために新規有効化しません。

404やtimeoutなど予想と異なる結果の場合は、同じ入力を連打せず、Function URL、AWS identity、Region、policy、対象関数、CloudFormation状態を確認します。原因を切り分けてから再試行は最大1回です。

## cleanupと費用

専用caller roleの追加policyとroleを削除し、アプリstackを削除します。Lambda、Function URL、実行role、CloudWatch log group、caller role、policyの不在をそれぞれ確認します。CloudShell一時fileやprocessは必要な根拠を保存した後に整理します。CloudShell terminalを終了してからtabを閉じます。STS sessionの期限が残る場合は期限と利用可否を明記し、削除可能と断定しません。

Lambdaのrequest/実行時間、CloudWatch Logs取り込み/保存、resource稼働時間などの料金はregion・利用条件で異なります。Pricing Calculatorの仮定に基づく見積もり、確認できた実費、未確認の実費を分けて記録します。無料利用枠や厳密な費用上限は保証しません。

## 架空tabletopの判断例

開始カードだけでは、不正アクセス、データ漏えい、deploy起因の障害のいずれも確定しません。最初にincident lead、サービス担当、security担当、記録担当と変更承認者を決め、影響時間帯のdeploy、対象サービス、該当するrequest/log、role sessionのprincipal・送信元・操作記録を照合します。組織の保全方針に従いread-only証跡を保全します。

rollbackや展開停止は、利用者影響、変更との関連、切り戻しの副作用、判断権限を踏まえて責任者が決めます。選んだ措置、その期待効果、確認指標、rollback条件と次回報告時刻を記録します。これは架空の判断例で、実アプリの500やIAM拒否をNorthstarの証拠として使うものではありません。
