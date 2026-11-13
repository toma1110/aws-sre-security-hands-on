# AWS SREのためのセキュリティ演習資料

このrepositoryには、注文受付サンプルアプリを使うS02実践と、架空の状況を使う判断演習があります。

- **視聴のみ:** AWSアカウントは不要です。講義で示す検証済みの画面・出力を使って学べます。
- **自分で実践:** 組織から利用を認められた、専用または隔離済みの非本番AWSアカウントが必要です。AWS資源とログに料金が発生する可能性があります。実施前にアカウント、Region、権限、料金要因、cleanupを確認してください。
- **Northstar Retail tabletop:** 架空の資産・出来事を使う机上演習です。AWSへ接続しません。

S02実践は、固定版アプリの取得から、5ケースの応答・ログ照合、専用callerのIAM許可・拒否・復元、resourceの削除と残存確認までを順に実行します。手順・停止条件・前提・cleanupは[演習README](exercises/s02/README.md)を先頭から読んでください。

AWS実践では、受講者自身が許可されたアカウントで取得した一時sessionを使います。顧客データや長期credentialを使わず、password、MFA、secret、temporary credentialを教材や記録へ貼り付けたり公開したりしないでください。Function URLはAWS_IAM認証のままにし、既存のlogin roleやpermission setを変更しません。

実測したアプリ出力と、架空の事件・資産・logを混ぜないでください。演習資料の記入例は実行結果や自動採点ではありません。
