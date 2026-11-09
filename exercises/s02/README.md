# S2 初動判断演習：shop-api のエラー増加

## この演習で行うこと

架空の `shop-api` で5xxエラーが増え、`orders-db` に遅延が見つかりました。リリース展開と通常と異なるrole sessionの記録もあります。限られた情報から、確認できた事実、仮説、まだ分からないことを分け、担当と次の判断を記録します。

AWSアカウントへの接続、実resourceの変更、攻撃やスキャンは行いません。会社・サービス・役割・記録・時刻はすべて演習用の架空設定です。実際の運用へそのまま適用する手順ではありません。

## 使う資料

1. `scenario.md` を読みます。
2. `worksheet.md` を手元に用意します。
3. 12分を目安に自分で記入します。複数人なら役割を読み分けて相談できます。
4. 記入を終えてから `answer-guide.md` を開き、判断理由と未確認事項を照合します。

解答例は唯一の正解や自動採点ではありません。異なる選択でも、根拠、影響、担当、承認、戻す条件が説明でき、分からないことを断定していなければ、その違いを検討できます。

## 進め方の目安

| 時間 | 作業 |
|---:|---|
| 0〜2分 | 役割と本番変更の判断者を決める |
| 2〜5分 | 時系列を事実・仮説・未確認に分けて記録する |
| 5〜7分 | 読み取りだけで先に確認する証跡を3点選び、問いと担当を書く |
| 7〜9分 | 展開停止・変更を戻す・調査継続を条件付きで判断する |
| 9〜12分 | 関係者向けの4文更新と次回報告時刻を書く |

状況により進め方や優先順は変わります。証跡の確認を緊急の影響抑制より先に完了させる必要はありません。

## 参照

- この演習の資料: [`scenario.md`](scenario.md)、[`worksheet.md`](worksheet.md)、[`answer-guide.md`](answer-guide.md)
- CloudTrailの `userIdentity` とrole session: [AWS CloudTrail userIdentity element](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-user-identity.html)

CloudTrailイベントのidentity項目には任意項目があります。`sourceIdentity` はSTSの設定やrole引受時の指定などに依存し、すべてのイベントに必ずあるものではありません。記録されたsessionだけで実際の利用者や不正アクセスを断定せず、イベントの出所、時刻、項目の有無を確かめます。
