# AWS

| ドキュメント                                                                                   | 概要                                                                                                          |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| [Lambda で Web アプリケーションを公開するときの入口](lambda-web-app-entrypoint.md)             | CloudFront の後ろに API Gateway を置くか、Lambda 関数 URL を置くか、Lambda Web Functions を使うか             |
| [AWS で非同期処理とイベント配信を組むときの部品](async-messaging.md)                           | SQS 単体、SNS + SQS、EventBridge、Kinesis Data Streams のどれで組むか。受け手の数、順序、保持と読み直しの違い |
| [CloudWatch Logs のログを長期間保存するときの置き場所](cloudwatch-logs-long-term-retention.md) | CloudWatch Logs に置き続けるか、短期間だけ置いて S3 にアーカイブするか。保存期間ごとの損益分岐点              |
