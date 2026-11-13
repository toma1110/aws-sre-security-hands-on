# S02シナリオ：注文受付の応答と呼び出し権限

## 学習目標

観測したHTTP応答、Lambda request ID、CloudWatch Logsを一組ずつ照合し、アプリ内のエラーと呼び出し前のIAM拒否を区別します。変更対象を演習専用caller roleに限定し、同じ入力による復元確認とcleanupを行います。

## 実アプリ演習の事実

教材で指定する共有アプリ `aws-observability-sample-app v1.0.3` は、Python 3.12 Lambda、AWS_IAM認証Function URL、実行role、CloudWatch Logsを使います。固定runnerは次の5ケースを送ります。

| ケース | 期待される応答 | 読むポイント |
|---|---:|---|
| success | 200 | 注文処理の成功、応答のrequestIdと完了log |
| validation | 400 | 入力検証エラー。関数自体の到達失敗ではない |
| injected_failure | 500 | アプリ内で意図的に発生させた例外。Lambdaサービス障害ではない |
| bounded_delay | 200 | 上限付き1.5秒待機。アプリ計測durationと課金時間を混同しない |
| untrusted_input | 200 | 固定fixtureの文字列がlogへ記録されたこと。攻撃検知ではない |

実行前に使用account、東京Region、stack名とowner、Function URLを確認します。各ケースは一度だけ実行し、responseのrequestIdと同じIDのCloudWatchアプリJSON logを記録します。固定手順は公開配布READMEに従います。

## IAM比較のケース

実アプリの5ケースを終えたら、新しく作成した専用caller roleを使います。成功入力 `{"sku":"SKU-RED-01","quantity":3}` を固定し、許可→拒否→再許可の各状態で一度ずつ送ります。

| 状態 | 変更 | 観測すること |
|---|---|---|
| 許可 | `lambda:InvokeFunctionUrl`と、Function URL経由に限定した`lambda:InvokeFunction`を対象Lambdaだけに許可 | 200、本文、requestId、同一IDのアプリ完了log |
| 拒否 | 専用caller roleから呼び出しpolicyだけを外す | HTTP拒否、AccessDenied等の本文、時刻、AWS request identifierが返る場合はその値。アプリrequestId/logの有無をそのまま記録 |
| 復元 | 同じ最小policyを専用caller roleへ戻す | 同一入力で200、requestId、対応logが再び得られること |

この比較では、講師/受講者の既存identity、AWS login/session、permission set、account assignment、Function URLのAWS_IAM認証設定を変更しません。Function URLで拒否されたrequestはLambda handlerへ届かず、アプリのrequestIdや完了logが作られない場合があります。これを架空のアプリログで補いません。

## 架空のNorthstar Retail tabletop

以下は実AWS演習と分けた架空状況です。`shop-api`、`orders-db`、Pager alert、role sessionはサンプルアプリの実resource・ログ・実障害ではありません。

### 開始カード

架空の`shop-api`で5xxが増加したとPagerが通知し、`orders-db`の応答も遅くなっています。「不正アクセスの可能性」とありますが、攻撃者、データ漏えい、原因はまだ確認されていません。

### 追加情報

1. アラートの18分前にdeployが始まっています。リリース担当はrequest解析の変更を説明しましたが、rollback判断はまだです。
2. セキュリティ担当が通常と異なるrole sessionを見つけました。principalと送信元は未照合です。

### 進め方（12分）

1. Incident lead、サービス担当、セキュリティ担当、記録担当を決め、本番変更の判断者を確認します。
2. 時系列を「観測事実」「仮説」「未確認事項」に分けます。
3. 先に保全するread-only証跡を3つ選び、問いと担当を決めます。
4. rollout停止、rollback、調査継続を条件付きで判断し、承認者と戻す条件を記録します。
5. 影響・未確認事項・担当・次回報告時刻を含む関係者向け更新を書きます。

この架空scenarioへ実アプリのrequest、500、IAM拒否、logを証拠として流用しません。実測記録とtabletop記録は別に保管します。
