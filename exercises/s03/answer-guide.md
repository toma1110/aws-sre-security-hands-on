# S03 解答例と理由

先に[記入シート](worksheet.md)へ自分の判断を書いてから読んでください。以下は架空資料に基づく一つの案です。AWS変更やテストを実行した結果ではありません。

## 1. 調査対象

`NS-WAF-03`、架空資料A〜F（2026-11-16 JST作成）、`northstar-training / ap-northeast-1 / REGIONAL`、ALB `ns-shop-alb`、ACL `ns-checkout-acl / r0` を記録します。対象はgroup `NorthstarTrainingManagedGroup` 内 `SQLi_BODY_17`、`POST /checkout`、2026-11-15 10:00〜10:05 UTCです。

Web ACLではTemporaryDeny 0、group 10、rate guard 30。group内はSQLi 10、XSS 20です。小さい数から評価しますが、Web ACLとgroupのpriorityは別の並びです。初期状態のSQLiはBlockであり、一致すればgroup内XSSやWeb ACL後続へ進みません。

資料Bの3つのIDは `fictional-alb-base-a`、`fictional-alb-base-b`、`fictional-alb-base-c` です。A/Bの正常分類はfixture台帳、Cの不正形式分類はテスト担当の相関記録が根拠です。WAFのrule一致だけから分類しません。WAFのALB trace IDとアプリ独自IDは同一ではなく、別の相関記録で結びます。r0でBlockされた3件にアプリIDを補完しません。

## 2. 影響の分母

| 範囲 | 計算 | 読み取れること |
|---|---|---|
| 全path | 300 ÷ 10,000 = **3%** | ACL全体の最終Block率 |
| `POST /checkout` | 40 ÷ 200 = **20%** | 注文経路では全体より大きい影響 |
| 既知正常checkout | 30 ÷ 160 = **18.75%** | 分類済みの正常通信が止まった割合 |

40件すべてを誤検知とは呼びません。40件のうち30件が既知正常、10件が既知不審です。未分類20件の2xxは安全の証拠ではなく、WAF Allowも注文成立の証明ではありません。全pathの3%だけを見て影響が小さいとは判断できません。

## 3. 確認事実・仮説・未確認

| 確認事実 | 仮説 | 未確認と担当 |
|---|---|---|
| リリース後の資料では正常30件がSQLiでBlock | memo入力の変更が一致条件に影響した可能性 | 根本原因、アプリのSQL実装・修正候補はアプリ担当、10:45 UTC報告 |
| 正常A/Bと不審Cが同じSQLiで止まった | rule名だけでは正常と不審を区別できない | 全利用者・未分類20件の影響をService ownerが10:30に再確認 |
| WAFのBlockとアプリ2xx/4xxは別資料 | Countによる回復と防御低下が同時に起こり得る | その他path20件の分類をSecurity担当が10:30に報告 |

先に確保する例は、(1) ACL/group設定版・優先度・action、(2) 同じ時刻帯と対象IDのWAFログ必要項目、(3) fixture台帳とホストID・アプリ結果の安全な相関記録です。収集担当は当番SRE、分類と内容確認はテスト担当・アプリ担当、保存責任者はIncident leadとします。生bodyや顧客情報をticketへ転載する必要はありません。

## 4. Countで分かること

`r1` は `SQLi_BODY_17` 一つの `RuleActionOverrides` をCountにしています。これは非終了actionなので、group内の後続ruleも評価されます。正常A/Bは後続XSSとrateに一致せずDefault Allow、Cは後続XSSでBlockです。CountはAllow actionではありません。

group参照の **return actionだけ**をCountへ変える方式では、group内部は元の終了actionまで通常どおり評価されます。最初のSQLi Blockでgroup内部の評価が終わり、その戻り値だけがCountへ変わるため、同じrequestの後続XSS一致を調べる代わりにはなりません。

Count60件は最終Allow/Blockとは重なる集合です。Cを含む10件はCount後にXSSでBlockされます。`250 + 9750 = 10000` が最終actionの全件数で、そこへCount60を加えません。さらに、r1では他pathの20件が新しくAllowになり、分類未完了の通信へのSQLi防御が弱まっています。正常の回復だけでr1を継続する案には、この残るリスクへの判断が必要です。

`/checkout` を前段で広くAllowすると、その一致は終了actionになり、後続managed ruleやrate guardによる検査も受けません。group全体の無効化、またはscope-downでcheckoutをgroup全体から除く案も、この誤検知rule以外の検査を失います。今回の影響範囲より広いため採用しません。

## 5. 限定例外の案

概念上の設定案を `r2` とします。**この設定をAWSへ適用した実測はありません。**

1. TemporaryDeny priority 0とmanaged group priority 10を維持します。
2. group内 `SQLi_BODY_17` のみCount上書きを維持し、label Lを得ます。他のgroup内ruleはBlockのままです。
3. groupより後、rate guardより前の **Web ACL priority 20** に独自rule `SQLiExceptKnownCheckout` を置く案とします。条件は **Lに一致 AND NOT E**、actionはBlockです。labelを付ける前へ配置すると照合できません。
4. Eに含まれる通信にもAllow actionは付けません。SQLiの再Block条件からだけ除き、後続のrate guard等を引き続き評価し、終了ruleに一致しなければDefault Allowとなります。

Eの内容は資料Eのpath、method、Content-Type、JSON `/memo` の2固定値の完全一致です。SQLi rule自体にpath条件付きoverrideを設定したという説明ではありません。group全体を検査対象から外す案でもありません。

この例外は正常fixtureの回復範囲を狭く定義した暫定策です。クライアントが同じmemoを送ることは可能で、他fieldの不正入力を許す残余リスクがあります。認証、アプリ入力検証、パラメーター化されたSQL等の根本対策の確認を別途進めます。未知の正常入力、JSON解析失敗、検査上限超過、text transformationの設定や実装可能性を確認せず、本番で安全と結論しません。

| 確認ケース | 期待する評価と最終action | 確認に使う資料・結果 |
|---|---|---|
| 正常A/B | SQLi Count+L → XSS非一致 → L AND NOT E非一致 → rate非一致 → Default Allow | Count一致、最終Allow、ホストIDと別アプリIDの相関、2xx・注文処理の結果 |
| 不審C | SQLi Count+L → group内XSS Block | group内XSSの停止記録、最終Block、アプリ未到達 |
| E外のSQLi一致、他rule非一致 | SQLi Count+L → priority 20でBlock | 独自rule停止、最終Block、アプリ未到達 |
| rate guard対象（前段に停止条件なし） | group等を通過 → priority 30でBlock | rate rule停止、最終Block、アプリ未到達 |
| 一時IP拒否対象 | priority 0でBlock | TemporaryDeny停止、後続評価を期待しない、アプリ未到達 |

資料Eと同じ交通構成で、他の条件も維持できると仮定すると、r2の**予測**はcheckout Block10/Allow190、その他 Block260/Allow9540、全体 Block270/Allow9730です。SQLi Count60件のうち、C等10件はXSS、他path20件は後続独自ruleで止まります。一時IP拒否を追加した後の件数は、影響するrequestの重複が資料にないため算出しません。

当番SREが担当、Service ownerが承認、Security担当が不審通信と例外範囲を確認します。10:30 UTCに再評価し、期限は11:00。根本原因の調査・修正は別に続けます。

## 6. 期限付きIP拒否の案

一例として資料Fの相関記録に基づき `203.0.113.77/32` だけを `ns-temp-deny` へ入れる案を選びます。IP拒否は正常A/Bの誤検知を直す処置ではなく、繰り返しの不正形式に対する別の暫定策です。共有NATや送信元変更のリスクが残ります。見送る場合も、追加確認、既存rate guardによる監視、再判断時刻を示せれば比較できます。

priority 0の `TemporaryDeny` が先にBlockするため、対象requestはSQLi Countやpriority 20まで進みません。正常な共有利用を止める兆候が出たら、このentryだけを戻す判断を先に行います。

担当Security、確認は当番SRE、承認はService owner。開始10:20、見直し10:30、解除期限10:40 **UTC**を記録します。期限前に担当不在ならIncident leadへ引き継ぎます。延長には再評価した根拠、新期限、承認が必要です。

期限になってもIP set entryは自動で消えません。担当が対象entryを除去し、ACL/ruleの対象と残るaddressを確認します。今回専用のsetとruleを廃止するなら、正確な参照ruleを解除し、Web ACL・rule group等に参照がないことを確認してから専用setを削除、不在を確認します。共有setや他entryを削除しません。除去後のWAF最終action・他防御の継続、正常通信とアプリ結果、確認者と時刻を記録します。これは机上の終了計画であり、削除実績ではありません。

## 7. 監視・復帰・切り戻しと引き継ぎ

設定版、開始・終了時刻、集計窓を固定し、全pathとcheckout、既知正常・不審・未分類を別々に比較します。WAFの最終action、SQLi Count、後続独自Block、XSS/rate/IPの停止をログと対応するmetricで確認します。metricにはscope/dimension・Sumを残し、path別はログ等で集計します。アプリ側では成功、入力検証4xx、5xx、注文処理の結果を別に追います。

資料Fの条件を使い、既知正常Block、既知不審の意図しないAllow、他pathのSQLi防御欠落、アプリ5xx率の悪化を見たら追加展開を停止します。Service ownerとSecurityが判断し、当番SREが原因となる変更だけを保存済みの設定へ切り戻す計画です。IP拒否が原因ならそのentry、r2のSQLi変更が原因なら独自ruleと個別overrideが対象です。SQLiについては既知の `r0` に戻すと正常30件のBlockが再発し得るため、復旧担当・連絡先を同時に確保します。広いAllowで解決せず、サービス影響を伴う安全側の切り戻しであることを共有します。

**期限付き例外から元のSQLi Blockへ戻す場合:** 原因修正の版と根拠を得て、正常A/B、E外、不審、rateの期待結果を満たすことを別途検証し、Service owner・Securityが復帰を判断します。group内SQLiの個別Count overrideを元へ戻した後に、不要になった後続label ruleを除去する計画です。Countのまま先に後続Blockを消すと、E外のSQLi一致が通る空白が生じます。label方式を暫定継続する間は元ruleのCountを保つ必要があり、「Blockへ戻す」を単に後続rule削除とは扱いません。実変更では伝播中の差も監視し、設定反映と期待actionを確認します。

完了根拠は、原因修正と復帰設定の確認、正常回復と不審Block・他path防御・アプリ結果、期限付きentry/例外の解除確認、Service owner確認です。未分類通信や残余リスクには担当と次回確認日を残します。資料の予測値だけで本番の修正完了にはしません。

連絡例:

> Service ownerとSecurity担当へ。架空演習NS-WAF-03では、10:00〜10:05 UTCのcheckout200件中40件がBlockされ、うち30件は既知正常でした。SQLi一つのCount観察では正常が回復する一方、他pathの未分類20件もAllowへ変わるため、L AND NOT Eを後続でBlockする限定案を提案します。例外は当番SREが11:00まで管理し、一時IP拒否を採用する場合はSecurity担当が10:40までに解除を確認します。原因とSQL実装、未分類通信、共有NATの影響は未確認であり、意図しないAllowや正常Block等を認めたら追加展開を止め、保存した設定への切り戻しを判断します。次回報告は10:30 UTCです。
