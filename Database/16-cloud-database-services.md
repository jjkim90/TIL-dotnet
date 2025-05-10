# 클라우드 데이터베이스 서비스

## 개요

클라우드 컴퓨팅의 발전과 함께 데이터베이스도 클라우드 환경으로 빠르게 이동하고 있습니다. 클라우드 데이터베이스 서비스는 확장성, 가용성, 관리 편의성을 제공하며, 온프레미스 환경의 한계를 극복합니다. 이 장에서는 주요 클라우드 제공업체의 데이터베이스 서비스와 구현 전략을 학습합니다.

## 1. AWS 데이터베이스 서비스

### 1.1 Amazon RDS (Relational Database Service)

```python
import boto3
import pymysql
from datetime import datetime, timedelta
import json

class AWSRDSManager:
    """AWS RDS 관리 클래스"""
    
    def __init__(self, region='us-east-1'):
        self.rds_client = boto3.client('rds', region_name=region)
        self.cloudwatch = boto3.client('cloudwatch', region_name=region)
        
    def create_database_instance(self, instance_config):
        """RDS 인스턴스 생성"""
        try:
            response = self.rds_client.create_db_instance(
                DBInstanceIdentifier=instance_config['identifier'],
                DBInstanceClass=instance_config['instance_class'],
                Engine=instance_config['engine'],
                MasterUsername=instance_config['master_username'],
                MasterUserPassword=instance_config['master_password'],
                AllocatedStorage=instance_config['storage_size'],
                StorageType='gp3',  # General Purpose SSD
                StorageEncrypted=True,
                MultiAZ=True,  # 고가용성
                BackupRetentionPeriod=7,
                PreferredBackupWindow='03:00-04:00',
                PreferredMaintenanceWindow='sun:04:00-sun:05:00',
                EnablePerformanceInsights=True,
                PerformanceInsightsRetentionPeriod=7,
                DeletionProtection=True,
                Tags=[
                    {'Key': 'Environment', 'Value': 'Production'},
                    {'Key': 'Application', 'Value': instance_config['app_name']}
                ]
            )
            return response['DBInstance']
        except Exception as e:
            print(f"RDS 인스턴스 생성 실패: {e}")
            raise
    
    def create_read_replica(self, source_instance_id, replica_config):
        """읽기 전용 복제본 생성"""
        response = self.rds_client.create_db_instance_read_replica(
            DBInstanceIdentifier=replica_config['identifier'],
            SourceDBInstanceIdentifier=source_instance_id,
            DBInstanceClass=replica_config['instance_class'],
            PubliclyAccessible=False,
            MultiAZ=False,  # 읽기 복제본은 일반적으로 단일 AZ
            StorageEncrypted=True,
            Tags=[
                {'Key': 'Type', 'Value': 'ReadReplica'},
                {'Key': 'Source', 'Value': source_instance_id}
            ]
        )
        return response['DBInstance']
    
    def setup_automated_backup(self, instance_id):
        """자동 백업 설정"""
        # 백업 정책 설정
        backup_policy = {
            'BackupRetentionPeriod': 30,  # 30일 보관
            'PreferredBackupWindow': '03:00-04:00',
            'BackupPolicy': 'AUTOMATED'
        }
        
        # 스냅샷 스케줄 생성
        self.rds_client.modify_db_instance(
            DBInstanceIdentifier=instance_id,
            BackupRetentionPeriod=backup_policy['BackupRetentionPeriod'],
            PreferredBackupWindow=backup_policy['PreferredBackupWindow'],
            ApplyImmediately=False
        )
        
        # 수동 스냅샷 생성
        snapshot_id = f"{instance_id}-manual-{datetime.now().strftime('%Y%m%d%H%M%S')}"
        self.rds_client.create_db_snapshot(
            DBSnapshotIdentifier=snapshot_id,
            DBInstanceIdentifier=instance_id,
            Tags=[
                {'Key': 'Type', 'Value': 'Manual'},
                {'Key': 'CreatedDate', 'Value': datetime.now().isoformat()}
            ]
        )
        
        return snapshot_id
    
    def monitor_performance(self, instance_id, metric_name, start_time, end_time):
        """성능 모니터링"""
        response = self.cloudwatch.get_metric_statistics(
            Namespace='AWS/RDS',
            MetricName=metric_name,
            Dimensions=[
                {
                    'Name': 'DBInstanceIdentifier',
                    'Value': instance_id
                }
            ],
            StartTime=start_time,
            EndTime=end_time,
            Period=300,  # 5분 간격
            Statistics=['Average', 'Maximum', 'Minimum']
        )
        
        return response['Datapoints']
    
    def implement_disaster_recovery(self, instance_id):
        """재해 복구 구현"""
        # 1. 다른 리전에 읽기 복제본 생성
        dr_config = {
            'source_region': 'us-east-1',
            'target_region': 'us-west-2',
            'kms_key_id': 'arn:aws:kms:us-west-2:123456789012:key/12345678'
        }
        
        # 크로스 리전 읽기 복제본
        dr_client = boto3.client('rds', region_name=dr_config['target_region'])
        
        response = dr_client.create_db_instance_read_replica(
            DBInstanceIdentifier=f"{instance_id}-dr",
            SourceDBInstanceIdentifier=f"arn:aws:rds:{dr_config['source_region']}:123456789012:db:{instance_id}",
            DBInstanceClass='db.t3.large',
            KmsKeyId=dr_config['kms_key_id'],
            StorageEncrypted=True
        )
        
        return response

# Aurora Serverless v2 구현
class AuroraServerlessManager:
    """Aurora Serverless v2 관리"""
    
    def __init__(self):
        self.rds_client = boto3.client('rds')
        
    def create_serverless_cluster(self, cluster_config):
        """Aurora Serverless 클러스터 생성"""
        response = self.rds_client.create_db_cluster(
            DBClusterIdentifier=cluster_config['cluster_id'],
            Engine='aurora-mysql',
            EngineVersion='8.0.mysql_aurora.3.02.0',
            EngineMode='provisioned',  # Serverless v2
            ServerlessV2ScalingConfiguration={
                'MinCapacity': 0.5,  # 최소 0.5 ACU
                'MaxCapacity': 128   # 최대 128 ACU
            },
            MasterUsername=cluster_config['master_username'],
            MasterUserPassword=cluster_config['master_password'],
            DatabaseName=cluster_config['database_name'],
            StorageEncrypted=True,
            EnableHttpEndpoint=True,  # Data API 활성화
            BackupRetentionPeriod=7,
            DeletionProtection=True
        )
        
        # 클러스터 인스턴스 추가
        instance_response = self.rds_client.create_db_instance(
            DBInstanceIdentifier=f"{cluster_config['cluster_id']}-instance-1",
            DBClusterIdentifier=cluster_config['cluster_id'],
            DBInstanceClass='db.serverless',
            Engine='aurora-mysql'
        )
        
        return response, instance_response
```

### 1.2 Amazon DynamoDB

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from decimal import Decimal
import json

class DynamoDBManager:
    """DynamoDB 관리 클래스"""
    
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.client = boto3.client('dynamodb')
        
    def create_global_table(self, table_name, regions):
        """글로벌 테이블 생성"""
        # 첫 번째 리전에 테이블 생성
        primary_table = self.dynamodb.create_table(
            TableName=table_name,
            KeySchema=[
                {'AttributeName': 'pk', 'KeyType': 'HASH'},
                {'AttributeName': 'sk', 'KeyType': 'RANGE'}
            ],
            AttributeDefinitions=[
                {'AttributeName': 'pk', 'AttributeType': 'S'},
                {'AttributeName': 'sk', 'AttributeType': 'S'},
                {'AttributeName': 'gsi1pk', 'AttributeType': 'S'},
                {'AttributeName': 'gsi1sk', 'AttributeType': 'S'}
            ],
            BillingMode='PAY_PER_REQUEST',
            StreamSpecification={
                'StreamEnabled': True,
                'StreamViewType': 'NEW_AND_OLD_IMAGES'
            },
            GlobalSecondaryIndexes=[
                {
                    'IndexName': 'GSI1',
                    'KeySchema': [
                        {'AttributeName': 'gsi1pk', 'KeyType': 'HASH'},
                        {'AttributeName': 'gsi1sk', 'KeyType': 'RANGE'}
                    ],
                    'Projection': {'ProjectionType': 'ALL'}
                }
            ],
            SSESpecification={
                'Enabled': True,
                'SSEType': 'KMS',
                'KMSMasterKeyId': 'alias/aws/dynamodb'
            }
        )
        
        # 테이블이 활성화될 때까지 대기
        primary_table.wait_until_exists()
        
        # 글로벌 테이블로 변환
        self.client.create_global_table(
            GlobalTableName=table_name,
            ReplicationGroup=[
                {'RegionName': region} for region in regions
            ]
        )
        
        return primary_table
    
    def implement_single_table_design(self):
        """단일 테이블 설계 패턴"""
        table = self.dynamodb.Table('ApplicationData')
        
        # 다양한 엔티티를 하나의 테이블에 저장
        items = [
            # 사용자 엔티티
            {
                'pk': 'USER#12345',
                'sk': 'PROFILE',
                'entity_type': 'User',
                'username': 'john_doe',
                'email': 'john@example.com',
                'created_at': '2024-01-15T10:00:00Z'
            },
            # 사용자의 주문
            {
                'pk': 'USER#12345',
                'sk': 'ORDER#2024-001',
                'entity_type': 'Order',
                'order_id': '2024-001',
                'total': Decimal('150.00'),
                'status': 'DELIVERED',
                'gsi1pk': 'ORDER#2024-001',
                'gsi1sk': 'USER#12345'
            },
            # 제품 엔티티
            {
                'pk': 'PRODUCT#ABC123',
                'sk': 'INFO',
                'entity_type': 'Product',
                'name': 'Sample Product',
                'price': Decimal('50.00'),
                'category': 'Electronics',
                'gsi1pk': 'CATEGORY#Electronics',
                'gsi1sk': 'PRODUCT#ABC123'
            }
        ]
        
        # 배치 쓰기
        with table.batch_writer() as batch:
            for item in items:
                batch.put_item(Item=item)
        
        # 액세스 패턴 예제
        # 1. 특정 사용자의 모든 주문 조회
        response = table.query(
            KeyConditionExpression=Key('pk').eq('USER#12345') & Key('sk').begins_with('ORDER#')
        )
        
        # 2. 특정 카테고리의 모든 제품 조회 (GSI 사용)
        response = table.query(
            IndexName='GSI1',
            KeyConditionExpression=Key('gsi1pk').eq('CATEGORY#Electronics')
        )
        
        return response
    
    def setup_auto_scaling(self, table_name):
        """오토 스케일링 설정"""
        autoscaling_client = boto3.client('application-autoscaling')
        
        # 읽기 용량 오토스케일링
        autoscaling_client.register_scalable_target(
            ServiceNamespace='dynamodb',
            ResourceId=f'table/{table_name}',
            ScalableDimension='dynamodb:table:ReadCapacityUnits',
            MinCapacity=5,
            MaxCapacity=40000
        )
        
        # 스케일링 정책
        autoscaling_client.put_scaling_policy(
            PolicyName=f'{table_name}-read-scaling-policy',
            ServiceNamespace='dynamodb',
            ResourceId=f'table/{table_name}',
            ScalableDimension='dynamodb:table:ReadCapacityUnits',
            PolicyType='TargetTrackingScaling',
            TargetTrackingScalingPolicyConfiguration={
                'TargetValue': 70.0,
                'PredefinedMetricSpecification': {
                    'PredefinedMetricType': 'DynamoDBReadCapacityUtilization'
                },
                'ScaleInCooldown': 300,
                'ScaleOutCooldown': 60
            }
        )
```

## 2. Azure 데이터베이스 서비스

### 2.1 Azure SQL Database

```python
import pyodbc
import azure.identity
from azure.mgmt.sql import SqlManagementClient
from azure.mgmt.monitor import MonitorManagementClient
from datetime import datetime, timedelta

class AzureSQLManager:
    """Azure SQL Database 관리 클래스"""
    
    def __init__(self, subscription_id):
        self.credential = azure.identity.DefaultAzureCredential()
        self.subscription_id = subscription_id
        self.sql_client = SqlManagementClient(self.credential, subscription_id)
        self.monitor_client = MonitorManagementClient(self.credential, subscription_id)
        
    def create_elastic_pool(self, resource_group, server_name, pool_config):
        """탄력적 풀 생성"""
        elastic_pool = self.sql_client.elastic_pools.begin_create_or_update(
            resource_group_name=resource_group,
            server_name=server_name,
            elastic_pool_name=pool_config['name'],
            parameters={
                'location': pool_config['location'],
                'sku': {
                    'name': 'GP_Gen5',
                    'tier': 'GeneralPurpose',
                    'capacity': pool_config['capacity']  # vCores
                },
                'per_database_settings': {
                    'min_capacity': 0.25,
                    'max_capacity': 4
                },
                'zone_redundant': True,
                'license_type': 'BasePrice',
                'tags': {
                    'Environment': 'Production',
                    'CostCenter': pool_config['cost_center']
                }
            }
        ).result()
        
        return elastic_pool
    
    def implement_geo_replication(self, primary_config, secondary_config):
        """지역 복제 구현"""
        # 보조 서버 생성
        secondary_server = self.sql_client.servers.begin_create_or_update(
            resource_group_name=secondary_config['resource_group'],
            server_name=secondary_config['server_name'],
            parameters={
                'location': secondary_config['location'],
                'administrator_login': secondary_config['admin_login'],
                'administrator_login_password': secondary_config['admin_password'],
                'version': '12.0'
            }
        ).result()
        
        # 지역 복제 데이터베이스 생성
        geo_replica = self.sql_client.databases.begin_create_or_update(
            resource_group_name=secondary_config['resource_group'],
            server_name=secondary_config['server_name'],
            database_name=primary_config['database_name'],
            parameters={
                'location': secondary_config['location'],
                'create_mode': 'Secondary',
                'source_database_id': f"/subscriptions/{self.subscription_id}/resourceGroups/{primary_config['resource_group']}/providers/Microsoft.Sql/servers/{primary_config['server_name']}/databases/{primary_config['database_name']}",
                'sku': {
                    'name': 'S3',
                    'tier': 'Standard'
                }
            }
        ).result()
        
        return geo_replica
    
    def setup_threat_detection(self, resource_group, server_name, database_name):
        """위협 탐지 설정"""
        threat_detection_policy = self.sql_client.database_threat_detection_policies.create_or_update(
            resource_group_name=resource_group,
            server_name=server_name,
            database_name=database_name,
            security_alert_policy_name='default',
            parameters={
                'state': 'Enabled',
                'disabled_alerts': [],
                'email_addresses': ['security@company.com'],
                'email_account_admins': True,
                'retention_days': 30,
                'storage_account_access_key': 'your-storage-key',
                'storage_endpoint': 'https://yourstorageaccount.blob.core.windows.net'
            }
        )
        
        return threat_detection_policy

# Azure Cosmos DB 구현
class CosmosDBManager:
    """Azure Cosmos DB 관리"""
    
    def __init__(self):
        from azure.cosmos import CosmosClient
        
        self.endpoint = "https://your-cosmos-account.documents.azure.com:443/"
        self.key = "your-primary-key"
        self.client = CosmosClient(self.endpoint, self.key)
        
    def create_multi_region_database(self, database_id, regions):
        """다중 지역 데이터베이스 생성"""
        database = self.client.create_database_if_not_exists(
            id=database_id,
            offer_throughput=4000  # RU/s
        )
        
        # 컨테이너 생성 (파티션 키 설정)
        container = database.create_container_if_not_exists(
            id='Items',
            partition_key='/category',
            default_ttl=86400,  # 1일
            indexing_policy={
                'automatic': True,
                'indexingMode': 'consistent',
                'includedPaths': [{'path': '/*'}],
                'excludedPaths': [{'path': '/_etag/?'}],
                'compositeIndexes': [
                    [
                        {'path': '/category', 'order': 'ascending'},
                        {'path': '/timestamp', 'order': 'descending'}
                    ]
                ]
            }
        )
        
        return database, container
    
    def implement_change_feed(self, container):
        """변경 피드 구현"""
        from azure.cosmos import PartitionKey
        import asyncio
        
        async def process_change_feed():
            # 변경 피드 프로세서
            change_feed_processor = container.query_items_change_feed(
                is_start_from_beginning=True,
                partition_key_range_id='0'
            )
            
            async for doc in change_feed_processor:
                # 변경 사항 처리
                print(f"변경 감지: {doc['id']}")
                
                # 실시간 분석 또는 이벤트 처리
                if doc.get('operation') == 'create':
                    await self.send_to_event_hub(doc)
                elif doc.get('operation') == 'update':
                    await self.update_cache(doc)
        
        return process_change_feed
```

## 3. Google Cloud 데이터베이스 서비스

### 3.1 Cloud SQL과 Cloud Spanner

```python
from google.cloud import sql_v1
from google.cloud import spanner
from google.oauth2 import service_account
import time

class GoogleCloudSQLManager:
    """Google Cloud SQL 관리"""
    
    def __init__(self, project_id):
        self.project_id = project_id
        self.sql_client = sql_v1.SqlInstancesServiceClient()
        
    def create_highly_available_instance(self, instance_config):
        """고가용성 Cloud SQL 인스턴스 생성"""
        instance = sql_v1.DatabaseInstance(
            name=instance_config['name'],
            database_version=instance_config['database_version'],
            settings=sql_v1.Settings(
                tier='db-n1-standard-4',
                pricing_plan='PER_USE',
                activation_policy='ALWAYS',
                data_disk_size_gb=100,
                data_disk_type='PD_SSD',
                backup_configuration=sql_v1.BackupConfiguration(
                    enabled=True,
                    start_time='03:00',
                    location='us-central1',
                    point_in_time_recovery_enabled=True,
                    transaction_log_retention_days=7
                ),
                ip_configuration=sql_v1.IpConfiguration(
                    ipv4_enabled=False,
                    private_network=f"projects/{self.project_id}/global/networks/default",
                    require_ssl=True
                ),
                database_flags=[
                    sql_v1.DatabaseFlag(name='max_connections', value='1000'),
                    sql_v1.DatabaseFlag(name='log_bin_trust_function_creators', value='on')
                ],
                availability_type='REGIONAL',  # 고가용성
                maintenance_window=sql_v1.MaintenanceWindow(
                    hour=4,
                    day=7,  # Sunday
                    update_track='stable'
                )
            )
        )
        
        operation = self.sql_client.insert(
            project=self.project_id,
            body=instance
        )
        
        return operation

class CloudSpannerManager:
    """Cloud Spanner 관리"""
    
    def __init__(self, project_id):
        self.project_id = project_id
        self.spanner_client = spanner.Client(project=project_id)
        
    def create_global_database(self, instance_id, database_id):
        """글로벌 분산 데이터베이스 생성"""
        # 멀티 리전 인스턴스 생성
        instance = self.spanner_client.instance(
            instance_id,
            configuration_name='nam-eur-asia3',  # 글로벌 구성
            display_name='Global Transaction Database',
            node_count=3
        )
        
        operation = instance.create()
        operation.result()  # 생성 대기
        
        # 데이터베이스 생성
        database = instance.database(
            database_id,
            ddl_statements=[
                """CREATE TABLE Users (
                    UserId INT64 NOT NULL,
                    Username STRING(100) NOT NULL,
                    Email STRING(255) NOT NULL,
                    CreatedAt TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
                    UpdatedAt TIMESTAMP OPTIONS (allow_commit_timestamp=true),
                ) PRIMARY KEY (UserId)""",
                
                """CREATE UNIQUE INDEX UsersByEmail ON Users(Email)""",
                
                """CREATE TABLE Orders (
                    OrderId INT64 NOT NULL,
                    UserId INT64 NOT NULL,
                    OrderDate TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
                    TotalAmount NUMERIC NOT NULL,
                    Status STRING(50) NOT NULL,
                    CONSTRAINT FK_User FOREIGN KEY (UserId) REFERENCES Users (UserId),
                ) PRIMARY KEY (OrderId),
                INTERLEAVE IN PARENT Users ON DELETE CASCADE"""
            ]
        )
        
        operation = database.create()
        operation.result()
        
        return database
    
    def implement_distributed_transaction(self, database):
        """분산 트랜잭션 구현"""
        def transfer_funds(transaction, from_account, to_account, amount):
            # 계좌 잔액 확인
            from_balance_query = f"""
                SELECT Balance FROM Accounts 
                WHERE AccountId = @from_account
            """
            
            from_result = list(transaction.execute_sql(
                from_balance_query,
                params={'from_account': from_account},
                param_types={'from_account': spanner.param_types.INT64}
            ))
            
            if not from_result or from_result[0][0] < amount:
                raise ValueError("잔액 부족")
            
            # 송금 실행
            transaction.execute_update(
                """UPDATE Accounts 
                   SET Balance = Balance - @amount,
                       UpdatedAt = PENDING_COMMIT_TIMESTAMP()
                   WHERE AccountId = @from_account""",
                params={'from_account': from_account, 'amount': amount},
                param_types={
                    'from_account': spanner.param_types.INT64,
                    'amount': spanner.param_types.NUMERIC
                }
            )
            
            transaction.execute_update(
                """UPDATE Accounts 
                   SET Balance = Balance + @amount,
                       UpdatedAt = PENDING_COMMIT_TIMESTAMP()
                   WHERE AccountId = @to_account""",
                params={'to_account': to_account, 'amount': amount},
                param_types={
                    'to_account': spanner.param_types.INT64,
                    'amount': spanner.param_types.NUMERIC
                }
            )
            
            # 트랜잭션 로그 기록
            transaction.insert(
                'TransactionLog',
                columns=['TransactionId', 'FromAccount', 'ToAccount', 
                        'Amount', 'Timestamp', 'Status'],
                values=[[
                    spanner.generate_uuid(),
                    from_account,
                    to_account,
                    amount,
                    spanner.COMMIT_TIMESTAMP,
                    'COMPLETED'
                ]]
            )
        
        # 트랜잭션 실행
        database.run_in_transaction(transfer_funds, 12345, 67890, 100.00)
```

## 4. 클라우드 데이터베이스 마이그레이션

### 4.1 마이그레이션 전략과 도구

```python
import subprocess
import threading
import queue
from concurrent.futures import ThreadPoolExecutor
import hashlib

class CloudMigrationOrchestrator:
    """클라우드 마이그레이션 오케스트레이터"""
    
    def __init__(self, source_config, target_config):
        self.source = source_config
        self.target = target_config
        self.migration_queue = queue.Queue()
        self.executor = ThreadPoolExecutor(max_workers=10)
        
    def execute_zero_downtime_migration(self):
        """무중단 마이그레이션 실행"""
        migration_steps = [
            self.setup_replication,
            self.sync_initial_data,
            self.enable_change_data_capture,
            self.validate_data_consistency,
            self.switch_application_traffic,
            self.cleanup_source
        ]
        
        for step in migration_steps:
            try:
                result = step()
                print(f"{step.__name__} 완료: {result}")
            except Exception as e:
                print(f"{step.__name__} 실패: {e}")
                self.rollback()
                raise
    
    def setup_replication(self):
        """복제 설정"""
        # AWS DMS (Database Migration Service) 사용
        dms_config = {
            'replication_instance': {
                'ReplicationInstanceIdentifier': 'migration-instance',
                'ReplicationInstanceClass': 'dms.c5.large',
                'MultiAZ': True,
                'PubliclyAccessible': False
            },
            'source_endpoint': {
                'EndpointIdentifier': 'source-endpoint',
                'EndpointType': 'source',
                'EngineName': self.source['engine'],
                'ServerName': self.source['host'],
                'Port': self.source['port'],
                'Username': self.source['username'],
                'Password': self.source['password']
            },
            'target_endpoint': {
                'EndpointIdentifier': 'target-endpoint',
                'EndpointType': 'target',
                'EngineName': self.target['engine'],
                'ServerName': self.target['host'],
                'Port': self.target['port'],
                'Username': self.target['username'],
                'Password': self.target['password']
            }
        }
        
        # 마이그레이션 태스크 생성
        migration_task = {
            'MigrationTaskIdentifier': 'full-load-and-cdc',
            'SourceEndpointArn': 'source-endpoint-arn',
            'TargetEndpointArn': 'target-endpoint-arn',
            'ReplicationInstanceArn': 'replication-instance-arn',
            'MigrationType': 'full-load-and-cdc',
            'TableMappings': json.dumps({
                'rules': [{
                    'rule-type': 'selection',
                    'rule-id': '1',
                    'rule-name': '1',
                    'object-locator': {
                        'schema-name': '%',
                        'table-name': '%'
                    },
                    'rule-action': 'include'
                }]
            })
        }
        
        return dms_config
    
    def validate_data_consistency(self):
        """데이터 일관성 검증"""
        validation_results = {}
        
        # 테이블별 체크섬 비교
        for table in self.get_tables_list():
            source_checksum = self.calculate_table_checksum(
                self.source, table
            )
            target_checksum = self.calculate_table_checksum(
                self.target, table
            )
            
            validation_results[table] = {
                'source_checksum': source_checksum,
                'target_checksum': target_checksum,
                'match': source_checksum == target_checksum
            }
        
        # 행 수 비교
        for table in self.get_tables_list():
            source_count = self.get_row_count(self.source, table)
            target_count = self.get_row_count(self.target, table)
            
            validation_results[table]['source_count'] = source_count
            validation_results[table]['target_count'] = target_count
            validation_results[table]['count_match'] = source_count == target_count
        
        return validation_results
    
    def calculate_table_checksum(self, connection, table):
        """테이블 체크섬 계산"""
        query = f"""
        SELECT MD5(
            STRING_AGG(
                MD5(CAST(t.* AS TEXT)), 
                '' ORDER BY t.{self.get_primary_key(table)}
            )
        ) AS checksum
        FROM {table} t
        """
        
        # 실제 구현에서는 각 데이터베이스 유형에 맞는 쿼리 사용
        return hashlib.md5(query.encode()).hexdigest()
```

## 5. 클라우드 네이티브 데이터베이스 패턴

### 5.1 서버리스 데이터베이스 아키텍처

```python
class ServerlessDatabaseArchitecture:
    """서버리스 데이터베이스 아키텍처"""
    
    def __init__(self):
        self.lambda_client = boto3.client('lambda')
        self.api_gateway = boto3.client('apigatewayv2')
        
    def create_data_api_layer(self):
        """데이터 API 레이어 생성"""
        # Lambda 함수로 데이터베이스 액세스 추상화
        lambda_code = '''
import json
import boto3
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit

logger = Logger()
tracer = Tracer()
metrics = Metrics()

@tracer.capture_lambda_handler
@logger.inject_lambda_context
@metrics.log_metrics
def lambda_handler(event, context):
    # RDS Data API 사용
    rds_data = boto3.client('rds-data')
    
    database_arn = 'arn:aws:rds:region:account:cluster:cluster-name'
    secret_arn = 'arn:aws:secretsmanager:region:account:secret:name'
    
    try:
        # SQL 실행
        response = rds_data.execute_statement(
            resourceArn=database_arn,
            secretArn=secret_arn,
            database='mydb',
            sql=event['sql'],
            parameters=event.get('parameters', [])
        )
        
        # 메트릭 기록
        metrics.add_metric(name="DatabaseQuery", unit=MetricUnit.Count, value=1)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'records': response['records'],
                'numberOfRecordsUpdated': response.get('numberOfRecordsUpdated', 0)
            })
        }
        
    except Exception as e:
        logger.error(f"Database error: {str(e)}")
        metrics.add_metric(name="DatabaseError", unit=MetricUnit.Count, value=1)
        
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
'''
        
        # Lambda 함수 배포
        lambda_function = self.lambda_client.create_function(
            FunctionName='database-api',
            Runtime='python3.9',
            Role='arn:aws:iam::account:role/lambda-role',
            Handler='index.lambda_handler',
            Code={'ZipFile': lambda_code.encode()},
            Timeout=30,
            MemorySize=512,
            Environment={
                'Variables': {
                    'POWERTOOLS_SERVICE_NAME': 'database-api',
                    'POWERTOOLS_METRICS_NAMESPACE': 'DatabaseAPI'
                }
            }
        )
        
        return lambda_function
    
    def implement_connection_pooling(self):
        """서버리스 환경의 연결 풀링"""
        # RDS Proxy를 통한 연결 관리
        rds_proxy_config = {
            'DBProxyName': 'serverless-db-proxy',
            'EngineFamily': 'MYSQL',
            'Auth': [{
                'AuthScheme': 'SECRETS',
                'SecretArn': 'arn:aws:secretsmanager:region:account:secret:name'
            }],
            'RoleArn': 'arn:aws:iam::account:role/proxy-role',
            'VpcSubnetIds': ['subnet-1', 'subnet-2'],
            'RequireTLS': True,
            'MaxConnectionsPercent': 100,
            'MaxIdleConnectionsPercent': 50,
            'ConnectionBorrowTimeout': 120,
            'Tags': [
                {'Key': 'Environment', 'Value': 'Production'}
            ]
        }
        
        return rds_proxy_config
```

## 마무리

클라우드 데이터베이스 서비스는 전통적인 온프레미스 데이터베이스 관리의 복잡성을 크게 줄여주며, 자동화된 백업, 고가용성, 글로벌 분산, 자동 스케일링 등의 기능을 제공합니다. 각 클라우드 제공업체는 고유한 장점과 특성을 가진 다양한 데이터베이스 서비스를 제공하므로, 애플리케이션의 요구사항에 맞는 적절한 서비스를 선택하는 것이 중요합니다.

다음 장에서는 실시간 데이터 처리와 스트리밍 데이터베이스에 대해 학습하겠습니다.