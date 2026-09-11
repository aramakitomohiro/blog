# Amazon Redshift × AWS Backup 公開ナレッジ

Amazon Redshiftのバックアップ、AWS Backup、ランサムウェア対策、RPO/RTO、費用、実環境検証について、AWS公式情報を基に整理した公開閲覧版

- 調査基準日：2026-09-10
- 成果物作成日：2026-09-11
- 対象リージョン：Asia Pacific (Tokyo) / ap-northeast-1
- 対象：Provisioned / RA3、Amazon Redshift Serverless、AWS Backup

## この公開版について

元のDOCX/XLSXを、GitHub上でそのまま読めるMarkdownへ変換

- DOCX：章立てと表をMarkdown化
- 費用試算XLSX：各シートの計算結果をMarkdownテーブル化
- 根拠一覧XLSX：AWS公式仕様とAWS CLI公式仕様をMarkdownテーブル化
- バイナリファイルのダウンロード不要

> 費用試算の公開版は作成時点の値を固定した閲覧用スナップショット。Excelの数式再計算、入力規則、色、書式は含まない。再試算には元の計算モデルが必要

## 調査報告書

1. [概要・結論・Redshift標準バックアップ](01_概要と標準バックアップ.md)
2. [AWS Backup・Snapshot比較・ランサムウェア対策](02_AWSBackupとランサムウェア対策.md)
3. [RPO / RTO・業務影響・費用体系・費用試算](03_RPO_RTOと費用.md)
4. [推奨構成・制約事項・AWS Support照会事項](04_推奨構成と制約.md)
5. [実環境検証方針・Definition of Done](05_実環境検証方針.md)
6. [AWS公式情報一覧](06_AWS公式情報一覧.md)

## 費用試算｜Excel公開閲覧版

7. [前提・入力パラメータ・Redshift料金・AWS Backup料金](07_費用試算_前提入力料金.md)
8. [Backup容量・Restore・Cross-Region](08_費用試算_容量復旧クロスリージョン.md)
9. [月額・年間・計算根拠・感応度](09_費用試算_月額年間計算根拠.md)

## AWS公式根拠｜Excel公開閲覧版

10. [AWS公式根拠 S01-S31](10_AWS公式根拠_S01-S31.md)
11. [AWS公式根拠 S32-S61](11_AWS公式根拠_S32-S61.md)
12. [AWS CLI公式根拠 C001-C052](12_AWS_CLI根拠_C001-C052.md)
13. [AWS CLI公式根拠 C054-C085](13_AWS_CLI根拠_C054-C085.md)

## 重要な前提

本資料はAWS公式公開情報を根拠にした調査・設計検討用成果物

実際のAWSアカウントでのBackup取得、Restore、攻撃模擬試験、RPO/RTO測定、料金測定は未実施。資料の完成は、本番採用の承認や実環境試験の完了を意味しない

公開版にはAWSアカウント固有の認証情報、秘密鍵、パスワード、実環境の内部IP、業務データを掲載しない運用を前提
