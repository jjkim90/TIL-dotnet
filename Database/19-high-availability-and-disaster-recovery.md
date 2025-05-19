# 고가용성과 재해 복구

## 개요

데이터베이스의 고가용성(High Availability)과 재해 복구(Disaster Recovery)는 비즈니스 연속성을 보장하는 핵심 요소입니다. 이 장에서는 다양한 고가용성 솔루션, 재해 복구 전략, 자동 장애 조치(Failover) 메커니즘, 그리고 복구 시간 목표(RTO)와 복구 시점 목표(RPO)를 달성하는 방법을 학습합니다.

## 1. 고가용성 아키텍처

### 1.1 Always On 가용성 그룹

```sql
-- SQL Server Always On 가용성 그룹 구성
-- 1. 가용성 그룹 생성
CREATE AVAILABILITY GROUP [AG_Production]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,
    FAILURE_CONDITION_LEVEL = 3,
    HEALTH_CHECK_TIMEOUT = 30000,
    DB_FAILOVER = ON,
    DTC_SUPPORT = PER_DB
)
FOR DATABASE [ProductionDB], [InventoryDB], [OrderDB]
REPLICA ON 
    N'SQL-PRIMARY' WITH (
        ENDPOINT_URL = N'TCP://sql-primary.company.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        BACKUP_PRIORITY = 50,
        SECONDARY_ROLE(ALLOW_CONNECTIONS = NO),
        PRIMARY_ROLE(ALLOW_CONNECTIONS = READ_WRITE,
                    READ_ONLY_ROUTING_LIST = ('SQL-SECONDARY1', 'SQL-SECONDARY2')),
        SESSION_TIMEOUT = 10
    ),
    N'SQL-SECONDARY1' WITH (
        ENDPOINT_URL = N'TCP://sql-secondary1.company.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        BACKUP_PRIORITY = 50,
        SECONDARY_ROLE(ALLOW_CONNECTIONS = READ_ONLY),
        PRIMARY_ROLE(ALLOW_CONNECTIONS = READ_WRITE,
                    READ_ONLY_ROUTING_LIST = ('SQL-SECONDARY2', 'SQL-PRIMARY')),
        SESSION_TIMEOUT = 10
    ),
    N'SQL-SECONDARY2' WITH (
        ENDPOINT_URL = N'TCP://sql-secondary2.company.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        BACKUP_PRIORITY = 50,
        SECONDARY_ROLE(ALLOW_CONNECTIONS = READ_ONLY),
        PRIMARY_ROLE(ALLOW_CONNECTIONS = READ_WRITE,
                    READ_ONLY_ROUTING_LIST = ('SQL-SECONDARY1', 'SQL-PRIMARY')),
        SESSION_TIMEOUT = 10
    );

-- 2. 리스너 구성
ALTER AVAILABILITY GROUP [AG_Production]
ADD LISTENER N'AG-Listener' (
    WITH IP ((N'10.0.0.100', N'255.255.255.0')),
    PORT = 1433
);

-- 3. 읽기 전용 라우팅 구성
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL-PRIMARY' WITH (
    PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = 
        (('SQL-SECONDARY1', 'SQL-SECONDARY2'), 'SQL-PRIMARY'))
);
```

### 1.2 고가용성 모니터링 시스템

```python
import pyodbc
import asyncio
import smtplib
from datetime import datetime, timedelta
import json
import logging
from enum import Enum

class HAStatus(Enum):
    HEALTHY = "HEALTHY"
    WARNING = "WARNING"
    CRITICAL = "CRITICAL"
    FAILED = "FAILED"

class AlwaysOnMonitor:
    """Always On 가용성 그룹 모니터"""
    
    def __init__(self, connection_string, alert_config):
        self.connection_string = connection_string
        self.alert_config = alert_config
        self.logger = logging.getLogger(__name__)
        
    async def monitor_availability_groups(self):
        """가용성 그룹 상태 모니터링"""
        while True:
            try:
                status = await self.check_ag_health()
                
                if status['overall_status'] != HAStatus.HEALTHY:
                    await self.send_alert(status)
                
                # 상세 메트릭 수집
                await self.collect_detailed_metrics()
                
                # 5분마다 체크
                await asyncio.sleep(300)
                
            except Exception as e:
                self.logger.error(f"모니터링 오류: {e}")
                await asyncio.sleep(60)
    
    async def check_ag_health(self):
        """가용성 그룹 건강 상태 확인"""
        health_query = """
        SELECT 
            ag.name AS ag_name,
            ar.replica_server_name,
            ar.availability_mode_desc,
            ar.failover_mode_desc,
            ars.role_desc,
            ars.synchronization_state_desc,
            ars.synchronization_health_desc,
            ars.connected_state_desc,
            ars.last_connect_error_number,
            ars.last_connect_error_description,
            ars.last_connect_error_timestamp,
            -- 지연 시간 계산
            CASE 
                WHEN ars.is_primary_replica = 0 AND ars.synchronization_state = 2
                THEN DATEDIFF(SECOND, ars.last_redone_time, GETDATE())
                ELSE 0
            END AS redo_lag_seconds,
            -- 데이터베이스별 상태
            (
                SELECT 
                    database_name,
                    synchronization_state_desc,
                    synchronization_health_desc,
                    log_send_queue_size,
                    log_send_rate,
                    redo_queue_size,
                    redo_rate
                FROM sys.dm_hadr_database_replica_states drs
                JOIN sys.availability_databases_cluster adc 
                    ON drs.group_id = adc.group_id 
                    AND drs.database_id = DB_ID(adc.database_name)
                WHERE drs.group_id = ag.group_id 
                    AND drs.replica_id = ar.replica_id
                FOR JSON PATH
            ) AS database_states
        FROM sys.availability_groups ag
        JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
        JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
        ORDER BY ag.name, ar.replica_server_name;
        """
        
        with pyodbc.connect(self.connection_string) as conn:
            cursor = conn.cursor()
            cursor.execute(health_query)
            results = cursor.fetchall()
        
        return self._analyze_health_results(results)
    
    def _analyze_health_results(self, results):
        """건강 상태 분석"""
        status = {
            'timestamp': datetime.utcnow().isoformat(),
            'overall_status': HAStatus.HEALTHY,
            'availability_groups': {},
            'issues': []
        }
        
        for row in results:
            ag_name = row.ag_name
            replica_name = row.replica_server_name
            
            if ag_name not in status['availability_groups']:
                status['availability_groups'][ag_name] = {
                    'replicas': {},
                    'status': HAStatus.HEALTHY
                }
            
            replica_status = {
                'role': row.role_desc,
                'sync_state': row.synchronization_state_desc,
                'sync_health': row.synchronization_health_desc,
                'connected': row.connected_state_desc,
                'redo_lag_seconds': row.redo_lag_seconds,
                'database_states': json.loads(row.database_states) if row.database_states else []
            }
            
            # 문제 감지
            if row.synchronization_health_desc != 'HEALTHY':
                status['issues'].append({
                    'severity': 'HIGH',
                    'ag': ag_name,
                    'replica': replica_name,
                    'issue': f"Synchronization health: {row.synchronization_health_desc}",
                    'timestamp': datetime.utcnow().isoformat()
                })
                status['overall_status'] = HAStatus.WARNING
            
            if row.connected_state_desc == 'DISCONNECTED':
                status['issues'].append({
                    'severity': 'CRITICAL',
                    'ag': ag_name,
                    'replica': replica_name,
                    'issue': 'Replica disconnected',
                    'timestamp': datetime.utcnow().isoformat()
                })
                status['overall_status'] = HAStatus.CRITICAL
            
            if row.redo_lag_seconds > 300:  # 5분 이상 지연
                status['issues'].append({
                    'severity': 'WARNING',
                    'ag': ag_name,
                    'replica': replica_name,
                    'issue': f"High redo lag: {row.redo_lag_seconds} seconds",
                    'timestamp': datetime.utcnow().isoformat()
                })
            
            status['availability_groups'][ag_name]['replicas'][replica_name] = replica_status
        
        return status
    
    async def perform_failover(self, ag_name, target_replica=None):
        """장애 조치 수행"""
        if target_replica:
            # 계획된 수동 장애 조치
            failover_query = f"""
            ALTER AVAILABILITY GROUP [{ag_name}]
            FAILOVER TO [{target_replica}]
            WITH DATA_LOSS;  -- 비동기 복제본으로의 강제 장애 조치
            """
        else:
            # 자동 장애 조치 (동기 복제본으로)
            failover_query = f"""
            ALTER AVAILABILITY GROUP [{ag_name}] FAILOVER;
            """
        
        try:
            with pyodbc.connect(self.connection_string) as conn:
                cursor = conn.cursor()
                cursor.execute(failover_query)
                conn.commit()
            
            self.logger.info(f"장애 조치 완료: {ag_name}")
            
            # 장애 조치 후 상태 확인
            await asyncio.sleep(10)
            return await self.check_ag_health()
            
        except Exception as e:
            self.logger.error(f"장애 조치 실패: {e}")
            raise
```

## 2. 재해 복구 전략

### 2.1 멀티 리전 재해 복구

```python
class DisasterRecoveryOrchestrator:
    """재해 복구 오케스트레이터"""
    
    def __init__(self, primary_config, dr_config):
        self.primary_config = primary_config
        self.dr_config = dr_config
        self.recovery_state = {
            'last_backup': None,
            'last_log_backup': None,
            'estimated_rpo': None,
            'estimated_rto': None
        }
    
    async def setup_geo_replication(self):
        """지역 간 복제 설정"""
        # Azure SQL Database 지역 복제 예제
        from azure.mgmt.sql import SqlManagementClient
        from azure.identity import DefaultAzureCredential
        
        credential = DefaultAzureCredential()
        sql_client = SqlManagementClient(
            credential, 
            self.primary_config['subscription_id']
        )
        
        # 보조 데이터베이스 생성
        secondary_database = sql_client.databases.begin_create_or_update(
            resource_group_name=self.dr_config['resource_group'],
            server_name=self.dr_config['server_name'],
            database_name=self.primary_config['database_name'],
            parameters={
                'location': self.dr_config['location'],
                'create_mode': 'Secondary',
                'source_database_id': (
                    f"/subscriptions/{self.primary_config['subscription_id']}"
                    f"/resourceGroups/{self.primary_config['resource_group']}"
                    f"/providers/Microsoft.Sql/servers/{self.primary_config['server_name']}"
                    f"/databases/{self.primary_config['database_name']}"
                ),
                'secondary_type': 'Geo',
                'sku': {
                    'name': 'S3',
                    'tier': 'Standard'
                }
            }
        ).result()
        
        return secondary_database
    
    async def execute_dr_drill(self):
        """재해 복구 훈련 실행"""
        drill_results = {
            'start_time': datetime.utcnow(),
            'steps': [],
            'success': True
        }
        
        try:
            # 1. 현재 복제 상태 확인
            replication_status = await self.check_replication_status()
            drill_results['steps'].append({
                'step': 'Check Replication Status',
                'status': 'Success',
                'details': replication_status
            })
            
            # 2. 테스트 장애 조치 (읽기 전용)
            await self.test_failover_readonly()
            drill_results['steps'].append({
                'step': 'Test Failover (Read-Only)',
                'status': 'Success',
                'duration': '45 seconds'
            })
            
            # 3. 애플리케이션 연결 테스트
            app_test = await self.test_application_connectivity()
            drill_results['steps'].append({
                'step': 'Application Connectivity Test',
                'status': 'Success' if app_test else 'Failed',
                'details': app_test
            })
            
            # 4. 데이터 무결성 확인
            integrity_check = await self.verify_data_integrity()
            drill_results['steps'].append({
                'step': 'Data Integrity Check',
                'status': 'Success' if integrity_check['passed'] else 'Failed',
                'details': integrity_check
            })
            
            # 5. 성능 벤치마크
            performance = await self.run_performance_benchmark()
            drill_results['steps'].append({
                'step': 'Performance Benchmark',
                'status': 'Success',
                'details': performance
            })
            
            # 6. 장애 복구
            await self.failback_to_primary()
            drill_results['steps'].append({
                'step': 'Failback to Primary',
                'status': 'Success'
            })
            
        except Exception as e:
            drill_results['success'] = False
            drill_results['error'] = str(e)
        
        drill_results['end_time'] = datetime.utcnow()
        drill_results['total_duration'] = (
            drill_results['end_time'] - drill_results['start_time']
        ).total_seconds()
        
        # 결과 저장 및 리포트 생성
        await self.save_drill_results(drill_results)
        return drill_results
    
    async def implement_backup_strategy(self):
        """백업 전략 구현"""
        backup_policy = {
            'full_backup': {
                'frequency': 'daily',
                'retention_days': 7,
                'time': '02:00',
                'compression': True,
                'checksum': True
            },
            'differential_backup': {
                'frequency': 'every_6_hours',
                'retention_days': 2
            },
            'log_backup': {
                'frequency': 'every_15_minutes',
                'retention_hours': 48
            },
            'offsite_copy': {
                'enabled': True,
                'destination': 's3://dr-backups/',
                'retention_days': 30
            }
        }
        
        # 백업 작업 스케줄링
        for backup_type, config in backup_policy.items():
            await self.schedule_backup_job(backup_type, config)
        
        return backup_policy
    
    def calculate_rto_rpo(self):
        """RTO/RPO 계산"""
        # RPO (Recovery Point Objective) 계산
        last_backup_time = self.recovery_state['last_backup']
        last_log_backup_time = self.recovery_state['last_log_backup']
        
        if last_log_backup_time:
            current_time = datetime.utcnow()
            time_since_last_log = (current_time - last_log_backup_time).total_seconds()
            
            # 트랜잭션 로그 백업 주기를 기반으로 RPO 계산
            estimated_rpo = time_since_last_log + 900  # 15분 백업 주기
        else:
            estimated_rpo = float('inf')
        
        # RTO (Recovery Time Objective) 계산
        # 복구 시간 = 백업 복원 시간 + 로그 적용 시간 + 검증 시간
        backup_size_gb = self.get_backup_size() / 1024 / 1024 / 1024
        restore_speed_mbps = 100  # 추정 복원 속도
        
        restore_time = (backup_size_gb * 1024) / restore_speed_mbps
        log_apply_time = estimated_rpo * 0.1  # 로그 적용은 생성 시간의 10%로 추정
        verification_time = 300  # 5분
        
        estimated_rto = restore_time + log_apply_time + verification_time
        
        return {
            'rpo_seconds': estimated_rpo,
            'rpo_formatted': self.format_duration(estimated_rpo),
            'rto_seconds': estimated_rto,
            'rto_formatted': self.format_duration(estimated_rto),
            'meets_sla': estimated_rpo <= 3600 and estimated_rto <= 14400  # 1시간 RPO, 4시간 RTO
        }
```

## 3. 자동 장애 감지 및 복구

### 3.1 지능형 장애 감지 시스템

```python
import numpy as np
from sklearn.ensemble import IsolationForest
import pickle

class IntelligentFailureDetection:
    """지능형 장애 감지 시스템"""
    
    def __init__(self):
        self.anomaly_detector = IsolationForest(
            contamination=0.1,
            random_state=42
        )
        self.metric_history = []
        self.failure_patterns = self.load_failure_patterns()
        
    async def detect_anomalies(self, metrics):
        """이상 징후 감지"""
        # 메트릭 정규화
        features = self.extract_features(metrics)
        
        # 이상 탐지
        anomaly_score = self.anomaly_detector.decision_function([features])[0]
        is_anomaly = self.anomaly_detector.predict([features])[0] == -1
        
        # 패턴 매칭
        matched_patterns = self.match_failure_patterns(metrics)
        
        # 종합 분석
        risk_assessment = {
            'timestamp': datetime.utcnow().isoformat(),
            'anomaly_score': float(anomaly_score),
            'is_anomaly': bool(is_anomaly),
            'matched_patterns': matched_patterns,
            'risk_level': self.calculate_risk_level(anomaly_score, matched_patterns),
            'recommended_actions': self.get_recommended_actions(metrics, matched_patterns)
        }
        
        # 높은 위험도일 경우 자동 대응
        if risk_assessment['risk_level'] == 'CRITICAL':
            await self.execute_auto_remediation(risk_assessment)
        
        return risk_assessment
    
    def extract_features(self, metrics):
        """메트릭에서 특징 추출"""
        features = []
        
        # CPU 관련 특징
        features.append(metrics.get('cpu_usage', 0))
        features.append(metrics.get('cpu_queue_length', 0))
        
        # 메모리 관련 특징
        features.append(metrics.get('memory_usage_percent', 0))
        features.append(metrics.get('page_life_expectancy', 0) / 1000)  # 정규화
        
        # I/O 관련 특징
        features.append(metrics.get('avg_disk_latency_ms', 0))
        features.append(metrics.get('disk_queue_length', 0))
        
        # 연결 관련 특징
        features.append(metrics.get('connection_count', 0) / 1000)  # 정규화
        features.append(metrics.get('blocked_process_count', 0))
        
        # 쿼리 관련 특징
        features.append(metrics.get('avg_query_duration_ms', 0) / 1000)  # 정규화
        features.append(metrics.get('deadlock_count', 0))
        
        return features
    
    def match_failure_patterns(self, metrics):
        """알려진 장애 패턴 매칭"""
        matched = []
        
        # 메모리 압박 패턴
        if (metrics.get('memory_usage_percent', 0) > 95 and 
            metrics.get('page_life_expectancy', float('inf')) < 300):
            matched.append({
                'pattern': 'memory_pressure',
                'confidence': 0.9,
                'description': 'Critical memory pressure detected'
            })
        
        # I/O 병목 패턴
        if (metrics.get('avg_disk_latency_ms', 0) > 20 and 
            metrics.get('disk_queue_length', 0) > 2):
            matched.append({
                'pattern': 'io_bottleneck',
                'confidence': 0.85,
                'description': 'I/O subsystem bottleneck detected'
            })
        
        # 연결 고갈 패턴
        max_connections = metrics.get('max_connections', 1000)
        if metrics.get('connection_count', 0) > max_connections * 0.9:
            matched.append({
                'pattern': 'connection_exhaustion',
                'confidence': 0.95,
                'description': 'Connection pool near exhaustion'
            })
        
        # 블로킹 체인 패턴
        if (metrics.get('blocked_process_count', 0) > 5 and 
            metrics.get('avg_block_duration_ms', 0) > 5000):
            matched.append({
                'pattern': 'blocking_chain',
                'confidence': 0.8,
                'description': 'Severe blocking chain detected'
            })
        
        return matched
    
    async def execute_auto_remediation(self, risk_assessment):
        """자동 복구 실행"""
        remediation_log = {
            'timestamp': datetime.utcnow().isoformat(),
            'risk_assessment': risk_assessment,
            'actions_taken': []
        }
        
        for pattern in risk_assessment['matched_patterns']:
            if pattern['pattern'] == 'memory_pressure':
                # 메모리 압박 해소
                action = await self.clear_memory_pressure()
                remediation_log['actions_taken'].append(action)
                
            elif pattern['pattern'] == 'io_bottleneck':
                # I/O 최적화
                action = await self.optimize_io_operations()
                remediation_log['actions_taken'].append(action)
                
            elif pattern['pattern'] == 'connection_exhaustion':
                # 연결 정리
                action = await self.cleanup_connections()
                remediation_log['actions_taken'].append(action)
                
            elif pattern['pattern'] == 'blocking_chain':
                # 블로킹 해소
                action = await self.resolve_blocking()
                remediation_log['actions_taken'].append(action)
        
        # 복구 결과 확인
        await asyncio.sleep(30)  # 30초 대기
        post_metrics = await self.collect_current_metrics()
        remediation_log['post_remediation_metrics'] = post_metrics
        
        # 결과 저장
        await self.save_remediation_log(remediation_log)
        
        return remediation_log
```

## 4. 비즈니스 연속성 계획

### 4.1 통합 BC/DR 플랫폼

```python
class BusinessContinuityPlatform:
    """비즈니스 연속성 플랫폼"""
    
    def __init__(self, config):
        self.config = config
        self.dr_sites = config['dr_sites']
        self.primary_site = config['primary_site']
        self.bc_state = 'NORMAL'
        
    async def execute_bc_plan(self, incident_type):
        """비즈니스 연속성 계획 실행"""
        bc_execution = {
            'incident_id': str(uuid.uuid4()),
            'incident_type': incident_type,
            'start_time': datetime.utcnow(),
            'steps': []
        }
        
        try:
            # 1. 인시던트 평가
            assessment = await self.assess_incident(incident_type)
            bc_execution['assessment'] = assessment
            
            # 2. 이해관계자 알림
            await self.notify_stakeholders(assessment)
            bc_execution['steps'].append({
                'step': 'Stakeholder Notification',
                'status': 'Completed',
                'timestamp': datetime.utcnow().isoformat()
            })
            
            # 3. 서비스 우선순위 결정
            service_priority = self.determine_service_priority()
            
            # 4. 복구 전략 선택
            recovery_strategy = self.select_recovery_strategy(
                assessment, 
                service_priority
            )
            bc_execution['recovery_strategy'] = recovery_strategy
            
            # 5. 복구 실행
            for service in service_priority:
                recovery_result = await self.recover_service(
                    service, 
                    recovery_strategy
                )
                bc_execution['steps'].append({
                    'step': f'Recover {service["name"]}',
                    'status': recovery_result['status'],
                    'duration': recovery_result['duration'],
                    'timestamp': datetime.utcnow().isoformat()
                })
            
            # 6. 검증 및 테스트
            validation = await self.validate_recovery()
            bc_execution['validation'] = validation
            
            # 7. 정상 운영 전환
            if validation['success']:
                await self.transition_to_normal_operations()
                bc_execution['status'] = 'SUCCESS'
            else:
                bc_execution['status'] = 'PARTIAL_SUCCESS'
            
        except Exception as e:
            bc_execution['status'] = 'FAILED'
            bc_execution['error'] = str(e)
            self.logger.error(f"BC 계획 실행 실패: {e}")
        
        bc_execution['end_time'] = datetime.utcnow()
        bc_execution['total_duration'] = (
            bc_execution['end_time'] - bc_execution['start_time']
        ).total_seconds()
        
        # 사후 분석을 위한 기록
        await self.save_bc_execution_log(bc_execution)
        
        return bc_execution
    
    def determine_service_priority(self):
        """서비스 우선순위 결정"""
        services = [
            {
                'name': 'Payment Processing',
                'tier': 1,
                'rto_minutes': 15,
                'rpo_minutes': 5,
                'dependencies': ['Core Database', 'Authentication Service']
            },
            {
                'name': 'Order Management',
                'tier': 1,
                'rto_minutes': 30,
                'rpo_minutes': 15,
                'dependencies': ['Core Database', 'Inventory Service']
            },
            {
                'name': 'Customer Portal',
                'tier': 2,
                'rto_minutes': 60,
                'rpo_minutes': 30,
                'dependencies': ['Core Database', 'Web Services']
            },
            {
                'name': 'Reporting Service',
                'tier': 3,
                'rto_minutes': 240,
                'rpo_minutes': 60,
                'dependencies': ['Data Warehouse']
            }
        ]
        
        # 티어와 의존성을 기반으로 정렬
        return sorted(services, key=lambda x: (x['tier'], x['rto_minutes']))
    
    async def test_dr_readiness(self):
        """DR 준비 상태 테스트"""
        readiness_report = {
            'test_date': datetime.utcnow().isoformat(),
            'primary_site': self.primary_site,
            'dr_sites': [],
            'overall_readiness': 100.0,
            'issues': []
        }
        
        for dr_site in self.dr_sites:
            site_test = {
                'site_name': dr_site['name'],
                'location': dr_site['location'],
                'tests': []
            }
            
            # 연결성 테스트
            connectivity = await self.test_site_connectivity(dr_site)
            site_test['tests'].append({
                'test': 'Connectivity',
                'result': connectivity['status'],
                'latency_ms': connectivity['latency']
            })
            
            # 복제 지연 테스트
            replication = await self.test_replication_lag(dr_site)
            site_test['tests'].append({
                'test': 'Replication Lag',
                'result': 'PASS' if replication['lag_seconds'] < 60 else 'FAIL',
                'lag_seconds': replication['lag_seconds']
            })
            
            # 용량 테스트
            capacity = await self.test_site_capacity(dr_site)
            site_test['tests'].append({
                'test': 'Capacity',
                'result': 'PASS' if capacity['available_percent'] > 20 else 'FAIL',
                'available_percent': capacity['available_percent']
            })
            
            # 자동 장애 조치 테스트
            failover = await self.test_auto_failover(dr_site)
            site_test['tests'].append({
                'test': 'Auto Failover',
                'result': failover['status'],
                'time_seconds': failover['duration']
            })
            
            readiness_report['dr_sites'].append(site_test)
        
        # 전체 준비도 계산
        total_tests = sum(len(site['tests']) for site in readiness_report['dr_sites'])
        passed_tests = sum(
            1 for site in readiness_report['dr_sites'] 
            for test in site['tests'] 
            if test['result'] in ['PASS', 'SUCCESS']
        )
        
        readiness_report['overall_readiness'] = (passed_tests / total_tests) * 100
        
        return readiness_report
```

## 5. 클라우드 네이티브 HA/DR

### 5.1 쿠버네티스 기반 데이터베이스 HA

```yaml
# PostgreSQL HA on Kubernetes
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-ha-cluster
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "1GB"
      synchronous_commit: "on"
      wal_level: "replica"
      max_wal_senders: "10"
      max_replication_slots: "10"
      hot_standby: "on"
      
  bootstrap:
    initdb:
      database: production
      owner: dbadmin
      secret:
        name: postgres-credentials
        
  monitoring:
    enabled: true
    customQueriesConfigMap:
      - name: postgres-monitoring-queries
        key: queries.yaml
        
  backup:
    retentionPolicy: "7d"
    barmanObjectStore:
      destinationPath: "s3://postgres-backups/ha-cluster"
      s3Credentials:
        accessKeyId:
          name: backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
        maxParallel: 8
      data:
        compression: gzip
        immediateCheckpoint: false
        jobs: 2
        
  nodeMaintenanceWindow:
    inProgress: false
    reusePVC: true
    
  affinity:
    enablePodAntiAffinity: true
    topologyKey: kubernetes.io/hostname
    podAntiAffinityType: required
    
  resources:
    requests:
      memory: "1Gi"
      cpu: "1"
    limits:
      memory: "2Gi"
      cpu: "2"
      
  storage:
    size: 100Gi
    storageClass: fast-ssd
    
---
# 모니터링 쿼리 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-monitoring-queries
data:
  queries.yaml: |
    pg_replication:
      query: |
        SELECT 
          application_name,
          client_addr,
          state,
          sync_state,
          pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS sent_lag,
          pg_wal_lsn_diff(sent_lsn, flush_lsn) AS flush_lag,
          pg_wal_lsn_diff(flush_lsn, replay_lsn) AS replay_lag
        FROM pg_stat_replication
      metrics:
        - application_name:
            usage: "LABEL"
        - sent_lag:
            usage: "GAUGE"
        - flush_lag:
            usage: "GAUGE"
        - replay_lag:
            usage: "GAUGE"
```

### 5.2 멀티 클라우드 DR 오케스트레이션

```python
class MultiCloudDROrchestrator:
    """멀티 클라우드 DR 오케스트레이터"""
    
    def __init__(self):
        self.cloud_providers = {
            'aws': AWSProvider(),
            'azure': AzureProvider(),
            'gcp': GCPProvider()
        }
        self.active_provider = 'aws'
        self.dr_providers = ['azure', 'gcp']
        
    async def orchestrate_multi_cloud_dr(self):
        """멀티 클라우드 DR 오케스트레이션"""
        dr_status = {
            'primary': {
                'provider': self.active_provider,
                'status': 'ACTIVE',
                'health': await self.check_provider_health(self.active_provider)
            },
            'secondary': []
        }
        
        # 각 DR 사이트 동기화
        for provider in self.dr_providers:
            sync_status = await self.sync_to_dr_site(
                self.active_provider, 
                provider
            )
            
            dr_status['secondary'].append({
                'provider': provider,
                'status': 'STANDBY',
                'sync_status': sync_status,
                'health': await self.check_provider_health(provider)
            })
        
        # 자동 건강 상태 모니터링
        asyncio.create_task(self.monitor_multi_cloud_health())
        
        return dr_status
    
    async def execute_cross_cloud_failover(self, target_provider):
        """클라우드 간 장애 조치"""
        failover_plan = {
            'source': self.active_provider,
            'target': target_provider,
            'start_time': datetime.utcnow(),
            'steps': []
        }
        
        try:
            # 1. 소스 클라우드 쓰기 중지
            await self.cloud_providers[self.active_provider].enable_readonly_mode()
            failover_plan['steps'].append({
                'action': 'Enable read-only mode on source',
                'status': 'SUCCESS'
            })
            
            # 2. 최종 동기화
            final_sync = await self.perform_final_sync(
                self.active_provider, 
                target_provider
            )
            failover_plan['steps'].append({
                'action': 'Final data synchronization',
                'status': 'SUCCESS',
                'records_synced': final_sync['record_count']
            })
            
            # 3. DNS 전환
            await self.update_dns_records(target_provider)
            failover_plan['steps'].append({
                'action': 'DNS failover',
                'status': 'SUCCESS'
            })
            
            # 4. 타겟 클라우드 활성화
            await self.cloud_providers[target_provider].promote_to_primary()
            failover_plan['steps'].append({
                'action': 'Promote target to primary',
                'status': 'SUCCESS'
            })
            
            # 5. 애플리케이션 검증
            validation = await self.validate_applications(target_provider)
            failover_plan['steps'].append({
                'action': 'Application validation',
                'status': 'SUCCESS' if validation['all_healthy'] else 'WARNING',
                'details': validation
            })
            
            # 업데이트 활성 프로바이더
            self.active_provider = target_provider
            self.dr_providers = [p for p in self.cloud_providers.keys() 
                               if p != target_provider]
            
            failover_plan['status'] = 'SUCCESS'
            
        except Exception as e:
            failover_plan['status'] = 'FAILED'
            failover_plan['error'] = str(e)
            
            # 롤백 시도
            await self.rollback_failover(failover_plan)
        
        failover_plan['end_time'] = datetime.utcnow()
        failover_plan['duration'] = (
            failover_plan['end_time'] - failover_plan['start_time']
        ).total_seconds()
        
        return failover_plan
```

## 마무리

고가용성과 재해 복구는 현대 데이터베이스 시스템의 필수 요소입니다. Always On 가용성 그룹, 지역 간 복제, 자동 장애 조치, 그리고 멀티 클라우드 DR 전략을 통해 비즈니스 연속성을 보장할 수 있습니다. 정기적인 DR 훈련과 자동화된 모니터링을 통해 RTO와 RPO 목표를 달성하고 예기치 않은 장애에 대비할 수 있습니다.

다음 장에서는 데이터베이스 자동화와 DevOps 통합에 대해 학습하겠습니다.