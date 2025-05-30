# 시계열 데이터베이스와 IoT 데이터 처리

## 개요

시계열 데이터베이스는 시간 기반 데이터를 효율적으로 저장하고 쿼리하도록 최적화된 데이터베이스입니다. IoT 센서 데이터, 금융 거래, 시스템 메트릭, 날씨 데이터 등 시간에 따라 변화하는 대량의 데이터를 처리하는 데 필수적입니다. 이 장에서는 주요 시계열 데이터베이스와 IoT 데이터 처리 아키텍처를 학습합니다.

## 1. InfluxDB 시계열 데이터베이스

### 1.1 InfluxDB 기본 구조와 데이터 모델

```python
from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS
from datetime import datetime, timedelta
import pandas as pd
import numpy as np

class InfluxDBTimeSeriesDB:
    """InfluxDB 시계열 데이터베이스 관리"""
    
    def __init__(self, url="http://localhost:8086", token="your-token", org="your-org"):
        self.client = InfluxDBClient(url=url, token=token, org=org)
        self.write_api = self.client.write_api(write_options=SYNCHRONOUS)
        self.query_api = self.client.query_api()
        self.bucket = "iot_data"
        self.org = org
        
    def create_bucket_with_retention(self, bucket_name, retention_days=30):
        """보존 정책이 있는 버킷 생성"""
        buckets_api = self.client.buckets_api()
        
        retention_rules = [{
            "type": "expire",
            "everySeconds": retention_days * 24 * 60 * 60,
            "shardGroupDurationSeconds": 24 * 60 * 60  # 1일 샤드
        }]
        
        bucket = buckets_api.create_bucket(
            bucket_name=bucket_name,
            retention_rules=retention_rules,
            org=self.org
        )
        
        return bucket
    
    def write_sensor_data(self, sensor_data):
        """센서 데이터 쓰기"""
        points = []
        
        for data in sensor_data:
            point = Point("sensor_reading") \
                .tag("sensor_id", data['sensor_id']) \
                .tag("location", data['location']) \
                .tag("sensor_type", data['sensor_type']) \
                .field("temperature", data['temperature']) \
                .field("humidity", data['humidity']) \
                .field("pressure", data['pressure']) \
                .field("battery_level", data['battery_level']) \
                .time(data['timestamp'], WritePrecision.NS)
            
            points.append(point)
        
        # 배치 쓰기
        self.write_api.write(bucket=self.bucket, org=self.org, record=points)
    
    def query_time_range(self, start_time, end_time, sensor_id=None):
        """시간 범위 쿼리"""
        filter_clause = f'|> filter(fn: (r) => r["sensor_id"] == "{sensor_id}")' if sensor_id else ""
        
        query = f'''
        from(bucket: "{self.bucket}")
            |> range(start: {start_time}, stop: {end_time})
            |> filter(fn: (r) => r["_measurement"] == "sensor_reading")
            {filter_clause}
            |> filter(fn: (r) => r["_field"] == "temperature" or 
                                r["_field"] == "humidity" or 
                                r["_field"] == "pressure")
            |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
        '''
        
        result = self.query_api.query(org=self.org, query=query)
        
        return self._process_query_result(result)
    
    def aggregate_data(self, window_period="5m", aggregation="mean"):
        """데이터 집계"""
        query = f'''
        from(bucket: "{self.bucket}")
            |> range(start: -24h)
            |> filter(fn: (r) => r["_measurement"] == "sensor_reading")
            |> filter(fn: (r) => r["_field"] == "temperature")
            |> aggregateWindow(
                every: {window_period},
                fn: {aggregation},
                createEmpty: false
            )
            |> yield(name: "{aggregation}")
        '''
        
        result = self.query_api.query(org=self.org, query=query)
        return self._process_query_result(result)
    
    def downsampling_task(self):
        """다운샘플링 태스크 생성"""
        task = '''
        option task = {
            name: "downsample_sensors",
            every: 1h,
        }

        from(bucket: "iot_data")
            |> range(start: -task.every)
            |> filter(fn: (r) => r["_measurement"] == "sensor_reading")
            |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
            |> to(
                bucket: "iot_data_downsampled",
                org: "your-org",
                tagColumns: ["sensor_id", "location", "sensor_type"]
            )
        '''
        
        tasks_api = self.client.tasks_api()
        task = tasks_api.create_task_every(
            name="downsample_sensors",
            flux=task,
            every="1h",
            organization=self.org
        )
        
        return task
    
    def anomaly_detection(self):
        """이상 탐지 쿼리"""
        query = '''
        import "contrib/anaisdg/anomalydetection"
        
        from(bucket: "iot_data")
            |> range(start: -7d)
            |> filter(fn: (r) => r["_measurement"] == "sensor_reading")
            |> filter(fn: (r) => r["_field"] == "temperature")
            |> anomalydetection.mad(
                threshold: 3.5,
                windowSize: 12h,
                columns: ["_value"]
            )
            |> filter(fn: (r) => r["level"] == "anomaly")
        '''
        
        result = self.query_api.query(org=self.org, query=query)
        return self._process_query_result(result)
    
    def _process_query_result(self, result):
        """쿼리 결과 처리"""
        processed_data = []
        
        for table in result:
            for record in table.records:
                processed_data.append({
                    'time': record.get_time(),
                    'sensor_id': record.values.get('sensor_id'),
                    'field': record.get_field(),
                    'value': record.get_value(),
                    'measurement': record.get_measurement()
                })
        
        return pd.DataFrame(processed_data)
```

### 1.2 고급 시계열 분석

```python
class TimeSeriesAnalytics:
    """시계열 분석 기능"""
    
    def __init__(self, influx_client):
        self.client = influx_client
        
    def seasonal_decomposition(self, sensor_id, field="temperature"):
        """계절성 분해"""
        query = f'''
        from(bucket: "iot_data")
            |> range(start: -30d)
            |> filter(fn: (r) => r["sensor_id"] == "{sensor_id}")
            |> filter(fn: (r) => r["_field"] == "{field}")
            |> aggregateWindow(every: 1h, fn: mean)
        '''
        
        # Flux의 내장 함수를 사용한 분해
        decomposition_query = f'''
        import "contrib/anaisdg/statsmodels"
        
        data = from(bucket: "iot_data")
            |> range(start: -30d)
            |> filter(fn: (r) => r["sensor_id"] == "{sensor_id}")
            |> filter(fn: (r) => r["_field"] == "{field}")
            |> aggregateWindow(every: 1h, fn: mean)
        
        statsmodels.seasonalDecompose(
            tables: data,
            timeColumn: "_time",
            valueColumn: "_value",
            period: 24,  // 일별 계절성
            model: "additive"
        )
        '''
        
        result = self.client.query_api.query(query=decomposition_query)
        
        return {
            'trend': self._extract_component(result, 'trend'),
            'seasonal': self._extract_component(result, 'seasonal'),
            'residual': self._extract_component(result, 'residual')
        }
    
    def forecasting(self, sensor_id, field="temperature", forecast_hours=24):
        """시계열 예측"""
        # Holt-Winters 예측
        forecast_query = f'''
        import "contrib/anaisdg/forecast"
        
        data = from(bucket: "iot_data")
            |> range(start: -7d)
            |> filter(fn: (r) => r["sensor_id"] == "{sensor_id}")
            |> filter(fn: (r) => r["_field"] == "{field}")
            |> aggregateWindow(every: 1h, fn: mean)
        
        forecast.holtWinters(
            tables: data,
            n: {forecast_hours},
            seasonality: 24,
            withFit: true
        )
        '''
        
        result = self.client.query_api.query(query=forecast_query)
        
        return self._process_forecast_result(result)
    
    def correlation_analysis(self, sensor_ids, window="1h"):
        """센서 간 상관관계 분석"""
        correlation_query = f'''
        sensor1 = from(bucket: "iot_data")
            |> range(start: -7d)
            |> filter(fn: (r) => r["sensor_id"] == "{sensor_ids[0]}")
            |> filter(fn: (r) => r["_field"] == "temperature")
            |> aggregateWindow(every: {window}, fn: mean)
            |> keep(columns: ["_time", "_value"])
            |> rename(columns: {{_value: "temp1"}})
        
        sensor2 = from(bucket: "iot_data")
            |> range(start: -7d)
            |> filter(fn: (r) => r["sensor_id"] == "{sensor_ids[1]}")
            |> filter(fn: (r) => r["_field"] == "temperature")
            |> aggregateWindow(every: {window}, fn: mean)
            |> keep(columns: ["_time", "_value"])
            |> rename(columns: {{_value: "temp2"}})
        
        join(
            tables: {{s1: sensor1, s2: sensor2}},
            on: ["_time"]
        )
        |> pearsonr(columns: ["temp1", "temp2"])
        '''
        
        result = self.client.query_api.query(query=correlation_query)
        
        return self._extract_correlation(result)
    
    def pattern_recognition(self, sensor_id, pattern_window="6h"):
        """패턴 인식"""
        pattern_query = f'''
        import "contrib/anaisdg/anomalydetection"
        
        // 기본 데이터 로드
        data = from(bucket: "iot_data")
            |> range(start: -30d)
            |> filter(fn: (r) => r["sensor_id"] == "{sensor_id}")
            |> filter(fn: (r) => r["_field"] == "temperature")
        
        // 패턴 감지를 위한 특징 추출
        features = data
            |> window(every: {pattern_window})
            |> reduce(
                fn: (r, accumulator) => ({{
                    count: accumulator.count + 1,
                    sum: accumulator.sum + r._value,
                    min: if r._value < accumulator.min then r._value else accumulator.min,
                    max: if r._value > accumulator.max then r._value else accumulator.max,
                    sum_sq: accumulator.sum_sq + (r._value * r._value)
                }}),
                identity: {{count: 0, sum: 0.0, min: 1000.0, max: -1000.0, sum_sq: 0.0}}
            )
            |> map(fn: (r) => ({{
                r with
                mean: r.sum / float(v: r.count),
                range: r.max - r.min,
                variance: (r.sum_sq / float(v: r.count)) - ((r.sum / float(v: r.count)) * (r.sum / float(v: r.count)))
            }}))
        
        // 클러스터링을 통한 패턴 식별
        patterns = features
            |> kmeans(
                columns: ["mean", "range", "variance"],
                k: 5,
                maxIterations: 20
            )
        '''
        
        result = self.client.query_api.query(query=pattern_query)
        
        return self._extract_patterns(result)
```

## 2. TimescaleDB - PostgreSQL 기반 시계열

### 2.1 TimescaleDB 하이퍼테이블

```python
import psycopg2
from psycopg2.extras import RealDictCursor
import pandas as pd

class TimescaleDBManager:
    """TimescaleDB 관리"""
    
    def __init__(self, connection_string):
        self.conn = psycopg2.connect(connection_string)
        self.cursor = self.conn.cursor(cursor_factory=RealDictCursor)
        
    def setup_hypertable(self):
        """하이퍼테이블 설정"""
        # TimescaleDB 확장 활성화
        self.cursor.execute("CREATE EXTENSION IF NOT EXISTS timescaledb")
        
        # 센서 데이터 테이블 생성
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS sensor_data (
                time TIMESTAMPTZ NOT NULL,
                sensor_id TEXT NOT NULL,
                location TEXT NOT NULL,
                temperature DOUBLE PRECISION,
                humidity DOUBLE PRECISION,
                pressure DOUBLE PRECISION,
                co2_level DOUBLE PRECISION,
                battery_level DOUBLE PRECISION,
                metadata JSONB
            );
        """)
        
        # 하이퍼테이블로 변환
        self.cursor.execute("""
            SELECT create_hypertable(
                'sensor_data',
                'time',
                chunk_time_interval => INTERVAL '1 day',
                if_not_exists => TRUE
            );
        """)
        
        # 인덱스 생성
        indexes = [
            "CREATE INDEX idx_sensor_id_time ON sensor_data (sensor_id, time DESC)",
            "CREATE INDEX idx_location_time ON sensor_data (location, time DESC)",
            "CREATE INDEX idx_metadata ON sensor_data USING GIN (metadata)"
        ]
        
        for index in indexes:
            self.cursor.execute(f"{index} IF NOT EXISTS")
        
        self.conn.commit()
    
    def create_continuous_aggregates(self):
        """연속 집계 뷰 생성"""
        # 시간별 집계
        self.cursor.execute("""
            CREATE MATERIALIZED VIEW sensor_hourly
            WITH (timescaledb.continuous) AS
            SELECT 
                time_bucket('1 hour', time) AS hour,
                sensor_id,
                location,
                AVG(temperature) AS avg_temp,
                MIN(temperature) AS min_temp,
                MAX(temperature) AS max_temp,
                AVG(humidity) AS avg_humidity,
                AVG(pressure) AS avg_pressure,
                AVG(co2_level) AS avg_co2,
                MIN(battery_level) AS min_battery,
                COUNT(*) AS reading_count
            FROM sensor_data
            GROUP BY hour, sensor_id, location
            WITH NO DATA;
        """)
        
        # 일별 집계
        self.cursor.execute("""
            CREATE MATERIALIZED VIEW sensor_daily
            WITH (timescaledb.continuous) AS
            SELECT 
                time_bucket('1 day', time) AS day,
                sensor_id,
                location,
                AVG(temperature) AS avg_temp,
                STDDEV(temperature) AS stddev_temp,
                percentile_cont(0.5) WITHIN GROUP (ORDER BY temperature) AS median_temp,
                AVG(humidity) AS avg_humidity,
                AVG(pressure) AS avg_pressure,
                MAX(co2_level) AS max_co2,
                MIN(battery_level) AS min_battery,
                COUNT(*) AS reading_count
            FROM sensor_data
            GROUP BY day, sensor_id, location
            WITH NO DATA;
        """)
        
        # 자동 새로고침 정책 추가
        self.cursor.execute("""
            SELECT add_continuous_aggregate_policy(
                'sensor_hourly',
                start_offset => INTERVAL '3 hours',
                end_offset => INTERVAL '1 hour',
                schedule_interval => INTERVAL '30 minutes'
            );
        """)
        
        self.cursor.execute("""
            SELECT add_continuous_aggregate_policy(
                'sensor_daily',
                start_offset => INTERVAL '3 days',
                end_offset => INTERVAL '1 day',
                schedule_interval => INTERVAL '1 hour'
            );
        """)
        
        self.conn.commit()
    
    def setup_data_retention(self):
        """데이터 보존 정책 설정"""
        # 압축 정책
        self.cursor.execute("""
            SELECT add_compression_policy(
                'sensor_data',
                INTERVAL '7 days',
                if_not_exists => TRUE
            );
        """)
        
        # 보존 정책 (90일 이상 데이터 삭제)
        self.cursor.execute("""
            SELECT add_retention_policy(
                'sensor_data',
                INTERVAL '90 days',
                if_not_exists => TRUE
            );
        """)
        
        # 재정렬 정책 (성능 최적화)
        self.cursor.execute("""
            SELECT add_reorder_policy(
                'sensor_data',
                'sensor_data_sensor_id_time_idx',
                if_not_exists => TRUE
            );
        """)
        
        self.conn.commit()
    
    def time_series_functions(self):
        """시계열 함수 사용 예제"""
        # 이동 평균
        moving_avg_query = """
        SELECT 
            time,
            sensor_id,
            temperature,
            AVG(temperature) OVER (
                PARTITION BY sensor_id 
                ORDER BY time 
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
            ) AS moving_avg_7,
            temperature - LAG(temperature) OVER (
                PARTITION BY sensor_id 
                ORDER BY time
            ) AS temp_change
        FROM sensor_data
        WHERE time > NOW() - INTERVAL '24 hours'
        ORDER BY sensor_id, time;
        """
        
        # 시간대별 패턴 분석
        pattern_query = """
        WITH hourly_patterns AS (
            SELECT 
                EXTRACT(HOUR FROM time) AS hour_of_day,
                EXTRACT(DOW FROM time) AS day_of_week,
                sensor_id,
                AVG(temperature) AS avg_temp,
                STDDEV(temperature) AS stddev_temp,
                COUNT(*) AS sample_count
            FROM sensor_data
            WHERE time > NOW() - INTERVAL '30 days'
            GROUP BY hour_of_day, day_of_week, sensor_id
        )
        SELECT 
            hour_of_day,
            day_of_week,
            sensor_id,
            avg_temp,
            stddev_temp,
            CASE 
                WHEN day_of_week IN (0, 6) THEN 'Weekend'
                ELSE 'Weekday'
            END AS day_type
        FROM hourly_patterns
        ORDER BY sensor_id, day_of_week, hour_of_day;
        """
        
        # 갭 검출 (데이터 누락)
        gap_detection_query = """
        WITH expected_times AS (
            SELECT generate_series(
                NOW() - INTERVAL '24 hours',
                NOW(),
                INTERVAL '5 minutes'
            ) AS expected_time
        ),
        sensor_list AS (
            SELECT DISTINCT sensor_id 
            FROM sensor_data 
            WHERE time > NOW() - INTERVAL '24 hours'
        ),
        expected_readings AS (
            SELECT 
                s.sensor_id,
                e.expected_time
            FROM sensor_list s
            CROSS JOIN expected_times e
        )
        SELECT 
            er.sensor_id,
            er.expected_time,
            sd.time AS actual_time,
            CASE 
                WHEN sd.time IS NULL THEN 'Missing'
                ELSE 'Present'
            END AS status
        FROM expected_readings er
        LEFT JOIN sensor_data sd 
            ON er.sensor_id = sd.sensor_id 
            AND sd.time BETWEEN er.expected_time - INTERVAL '2.5 minutes' 
                            AND er.expected_time + INTERVAL '2.5 minutes'
        WHERE sd.time IS NULL
        ORDER BY er.sensor_id, er.expected_time;
        """
        
        return {
            'moving_average': pd.read_sql(moving_avg_query, self.conn),
            'patterns': pd.read_sql(pattern_query, self.conn),
            'gaps': pd.read_sql(gap_detection_query, self.conn)
        }
```

## 3. IoT 데이터 처리 아키텍처

### 3.1 실시간 IoT 데이터 파이프라인

```python
import asyncio
import aiohttp
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import json
import msgpack
from datetime import datetime
import redis.asyncio as redis

class IoTDataPipeline:
    """IoT 데이터 처리 파이프라인"""
    
    def __init__(self):
        self.kafka_producer = None
        self.kafka_consumer = None
        self.redis_client = None
        self.time_series_db = None
        
    async def initialize(self):
        """파이프라인 초기화"""
        # Kafka 프로듀서
        self.kafka_producer = AIOKafkaProducer(
            bootstrap_servers='localhost:9092',
            value_serializer=lambda v: msgpack.packb(v),
            compression_type="lz4",
            acks='all'
        )
        await self.kafka_producer.start()
        
        # Kafka 컨슈머
        self.kafka_consumer = AIOKafkaConsumer(
            'iot-raw-data',
            bootstrap_servers='localhost:9092',
            group_id="iot-processor",
            value_deserializer=lambda m: msgpack.unpackb(m, raw=False),
            enable_auto_commit=False
        )
        await self.kafka_consumer.start()
        
        # Redis (캐싱 및 실시간 상태)
        self.redis_client = await redis.create_redis_pool(
            'redis://localhost',
            encoding='utf-8'
        )
    
    async def ingest_sensor_data(self, sensor_data):
        """센서 데이터 수집"""
        # 데이터 검증
        validated_data = self.validate_sensor_data(sensor_data)
        
        # 데이터 보강
        enriched_data = await self.enrich_data(validated_data)
        
        # Kafka로 전송
        await self.kafka_producer.send(
            'iot-raw-data',
            value=enriched_data,
            key=enriched_data['sensor_id'].encode()
        )
        
        # 실시간 상태 업데이트
        await self.update_real_time_state(enriched_data)
        
        return enriched_data
    
    def validate_sensor_data(self, data):
        """데이터 검증"""
        required_fields = ['sensor_id', 'timestamp', 'temperature', 'humidity']
        
        # 필수 필드 확인
        for field in required_fields:
            if field not in data:
                raise ValueError(f"Missing required field: {field}")
        
        # 범위 검증
        if not -50 <= data['temperature'] <= 100:
            raise ValueError(f"Temperature out of range: {data['temperature']}")
            
        if not 0 <= data['humidity'] <= 100:
            raise ValueError(f"Humidity out of range: {data['humidity']}")
        
        # 타임스탬프 검증
        try:
            timestamp = datetime.fromisoformat(data['timestamp'])
            if timestamp > datetime.now():
                raise ValueError("Future timestamp not allowed")
        except:
            raise ValueError("Invalid timestamp format")
        
        return data
    
    async def enrich_data(self, data):
        """데이터 보강"""
        # 센서 메타데이터 추가
        sensor_metadata = await self.redis_client.hgetall(f"sensor:{data['sensor_id']}")
        
        enriched = data.copy()
        enriched.update({
            'location': sensor_metadata.get('location', 'unknown'),
            'sensor_type': sensor_metadata.get('type', 'unknown'),
            'calibration_offset': float(sensor_metadata.get('calibration_offset', 0)),
            'ingestion_time': datetime.now().isoformat()
        })
        
        # 보정된 값 계산
        enriched['temperature_calibrated'] = (
            enriched['temperature'] + enriched['calibration_offset']
        )
        
        return enriched
    
    async def process_stream(self):
        """스트림 처리"""
        async for msg in self.kafka_consumer:
            try:
                data = msg.value
                
                # 실시간 분석
                analytics_result = await self.real_time_analytics(data)
                
                # 이상 탐지
                if await self.detect_anomaly(data):
                    await self.trigger_alert(data)
                
                # 시계열 DB 저장
                await self.store_time_series(data)
                
                # 집계 업데이트
                await self.update_aggregations(data)
                
                # 커밋
                await self.kafka_consumer.commit()
                
            except Exception as e:
                print(f"Processing error: {e}")
                # 에러 처리 및 DLQ로 전송
                await self.send_to_dlq(msg, str(e))
    
    async def real_time_analytics(self, data):
        """실시간 분석"""
        sensor_id = data['sensor_id']
        
        # 슬라이딩 윈도우 통계
        window_key = f"window:{sensor_id}:temperature"
        await self.redis_client.zadd(
            window_key,
            {json.dumps(data): data['timestamp']}
        )
        
        # 5분 윈도우 유지
        cutoff = datetime.now().timestamp() - 300
        await self.redis_client.zremrangebyscore(window_key, 0, cutoff)
        
        # 통계 계산
        window_data = await self.redis_client.zrange(window_key, 0, -1)
        temperatures = [json.loads(d)['temperature'] for d in window_data]
        
        if temperatures:
            stats = {
                'mean': np.mean(temperatures),
                'std': np.std(temperatures),
                'min': np.min(temperatures),
                'max': np.max(temperatures),
                'count': len(temperatures)
            }
            
            # 통계 저장
            await self.redis_client.hmset(
                f"stats:{sensor_id}",
                {k: str(v) for k, v in stats.items()}
            )
            
            return stats
        
        return None
    
    async def detect_anomaly(self, data):
        """이상 탐지"""
        sensor_id = data['sensor_id']
        
        # 통계 기반 이상 탐지
        stats = await self.redis_client.hgetall(f"stats:{sensor_id}")
        
        if stats and 'mean' in stats and 'std' in stats:
            mean = float(stats['mean'])
            std = float(stats['std'])
            
            # Z-score 계산
            z_score = abs((data['temperature'] - mean) / (std + 0.001))
            
            if z_score > 3:  # 3 표준편차 이상
                return True
        
        # 급격한 변화 감지
        last_value_key = f"last_value:{sensor_id}"
        last_value = await self.redis_client.get(last_value_key)
        
        if last_value:
            change = abs(data['temperature'] - float(last_value))
            if change > 10:  # 10도 이상 급변
                return True
        
        await self.redis_client.set(last_value_key, data['temperature'])
        
        return False
    
    async def trigger_alert(self, data):
        """알림 발생"""
        alert = {
            'alert_id': f"alert_{datetime.now().timestamp()}",
            'sensor_id': data['sensor_id'],
            'alert_type': 'anomaly',
            'severity': 'high',
            'value': data['temperature'],
            'timestamp': datetime.now().isoformat(),
            'message': f"Anomalous temperature detected: {data['temperature']}°C"
        }
        
        # 알림 큐로 전송
        await self.kafka_producer.send('alerts', value=alert)
        
        # 실시간 알림 (WebSocket)
        await self.send_websocket_alert(alert)
```

### 3.2 엣지 컴퓨팅 통합

```python
class EdgeComputingIntegration:
    """엣지 컴퓨팅 통합"""
    
    def __init__(self):
        self.edge_nodes = {}
        self.aggregation_rules = {}
        
    async def register_edge_node(self, node_id, capabilities):
        """엣지 노드 등록"""
        self.edge_nodes[node_id] = {
            'id': node_id,
            'capabilities': capabilities,
            'status': 'active',
            'last_heartbeat': datetime.now(),
            'processing_rules': [],
            'connected_sensors': []
        }
        
        # 처리 규칙 할당
        await self.assign_processing_rules(node_id)
    
    async def edge_preprocessing(self, node_id, sensor_data_batch):
        """엣지에서 전처리"""
        processed_batch = []
        
        for data in sensor_data_batch:
            # 노이즈 필터링
            filtered_data = self.apply_kalman_filter(data)
            
            # 특징 추출
            features = self.extract_features(filtered_data)
            
            # 압축
            compressed = self.compress_data(filtered_data)
            
            processed_batch.append({
                'original': compressed,
                'features': features,
                'edge_node': node_id,
                'processing_time': datetime.now().isoformat()
            })
        
        return processed_batch
    
    def apply_kalman_filter(self, data):
        """칼만 필터 적용"""
        # 간단한 칼만 필터 구현
        if not hasattr(self, 'kalman_state'):
            self.kalman_state = {}
        
        sensor_id = data['sensor_id']
        
        if sensor_id not in self.kalman_state:
            self.kalman_state[sensor_id] = {
                'x': data['temperature'],  # 상태 추정
                'P': 1.0,  # 오차 공분산
                'Q': 0.01,  # 프로세스 노이즈
                'R': 0.1   # 측정 노이즈
            }
        
        state = self.kalman_state[sensor_id]
        
        # 예측 단계
        x_pred = state['x']
        P_pred = state['P'] + state['Q']
        
        # 업데이트 단계
        K = P_pred / (P_pred + state['R'])  # 칼만 이득
        state['x'] = x_pred + K * (data['temperature'] - x_pred)
        state['P'] = (1 - K) * P_pred
        
        filtered_data = data.copy()
        filtered_data['temperature_filtered'] = state['x']
        
        return filtered_data
    
    def extract_features(self, data):
        """특징 추출"""
        features = {
            'sensor_id': data['sensor_id'],
            'timestamp': data['timestamp']
        }
        
        # 통계적 특징
        if hasattr(self, 'feature_buffer'):
            buffer = self.feature_buffer.get(data['sensor_id'], [])
            buffer.append(data['temperature'])
            
            if len(buffer) > 100:
                buffer = buffer[-100:]  # 최근 100개 유지
            
            self.feature_buffer[data['sensor_id']] = buffer
            
            if len(buffer) >= 10:
                features.update({
                    'mean_10': np.mean(buffer[-10:]),
                    'std_10': np.std(buffer[-10:]),
                    'trend': np.polyfit(range(10), buffer[-10:], 1)[0],
                    'range': max(buffer[-10:]) - min(buffer[-10:])
                })
        else:
            self.feature_buffer = {data['sensor_id']: [data['temperature']]}
        
        return features
    
    def compress_data(self, data):
        """데이터 압축"""
        # Delta 인코딩
        if hasattr(self, 'last_values'):
            last = self.last_values.get(data['sensor_id'], {})
            
            compressed = {
                'sensor_id': data['sensor_id'],
                'timestamp': data['timestamp'],
                'deltas': {}
            }
            
            for field in ['temperature', 'humidity', 'pressure']:
                if field in data and field in last:
                    compressed['deltas'][field] = data[field] - last[field]
                else:
                    compressed['deltas'][field] = data.get(field, 0)
                    
            self.last_values[data['sensor_id']] = data
        else:
            self.last_values = {data['sensor_id']: data}
            compressed = data
        
        return compressed
    
    async def federated_learning(self):
        """연합 학습"""
        # 각 엣지 노드에서 로컬 모델 학습
        local_models = {}
        
        for node_id, node_info in self.edge_nodes.items():
            if node_info['status'] == 'active':
                # 로컬 데이터로 학습
                local_model = await self.train_local_model(node_id)
                local_models[node_id] = local_model
        
        # 모델 집계
        global_model = self.aggregate_models(local_models)
        
        # 업데이트된 모델 배포
        for node_id in self.edge_nodes:
            await self.deploy_model_to_edge(node_id, global_model)
        
        return global_model
```

## 4. 실시간 모니터링 대시보드

### 4.1 Grafana 통합

```python
import requests
from grafana_api.grafana_face import GrafanaFace

class IoTMonitoringDashboard:
    """IoT 모니터링 대시보드"""
    
    def __init__(self, grafana_url, api_key):
        self.grafana = GrafanaFace(
            auth=api_key,
            host=grafana_url
        )
        
    def create_iot_dashboard(self):
        """IoT 대시보드 생성"""
        dashboard = {
            "dashboard": {
                "title": "IoT Sensor Monitoring",
                "tags": ["iot", "sensors", "real-time"],
                "timezone": "browser",
                "panels": [
                    self.create_temperature_panel(),
                    self.create_humidity_panel(),
                    self.create_anomaly_panel(),
                    self.create_battery_panel(),
                    self.create_location_map(),
                    self.create_alert_table()
                ],
                "refresh": "5s",
                "time": {
                    "from": "now-6h",
                    "to": "now"
                }
            },
            "overwrite": True
        }
        
        response = self.grafana.dashboard.update_dashboard(dashboard)
        return response
    
    def create_temperature_panel(self):
        """온도 패널 생성"""
        return {
            "id": 1,
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
            "type": "graph",
            "title": "Temperature Trends",
            "datasource": "InfluxDB",
            "targets": [{
                "query": '''
                    from(bucket: "iot_data")
                        |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
                        |> filter(fn: (r) => r["_measurement"] == "sensor_reading")
                        |> filter(fn: (r) => r["_field"] == "temperature")
                        |> aggregateWindow(every: v.windowPeriod, fn: mean)
                ''',
                "refId": "A"
            }],
            "fieldConfig": {
                "defaults": {
                    "unit": "celsius",
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {"color": "blue", "value": null},
                            {"color": "green", "value": 20},
                            {"color": "yellow", "value": 30},
                            {"color": "red", "value": 40}
                        ]
                    }
                }
            }
        }
    
    def create_anomaly_panel(self):
        """이상 탐지 패널"""
        return {
            "id": 3,
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
            "type": "stat",
            "title": "Anomaly Detection",
            "datasource": "InfluxDB",
            "targets": [{
                "query": '''
                    from(bucket: "iot_data")
                        |> range(start: -1h)
                        |> filter(fn: (r) => r["_measurement"] == "anomalies")
                        |> count()
                ''',
                "refId": "A"
            }],
            "fieldConfig": {
                "defaults": {
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {"color": "green", "value": null},
                            {"color": "yellow", "value": 5},
                            {"color": "red", "value": 10}
                        ]
                    },
                    "unit": "short"
                },
                "overrides": []
            },
            "options": {
                "reduceOptions": {
                    "values": False,
                    "calcs": ["lastNotNull"],
                    "fields": ""
                },
                "orientation": "auto",
                "textMode": "auto",
                "colorMode": "background",
                "graphMode": "area",
                "justifyMode": "auto"
            }
        }
    
    def create_alert_rules(self):
        """알림 규칙 생성"""
        alert_rules = [
            {
                "uid": "temperature_high",
                "title": "High Temperature Alert",
                "condition": "A",
                "data": [{
                    "refId": "A",
                    "queryType": "",
                    "model": {
                        "expr": 'sensor_temperature > 40',
                        "intervalMs": 1000,
                        "maxDataPoints": 43200
                    }
                }],
                "noDataState": "NoData",
                "execErrState": "Alerting",
                "for": "5m",
                "annotations": {
                    "description": "Temperature exceeded threshold",
                    "runbook_url": "https://wiki.company.com/runbooks/high-temp"
                },
                "labels": {
                    "severity": "critical",
                    "team": "iot-ops"
                }
            },
            {
                "uid": "sensor_offline",
                "title": "Sensor Offline Alert",
                "condition": "A",
                "data": [{
                    "refId": "A",
                    "queryType": "",
                    "model": {
                        "expr": 'up{job="iot-sensors"} == 0',
                        "intervalMs": 1000,
                        "maxDataPoints": 43200
                    }
                }],
                "for": "10m",
                "annotations": {
                    "description": "Sensor has been offline for 10 minutes"
                },
                "labels": {
                    "severity": "warning"
                }
            }
        ]
        
        for rule in alert_rules:
            self.grafana.alerting.create_alert_rule(rule)
```

## 5. 시계열 데이터 최적화

### 5.1 압축과 다운샘플링 전략

```python
class TimeSeriesOptimization:
    """시계열 데이터 최적화"""
    
    def __init__(self, tsdb_connection):
        self.db = tsdb_connection
        
    def implement_compression(self):
        """압축 구현"""
        # Gorilla 압축 알고리즘 (Facebook)
        class GorillaCompression:
            def __init__(self):
                self.previous_value = None
                self.previous_timestamp = None
                
            def compress_value(self, value, timestamp):
                if self.previous_value is None:
                    # 첫 번째 값은 압축하지 않음
                    self.previous_value = value
                    self.previous_timestamp = timestamp
                    return {'type': 'first', 'value': value, 'timestamp': timestamp}
                
                # 타임스탬프 델타 인코딩
                time_delta = timestamp - self.previous_timestamp
                
                # 값 XOR 인코딩
                xor_value = self.float_to_bits(value) ^ self.float_to_bits(self.previous_value)
                
                # 리딩/트레일링 제로 계산
                leading_zeros = self.count_leading_zeros(xor_value)
                trailing_zeros = self.count_trailing_zeros(xor_value)
                
                self.previous_value = value
                self.previous_timestamp = timestamp
                
                return {
                    'type': 'delta',
                    'time_delta': time_delta,
                    'leading_zeros': leading_zeros,
                    'trailing_zeros': trailing_zeros,
                    'significant_bits': xor_value >> trailing_zeros
                }
            
            def float_to_bits(self, value):
                # IEEE 754 변환
                import struct
                return struct.unpack('Q', struct.pack('d', value))[0]
        
        return GorillaCompression()
    
    def adaptive_downsampling(self, data, target_points=1000):
        """적응형 다운샘플링"""
        # Largest Triangle Three Buckets (LTTB) 알고리즘
        def lttb_downsample(data, threshold):
            if len(data) <= threshold:
                return data
            
            # 버킷 크기 계산
            every = (len(data) - 2) / (threshold - 2)
            
            downsampled = [data[0]]  # 첫 번째 포인트
            
            for i in range(1, threshold - 1):
                # 버킷 범위 계산
                avg_start = int(i * every)
                avg_end = int((i + 1) * every)
                
                # 다음 버킷의 평균점 계산
                avg_x = np.mean([d[0] for d in data[avg_end:avg_end + int(every)]])
                avg_y = np.mean([d[1] for d in data[avg_end:avg_end + int(every)]])
                
                # 현재 버킷에서 최대 면적을 가진 점 선택
                max_area = -1
                max_area_point = None
                
                for j in range(avg_start, avg_end):
                    # 삼각형 면적 계산
                    area = abs(
                        (downsampled[-1][0] - avg_x) * (data[j][1] - downsampled[-1][1]) -
                        (downsampled[-1][0] - data[j][0]) * (avg_y - downsampled[-1][1])
                    )
                    
                    if area > max_area:
                        max_area = area
                        max_area_point = data[j]
                
                downsampled.append(max_area_point)
            
            downsampled.append(data[-1])  # 마지막 포인트
            
            return downsampled
        
        # 데이터 형식 변환
        time_value_pairs = [(d['timestamp'], d['value']) for d in data]
        
        # 다운샘플링 적용
        downsampled = lttb_downsample(time_value_pairs, target_points)
        
        # 결과 형식 변환
        return [{'timestamp': t, 'value': v} for t, v in downsampled]
    
    def hierarchical_aggregation(self):
        """계층적 집계"""
        # 다중 해상도 저장
        aggregation_levels = [
            {'interval': '1m', 'retention': '7d', 'function': 'mean'},
            {'interval': '5m', 'retention': '30d', 'function': 'mean'},
            {'interval': '1h', 'retention': '90d', 'function': 'mean'},
            {'interval': '1d', 'retention': '1y', 'function': 'mean'},
        ]
        
        for level in aggregation_levels:
            query = f"""
            CREATE CONTINUOUS AGGREGATE sensor_agg_{level['interval']}
            WITH (timescaledb.continuous) AS
            SELECT 
                time_bucket('{level['interval']}', time) AS bucket,
                sensor_id,
                {level['function']}(temperature) AS temperature,
                {level['function']}(humidity) AS humidity,
                COUNT(*) AS sample_count
            FROM sensor_data
            GROUP BY bucket, sensor_id
            WITH NO DATA;
            
            SELECT add_continuous_aggregate_policy(
                'sensor_agg_{level['interval']}',
                start_offset => INTERVAL '2 {level["interval"]}',
                end_offset => INTERVAL '1 {level["interval"]}',
                schedule_interval => INTERVAL '{level["interval"]}'
            );
            
            SELECT add_retention_policy(
                'sensor_agg_{level['interval']}',
                INTERVAL '{level["retention"]}'
            );
            """
            
            self.db.execute(query)
```

## 마무리

시계열 데이터베이스는 IoT 시대의 핵심 인프라입니다. InfluxDB, TimescaleDB와 같은 전문 시계열 데이터베이스를 활용하여 대량의 센서 데이터를 효율적으로 저장하고 분석할 수 있습니다. 엣지 컴퓨팅 통합, 실시간 이상 탐지, 예측 분석 등의 고급 기능을 통해 IoT 데이터에서 가치 있는 인사이트를 추출할 수 있습니다. 효율적인 압축과 다운샘플링 전략으로 스토리지 비용을 최적화하면서도 데이터의 정확성을 유지할 수 있습니다.

다음 장에서는 데이터베이스와 머신러닝의 통합에 대해 학습하겠습니다.