# 빅데이터와 분석 데이터베이스

## 개요

빅데이터 시대에는 전통적인 관계형 데이터베이스로는 처리하기 어려운 대용량, 고속, 다양한 형태의 데이터를 다뤄야 합니다. 이 장에서는 빅데이터 처리를 위한 분산 컴퓨팅 프레임워크, 분석 전용 데이터베이스, 실시간 분석 기술, 그리고 데이터 레이크하우스 아키텍처를 학습합니다.

## 1. Apache Spark 기반 빅데이터 처리

### 1.1 Spark SQL과 DataFrame API

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from pyspark.sql.window import Window
import pyspark.sql.functions as F

class SparkBigDataProcessor:
    """Spark 기반 빅데이터 처리기"""
    
    def __init__(self, app_name="BigDataAnalytics"):
        self.spark = SparkSession.builder \
            .appName(app_name) \
            .config("spark.sql.adaptive.enabled", "true") \
            .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
            .config("spark.sql.adaptive.skewJoin.enabled", "true") \
            .config("spark.sql.adaptive.localShuffleReader.enabled", "true") \
            .config("spark.sql.catalog.spark_catalog", 
                   "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
            .config("spark.sql.extensions", 
                   "io.delta.sql.DeltaSparkSessionExtension") \
            .getOrCreate()
        
        # 성능 최적화 설정
        self.spark.conf.set("spark.sql.shuffle.partitions", "200")
        self.spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "10MB")
        
    def process_large_dataset(self, input_path, output_path):
        """대용량 데이터셋 처리"""
        # 파티션된 데이터 읽기
        df = self.spark.read \
            .option("mergeSchema", "true") \
            .option("recursiveFileLookup", "true") \
            .parquet(input_path)
        
        # 데이터 프로파일링
        data_profile = self.profile_dataset(df)
        
        # 복잡한 변환 수행
        transformed_df = self.apply_transformations(df)
        
        # 최적화된 쓰기
        transformed_df.repartition(100) \
            .sortWithinPartitions("timestamp") \
            .write \
            .mode("overwrite") \
            .option("compression", "snappy") \
            .partitionBy("year", "month", "day") \
            .parquet(output_path)
        
        return data_profile
    
    def profile_dataset(self, df):
        """데이터셋 프로파일링"""
        profile = {
            'row_count': df.count(),
            'column_count': len(df.columns),
            'columns': {}
        }
        
        # 각 컬럼별 통계
        for col_name in df.columns:
            col_type = dict(df.dtypes)[col_name]
            
            if col_type in ['int', 'bigint', 'float', 'double']:
                # 수치형 컬럼 통계
                stats = df.select(
                    F.min(col_name).alias('min'),
                    F.max(col_name).alias('max'),
                    F.avg(col_name).alias('avg'),
                    F.stddev(col_name).alias('stddev'),
                    F.count(F.when(F.col(col_name).isNull(), 1)).alias('null_count'),
                    F.approx_count_distinct(col_name).alias('distinct_count')
                ).collect()[0].asDict()
                
                profile['columns'][col_name] = {
                    'type': col_type,
                    'stats': stats
                }
                
            elif col_type == 'string':
                # 문자열 컬럼 통계
                stats = df.select(
                    F.count(F.when(F.col(col_name).isNull(), 1)).alias('null_count'),
                    F.approx_count_distinct(col_name).alias('distinct_count'),
                    F.min(F.length(col_name)).alias('min_length'),
                    F.max(F.length(col_name)).alias('max_length'),
                    F.avg(F.length(col_name)).alias('avg_length')
                ).collect()[0].asDict()
                
                # 상위 빈도 값
                top_values = df.groupBy(col_name) \
                    .count() \
                    .orderBy(F.desc('count')) \
                    .limit(10) \
                    .collect()
                
                profile['columns'][col_name] = {
                    'type': col_type,
                    'stats': stats,
                    'top_values': [(row[col_name], row['count']) for row in top_values]
                }
        
        return profile
    
    def apply_transformations(self, df):
        """복잡한 데이터 변환"""
        # 윈도우 함수를 사용한 시계열 분석
        time_window = Window.partitionBy("user_id") \
            .orderBy("timestamp") \
            .rowsBetween(-3600, 0)  # 1시간 윈도우
        
        # 세션화
        session_window = Window.partitionBy("user_id").orderBy("timestamp")
        
        transformed_df = df \
            .withColumn("prev_timestamp", 
                       F.lag("timestamp").over(session_window)) \
            .withColumn("time_diff", 
                       F.when(F.col("prev_timestamp").isNotNull(),
                              F.col("timestamp") - F.col("prev_timestamp"))
                       .otherwise(0)) \
            .withColumn("new_session", 
                       F.when(F.col("time_diff") > 1800, 1).otherwise(0)) \
            .withColumn("session_id", 
                       F.sum("new_session").over(session_window)) \
            .withColumn("events_in_window", 
                       F.count("event_id").over(time_window)) \
            .withColumn("unique_pages_in_window", 
                       F.approx_count_distinct("page_url").over(time_window)) \
            .withColumn("revenue_in_window", 
                       F.sum("revenue").over(time_window))
        
        # 사용자 행동 패턴 추출
        user_patterns = transformed_df \
            .groupBy("user_id", "session_id") \
            .agg(
                F.min("timestamp").alias("session_start"),
                F.max("timestamp").alias("session_end"),
                F.count("event_id").alias("event_count"),
                F.sum("revenue").alias("session_revenue"),
                F.collect_list("page_url").alias("page_sequence"),
                F.collect_list("event_type").alias("event_sequence")
            ) \
            .withColumn("session_duration", 
                       F.col("session_end") - F.col("session_start")) \
            .withColumn("conversion", 
                       F.when(F.col("session_revenue") > 0, 1).otherwise(0))
        
        # 머신러닝 특징 생성
        ml_features = user_patterns \
            .groupBy("user_id") \
            .agg(
                F.count("session_id").alias("total_sessions"),
                F.avg("session_duration").alias("avg_session_duration"),
                F.avg("event_count").alias("avg_events_per_session"),
                F.sum("session_revenue").alias("total_revenue"),
                F.avg("conversion").alias("conversion_rate"),
                F.stddev("session_duration").alias("session_duration_variance")
            )
        
        return transformed_df.join(ml_features, on="user_id", how="left")
    
    def optimize_query_performance(self, query):
        """쿼리 성능 최적화"""
        # 쿼리 계획 분석
        df = self.spark.sql(query)
        
        # 물리적 계획 확인
        print("Physical Plan:")
        df.explain(mode="formatted")
        
        # 비용 기반 최적화 힌트 추가
        optimized_query = f"""
        WITH /*+ BROADCAST(small_table) */ 
        optimized AS (
            {query}
        )
        SELECT /*+ REPARTITION(100) */ * FROM optimized
        """
        
        # 적응형 쿼리 실행
        self.spark.conf.set("spark.sql.adaptive.enabled", "true")
        self.spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
        
        return self.spark.sql(optimized_query)
```

### 1.2 Delta Lake 구현

```python
from delta import *
from pyspark.sql.streaming import StreamingQuery
import concurrent.futures

class DeltaLakeManager:
    """Delta Lake 관리자"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.delta_table_path = "/data/delta-tables"
        
    def create_delta_table(self, table_name, schema, partition_cols=None):
        """Delta 테이블 생성"""
        # 스키마 정의
        schema_ddl = self.generate_schema_ddl(schema)
        
        create_sql = f"""
        CREATE TABLE IF NOT EXISTS {table_name}
        ({schema_ddl})
        USING DELTA
        """
        
        if partition_cols:
            create_sql += f" PARTITIONED BY ({', '.join(partition_cols)})"
            
        create_sql += f" LOCATION '{self.delta_table_path}/{table_name}'"
        
        self.spark.sql(create_sql)
        
        # 테이블 속성 설정
        self.spark.sql(f"""
            ALTER TABLE {table_name} SET TBLPROPERTIES (
                'delta.autoOptimize.optimizeWrite' = 'true',
                'delta.autoOptimize.autoCompact' = 'true',
                'delta.dataSkippingNumIndexedCols' = '32',
                'delta.logRetentionDuration' = 'interval 30 days',
                'delta.deletedFileRetentionDuration' = 'interval 7 days'
            )
        """)
        
    def upsert_data(self, target_table, source_df, merge_keys):
        """Delta Lake UPSERT (MERGE) 작업"""
        # 임시 뷰 생성
        source_df.createOrReplaceTempView("source_data")
        
        # MERGE 조건 생성
        merge_condition = " AND ".join([
            f"target.{key} = source.{key}" for key in merge_keys
        ])
        
        # MERGE 실행
        merge_sql = f"""
        MERGE INTO {target_table} AS target
        USING source_data AS source
        ON {merge_condition}
        WHEN MATCHED THEN
            UPDATE SET *
        WHEN NOT MATCHED THEN
            INSERT *
        """
        
        self.spark.sql(merge_sql)
        
        # 최적화 실행
        self.optimize_table(target_table)
        
    def time_travel_query(self, table_name, version=None, timestamp=None):
        """시간 여행 쿼리"""
        if version is not None:
            return self.spark.read \
                .format("delta") \
                .option("versionAsOf", version) \
                .table(table_name)
        elif timestamp is not None:
            return self.spark.read \
                .format("delta") \
                .option("timestampAsOf", timestamp) \
                .table(table_name)
        else:
            return self.spark.table(table_name)
    
    def implement_cdc(self, table_name, start_version=0):
        """Change Data Capture 구현"""
        # 변경 사항 읽기
        changes_df = self.spark.read \
            .format("delta") \
            .option("readChangeFeed", "true") \
            .option("startingVersion", start_version) \
            .table(table_name)
        
        # 변경 유형별 처리
        inserts = changes_df.filter("_change_type = 'insert'")
        updates = changes_df.filter("_change_type = 'update_postimage'")
        deletes = changes_df.filter("_change_type = 'delete'")
        
        return {
            'inserts': inserts,
            'updates': updates,
            'deletes': deletes,
            'total_changes': changes_df.count()
        }
    
    def optimize_table(self, table_name):
        """테이블 최적화"""
        # Z-ORDER 최적화
        self.spark.sql(f"""
            OPTIMIZE {table_name}
            ZORDER BY (timestamp, user_id)
        """)
        
        # 오래된 파일 정리
        self.spark.sql(f"VACUUM {table_name} RETAIN 168 HOURS")
        
    def create_streaming_pipeline(self):
        """스트리밍 파이프라인 생성"""
        # 스트리밍 소스 (Kafka)
        stream_df = self.spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "localhost:9092") \
            .option("subscribe", "events") \
            .option("startingOffsets", "latest") \
            .load()
        
        # 데이터 파싱 및 변환
        parsed_df = stream_df \
            .select(F.from_json(F.col("value").cast("string"), 
                               self.get_event_schema()).alias("data")) \
            .select("data.*") \
            .withColumn("processing_time", F.current_timestamp())
        
        # 워터마크 설정
        windowed_df = parsed_df \
            .withWatermark("event_time", "10 minutes") \
            .groupBy(
                F.window("event_time", "5 minutes", "1 minute"),
                "user_id"
            ) \
            .agg(
                F.count("event_id").alias("event_count"),
                F.sum("revenue").alias("total_revenue"),
                F.approx_count_distinct("session_id").alias("unique_sessions")
            )
        
        # Delta Lake에 스트리밍 쓰기
        query = windowed_df \
            .writeStream \
            .format("delta") \
            .outputMode("append") \
            .option("checkpointLocation", "/checkpoint/streaming") \
            .trigger(processingTime="1 minute") \
            .table("real_time_analytics")
        
        return query
```

## 2. 분석 전용 데이터베이스

### 2.1 ClickHouse 구현

```python
from clickhouse_driver import Client
import pandas as pd
from typing import List, Dict, Any

class ClickHouseAnalytics:
    """ClickHouse 분석 시스템"""
    
    def __init__(self, host='localhost', port=9000):
        self.client = Client(host=host, port=port)
        
    def create_analytics_tables(self):
        """분석용 테이블 생성"""
        # 분산 테이블 생성
        self.client.execute("""
        CREATE TABLE IF NOT EXISTS events_distributed ON CLUSTER analytics_cluster
        (
            event_id UUID,
            user_id UInt64,
            event_time DateTime,
            event_type Enum8('view' = 1, 'click' = 2, 'purchase' = 3),
            page_url String,
            product_id UInt32,
            revenue Decimal(10, 2),
            session_id UUID,
            device_type LowCardinality(String),
            country LowCardinality(String),
            INDEX idx_user_time (user_id, event_time) TYPE minmax GRANULARITY 8192,
            INDEX idx_url (page_url) TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4,
            INDEX idx_product (product_id) TYPE set(1000) GRANULARITY 4
        )
        ENGINE = MergeTree()
        PARTITION BY toYYYYMM(event_time)
        ORDER BY (event_time, user_id, event_id)
        TTL event_time + INTERVAL 90 DAY
        SETTINGS index_granularity = 8192
        """)
        
        # 물질화된 뷰 생성 (실시간 집계)
        self.client.execute("""
        CREATE MATERIALIZED VIEW IF NOT EXISTS hourly_stats
        ENGINE = SummingMergeTree()
        PARTITION BY toYYYYMMDD(hour)
        ORDER BY (hour, event_type, country)
        AS SELECT
            toStartOfHour(event_time) AS hour,
            event_type,
            country,
            count() AS events,
            uniq(user_id) AS unique_users,
            sum(revenue) AS total_revenue,
            avg(revenue) AS avg_revenue
        FROM events_distributed
        GROUP BY hour, event_type, country
        """)
        
        # 사용자 세그먼트 테이블
        self.client.execute("""
        CREATE TABLE IF NOT EXISTS user_segments
        (
            user_id UInt64,
            segment_date Date,
            total_events UInt32,
            total_revenue Decimal(10, 2),
            days_active UInt16,
            last_active DateTime,
            segment LowCardinality(String),
            ltv_prediction Decimal(10, 2)
        )
        ENGINE = ReplacingMergeTree(last_active)
        PARTITION BY toYYYYMM(segment_date)
        ORDER BY (user_id, segment_date)
        """)
        
    def execute_complex_analytics(self):
        """복잡한 분석 쿼리 실행"""
        # 코호트 분석
        cohort_analysis = """
        WITH cohort_data AS (
            SELECT 
                user_id,
                toStartOfMonth(min(event_time)) AS cohort_month,
                toStartOfMonth(event_time) AS event_month,
                sum(revenue) AS monthly_revenue
            FROM events_distributed
            WHERE event_type = 'purchase'
            GROUP BY user_id, event_month
        )
        SELECT 
            cohort_month,
            dateDiff('month', cohort_month, event_month) AS months_since_join,
            count(DISTINCT user_id) AS users,
            sum(monthly_revenue) AS revenue,
            avg(monthly_revenue) AS avg_revenue_per_user
        FROM cohort_data
        GROUP BY cohort_month, months_since_join
        ORDER BY cohort_month, months_since_join
        """
        
        # 퍼널 분석
        funnel_analysis = """
        SELECT 
            windowFunnel(3600)(
                event_time,
                event_type = 'view',
                event_type = 'click',
                event_type = 'purchase'
            ) AS funnel_level,
            count() AS users
        FROM (
            SELECT 
                user_id,
                event_time,
                event_type
            FROM events_distributed
            WHERE event_time >= today() - 7
            ORDER BY user_id, event_time
        )
        GROUP BY user_id
        """
        
        # 실시간 대시보드 쿼리
        dashboard_query = """
        SELECT 
            toStartOfMinute(event_time) AS minute,
            event_type,
            count() AS events,
            uniq(user_id) AS unique_users,
            sum(revenue) AS revenue,
            quantile(0.5)(revenue) AS median_revenue,
            quantile(0.95)(revenue) AS p95_revenue
        FROM events_distributed
        WHERE event_time >= now() - INTERVAL 1 HOUR
        GROUP BY minute, event_type
        ORDER BY minute DESC
        FORMAT JSON
        """
        
        results = {
            'cohort': self.client.execute(cohort_analysis),
            'funnel': self.client.execute(funnel_analysis),
            'dashboard': self.client.execute(dashboard_query)
        }
        
        return results
    
    def optimize_query_performance(self):
        """쿼리 성능 최적화"""
        # 쿼리 프로파일링
        self.client.execute("SET send_logs_level = 'trace'")
        self.client.execute("SET log_queries = 1")
        self.client.execute("SET log_query_threads = 1")
        
        # 샘플링을 사용한 근사 쿼리
        approximate_query = """
        SELECT 
            event_type,
            countIf(revenue > 100) AS high_value_events,
            uniqExact(user_id) AS exact_users,
            uniqCombined(user_id) AS approx_users,
            quantilesTDigest(0.5, 0.9, 0.99)(revenue) AS revenue_percentiles
        FROM events_distributed
        SAMPLE 0.1  -- 10% 샘플링
        WHERE event_time >= today()
        GROUP BY event_type
        """
        
        # 프로젝션 사용
        self.client.execute("""
        ALTER TABLE events_distributed
        ADD PROJECTION revenue_by_country
        (
            SELECT 
                country,
                event_type,
                sum(revenue),
                count()
            GROUP BY country, event_type
        )
        """)
        
        # 분산 처리 최적화
        self.client.execute("SET max_threads = 8")
        self.client.execute("SET max_memory_usage = 10000000000")  # 10GB
        self.client.execute("SET distributed_aggregation_memory_efficient = 1")
```

### 2.2 Apache Druid 실시간 분석

```python
import requests
import json
from datetime import datetime, timedelta

class DruidRealtimeAnalytics:
    """Apache Druid 실시간 분석"""
    
    def __init__(self, coordinator_url='http://localhost:8081', 
                 broker_url='http://localhost:8082'):
        self.coordinator_url = coordinator_url
        self.broker_url = broker_url
        
    def create_ingestion_spec(self):
        """실시간 수집 사양 생성"""
        ingestion_spec = {
            "type": "kafka",
            "spec": {
                "dataSchema": {
                    "dataSource": "real_time_events",
                    "timestampSpec": {
                        "column": "timestamp",
                        "format": "auto"
                    },
                    "dimensionsSpec": {
                        "dimensions": [
                            "user_id",
                            "event_type",
                            "product_id",
                            "country",
                            {
                                "type": "long",
                                "name": "session_duration"
                            }
                        ],
                        "dimensionExclusions": [],
                        "spatialDimensions": []
                    },
                    "metricsSpec": [
                        {"type": "count", "name": "count"},
                        {"type": "doubleSum", "name": "revenue", "fieldName": "revenue"},
                        {"type": "longSum", "name": "quantity", "fieldName": "quantity"},
                        {
                            "type": "thetaSketch",
                            "name": "unique_users",
                            "fieldName": "user_id",
                            "size": 16384
                        },
                        {
                            "type": "HLLSketchBuild",
                            "name": "unique_sessions",
                            "fieldName": "session_id"
                        },
                        {
                            "type": "quantilesDoublesSketch",
                            "name": "revenue_quantiles",
                            "fieldName": "revenue"
                        }
                    ],
                    "granularitySpec": {
                        "type": "uniform",
                        "segmentGranularity": "HOUR",
                        "queryGranularity": "MINUTE",
                        "rollup": True
                    },
                    "transformSpec": {
                        "transforms": [
                            {
                                "type": "expression",
                                "name": "high_value",
                                "expression": "revenue > 100 ? 1 : 0"
                            }
                        ]
                    }
                },
                "tuningConfig": {
                    "type": "kafka",
                    "maxRowsInMemory": 1000000,
                    "maxBytesInMemory": 1073741824,
                    "intermediatePersistPeriod": "PT10M",
                    "maxPendingPersists": 0,
                    "indexSpec": {
                        "bitmap": {"type": "roaring"},
                        "dimensionCompression": "lz4",
                        "metricCompression": "lz4"
                    }
                },
                "ioConfig": {
                    "topic": "events",
                    "consumerProperties": {
                        "bootstrap.servers": "localhost:9092"
                    },
                    "taskCount": 2,
                    "replicas": 1,
                    "taskDuration": "PT1H"
                }
            }
        }
        
        # 수집 작업 제출
        response = requests.post(
            f"{self.coordinator_url}/druid/indexer/v1/supervisor",
            headers={'Content-Type': 'application/json'},
            data=json.dumps(ingestion_spec)
        )
        
        return response.json()
    
    def execute_olap_queries(self):
        """OLAP 스타일 쿼리 실행"""
        # 다차원 집계 쿼리
        cube_query = {
            "queryType": "groupBy",
            "dataSource": "real_time_events",
            "intervals": ["2024-01-01/2024-12-31"],
            "granularity": "hour",
            "dimensions": ["country", "event_type", "product_category"],
            "aggregations": [
                {"type": "doubleSum", "name": "total_revenue", "fieldName": "revenue"},
                {"type": "longSum", "name": "total_quantity", "fieldName": "quantity"},
                {"type": "thetaSketch", "name": "unique_users", "fieldName": "unique_users"}
            ],
            "postAggregations": [
                {
                    "type": "arithmetic",
                    "name": "avg_order_value",
                    "fn": "/",
                    "fields": [
                        {"type": "fieldAccess", "fieldName": "total_revenue"},
                        {"type": "fieldAccess", "fieldName": "total_quantity"}
                    ]
                },
                {
                    "type": "thetaSketchEstimate",
                    "name": "user_count",
                    "field": {"type": "fieldAccess", "fieldName": "unique_users"}
                }
            ],
            "having": {
                "type": "greaterThan",
                "aggregation": "total_revenue",
                "value": 1000
            },
            "limitSpec": {
                "type": "default",
                "limit": 100,
                "columns": [{"dimension": "total_revenue", "direction": "descending"}]
            }
        }
        
        # 시계열 분석 쿼리
        timeseries_query = {
            "queryType": "timeseries",
            "dataSource": "real_time_events",
            "intervals": [f"{datetime.now() - timedelta(days=7)}/{datetime.now()}"],
            "granularity": {
                "type": "period",
                "period": "PT1H",
                "timeZone": "UTC"
            },
            "aggregations": [
                {"type": "count", "name": "events"},
                {"type": "doubleSum", "name": "revenue", "fieldName": "revenue"},
                {"type": "thetaSketch", "name": "users", "fieldName": "unique_users"}
            ],
            "postAggregations": [
                {
                    "type": "thetaSketchEstimate",
                    "name": "unique_user_count",
                    "field": {"type": "fieldAccess", "fieldName": "users"}
                }
            ],
            "filter": {
                "type": "and",
                "fields": [
                    {"type": "selector", "dimension": "country", "value": "US"},
                    {"type": "in", "dimension": "event_type", "values": ["purchase", "add_to_cart"]}
                ]
            }
        }
        
        # 쿼리 실행
        results = {}
        for query_name, query in [("cube", cube_query), ("timeseries", timeseries_query)]:
            response = requests.post(
                f"{self.broker_url}/druid/v2/",
                headers={'Content-Type': 'application/json'},
                data=json.dumps(query)
            )
            results[query_name] = response.json()
        
        return results
```

## 3. 데이터 레이크하우스 아키텍처

### 3.1 Databricks Delta Lake 구현

```python
class DataLakehouseArchitecture:
    """데이터 레이크하우스 아키텍처"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.bronze_path = "/mnt/datalake/bronze"
        self.silver_path = "/mnt/datalake/silver"
        self.gold_path = "/mnt/datalake/gold"
        
    def implement_medallion_architecture(self):
        """메달리온 아키텍처 구현"""
        # Bronze Layer - 원시 데이터
        self.ingest_to_bronze()
        
        # Silver Layer - 정제된 데이터
        self.process_to_silver()
        
        # Gold Layer - 비즈니스 레벨 집계
        self.aggregate_to_gold()
        
    def ingest_to_bronze(self):
        """Bronze 레이어로 데이터 수집"""
        # 다양한 소스에서 원시 데이터 수집
        # JSON 파일
        json_df = self.spark.read \
            .option("multiline", "true") \
            .json("/source/json/*.json") \
            .withColumn("ingestion_time", F.current_timestamp()) \
            .withColumn("source_file", F.input_file_name())
        
        json_df.write \
            .format("delta") \
            .mode("append") \
            .option("mergeSchema", "true") \
            .partitionBy("ingestion_date") \
            .save(f"{self.bronze_path}/json_events")
        
        # CSV 파일
        csv_df = self.spark.read \
            .option("header", "true") \
            .option("inferSchema", "true") \
            .csv("/source/csv/*.csv") \
            .withColumn("ingestion_time", F.current_timestamp())
        
        csv_df.write \
            .format("delta") \
            .mode("append") \
            .save(f"{self.bronze_path}/csv_data")
        
        # 스트리밍 데이터
        stream_df = self.spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "localhost:9092") \
            .option("subscribe", "raw_events") \
            .load()
        
        stream_df \
            .selectExpr("CAST(value AS STRING)", "timestamp") \
            .writeStream \
            .format("delta") \
            .outputMode("append") \
            .option("checkpointLocation", f"{self.bronze_path}/_checkpoint") \
            .trigger(processingTime="5 minutes") \
            .start(f"{self.bronze_path}/streaming_events")
    
    def process_to_silver(self):
        """Silver 레이어로 데이터 처리"""
        # Bronze 데이터 읽기
        bronze_df = self.spark.read \
            .format("delta") \
            .load(f"{self.bronze_path}/json_events")
        
        # 데이터 정제 및 변환
        silver_df = bronze_df \
            .filter(F.col("event_type").isNotNull()) \
            .withColumn("event_date", F.to_date("timestamp")) \
            .withColumn("user_id_clean", F.regexp_replace("user_id", "[^0-9]", "")) \
            .withColumn("revenue_usd", 
                       F.when(F.col("currency") == "EUR", F.col("amount") * 1.1)
                        .when(F.col("currency") == "GBP", F.col("amount") * 1.3)
                        .otherwise(F.col("amount"))) \
            .drop("_corrupt_record")
        
        # 데이터 품질 검증
        quality_checks = [
            F.col("user_id_clean").isNotNull(),
            F.col("timestamp").isNotNull(),
            F.col("revenue_usd") >= 0
        ]
        
        validated_df = silver_df.filter(F.expr(" AND ".join([str(check) for check in quality_checks])))
        
        # Silver 테이블에 쓰기
        validated_df.write \
            .format("delta") \
            .mode("overwrite") \
            .option("overwriteSchema", "true") \
            .partitionBy("event_date") \
            .save(f"{self.silver_path}/cleaned_events")
        
        # 데이터 품질 메트릭 저장
        quality_metrics = {
            'total_records': bronze_df.count(),
            'valid_records': validated_df.count(),
            'invalid_records': bronze_df.count() - validated_df.count(),
            'validation_rate': validated_df.count() / bronze_df.count()
        }
        
        self.save_quality_metrics(quality_metrics)
    
    def aggregate_to_gold(self):
        """Gold 레이어로 집계"""
        # Silver 데이터 읽기
        silver_df = self.spark.read \
            .format("delta") \
            .load(f"{self.silver_path}/cleaned_events")
        
        # 일별 사용자 메트릭
        daily_user_metrics = silver_df \
            .groupBy("event_date", "user_id_clean") \
            .agg(
                F.count("event_id").alias("event_count"),
                F.sum("revenue_usd").alias("daily_revenue"),
                F.countDistinct("session_id").alias("session_count"),
                F.collect_set("event_type").alias("event_types"),
                F.min("timestamp").alias("first_event_time"),
                F.max("timestamp").alias("last_event_time")
            ) \
            .withColumn("active_hours", 
                       (F.col("last_event_time").cast("long") - 
                        F.col("first_event_time").cast("long")) / 3600)
        
        # 제품 분석
        product_analytics = silver_df \
            .filter(F.col("event_type").isin(["view", "purchase"])) \
            .groupBy("event_date", "product_id") \
            .pivot("event_type") \
            .agg(
                F.count("event_id").alias("count"),
                F.sum("revenue_usd").alias("revenue")
            ) \
            .withColumn("conversion_rate", 
                       F.col("purchase_count") / F.col("view_count"))
        
        # Gold 테이블 저장
        daily_user_metrics.write \
            .format("delta") \
            .mode("overwrite") \
            .save(f"{self.gold_path}/daily_user_metrics")
        
        product_analytics.write \
            .format("delta") \
            .mode("overwrite") \
            .save(f"{self.gold_path}/product_analytics")
    
    def setup_unity_catalog(self):
        """Unity Catalog 설정"""
        # 카탈로그 생성
        self.spark.sql("CREATE CATALOG IF NOT EXISTS analytics")
        self.spark.sql("USE CATALOG analytics")
        
        # 스키마 생성
        self.spark.sql("CREATE SCHEMA IF NOT EXISTS bronze")
        self.spark.sql("CREATE SCHEMA IF NOT EXISTS silver")
        self.spark.sql("CREATE SCHEMA IF NOT EXISTS gold")
        
        # 테이블 등록
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS bronze.raw_events
            USING DELTA
            LOCATION '{self.bronze_path}/streaming_events'
        """)
        
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS silver.cleaned_events
            USING DELTA
            LOCATION '{self.silver_path}/cleaned_events'
        """)
        
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS gold.daily_user_metrics
            USING DELTA
            LOCATION '{self.gold_path}/daily_user_metrics'
        """)
        
        # 데이터 계보 및 품질 메타데이터
        self.spark.sql("""
            COMMENT ON TABLE gold.daily_user_metrics IS 
            'Daily aggregated user metrics for business analytics'
        """)
        
        self.spark.sql("""
            ALTER TABLE gold.daily_user_metrics 
            SET TAGS ('quality' = 'high', 'sla' = '99.9%', 'owner' = 'analytics-team')
        """)
```

## 4. 실시간 머신러닝 파이프라인

### 4.1 Feature Store와 실시간 추론

```python
from feast import FeatureStore, Entity, Feature, FeatureView, FileSource
from feast.types import Float32, Int64, String
import mlflow
import pickle

class RealtimeMLPipeline:
    """실시간 머신러닝 파이프라인"""
    
    def __init__(self):
        self.feature_store = FeatureStore(repo_path="./feature_repo")
        self.model_registry = mlflow.tracking.MlflowClient()
        
    def create_feature_engineering_pipeline(self):
        """특징 엔지니어링 파이프라인"""
        # 실시간 특징 계산
        streaming_features = self.spark \
            .readStream \
            .format("delta") \
            .load("/silver/user_events") \
            .groupBy(
                F.window("event_time", "1 hour", "5 minutes"),
                "user_id"
            ) \
            .agg(
                F.count("event_id").alias("hourly_events"),
                F.sum("revenue").alias("hourly_revenue"),
                F.avg("session_duration").alias("avg_session_duration"),
                F.stddev("page_views").alias("page_view_variance"),
                F.collect_list("event_type").alias("event_sequence")
            ) \
            .select(
                F.col("window.start").alias("window_start"),
                F.col("window.end").alias("window_end"),
                F.col("user_id"),
                F.col("hourly_events"),
                F.col("hourly_revenue"),
                F.col("avg_session_duration"),
                F.col("page_view_variance"),
                F.size("event_sequence").alias("unique_event_types")
            )
        
        # Feature Store에 저장
        def write_to_feature_store(df, epoch_id):
            # Pandas 변환
            pandas_df = df.toPandas()
            
            # Feature Store 업데이트
            self.feature_store.write_to_online_store(
                feature_view_name="user_activity_features",
                df=pandas_df
            )
            
            # 오프라인 저장소에도 기록
            df.write \
                .format("delta") \
                .mode("append") \
                .save("/features/user_activity")
        
        # 스트리밍 시작
        query = streaming_features \
            .writeStream \
            .foreachBatch(write_to_feature_store) \
            .outputMode("update") \
            .trigger(processingTime="1 minute") \
            .start()
        
        return query
    
    def setup_online_inference(self):
        """온라인 추론 설정"""
        # 모델 로드
        model_name = "user_churn_prediction"
        model_version = self.model_registry.get_latest_versions(
            model_name, 
            stages=["Production"]
        )[0]
        
        model = mlflow.pyfunc.load_model(
            f"models:/{model_name}/{model_version.version}"
        )
        
        # 실시간 스코어링 함수
        def score_user(user_id):
            # Feature Store에서 특징 조회
            feature_vector = self.feature_store.get_online_features(
                features=[
                    "user_activity_features:hourly_events",
                    "user_activity_features:hourly_revenue",
                    "user_activity_features:avg_session_duration",
                    "user_profile:user_tenure_days",
                    "user_profile:total_purchases"
                ],
                entity_rows=[{"user_id": user_id}]
            ).to_dict()
            
            # 예측
            prediction = model.predict([feature_vector])
            
            # 결과 저장
            result = {
                'user_id': user_id,
                'churn_probability': float(prediction[0]),
                'prediction_time': datetime.now(),
                'model_version': model_version.version,
                'features': feature_vector
            }
            
            # 예측 로그 저장
            self.log_prediction(result)
            
            return result
        
        return score_user
    
    def create_ab_testing_framework(self):
        """A/B 테스팅 프레임워크"""
        class ABTestAnalyzer:
            def __init__(self, experiment_name):
                self.experiment_name = experiment_name
                self.control_model = "model_v1"
                self.treatment_model = "model_v2"
                
            def analyze_experiment_results(self):
                # 실험 데이터 읽기
                experiment_df = self.spark.read \
                    .format("delta") \
                    .load(f"/experiments/{self.experiment_name}")
                
                # 통계 분석
                results = experiment_df \
                    .groupBy("variant") \
                    .agg(
                        F.count("user_id").alias("users"),
                        F.avg("conversion").alias("conversion_rate"),
                        F.avg("revenue").alias("avg_revenue"),
                        F.stddev("revenue").alias("revenue_std"),
                        F.expr("percentile_approx(revenue, 0.5)").alias("median_revenue")
                    )
                
                # 통계적 유의성 검정
                from scipy import stats
                
                control_data = experiment_df \
                    .filter(F.col("variant") == "control") \
                    .select("revenue") \
                    .toPandas()["revenue"]
                
                treatment_data = experiment_df \
                    .filter(F.col("variant") == "treatment") \
                    .select("revenue") \
                    .toPandas()["revenue"]
                
                # T-검정
                t_stat, p_value = stats.ttest_ind(treatment_data, control_data)
                
                # 효과 크기 계산
                effect_size = (treatment_data.mean() - control_data.mean()) / control_data.std()
                
                return {
                    'summary_stats': results.toPandas(),
                    't_statistic': t_stat,
                    'p_value': p_value,
                    'effect_size': effect_size,
                    'significant': p_value < 0.05
                }
        
        return ABTestAnalyzer
```

## 마무리

빅데이터와 분석 데이터베이스는 현대 데이터 중심 의사결정의 핵심입니다. Apache Spark를 통한 대규모 데이터 처리, ClickHouse와 Druid 같은 분석 전용 데이터베이스, Delta Lake 기반의 레이크하우스 아키텍처, 그리고 실시간 머신러닝 파이프라인을 통해 데이터에서 가치 있는 인사이트를 추출할 수 있습니다. 이러한 기술들을 효과적으로 활용하면 비즈니스에 실질적인 가치를 제공할 수 있습니다.

다음 장에서는 그래프 데이터베이스와 지식 그래프에 대해 학습하겠습니다.