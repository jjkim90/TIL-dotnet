# 데이터베이스와 머신러닝 통합

## 개요

데이터베이스와 머신러닝의 통합은 데이터가 저장된 곳에서 직접 분석과 예측을 수행할 수 있게 해주는 강력한 패러다임입니다. 이 장에서는 인-데이터베이스 머신러닝, 피처 스토어, 모델 관리, 실시간 추론 시스템 구축 방법을 학습합니다.

## 1. 인-데이터베이스 머신러닝

### 1.1 SQL Server Machine Learning Services

```python
import pyodbc
import pandas as pd
import pickle
from sqlalchemy import create_engine

class SQLServerMLIntegration:
    """SQL Server 머신러닝 통합"""
    
    def __init__(self, connection_string):
        self.conn = pyodbc.connect(connection_string)
        self.engine = create_engine(f"mssql+pyodbc:///?odbc_connect={connection_string}")
        
    def enable_external_scripts(self):
        """외부 스크립트 실행 활성화"""
        cursor = self.conn.cursor()
        
        # sp_configure로 외부 스크립트 활성화
        cursor.execute("""
            EXEC sp_configure 'external scripts enabled', 1;
            RECONFIGURE WITH OVERRIDE;
        """)
        
        self.conn.commit()
        
    def train_model_in_database(self):
        """데이터베이스 내에서 모델 학습"""
        train_script = """
        EXECUTE sp_execute_external_script
            @language = N'Python',
            @script = N'
import pandas as pd
import pickle
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report

# 입력 데이터 받기
df = InputDataSet

# 특징과 레이블 분리
features = df.drop(["customer_id", "churn"], axis=1)
labels = df["churn"]

# 학습/테스트 분할
X_train, X_test, y_train, y_test = train_test_split(
    features, labels, test_size=0.2, random_state=42
)

# 모델 학습
model = RandomForestClassifier(
    n_estimators=100,
    max_depth=10,
    random_state=42
)
model.fit(X_train, y_train)

# 모델 평가
predictions = model.predict(X_test)
accuracy = accuracy_score(y_test, predictions)

# 모델 직렬화
model_bytes = pickle.dumps(model)

# 결과 반환
OutputDataSet = pd.DataFrame({
    "model_name": ["churn_prediction_rf"],
    "model_binary": [model_bytes],
    "accuracy": [accuracy],
    "feature_importance": [str(dict(zip(features.columns, model.feature_importances_)))]
})
',
            @input_data_1 = N'
                SELECT 
                    customer_id,
                    tenure_months,
                    monthly_charges,
                    total_charges,
                    number_of_services,
                    contract_type,
                    payment_method,
                    churn
                FROM customer_data
                WHERE training_set = 1
            ',
            @input_data_1_name = N'InputDataSet',
            @output_data_1_name = N'OutputDataSet'
        WITH RESULT SETS ((
            model_name NVARCHAR(100),
            model_binary VARBINARY(MAX),
            accuracy FLOAT,
            feature_importance NVARCHAR(MAX)
        ));
        """
        
        cursor = self.conn.cursor()
        cursor.execute(train_script)
        result = cursor.fetchone()
        
        # 모델 저장
        self.save_model_to_table(
            result.model_name,
            result.model_binary,
            result.accuracy,
            result.feature_importance
        )
        
        return result
    
    def save_model_to_table(self, model_name, model_binary, accuracy, feature_importance):
        """모델을 테이블에 저장"""
        cursor = self.conn.cursor()
        
        # 모델 테이블 생성
        cursor.execute("""
            IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'ml_models')
            CREATE TABLE ml_models (
                model_id INT IDENTITY(1,1) PRIMARY KEY,
                model_name NVARCHAR(100) NOT NULL,
                model_binary VARBINARY(MAX) NOT NULL,
                accuracy FLOAT,
                feature_importance NVARCHAR(MAX),
                created_date DATETIME DEFAULT GETDATE(),
                is_active BIT DEFAULT 1
            );
        """)
        
        # 모델 삽입
        cursor.execute("""
            INSERT INTO ml_models (model_name, model_binary, accuracy, feature_importance)
            VALUES (?, ?, ?, ?)
        """, model_name, model_binary, accuracy, feature_importance)
        
        self.conn.commit()
    
    def create_prediction_procedure(self):
        """예측 저장 프로시저 생성"""
        procedure_sql = """
        CREATE OR ALTER PROCEDURE sp_predict_churn
            @customer_id INT
        AS
        BEGIN
            DECLARE @model_binary VARBINARY(MAX);
            
            -- 활성 모델 가져오기
            SELECT TOP 1 @model_binary = model_binary
            FROM ml_models
            WHERE model_name = 'churn_prediction_rf' AND is_active = 1
            ORDER BY created_date DESC;
            
            -- 예측 실행
            EXEC sp_execute_external_script
                @language = N'Python',
                @script = N'
import pickle
import pandas as pd
import numpy as np

# 모델 역직렬화
model = pickle.loads(model_binary)

# 입력 데이터
df = InputDataSet

# 특징 준비
features = df.drop(["customer_id"], axis=1)

# 예측
predictions = model.predict(features)
probabilities = model.predict_proba(features)

# 결과 준비
OutputDataSet = pd.DataFrame({
    "customer_id": df["customer_id"],
    "prediction": predictions,
    "probability_no_churn": probabilities[:, 0],
    "probability_churn": probabilities[:, 1]
})
',
                @input_data_1 = N'
                    SELECT 
                        customer_id,
                        tenure_months,
                        monthly_charges,
                        total_charges,
                        number_of_services,
                        contract_type,
                        payment_method
                    FROM customer_data
                    WHERE customer_id = @customer_id
                ',
                @params = N'@model_binary VARBINARY(MAX), @customer_id INT',
                @model_binary = @model_binary,
                @customer_id = @customer_id,
                @input_data_1_name = N'InputDataSet',
                @output_data_1_name = N'OutputDataSet'
            WITH RESULT SETS ((
                customer_id INT,
                prediction INT,
                probability_no_churn FLOAT,
                probability_churn FLOAT
            ));
        END;
        """
        
        cursor = self.conn.cursor()
        cursor.execute(procedure_sql)
        self.conn.commit()
    
    def real_time_scoring_trigger(self):
        """실시간 스코어링 트리거"""
        trigger_sql = """
        CREATE OR ALTER TRIGGER trg_score_new_customer
        ON customer_data
        AFTER INSERT
        AS
        BEGIN
            -- 새로 삽입된 고객에 대해 예측 실행
            INSERT INTO customer_predictions (
                customer_id,
                prediction,
                probability_churn,
                prediction_date
            )
            SELECT 
                i.customer_id,
                p.prediction,
                p.probability_churn,
                GETDATE()
            FROM inserted i
            CROSS APPLY (
                SELECT prediction, probability_churn
                FROM OPENQUERY(
                    LOOPBACK,
                    'EXEC sp_predict_churn @customer_id = ' + CAST(i.customer_id AS VARCHAR)
                )
            ) p;
        END;
        """
        
        cursor = self.conn.cursor()
        cursor.execute(trigger_sql)
        self.conn.commit()
```

### 1.2 PostgreSQL MADlib

```python
import psycopg2
import pandas as pd
from sqlalchemy import create_engine

class PostgreSQLMADlib:
    """PostgreSQL MADlib 머신러닝"""
    
    def __init__(self, connection_string):
        self.conn = psycopg2.connect(connection_string)
        self.cursor = self.conn.cursor()
        
    def install_madlib(self):
        """MADlib 설치 및 초기화"""
        self.cursor.execute("CREATE EXTENSION IF NOT EXISTS madlib")
        self.conn.commit()
        
    def train_logistic_regression(self):
        """로지스틱 회귀 학습"""
        # 학습 데이터 준비
        self.cursor.execute("""
            DROP TABLE IF EXISTS customers_train;
            CREATE TABLE customers_train AS
            SELECT 
                customer_id,
                ARRAY[
                    tenure_months::FLOAT,
                    monthly_charges,
                    total_charges,
                    number_of_services::FLOAT,
                    CASE WHEN contract_type = 'Month-to-month' THEN 1 ELSE 0 END::FLOAT,
                    CASE WHEN contract_type = 'One year' THEN 1 ELSE 0 END::FLOAT,
                    CASE WHEN payment_method = 'Electronic check' THEN 1 ELSE 0 END::FLOAT
                ] AS features,
                churn::INT AS label
            FROM customer_data
            WHERE training_set = TRUE;
        """)
        
        # 로지스틱 회귀 학습
        self.cursor.execute("""
            SELECT madlib.logregr_train(
                'customers_train',      -- 소스 테이블
                'churn_model',          -- 모델 테이블
                'label',                -- 종속 변수
                'features',             -- 독립 변수
                NULL,                   -- 그룹화 컬럼
                20,                     -- 최대 반복
                'irls'                  -- 최적화 방법
            );
        """)
        
        # 모델 요약
        self.cursor.execute("""
            SELECT * FROM churn_model_summary;
        """)
        
        summary = self.cursor.fetchone()
        self.conn.commit()
        
        return summary
    
    def evaluate_model(self):
        """모델 평가"""
        # 예측 수행
        self.cursor.execute("""
            DROP TABLE IF EXISTS predictions;
            CREATE TABLE predictions AS
            SELECT 
                t.customer_id,
                t.label AS actual,
                madlib.logregr_predict(
                    m.coef, 
                    m.intercept, 
                    t.features
                ) AS predicted,
                madlib.logregr_predict_prob(
                    m.coef, 
                    m.intercept, 
                    t.features
                ) AS probability
            FROM customers_train t, churn_model m;
        """)
        
        # 평가 메트릭 계산
        self.cursor.execute("""
            SELECT 
                madlib.binary_classifier(
                    'predictions',
                    'evaluation_metrics',
                    'predicted',
                    'actual'
                );
        """)
        
        # 혼동 행렬
        self.cursor.execute("""
            SELECT 
                actual,
                predicted,
                COUNT(*) as count
            FROM predictions
            GROUP BY actual, predicted
            ORDER BY actual, predicted;
        """)
        
        confusion_matrix = self.cursor.fetchall()
        
        # ROC 곡선 데이터
        self.cursor.execute("""
            SELECT 
                threshold,
                tpr,
                fpr,
                precision,
                recall,
                f1_score
            FROM (
                SELECT 
                    madlib.binary_classifier_auc(
                        'predictions',
                        'roc_data',
                        'probability',
                        'actual'
                    )
            ) t;
        """)
        
        self.conn.commit()
        
        return {
            'confusion_matrix': confusion_matrix,
            'roc_data': self.cursor.fetchall()
        }
    
    def deploy_model_as_function(self):
        """모델을 함수로 배포"""
        self.cursor.execute("""
            CREATE OR REPLACE FUNCTION predict_churn(
                tenure_months INTEGER,
                monthly_charges NUMERIC,
                total_charges NUMERIC,
                number_of_services INTEGER,
                contract_type VARCHAR,
                payment_method VARCHAR
            ) RETURNS TABLE(
                prediction INTEGER,
                probability NUMERIC
            ) AS $$
            DECLARE
                features FLOAT[];
                coef FLOAT[];
                intercept FLOAT;
            BEGIN
                -- 모델 계수 가져오기
                SELECT m.coef, m.intercept 
                INTO coef, intercept
                FROM churn_model m;
                
                -- 특징 벡터 구성
                features := ARRAY[
                    tenure_months::FLOAT,
                    monthly_charges,
                    total_charges,
                    number_of_services::FLOAT,
                    CASE WHEN contract_type = 'Month-to-month' THEN 1 ELSE 0 END::FLOAT,
                    CASE WHEN contract_type = 'One year' THEN 1 ELSE 0 END::FLOAT,
                    CASE WHEN payment_method = 'Electronic check' THEN 1 ELSE 0 END::FLOAT
                ];
                
                -- 예측 반환
                RETURN QUERY
                SELECT 
                    madlib.logregr_predict(coef, intercept, features),
                    madlib.logregr_predict_prob(coef, intercept, features);
            END;
            $$ LANGUAGE plpgsql;
        """)
        
        self.conn.commit()
    
    def create_deep_learning_model(self):
        """심층 학습 모델 생성"""
        # 신경망 아키텍처 정의
        self.cursor.execute("""
            DROP TABLE IF EXISTS nn_architecture;
            CREATE TABLE nn_architecture (
                layer_id SERIAL,
                layer_type VARCHAR,
                layer_size INTEGER[],
                activation VARCHAR
            );
            
            INSERT INTO nn_architecture VALUES
                (1, 'input', ARRAY[7], NULL),
                (2, 'hidden', ARRAY[128], 'relu'),
                (3, 'hidden', ARRAY[64], 'relu'),
                (4, 'hidden', ARRAY[32], 'relu'),
                (5, 'output', ARRAY[2], 'softmax');
        """)
        
        # 신경망 학습
        self.cursor.execute("""
            SELECT madlib.mlp_train(
                'customers_train',      -- 소스 테이블
                'nn_model',             -- 모델 테이블  
                'features',             -- 독립 변수
                'label',                -- 종속 변수
                'nn_architecture',      -- 네트워크 구조
                'classification',       -- 모델 타입
                'learning_rate_init=0.001,
                 n_iterations=500,
                 n_tries=3,
                 optimizer=adam,
                 batch_size=32,
                 n_epochs=100',         -- 학습 파라미터
                'sigmoid',              -- 활성화 함수
                NULL,                   -- 가중치
                FALSE,                  -- 웜 스타트
                FALSE                   -- 상세 출력
            );
        """)
        
        self.conn.commit()
```

## 2. 피처 스토어 구현

### 2.1 실시간 피처 엔지니어링

```python
from feast import FeatureStore, Entity, Feature, FeatureView, Field
from feast.types import Float32, Int64, String
import redis
import pandas as pd
from datetime import datetime, timedelta

class MLFeatureStore:
    """머신러닝 피처 스토어"""
    
    def __init__(self):
        self.fs = FeatureStore(repo_path="feature_repo/")
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
        
    def create_feature_definitions(self):
        """피처 정의 생성"""
        # 엔티티 정의
        customer = Entity(
            name="customer",
            value_type=Int64,
            description="Customer ID"
        )
        
        # 피처 뷰 정의
        customer_features = FeatureView(
            name="customer_features",
            entities=["customer"],
            ttl=timedelta(days=1),
            features=[
                Feature(name="tenure_months", dtype=Int64),
                Feature(name="monthly_charges", dtype=Float32),
                Feature(name="total_charges", dtype=Float32),
                Feature(name="avg_monthly_usage", dtype=Float32),
                Feature(name="customer_lifetime_value", dtype=Float32),
                Feature(name="churn_risk_score", dtype=Float32)
            ],
            online=True,
            batch_source=FileSource(
                path="s3://feature-store/customer_features.parquet",
                event_timestamp_column="event_timestamp"
            ),
            tags={"team": "ml", "version": "1.0"}
        )
        
        # 실시간 피처 뷰
        realtime_features = FeatureView(
            name="realtime_customer_activity",
            entities=["customer"],
            ttl=timedelta(minutes=10),
            features=[
                Feature(name="last_activity_minutes", dtype=Int64),
                Feature(name="session_count_today", dtype=Int64),
                Feature(name="page_views_last_hour", dtype=Int64),
                Feature(name="current_session_duration", dtype=Float32),
                Feature(name="is_currently_active", dtype=Int64)
            ],
            online=True,
            stream_source=KafkaSource(
                bootstrap_servers="localhost:9092",
                topic="customer-activity",
                event_timestamp_column="timestamp",
                value_format="json"
            )
        )
        
        return [customer, customer_features, realtime_features]
    
    def compute_streaming_features(self, customer_id, event_data):
        """스트리밍 피처 계산"""
        # 실시간 활동 피처
        current_time = datetime.now()
        
        # Redis에서 세션 데이터 가져오기
        session_key = f"session:{customer_id}"
        session_data = self.redis_client.hgetall(session_key)
        
        # 피처 계산
        features = {
            'customer_id': customer_id,
            'event_timestamp': current_time,
            'last_activity_minutes': self._calculate_minutes_since_last_activity(
                customer_id, current_time
            ),
            'session_count_today': self._get_session_count_today(customer_id),
            'page_views_last_hour': self._get_page_views_last_hour(customer_id),
            'current_session_duration': self._get_current_session_duration(
                session_data, current_time
            ),
            'is_currently_active': 1 if session_data else 0
        }
        
        # 피처 저장
        self._store_streaming_features(features)
        
        return features
    
    def _calculate_minutes_since_last_activity(self, customer_id, current_time):
        """마지막 활동 이후 경과 시간"""
        last_activity = self.redis_client.get(f"last_activity:{customer_id}")
        if last_activity:
            last_time = datetime.fromisoformat(last_activity.decode())
            return int((current_time - last_time).total_seconds() / 60)
        return 999999  # 활동 없음
    
    def create_feature_pipeline(self):
        """피처 파이프라인 생성"""
        from pyspark.sql import SparkSession
        from pyspark.sql.functions import *
        from pyspark.sql.window import Window
        
        spark = SparkSession.builder \
            .appName("FeatureEngineering") \
            .config("spark.sql.adaptive.enabled", "true") \
            .getOrCreate()
        
        # 원시 데이터 읽기
        raw_events = spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "localhost:9092") \
            .option("subscribe", "customer-events") \
            .load()
        
        # 이벤트 파싱
        parsed_events = raw_events \
            .select(from_json(col("value").cast("string"), event_schema).alias("data")) \
            .select("data.*") \
            .withColumn("event_time", to_timestamp("timestamp"))
        
        # 윈도우 기반 피처 생성
        window_features = parsed_events \
            .withWatermark("event_time", "10 minutes") \
            .groupBy(
                window("event_time", "1 hour", "5 minutes"),
                "customer_id"
            ).agg(
                count("event_id").alias("event_count"),
                avg("session_duration").alias("avg_session_duration"),
                sum(when(col("event_type") == "purchase", 1).otherwise(0)).alias("purchase_count"),
                sum("revenue").alias("total_revenue"),
                collect_list("page_url").alias("pages_visited")
            )
        
        # 시퀀스 피처 생성
        sequence_features = parsed_events \
            .groupBy("customer_id", "session_id") \
            .agg(
                collect_list(struct("event_time", "event_type")).alias("event_sequence")
            ) \
            .select(
                "customer_id",
                expr("transform(event_sequence, x -> x.event_type)").alias("event_types"),
                size("event_sequence").alias("sequence_length")
            )
        
        # 피처 스토어에 쓰기
        query = window_features \
            .writeStream \
            .foreachBatch(lambda df, epoch_id: self.write_to_feature_store(df)) \
            .outputMode("append") \
            .trigger(processingTime="1 minute") \
            .start()
        
        return query
    
    def write_to_feature_store(self, df):
        """피처 스토어에 배치 쓰기"""
        # Pandas 변환
        pandas_df = df.toPandas()
        
        # 피처 스토어 업데이트
        self.fs.write_to_online_store(
            feature_view_name="realtime_customer_activity",
            df=pandas_df
        )
        
        # 오프라인 스토어에도 저장
        self.fs.write_to_offline_store(
            feature_view_name="realtime_customer_activity",
            df=pandas_df,
            data_source_name="s3://feature-store/realtime/"
        )
```

### 2.2 피처 서빙 레이어

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
from typing import List, Dict, Any
import asyncio

class FeatureServingAPI:
    """피처 서빙 API"""
    
    def __init__(self):
        self.app = FastAPI(title="Feature Store API")
        self.feature_store = MLFeatureStore()
        self.setup_routes()
        
    def setup_routes(self):
        """API 라우트 설정"""
        
        @self.app.post("/features/batch")
        async def get_batch_features(request: BatchFeatureRequest):
            """배치 피처 조회"""
            try:
                features = await self.get_historical_features(
                    entity_ids=request.entity_ids,
                    feature_names=request.feature_names,
                    timestamp=request.timestamp
                )
                return {"features": features}
            except Exception as e:
                raise HTTPException(status_code=500, detail=str(e))
        
        @self.app.post("/features/realtime")
        async def get_realtime_features(request: RealtimeFeatureRequest):
            """실시간 피처 조회"""
            try:
                features = await self.get_online_features(
                    entity_id=request.entity_id,
                    feature_names=request.feature_names
                )
                return {"features": features}
            except Exception as e:
                raise HTTPException(status_code=500, detail=str(e))
        
        @self.app.post("/features/compute")
        async def compute_on_demand_features(request: OnDemandFeatureRequest):
            """온디맨드 피처 계산"""
            try:
                features = await self.compute_features_on_demand(
                    entity_id=request.entity_id,
                    raw_data=request.raw_data,
                    feature_names=request.feature_names
                )
                return {"features": features}
            except Exception as e:
                raise HTTPException(status_code=500, detail=str(e))
    
    async def get_historical_features(self, entity_ids: List[int], 
                                    feature_names: List[str], 
                                    timestamp: str) -> Dict[str, Any]:
        """과거 피처 조회"""
        # 엔티티 데이터프레임 생성
        entity_df = pd.DataFrame({
            'customer': entity_ids,
            'event_timestamp': [timestamp] * len(entity_ids)
        })
        
        # 피처 스토어에서 조회
        historical_features = self.feature_store.fs.get_historical_features(
            entity_df=entity_df,
            features=feature_names
        ).to_df()
        
        return historical_features.to_dict('records')
    
    async def get_online_features(self, entity_id: int, 
                                feature_names: List[str]) -> Dict[str, Any]:
        """온라인 피처 조회"""
        # 캐시 확인
        cache_key = f"features:{entity_id}:{':'.join(sorted(feature_names))}"
        cached = await self.check_cache(cache_key)
        
        if cached:
            return cached
        
        # 피처 스토어에서 조회
        online_features = self.feature_store.fs.get_online_features(
            features=feature_names,
            entity_rows=[{"customer": entity_id}]
        ).to_dict()
        
        # 캐시 저장
        await self.save_to_cache(cache_key, online_features, ttl=60)
        
        return online_features
    
    async def compute_features_on_demand(self, entity_id: int,
                                       raw_data: Dict[str, Any],
                                       feature_names: List[str]) -> Dict[str, Any]:
        """온디맨드 피처 계산"""
        computed_features = {}
        
        # 피처별 계산 로직
        feature_computers = {
            'interaction_score': self._compute_interaction_score,
            'trend_indicator': self._compute_trend_indicator,
            'anomaly_score': self._compute_anomaly_score,
            'similarity_score': self._compute_similarity_score
        }
        
        # 병렬 계산
        tasks = []
        for feature_name in feature_names:
            if feature_name in feature_computers:
                task = feature_computers[feature_name](entity_id, raw_data)
                tasks.append(task)
        
        results = await asyncio.gather(*tasks)
        
        for feature_name, result in zip(feature_names, results):
            computed_features[feature_name] = result
        
        return computed_features
    
    async def _compute_interaction_score(self, entity_id: int, 
                                       raw_data: Dict[str, Any]) -> float:
        """상호작용 점수 계산"""
        # 가중치 기반 점수 계산
        weights = {
            'page_views': 0.2,
            'clicks': 0.3,
            'purchases': 0.5
        }
        
        score = sum(
            raw_data.get(key, 0) * weight 
            for key, weight in weights.items()
        )
        
        return float(np.clip(score / 100, 0, 1))

class BatchFeatureRequest(BaseModel):
    entity_ids: List[int]
    feature_names: List[str]
    timestamp: str

class RealtimeFeatureRequest(BaseModel):
    entity_id: int
    feature_names: List[str]

class OnDemandFeatureRequest(BaseModel):
    entity_id: int
    raw_data: Dict[str, Any]
    feature_names: List[str]
```

## 3. 모델 관리와 버전 관리

### 3.1 MLOps 파이프라인

```python
import mlflow
import mlflow.sklearn
from mlflow.tracking import MlflowClient
import joblib
from typing import Dict, List, Any
import hashlib

class ModelRegistry:
    """모델 레지스트리"""
    
    def __init__(self, tracking_uri="http://localhost:5000"):
        mlflow.set_tracking_uri(tracking_uri)
        self.client = MlflowClient()
        
    def register_model(self, model, model_name: str, metrics: Dict[str, float],
                      parameters: Dict[str, Any], tags: Dict[str, str] = None):
        """모델 등록"""
        with mlflow.start_run() as run:
            # 파라미터 로깅
            mlflow.log_params(parameters)
            
            # 메트릭 로깅
            mlflow.log_metrics(metrics)
            
            # 태그 추가
            if tags:
                mlflow.set_tags(tags)
            
            # 모델 저장
            mlflow.sklearn.log_model(
                model, 
                artifact_path="model",
                registered_model_name=model_name,
                signature=self._infer_signature(model)
            )
            
            # 추가 아티팩트
            self._log_model_artifacts(model, run.info.run_id)
            
            return run.info.run_id
    
    def promote_model(self, model_name: str, version: int, stage: str):
        """모델 스테이지 변경"""
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage=stage,  # "Staging", "Production", "Archived"
            archive_existing_versions=True
        )
        
        # 배포 이벤트 로깅
        self._log_deployment_event(model_name, version, stage)
    
    def get_production_model(self, model_name: str):
        """프로덕션 모델 조회"""
        latest_version = self.client.get_latest_versions(
            model_name,
            stages=["Production"]
        )[0]
        
        model_uri = f"models:/{model_name}/{latest_version.version}"
        model = mlflow.sklearn.load_model(model_uri)
        
        return model, latest_version
    
    def compare_models(self, model_name: str, version1: int, version2: int):
        """모델 비교"""
        # 모델 정보 조회
        model1 = self.client.get_model_version(model_name, version1)
        model2 = self.client.get_model_version(model_name, version2)
        
        # 메트릭 비교
        run1 = self.client.get_run(model1.run_id)
        run2 = self.client.get_run(model2.run_id)
        
        metrics1 = run1.data.metrics
        metrics2 = run2.data.metrics
        
        comparison = {
            'version1': {
                'version': version1,
                'metrics': metrics1,
                'created': model1.creation_timestamp
            },
            'version2': {
                'version': version2,
                'metrics': metrics2,
                'created': model2.creation_timestamp
            },
            'improvements': {
                metric: metrics2[metric] - metrics1.get(metric, 0)
                for metric in metrics2.keys()
            }
        }
        
        return comparison
    
    def setup_model_monitoring(self, model_name: str, version: int):
        """모델 모니터링 설정"""
        monitoring_config = {
            'model_name': model_name,
            'version': version,
            'metrics': {
                'prediction_drift': {
                    'threshold': 0.1,
                    'window': '1h',
                    'aggregation': 'mean'
                },
                'feature_drift': {
                    'threshold': 0.15,
                    'window': '1h',
                    'aggregation': 'max'
                },
                'performance_degradation': {
                    'threshold': -0.05,  # 5% 성능 저하
                    'window': '24h',
                    'baseline_metric': 'accuracy'
                }
            },
            'alerts': {
                'email': ['ml-team@company.com'],
                'slack': '#ml-alerts'
            }
        }
        
        # 모니터링 작업 생성
        self._create_monitoring_job(monitoring_config)
        
        return monitoring_config
```

### 3.2 A/B 테스팅 프레임워크

```python
class ModelABTesting:
    """모델 A/B 테스팅"""
    
    def __init__(self, database_connection):
        self.db = database_connection
        self.experiments = {}
        
    def create_experiment(self, experiment_name: str, 
                         model_a_config: Dict, 
                         model_b_config: Dict,
                         traffic_split: float = 0.5):
        """A/B 테스트 실험 생성"""
        experiment = {
            'name': experiment_name,
            'start_time': datetime.now(),
            'model_a': model_a_config,
            'model_b': model_b_config,
            'traffic_split': traffic_split,
            'status': 'active',
            'results': {
                'model_a': {'predictions': 0, 'successes': 0},
                'model_b': {'predictions': 0, 'successes': 0}
            }
        }
        
        self.experiments[experiment_name] = experiment
        
        # 데이터베이스에 실험 저장
        self._save_experiment_to_db(experiment)
        
        return experiment
    
    def route_prediction(self, experiment_name: str, entity_id: int):
        """예측 라우팅"""
        experiment = self.experiments[experiment_name]
        
        # 결정적 라우팅 (같은 엔티티는 항상 같은 모델로)
        hash_value = int(hashlib.md5(str(entity_id).encode()).hexdigest(), 16)
        use_model_a = (hash_value % 100) < (experiment['traffic_split'] * 100)
        
        selected_model = 'model_a' if use_model_a else 'model_b'
        model_config = experiment[selected_model]
        
        return selected_model, model_config
    
    def record_prediction_result(self, experiment_name: str, 
                               model_variant: str,
                               entity_id: int,
                               prediction: Any,
                               actual_outcome: Any = None):
        """예측 결과 기록"""
        experiment = self.experiments[experiment_name]
        
        # 예측 기록
        prediction_record = {
            'experiment': experiment_name,
            'variant': model_variant,
            'entity_id': entity_id,
            'prediction': prediction,
            'timestamp': datetime.now(),
            'outcome': actual_outcome
        }
        
        # 데이터베이스에 저장
        self._save_prediction_to_db(prediction_record)
        
        # 실시간 통계 업데이트
        experiment['results'][model_variant]['predictions'] += 1
        
        if actual_outcome is not None:
            if self._is_successful_prediction(prediction, actual_outcome):
                experiment['results'][model_variant]['successes'] += 1
    
    def analyze_experiment(self, experiment_name: str):
        """실험 분석"""
        experiment = self.experiments[experiment_name]
        
        # 통계적 유의성 검정
        from scipy import stats
        
        # 각 모델의 성공률
        a_success = experiment['results']['model_a']['successes']
        a_total = experiment['results']['model_a']['predictions']
        b_success = experiment['results']['model_b']['successes']
        b_total = experiment['results']['model_b']['predictions']
        
        # 비율 검정
        if a_total > 30 and b_total > 30:  # 충분한 샘플
            # Z-검정
            p_a = a_success / a_total if a_total > 0 else 0
            p_b = b_success / b_total if b_total > 0 else 0
            p_pooled = (a_success + b_success) / (a_total + b_total)
            
            se = np.sqrt(p_pooled * (1 - p_pooled) * (1/a_total + 1/b_total))
            z_score = (p_a - p_b) / se if se > 0 else 0
            p_value = 2 * (1 - stats.norm.cdf(abs(z_score)))
            
            # 신뢰구간
            ci_width = 1.96 * se
            ci_lower = (p_a - p_b) - ci_width
            ci_upper = (p_a - p_b) + ci_width
            
            analysis = {
                'model_a_success_rate': p_a,
                'model_b_success_rate': p_b,
                'difference': p_a - p_b,
                'z_score': z_score,
                'p_value': p_value,
                'confidence_interval': (ci_lower, ci_upper),
                'significant': p_value < 0.05,
                'winner': 'model_a' if p_a > p_b else 'model_b',
                'sample_sizes': {
                    'model_a': a_total,
                    'model_b': b_total
                }
            }
        else:
            analysis = {
                'status': 'insufficient_data',
                'model_a_samples': a_total,
                'model_b_samples': b_total,
                'required_samples': 30
            }
        
        return analysis
```

## 4. 실시간 추론 시스템

### 4.1 스트리밍 추론 파이프라인

```python
from kafka import KafkaConsumer, KafkaProducer
import tensorflow as tf
import torch
import onnxruntime as rt
from concurrent.futures import ThreadPoolExecutor
import asyncio

class RealtimeInferencePipeline:
    """실시간 추론 파이프라인"""
    
    def __init__(self):
        self.models = {}
        self.preprocessors = {}
        self.postprocessors = {}
        self.executor = ThreadPoolExecutor(max_workers=10)
        
    def load_models(self):
        """모델 로드"""
        # TensorFlow 모델
        self.models['tf_model'] = tf.keras.models.load_model('models/tf_model.h5')
        
        # PyTorch 모델
        self.models['torch_model'] = torch.jit.load('models/torch_model.pt')
        self.models['torch_model'].eval()
        
        # ONNX 모델
        self.models['onnx_model'] = rt.InferenceSession('models/model.onnx')
        
        # 모델 워밍업
        self._warmup_models()
    
    def _warmup_models(self):
        """모델 워밍업"""
        # 더미 입력으로 초기 추론 실행
        dummy_input = np.random.randn(1, 10).astype(np.float32)
        
        # TensorFlow
        _ = self.models['tf_model'].predict(dummy_input)
        
        # PyTorch
        with torch.no_grad():
            _ = self.models['torch_model'](torch.tensor(dummy_input))
        
        # ONNX
        _ = self.models['onnx_model'].run(None, {'input': dummy_input})
    
    async def process_stream(self):
        """스트림 처리"""
        consumer = KafkaConsumer(
            'inference-requests',
            bootstrap_servers=['localhost:9092'],
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
        
        producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        
        batch = []
        batch_size = 32
        batch_timeout = 0.1  # 100ms
        last_batch_time = time.time()
        
        for message in consumer:
            batch.append(message.value)
            
            # 배치 처리 조건
            if len(batch) >= batch_size or (time.time() - last_batch_time) > batch_timeout:
                # 비동기 배치 추론
                results = await self.batch_inference(batch)
                
                # 결과 전송
                for result in results:
                    producer.send('inference-results', value=result)
                
                batch = []
                last_batch_time = time.time()
    
    async def batch_inference(self, requests):
        """배치 추론"""
        # 요청별 모델 그룹화
        model_groups = {}
        for req in requests:
            model_name = req['model']
            if model_name not in model_groups:
                model_groups[model_name] = []
            model_groups[model_name].append(req)
        
        # 병렬 추론
        tasks = []
        for model_name, group_requests in model_groups.items():
            task = self._run_model_inference(model_name, group_requests)
            tasks.append(task)
        
        results = await asyncio.gather(*tasks)
        
        # 결과 평탄화
        flat_results = []
        for group_result in results:
            flat_results.extend(group_result)
        
        return flat_results
    
    async def _run_model_inference(self, model_name, requests):
        """모델별 추론 실행"""
        # 전처리
        preprocessed = await self._preprocess_batch(model_name, requests)
        
        # 추론
        if 'tf' in model_name:
            predictions = await self._tensorflow_inference(
                self.models[model_name], 
                preprocessed
            )
        elif 'torch' in model_name:
            predictions = await self._pytorch_inference(
                self.models[model_name], 
                preprocessed
            )
        elif 'onnx' in model_name:
            predictions = await self._onnx_inference(
                self.models[model_name], 
                preprocessed
            )
        
        # 후처리
        results = await self._postprocess_batch(model_name, predictions, requests)
        
        return results
    
    async def _tensorflow_inference(self, model, inputs):
        """TensorFlow 추론"""
        loop = asyncio.get_event_loop()
        
        def predict():
            return model.predict(inputs, batch_size=len(inputs))
        
        predictions = await loop.run_in_executor(self.executor, predict)
        return predictions
    
    async def _pytorch_inference(self, model, inputs):
        """PyTorch 추론"""
        loop = asyncio.get_event_loop()
        
        def predict():
            with torch.no_grad():
                tensor_inputs = torch.tensor(inputs)
                outputs = model(tensor_inputs)
                return outputs.numpy()
        
        predictions = await loop.run_in_executor(self.executor, predict)
        return predictions
```

### 4.2 모델 서빙 최적화

```python
class OptimizedModelServing:
    """최적화된 모델 서빙"""
    
    def __init__(self):
        self.model_cache = {}
        self.request_queue = asyncio.Queue()
        self.result_cache = TTLCache(maxsize=10000, ttl=300)  # 5분 캐시
        
    def optimize_model_for_inference(self, model, optimization_level='O2'):
        """추론을 위한 모델 최적화"""
        if isinstance(model, torch.nn.Module):
            # PyTorch 최적화
            # 1. 양자화
            quantized_model = torch.quantization.quantize_dynamic(
                model, 
                {torch.nn.Linear, torch.nn.Conv2d}, 
                dtype=torch.qint8
            )
            
            # 2. TorchScript 변환
            example_input = torch.randn(1, *model.input_shape)
            traced_model = torch.jit.trace(quantized_model, example_input)
            
            # 3. 최적화
            optimized_model = torch.jit.optimize_for_inference(traced_model)
            
            return optimized_model
            
        elif hasattr(model, 'save'):  # TensorFlow/Keras
            # TensorFlow Lite 변환
            converter = tf.lite.TFLiteConverter.from_keras_model(model)
            converter.optimizations = [tf.lite.Optimize.DEFAULT]
            converter.representative_dataset = self._representative_dataset
            converter.target_spec.supported_ops = [
                tf.lite.OpsSet.TFLITE_BUILTINS_INT8
            ]
            
            tflite_model = converter.convert()
            
            # TensorRT 최적화 (NVIDIA GPU)
            if tf.config.list_physical_devices('GPU'):
                from tensorflow.python.compiler.tensorrt import trt_convert as trt
                
                converter = trt.TrtGraphConverterV2(
                    input_saved_model_dir=model_path,
                    precision_mode=trt.TrtPrecisionMode.FP16
                )
                trt_model = converter.convert()
                
                return trt_model
            
            return tflite_model
    
    async def adaptive_batching(self):
        """적응적 배칭"""
        batch = []
        batch_size = 1
        max_batch_size = 128
        timeout = 0.01  # 10ms
        
        while True:
            try:
                # 동적 타임아웃
                adaptive_timeout = timeout * (1 + len(batch) / max_batch_size)
                
                request = await asyncio.wait_for(
                    self.request_queue.get(), 
                    timeout=adaptive_timeout
                )
                batch.append(request)
                
                # 동적 배치 크기 조정
                if len(batch) >= batch_size:
                    # 처리 및 지연 시간 측정
                    start_time = time.time()
                    await self.process_batch(batch)
                    process_time = time.time() - start_time
                    
                    # 배치 크기 조정
                    if process_time < 0.05:  # 50ms 미만
                        batch_size = min(batch_size * 2, max_batch_size)
                    elif process_time > 0.1:  # 100ms 초과
                        batch_size = max(batch_size // 2, 1)
                    
                    batch = []
                    
            except asyncio.TimeoutError:
                if batch:
                    await self.process_batch(batch)
                    batch = []
    
    def setup_gpu_inference(self):
        """GPU 추론 설정"""
        # CUDA 스트림 생성
        self.cuda_streams = [
            torch.cuda.Stream() for _ in range(4)
        ]
        
        # 멀티 GPU 설정
        if torch.cuda.device_count() > 1:
            self.model = torch.nn.DataParallel(self.model)
        
        # Mixed Precision 설정
        self.scaler = torch.cuda.amp.GradScaler()
        
        # GPU 메모리 최적화
        torch.cuda.empty_cache()
        torch.backends.cudnn.benchmark = True
        
    async def streaming_inference(self, input_stream):
        """스트리밍 추론"""
        # 슬라이딩 윈도우
        window_size = 100
        window = deque(maxlen=window_size)
        
        async for data in input_stream:
            window.append(data)
            
            if len(window) == window_size:
                # 윈도우 데이터로 추론
                window_data = np.array(window)
                
                # 캐시 확인
                cache_key = hashlib.md5(window_data.tobytes()).hexdigest()
                if cache_key in self.result_cache:
                    yield self.result_cache[cache_key]
                    continue
                
                # 추론 실행
                result = await self.run_inference(window_data)
                
                # 캐시 저장
                self.result_cache[cache_key] = result
                
                yield result
```

## 5. 자동화된 ML 파이프라인

### 5.1 AutoML 통합

```python
class DatabaseAutoML:
    """데이터베이스 통합 AutoML"""
    
    def __init__(self, db_connection):
        self.db = db_connection
        self.auto_sklearn = None
        self.tpot = None
        
    def automated_feature_engineering(self, table_name, target_column):
        """자동 피처 엔지니어링"""
        # Featuretools를 사용한 자동 피처 생성
        import featuretools as ft
        
        # 엔티티 세트 생성
        es = ft.EntitySet()
        
        # 데이터 로드
        df = pd.read_sql(f"SELECT * FROM {table_name}", self.db)
        
        # 엔티티 추가
        es.entity_from_dataframe(
            entity_id=table_name,
            dataframe=df,
            index='id',
            time_index='timestamp'
        )
        
        # 자동 피처 생성
        feature_matrix, feature_defs = ft.dfs(
            entityset=es,
            target_entity=table_name,
            agg_primitives=['sum', 'mean', 'std', 'max', 'min', 'count'],
            trans_primitives=['day', 'month', 'year', 'weekday', 'is_weekend'],
            max_depth=2
        )
        
        # 피처 중요도 기반 선택
        selected_features = self.select_important_features(
            feature_matrix, 
            df[target_column]
        )
        
        return selected_features, feature_defs
    
    def run_automl_experiment(self, features, target, time_limit=3600):
        """AutoML 실험 실행"""
        from autosklearn.classification import AutoSklearnClassifier
        from autosklearn.metrics import roc_auc
        
        # Auto-sklearn 설정
        automl = AutoSklearnClassifier(
            time_left_for_this_task=time_limit,
            per_run_time_limit=300,
            ensemble_size=50,
            ensemble_nbest=50,
            metric=roc_auc,
            include_estimators=['random_forest', 'gradient_boosting', 
                              'extra_trees', 'adaboost', 'mlp'],
            include_preprocessors=['no_preprocessing', 'select_percentile',
                                 'pca', 'truncatedSVD'],
            resampling_strategy='cv',
            resampling_strategy_arguments={'folds': 5}
        )
        
        # 모델 학습
        automl.fit(features.copy(), target.copy())
        
        # 결과 분석
        results = {
            'best_model': automl.show_models(),
            'cv_results': automl.cv_results_,
            'leaderboard': automl.leaderboard(),
            'ensemble_performance': automl.score(features, target)
        }
        
        # 최적 모델 저장
        self.save_automl_model(automl, results)
        
        return automl, results
    
    def continuous_learning_pipeline(self):
        """지속적 학습 파이프라인"""
        # 새 데이터 모니터링
        data_monitor_query = """
        CREATE OR REPLACE FUNCTION monitor_new_data()
        RETURNS TRIGGER AS $$
        BEGIN
            -- 새 데이터 통계
            INSERT INTO data_statistics (
                timestamp,
                table_name,
                row_count,
                feature_stats
            )
            SELECT 
                NOW(),
                TG_TABLE_NAME,
                COUNT(*),
                jsonb_build_object(
                    'mean', AVG(NEW.value),
                    'std', STDDEV(NEW.value),
                    'min', MIN(NEW.value),
                    'max', MAX(NEW.value)
                )
            FROM NEW;
            
            -- 재학습 트리거 확인
            IF (SELECT COUNT(*) FROM NEW) > 1000 THEN
                PERFORM pg_notify('retrain_model', 
                    json_build_object(
                        'table', TG_TABLE_NAME,
                        'count', COUNT(*)
                    )::text
                );
            END IF;
            
            RETURN NULL;
        END;
        $$ LANGUAGE plpgsql;
        
        CREATE TRIGGER monitor_training_data
        AFTER INSERT ON training_data
        REFERENCING NEW TABLE AS NEW
        FOR EACH STATEMENT
        EXECUTE FUNCTION monitor_new_data();
        """
        
        cursor = self.db.cursor()
        cursor.execute(data_monitor_query)
        self.db.commit()
        
        # 재학습 리스너
        self.listen_for_retraining()
    
    def listen_for_retraining(self):
        """재학습 이벤트 리스닝"""
        import select
        
        cursor = self.db.cursor()
        cursor.execute("LISTEN retrain_model;")
        
        while True:
            if select.select([self.db], [], [], 5) == ([], [], []):
                continue
            else:
                self.db.poll()
                while self.db.notifies:
                    notify = self.db.notifies.pop(0)
                    payload = json.loads(notify.payload)
                    
                    # 재학습 실행
                    if payload['count'] > 1000:
                        self.trigger_retraining(payload['table'])
```

## 마무리

데이터베이스와 머신러닝의 통합은 데이터가 있는 곳에서 직접 인사이트를 도출할 수 있게 해주는 강력한 패러다임입니다. 인-데이터베이스 머신러닝, 피처 스토어, 실시간 추론 시스템을 통해 데이터 이동을 최소화하고 지연 시간을 줄일 수 있습니다. MLOps 파이프라인과 자동화된 모델 관리를 통해 머신러닝 모델의 전체 생명주기를 효과적으로 관리할 수 있습니다.

다음 장에서는 데이터베이스 아키텍처의 미래 트렌드에 대해 학습하겠습니다.