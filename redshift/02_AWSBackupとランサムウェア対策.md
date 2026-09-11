# Redshift × AWS Backup｜調査報告書（公開Markdown版）

# 6. AWS Backup概要

| **AWS Backup一般の機能**   | **意味**                                     | **Redshiftでの扱い**                               |
|----------------------------|----------------------------------------------|----------------------------------------------------|
| Backup Plan / Rule         | 取得時刻・保持・コピー等の方針と個別ルール   | 両形態のmanual取得と保持に利用可能                 |
| Backup Selection           | ARN、resource type、tag条件で保護元を選ぶ    | cluster/namespaceを指定。Tag誤変更は保護漏れリスク |
| Backup Vault               | RPとアクセス/保持制御をまとめる管理容器      | 利用可能。元snapshot鍵からの独立暗号化ではない     |
| Recovery Point             | 復旧できる時点のバックアップと関連metadata   | 実体はRedshift manual snapshot                     |
| Retention / Lifecycle      | 何日保持し、いつ削除・cold移行するか         | 保持/期限削除を使用。Coldは本設計で未採用          |
| Cross-Region Copy          | 対応型を別RegionのVaultへcopy                | SS対象外。Provisionedは固有対応が未確認            |
| Cross-account Copy         | 対応型を別accountへcopy                      | SS対象外。Provisioned未確認。native共有と区別      |
| Vault Lock                 | 保持中の削除・短縮を制限                     | 共通機能として利用。実環境のnative削除経路も試験   |
| Logically air-gapped vault | 対応型向けの分離保管、Compliance、共有復旧等 | SS対象外。Provisioned未確認。汎用説明を流用不可    |
| AWS Organizations          | 組織のbackup policyや統合管理                | 共通管理機能。個別resourceのcopy対応は増えない     |
| Backup Audit Manager       | framework/controlで準拠状況を評価            | SS対象外。Provisioned未確認。ログ監査は別に実施    |
| Restore / Restore Job      | 選んだRPから復旧処理を管理                   | 全体と単一表に対応。型ごとの制約を継承             |
| 暗号化                     | 独立暗号化か元service継承かは型ごとに異なる  | Redshiftは元serviceのKMS鍵を継承                   |

根拠：\[S01\]、\[S02\]、\[S18\]、\[S19\]、\[S21\]、\[S37\]、\[S38\]、\[S39\]、\[S58\]、\[S59\]

【AWS公式仕様】Backup Ruleはcronの取得時刻と開始window・完了windowを持つ。予定時刻はsnapshotのデータ時刻や完了時刻ではない。開始windowには60分以上の指定制約があり、予定時刻から遅れて開始する場合もある。\[C059、C061\]

【設計上の解釈】OrganizationsやAudit Managerは運用統制を助ける仕組み。データ復旧の成功そのものを証明するものではない。SCPも組織の管理アカウントやservice-linked role等の例外があるため、万能な拒否壁としない。\[S53\]

# 7. Redshift × AWS Backup

## 7.1 保護単位とObjectの対応

| **段階**   | **Provisioned**                                                                        | **Serverless**                                                                          |
|------------|----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| 保護元     | cluster ARN：arn:aws:redshift:ap-northeast-1:\<Account\>:cluster:\<ClusterIdentifier\> | namespaceArn：arn:aws:redshift-serverless:ap-northeast-1:\<Account\>:namespace/\<UUID\> |
| Selection  | cluster ARNか限定tag条件。ClusterNamespaceArnは使用しない                              | get-namespaceのnamespaceArn。snapshot ARNは保護元ではない                               |
| 取得処理   | StartBackupJob / Plan Ruleがmanual snapshotを作成・管理                                | 同様にnamespace全体のmanual snapshot                                                    |
| Backup Job | BackupJobId、State、CreationDate、CompletionDateを確認                                 | 同左。service固有のResourceType表記を確認                                               |
| 復旧点     | RecoveryPointArnとnative snapshot ID/ARN・時刻を突合                                   | RecoveryPointArnとnamespace snapshotを突合                                              |
| 保管       | VaultにRP管理情報、実体はRedshift snapshot                                             | 同左。元KMSを継承                                                                       |
| 復旧       | CLUSTER_RESTOREまたはTABLE_RESTORE                                                     | NAMESPACE_RESTOREまたは表復旧                                                           |

【AWS公式仕様】StartBackupJobの応答にはJob IDと作成時刻が返る。応答時のRecoveryPointArn返却は一部service向けであり、RedshiftのRPは完了後のDescribeBackupJobやVault一覧で取得する。BackupSizeInBytesはRedshiftで未提供。空欄から容量ゼロを推定しない。\[S03、C051、C061\]

【未確認】Serverless取得ガイドのBackupSelection例にsnapshot ARNがある一方、保護単位の説明とオンデマンド例はnamespace。保護元の固有説明を優先し、namespace指定の成功をPhase 2/3で確認する。コマンド参照のResourceType列挙にも遅れがあり得るため、固有例・実際のAPI応答を照合する。\[S04、S06、C041、C062\]

## 7.2 利用できる範囲の判定表

| **機能**                                          | **Provisioned**          | **Serverless**     | **根拠/扱い**                                  |
|---------------------------------------------------|--------------------------|--------------------|------------------------------------------------|
| On-demand / Plan / Rule                           | 対応                     | 対応               | S03/S04。manualのみ                            |
| ARN/Tag Selection                                 | 共通方式で利用           | 共通方式で利用     | S37。実選択の証跡必須                          |
| Vault / Job / RP / Restore Job                    | 対応                     | 対応               | S01/S03〜S06                                   |
| 全体 / 単一表Restore                              | 対応、native制約あり     | 対応、全体は上書き | S05/S06/S09/S10                                |
| Retention / Delete lifecycle                      | 対応                     | 対応               | S01/S38。具体保持は実環境確認                  |
| 独立AWS Backup暗号化                              | 対象外                   | 対象外             | S18。元Redshift鍵を継承                        |
| PITR / continuous                                 | 対象外                   | 対象外             | S03/S04ではmanual。S02                         |
| AWS Backup copy（Region/account/同Region別Vault） | 公式情報では確認できない | 対象外             | S02のcopy空欄を個別に優先                      |
| Cold Storage                                      | 公式情報では確認できない | 対象外             | S02。未採用・未計上                            |
| Logically air-gapped vault                        | 公式情報では確認できない | 対象外             | S02。汎用Vaultの存在だけでは対応証明にならない |
| Vault Lock                                        | 共通機能の対象           | 共通機能の対象     | S02/S19。鍵・account閉鎖は保護範囲外           |
| Organizations                                     | 共通統合管理             | 共通統合管理       | S02/S39。copyの対応とは別                      |
| Backup Audit Manager                              | 公式情報では確認できない | 対象外             | S02個別表を共通説明より優先                    |
| 自動Restore testing                               | 対象外                   | 対象外             | S45対応リストに含まれない。手動訓練は可能      |
| Backup search / index等                           | Redshift固有対応を未確認 | 対象外             | S02/C061。EBS/S3等向け機能を流用しない         |

【AWS公式仕様】機能表のServerless「incremental backup」欄は空欄。一方、Redshift固有の保存・料金資料は差分/unique blockを説明する。これは「AWS Backup機能表上の増分対応」と「Redshift保存層・請求上の増分性」の文脈が異なるため、空欄から毎回フル保存課金と結論しない。\[S02、S07、S23、S29\]

## 7.3 復旧設定とデータ範囲

【AWS公式仕様】クラスター/namespace全体を取得し、表だけを独立してバックアップする選択はできない。RA3/RG/ServerlessではBACKUP NOによる除外指定が効かずバックアップ対象となる。外部S3の実データ、ETLコード、IAM/VPC等の復元は別設計。\[S01、S03、S35\]

| **復旧後の確認項目**      | **必要な確認**                                                                                                       |
|---------------------------|----------------------------------------------------------------------------------------------------------------------|
| Provisioned新規resource   | ClusterIdentifier、Endpoint、AZ、node、subnet group、SG、parameter group、IAM role、暗号化鍵。省略時の既定設定に注意 |
| Serverless既存resource    | 上書き先namespaceの名称/ID、関連workgroup、endpoint、RPU、SG/subnet、role。データ上書きの対象を二重確認              |
| Database / Schema / Table | 一覧、row count、代表行、SUM等の業務集計。件数だけでは改ざんを検出できない                                           |
| View / Permission         | view依存、grant、owner、role、外部キー。単一表復旧では再作成が必要となる項目を確認                                   |
| パスワード/Secret         | 管理パスワード利用時はManageMasterPasswordとSecrets Manager/KMS権限を確認                                            |
| 接続と性能                | BI/ETLのendpoint切替、認証、外部連携、代表クエリの応答時間。AWS job成功だけで完了にしない                            |

根拠：\[S05\]、\[S06\]、\[S09\]、\[S10\]、\[S13\]、\[S18\]

# 8. Snapshot比較

【設計上の解釈】標準manualとAWS Backupは、保存データの別方式というより、同じRedshift snapshotを誰が計画・管理・保護するかの違い。併用時は無料の短期保護と、統制された有料manualの役割を分ける。

| **比較項目**       | **A. Redshift標準**               | **B. AWS Backup**                  | **用途/制約**                         |
|--------------------|-----------------------------------|------------------------------------|---------------------------------------|
| 管理               | Redshift Console/API              | Backup Plan/Vault/Jobを中心に管理  | 対象ObjectはPhase4で突合              |
| スケジュール       | 自動取得、manual schedule         | Rule cronとwindow                  | 予定時刻とデータ時刻は別              |
| 保持               | 自動/RPの制約、manual保持         | Rule/Lifecycle＋Vault保持条件      | Lockによる長期保管費を見込む          |
| 取得単位           | cluster/namespace                 | 同左、manual snapshot              | 表単独Backupは不可                    |
| 復旧単位           | 全体または表                      | 全体または表をJob管理              | 型ごとのnative制約を継承              |
| Cross-Region       | native copyあり                   | SS対象外、Provisioned未確認        | 標準経路を別設計                      |
| Cross-account      | manual共有・復旧                  | SS copy対象外、Provisioned未確認   | 共有を独立copyとしない                |
| 削除防止           | IAM/KMS/運用統制                  | Vault policy / IAM / Vault Lock    | 鍵障害と分離評価                      |
| イミュータビリティ | 通常manual保持だけではなし        | Compliance猶予後に保持中の削除保護 | 通常VaultやGovernanceだけで確定しない |
| 管理者侵害         | 通常manualを削除可能              | Complianceは削除抑止に有用         | KMS、正常性、閉鎖は残余リスク         |
| 監査               | Redshift Events/CloudTrail/DB監査 | Backup/Restore Job、CloudTrail追加 | SQL操作はDB監査が必要                 |
| 一元管理           | Redshift内が中心                  | 対応service間で統合                | 未対応機能が増えるわけではない        |
| 運用負荷           | 小規模では単純                    | 計画/role/Vault/監視の追加管理     | 規模拡大で統制を共通化                |
| RPO                | 自動/RPの頻度、manual計画         | Rule頻度、window、ジョブ成功状況   | 正常点と検知遅延が重要                |
| RTO                | native復元＋設定/データ/接続      | native復元をBackup Jobから管理     | 固定保証時間は確認できない            |
| 料金               | Compute/RMS/manual保存/転送       | 同じsnapshot保存をRedshiftへ計上   | 二重計上禁止                          |
| ランサムウェア     | 正常世代が残れば復旧候補          | 削除保護と統制を追加できる         | 暗号鍵やアカウント全体は別対策        |
| 主な制約           | 短期保持、手動削除、共有依存      | 未対応高度機能、元鍵依存           | 要件に応じて組合せ                    |

根拠：\[S01\]、\[S02\]、\[S07\]、\[S08\]、\[S18\]、\[S19\]、\[S22\]

## Phase4で採取する比較証跡

同じ検証元からA=標準manual、B=AWS Backupを取得。Object名、Snapshot ID/ARN、RecoveryPointArn、作成/完了時刻、暗号化鍵、保持、削除主体、復元先、CloudTrail、料金UsageTypeを並べる。「Vault一覧にある」と「別の保存実体がもう一つある」を混同しない。

# 9. メリット／デメリット

| **観点** | **追加で改善すること**                     | **追加しても改善しないこと**                 |
|----------|--------------------------------------------|----------------------------------------------|
| 管理     | 複数resourceの計画、保持、ジョブ監視の統合 | 元のSQL改ざんや悪いデータの識別              |
| 保護     | Vault Lockによる保持中の削除・短縮の拒否   | 元CMKの無効化/永久削除、アカウント閉鎖       |
| 監査     | JobとRPに基づく取得漏れ・復旧記録          | DB操作ログが無効な環境のSQL追跡              |
| 復旧     | 復旧対象選択や復旧Jobの手順共通化          | NW/IAM/ETL/BI切替と業務照合の自動完了        |
| 頻度     | 長期manual取得の自動化                     | 標準RPより細かなPITRの追加                   |
| コピー   | 対応する他serviceのcopy一元化              | Redshift未対応/未確認のcopyやair-gapの有効化 |

【設計上の解釈】併用デメリットは、同じ時刻の重複取得、保持ルールの競合、role/Tag/Vault/鍵の運用増、Lock満了まで消せない費用、標準とBackupの画面を突合する負担。追加のmanual保管量は増え得るが、単に「AWS Backupを使ったのでフルデータをもう1組」という計算にはしない。

【実環境確認事項】既存manualの取得理由を棚卸しし、重複計画を統合。月次/年次と日次が同時刻の場合のルール最適化や保持選択は、実際のCreatedBy/Rule/期限で確認する。\[S38\]

# 10. ランサムウェア対策

以下は【設計上の解釈】による脅威評価。◯=条件を満たせば有効、△=部分的/依存あり、×=防げない、―=非適用。Vault Lock列はComplianceの猶予後を前提。Cross-account列は標準snapshot共有の評価。Air-gapped列はRedshift Serverless対象外、Provisioned未確認のため防御層として数えない。\[S02、S16、S18–S21、S31、S53\]

| **攻撃シナリオ** | **Redshift標準** | **AWS Backup**    | **Vault Lock** | **Cross-account** | **Air-gapped Vault** | **復旧可能性**      | **注意点**             |
|------------------|------------------|-------------------|----------------|-------------------|----------------------|---------------------|------------------------|
| データ削除       | △正常点          | △正常点           | ◯点を保持      | △共有依存         | 対象外/未確認        | 正常点/鍵あり       | 削除後の取得点では不可 |
| 大量改ざん       | △正常点          | △正常点           | ◯旧点保持      | △共有依存         | 同上                 | 最終正常点次第      | 潜伏期間＞保持は失敗   |
| resource削除     | △manual          | ◯manual管理       | ◯削除抑止      | △所有元依存       | 同上                 | manual/鍵あり       | 自動のみでは不足       |
| snapshot削除     | ×権限次第        | △管理/制御        | ◯保護期待      | △元が取消可       | 同上                 | 別正常点次第        | native迂回を実測       |
| Backup RP削除    | ―                | ×Lockなし         | ◯拒否          | △                 | 同上                 | 保護点あれば可      | 期限満了は正常削除     |
| IAM奪取          | △最小権限        | △最小権限         | ◯保持中        | △別境界次第       | 同上                 | 権限範囲次第        | IAM修正権限も評価      |
| AWS管理者侵害    | ×通常削除        | △Lock次第         | ◯削除耐性      | △共有は弱い       | 同上                 | 鍵/正常点次第       | 鍵まで使えなければ不可 |
| Vault削除        | ―                | △空でなければ拒否 | ◯RP保持中      | ―                 | 同上                 | RP/鍵あり           | 空Vaultと区別          |
| CMK無効化        | ×復号不能        | ×元鍵依存         | ×鍵は保護外    | △元鍵依存         | 同上                 | 再有効化後候補      | キャッシュ遅延等を実測 |
| CMK永久削除      | ×通常復号不可    | ×元鍵依存         | ×鍵は保護外    | △独立鍵必要       | 同上                 | 独立コピー/原本次第 | 同名alias再作成は無効  |
| account全体侵害  | ×単一境界        | △部分保護         | △閉鎖例外      | △共有のみ不足     | 同上                 | 独立再構築次第      | 閉鎖90日後の削除例外   |

## 10.1 管理者を奪われても残る保護と限界

【AWS公式仕様】Governanceは必要権限を持つ主体が解除可能。Complianceは指定した猶予を過ぎると、保持中のバックアップに対する削除・保持短縮やLock変更をroot等でも行えない。Locked=trueだけでは最終Complianceの証明にならず、LockDate、モード、猶予経過を確認する。保持満了による削除は許容される。\[S19、C063\]

【AWS公式仕様】アカウントを閉鎖した場合、90日の猶予後にバックアップが削除される説明があり、Vault Lockもアカウント閉鎖を永続保管へ変えるものではない。KMS key policy、鍵の無効化・削除、アカウント管理権限の統制が別途必要。\[S19、S31\]

【実環境確認事項】BackupのアクセスpolicyがあるだけでRedshift native APIからの削除も防げると断定しない。専用の犠牲snapshotでnative削除経路とBackup削除経路の両方を試験し、エラー、主体、対象、時刻を保存。削除に成功した場合は保護設計を不合格とする。\[S20\]

## 10.2 対策の組合せ

【設計上の解釈】通常運用roleからsnapshot/RP削除、KMS Disable/ScheduleKeyDeletion、Vault変更を分離。緊急roleの発動と監査を別管理にし、OrganizationsのSCPは適用範囲と例外を確認する。予防に加え、Backup失敗、保持変更、鍵状態変更、snapshot削除、CloudTrail停止を監視する。

【設計上の解釈】C案では、元アカウントと元鍵を失っても利用できる別管理の原本またはUNLOADデータとDDL・構成コードを保管し、再ロード復旧を実証する。S3 Object LockはS3 objectの保護であり、Redshift native snapshotへの直接設定ではない。これは補助的な再構築経路で、snapshotと同じ復旧範囲/RTOではない。\[S50–S52\]
