# 실시간 데이터 처리

## 개요

실시간 데이터 처리는 데이터가 생성되는 즉시 처리하고 분석하는 기술로, 금융 거래, IoT 센서 데이터, 소셜 미디어 스트림, 로그 분석 등 다양한 분야에서 필수적입니다. 이 장에서는 스트리밍 데이터 처리 아키텍처, 주요 기술 스택, 그리고 실시간 분석 시스템 구현 방법을 학습합니다.

## 1. 스트리밍 데이터 기초

### 1.1 Apache Kafka 기반 아키텍처

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.admin import KafkaAdminClient, NewTopic
import json
import time
from datetime import datetime
import threading
import avro.schema
import avro.io
import io

class KafkaStreamingPlatform:
    """Kafka 기반 스트리밍 플랫폼"""
    
    def __init__(self, bootstrap_servers='localhost:9092'):
        self.bootstrap_servers = bootstrap_servers
        self.admin_client = KafkaAdminClient(
            bootstrap_servers=bootstrap_servers,
            client_id='streaming_platform'
        )
        
    def create_topics_with_configuration(self):
        """최적화된 토픽 생성"""
        topics = [
            NewTopic(
                name='transactions',
                num_partitions=12,
                replication_factor=3,
                topic_configs={
                    'retention.ms': '86400000',  # 1일
                    'segment.ms': '3600000',     # 1시간
                    'cleanup.policy': 'delete',
                    'compression.type': 'lz4',
                    'min.insync.replicas': '2'
                }
            ),
            NewTopic(
                name='user-events',
                num_partitions=24,
                replication_factor=3,
                topic_configs={
                    'retention.ms': '604800000',  # 7일
                    'compression.type': 'snappy',
                    'max.message.bytes': '1048576'  # 1MB
                }
            ),
            NewTopic(
                name='metrics',
                num_partitions=6,
                replication_factor=2,
                topic_configs={
                    'retention.ms': '21600000',   # 6시간
                    'cleanup.policy': 'compact',
                    'segment.ms': '600000',       # 10분
                    'min.cleanable.dirty.ratio': '0.1'
                }
            )
        ]
        
        try:
            self.admin_client.create_topics(topics)
            print("토픽 생성 완료")
        except Exception as e:
            print(f"토픽 생성 오류: {e}")
    
    def implement_exactly_once_producer(self):
        """Exactly-Once 시맨틱 프로듀서"""
        producer = KafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            key_serializer=lambda k: k.encode('utf-8'),
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',  # 모든 복제본 확인
            enable_idempotence=True,  # 멱등성 활성화
            max_in_flight_requests_per_connection=5,
            retries=10,
            retry_backoff_ms=100,
            transactional_id='transaction-producer-1'  # 트랜잭션 ID
        )
        
        # 트랜잭션 초기화
        producer.init_transactions()
        
        try:
            # 트랜잭션 시작
            producer.begin_transaction()
            
            # 여러 메시지를 원자적으로 전송
            for i in range(100):
                transaction = {
                    'transaction_id': f'TXN-{datetime.now().timestamp()}-{i}',
                    'user_id': f'USER-{i % 10}',
                    'amount': 100.0 + i,
                    'timestamp': datetime.now().isoformat(),
                    'type': 'PURCHASE'
                }
                
                producer.send(
                    'transactions',
                    key=transaction['transaction_id'],
                    value=transaction
                )
            
            # 트랜잭션 커밋
            producer.commit_transaction()
            print("트랜잭션 커밋 완료")
            
        except Exception as e:
            producer.abort_transaction()
            print(f"트랜잭션 실패: {e}")
        finally:
            producer.close()
    
    def create_stream_processor(self):
        """스트림 프로세서 구현"""
        from kafka import TopicPartition
        
        consumer = KafkaConsumer(
            'transactions',
            bootstrap_servers=self.bootstrap_servers,
            group_id='stream-processor',
            enable_auto_commit=False,  # 수동 커밋
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            max_poll_records=500,
            session_timeout_ms=30000,
            heartbeat_interval_ms=10000
        )
        
        # 상태 저장소 (실제로는 RocksDB 등 사용)
        state_store = {}
        
        def process_batch(records):
            """배치 처리 로직"""
            aggregations = {}
            
            for record in records:
                user_id = record.value['user_id']
                amount = record.value['amount']
                
                # 사용자별 집계
                if user_id not in aggregations:
                    aggregations[user_id] = {
                        'count': 0,
                        'total_amount': 0,
                        'last_updated': None
                    }
                
                aggregations[user_id]['count'] += 1
                aggregations[user_id]['total_amount'] += amount
                aggregations[user_id]['last_updated'] = record.value['timestamp']
            
            # 상태 저장소 업데이트
            for user_id, stats in aggregations.items():
                if user_id not in state_store:
                    state_store[user_id] = stats
                else:
                    state_store[user_id]['count'] += stats['count']
                    state_store[user_id]['total_amount'] += stats['total_amount']
                    state_store[user_id]['last_updated'] = stats['last_updated']
            
            return aggregations
        
        # 처리 루프
        try:
            while True:
                records = consumer.poll(timeout_ms=1000)
                
                if records:
                    # 파티션별 처리
                    for partition, partition_records in records.items():
                        results = process_batch(partition_records)
                        print(f"파티션 {partition.partition} 처리: {len(partition_records)} 레코드")
                    
                    # 오프셋 커밋
                    consumer.commit()
                    
        except KeyboardInterrupt:
            print("프로세서 종료")
        finally:
            consumer.close()
```

### 1.2 Apache Flink 실시간 처리

```python
# Flink Python API (PyFlink) 예제
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings
from pyflink.table.udf import udf
from pyflink.table.expressions import col, lit
import pandas as pd

class FlinkStreamProcessor:
    """Apache Flink 스트림 프로세서"""
    
    def __init__(self):
        # 실행 환경 설정
        env_settings = EnvironmentSettings.new_instance()\
            .in_streaming_mode()\
            .use_blink_planner()\
            .build()
            
        self.env = StreamExecutionEnvironment.get_execution_environment()
        self.t_env = StreamTableEnvironment.create(self.env, env_settings)
        
        # 체크포인트 설정
        self.env.enable_checkpointing(10000)  # 10초마다
        
    def create_kafka_source_table(self):
        """Kafka 소스 테이블 생성"""
        kafka_source_ddl = """
        CREATE TABLE transactions (
            transaction_id STRING,
            user_id STRING,
            amount DOUBLE,
            transaction_time TIMESTAMP(3),
            category STRING,
            WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
        ) WITH (
            'connector' = 'kafka',
            'topic' = 'transactions',
            'properties.bootstrap.servers' = 'localhost:9092',
            'properties.group.id' = 'flink-consumer',
            'scan.startup.mode' = 'latest-offset',
            'format' = 'json',
            'json.fail-on-missing-field' = 'false',
            'json.ignore-parse-errors' = 'true'
        )
        """
        
        self.t_env.execute_sql(kafka_source_ddl)
        
    def implement_windowed_aggregation(self):
        """윈도우 집계 구현"""
        # 세션 윈도우 집계
        session_window_query = """
        SELECT 
            user_id,
            SESSION_START(transaction_time, INTERVAL '30' MINUTE) as session_start,
            SESSION_END(transaction_time, INTERVAL '30' MINUTE) as session_end,
            COUNT(*) as transaction_count,
            SUM(amount) as total_amount,
            AVG(amount) as avg_amount,
            MAX(amount) as max_amount,
            COLLECT(DISTINCT category) as categories
        FROM transactions
        GROUP BY 
            user_id,
            SESSION(transaction_time, INTERVAL '30' MINUTE)
        """
        
        # 슬라이딩 윈도우 집계
        sliding_window_query = """
        SELECT 
            user_id,
            HOP_START(transaction_time, INTERVAL '1' MINUTE, INTERVAL '5' MINUTE) as window_start,
            HOP_END(transaction_time, INTERVAL '1' MINUTE, INTERVAL '5' MINUTE) as window_end,
            COUNT(*) as transaction_count,
            SUM(amount) as total_amount,
            -- 이동 평균
            SUM(amount) / COUNT(*) as moving_average
        FROM transactions
        GROUP BY 
            user_id,
            HOP(transaction_time, INTERVAL '1' MINUTE, INTERVAL '5' MINUTE)
        """
        
        # 텀블링 윈도우와 조기 발생
        tumbling_window_query = """
        SELECT 
            TUMBLE_START(transaction_time, INTERVAL '1' HOUR) as hour_start,
            category,
            COUNT(*) as count,
            SUM(amount) as revenue,
            -- 지연 데이터 처리
            COUNT(*) FILTER (WHERE transaction_time < CURRENT_WATERMARK()) as on_time_count
        FROM transactions
        GROUP BY 
            TUMBLE(transaction_time, INTERVAL '1' HOUR),
            category
        EMIT 
            -- 조기 결과 발생
            WITH DELAY '10' SECOND,
            -- 최종 결과
            WITHOUT DELAY AFTER WATERMARK
        """
        
        return session_window_query, sliding_window_query, tumbling_window_query
    
    def implement_pattern_detection(self):
        """복잡한 이벤트 패턴 감지 (CEP)"""
        fraud_detection_query = """
        SELECT *
        FROM transactions
        MATCH_RECOGNIZE (
            PARTITION BY user_id
            ORDER BY transaction_time
            MEASURES
                FIRST(A.transaction_id) as first_transaction,
                LAST(D.transaction_id) as last_transaction,
                SUM(A.amount + B.amount + C.amount + D.amount) as total_amount
            ONE ROW PER MATCH
            AFTER MATCH SKIP TO NEXT ROW
            PATTERN (A B C D) WITHIN INTERVAL '10' MINUTE
            DEFINE
                A AS A.amount > 1000,
                B AS B.amount > A.amount * 1.5,
                C AS C.amount > B.amount * 1.5,
                D AS D.amount > C.amount * 1.5
        ) AS suspected_fraud
        """
        
        # 이상 탐지 패턴
        anomaly_detection_query = """
        SELECT 
            user_id,
            transaction_time,
            amount,
            -- 이동 표준편차 계산
            STDDEV(amount) OVER (
                PARTITION BY user_id 
                ORDER BY transaction_time 
                RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
            ) as moving_stddev,
            -- 이상치 스코어
            ABS(amount - AVG(amount) OVER (
                PARTITION BY user_id 
                ORDER BY transaction_time 
                RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
            )) / NULLIF(STDDEV(amount) OVER (
                PARTITION BY user_id 
                ORDER BY transaction_time 
                RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
            ), 0) as z_score
        FROM transactions
        """
        
        return fraud_detection_query, anomaly_detection_query
```

## 2. 스트림-테이블 조인

### 2.1 실시간 데이터 보강

```python
class StreamTableJoinProcessor:
    """스트림-테이블 조인 처리"""
    
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.postgres_pool = psycopg2.pool.SimpleConnectionPool(1, 20, 
            host='localhost', database='userdb', user='user', password='password')
    
    def enrich_stream_with_reference_data(self, kafka_consumer):
        """참조 데이터로 스트림 보강"""
        # 참조 데이터 캐시
        reference_cache = {}
        cache_ttl = 300  # 5분
        
        def get_user_profile(user_id):
            """사용자 프로필 조회 (캐시 우선)"""
            # Redis 캐시 확인
            cached_profile = self.redis_client.get(f"user:{user_id}")
            if cached_profile:
                return json.loads(cached_profile)
            
            # 데이터베이스 조회
            conn = self.postgres_pool.getconn()
            try:
                with conn.cursor() as cursor:
                    cursor.execute("""
                        SELECT user_id, name, email, tier, credit_limit, 
                               risk_score, created_at
                        FROM users 
                        WHERE user_id = %s
                    """, (user_id,))
                    
                    result = cursor.fetchone()
                    if result:
                        profile = {
                            'user_id': result[0],
                            'name': result[1],
                            'email': result[2],
                            'tier': result[3],
                            'credit_limit': float(result[4]),
                            'risk_score': float(result[5]),
                            'created_at': result[6].isoformat()
                        }
                        
                        # 캐시 저장
                        self.redis_client.setex(
                            f"user:{user_id}", 
                            cache_ttl, 
                            json.dumps(profile)
                        )
                        
                        return profile
            finally:
                self.postgres_pool.putconn(conn)
            
            return None
        
        # 스트림 처리
        for message in kafka_consumer:
            transaction = message.value
            user_id = transaction['user_id']
            
            # 사용자 프로필로 보강
            user_profile = get_user_profile(user_id)
            
            if user_profile:
                # 보강된 이벤트
                enriched_event = {
                    **transaction,
                    'user_profile': user_profile,
                    'risk_assessment': self.assess_risk(
                        transaction, user_profile
                    )
                }
                
                # 다음 처리 단계로 전송
                yield enriched_event
    
    def assess_risk(self, transaction, user_profile):
        """실시간 리스크 평가"""
        risk_factors = []
        risk_score = 0
        
        # 거래 금액 vs 신용 한도
        if transaction['amount'] > user_profile['credit_limit'] * 0.8:
            risk_factors.append('HIGH_AMOUNT')
            risk_score += 30
        
        # 사용자 리스크 점수
        if user_profile['risk_score'] > 0.7:
            risk_factors.append('HIGH_RISK_USER')
            risk_score += 40
        
        # 시간대 분석
        hour = datetime.fromisoformat(transaction['timestamp']).hour
        if hour >= 0 and hour <= 6:
            risk_factors.append('UNUSUAL_TIME')
            risk_score += 20
        
        return {
            'risk_score': min(risk_score, 100),
            'risk_factors': risk_factors,
            'requires_review': risk_score > 50
        }
```

## 3. 실시간 분석 엔진

### 3.1 Apache Druid 구현

```python
import requests
import json
from datetime import datetime, timedelta

class DruidRealtimeAnalytics:
    """Apache Druid 실시간 분석"""
    
    def __init__(self, coordinator_url='http://localhost:8081'):
        self.coordinator_url = coordinator_url
        self.broker_url = 'http://localhost:8082'
        
    def create_streaming_ingestion_spec(self):
        """스트리밍 수집 명세 생성"""
        ingestion_spec = {
            "type": "kafka",
            "dataSchema": {
                "dataSource": "transactions",
                "parser": {
                    "type": "string",
                    "parseSpec": {
                        "format": "json",
                        "timestampSpec": {
                            "column": "timestamp",
                            "format": "iso"
                        },
                        "dimensionsSpec": {
                            "dimensions": [
                                "transaction_id",
                                "user_id",
                                "category",
                                "payment_method",
                                "merchant_id"
                            ],
                            "dimensionExclusions": [],
                            "spatialDimensions": []
                        }
                    }
                },
                "metricsSpec": [
                    {"type": "count", "name": "count"},
                    {"type": "doubleSum", "name": "total_amount", "fieldName": "amount"},
                    {"type": "doubleMin", "name": "min_amount", "fieldName": "amount"},
                    {"type": "doubleMax", "name": "max_amount", "fieldName": "amount"},
                    {
                        "type": "thetaSketch",
                        "name": "unique_users",
                        "fieldName": "user_id"
                    },
                    {
                        "type": "hyperUnique",
                        "name": "approx_unique_merchants",
                        "fieldName": "merchant_id"
                    }
                ],
                "granularitySpec": {
                    "type": "uniform",
                    "segmentGranularity": "HOUR",
                    "queryGranularity": "MINUTE",
                    "rollup": True
                }
            },
            "tuningConfig": {
                "type": "kafka",
                "reportParseExceptions": True,
                "handoffConditionTimeout": 0,
                "resetOffsetAutomatically": False,
                "segmentWriteOutMediumFactory": {
                    "type": "onHeapMemory"
                },
                "intermediateHandoffPeriod": "PT10M",
                "maxRowsPerSegment": 5000000,
                "maxTotalRows": 20000000
            },
            "ioConfig": {
                "topic": "transactions",
                "consumerProperties": {
                    "bootstrap.servers": "localhost:9092"
                },
                "taskDuration": "PT1H",
                "useEarliestOffset": True,
                "completionTimeout": "PT30M",
                "lateMessageRejectionPeriod": "PT1H"
            }
        }
        
        # 수집 작업 제출
        response = requests.post(
            f"{self.coordinator_url}/druid/indexer/v1/supervisor",
            headers={'Content-Type': 'application/json'},
            data=json.dumps(ingestion_spec)
        )
        
        return response.json()
    
    def execute_realtime_queries(self):
        """실시간 쿼리 실행"""
        # TopN 쿼리 - 상위 소비자
        topn_query = {
            "queryType": "topN",
            "dataSource": "transactions",
            "intervals": ["2024-01-01/2024-12-31"],
            "granularity": "hour",
            "dimension": "user_id",
            "metric": "total_amount",
            "threshold": 10,
            "aggregations": [
                {"type": "doubleSum", "name": "total_amount", "fieldName": "total_amount"},
                {"type": "count", "name": "transaction_count"}
            ],
            "postAggregations": [
                {
                    "type": "arithmetic",
                    "name": "avg_transaction",
                    "fn": "/",
                    "fields": [
                        {"type": "fieldAccess", "fieldName": "total_amount"},
                        {"type": "fieldAccess", "fieldName": "transaction_count"}
                    ]
                }
            ]
        }
        
        # 시계열 쿼리
        timeseries_query = {
            "queryType": "timeseries",
            "dataSource": "transactions",
            "intervals": [f"{datetime.now() - timedelta(hours=24)}/{datetime.now()}"],
            "granularity": {
                "type": "period",
                "period": "PT5M"  # 5분 간격
            },
            "aggregations": [
                {"type": "count", "name": "count"},
                {"type": "doubleSum", "name": "revenue", "fieldName": "total_amount"},
                {"type": "thetaSketch", "name": "unique_users", "fieldName": "unique_users"}
            ],
            "postAggregations": [
                {
                    "type": "thetaSketchEstimate",
                    "name": "unique_user_count",
                    "field": {"type": "fieldAccess", "fieldName": "unique_users"}
                }
            ]
        }
        
        # 쿼리 실행
        response = requests.post(
            f"{self.broker_url}/druid/v2",
            headers={'Content-Type': 'application/json'},
            data=json.dumps(timeseries_query)
        )
        
        return response.json()
```

## 4. 이벤트 소싱과 CQRS

### 4.1 이벤트 소싱 구현

```python
import uuid
from abc import ABC, abstractmethod
from typing import List, Dict, Any
import asyncio
import aioredis

class Event(ABC):
    """이벤트 기본 클래스"""
    def __init__(self, aggregate_id: str, event_data: Dict[str, Any]):
        self.event_id = str(uuid.uuid4())
        self.aggregate_id = aggregate_id
        self.event_type = self.__class__.__name__
        self.event_data = event_data
        self.timestamp = datetime.utcnow()
        self.version = None

class OrderCreated(Event):
    pass

class OrderItemAdded(Event):
    pass

class OrderConfirmed(Event):
    pass

class OrderShipped(Event):
    pass

class EventStore:
    """이벤트 저장소"""
    
    def __init__(self, redis_url='redis://localhost'):
        self.redis_url = redis_url
        self.redis = None
        
    async def connect(self):
        self.redis = await aioredis.create_redis_pool(self.redis_url)
    
    async def append_events(self, aggregate_id: str, events: List[Event], 
                          expected_version: int = -1):
        """이벤트 추가"""
        async with self.redis.pipeline() as pipe:
            # 낙관적 동시성 제어
            current_version = await self.redis.get(f"version:{aggregate_id}")
            if current_version and int(current_version) != expected_version:
                raise Exception("Concurrency conflict")
            
            for i, event in enumerate(events):
                event.version = expected_version + i + 1
                event_data = {
                    'event_id': event.event_id,
                    'aggregate_id': event.aggregate_id,
                    'event_type': event.event_type,
                    'event_data': json.dumps(event.event_data),
                    'timestamp': event.timestamp.isoformat(),
                    'version': event.version
                }
                
                # 이벤트 저장
                pipe.zadd(
                    f"events:{aggregate_id}",
                    {json.dumps(event_data): event.version}
                )
                
                # 글로벌 이벤트 스트림
                pipe.xadd(
                    "global-event-stream",
                    {
                        'aggregate_id': aggregate_id,
                        'event_type': event.event_type,
                        'event_data': json.dumps(event_data)
                    }
                )
            
            # 버전 업데이트
            pipe.set(f"version:{aggregate_id}", expected_version + len(events))
            
            await pipe.execute()
    
    async def get_events(self, aggregate_id: str, from_version: int = 0):
        """이벤트 조회"""
        events_data = await self.redis.zrangebyscore(
            f"events:{aggregate_id}",
            from_version,
            '+inf',
            withscores=True
        )
        
        events = []
        for event_json, version in events_data:
            event_dict = json.loads(event_json)
            events.append(event_dict)
        
        return events

class EventProjector:
    """이벤트 프로젝션"""
    
    def __init__(self, event_store: EventStore):
        self.event_store = event_store
        self.projections = {}
        
    async def project_order_summary(self):
        """주문 요약 프로젝션"""
        # 글로벌 이벤트 스트림 구독
        last_id = '0'
        
        while True:
            # 새 이벤트 읽기
            events = await self.event_store.redis.xread(
                ['global-event-stream'],
                latest_ids=[last_id],
                count=100,
                timeout=1000
            )
            
            for stream_name, stream_events in events:
                for event_id, event_data in stream_events:
                    await self.handle_event(event_data)
                    last_id = event_id
    
    async def handle_event(self, event_data):
        """이벤트 처리"""
        event_type = event_data[b'event_type'].decode()
        aggregate_id = event_data[b'aggregate_id'].decode()
        
        if event_type == 'OrderCreated':
            await self.handle_order_created(aggregate_id, event_data)
        elif event_type == 'OrderItemAdded':
            await self.handle_order_item_added(aggregate_id, event_data)
        # ... 기타 이벤트 처리
```

## 5. 실시간 모니터링과 알림

### 5.1 메트릭 수집과 알림

```python
import prometheus_client
from prometheus_client import Counter, Histogram, Gauge
import aiohttp
import asyncio

class RealtimeMonitoringSystem:
    """실시간 모니터링 시스템"""
    
    def __init__(self):
        # Prometheus 메트릭 정의
        self.transaction_counter = Counter(
            'transactions_total', 
            'Total number of transactions',
            ['status', 'payment_method']
        )
        
        self.transaction_amount = Histogram(
            'transaction_amount_dollars',
            'Transaction amount in dollars',
            buckets=(10, 50, 100, 500, 1000, 5000, 10000)
        )
        
        self.active_users = Gauge(
            'active_users_current',
            'Currently active users'
        )
        
        self.fraud_detection_latency = Histogram(
            'fraud_detection_latency_seconds',
            'Fraud detection processing time'
        )
        
        # 알림 임계값
        self.alert_thresholds = {
            'high_failure_rate': 0.05,  # 5% 실패율
            'high_latency': 1.0,        # 1초
            'low_throughput': 100       # 분당 100건 미만
        }
        
    async def process_metrics_stream(self, kafka_consumer):
        """메트릭 스트림 처리"""
        window_size = 60  # 1분 윈도우
        metrics_window = []
        
        async for message in kafka_consumer:
            transaction = message.value
            
            # 메트릭 수집
            self.transaction_counter.labels(
                status=transaction['status'],
                payment_method=transaction['payment_method']
            ).inc()
            
            self.transaction_amount.observe(transaction['amount'])
            
            # 윈도우에 추가
            metrics_window.append({
                'timestamp': datetime.now(),
                'transaction': transaction
            })
            
            # 오래된 데이터 제거
            cutoff_time = datetime.now() - timedelta(seconds=window_size)
            metrics_window = [m for m in metrics_window 
                            if m['timestamp'] > cutoff_time]
            
            # 실시간 분석
            await self.analyze_window(metrics_window)
    
    async def analyze_window(self, metrics_window):
        """윈도우 분석 및 알림"""
        if not metrics_window:
            return
        
        # 실패율 계산
        total_transactions = len(metrics_window)
        failed_transactions = sum(1 for m in metrics_window 
                                if m['transaction']['status'] == 'FAILED')
        failure_rate = failed_transactions / total_transactions
        
        # 처리량 계산 (분당)
        throughput = total_transactions
        
        # 알림 확인
        alerts = []
        
        if failure_rate > self.alert_thresholds['high_failure_rate']:
            alerts.append({
                'type': 'HIGH_FAILURE_RATE',
                'severity': 'critical',
                'value': failure_rate,
                'threshold': self.alert_thresholds['high_failure_rate'],
                'message': f'High failure rate detected: {failure_rate:.2%}'
            })
        
        if throughput < self.alert_thresholds['low_throughput']:
            alerts.append({
                'type': 'LOW_THROUGHPUT',
                'severity': 'warning',
                'value': throughput,
                'threshold': self.alert_thresholds['low_throughput'],
                'message': f'Low throughput detected: {throughput} transactions/min'
            })
        
        # 알림 전송
        for alert in alerts:
            await self.send_alert(alert)
    
    async def send_alert(self, alert):
        """알림 전송"""
        # Slack 알림
        slack_webhook = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        
        async with aiohttp.ClientSession() as session:
            await session.post(slack_webhook, json={
                'text': f"🚨 {alert['severity'].upper()}: {alert['message']}",
                'attachments': [{
                    'color': 'danger' if alert['severity'] == 'critical' else 'warning',
                    'fields': [
                        {'title': 'Alert Type', 'value': alert['type'], 'short': True},
                        {'title': 'Current Value', 'value': str(alert['value']), 'short': True},
                        {'title': 'Threshold', 'value': str(alert['threshold']), 'short': True},
                        {'title': 'Time', 'value': datetime.now().isoformat(), 'short': True}
                    ]
                }]
            })
```

## 마무리

실시간 데이터 처리는 현대 데이터 아키텍처의 핵심 구성 요소입니다. Apache Kafka, Flink, Druid와 같은 도구들을 활용하여 대용량 스트리밍 데이터를 효과적으로 처리하고 분석할 수 있습니다. 이벤트 소싱과 CQRS 패턴은 복잡한 비즈니스 로직을 처리하면서도 높은 성능과 확장성을 제공합니다.

다음 장에서는 데이터베이스 모니터링과 성능 진단에 대해 학습하겠습니다.