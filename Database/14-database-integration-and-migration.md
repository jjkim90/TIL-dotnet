# 데이터베이스 통합과 마이그레이션

## 개요

현대의 엔터프라이즈 환경에서는 다양한 데이터베이스 시스템이 공존하며, 이들 간의 통합과 데이터 마이그레이션은 필수적인 과제입니다. 이 장에서는 이기종 데이터베이스 간의 통합 전략, 데이터 마이그레이션 방법론, ETL/ELT 프로세스, 그리고 실시간 데이터 동기화 기술을 학습합니다.

## 1. 데이터베이스 통합 아키텍처

### 1.1 통합 패턴과 전략

```python
import asyncio
import aioredis
import asyncpg
import motor.motor_asyncio
from kafka import KafkaProducer, KafkaConsumer
import json
from datetime import datetime

class DatabaseIntegrationHub:
    """다중 데이터베이스 통합 허브"""
    
    def __init__(self):
        self.connections = {}
        self.kafka_producer = None
        
    async def initialize(self):
        """모든 데이터베이스 연결 초기화"""
        # PostgreSQL
        self.connections['postgres'] = await asyncpg.create_pool(
            'postgresql://user:password@localhost/maindb',
            min_size=10,
            max_size=20
        )
        
        # MongoDB
        self.connections['mongodb'] = motor.motor_asyncio.AsyncIOMotorClient(
            'mongodb://localhost:27017'
        )
        
        # Redis
        self.connections['redis'] = await aioredis.create_redis_pool(
            'redis://localhost',
            minsize=5,
            maxsize=10
        )
        
        # Kafka
        self.kafka_producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
    
    async def execute_cross_database_transaction(self, operations):
        """분산 트랜잭션 실행 (Saga 패턴)"""
        completed_operations = []
        
        try:
            # 각 데이터베이스에서 작업 수행
            for operation in operations:
                result = await self._execute_operation(operation)
                completed_operations.append({
                    'operation': operation,
                    'result': result
                })
                
                # 이벤트 발행
                self.kafka_producer.send(
                    'database-events',
                    {
                        'event': 'operation_completed',
                        'database': operation['database'],
                        'operation': operation['type'],
                        'timestamp': datetime.utcnow().isoformat()
                    }
                )
            
            return {'status': 'success', 'operations': completed_operations}
            
        except Exception as e:
            # 보상 트랜잭션 실행
            await self._compensate_operations(completed_operations)
            return {'status': 'failed', 'error': str(e)}
    
    async def _execute_operation(self, operation):
        """개별 데이터베이스 작업 실행"""
        db_type = operation['database']
        
        if db_type == 'postgres':
            return await self._execute_postgres(operation)
        elif db_type == 'mongodb':
            return await self._execute_mongodb(operation)
        elif db_type == 'redis':
            return await self._execute_redis(operation)
    
    async def _execute_postgres(self, operation):
        """PostgreSQL 작업 실행"""
        async with self.connections['postgres'].acquire() as conn:
            if operation['type'] == 'insert':
                return await conn.execute(
                    operation['query'],
                    *operation.get('params', [])
                )
            elif operation['type'] == 'select':
                return await conn.fetch(
                    operation['query'],
                    *operation.get('params', [])
                )
    
    async def _execute_mongodb(self, operation):
        """MongoDB 작업 실행"""
        db = self.connections['mongodb'][operation['database_name']]
        collection = db[operation['collection']]
        
        if operation['type'] == 'insert':
            return await collection.insert_one(operation['document'])
        elif operation['type'] == 'find':
            cursor = collection.find(operation.get('filter', {}))
            return await cursor.to_list(length=operation.get('limit', 100))
    
    async def _compensate_operations(self, completed_operations):
        """실패 시 보상 트랜잭션"""
        for op_record in reversed(completed_operations):
            operation = op_record['operation']
            
            # 각 작업에 대한 역작업 수행
            if operation['type'] == 'insert':
                await self._execute_operation({
                    'database': operation['database'],
                    'type': 'delete',
                    'compensate_for': operation
                })

# 데이터 가상화 레이어
class DataVirtualizationLayer:
    """여러 데이터베이스를 단일 인터페이스로 제공"""
    
    def __init__(self, integration_hub):
        self.hub = integration_hub
        self.schema_mapping = self._load_schema_mapping()
    
    def _load_schema_mapping(self):
        """스키마 매핑 정의"""
        return {
            'customer': {
                'postgres': {
                    'table': 'customers',
                    'fields': {
                        'id': 'customer_id',
                        'name': 'full_name',
                        'email': 'email_address'
                    }
                },
                'mongodb': {
                    'collection': 'users',
                    'fields': {
                        'id': '_id',
                        'name': 'profile.name',
                        'email': 'contact.email'
                    }
                }
            },
            'order': {
                'postgres': {
                    'table': 'orders',
                    'fields': {
                        'id': 'order_id',
                        'customer_id': 'customer_id',
                        'total': 'total_amount',
                        'status': 'order_status'
                    }
                },
                'mongodb': {
                    'collection': 'transactions',
                    'fields': {
                        'id': '_id',
                        'customer_id': 'userId',
                        'total': 'amount.total',
                        'status': 'state'
                    }
                }
            }
        }
    
    async def query(self, entity_type, filters=None, fields=None):
        """통합 쿼리 인터페이스"""
        results = []
        
        # 각 데이터베이스에서 데이터 조회
        for db_type, mapping in self.schema_mapping[entity_type].items():
            if db_type == 'postgres':
                data = await self._query_postgres(entity_type, filters, fields)
            elif db_type == 'mongodb':
                data = await self._query_mongodb(entity_type, filters, fields)
            
            # 결과 정규화
            normalized_data = self._normalize_results(data, db_type, entity_type)
            results.extend(normalized_data)
        
        # 중복 제거 및 병합
        return self._merge_results(results)
    
    async def _query_postgres(self, entity_type, filters, fields):
        """PostgreSQL 쿼리 생성 및 실행"""
        mapping = self.schema_mapping[entity_type]['postgres']
        
        # SELECT 절 구성
        select_fields = []
        if fields:
            for field in fields:
                pg_field = mapping['fields'].get(field, field)
                select_fields.append(f"{pg_field} AS {field}")
        else:
            select_fields = ['*']
        
        # WHERE 절 구성
        where_clauses = []
        params = []
        if filters:
            for field, value in filters.items():
                pg_field = mapping['fields'].get(field, field)
                where_clauses.append(f"{pg_field} = ${len(params) + 1}")
                params.append(value)
        
        query = f"""
            SELECT {', '.join(select_fields)}
            FROM {mapping['table']}
            {' WHERE ' + ' AND '.join(where_clauses) if where_clauses else ''}
        """
        
        return await self.hub._execute_postgres({
            'type': 'select',
            'query': query,
            'params': params
        })
```

### 1.2 CDC (Change Data Capture) 구현

```python
import psycopg2
from psycopg2.extras import LogicalReplicationConnection
import pymongo
from confluent_kafka import Producer, Consumer
import json
import threading

class ChangeDataCapture:
    """변경 데이터 캡처 시스템"""
    
    def __init__(self):
        self.replication_slots = {}
        self.kafka_producer = Producer({
            'bootstrap.servers': 'localhost:9092',
            'client.id': 'cdc-producer'
        })
        self.running = True
    
    def setup_postgres_cdc(self, connection_params, slot_name='cdc_slot'):
        """PostgreSQL 논리적 복제 설정"""
        # 복제 슬롯 생성
        conn = psycopg2.connect(
            connection_factory=LogicalReplicationConnection,
            **connection_params
        )
        cur = conn.cursor()
        
        try:
            cur.execute(f"""
                CREATE_REPLICATION_SLOT {slot_name} 
                LOGICAL pgoutput
            """)
        except psycopg2.ProgrammingError:
            # 슬롯이 이미 존재
            pass
        
        # 복제 시작
        cur.start_replication(
            slot_name=slot_name,
            decode=True,
            options={'publication_names': 'cdc_publication'}
        )
        
        self.replication_slots['postgres'] = {
            'connection': conn,
            'cursor': cur,
            'slot_name': slot_name
        }
    
    def capture_postgres_changes(self):
        """PostgreSQL 변경사항 캡처"""
        cur = self.replication_slots['postgres']['cursor']
        
        while self.running:
            msg = cur.read_message()
            if msg:
                change_event = self._parse_postgres_message(msg)
                
                # Kafka로 전송
                self.kafka_producer.produce(
                    'postgres-cdc',
                    key=change_event['key'].encode('utf-8'),
                    value=json.dumps(change_event).encode('utf-8')
                )
                
                # 확인 응답
                cur.send_feedback(flush_lsn=msg.data_start)
    
    def _parse_postgres_message(self, msg):
        """PostgreSQL 복제 메시지 파싱"""
        payload = json.loads(msg.payload)
        
        return {
            'timestamp': msg.send_time.isoformat(),
            'operation': payload.get('action', 'unknown'),
            'schema': payload.get('schema'),
            'table': payload.get('table'),
            'key': f"{payload.get('schema')}.{payload.get('table')}.{payload.get('id')}",
            'before': payload.get('oldkeys'),
            'after': payload.get('columns'),
            'source': {
                'database': 'postgres',
                'lsn': str(msg.data_start)
            }
        }
    
    def setup_mongodb_cdc(self, connection_string):
        """MongoDB Change Streams 설정"""
        client = pymongo.MongoClient(connection_string)
        db = client.get_database('mydb')
        
        # Change Stream 시작
        pipeline = [
            {'$match': {
                'operationType': {'$in': ['insert', 'update', 'delete', 'replace']}
            }}
        ]
        
        change_stream = db.watch(pipeline=pipeline)
        
        thread = threading.Thread(
            target=self._monitor_mongodb_changes,
            args=(change_stream,)
        )
        thread.start()
    
    def _monitor_mongodb_changes(self, change_stream):
        """MongoDB 변경사항 모니터링"""
        for change in change_stream:
            change_event = {
                'timestamp': change['clusterTime'].as_datetime().isoformat(),
                'operation': change['operationType'],
                'database': change['ns']['db'],
                'collection': change['ns']['coll'],
                'key': str(change['documentKey']['_id']),
                'fullDocument': change.get('fullDocument'),
                'updateDescription': change.get('updateDescription'),
                'source': {
                    'database': 'mongodb',
                    'resumeToken': change['_id']
                }
            }
            
            # Kafka로 전송
            self.kafka_producer.produce(
                'mongodb-cdc',
                key=change_event['key'].encode('utf-8'),
                value=json.dumps(change_event, default=str).encode('utf-8')
            )

# CDC 이벤트 처리기
class CDCEventProcessor:
    """CDC 이벤트를 처리하고 다른 시스템에 적용"""
    
    def __init__(self):
        self.consumer = Consumer({
            'bootstrap.servers': 'localhost:9092',
            'group.id': 'cdc-processor',
            'auto.offset.reset': 'earliest'
        })
        self.consumer.subscribe(['postgres-cdc', 'mongodb-cdc'])
        self.handlers = self._setup_handlers()
    
    def _setup_handlers(self):
        """이벤트 핸들러 설정"""
        return {
            'customer_updated': self.handle_customer_update,
            'order_created': self.handle_order_creation,
            'inventory_changed': self.handle_inventory_change
        }
    
    async def process_events(self):
        """CDC 이벤트 처리"""
        while True:
            msg = self.consumer.poll(1.0)
            
            if msg is None:
                continue
            if msg.error():
                print(f"Consumer error: {msg.error()}")
                continue
            
            # 이벤트 파싱
            event = json.loads(msg.value().decode('utf-8'))
            
            # 이벤트 유형 결정
            event_type = self._determine_event_type(event)
            
            # 적절한 핸들러 호출
            if event_type in self.handlers:
                await self.handlers[event_type](event)
    
    async def handle_customer_update(self, event):
        """고객 정보 업데이트 처리"""
        if event['source']['database'] == 'postgres':
            # MongoDB에 동기화
            customer_data = event['after']
            
            await self.sync_to_mongodb({
                'collection': 'users',
                'filter': {'legacy_id': customer_data['customer_id']},
                'update': {
                    '$set': {
                        'profile.name': customer_data['full_name'],
                        'contact.email': customer_data['email_address'],
                        'sync.last_updated': datetime.utcnow()
                    }
                }
            })
            
            # 캐시 무효화
            await self.invalidate_cache(f"customer:{customer_data['customer_id']}")
```

## 2. ETL/ELT 프로세스

### 2.1 ETL 파이프라인 구현

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
import pandas as pd
from sqlalchemy import create_engine
import boto3

class ETLPipeline:
    """확장 가능한 ETL 파이프라인"""
    
    def __init__(self, source_config, target_config):
        self.source_config = source_config
        self.target_config = target_config
        self.transformations = []
    
    def add_transformation(self, transform_fn):
        """변환 함수 추가"""
        self.transformations.append(transform_fn)
        return self
    
    def run(self):
        """파이프라인 실행"""
        # Apache Beam 파이프라인 설정
        pipeline_options = PipelineOptions([
            '--runner=DirectRunner',
            '--project=my-project',
            '--temp_location=gs://my-bucket/temp'
        ])
        
        with beam.Pipeline(options=pipeline_options) as p:
            # Extract
            data = p | 'Extract' >> self._create_extract_step()
            
            # Transform
            for i, transform in enumerate(self.transformations):
                data = data | f'Transform{i}' >> beam.Map(transform)
            
            # Load
            data | 'Load' >> self._create_load_step()
    
    def _create_extract_step(self):
        """추출 단계 생성"""
        source_type = self.source_config['type']
        
        if source_type == 'postgres':
            return beam.io.jdbc.ReadFromJdbc(
                table_name=self.source_config['table'],
                driver_class_name='org.postgresql.Driver',
                jdbc_url=self.source_config['connection_string']
            )
        elif source_type == 'csv':
            return beam.io.ReadFromText(
                self.source_config['file_path'],
                skip_header_lines=1
            ) | beam.Map(self._parse_csv_line)
        elif source_type == 's3':
            return beam.Create(self._read_from_s3())
    
    def _create_load_step(self):
        """적재 단계 생성"""
        target_type = self.target_config['type']
        
        if target_type == 'bigquery':
            return beam.io.WriteToBigQuery(
                table=self.target_config['table'],
                dataset=self.target_config['dataset'],
                project=self.target_config['project'],
                schema=self.target_config['schema'],
                write_disposition=beam.io.BigQueryDisposition.WRITE_TRUNCATE
            )
        elif target_type == 'postgres':
            return beam.ParDo(PostgresWriter(self.target_config))

# 변환 함수 예제
class DataTransformations:
    """일반적인 데이터 변환 함수들"""
    
    @staticmethod
    def clean_phone_number(record):
        """전화번호 정규화"""
        if 'phone' in record:
            # 숫자만 추출
            cleaned = ''.join(filter(str.isdigit, record['phone']))
            # 국가 코드 추가
            if len(cleaned) == 10:
                cleaned = '1' + cleaned
            record['phone'] = cleaned
        return record
    
    @staticmethod
    def enrich_with_geolocation(record):
        """주소 기반 지리 정보 추가"""
        if 'address' in record:
            # 지오코딩 API 호출 (예시)
            lat, lng = geocode_address(record['address'])
            record['latitude'] = lat
            record['longitude'] = lng
        return record
    
    @staticmethod
    def aggregate_sales_data(records):
        """판매 데이터 집계"""
        df = pd.DataFrame(records)
        
        # 일별 집계
        daily_sales = df.groupby(['date', 'product_id']).agg({
            'quantity': 'sum',
            'revenue': 'sum',
            'transactions': 'count'
        }).reset_index()
        
        return daily_sales.to_dict('records')

# 증분 적재 관리
class IncrementalETL:
    """증분 ETL 처리"""
    
    def __init__(self, checkpoint_manager):
        self.checkpoint_manager = checkpoint_manager
    
    def extract_incremental(self, source_config):
        """마지막 체크포인트 이후 데이터만 추출"""
        last_checkpoint = self.checkpoint_manager.get_checkpoint(
            source_config['name']
        )
        
        if source_config['type'] == 'postgres':
            query = f"""
                SELECT * FROM {source_config['table']}
                WHERE updated_at > '{last_checkpoint}'
                ORDER BY updated_at
            """
            
            engine = create_engine(source_config['connection_string'])
            df = pd.read_sql(query, engine)
            
            # 새 체크포인트 저장
            if not df.empty:
                new_checkpoint = df['updated_at'].max()
                self.checkpoint_manager.save_checkpoint(
                    source_config['name'],
                    new_checkpoint
                )
            
            return df
    
    def handle_late_arriving_data(self, data, watermark_column):
        """지연 도착 데이터 처리"""
        current_time = datetime.utcnow()
        watermark = current_time - timedelta(hours=1)  # 1시간 워터마크
        
        # 정시 데이터와 지연 데이터 분리
        on_time_data = data[data[watermark_column] >= watermark]
        late_data = data[data[watermark_column] < watermark]
        
        # 지연 데이터는 별도 처리
        if not late_data.empty:
            self._process_late_data(late_data)
        
        return on_time_data
```

### 2.2 실시간 스트리밍 ETL

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import pyspark.sql.functions as F

class StreamingETL:
    """실시간 스트리밍 ETL"""
    
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("StreamingETL") \
            .config("spark.sql.adaptive.enabled", "true") \
            .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
            .getOrCreate()
    
    def create_streaming_pipeline(self):
        """스트리밍 파이프라인 생성"""
        # Kafka 소스
        kafka_stream = self.spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "localhost:9092") \
            .option("subscribe", "raw-events") \
            .option("startingOffsets", "latest") \
            .load()
        
        # 스키마 정의
        event_schema = StructType([
            StructField("event_id", StringType(), True),
            StructField("user_id", StringType(), True),
            StructField("event_type", StringType(), True),
            StructField("timestamp", TimestampType(), True),
            StructField("properties", MapType(StringType(), StringType()), True)
        ])
        
        # 데이터 파싱
        parsed_stream = kafka_stream \
            .select(from_json(col("value").cast("string"), event_schema).alias("data")) \
            .select("data.*")
        
        # 변환 적용
        transformed_stream = self.apply_transformations(parsed_stream)
        
        # 윈도우 집계
        windowed_stream = self.apply_windowing(transformed_stream)
        
        # 여러 싱크로 출력
        self.write_to_multiple_sinks(windowed_stream)
    
    def apply_transformations(self, stream):
        """스트림 변환"""
        return stream \
            .withColumn("event_date", to_date("timestamp")) \
            .withColumn("event_hour", hour("timestamp")) \
            .withColumn("is_mobile", 
                when(col("properties.device_type").isin(["ios", "android"]), True)
                .otherwise(False)
            ) \
            .filter(col("event_type").isNotNull())
    
    def apply_windowing(self, stream):
        """윈도우 기반 집계"""
        return stream \
            .withWatermark("timestamp", "10 minutes") \
            .groupBy(
                window("timestamp", "5 minutes", "1 minute"),
                "event_type",
                "is_mobile"
            ) \
            .agg(
                count("*").alias("event_count"),
                countDistinct("user_id").alias("unique_users"),
                avg("properties.duration").alias("avg_duration")
            )
    
    def write_to_multiple_sinks(self, stream):
        """여러 대상에 동시 쓰기"""
        # PostgreSQL로 집계 결과 저장
        stream.writeStream \
            .outputMode("append") \
            .foreachBatch(self._write_to_postgres) \
            .trigger(processingTime='30 seconds') \
            .start()
        
        # Elasticsearch로 원시 이벤트 저장
        stream.writeStream \
            .outputMode("append") \
            .format("org.elasticsearch.spark.sql") \
            .option("es.resource", "events/event") \
            .option("es.nodes", "localhost:9200") \
            .start()
        
        # Parquet 파일로 아카이브
        stream.writeStream \
            .outputMode("append") \
            .format("parquet") \
            .option("path", "s3://my-bucket/events/archive") \
            .option("checkpointLocation", "s3://my-bucket/checkpoints") \
            .partitionBy("event_date", "event_hour") \
            .trigger(processingTime='1 hour') \
            .start()
    
    def _write_to_postgres(self, batch_df, batch_id):
        """PostgreSQL에 배치 쓰기"""
        batch_df.write \
            .format("jdbc") \
            .option("url", "jdbc:postgresql://localhost:5432/analytics") \
            .option("dbtable", "event_aggregates") \
            .option("user", "user") \
            .option("password", "password") \
            .mode("append") \
            .save()

# 데이터 품질 검증
class DataQualityValidator:
    """ETL 과정의 데이터 품질 검증"""
    
    def __init__(self):
        self.rules = []
        self.metrics = {}
    
    def add_rule(self, rule):
        """검증 규칙 추가"""
        self.rules.append(rule)
        return self
    
    def validate(self, data):
        """데이터 검증 실행"""
        results = {
            'passed': True,
            'violations': [],
            'metrics': {}
        }
        
        for rule in self.rules:
            validation_result = rule.validate(data)
            
            if not validation_result['passed']:
                results['passed'] = False
                results['violations'].append({
                    'rule': rule.name,
                    'description': rule.description,
                    'violations': validation_result['violations']
                })
            
            results['metrics'][rule.name] = validation_result['metrics']
        
        return results

class NotNullRule:
    """NULL 값 검증 규칙"""
    
    def __init__(self, columns):
        self.name = "not_null"
        self.columns = columns
        self.description = f"Columns {columns} should not contain NULL values"
    
    def validate(self, df):
        violations = []
        metrics = {}
        
        for column in self.columns:
            null_count = df[column].isnull().sum()
            metrics[f"{column}_null_count"] = int(null_count)
            
            if null_count > 0:
                violations.append({
                    'column': column,
                    'null_count': null_count,
                    'null_percentage': null_count / len(df) * 100
                })
        
        return {
            'passed': len(violations) == 0,
            'violations': violations,
            'metrics': metrics
        }
```

## 3. 데이터 마이그레이션

### 3.1 마이그레이션 전략

```python
import sqlalchemy
from sqlalchemy import create_engine, MetaData, Table
import pandas as pd
from concurrent.futures import ThreadPoolExecutor, as_completed
import hashlib

class DatabaseMigration:
    """데이터베이스 마이그레이션 관리"""
    
    def __init__(self, source_engine, target_engine):
        self.source_engine = source_engine
        self.target_engine = target_engine
        self.metadata = MetaData()
        self.batch_size = 10000
        self.parallel_workers = 4
    
    def migrate_schema(self, tables=None):
        """스키마 마이그레이션"""
        # 소스 스키마 읽기
        self.metadata.reflect(bind=self.source_engine)
        
        tables_to_migrate = tables or self.metadata.tables.keys()
        
        for table_name in tables_to_migrate:
            table = self.metadata.tables[table_name]
            
            # 타겟에 테이블 생성
            print(f"Creating table: {table_name}")
            table.create(self.target_engine, checkfirst=True)
            
            # 인덱스 생성
            for index in table.indexes:
                print(f"Creating index: {index.name}")
                index.create(self.target_engine, checkfirst=True)
    
    def migrate_data(self, table_name, where_clause=None):
        """데이터 마이그레이션"""
        # 전체 레코드 수 확인
        count_query = f"SELECT COUNT(*) FROM {table_name}"
        if where_clause:
            count_query += f" WHERE {where_clause}"
        
        total_rows = self.source_engine.execute(count_query).scalar()
        print(f"Total rows to migrate: {total_rows}")
        
        # 병렬 마이그레이션
        with ThreadPoolExecutor(max_workers=self.parallel_workers) as executor:
            futures = []
            
            for offset in range(0, total_rows, self.batch_size):
                future = executor.submit(
                    self._migrate_batch,
                    table_name,
                    offset,
                    self.batch_size,
                    where_clause
                )
                futures.append(future)
            
            # 진행 상황 모니터링
            completed = 0
            for future in as_completed(futures):
                completed += self.batch_size
                progress = min(completed / total_rows * 100, 100)
                print(f"Progress: {progress:.2f}%")
    
    def _migrate_batch(self, table_name, offset, limit, where_clause):
        """배치 단위 마이그레이션"""
        # 데이터 읽기
        query = f"SELECT * FROM {table_name}"
        if where_clause:
            query += f" WHERE {where_clause}"
        query += f" LIMIT {limit} OFFSET {offset}"
        
        df = pd.read_sql(query, self.source_engine)
        
        # 데이터 변환 (필요시)
        df = self._transform_data(df, table_name)
        
        # 타겟에 쓰기
        df.to_sql(
            table_name,
            self.target_engine,
            if_exists='append',
            index=False,
            method='multi'
        )
    
    def _transform_data(self, df, table_name):
        """데이터 변환 로직"""
        # 테이블별 변환 규칙 적용
        transformations = {
            'customers': self._transform_customers,
            'orders': self._transform_orders,
            'products': self._transform_products
        }
        
        if table_name in transformations:
            return transformations[table_name](df)
        
        return df
    
    def _transform_customers(self, df):
        """고객 데이터 변환"""
        # 전화번호 형식 통일
        df['phone'] = df['phone'].str.replace('[^0-9]', '', regex=True)
        
        # 이메일 소문자 변환
        df['email'] = df['email'].str.lower()
        
        # 생년월일 형식 변환
        df['birth_date'] = pd.to_datetime(df['birth_date'])
        
        return df
    
    def validate_migration(self, table_name):
        """마이그레이션 검증"""
        validation_results = {
            'table': table_name,
            'source_count': 0,
            'target_count': 0,
            'checksum_match': False,
            'sample_comparison': []
        }
        
        # 레코드 수 비교
        source_count = self.source_engine.execute(
            f"SELECT COUNT(*) FROM {table_name}"
        ).scalar()
        target_count = self.target_engine.execute(
            f"SELECT COUNT(*) FROM {table_name}"
        ).scalar()
        
        validation_results['source_count'] = source_count
        validation_results['target_count'] = target_count
        
        # 체크섬 비교
        source_checksum = self._calculate_table_checksum(
            self.source_engine, table_name
        )
        target_checksum = self._calculate_table_checksum(
            self.target_engine, table_name
        )
        
        validation_results['checksum_match'] = (source_checksum == target_checksum)
        
        # 샘플 데이터 비교
        sample_size = min(100, source_count)
        source_sample = pd.read_sql(
            f"SELECT * FROM {table_name} ORDER BY RANDOM() LIMIT {sample_size}",
            self.source_engine
        )
        
        # 동일한 레코드를 타겟에서 조회하여 비교
        # ... (상세 구현)
        
        return validation_results
    
    def _calculate_table_checksum(self, engine, table_name):
        """테이블 체크섬 계산"""
        # 모든 데이터를 문자열로 연결하여 해시 계산
        df = pd.read_sql(f"SELECT * FROM {table_name} ORDER BY 1", engine)
        data_string = df.to_csv(index=False, header=False)
        
        return hashlib.sha256(data_string.encode()).hexdigest()

# 제로 다운타임 마이그레이션
class ZeroDowntimeMigration:
    """무중단 마이그레이션"""
    
    def __init__(self, source_config, target_config):
        self.source_config = source_config
        self.target_config = target_config
        self.cdc = ChangeDataCapture()
    
    async def execute(self):
        """무중단 마이그레이션 실행"""
        # 1. 초기 데이터 복사
        print("Phase 1: Initial data copy")
        await self._initial_copy()
        
        # 2. CDC 시작
        print("Phase 2: Start CDC")
        self.cdc.setup_postgres_cdc(self.source_config)
        cdc_thread = threading.Thread(target=self.cdc.capture_postgres_changes)
        cdc_thread.start()
        
        # 3. 증분 동기화
        print("Phase 3: Incremental sync")
        await self._incremental_sync()
        
        # 4. 최종 동기화 및 검증
        print("Phase 4: Final sync and validation")
        await self._final_sync()
        
        # 5. 컷오버
        print("Phase 5: Cutover")
        await self._cutover()
    
    async def _initial_copy(self):
        """초기 데이터 복사"""
        migration = DatabaseMigration(
            create_engine(self.source_config['connection_string']),
            create_engine(self.target_config['connection_string'])
        )
        
        # 스키마 마이그레이션
        migration.migrate_schema()
        
        # 데이터 마이그레이션 (과거 데이터만)
        cutoff_time = datetime.utcnow()
        for table in migration.metadata.tables.keys():
            migration.migrate_data(
                table,
                where_clause=f"updated_at < '{cutoff_time}'"
            )
    
    async def _incremental_sync(self):
        """증분 동기화"""
        lag_threshold = timedelta(seconds=5)
        
        while True:
            # 복제 지연 확인
            lag = await self._get_replication_lag()
            
            if lag < lag_threshold:
                break
            
            await asyncio.sleep(1)
    
    async def _cutover(self):
        """애플리케이션 전환"""
        # 1. 소스 데이터베이스 읽기 전용 모드
        await self._set_readonly(self.source_config)
        
        # 2. 최종 동기화 대기
        await self._wait_for_sync_completion()
        
        # 3. 애플리케이션 설정 변경
        await self._update_application_config()
        
        # 4. 연결 확인
        await self._verify_connections()
        
        # 5. 읽기 전용 모드 해제
        await self._unset_readonly(self.target_config)
```

## 4. 데이터 통합 모니터링

### 4.1 통합 모니터링 시스템

```python
import prometheus_client
from prometheus_client import Counter, Histogram, Gauge
import logging
from datadog import initialize, statsd
import time

class IntegrationMonitor:
    """데이터 통합 모니터링"""
    
    def __init__(self):
        # Prometheus 메트릭
        self.sync_operations = Counter(
            'data_sync_operations_total',
            'Total number of sync operations',
            ['source', 'target', 'status']
        )
        
        self.sync_duration = Histogram(
            'data_sync_duration_seconds',
            'Duration of sync operations',
            ['source', 'target']
        )
        
        self.replication_lag = Gauge(
            'replication_lag_seconds',
            'Current replication lag',
            ['source', 'target']
        )
        
        self.data_quality_score = Gauge(
            'data_quality_score',
            'Data quality score (0-100)',
            ['database', 'table']
        )
        
        # Datadog 초기화
        initialize(
            statsd_host='localhost',
            statsd_port=8125
        )
        
        # 로깅 설정
        self.logger = logging.getLogger(__name__)
    
    def record_sync_operation(self, source, target, duration, status='success'):
        """동기화 작업 기록"""
        # Prometheus
        self.sync_operations.labels(
            source=source,
            target=target,
            status=status
        ).inc()
        
        self.sync_duration.labels(
            source=source,
            target=target
        ).observe(duration)
        
        # Datadog
        statsd.increment(
            'data.sync.operations',
            tags=[f'source:{source}', f'target:{target}', f'status:{status}']
        )
        
        statsd.histogram(
            'data.sync.duration',
            duration,
            tags=[f'source:{source}', f'target:{target}']
        )
        
        # 로깅
        self.logger.info(
            f"Sync operation completed: {source} -> {target}, "
            f"duration: {duration:.2f}s, status: {status}"
        )
    
    def update_replication_lag(self, source, target, lag_seconds):
        """복제 지연 업데이트"""
        self.replication_lag.labels(
            source=source,
            target=target
        ).set(lag_seconds)
        
        # 임계값 초과 시 알림
        if lag_seconds > 60:  # 1분 이상
            self.logger.warning(
                f"High replication lag detected: {source} -> {target}, "
                f"lag: {lag_seconds}s"
            )
            
            # Datadog 이벤트 생성
            statsd.event(
                'High Replication Lag',
                f'Replication lag between {source} and {target} '
                f'is {lag_seconds} seconds',
                alert_type='warning',
                tags=[f'source:{source}', f'target:{target}']
            )
    
    def check_data_consistency(self, comparisons):
        """데이터 일관성 확인"""
        for comparison in comparisons:
            source_count = comparison['source_count']
            target_count = comparison['target_count']
            table = comparison['table']
            
            if source_count != target_count:
                discrepancy = abs(source_count - target_count)
                discrepancy_pct = discrepancy / source_count * 100
                
                self.logger.error(
                    f"Data inconsistency detected in {table}: "
                    f"source={source_count}, target={target_count}, "
                    f"discrepancy={discrepancy_pct:.2f}%"
                )
                
                # 메트릭 업데이트
                statsd.gauge(
                    'data.consistency.discrepancy',
                    discrepancy,
                    tags=[f'table:{table}']
                )

# 통합 상태 대시보드
class IntegrationDashboard:
    """실시간 통합 상태 대시보드"""
    
    def __init__(self):
        self.monitor = IntegrationMonitor()
        self.health_checks = {}
    
    def register_health_check(self, name, check_fn):
        """헬스 체크 등록"""
        self.health_checks[name] = check_fn
    
    async def run_health_checks(self):
        """모든 헬스 체크 실행"""
        results = {}
        
        for name, check_fn in self.health_checks.items():
            try:
                start_time = time.time()
                result = await check_fn()
                duration = time.time() - start_time
                
                results[name] = {
                    'status': 'healthy' if result else 'unhealthy',
                    'duration': duration,
                    'timestamp': datetime.utcnow().isoformat()
                }
                
                # 메트릭 업데이트
                statsd.gauge(
                    'integration.health',
                    1 if result else 0,
                    tags=[f'check:{name}']
                )
                
            except Exception as e:
                results[name] = {
                    'status': 'error',
                    'error': str(e),
                    'timestamp': datetime.utcnow().isoformat()
                }
                
                self.logger.error(f"Health check failed: {name}, error: {e}")
        
        return results
    
    def generate_summary_report(self):
        """통합 상태 요약 리포트 생성"""
        report = {
            'timestamp': datetime.utcnow().isoformat(),
            'overall_health': 'healthy',
            'components': {},
            'metrics': {},
            'alerts': []
        }
        
        # 각 컴포넌트 상태 수집
        # ... (구현)
        
        return report

# 사용 예제
async def main():
    # 통합 허브 초기화
    hub = DatabaseIntegrationHub()
    await hub.initialize()
    
    # 데이터 가상화 레이어
    virtualization = DataVirtualizationLayer(hub)
    
    # 통합 쿼리 실행
    results = await virtualization.query(
        'customer',
        filters={'city': 'Seoul'},
        fields=['id', 'name', 'email']
    )
    
    # CDC 시작
    cdc = ChangeDataCapture()
    cdc.setup_postgres_cdc({'host': 'localhost', 'database': 'mydb'})
    
    # ETL 파이프라인
    etl = ETLPipeline(
        source_config={'type': 'postgres', 'table': 'raw_data'},
        target_config={'type': 'bigquery', 'dataset': 'analytics'}
    )
    etl.add_transformation(DataTransformations.clean_phone_number)
    etl.add_transformation(DataTransformations.enrich_with_geolocation)
    etl.run()
    
    # 모니터링
    monitor = IntegrationMonitor()
    monitor.record_sync_operation('postgres', 'mongodb', 5.2)

if __name__ == "__main__":
    asyncio.run(main())
```

## 마무리

데이터베이스 통합과 마이그레이션은 현대 데이터 아키텍처의 핵심 과제입니다. CDC를 통한 실시간 동기화, 효율적인 ETL/ELT 프로세스, 무중단 마이그레이션 전략을 통해 이기종 데이터베이스 환경을 효과적으로 관리할 수 있습니다. 지속적인 모니터링과 데이터 품질 관리를 통해 안정적인 데이터 통합 환경을 유지하는 것이 중요합니다. 다음 장에서는 데이터 웨어하우스와 데이터 레이크의 개념과 구현 방법을 학습하겠습니다.