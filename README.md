# RDS-Aurora-GlobalDatabase-PosgreSQL-OperationPoC-CLI
Aurora Global Databaseの運用に関する検証



# 前提

下記手順で作成したAurora Global Database(PostgreSQL)環境があること
https://github.com/Noppy/RDS-Aurora-GlobalDatabase-PosgreSQL-CLI


#　検証手順
## (1) 事前準備
### (1)-(a) 作業環境の準備
下記を準備します。
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定
### (1)-(b) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE="default" #AdministratorAccess権限のあるプロファイルを指定
export PRIMARY_REGION="ap-northeast-1"    #プライマリ側リージョン指定(この手順では東京リージョン)
export SECONDARY_REGION="ap-northeast-3"  #セカンダリ側リージョン指定(この手順では大阪リージョン)
```
### (1)-(c) VPC構成情報の設定
```shell
#プライマリリージョン
Primary_VpcId=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')
Primary_PrivateSubnet1Id=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')
Primary_PrivateSubnet2Id=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')
Primary_PrivateSubnet3Id=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet3Id`].[OutputValue]')
Primary_Client_SGId=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-Client \
        --query 'Stacks[].Outputs[?OutputKey==`SGId`].[OutputValue]')

echo -e "Primary_VpcId= $Primary_VpcId\nPrimary_PrivateSubnet1Id = ${Primary_PrivateSubnet1Id}\nPrimary_PrivateSubnet2Id = ${Primary_PrivateSubnet2Id}\nPrimary_PrivateSubnet3Id = ${Primary_PrivateSubnet3Id}\nPrimary_Client_SGId = ${Primary_Client_SGId}"
```
セカンダリリージョンのVP設定情報取得
```shell
#セカンダリリージョン
Secondary_VpcId=$(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')
Secondary_PrivateSubnet1Id=$(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')
Secondary_PrivateSubnet2Id=$(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')
Secondary_PrivateSubnet3Id=$(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet3Id`].[OutputValue]')
Secondary_Client_SGId=$(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name Aurora-GlobalDB-Client \
        --query 'Stacks[].Outputs[?OutputKey==`SGId`].[OutputValue]')

echo -e "Secondary_VpcId= $Secondary_VpcId\nSecondary_PrivateSubnet1Id = ${Secondary_PrivateSubnet1Id}\nSecondary_PrivateSubnet2Id = ${Secondary_PrivateSubnet2Id}\nSecondary_PrivateSubnet3Id = ${Secondary_PrivateSubnet3Id}\nSecondary_Client_SGId = ${Secondary_Client_SGId}"

```
### (1)-(d) その他構成情報
```shell
#SG情報
PRIMARY_RDS_SG_ID=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
  ec2 describe-security-groups \
    --filters 'Name=group-name,Values=RdsSG' \
  --query 'SecurityGroups[0].GroupId' )

SECONDARY_RDS_SG_ID=$(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
  ec2 describe-security-groups \
    --filters 'Name=group-name,Values=RdsSG' \
  --query 'SecurityGroups[0].GroupId' )

echo -n "PRIMARY_RDS_SG_ID = ${PRIMARY_RDS_SG_ID}\nSECONDARY_RDS_SG_ID = ${SECONDARY_RDS_SG_ID}"

#KMS情報
ACCOUNT_ID=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
  sts get-caller-identity --query 'Account')
PRIMARY_KEY_ID="arn:aws:kms:${PRIMARY_REGION}:${ACCOUNT_ID}:alias/Key_For_Aurora"
SECONDARY_KEY_ID="arn:aws:kms:${SECONDARY_REGION}:${ACCOUNT_ID}:alias/Key_For_Aurora"
echo -n "PRIMARY_KEY_ID = ${PRIMARY_KEY_ID}\nSECONDARY_KEY_ID = ${SECONDARY_KEY_ID}"

#IAM Role情報
# RDS用IAMロールのARN取得
RDS_MONITOR_ROLE_ARN=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
    iam get-role \
        --role-name RdsEnhancedMonitoringRole \
    --query 'Role.Arn' )
echo $RDS_MONITOR_ROLE_ARN
```

### (1)-(e) RDS情報設定
実環境に合わせて変更すること。
```shell
#サブネットグループ
RDS_SUBNETGROUP_NAME="aurorapoc-subnetgroup"       #全て小文字で設定
#パラメータグループ
RDS_PARAMETERGROUP_NAME="aurorapoc-parametergroup" #全て小文字で設定
RDS_PARAMETER_GROUP_FAMILY="aurora-postgresql13"
#DBクラスター/エンジン/バージョン
RDS_GLOBALDB_NAME='aurorapoc-globaldb-1'
RDS_PRIMARY_DBCLUSTER_NAME="${RDS_GLOBALDB_NAME}-primary-cluster"
RDS_SECONDARY_DBCLUSTER_NAME="${RDS_GLOBALDB_NAME}-secondary-cluster"
RDS_ENGINE_NAME='aurora-postgresql'
RDS_ENGINE_VERSION='13.4'

#DBインスタンス設定
RDS_INSTANCE_CLASS='db.r5.large'
RDS_STORAGE_SIZE='100'
```



## (2) バックアップ・リストア検証
### (2)-(a) バックアップ説明

Aurora DBには以下の２つのバックアップがある。
- 自動バックアップ
  - 自動バックアップでは以下の２つのバックアップデータが取得され、S3に保管される(*1)
    - バックアップウィンドウで取得されるスナップショット(システムスナップショット)
    - 随時取得されるジャーナル(WALのバックアップ)
- 手動バックアップ
  - 手動で取得するスナップショット


- *1 [Aurora DB バックアップと復元の概要](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Backups.html)


### (2)-(b) 自動バックアップ検証(スナップショットからのリストア)
#### (i) バックアップ状況の確認
自動バックアップで取得したスナップショットの確認
```shell
aws --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
    rds describe-db-cluster-snapshots \
      --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}" \
      --snapshot-type "automated"
```

手動バックアップで取得したスナップショットの確認
```shell
aws --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
    rds describe-db-cluster-snapshots \
      --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}" \
      --snapshot-type "manual"
```
#### (ii) (プライマリー側)自動バックアップからDBクラスターをリストア(スナップショットからのリストア)
プライマリーリージョンでスナップショットから新しいDBクラスターを作成する。<br>ここで作成するのは、Global Databaseではなく、プライマリーリージョンで単独のDBクラスターを作成している。
```shell
#リストアの基となるスナップショットのARNを指定
#Identifierは、先のaws rds describe-db-cluster-snapshotsコマンドで確認
#表示した中の"DBClusterSnapshotIdentifier"がARN情報)
SOURCE_SNAPSHOT_IDENTIFIER="<対象のスナップショットのARN>"


#スナップショットからDBクラスター作成
aws --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
  rds restore-db-cluster-from-snapshot \
    --snapshot-identifier "${SOURCE_SNAPSHOT_IDENTIFIER}" \
    --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}-from-snapshot" \
    --engine "${RDS_ENGINE_NAME}" \
    --engine-version "${RDS_ENGINE_VERSION}" \
    --db-cluster-parameter-group-name "${RDS_PARAMETERGROUP_NAME}" \
    --vpc-security-group-ids "${PRIMARY_RDS_SG_ID}" \
    --db-subnet-group-name "${RDS_SUBNETGROUP_NAME}" \
    --kms-key-id  "${PRIMARY_KEY_ID}" \
    --engine-mode "provisioned"
```

#### (ii) (プライマリー側)作成したDBクラスターにインスタンスを追加
```shell
#DBインスタンスの作成
for az in $(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
        rds describe-db-subnet-groups \
        --query 'DBSubnetGroups[?DBSubnetGroupName==`'"${RDS_SUBNETGROUP_NAME}"'`].Subnets[].SubnetAvailabilityZone.Name');
do
  azlabel="$(echo ${az} | cut -f 3 -d '-')"
  aws --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
    rds create-db-instance \
      --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}-from-snapshot" \
      --db-instance-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}-from-snapshot-${azlabel}" \
      --availability-zone "${az}" \
      --db-instance-class "${RDS_INSTANCE_CLASS}" \
      --engine "${RDS_ENGINE_NAME}" \
      --engine-version "${RDS_ENGINE_VERSION}" \
      --db-parameter-group-name "${RDS_PARAMETERGROUP_NAME}" \
      --no-publicly-accessible \
      --no-auto-minor-version-upgrade \
      --preferred-maintenance-window "Mon:15:00-Mon:15:30" \
      --monitoring-interval 1 \
      --monitoring-role-arn ${RDS_MONITOR_ROLE_ARN} ;
done
```
### (iii) Globalクラスター作成
DBクラスターおよびDBインスタンス(ライターインスタンス)が`Available`になったらGlobalクラスターを作成する。
```shell
# Primary DBクラスターのARN取得
SRC_DB_CLUSTER_ARN=$(aws --profile ${PROFILE} --region ${PRIMARY_REGION} --output text \
  rds describe-db-clusters \
    --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}-from-snapshot" \
  --query 'DBClusters[].DBClusterArn' )
echo "SRC_DB_CLUSTER_ARN = $SRC_DB_CLUSTER_ARN"


# Globalクラスターの作成
aws --profile ${PROFILE} --region ${PRIMARY_REGION} \
    rds create-global-cluster \
        --global-cluster-identifier "${RDS_GLOBALDB_NAME}-from-snapshot" \
        --source-db-cluster-identifier "${SRC_DB_CLUSTER_ARN}" \
        --no-deletion-protection 
```

#### (iv) (セカンダリ側)DBクラスターの作成
```shell
#DBクラスターの作成
aws --profile ${PROFILE} --region ${SECONDARY_REGION} --no-cli-pager \
  rds create-db-cluster \
    --global-cluster-identifier "${RDS_GLOBALDB_NAME}-from-snapshot" \
    --db-cluster-identifier "${RDS_SECONDARY_DBCLUSTER_NAME}-from-snapshot" \
    --engine "${RDS_ENGINE_NAME}" \
    --engine-version "${RDS_ENGINE_VERSION}" \
    --db-cluster-parameter-group-name "${RDS_PARAMETERGROUP_NAME}" \
    --vpc-security-group-ids "${SECONDARY_RDS_SG_ID}" \
    --db-subnet-group-name "${RDS_SUBNETGROUP_NAME}" \
    --kms-key-id "${SECONDARY_KEY_ID}" \
    --backup-retention-period 3 \
    --preferred-backup-window "17:30-18:00" \
    --preferred-maintenance-window "Mon:19:00-Mon:19:30" ;
```

#### (v) (セカンダリ側)DBインスタンスの作成
```shell
#DBインスタンスの作成
for az in $(aws --profile ${PROFILE} --region ${SECONDARY_REGION} --output text \
        rds describe-db-subnet-groups \
        --query 'DBSubnetGroups[?DBSubnetGroupName==`'"${RDS_SUBNETGROUP_NAME}"'`].Subnets[].SubnetAvailabilityZone.Name');
do
    azlabel="$(echo ${az} | cut -f 3 -d '-')"
    aws --profile ${PROFILE} --region ${SECONDARY_REGION} --no-cli-pager \
        rds create-db-instance \
            --db-cluster-identifier "${RDS_SECONDARY_DBCLUSTER_NAME}-from-snapshot" \
            --db-instance-identifier "${RDS_SECONDARY_DBCLUSTER_NAME}-from-snapshot-${azlabel}" \
            --availability-zone "${az}" \
            --db-instance-class "${RDS_INSTANCE_CLASS}" \
            --engine "${RDS_ENGINE_NAME}" \
            --engine-version "${RDS_ENGINE_VERSION}" \
            --db-parameter-group-name "${RDS_PARAMETERGROUP_NAME}" \
            --no-publicly-accessible \
            --no-auto-minor-version-upgrade \
            --preferred-maintenance-window "Mon:19:00-Mon:19:30" \
            --monitoring-interval 1 \
            --monitoring-role-arn ${RDS_MONITOR_ROLE_ARN} ;
done
```






## (3)フェイルオーバーテスト
### (6)-(a)プライマリーリージョン内でのインスタンスフェイルオーバーテスト
以下は作業用コンソールで実行します。事前に以下のパラメータ設定をしている前提とします。
- `(1)-(b) CLI実行用の事前準備`のプロファイルとリージョンに関する変数
- `(4)-(a) RDS設定`のRDBに関する変数

```shell
#フェイルオーバーの実行
aws --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
    rds failover-db-cluster \
        --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}"
```

指定したインスタンスを昇格したい場合は以下のコマンドを実行
```shell
#フェイルオーバーの実行
aws --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
    rds failover-db-cluster \
        --db-cluster-identifier "${RDS_PRIMARY_DBCLUSTER_NAME}" \
        --target-db-instance-identifier "<昇格させたいDBインスタンスの識別子>"
```

### (6)-(b)グローバルデータベースのセカンダリリージョンへのフェイルオーバーテスト(計画切替)
計画切り替えの場合は、プライマリーリージョンで操作します。
#### (i)プライマリーリージョンからセカンダリリージョンへのフェイルオーバー
```shell
SECONDARY_DB_CLUSTER_ARN=$(aws rds --profile ${PROFILE} --region ${PRIMARY_REGION} describe-global-clusters | \
    jq -r '
        .GlobalClusters[] | 
        select(.GlobalClusterIdentifier == "'${RDS_GLOBALDB_NAME}'").GlobalClusterMembers[] |
        select(.IsWriter == false).DBClusterArn'
)

#フェイルオーバー
aws rds --profile ${PROFILE} --region ${PRIMARY_REGION} --no-cli-pager \
   failover-global-cluster \
    --global-cluster-identifier "${RDS_GLOBALDB_NAME}" \
    --target-db-cluster-identifier "${SECONDARY_DB_CLUSTER_ARN}"
```
#### (ii)セカンダリリージョンからプライマリーリージョンからへの切り戻し

```shell
PRIMARY_DB_CLUSTER_ARN=$(aws rds --profile ${PROFILE} --region ${SECONDARY_REGION} describe-global-clusters | \
    jq -r '
        .GlobalClusters[] | 
        select(.GlobalClusterIdentifier == "'${RDS_GLOBALDB_NAME}'").GlobalClusterMembers[] |
        select(.IsWriter == false).DBClusterArn'
)

#フェイルオーバー
aws rds --profile ${PROFILE} --region ${SECONDARY_REGION} --no-cli-pager \
   failover-global-cluster \
    --global-cluster-identifier "${RDS_GLOBALDB_NAME}" \
    --target-db-cluster-identifier "${PRIMARY_DB_CLUSTER_ARN}"
```


### (6)-(c)被災時のセカンダリリージョンのぃのクラスター昇格
こちらは別途検証。

ドキュメント[予期しない停止からの Amazon Aurora グローバルデータベースの復旧](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover)参照。

手順
- プライマリ Aurora DB クラスターの更新を完全に停止させる
- セカンダリリージョンのAuroraDBをグローバルデータベースからでタッチする(単独のDBクラスターに昇格する)
- 新しいエンドポイントを確認し、アプリケーションに設定する
