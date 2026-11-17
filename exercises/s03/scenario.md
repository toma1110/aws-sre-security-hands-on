# S03 状況と調査資料

## 開始カード

Northstar Retailのリリース後、注文画面から「注文を送信できない」という問い合わせが届きました。サポートは3件のrequest IDを渡し、Product担当は「復旧のために `/checkout` をすべてAllowにしたい」と求めています。あなたは当番として、誤検知の影響と変更案を整理します。

**問い:** 何を根拠に変更対象を絞り、どの結果が得られたら継続・復帰・切り戻しを判断しますか。

以下の資料A〜Fはすべて架空です。資料作成日は2026-11-16 JST、公式仕様の確認日は2026-11-15 JST、事件の時刻は2026-11-15 **UTC**です。架空rule、label、IDは説明用の識別子で、実在するAWS Managed Rules名や、そのまま実行できる設定ではありません。公式sampleログの転載でもAWS実測でもありません。本文、顧客情報、認証情報は含めていません。

## 資料A：対象と変更前の評価順

| 項目 | 架空の設定 |
|---|---|
| 演習ID | `NS-WAF-03` |
| account alias / region / scope | `northstar-training` / `ap-northeast-1` / `REGIONAL` |
| Web ACL / 設定版 | `ns-checkout-acl` / `r0` |
| 保護対象 | Application Load Balancer `ns-shop-alb`（共有アプリとは別の架空構成） |
| WAFログの資料元 | 演習用WAF記録表。アプリログとは別の資料 |
| Default action | Allow |

| Web ACL内priority（小さい数が先） | ルール・参照先 | action / 状態 |
|---:|---|---|
| 0 | `TemporaryDeny`：一時IP set `ns-temp-deny` | Block。変更前のIP setは空 |
| 10 | managed rule group `NorthstarTrainingManagedGroup` | 個別上書きなし、group return actionの上書きなし |
| 30 | `CheckoutRateGuard` | 既存のrate-based Block。条件はこの演習では変更しない |

priority 10のgroup内は、priority 10の `SQLi_BODY_17`（Block）、priority 20の `XSS_BODY_04`（Block）の順です。priority番号の比較は同じWeb ACL内または同じgroup内で行います。group内部の番号を、Web ACL内の番号と直接比較しません。

`SQLi_BODY_17` は一致時に架空label `awswaf:managed:fictional:NorthstarTrainingManagedGroup:SQLi_BODY_17` を付けます。以降、この全文を **L** と呼びます。実環境では、提供元の現行rule名・version・label全文を別途確認する必要があります。

## 資料B：問い合わせと変更前のWAF記録

全行が `ns-checkout-acl / r0`、`POST /checkout`、2026-11-15 UTCです。時刻とホスト側request IDを照合済みという架空設定です。

| 時刻 | WAFのrequest ID（ALBのtrace IDに相当する架空値） | group内の停止rule / action | 補足資料による分類 |
|---|---|---|---|
| 10:01:11 | `fictional-alb-base-a` | `SQLi_BODY_17` / Block | 正常A：固定の既知正常fixture。memoは `gift's ready` |
| 10:02:12 | `fictional-alb-base-b` | `SQLi_BODY_17` / Block | 正常B：固定の既知正常fixture。memoは `customer's bag` |
| 10:03:13 | `fictional-alb-base-c` | `SQLi_BODY_17` / Block | 不審C：テスト担当が不正形式と確認。正常fixtureの条件を満たさない |

この表はWAFログの必要項目を教材用に整理したものです。実ログJSONそのものではありません。実ログでは `webaclId`、`httpRequest.uri`、`httpRequest.httpMethod`、`httpRequest.requestId`、`timestamp`、最終 `action` と `terminatingRuleId`、`ruleGroupList` 内のrule情報を照合します。priorityはACL/groupの設定資料から確認します。通常のWAFログにpriorityが常に記載されると仮定しません。

WAFのrequest IDはホストサービスが生成します。ALBではtrace IDです。アプリが独自に作るrequest IDや、s02のLambda request IDと同じとは限りません。資料の分類は、テスト担当のfixture台帳と安全な相関記録を併用した結果です。rule名や403だけで正常・攻撃を判定したものではありません。

## 資料C：影響の集計（変更前r0）

時間範囲は **10:00以上10:05未満 UTC**。架空の全件WAF記録を、ACL・設定版・時刻で限定して集計しました。path別の表はログ分析結果であり、標準CloudWatchメトリクスにURIのdimensionがあるという意味ではありません。

| 範囲 | 全request | 最終Block | 最終Allow |
|---|---:|---:|---:|
| 全path | 10,000 | 300 | 9,700 |
| `POST /checkout` | 200 | 40 | 160 |
| その他 | 9,800 | 260 | 9,540 |

`POST /checkout` の200件をfixture台帳・相関記録で分類すると次のようになります。資料Bの3件はこの集計の一部であり、3件だけから200件へ外挿したものではありません。

| 分類 | 全request | 最終Block | 最終Allow | アプリ結果（Allowの通信だけ） |
|---|---:|---:|---:|---|
| 既知正常 | 160 | 30 | 130 | 130件が2xx |
| 既知不審・不正形式 | 20 | 10 | 10 | 10件が入力検証4xx |
| 未分類 | 20 | 0 | 20 | 20件が2xx。正常性・安全性は未確認 |

別資料のCloudWatch相当の**架空metric集計**は、同じ5分・Sumで、最終 `BlockedRequests=300`、`AllowedRequests=9700`。group内 `SQLi_BODY_17` のBlock一致60件、`XSS_BODY_04` のBlock一致90件、既存rate ruleのBlock150件です。`SQLi_BODY_17` の60件はcheckout40件とその他20件です。各ruleのmetric scope/dimensionと最終ACL actionの集計を区別します。

アプリ2xx、入力検証4xxはWAFログから読み取った値ではなく、架空のアプリ集計と相関資料による値です。WAFのAllowは注文成立や安全性を保証しません。Blockされた3件について、アプリ内request IDや完了ログを作り足さないでください。

## 資料D：変更前に分かっていること・足りないこと

- リリース記録には、正常fixtureのmemo入力を追加したとあります。既知正常160件のうち30件が `SQLi_BODY_17` で止まりました。原因の完全な特定、アプリのSQL実装、全利用者への影響は未確認です。
- 正常A/Bは `POST /checkout`、`Content-Type: application/json`、小さなJSON bodyの `/memo` が上表のどちらかの固定値と完全一致するfixtureです。不審Cはこの条件を満たしません。検証用bodyはWAFの検査上限内という前提です。未知の入力やoversize時の安全性は未確認です。
- 顧客の生bodyを集めず、既知正常・不審fixtureの識別、ログの必要項目、アプリ側の入力検証と相関結果を先に確保できます。
- Incident leadは当番SRE、判断・変更案の承認者はService owner、セキュリティ確認者はSecurity担当とします。一人で解く場合も、記録上の役割は分けます。

**ここで記入シート1〜3を書いてから、追加資料へ進んでください。**

## 資料E：Count観察で得た追加情報

以下は、別時間帯の**架空の観察結果**です。あなたがAWS変更を行う手順ではありません。

`r1`（10:10以上10:15未満 UTC）は、group内 `SQLi_BODY_17` **だけ**を `RuleActionOverrides` でCountにした設定です。上書き自体にはpath/method条件がなく、すべての対象pathに及びます。他ruleとrate guardはBlockを維持、group return actionは変更していません。この5分は比較用にrequest数・分類・fixture構成が資料Cと同じという架空条件です。実トラフィックで同じ構成が保証されるとは扱いません。

| 同じ入力の別試行 | WAF request ID | `SQLi_BODY_17` | その後の評価と最終action | 別の相関資料で確認したアプリ結果 |
|---|---|---|---|---|
| 正常A | `fictional-alb-count-a` | Count、Lあり | XSSとrateに一致せずDefault Allow | `fictional-app-count-a` と対応、2xx |
| 正常B | `fictional-alb-count-b` | Count、Lあり | XSSとrateに一致せずDefault Allow | `fictional-app-count-b` と対応、2xx |
| 不審C | `fictional-alb-count-c` | Count、Lあり | group内の後続 `XSS_BODY_04` でBlock | 未到達。アプリIDなし |

実ログではCount一致は `nonTerminatingMatchingRules`（group内なら `ruleGroupList` 内）、上書き前のactionは該当項目の `overriddenAction`、最終actionはトップレベルの `action` 等で確認します。Countされたrequestが最終Allowとは限りません。

| r1の範囲 | 全request | 最終Block | 最終Allow | `SQLi_BODY_17` のCount一致 |
|---|---:|---:|---:|---:|
| 全path | 10,000 | 250 | 9,750 | 60 |
| `POST /checkout` | 200 | 10 | 190 | 40 |
| その他 | 9,800 | 240 | 9,560 | 20 |

checkoutの既知正常は160件すべて2xxへ、不審20件は10件が後続XSSでBlock・10件がアプリ入力検証4xxへ、未分類20件は2xxです。Count60件には後続でBlockされた10件も含まれ、最終Allow/Blockと足してrequest総数を求めることはできません。その他pathの20件はCount化で新しくAllowになり、正常・不審の分類は未完了です。

限定例外の検討に使う条件を **E** と呼びます。Eは「URIが完全一致 `/checkout` AND methodがPOST AND `Content-Type` が `application/json` AND JSON `/memo` が `gift's ready` または `customer's bag` と完全一致」です。大文字小文字や変換、JSON解析失敗・oversize時の扱いも実装時の確認対象です。ここでは既知正常160件はEに含まれ、既知不審20件・未分類20件・その他pathはEに含まれないという資料上の条件とします。

Eは送信者の認証やpayload全体の安全性を証明しません。他のfieldに有害な入力があってもEに入るリスク、未知の正常入力を回復できないリスクが残ります。WAFで完全なアプリschema検証を実装できたとは扱いません。

## 資料F：期限付き拒否と復帰の条件

Security担当は、別の相関記録から架空送信元 `203.0.113.77/32` による繰り返しの不正形式を確認しました。1件のrule一致だけで送信元を攻撃者と断定した結果ではありません。現在の記録には共有NATの正常利用を示す証拠はありませんが、共用の可能性は未評価です。このアドレスは文書用の例であり、通信を送ってはいけません。

- Service ownerは、必要ならこの **1 addressだけ**の一時拒否案を検討できます。既存rate guardの変更や、広いCIDRの追加は依頼していません。
- 想定する開始は10:20 UTC、最初の見直しは10:30、解除期限は10:40。担当はSecurity担当、変更・解除の確認は当番SREが行います。継続が必要なら期限前に根拠と承認を更新します。
- 限定例外の担当は当番SRE、見直しは10:30、期限は11:00。アプリ担当は原因調査・入力処理の修正候補を10:45に報告する予定です。調査が完了したという情報はまだありません。
- 暫定変更を認める場合、既知正常A/B、既知不審C、E外のSQLi一致fixture、rate guard対象fixture、一時IP拒否対象fixtureで期待結果を先に定めます。テストの提案を記録するだけで、ここでは実行しません。
- 仮の運用基準として、5分窓で既知正常のWAF Blockが1件以上、既知不審の意図しないAllowが1件以上、他pathでSQLi一致が意図せずAllowになった場合、またはcheckoutのアプリ5xx率が変更前基準1%を超えた場合、追加展開を止めて調査・切り戻し判断を行います。これは本演習で与えた基準であり、AWSの推奨値や全サービス共通値ではありません。

IP set entryには本演習の期限を自動処理する仕組みはありません。設定が自動失効する前提を置かず、解除、必要なら参照解除後の専用set削除、設定とログの再確認まで担当と時刻を記録してください。

## 仕様を確かめる公式資料

以下の英語原文を2026-11-15 JSTに確認しました。公開日・更新日は各本文に表示がないため、確認日と分け、推測の日付を付けていません。資料A〜Fはこれらの公式sampleではありません。

| 出典 | この演習で参照する仕様 |
|---|---|
| [Rule group action overrides](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-rule-group-override-options.html) | 個別ruleの上書きとgroup return actionの上書きは別。Countは非終了、Allow/Blockは終了action |
| [Setting rule priority](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-processing-order.html) | 同一範囲内でpriorityの小さい数から評価 |
| [Web request labeling](https://docs.aws.amazon.com/waf/latest/developerguide/waf-labels.html) | labelは後続ruleから照合でき、個別Countと後続label ruleで例外を組み立てられる |
| [Logging](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html) / [Log fields](https://docs.aws.amazon.com/waf/latest/developerguide/logging-fields.html) | WAFログとアプリログの区別、最終action、rule情報、ホストサービス生成のrequest ID |
| [Monitoring and tuning](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-testing-activities.html) / [Metrics and dimensions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-metrics.html) | rule一致・label・最終actionの監視、Sumと集計範囲。URI別の判断にはログ等を併用 |
| [Managing IP sets](https://docs.aws.amazon.com/waf/latest/developerguide/waf-ip-set-managing.html) | CIDRのsetをruleから参照する。削除前に参照を確認する |

実環境への構築・適用、現在のmanaged rule version、例外条件の実装、性能・費用、伝播中の挙動は本演習では検証していません。
