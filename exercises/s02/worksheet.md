# S02ワークシート

実AWS演習と架空tabletopは別々に記録します。secret、credential、顧客情報、完全なFunction URLはこの教材や公開場所へ記載しないでください。実記録は組織の管理ルールに沿う場所へ保存してください。

## 実AWS runの識別

| 項目 | 記録 |
|---|---|
| 固定アプリtag / commit | `v1.0.3` / `4c47b860e4c373629f93bed87eb1effa481565fe` |
| 実行日時（UTCまたはtimezone付き） | |
| OS / shell / CLI版 | |
| 開始directory | |
| AWS accountの識別（公開しない） / caller identity | |
| Region / stack名 / owner tag | |
| Function URL確認済み | はい / いいえ（URL値自体はここへ貼らない） |
| 料金見積もりの前提と確認先 | |

## 5ケースの応答とログ

実行した各ケースについて、status・必要な応答項目・requestId・対応するアプリJSON logの有無を記録します。長い本文や秘密値を転記せず、必要な箇所だけ残します。

| ケース | 実測status | 応答requestId | 同じIDのアプリlog | 観測した要点 / 未確認 |
|---|---:|---|---|---|
| success | | | | |
| validation | | | | |
| injected_failure | | | | |
| bounded_delay | | | | |
| untrusted_input | | | | |

### 専用callerの許可・拒否・復元

3回とも入力は `{"sku":"SKU-RED-01","quantity":3}` とします。拒否時にrequestIdやapp logがない場合は「未到達/生成なし」と書き、値を推測しません。

| 状態 | HTTP/statusと必要な応答 | Lambda requestId | app完了log | policy差分 / 時刻 / AWS request identifier |
|---|---|---|---|---|
| 許可（変更前） | | | | |
| 拒否（専用caller policy除去） | | | | |
| 復元（同じ最小policy） | | | | |

### 判断メモ

- 400または500はどの層で返りましたか。観測根拠は何ですか。
- IAM拒否時にアプリへ届いたことを示す証拠はありますか。
- 同一入力の復元後、何が戻り、何が確認できませんでしたか。
- `durationMs`とLambda課金時間を区別できていますか。
- untrusted inputの記録から推測できないことは何ですか。

### cleanup確認

| 対象 | 削除または不在確認 |
|---|---|
| 専用caller roleとinline/attached policy | |
| CloudFormation stack | |
| Lambda function / Function URL / 実行role | |
| CloudWatch Logs log group | |
| 一時証跡・process・環境変数・shell history | |
| CloudShell terminal終了確認 / 残存するSTS sessionの有効期限 | |

実費が未確認なら未確認と記録します。見積もりと実請求額を同じものとして記載しません。

## 架空Northstar Retail tabletop

これは実AWS runと別の架空scenarioです。

| 時刻 | 観測事実（根拠） | 仮説 | 未確認事項 | 担当 |
|---|---|---|---|---|
| | | | | |

- 影響範囲と、現在確認できていない範囲:
- 先に保全するread-only証跡3点と各担当:
- 暫定判断、承認者、期待する効果:
- rollback条件と確認方法:
- 次回報告時刻:
- 関係者向け更新（事実・不確実性・次の行動を明記）:
