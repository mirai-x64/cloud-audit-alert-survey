# 監査ログアラートの既存製品調査と自作可否の検討

**前提**: Mikage32さん個人が運用する一般用途の AWS / GCP / Cloudflare 環境
(企業向けSOCではない、個人〜小規模の開発・運用スケール)に対して、
「IAM変更・不審なAPI呼び出し・権限昇格」などの監査ログイベントを
検知して通知するアラートシステムが欲しい。

**結論を先に書く**: **自作は不要。** 各クラウドのネイティブ機能だけで
個人スケールの要件は無料〜数百円/月で満たせる。三社をまとめて見たい
という欲求だけなら OSS の Wazuh を足せばよいが、それも「自作」では
なく既存OSSの設置で済む。独自のプロダクトを書く理由が無い。

---

## 1. 各クラウドのネイティブ機能(個人スケールでの第一候補)

### AWS
- **CloudTrail**: 管理イベントは過去90日分が追加料金なしで既定有効
  (Event History)。恒久保存・リアルタイム処理が必要なら S3 への
  Trail 配信(数十円/月)を追加する。
- **EventBridge + SNS**: CloudTrail/GuardDuty のイベントパターンを
  EventBridge のルールでフィルタし、SNS でメール/Slack/Discord
  Webhookへ飛ばす。サーバ管理ゼロ、処理したイベント分だけの課金。
  個人用途なら実質無料〜数十円/月。
- **GuardDuty**: 脅威検知の前払い費用はゼロ、分析イベント量に応じた
  従量課金のみ。Finding ごとに EventBridge イベントが飛ぶので
  上記の通知経路にそのまま載せられる。
- **CloudWatch Alarms**: CloudTrailをCloudWatch Logsに送ってメトリクス
  フィルタ+アラームという経路も可(IAM変更検知の定番パターンとして
  AWS公式ドキュメントにサンプルあり)。

個人用途の第一候補: **CloudTrail(無料のEvent History) → EventBridge
ルール → SNS → Discord/Slack Webhook**。追加コストはほぼゼロ。

### GCP
- **Cloud Audit Logs**: Admin Activity ログは常時有効・無料・400日保持。
- **Cloud Monitoring のログベースアラート**: 特定のログメッセージや
  ログベースメトリクスに対してアラートポリシーを組める
  (IAM変更・リソース削除・不審なデータアクセスの検知が公式に
  想定されたユースケース)。
- **料金**: Cloud Logging は月50GiBまで無料、Cloud Monitoring の
  メトリクス取り込みも月150MiBまで無料。個人スケールなら
  まず無料枠に収まる。
  - 注意: 2027-09-01以降、アラートポリシー内のメトリクス参照ごとに
    月$0.35の課金が始まる予定とアナウンスされている(現時点ではまだ
    無料)。個人用途でも頭に入れておく価値がある。
- **Security Command Center**: 企業向けの上位ティアは有料だが、
  個人用途では Cloud Monitoring のログベースアラートで十分
  (SCCはIAM変更単体の通知が目的なら過剰)。

### Cloudflare
- **Audit Logs**: 全プランで標準提供。APIで取得可能。
- 弱点: Cloudflareの「Notifications」機能は特定のセキュリティイベント
  (例: WAFブロック急増など)向けで、汎用の監査ログ全般への
  アラートには対応していない。IAM変更など監査ログ全体を監視したい
  場合は **Logpush で監査ログをS3/GCSなど外部ストレージへ出力し、
  そこをトリガーに Lambda/Cloud Functions で判定・通知する**
  という自前の配線が必要になる。
  - ここは三社の中で唯一「ネイティブだけでは完結しない」部分。
  - ただし必要なのは数十行のスクリプト(Logpush先のオブジェクト
    作成イベント→パース→パターンマッチ→Webhook)であり、
    これは「アラートシステムの自作」というより「配線」の範疇。

---

## 2. OSS / マネージドSIEMの選択肢(三社を一枚のダッシュボードで見たい場合)

個人の複数クラウドをまとめて1箇所で見たい、という欲求が出た場合の
既存製品:

- **Wazuh**(GPLv2, 無料・セルフホスト): AWS(CloudTrail/GuardDuty/
  VPC Flow Logs)・GCP・Cloudflareのログを取り込むモジュールが
  標準搭載。2026時点でOSS SIEMの中では「組み立てが最も少なくて
  済む」との評価が複数の比較記事で一致している。個人用途なら
  小さいVPS(月数百〜千円)に載せられる規模。
- **Cloud Custodian**: AWS/Azure/GCPを横断したポリシーベースの
  監視・通知。「ルールに違反したらアクションを実行する」用途に強く、
  ログの集中ダッシュボードというより定期実行のポリシーエンジン。
- **Panther / Elastic Security / Security Onion**: いずれも既存製品
  として機能は揃っているが、Pantherは商用(OSSではない)、
  Elastic/Security Onionは企業〜チーム向けの運用コストが
  個人用途には明らかに過剰。

個人スケールでの判断: **まずはネイティブ機能(1.)だけで十分**。
三社をまとめたダッシュボードが本当に要るようになったら Wazuh を
足す、で足りる。Panther/Elastic級の導入は不要。

---

## 3. 自作の必要性についての検討

調べた範囲で「個人〜小規模のソロ開発者向けに、AWS/GCP/Cloudflareを
横断した監査ログアラートに特化した軽量な既存プロダクト」という
ニッチは見当たらなかった(検索で出てくるのは企業向けSIEM/CSPMか、
OSSの汎用スキャナ/ポリシーエンジンのみ)。一見すると
「ニッチが空いている=事業機会」に見えるが、以下の理由で
**自作もBenzenGamesの事業候補としても優先度は低い**と判断する。

1. **個人スケールの需要が各クラウドのネイティブ機能(ほぼ無料)で
   既に満たされている。** 有料で乗り換えるインセンティブが弱い
   市場 = 事業として立ち上げる根拠も弱い。
2. **Cloudflareの配線を除けば「自作」する部分がほぼ無い。**
   EventBridge/Cloud Monitoringのルール設定で終わる。
3. Wazuhのような既存OSSが「複数クラウドを一枚で見る」というユーザー
   メリットをすでに無料で提供しており、差別化の軸が見えない。

**今回の個人用途での実務的な結論**: CloudTrail→EventBridge→SNS、
Cloud Audit Logs→Cloud Monitoringログベースアラート、
Cloudflare Logpush→軽量webhook変換、の3本をそれぞれDiscordへの
通知に向ける。合計で数十行の設定(Terraform/CLIどちらでも可)で
完結し、月額コストはほぼゼロ(Cloudflareの配信先に使うLambda/
Cloud Functionsの無料枠内)。

## 参考

- [Creating CloudWatch alarms for CloudTrail events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudwatch-alarms-for-cloudtrail.html)
- [Amazon GuardDuty Best Practices](https://aws.github.io/aws-security-services-best-practices/guides/guardduty/)
- [Google Cloud Observability pricing](https://cloud.google.com/monitoring/pricing)
- [Cloudflare Logpush integration](https://developers.cloudflare.com/cloudflare-one/insights/logs/logpush/)
- [Access authentication logs (Cloudflare audit logs)](https://developers.cloudflare.com/cloudflare-one/insights/logs/audit-logs/)
- [The (Definitive) List of Open Source SIEM Platforms Worth Evaluating](https://filigran.io/guides/detect-and-hunt/open-source-siem-platforms)
- [Self-Hosted SIEM Guide: Wazuh vs Security Onion vs Elastic Security](https://www.pistack.xyz/posts/self-hosted-siem-wazuh-security-onion-elastic-guide/)
