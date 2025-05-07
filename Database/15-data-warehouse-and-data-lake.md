# 데이터 웨어하우스와 데이터 레이크

## 개요

빅데이터 시대에 기업들은 방대한 양의 정형 및 비정형 데이터를 효과적으로 저장하고 분석해야 합니다. 데이터 웨어하우스는 구조화된 비즈니스 인텔리전스를 위한 전통적인 솔루션이며, 데이터 레이크는 모든 형태의 데이터를 원시 형태로 저장하는 현대적 접근법입니다. 이 장에서는 두 개념의 설계, 구현, 그리고 통합 방법을 학습합니다.

## 1. 데이터 웨어하우스 기초

### 1.1 차원 모델링 (Kimball 방법론)

```sql
-- 스타 스키마 설계 예제: 전자상거래 판매 분석

-- 1. 날짜 차원 테이블
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    date DATE NOT NULL,
    year INT NOT NULL,
    quarter INT NOT NULL,
    month INT NOT NULL,
    month_name VARCHAR(20) NOT NULL,
    week INT NOT NULL,
    day_of_month INT NOT NULL,
    day_of_week INT NOT NULL,
    day_name VARCHAR(20) NOT NULL,
    is_weekend BOOLEAN NOT NULL,
    is_holiday BOOLEAN DEFAULT FALSE,
    fiscal_year INT,
    fiscal_quarter INT,
    season VARCHAR(20),
    INDEX idx_date (date),
    INDEX idx_year_month (year, month)
);

-- 날짜 차원 데이터 생성 프로시저
DELIMITER //
CREATE PROCEDURE sp_populate_dim_date(
    IN start_date DATE,
    IN end_date DATE
)
BEGIN
    DECLARE current_date DATE;
    SET current_date = start_date;
    
    WHILE current_date <= end_date DO
        INSERT INTO dim_date (
            date_key,
            date,
            year,
            quarter,
            month,
            month_name,
            week,
            day_of_month,
            day_of_week,
            day_name,
            is_weekend,
            fiscal_year,
            fiscal_quarter,
            season
        ) VALUES (
            YEAR(current_date) * 10000 + MONTH(current_date) * 100 + DAY(current_date),
            current_date,
            YEAR(current_date),
            QUARTER(current_date),
            MONTH(current_date),
            MONTHNAME(current_date),
            WEEK(current_date),
            DAY(current_date),
            DAYOFWEEK(current_date),
            DAYNAME(current_date),
            CASE WHEN DAYOFWEEK(current_date) IN (1, 7) THEN TRUE ELSE FALSE END,
            CASE WHEN MONTH(current_date) >= 4 THEN YEAR(current_date) ELSE YEAR(current_date) - 1 END,
            CASE 
                WHEN MONTH(current_date) BETWEEN 4 AND 6 THEN 1
                WHEN MONTH(current_date) BETWEEN 7 AND 9 THEN 2
                WHEN MONTH(current_date) BETWEEN 10 AND 12 THEN 3
                ELSE 4
            END,
            CASE 
                WHEN MONTH(current_date) IN (12, 1, 2) THEN 'Winter'
                WHEN MONTH(current_date) IN (3, 4, 5) THEN 'Spring'
                WHEN MONTH(current_date) IN (6, 7, 8) THEN 'Summer'
                ELSE 'Fall'
            END
        );
        
        SET current_date = DATE_ADD(current_date, INTERVAL 1 DAY);
    END WHILE;
END//
DELIMITER ;

-- 2. 고객 차원 테이블
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY AUTO_INCREMENT,
    customer_id VARCHAR(50) NOT NULL,  -- 비즈니스 키
    customer_name VARCHAR(200) NOT NULL,
    email VARCHAR(200),
    phone VARCHAR(50),
    registration_date DATE,
    customer_segment VARCHAR(50),
    customer_lifetime_value DECIMAL(15, 2),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100),
    postal_code VARCHAR(20),
    -- SCD Type 2 필드
    effective_date DATE NOT NULL,
    expiration_date DATE DEFAULT '9999-12-31',
    is_current BOOLEAN DEFAULT TRUE,
    version INT DEFAULT 1,
    INDEX idx_customer_id (customer_id),
    INDEX idx_current (is_current)
);

-- 3. 제품 차원 테이블
CREATE TABLE dim_product (
    product_key INT PRIMARY KEY AUTO_INCREMENT,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    supplier VARCHAR(100),
    unit_cost DECIMAL(10, 2),
    unit_price DECIMAL(10, 2),
    -- SCD Type 1 (현재 값만 유지)
    created_date DATE,
    modified_date DATE,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_product_id (product_id),
    INDEX idx_category (category, subcategory)
);

-- 4. 판매 팩트 테이블
CREATE TABLE fact_sales (
    sales_key BIGINT PRIMARY KEY AUTO_INCREMENT,
    date_key INT NOT NULL,
    customer_key INT NOT NULL,
    product_key INT NOT NULL,
    store_key INT,
    promotion_key INT,
    order_number VARCHAR(50),
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    discount_amount DECIMAL(10, 2) DEFAULT 0,
    tax_amount DECIMAL(10, 2) DEFAULT 0,
    total_amount DECIMAL(10, 2) NOT NULL,
    cost_amount DECIMAL(10, 2),
    profit_amount DECIMAL(10, 2),
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    INDEX idx_date (date_key),
    INDEX idx_customer (customer_key),
    INDEX idx_product (product_key),
    INDEX idx_composite (date_key, customer_key, product_key)
) PARTITION BY RANGE (date_key) (
    PARTITION p_2023 VALUES LESS THAN (20240101),
    PARTITION p_2024 VALUES LESS THAN (20250101),
    PARTITION p_2025 VALUES LESS THAN (20260101),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- SCD Type 2 처리 프로시저
DELIMITER //
CREATE PROCEDURE sp_update_customer_scd2(
    IN p_customer_id VARCHAR(50),
    IN p_customer_name VARCHAR(200),
    IN p_email VARCHAR(200),
    IN p_city VARCHAR(100)
)
BEGIN
    DECLARE v_current_key INT;
    DECLARE v_has_changes BOOLEAN DEFAULT FALSE;
    
    -- 현재 레코드 확인
    SELECT customer_key INTO v_current_key
    FROM dim_customer
    WHERE customer_id = p_customer_id
    AND is_current = TRUE
    LIMIT 1;
    
    -- 변경사항 확인
    SELECT COUNT(*) > 0 INTO v_has_changes
    FROM dim_customer
    WHERE customer_key = v_current_key
    AND (customer_name != p_customer_name
         OR email != p_email
         OR city != p_city);
    
    IF v_has_changes THEN
        -- 기존 레코드 만료
        UPDATE dim_customer
        SET expiration_date = CURDATE() - INTERVAL 1 DAY,
            is_current = FALSE
        WHERE customer_key = v_current_key;
        
        -- 새 레코드 삽입
        INSERT INTO dim_customer (
            customer_id, customer_name, email, city,
            effective_date, is_current, version
        )
        SELECT 
            customer_id, p_customer_name, p_email, p_city,
            CURDATE(), TRUE, version + 1
        FROM dim_customer
        WHERE customer_key = v_current_key;
    END IF;
END//
DELIMITER ;
```

### 1.2 ETL 프로세스 구현

```python
import pandas as pd
import numpy as np
from sqlalchemy import create_engine
from datetime import datetime, timedelta
import logging

class DataWarehouseETL:
    """데이터 웨어하우스 ETL 프로세스"""
    
    def __init__(self, source_db_config, warehouse_db_config):
        self.source_engine = create_engine(source_db_config)
        self.warehouse_engine = create_engine(warehouse_db_config)
        self.logger = logging.getLogger(__name__)
        
    def extract_incremental_data(self, table_name, watermark_column, last_watermark):
        """증분 데이터 추출"""
        query = f"""
            SELECT * FROM {table_name}
            WHERE {watermark_column} > '{last_watermark}'
            ORDER BY {watermark_column}
        """
        
        self.logger.info(f"Extracting data from {table_name} after {last_watermark}")
        df = pd.read_sql(query, self.source_engine)
        
        return df
    
    def transform_sales_data(self, sales_df, customers_df, products_df):
        """판매 데이터 변환"""
        # 날짜 키 생성
        sales_df['date_key'] = pd.to_datetime(sales_df['order_date']).dt.strftime('%Y%m%d').astype(int)
        
        # 고객 키 조회
        sales_df = sales_df.merge(
            customers_df[['customer_id', 'customer_key']],
            on='customer_id',
            how='left'
        )
        
        # 제품 키 조회
        sales_df = sales_df.merge(
            products_df[['product_id', 'product_key']],
            on='product_id',
            how='left'
        )
        
        # 파생 측정값 계산
        sales_df['total_amount'] = (
            sales_df['quantity'] * sales_df['unit_price'] - 
            sales_df['discount_amount'] + 
            sales_df['tax_amount']
        )
        
        sales_df['profit_amount'] = (
            sales_df['total_amount'] - 
            sales_df['quantity'] * sales_df['unit_cost']
        )
        
        # 데이터 품질 검증
        invalid_records = sales_df[
            sales_df['customer_key'].isna() | 
            sales_df['product_key'].isna()
        ]
        
        if len(invalid_records) > 0:
            self.logger.warning(f"Found {len(invalid_records)} records with missing dimensions")
        
        # 필요한 컬럼만 선택
        fact_columns = [
            'date_key', 'customer_key', 'product_key', 'order_number',
            'quantity', 'unit_price', 'discount_amount', 'tax_amount',
            'total_amount', 'cost_amount', 'profit_amount'
        ]
        
        return sales_df[fact_columns].dropna()
    
    def load_fact_table(self, transformed_df, table_name):
        """팩트 테이블에 데이터 적재"""
        # 배치 크기로 분할하여 적재
        batch_size = 10000
        total_rows = len(transformed_df)
        
        for start_idx in range(0, total_rows, batch_size):
            end_idx = min(start_idx + batch_size, total_rows)
            batch_df = transformed_df.iloc[start_idx:end_idx]
            
            batch_df.to_sql(
                table_name,
                self.warehouse_engine,
                if_exists='append',
                index=False,
                method='multi'
            )
            
            self.logger.info(f"Loaded {end_idx}/{total_rows} rows")
    
    def update_aggregate_tables(self, date_key):
        """집계 테이블 업데이트"""
        # 일별 판매 집계
        daily_sales_query = f"""
            INSERT INTO agg_daily_sales (date_key, total_sales, total_quantity, unique_customers)
            SELECT 
                date_key,
                SUM(total_amount) as total_sales,
                SUM(quantity) as total_quantity,
                COUNT(DISTINCT customer_key) as unique_customers
            FROM fact_sales
            WHERE date_key = {date_key}
            GROUP BY date_key
            ON DUPLICATE KEY UPDATE
                total_sales = VALUES(total_sales),
                total_quantity = VALUES(total_quantity),
                unique_customers = VALUES(unique_customers)
        """
        
        self.warehouse_engine.execute(daily_sales_query)
        
        # 월별 제품 판매 집계
        monthly_product_query = f"""
            INSERT INTO agg_monthly_product_sales (year_month, product_key, total_sales, total_quantity)
            SELECT 
                FLOOR(date_key/100) as year_month,
                product_key,
                SUM(total_amount) as total_sales,
                SUM(quantity) as total_quantity
            FROM fact_sales
            WHERE date_key = {date_key}
            GROUP BY FLOOR(date_key/100), product_key
            ON DUPLICATE KEY UPDATE
                total_sales = total_sales + VALUES(total_sales),
                total_quantity = total_quantity + VALUES(total_quantity)
        """
        
        self.warehouse_engine.execute(monthly_product_query)

# OLAP 큐브 구현
class OLAPCube:
    """다차원 분석을 위한 OLAP 큐브"""
    
    def __init__(self, warehouse_engine):
        self.engine = warehouse_engine
        
    def build_sales_cube(self, dimensions, measures, filters=None):
        """판매 큐브 생성"""
        # 기본 쿼리 구성
        dim_columns = []
        join_clauses = []
        
        # 차원 처리
        dimension_mapping = {
            'date': ('dim_date d', 'f.date_key = d.date_key', ['d.year', 'd.month', 'd.day_name']),
            'customer': ('dim_customer c', 'f.customer_key = c.customer_key', ['c.customer_segment', 'c.city']),
            'product': ('dim_product p', 'f.product_key = p.product_key', ['p.category', 'p.subcategory', 'p.brand'])
        }
        
        for dim in dimensions:
            if dim in dimension_mapping:
                table, join, columns = dimension_mapping[dim]
                join_clauses.append(f"JOIN {table} ON {join}")
                dim_columns.extend(columns)
        
        # 측정값 처리
        measure_columns = []
        for measure in measures:
            if measure == 'sales':
                measure_columns.append('SUM(f.total_amount) as total_sales')
            elif measure == 'quantity':
                measure_columns.append('SUM(f.quantity) as total_quantity')
            elif measure == 'profit':
                measure_columns.append('SUM(f.profit_amount) as total_profit')
            elif measure == 'avg_order_value':
                measure_columns.append('AVG(f.total_amount) as avg_order_value')
        
        # 쿼리 생성
        query = f"""
            SELECT 
                {', '.join(dim_columns)},
                {', '.join(measure_columns)}
            FROM fact_sales f
            {' '.join(join_clauses)}
        """
        
        if filters:
            where_clauses = []
            for key, value in filters.items():
                where_clauses.append(f"{key} = '{value}'")
            query += f" WHERE {' AND '.join(where_clauses)}"
        
        query += f" GROUP BY {', '.join(dim_columns)}"
        
        # 결과를 DataFrame으로 반환
        return pd.read_sql(query, self.engine)
    
    def drill_down(self, cube_df, dimension, level):
        """드릴다운 작업"""
        # 차원 계층 정의
        dimension_hierarchy = {
            'date': ['year', 'quarter', 'month', 'day'],
            'customer': ['country', 'state', 'city', 'customer_name'],
            'product': ['category', 'subcategory', 'product_name']
        }
        
        if dimension not in dimension_hierarchy:
            raise ValueError(f"Unknown dimension: {dimension}")
        
        hierarchy = dimension_hierarchy[dimension]
        current_level_idx = hierarchy.index(level)
        
        if current_level_idx < len(hierarchy) - 1:
            next_level = hierarchy[current_level_idx + 1]
            # 다음 레벨로 그룹화
            return cube_df.groupby(hierarchy[:current_level_idx + 2]).sum().reset_index()
        
        return cube_df
    
    def slice_and_dice(self, cube_df, slice_conditions):
        """슬라이스 앤 다이스 작업"""
        result_df = cube_df.copy()
        
        for column, value in slice_conditions.items():
            if isinstance(value, list):
                result_df = result_df[result_df[column].isin(value)]
            else:
                result_df = result_df[result_df[column] == value]
        
        return result_df
```

## 2. 데이터 레이크 아키텍처

### 2.1 데이터 레이크 구현

```python
import boto3
import pyarrow as pa
import pyarrow.parquet as pq
from delta import DeltaTable
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
import json
from typing import Dict, List, Any

class DataLakeManager:
    """데이터 레이크 관리 시스템"""
    
    def __init__(self, lake_config):
        self.s3_client = boto3.client('s3')
        self.glue_client = boto3.client('glue')
        self.bucket_name = lake_config['bucket_name']
        self.spark = SparkSession.builder \
            .appName("DataLake") \
            .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
            .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
            .getOrCreate()
    
    def ingest_raw_data(self, source_type, source_config, target_path):
        """원시 데이터 수집"""
        if source_type == 'database':
            df = self.spark.read \
                .format("jdbc") \
                .option("url", source_config['jdbc_url']) \
                .option("dbtable", source_config['table']) \
                .option("user", source_config['user']) \
                .option("password", source_config['password']) \
                .load()
        
        elif source_type == 'streaming':
            df = self.spark \
                .readStream \
                .format("kafka") \
                .option("kafka.bootstrap.servers", source_config['brokers']) \
                .option("subscribe", source_config['topic']) \
                .load()
        
        elif source_type == 'files':
            df = self.spark.read \
                .format(source_config['format']) \
                .load(source_config['path'])
        
        # 메타데이터 추가
        df = df.withColumn("ingestion_timestamp", current_timestamp()) \
               .withColumn("source_system", lit(source_config.get('system_name', 'unknown'))) \
               .withColumn("data_version", lit(1))
        
        # Delta Lake 형식으로 저장
        if source_type == 'streaming':
            df.writeStream \
                .format("delta") \
                .outputMode("append") \
                .option("checkpointLocation", f"{target_path}/_checkpoint") \
                .start(target_path)
        else:
            df.write \
                .format("delta") \
                .mode("append") \
                .save(target_path)
        
        # 카탈로그 업데이트
        self._update_catalog(target_path, df.schema)
    
    def _update_catalog(self, path, schema):
        """데이터 카탈로그 업데이트 (AWS Glue)"""
        # 스키마를 Glue 형식으로 변환
        glue_columns = []
        for field in schema.fields:
            glue_columns.append({
                'Name': field.name,
                'Type': self._spark_to_glue_type(field.dataType),
                'Comment': field.metadata.get('comment', '')
            })
        
        # 테이블 정보 추출
        table_name = path.split('/')[-1]
        database_name = 'data_lake'
        
        try:
            # 테이블 업데이트 또는 생성
            self.glue_client.update_table(
                DatabaseName=database_name,
                TableInput={
                    'Name': table_name,
                    'StorageDescriptor': {
                        'Columns': glue_columns,
                        'Location': f's3://{self.bucket_name}/{path}',
                        'InputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
                        'OutputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat',
                        'SerdeInfo': {
                            'SerializationLibrary': 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
                        }
                    },
                    'Parameters': {
                        'classification': 'delta',
                        'typeOfData': 'file'
                    }
                }
            )
        except self.glue_client.exceptions.EntityNotFoundException:
            # 테이블이 없으면 생성
            self.glue_client.create_table(
                DatabaseName=database_name,
                TableInput={
                    'Name': table_name,
                    'StorageDescriptor': {
                        'Columns': glue_columns,
                        'Location': f's3://{self.bucket_name}/{path}',
                        'InputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
                        'OutputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
                    }
                }
            )
    
    def create_data_zones(self):
        """데이터 레이크 존 생성"""
        zones = {
            'raw': 'Raw ingested data',
            'cleansed': 'Cleaned and validated data',
            'curated': 'Business-ready datasets',
            'sandbox': 'Data science experimentation'
        }
        
        for zone, description in zones.items():
            # S3 버킷 구조 생성
            self.s3_client.put_object(
                Bucket=self.bucket_name,
                Key=f'{zone}/_zone_info.json',
                Body=json.dumps({
                    'zone': zone,
                    'description': description,
                    'created_at': datetime.utcnow().isoformat(),
                    'policies': self._get_zone_policies(zone)
                })
            )
    
    def _get_zone_policies(self, zone):
        """존별 데이터 정책 정의"""
        policies = {
            'raw': {
                'retention_days': 90,
                'compression': 'snappy',
                'encryption': 'SSE-S3',
                'access_control': 'restricted'
            },
            'cleansed': {
                'retention_days': 365,
                'compression': 'snappy',
                'encryption': 'SSE-KMS',
                'access_control': 'read-only'
            },
            'curated': {
                'retention_days': 730,
                'compression': 'gzip',
                'encryption': 'SSE-KMS',
                'access_control': 'governed'
            },
            'sandbox': {
                'retention_days': 30,
                'compression': 'none',
                'encryption': 'SSE-S3',
                'access_control': 'open'
            }
        }
        
        return policies.get(zone, {})

# 데이터 레이크 처리 파이프라인
class DataLakeProcessor:
    """데이터 레이크 처리 및 변환"""
    
    def __init__(self, spark_session, lake_path):
        self.spark = spark_session
        self.lake_path = lake_path
    
    def clean_and_validate(self, source_path, target_path, validation_rules):
        """데이터 정제 및 검증"""
        # Delta Lake 테이블 읽기
        df = self.spark.read.format("delta").load(source_path)
        
        # 데이터 정제
        cleaned_df = df
        
        # NULL 값 처리
        for col_name, default_value in validation_rules.get('null_defaults', {}).items():
            cleaned_df = cleaned_df.fillna({col_name: default_value})
        
        # 데이터 타입 변환
        for col_name, target_type in validation_rules.get('type_conversions', {}).items():
            cleaned_df = cleaned_df.withColumn(col_name, col(col_name).cast(target_type))
        
        # 이상치 제거
        for col_name, bounds in validation_rules.get('outlier_bounds', {}).items():
            if 'min' in bounds:
                cleaned_df = cleaned_df.filter(col(col_name) >= bounds['min'])
            if 'max' in bounds:
                cleaned_df = cleaned_df.filter(col(col_name) <= bounds['max'])
        
        # 중복 제거
        if validation_rules.get('remove_duplicates', False):
            cleaned_df = cleaned_df.dropDuplicates(validation_rules.get('duplicate_keys', []))
        
        # 데이터 품질 메트릭 계산
        quality_metrics = self._calculate_quality_metrics(df, cleaned_df)
        
        # 정제된 데이터 저장
        cleaned_df.write \
            .format("delta") \
            .mode("overwrite") \
            .option("overwriteSchema", "true") \
            .save(target_path)
        
        # 품질 메트릭 저장
        self._save_quality_metrics(target_path, quality_metrics)
        
        return cleaned_df
    
    def _calculate_quality_metrics(self, original_df, cleaned_df):
        """데이터 품질 메트릭 계산"""
        original_count = original_df.count()
        cleaned_count = cleaned_df.count()
        
        metrics = {
            'original_row_count': original_count,
            'cleaned_row_count': cleaned_count,
            'rows_removed': original_count - cleaned_count,
            'removal_percentage': (original_count - cleaned_count) / original_count * 100,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        # 컬럼별 NULL 비율
        null_percentages = {}
        for column in cleaned_df.columns:
            null_count = cleaned_df.filter(col(column).isNull()).count()
            null_percentages[column] = null_count / cleaned_count * 100
        
        metrics['null_percentages'] = null_percentages
        
        return metrics
    
    def create_curated_dataset(self, query, target_name, partitions=None):
        """큐레이션된 데이터셋 생성"""
        # SQL 쿼리 실행
        curated_df = self.spark.sql(query)
        
        # 파티셔닝 적용
        writer = curated_df.write.format("delta").mode("overwrite")
        
        if partitions:
            writer = writer.partitionBy(*partitions)
        
        # 최적화 힌트 적용
        writer = writer.option("optimizeWrite", "true") \
                      .option("autoCompact", "true")
        
        # 저장
        target_path = f"{self.lake_path}/curated/{target_name}"
        writer.save(target_path)
        
        # Z-Order 최적화 (쿼리 성능 향상)
        if partitions:
            delta_table = DeltaTable.forPath(self.spark, target_path)
            delta_table.optimize().executeZOrderBy(*partitions[:2])  # 최대 2개 컬럼
        
        return target_path
```

### 2.2 데이터 레이크하우스

```python
import databricks.koalas as ks
from delta.tables import DeltaTable
import mlflow
from pyspark.ml import Pipeline
from pyspark.ml.feature import VectorAssembler, StandardScaler
from pyspark.ml.classification import RandomForestClassifier

class DataLakehouse:
    """데이터 레이크하우스 구현"""
    
    def __init__(self, spark_session, storage_path):
        self.spark = spark_session
        self.storage_path = storage_path
        
        # Delta Lake 설정
        self.spark.sql(f"CREATE DATABASE IF NOT EXISTS lakehouse LOCATION '{storage_path}/lakehouse'")
        self.spark.sql("USE lakehouse")
    
    def create_bronze_table(self, source_path, table_name):
        """Bronze 레이어 - 원시 데이터"""
        # 원시 데이터 읽기
        df = self.spark.read \
            .format("json") \
            .option("multiline", "true") \
            .load(source_path)
        
        # 메타데이터 추가
        bronze_df = df.withColumn("ingestion_time", current_timestamp()) \
                     .withColumn("source_file", input_file_name()) \
                     .withColumn("processing_time", lit(None)) \
                     .withColumn("quality_check", lit("pending"))
        
        # Delta 테이블로 저장
        bronze_path = f"{self.storage_path}/lakehouse/bronze/{table_name}"
        bronze_df.write \
            .format("delta") \
            .mode("append") \
            .save(bronze_path)
        
        # 테이블 등록
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS bronze_{table_name}
            USING DELTA
            LOCATION '{bronze_path}'
        """)
        
        return bronze_path
    
    def create_silver_table(self, bronze_table, silver_table, transformation_logic):
        """Silver 레이어 - 정제된 데이터"""
        # Bronze 데이터 읽기
        bronze_df = self.spark.table(f"bronze_{bronze_table}")
        
        # 변환 로직 적용
        silver_df = transformation_logic(bronze_df)
        
        # 데이터 품질 검증
        quality_checks = self._run_quality_checks(silver_df)
        
        # 품질 검증 통과한 데이터만 저장
        silver_df_validated = silver_df.filter(col("quality_score") >= 0.8)
        
        # Silver 테이블 저장
        silver_path = f"{self.storage_path}/lakehouse/silver/{silver_table}"
        
        # Merge 작업 (업데이트 또는 삽입)
        if DeltaTable.isDeltaTable(self.spark, silver_path):
            delta_table = DeltaTable.forPath(self.spark, silver_path)
            
            delta_table.alias("target").merge(
                silver_df_validated.alias("source"),
                "target.id = source.id"
            ).whenMatchedUpdateAll() \
             .whenNotMatchedInsertAll() \
             .execute()
        else:
            silver_df_validated.write \
                .format("delta") \
                .mode("overwrite") \
                .save(silver_path)
        
        # 테이블 등록
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS silver_{silver_table}
            USING DELTA
            LOCATION '{silver_path}'
        """)
        
        # 변경 데이터 캡처 활성화
        self.spark.sql(f"""
            ALTER TABLE silver_{silver_table} 
            SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
        """)
        
        return silver_path
    
    def create_gold_table(self, silver_tables, gold_table, aggregation_logic):
        """Gold 레이어 - 비즈니스 레벨 데이터"""
        # Silver 테이블들 조인
        silver_dfs = {name: self.spark.table(f"silver_{name}") for name in silver_tables}
        
        # 집계 로직 적용
        gold_df = aggregation_logic(silver_dfs)
        
        # 성능 최적화를 위한 파티셔닝
        gold_path = f"{self.storage_path}/lakehouse/gold/{gold_table}"
        
        gold_df.write \
            .format("delta") \
            .mode("overwrite") \
            .partitionBy("year", "month") \
            .save(gold_path)
        
        # 테이블 등록
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS gold_{gold_table}
            USING DELTA
            LOCATION '{gold_path}'
        """)
        
        # 통계 정보 수집 (쿼리 최적화)
        self.spark.sql(f"ANALYZE TABLE gold_{gold_table} COMPUTE STATISTICS")
        
        return gold_path
    
    def _run_quality_checks(self, df):
        """데이터 품질 검증"""
        quality_scores = df
        
        # NULL 값 체크
        null_columns = []
        for column in df.columns:
            null_ratio = df.filter(col(column).isNull()).count() / df.count()
            if null_ratio > 0.1:  # 10% 이상 NULL
                null_columns.append(column)
        
        quality_scores = quality_scores.withColumn(
            "null_score",
            when(sum([col(c).isNull().cast("int") for c in null_columns]) > 0, 0).otherwise(1)
        )
        
        # 데이터 형식 검증
        quality_scores = quality_scores.withColumn(
            "format_score",
            when(col("email").rlike(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'), 1).otherwise(0)
        )
        
        # 전체 품질 점수
        quality_scores = quality_scores.withColumn(
            "quality_score",
            (col("null_score") + col("format_score")) / 2
        )
        
        return quality_scores
    
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
            # 변경 이력 조회
            return self.spark.sql(f"DESCRIBE HISTORY {table_name}")
    
    def implement_ml_pipeline(self, training_table, feature_columns, label_column):
        """머신러닝 파이프라인 구현"""
        # MLflow 실험 시작
        mlflow.set_experiment("lakehouse_ml_experiment")
        
        with mlflow.start_run():
            # 데이터 준비
            df = self.spark.table(training_table)
            
            # 특징 벡터 생성
            assembler = VectorAssembler(
                inputCols=feature_columns,
                outputCol="features"
            )
            
            # 정규화
            scaler = StandardScaler(
                inputCol="features",
                outputCol="scaled_features"
            )
            
            # 모델 정의
            rf = RandomForestClassifier(
                featuresCol="scaled_features",
                labelCol=label_column,
                numTrees=100
            )
            
            # 파이프라인 생성
            pipeline = Pipeline(stages=[assembler, scaler, rf])
            
            # 학습/테스트 분할
            train_df, test_df = df.randomSplit([0.8, 0.2], seed=42)
            
            # 모델 학습
            model = pipeline.fit(train_df)
            
            # 예측
            predictions = model.transform(test_df)
            
            # 모델 저장 (Delta Lake)
            model_path = f"{self.storage_path}/lakehouse/models/rf_model"
            model.write().overwrite().save(model_path)
            
            # MLflow에 모델 로깅
            mlflow.spark.log_model(model, "random_forest_model")
            
            # 메트릭 로깅
            from pyspark.ml.evaluation import MulticlassClassificationEvaluator
            evaluator = MulticlassClassificationEvaluator(
                labelCol=label_column,
                predictionCol="prediction",
                metricName="accuracy"
            )
            
            accuracy = evaluator.evaluate(predictions)
            mlflow.log_metric("accuracy", accuracy)
            
            return model
```

## 3. 하이브리드 아키텍처

### 3.1 데이터 웨어하우스와 데이터 레이크 통합

```python
class HybridDataPlatform:
    """하이브리드 데이터 플랫폼"""
    
    def __init__(self, warehouse_config, lake_config):
        self.warehouse = DataWarehouseManager(warehouse_config)
        self.lake = DataLakeManager(lake_config)
        self.spark = SparkSession.builder \
            .appName("HybridPlatform") \
            .config("spark.sql.warehouse.dir", warehouse_config['warehouse_dir']) \
            .enableHiveSupport() \
            .getOrCreate()
    
    def create_external_tables(self):
        """데이터 레이크를 외부 테이블로 연결"""
        # 데이터 레이크의 Delta 테이블을 웨어하우스에서 조회 가능하게 설정
        lake_tables = [
            ('customer_interactions', 'raw/customer_interactions'),
            ('product_reviews', 'curated/product_reviews'),
            ('social_media_mentions', 'curated/social_mentions')
        ]
        
        for table_name, lake_path in lake_tables:
            self.spark.sql(f"""
                CREATE TABLE IF NOT EXISTS ext_{table_name}
                USING DELTA
                LOCATION 's3://{self.lake.bucket_name}/{lake_path}'
            """)
    
    def federated_query(self, query):
        """웨어하우스와 레이크 데이터를 통합 쿼리"""
        # 임시 뷰 생성 - 웨어하우스 테이블
        warehouse_tables = ['dim_customer', 'dim_product', 'fact_sales']
        for table in warehouse_tables:
            df = self.spark.read \
                .format("jdbc") \
                .option("url", self.warehouse.jdbc_url) \
                .option("dbtable", table) \
                .load()
            df.createOrReplaceTempView(f"wh_{table}")
        
        # 통합 쿼리 실행
        result = self.spark.sql(query)
        
        return result
    
    def sync_aggregated_data(self):
        """레이크의 집계 데이터를 웨어하우스로 동기화"""
        # 실시간 스트리밍 집계
        streaming_agg = self.spark \
            .readStream \
            .format("delta") \
            .load(f"{self.lake.lake_path}/silver/events") \
            .groupBy(
                window("event_time", "1 hour"),
                "event_type",
                "user_segment"
            ) \
            .agg(
                count("*").alias("event_count"),
                avg("event_value").alias("avg_value")
            )
        
        # 웨어하우스에 스트리밍 쓰기
        streaming_agg.writeStream \
            .foreachBatch(self._write_to_warehouse) \
            .outputMode("update") \
            .trigger(processingTime='5 minutes') \
            .start()
    
    def _write_to_warehouse(self, batch_df, batch_id):
        """배치를 웨어하우스에 쓰기"""
        batch_df.write \
            .format("jdbc") \
            .option("url", self.warehouse.jdbc_url) \
            .option("dbtable", "streaming_aggregates") \
            .mode("append") \
            .save()
    
    def implement_data_mesh(self):
        """데이터 메시 아키텍처 구현"""
        domains = {
            'sales': {
                'owner': 'sales_team',
                'warehouse_tables': ['fact_sales', 'dim_customer'],
                'lake_datasets': ['customer_interactions', 'sales_predictions']
            },
            'marketing': {
                'owner': 'marketing_team',
                'warehouse_tables': ['fact_campaigns', 'dim_segments'],
                'lake_datasets': ['social_media', 'web_analytics']
            },
            'operations': {
                'owner': 'ops_team',
                'warehouse_tables': ['fact_inventory', 'dim_supplier'],
                'lake_datasets': ['iot_sensors', 'logistics_tracking']
            }
        }
        
        for domain_name, domain_config in domains.items():
            # 도메인별 데이터 제품 생성
            self._create_data_product(domain_name, domain_config)
    
    def _create_data_product(self, domain_name, config):
        """데이터 제품 생성"""
        # 도메인 스키마 생성
        self.spark.sql(f"CREATE SCHEMA IF NOT EXISTS {domain_name}")
        
        # 웨어하우스 테이블 뷰 생성
        for table in config['warehouse_tables']:
            self.spark.sql(f"""
                CREATE OR REPLACE VIEW {domain_name}.{table}_v AS
                SELECT * FROM wh_{table}
                WHERE domain = '{domain_name}'
            """)
        
        # 레이크 데이터셋 뷰 생성
        for dataset in config['lake_datasets']:
            self.spark.sql(f"""
                CREATE OR REPLACE VIEW {domain_name}.{dataset}_v AS
                SELECT * FROM ext_{dataset}
                WHERE domain = '{domain_name}'
            """)
        
        # 데이터 제품 메타데이터 저장
        metadata = {
            'domain': domain_name,
            'owner': config['owner'],
            'created_at': datetime.utcnow().isoformat(),
            'sla': {
                'freshness': '1 hour',
                'availability': '99.9%',
                'quality_score': 0.95
            },
            'schemas': {
                'warehouse': config['warehouse_tables'],
                'lake': config['lake_datasets']
            }
        }
        
        # 메타데이터를 카탈로그에 저장
        self._save_to_catalog(f"{domain_name}_product", metadata)

# 통합 모니터링 및 거버넌스
class DataGovernance:
    """데이터 거버넌스 및 모니터링"""
    
    def __init__(self, platform):
        self.platform = platform
        self.metrics_store = {}
    
    def implement_data_lineage(self):
        """데이터 리니지 추적"""
        # Apache Atlas 또는 DataHub 연동
        lineage_tracker = {
            'sources': [],
            'transformations': [],
            'targets': []
        }
        
        # Spark 리니지 리스너 등록
        class LineageListener(SparkListener):
            def onJobEnd(self, job_end):
                # 작업 완료 시 리니지 정보 수집
                lineage_info = {
                    'job_id': job_end.jobId,
                    'input_tables': self._extract_input_tables(job_end),
                    'output_tables': self._extract_output_tables(job_end),
                    'timestamp': datetime.utcnow().isoformat()
                }
                lineage_tracker['transformations'].append(lineage_info)
        
        self.platform.spark.sparkContext.addSparkListener(LineageListener())
    
    def monitor_data_quality(self):
        """데이터 품질 모니터링"""
        quality_rules = {
            'completeness': self._check_completeness,
            'accuracy': self._check_accuracy,
            'consistency': self._check_consistency,
            'timeliness': self._check_timeliness
        }
        
        quality_scores = {}
        
        for rule_name, rule_func in quality_rules.items():
            score = rule_func()
            quality_scores[rule_name] = score
            
            # 임계값 이하일 경우 알림
            if score < 0.9:
                self._send_quality_alert(rule_name, score)
        
        # 대시보드 업데이트
        self._update_quality_dashboard(quality_scores)
    
    def implement_access_control(self):
        """접근 제어 구현"""
        # Apache Ranger 연동
        policies = {
            'warehouse_read': {
                'resources': ['database:*', 'table:fact_*', 'column:*'],
                'users': ['analyst_group'],
                'permissions': ['select']
            },
            'lake_write': {
                'resources': ['s3://datalake/raw/*'],
                'users': ['etl_service'],
                'permissions': ['read', 'write']
            },
            'pii_access': {
                'resources': ['column:*email*', 'column:*ssn*', 'column:*phone*'],
                'users': ['privacy_team'],
                'permissions': ['select', 'update'],
                'conditions': ['purpose:gdpr_compliance']
            }
        }
        
        for policy_name, policy_config in policies.items():
            self._create_ranger_policy(policy_name, policy_config)
    
    def cost_optimization(self):
        """비용 최적화"""
        # 사용량 분석
        usage_metrics = {
            'warehouse': self._get_warehouse_usage(),
            'lake': self._get_lake_usage(),
            'compute': self._get_compute_usage()
        }
        
        # 비용 최적화 제안
        recommendations = []
        
        # 콜드 데이터 아카이빙
        cold_data = self._identify_cold_data()
        if cold_data:
            recommendations.append({
                'type': 'archive',
                'description': f'Archive {len(cold_data)} cold datasets to Glacier',
                'estimated_savings': self._calculate_archive_savings(cold_data)
            })
        
        # 파티션 정리
        unused_partitions = self._find_unused_partitions()
        if unused_partitions:
            recommendations.append({
                'type': 'cleanup',
                'description': f'Remove {len(unused_partitions)} unused partitions',
                'estimated_savings': self._calculate_partition_savings(unused_partitions)
            })
        
        return recommendations
```

## 4. 실제 구현 사례

### 4.1 실시간 분석 플랫폼

```python
class RealTimeAnalyticsPlatform:
    """실시간 분석 플랫폼 구현"""
    
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("RealTimeAnalytics") \
            .config("spark.sql.adaptive.enabled", "true") \
            .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
            .getOrCreate()
        
        self.kafka_params = {
            "kafka.bootstrap.servers": "localhost:9092",
            "subscribe": "events",
            "startingOffsets": "latest"
        }
    
    def create_streaming_pipeline(self):
        """스트리밍 파이프라인 생성"""
        # Kafka에서 실시간 이벤트 읽기
        raw_stream = self.spark \
            .readStream \
            .format("kafka") \
            .options(**self.kafka_params) \
            .load()
        
        # 이벤트 파싱
        parsed_stream = raw_stream \
            .select(
                from_json(col("value").cast("string"), self._get_event_schema()).alias("event"),
                col("timestamp")
            ) \
            .select("event.*", "timestamp")
        
        # 실시간 집계
        # 1. 세션 윈도우 집계
        session_aggregates = parsed_stream \
            .withWatermark("timestamp", "10 minutes") \
            .groupBy(
                session_window("timestamp", "30 minutes"),
                "user_id"
            ) \
            .agg(
                count("*").alias("event_count"),
                collect_list("event_type").alias("event_sequence"),
                min("timestamp").alias("session_start"),
                max("timestamp").alias("session_end")
            )
        
        # 2. 슬라이딩 윈도우 집계
        trending_events = parsed_stream \
            .withWatermark("timestamp", "1 minute") \
            .groupBy(
                window("timestamp", "5 minutes", "1 minute"),
                "event_type"
            ) \
            .agg(
                count("*").alias("event_count"),
                approx_count_distinct("user_id").alias("unique_users")
            ) \
            .select(
                col("window.start").alias("window_start"),
                col("window.end").alias("window_end"),
                col("event_type"),
                col("event_count"),
                col("unique_users"),
                (col("event_count") / col("unique_users")).alias("avg_events_per_user")
            )
        
        # 3. 실시간 이상 탐지
        anomaly_detection = self._create_anomaly_detection_stream(parsed_stream)
        
        # 멀티 싱크 출력
        # 데이터 레이크 (장기 저장)
        parsed_stream.writeStream \
            .format("delta") \
            .option("checkpointLocation", "/tmp/checkpoints/raw") \
            .outputMode("append") \
            .start("/data/lake/raw/events")
        
        # 데이터 웨어하우스 (집계 데이터)
        session_aggregates.writeStream \
            .foreachBatch(self._write_to_warehouse) \
            .outputMode("update") \
            .start()
        
        # 실시간 대시보드 (Redis)
        trending_events.writeStream \
            .foreachBatch(self._write_to_redis) \
            .outputMode("complete") \
            .start()
        
        # 알림 시스템 (이상 탐지)
        anomaly_detection.writeStream \
            .foreach(self._send_anomaly_alert) \
            .outputMode("append") \
            .start()
        
        # 모든 스트림 대기
        self.spark.streams.awaitAnyTermination()
    
    def _create_anomaly_detection_stream(self, stream):
        """실시간 이상 탐지 스트림"""
        # 기준선 통계 계산
        baseline_stats = stream \
            .groupBy("event_type", "user_segment") \
            .agg(
                avg("event_value").alias("avg_value"),
                stddev("event_value").alias("stddev_value")
            )
        
        # 실시간 이상 탐지
        anomalies = stream \
            .join(baseline_stats, ["event_type", "user_segment"]) \
            .withColumn(
                "z_score",
                abs(col("event_value") - col("avg_value")) / col("stddev_value")
            ) \
            .filter(col("z_score") > 3)  # 3 표준편차 이상
            .select(
                col("timestamp"),
                col("user_id"),
                col("event_type"),
                col("event_value"),
                col("z_score"),
                lit("statistical_anomaly").alias("anomaly_type")
            )
        
        return anomalies
    
    def _write_to_warehouse(self, batch_df, batch_id):
        """웨어하우스에 배치 쓰기"""
        # 임시 테이블 생성
        batch_df.createOrReplaceTempView("temp_session_data")
        
        # MERGE 작업
        merge_query = """
            MERGE INTO warehouse.session_aggregates AS target
            USING temp_session_data AS source
            ON target.user_id = source.user_id 
            AND target.session_start = source.session_start
            WHEN MATCHED THEN UPDATE SET
                event_count = source.event_count,
                event_sequence = source.event_sequence,
                session_end = source.session_end
            WHEN NOT MATCHED THEN INSERT *
        """
        
        self.spark.sql(merge_query)
    
    def _write_to_redis(self, batch_df, batch_id):
        """Redis에 실시간 메트릭 쓰기"""
        import redis
        r = redis.Redis(host='localhost', port=6379, decode_responses=True)
        
        # DataFrame을 딕셔너리로 변환
        metrics = batch_df.collect()
        
        for row in metrics:
            key = f"trending:{row['event_type']}:{row['window_start']}"
            value = {
                'event_count': row['event_count'],
                'unique_users': row['unique_users'],
                'avg_events_per_user': row['avg_events_per_user']
            }
            
            # Redis에 저장 (1시간 TTL)
            r.setex(key, 3600, json.dumps(value))
        
        # 전체 트렌드 업데이트
        r.zadd(
            "trending_events",
            {row['event_type']: row['event_count'] for row in metrics}
        )

# 사용 예제
if __name__ == "__main__":
    # 데이터 웨어하우스 설정
    warehouse_config = {
        'jdbc_url': 'jdbc:postgresql://localhost:5432/datawarehouse',
        'user': 'dw_user',
        'password': 'password'
    }
    
    # 데이터 레이크 설정
    lake_config = {
        'bucket_name': 'company-data-lake',
        'region': 'us-east-1'
    }
    
    # 하이브리드 플랫폼 초기화
    platform = HybridDataPlatform(warehouse_config, lake_config)
    
    # 실시간 분석 파이프라인 시작
    analytics = RealTimeAnalyticsPlatform()
    analytics.create_streaming_pipeline()
```

## 마무리

데이터 웨어하우스와 데이터 레이크는 각각의 장점을 가진 상호 보완적인 기술입니다. 데이터 웨어하우스는 구조화된 비즈니스 인텔리전스를 위한 최적의 솔루션이며, 데이터 레이크는 모든 형태의 데이터를 저장하고 탐색할 수 있는 유연성을 제공합니다. 최신 데이터 레이크하우스 아키텍처는 두 접근법의 장점을 결합하여 통합된 분석 플랫폼을 제공합니다. 다음 장에서는 클라우드 데이터베이스 서비스와 관리형 솔루션을 학습하겠습니다.