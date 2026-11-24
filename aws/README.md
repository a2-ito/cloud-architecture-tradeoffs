# AWS

| ドキュメント                                                                                   | 概要                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Lambda で Web アプリケーションを公開するときの入口](lambda-web-app-entrypoint.md)             | CloudFront の後ろに API Gateway を置くか、Lambda 関数 URL を置くか、Lambda Web Functions を使うか                                                                                |
| [Lambda を内部 API として呼び出すときの入口](lambda-internal-api.md)                           | PrivateLink + API Gateway、Lambda Web Functions、Lambda 関数 URL、内部 ALB のどれを入口にするか。インターネットから到達できるかどうかの違い                                      |
| [CloudWatch Logs のログを長期間保存するときの置き場所](cloudwatch-logs-long-term-retention.md) | CloudWatch Logs に置き続けるか、短期間だけ置いて S3 にアーカイブするか。保存期間ごとの損益分岐点                                                                                 |
| [AWS 上のアプリケーションから Slack に通知を送る仕組み](slack-notification.md)                 | SNS + Amazon Q Developer in chat applications（旧 AWS Chatbot）、Slack app の Incoming Webhook、Bot トークン + chat.postMessage のどれで送るか。認証情報の置き場所と表現力の違い |
