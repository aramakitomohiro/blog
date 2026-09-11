# AWS Technical Research

AWS公式情報をベースに作成した、バックアップ設計・セキュリティ・費用試算・実環境検証向けの公開ナレッジ

## Amazon Redshift × AWS Backup

調査基準日：2026-09-10

**[Redshift × AWS Backup 公開ナレッジを開く](redshift/README.md)**

報告書、費用試算、AWS公式根拠、AWS CLI根拠をすべてMarkdown化し、GitHubのブラウザ画面だけで閲覧できる構成

### 公開方針

- DOCX/XLSXのダウンロードを前提にしない
- 文書はMarkdown、スプレッドシートはMarkdownテーブルとして公開
- 費用試算テーブルは作成時点の値を固定した閲覧用スナップショット
- AWSアカウント固有情報、認証情報、秘密鍵、パスワード、実環境の内部情報は公開しない

### 注意

本成果物はAWS公式公開情報を根拠とした調査・設計検討用資料。実環境でのBackup取得、Restore、攻撃模擬試験、料金測定の完了や、本番採用の承認を意味しない
