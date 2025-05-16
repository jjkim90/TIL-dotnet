# 데이터베이스 모니터링과 성능 진단

## 개요

효과적인 데이터베이스 모니터링과 성능 진단은 안정적이고 고성능의 데이터베이스 시스템을 유지하는 데 필수적입니다. 이 장에서는 주요 모니터링 메트릭, 성능 진단 도구, 자동화된 모니터링 시스템 구축, 그리고 문제 해결 방법론을 학습합니다.

## 1. 핵심 모니터링 메트릭

### 1.1 시스템 레벨 메트릭

```sql
-- SQL Server 시스템 메트릭
-- 대기 통계 분석
WITH WaitStats AS (
    SELECT 
        wait_type,
        wait_time_ms,
        max_wait_time_ms,
        signal_wait_time_ms,
        waiting_tasks_count,
        CAST(wait_time_ms AS DECIMAL(20,2)) / 
            NULLIF(waiting_tasks_count, 0) AS avg_wait_time_ms,
        CAST(signal_wait_time_ms AS DECIMAL(20,2)) / 
            NULLIF(wait_time_ms, 0) * 100 AS signal_wait_percentage
    FROM sys.dm_os_wait_stats
    WHERE wait_type NOT IN (
        -- 제외할 대기 유형
        'BROKER_EVENTHANDLER', 'BROKER_RECEIVE_WAITFOR',
        'BROKER_TASK_STOP', 'BROKER_TO_FLUSH',
        'CHECKPOINT_QUEUE', 'CLR_AUTO_EVENT',
        'CLR_MANUAL_EVENT', 'CLR_SEMAPHORE',
        'DBMIRROR_DBM_EVENT', 'DBMIRROR_DBM_MUTEX',
        'DBMIRROR_EVENTS_QUEUE', 'DBMIRROR_WORKER_QUEUE',
        'DBMIRRORING_CMD', 'DIRTY_PAGE_POLL',
        'DISPATCHER_QUEUE_SEMAPHORE', 'FT_IFTS_SCHEDULER_IDLE_WAIT',
        'FT_IFTSHC_MUTEX', 'HADR_CLUSAPI_CALL',
        'HADR_FILESTREAM_IOMGR_IOCOMPLETION', 'HADR_LOGCAPTURE_WAIT',
        'HADR_NOTIFICATION_DEQUEUE', 'HADR_TIMER_TASK',
        'HADR_WORK_QUEUE', 'LAZYWRITER_SLEEP',
        'LOGMGR_QUEUE', 'ONDEMAND_TASK_QUEUE',
        'PREEMPTIVE_OS_LIBRARYOPS', 'PREEMPTIVE_OS_COMOPS',
        'PREEMPTIVE_OS_CRYPTOPS', 'PREEMPTIVE_OS_PIPEOPS',
        'PREEMPTIVE_OS_VERIFYTRUST', 'PREEMPTIVE_OS_WAITFORSINGLEOBJECT',
        'PREEMPTIVE_XE_DISPATCHER', 'PWAIT_ALL_COMPONENTS_INITIALIZED',
        'QDS_ASYNC_QUEUE', 'QDS_CLEANUP_STALE_QUERIES_TASK_MAIN_LOOP_SLEEP',
        'REQUEST_FOR_DEADLOCK_SEARCH', 'RESOURCE_QUEUE',
        'SERVER_IDLE_CHECK', 'SLEEP_BPOOL_FLUSH',
        'SLEEP_DBSTARTUP', 'SLEEP_DCOMSTARTUP',
        'SLEEP_MASTERDBREADY', 'SLEEP_MASTERMDREADY',
        'SLEEP_MASTERUPGRADED', 'SLEEP_MSDBSTARTUP',
        'SLEEP_SYSTEMTASK', 'SLEEP_TASK',
        'SP_SERVER_DIAGNOSTICS_SLEEP', 'SQLTRACE_BUFFER_FLUSH',
        'SQLTRACE_INCREMENTAL_FLUSH_SLEEP', 'SQLTRACE_WAIT_ENTRIES',
        'WAIT_FOR_RESULTS', 'WAITFOR',
        'WAITFOR_TASKSHUTDOWN', 'WAIT_XTP_HOST_WAIT',
        'WAIT_XTP_OFFLINE_CKPT_NEW_LOG', 'WAIT_XTP_CKPT_CLOSE',
        'XE_DISPATCHER_JOIN', 'XE_DISPATCHER_WAIT',
        'XE_TIMER_EVENT'
    )
)
SELECT TOP 20
    wait_type,
    waiting_tasks_count,
    CAST(wait_time_ms / 1000.0 AS DECIMAL(20,2)) AS wait_time_seconds,
    CAST(avg_wait_time_ms AS DECIMAL(20,2)) AS avg_wait_time_ms,
    CAST(max_wait_time_ms / 1000.0 AS DECIMAL(20,2)) AS max_wait_time_seconds,
    CAST(signal_wait_percentage AS DECIMAL(5,2)) AS signal_wait_pct,
    CASE 
        WHEN wait_type LIKE 'LCK%' THEN 'Locking'
        WHEN wait_type LIKE 'PAGEIO%' OR wait_type LIKE 'WRITELOG' THEN 'I/O'
        WHEN wait_type LIKE 'PAGELATCH%' THEN 'Buffer Latch'
        WHEN wait_type LIKE 'LATCH%' THEN 'Non-Buffer Latch'
        WHEN wait_type LIKE 'NETWORK%' THEN 'Network'
        WHEN wait_type LIKE 'OLEDB' THEN 'Linked Server'
        WHEN wait_type LIKE 'CXPACKET%' OR wait_type LIKE 'EXCHANGE' THEN 'Parallelism'
        ELSE 'Other'
    END AS wait_category
FROM WaitStats
WHERE waiting_tasks_count > 0
ORDER BY wait_time_ms DESC;

-- CPU 사용률 모니터링
DECLARE @cpu_usage TABLE (
    record_time DATETIME,
    sql_cpu_usage INT,
    system_idle INT,
    other_process_cpu INT
);

INSERT INTO @cpu_usage
SELECT 
    DATEADD(ms, -1 * (cpu_ticks / (cpu_ticks/ms_ticks)), GETDATE()) AS record_time,
    SQLProcessUtilization AS sql_cpu_usage,
    SystemIdle AS system_idle,
    100 - SystemIdle - SQLProcessUtilization AS other_process_cpu
FROM (
    SELECT 
        record.value('(./Record/@id)[1]', 'int') AS record_id,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
        timestamp
    FROM (
        SELECT 
            CONVERT(XML, record) AS record,
            timestamp
        FROM sys.dm_os_ring_buffers
        WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
        AND record LIKE '%<SystemHealth>%'
    ) AS x
) AS y
CROSS JOIN sys.dm_os_sys_info;

SELECT 
    record_time,
    sql_cpu_usage,
    system_idle,
    other_process_cpu,
    CASE 
        WHEN sql_cpu_usage > 80 THEN 'Critical'
        WHEN sql_cpu_usage > 60 THEN 'Warning'
        ELSE 'Normal'
    END AS cpu_status
FROM @cpu_usage
ORDER BY record_time DESC;
```

### 1.2 메모리 사용 분석

```sql
-- 메모리 사용 현황
SELECT 
    type AS memory_clerk_type,
    SUM(pages_kb) / 1024 AS memory_usage_mb,
    SUM(pages_kb) / 1024.0 / 1024.0 AS memory_usage_gb,
    CAST(SUM(pages_kb) * 100.0 / 
        (SELECT SUM(pages_kb) FROM sys.dm_os_memory_clerks) AS DECIMAL(5,2)) AS percentage,
    COUNT(*) AS clerk_count
FROM sys.dm_os_memory_clerks
WHERE pages_kb > 0
GROUP BY type
ORDER BY SUM(pages_kb) DESC;

-- 버퍼 풀 사용 분석
WITH BufferPoolUsage AS (
    SELECT 
        DB_NAME(database_id) AS database_name,
        COUNT(*) * 8 / 1024 AS buffer_pool_mb,
        COUNT(*) * 8 / 1024.0 / 1024.0 AS buffer_pool_gb
    FROM sys.dm_os_buffer_descriptors
    WHERE database_id > 4  -- 시스템 데이터베이스 제외
    GROUP BY database_id
)
SELECT 
    database_name,
    buffer_pool_mb,
    buffer_pool_gb,
    CAST(buffer_pool_mb * 100.0 / 
        (SELECT SUM(buffer_pool_mb) FROM BufferPoolUsage) AS DECIMAL(5,2)) AS percentage,
    REPLICATE('█', buffer_pool_mb * 50 / 
        (SELECT MAX(buffer_pool_mb) FROM BufferPoolUsage)) AS visual_bar
FROM BufferPoolUsage
ORDER BY buffer_pool_mb DESC;

-- 플랜 캐시 분석
SELECT 
    objtype AS cached_object_type,
    COUNT(*) AS plan_count,
    SUM(CAST(size_in_bytes AS BIGINT)) / 1024 / 1024 AS size_mb,
    AVG(usecounts) AS avg_use_count,
    MAX(usecounts) AS max_use_count,
    SUM(CASE WHEN usecounts = 1 THEN 1 ELSE 0 END) AS single_use_plans,
    CAST(SUM(CASE WHEN usecounts = 1 THEN 1 ELSE 0 END) * 100.0 / 
        COUNT(*) AS DECIMAL(5,2)) AS single_use_percentage
FROM sys.dm_exec_cached_plans
GROUP BY objtype
ORDER BY size_mb DESC;
```

### 1.3 I/O 성능 모니터링

```python
import psutil
import time
from collections import deque
import matplotlib.pyplot as plt
import pandas as pd

class IOPerformanceMonitor:
    """I/O 성능 모니터"""
    
    def __init__(self, sample_interval=1, history_size=300):
        self.sample_interval = sample_interval
        self.history_size = history_size
        self.io_history = deque(maxlen=history_size)
        self.disk_partitions = psutil.disk_partitions()
        
    def collect_io_metrics(self):
        """I/O 메트릭 수집"""
        # 디스크 I/O 통계
        disk_io = psutil.disk_io_counters(perdisk=True)
        
        metrics = {
            'timestamp': time.time(),
            'disks': {}
        }
        
        for disk_name, counters in disk_io.items():
            metrics['disks'][disk_name] = {
                'read_bytes': counters.read_bytes,
                'write_bytes': counters.write_bytes,
                'read_count': counters.read_count,
                'write_count': counters.write_count,
                'read_time': counters.read_time,
                'write_time': counters.write_time
            }
        
        # 이전 샘플과 비교하여 속도 계산
        if self.io_history:
            prev_metrics = self.io_history[-1]
            time_delta = metrics['timestamp'] - prev_metrics['timestamp']
            
            for disk_name in metrics['disks']:
                if disk_name in prev_metrics['disks']:
                    prev = prev_metrics['disks'][disk_name]
                    curr = metrics['disks'][disk_name]
                    
                    # 초당 I/O 계산
                    metrics['disks'][disk_name]['read_mbps'] = (
                        (curr['read_bytes'] - prev['read_bytes']) / 
                        time_delta / 1024 / 1024
                    )
                    metrics['disks'][disk_name]['write_mbps'] = (
                        (curr['write_bytes'] - prev['write_bytes']) / 
                        time_delta / 1024 / 1024
                    )
                    metrics['disks'][disk_name]['read_iops'] = (
                        (curr['read_count'] - prev['read_count']) / time_delta
                    )
                    metrics['disks'][disk_name]['write_iops'] = (
                        (curr['write_count'] - prev['write_count']) / time_delta
                    )
                    
                    # 평균 레이턴시 계산
                    read_count_delta = curr['read_count'] - prev['read_count']
                    write_count_delta = curr['write_count'] - prev['write_count']
                    
                    if read_count_delta > 0:
                        metrics['disks'][disk_name]['avg_read_latency'] = (
                            (curr['read_time'] - prev['read_time']) / 
                            read_count_delta
                        )
                    if write_count_delta > 0:
                        metrics['disks'][disk_name]['avg_write_latency'] = (
                            (curr['write_time'] - prev['write_time']) / 
                            write_count_delta
                        )
        
        self.io_history.append(metrics)
        return metrics
    
    def analyze_io_patterns(self):
        """I/O 패턴 분석"""
        if len(self.io_history) < 10:
            return None
        
        analysis = {}
        
        for disk_name in self.io_history[-1]['disks']:
            disk_data = []
            
            for metrics in self.io_history:
                if disk_name in metrics['disks'] and 'read_mbps' in metrics['disks'][disk_name]:
                    disk_data.append({
                        'timestamp': metrics['timestamp'],
                        'read_mbps': metrics['disks'][disk_name].get('read_mbps', 0),
                        'write_mbps': metrics['disks'][disk_name].get('write_mbps', 0),
                        'read_iops': metrics['disks'][disk_name].get('read_iops', 0),
                        'write_iops': metrics['disks'][disk_name].get('write_iops', 0)
                    })
            
            if disk_data:
                df = pd.DataFrame(disk_data)
                
                analysis[disk_name] = {
                    'avg_read_mbps': df['read_mbps'].mean(),
                    'max_read_mbps': df['read_mbps'].max(),
                    'avg_write_mbps': df['write_mbps'].mean(),
                    'max_write_mbps': df['write_mbps'].max(),
                    'avg_total_iops': (df['read_iops'] + df['write_iops']).mean(),
                    'max_total_iops': (df['read_iops'] + df['write_iops']).max(),
                    'read_write_ratio': df['read_mbps'].sum() / 
                                      (df['write_mbps'].sum() + 0.001)
                }
                
                # I/O 스파이크 감지
                total_mbps = df['read_mbps'] + df['write_mbps']
                mean_mbps = total_mbps.mean()
                std_mbps = total_mbps.std()
                
                spikes = total_mbps[total_mbps > mean_mbps + 2 * std_mbps]
                analysis[disk_name]['spike_count'] = len(spikes)
                analysis[disk_name]['spike_percentage'] = len(spikes) / len(df) * 100
        
        return analysis
```

## 2. 쿼리 성능 분석

### 2.1 쿼리 실행 계획 분석

```sql
-- 비용이 높은 쿼리 식별
WITH QueryStats AS (
    SELECT 
        qs.sql_handle,
        qs.statement_start_offset,
        qs.statement_end_offset,
        qs.plan_handle,
        qs.creation_time,
        qs.last_execution_time,
        qs.execution_count,
        qs.total_worker_time,
        qs.total_physical_reads,
        qs.total_logical_reads,
        qs.total_logical_writes,
        qs.total_elapsed_time,
        qs.total_rows,
        qs.total_dop,
        qs.total_grant_kb,
        qs.total_spills,
        -- 평균값 계산
        qs.total_worker_time / NULLIF(qs.execution_count, 0) AS avg_cpu_time,
        qs.total_elapsed_time / NULLIF(qs.execution_count, 0) AS avg_duration,
        qs.total_logical_reads / NULLIF(qs.execution_count, 0) AS avg_logical_reads,
        qs.total_physical_reads / NULLIF(qs.execution_count, 0) AS avg_physical_reads,
        qs.total_rows / NULLIF(qs.execution_count, 0) AS avg_rows
    FROM sys.dm_exec_query_stats qs
    WHERE qs.last_execution_time > DATEADD(HOUR, -24, GETDATE())  -- 최근 24시간
)
SELECT TOP 20
    DB_NAME(qt.dbid) AS database_name,
    OBJECT_NAME(qt.objectid, qt.dbid) AS object_name,
    qt.text AS query_text,
    SUBSTRING(qt.text, 
        (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS statement_text,
    qs.execution_count,
    CAST(qs.total_worker_time / 1000000.0 AS DECIMAL(10,2)) AS total_cpu_seconds,
    CAST(qs.avg_cpu_time / 1000.0 AS DECIMAL(10,2)) AS avg_cpu_ms,
    CAST(qs.total_elapsed_time / 1000000.0 AS DECIMAL(10,2)) AS total_duration_seconds,
    CAST(qs.avg_duration / 1000.0 AS DECIMAL(10,2)) AS avg_duration_ms,
    qs.total_logical_reads,
    qs.avg_logical_reads,
    qs.total_physical_reads,
    qs.avg_physical_reads,
    qs.total_rows,
    qs.avg_rows,
    qs.total_spills,
    qs.total_grant_kb / 1024 AS total_grant_mb,
    qs.creation_time,
    qs.last_execution_time,
    qp.query_plan,
    -- 성능 점수 (가중치 적용)
    (qs.total_worker_time * 0.4 + 
     qs.total_logical_reads * 0.3 + 
     qs.total_elapsed_time * 0.2 + 
     qs.total_physical_reads * 0.1) / NULLIF(qs.execution_count, 0) AS performance_score
FROM QueryStats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qt.text NOT LIKE '%sys.%'  -- 시스템 쿼리 제외
ORDER BY performance_score DESC;

-- 누락된 인덱스 분석
SELECT 
    DB_NAME(mid.database_id) AS database_name,
    OBJECT_NAME(mid.object_id, mid.database_id) AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.unique_compiles,
    migs.user_seeks,
    migs.user_scans,
    migs.avg_total_user_cost,
    migs.avg_user_impact,
    -- 인덱스 효과 점수
    CAST(migs.user_seeks * migs.avg_total_user_cost * migs.avg_user_impact AS BIGINT) AS impact_score,
    -- 제안된 인덱스 생성문
    'CREATE INDEX [IX_' + 
        OBJECT_NAME(mid.object_id, mid.database_id) + '_' + 
        REPLACE(REPLACE(REPLACE(ISNULL(mid.equality_columns, ''), ', ', '_'), '[', ''), ']', '') + 
        CASE 
            WHEN mid.inequality_columns IS NOT NULL 
            THEN '_' + REPLACE(REPLACE(REPLACE(mid.inequality_columns, ', ', '_'), '[', ''), ']', '')
            ELSE ''
        END + '] ON ' + 
        QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(mid.object_id, mid.database_id)) + 
        ' (' + ISNULL(mid.equality_columns, '') + 
        CASE 
            WHEN mid.inequality_columns IS NOT NULL 
            THEN CASE WHEN mid.equality_columns IS NOT NULL THEN ', ' ELSE '' END + mid.inequality_columns
            ELSE ''
        END + ')' +
        CASE 
            WHEN mid.included_columns IS NOT NULL 
            THEN ' INCLUDE (' + mid.included_columns + ')'
            ELSE ''
        END AS create_index_statement
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.objects o ON mid.object_id = o.object_id
WHERE mid.database_id = DB_ID()
    AND migs.user_seeks > 100  -- 최소 검색 횟수
    AND migs.avg_user_impact > 30  -- 최소 30% 개선 효과
ORDER BY impact_score DESC;
```

### 2.2 실행 계획 자동 분석

```python
import xml.etree.ElementTree as ET
import re
from typing import Dict, List, Tuple

class ExecutionPlanAnalyzer:
    """실행 계획 분석기"""
    
    def __init__(self):
        self.warning_patterns = {
            'missing_index': 'Missing Index',
            'key_lookup': 'Key Lookup',
            'scan_operation': 'Scan',
            'sort_operation': 'Sort',
            'hash_operation': 'Hash Match',
            'parallelism': 'Parallelism',
            'spool': 'Spool',
            'conversion': 'CONVERT_IMPLICIT'
        }
        
    def analyze_plan(self, query_plan_xml: str) -> Dict:
        """실행 계획 분석"""
        try:
            root = ET.fromstring(query_plan_xml)
            
            analysis = {
                'warnings': [],
                'expensive_operations': [],
                'missing_indexes': [],
                'statistics': {},
                'recommendations': []
            }
            
            # 네임스페이스 정의
            ns = {'ns': 'http://schemas.microsoft.com/sqlserver/2004/07/showplan'}
            
            # 비용 분석
            self._analyze_costs(root, ns, analysis)
            
            # 연산자 분석
            self._analyze_operators(root, ns, analysis)
            
            # 누락된 인덱스
            self._find_missing_indexes(root, ns, analysis)
            
            # 경고 찾기
            self._find_warnings(root, ns, analysis)
            
            # 권장사항 생성
            self._generate_recommendations(analysis)
            
            return analysis
            
        except Exception as e:
            return {'error': str(e)}
    
    def _analyze_costs(self, root, ns, analysis):
        """비용 분석"""
        # 전체 비용
        stmt_simple = root.find('.//ns:StmtSimple', ns)
        if stmt_simple is not None:
            analysis['statistics']['total_cost'] = float(
                stmt_simple.get('StatementSubTreeCost', 0)
            )
            analysis['statistics']['compile_time'] = float(
                stmt_simple.get('CompileTime', 0)
            )
            analysis['statistics']['compile_cpu'] = float(
                stmt_simple.get('CompileCPU', 0)
            )
        
        # 연산자별 비용
        operators = root.findall('.//ns:RelOp', ns)
        total_cost = 0
        operator_costs = []
        
        for op in operators:
            cost = float(op.get('EstimatedTotalSubtreeCost', 0))
            operator_costs.append({
                'type': op.get('PhysicalOp'),
                'cost': cost,
                'percentage': 0  # 나중에 계산
            })
            total_cost = max(total_cost, cost)
        
        # 비용 백분율 계산
        for op_cost in operator_costs:
            if total_cost > 0:
                op_cost['percentage'] = (op_cost['cost'] / total_cost) * 100
        
        # 가장 비싼 연산자들
        analysis['expensive_operations'] = sorted(
            operator_costs, 
            key=lambda x: x['cost'], 
            reverse=True
        )[:5]
    
    def _analyze_operators(self, root, ns, analysis):
        """연산자 분석"""
        operator_counts = {}
        problematic_ops = []
        
        operators = root.findall('.//ns:RelOp', ns)
        
        for op in operators:
            op_type = op.get('PhysicalOp')
            logical_op = op.get('LogicalOp')
            
            # 연산자 수 집계
            operator_counts[op_type] = operator_counts.get(op_type, 0) + 1
            
            # 문제가 될 수 있는 연산자 식별
            if 'Scan' in op_type and op_type != 'Constant Scan':
                table_scan = op.find('.//ns:TableScan', ns)
                if table_scan is not None:
                    object_elem = table_scan.find('.//ns:Object', ns)
                    if object_elem is not None:
                        problematic_ops.append({
                            'type': 'Table Scan',
                            'object': object_elem.get('Table', 'Unknown'),
                            'impact': 'High',
                            'rows': float(op.get('EstimateRows', 0))
                        })
            
            elif op_type == 'Key Lookup':
                problematic_ops.append({
                    'type': 'Key Lookup',
                    'object': self._get_object_name(op, ns),
                    'impact': 'Medium',
                    'rows': float(op.get('EstimateRows', 0))
                })
            
            elif op_type == 'Sort':
                memory_grant = op.find('.//ns:MemoryGrant', ns)
                if memory_grant is not None:
                    problematic_ops.append({
                        'type': 'Sort with Memory Grant',
                        'memory_kb': int(memory_grant.get('SerialRequiredMemory', 0)),
                        'impact': 'Medium',
                        'rows': float(op.get('EstimateRows', 0))
                    })
        
        analysis['statistics']['operator_counts'] = operator_counts
        analysis['warnings'].extend(problematic_ops)
    
    def _find_missing_indexes(self, root, ns, analysis):
        """누락된 인덱스 찾기"""
        missing_indexes = root.findall('.//ns:MissingIndex', ns)
        
        for idx in missing_indexes:
            index_info = {
                'impact': float(idx.get('Impact', 0)),
                'database': idx.get('Database'),
                'schema': idx.get('Schema'),
                'table': idx.get('Table'),
                'columns': []
            }
            
            # 컬럼 정보
            for column_group in idx.findall('.//ns:ColumnGroup', ns):
                usage = column_group.get('Usage')
                columns = []
                for col in column_group.findall('.//ns:Column', ns):
                    columns.append(col.get('Name'))
                
                index_info['columns'].append({
                    'usage': usage,
                    'columns': columns
                })
            
            analysis['missing_indexes'].append(index_info)
    
    def _generate_recommendations(self, analysis):
        """권장사항 생성"""
        recommendations = []
        
        # 누락된 인덱스
        for idx in analysis['missing_indexes']:
            if idx['impact'] > 50:
                recommendations.append({
                    'priority': 'High',
                    'type': 'Missing Index',
                    'description': f"Consider creating index on {idx['table']} with columns: {idx['columns']}",
                    'expected_improvement': f"{idx['impact']:.0f}%"
                })
        
        # 테이블 스캔
        for warning in analysis['warnings']:
            if warning['type'] == 'Table Scan' and warning['rows'] > 1000:
                recommendations.append({
                    'priority': 'High',
                    'type': 'Table Scan',
                    'description': f"Table scan detected on {warning['object']} affecting {warning['rows']:.0f} rows",
                    'suggestion': 'Consider adding appropriate index'
                })
        
        # 비싼 연산자
        for op in analysis['expensive_operations']:
            if op['percentage'] > 30:
                recommendations.append({
                    'priority': 'Medium',
                    'type': 'Expensive Operation',
                    'description': f"{op['type']} operation consuming {op['percentage']:.1f}% of query cost",
                    'suggestion': 'Review query logic or consider index optimization'
                })
        
        analysis['recommendations'] = sorted(
            recommendations, 
            key=lambda x: {'High': 0, 'Medium': 1, 'Low': 2}[x['priority']]
        )
```

## 3. 자동화된 모니터링 시스템

### 3.1 통합 모니터링 대시보드

```python
import asyncio
import aiohttp
from influxdb import InfluxDBClient
import json
from datetime import datetime
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class DatabaseMonitoringSystem:
    """통합 데이터베이스 모니터링 시스템"""
    
    def __init__(self, config):
        self.config = config
        self.influx_client = InfluxDBClient(
            host=config['influxdb']['host'],
            port=config['influxdb']['port'],
            database=config['influxdb']['database']
        )
        self.alert_rules = self._load_alert_rules()
        self.alert_history = {}
        
    async def collect_all_metrics(self):
        """모든 메트릭 수집"""
        tasks = [
            self.collect_system_metrics(),
            self.collect_query_metrics(),
            self.collect_replication_metrics(),
            self.collect_connection_metrics(),
            self.collect_lock_metrics()
        ]
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # InfluxDB에 저장
        points = []
        for result in results:
            if isinstance(result, list):
                points.extend(result)
        
        if points:
            self.influx_client.write_points(points)
    
    async def collect_system_metrics(self):
        """시스템 메트릭 수집"""
        metrics = []
        
        # CPU 사용률
        cpu_query = """
        SELECT 
            SQLProcessUtilization as sql_cpu,
            SystemIdle as system_idle,
            100 - SystemIdle - SQLProcessUtilization as other_cpu
        FROM (
            SELECT TOP 1
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle
            FROM (
                SELECT CONVERT(XML, record) AS record
                FROM sys.dm_os_ring_buffers
                WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
                AND record LIKE '%<SystemHealth>%'
            ) AS x
            ORDER BY record.value('(./Record/@id)[1]', 'int') DESC
        ) AS y
        """
        
        # 메모리 사용률
        memory_query = """
        SELECT 
            total_physical_memory_kb / 1024 AS total_memory_mb,
            available_physical_memory_kb / 1024 AS available_memory_mb,
            total_page_file_kb / 1024 AS total_page_file_mb,
            available_page_file_kb / 1024 AS available_page_file_mb,
            system_memory_state_desc
        FROM sys.dm_os_sys_memory
        """
        
        # 결과를 InfluxDB 포맷으로 변환
        timestamp = datetime.utcnow().isoformat() + 'Z'
        
        metrics.append({
            'measurement': 'system_metrics',
            'time': timestamp,
            'fields': {
                'cpu_usage': 75.5,  # 실제 쿼리 결과로 대체
                'memory_usage_mb': 8192,
                'available_memory_mb': 2048
            },
            'tags': {
                'server': self.config['server_name'],
                'environment': self.config['environment']
            }
        })
        
        return metrics
    
    def _load_alert_rules(self):
        """알림 규칙 로드"""
        return {
            'cpu_high': {
                'metric': 'cpu_usage',
                'threshold': 80,
                'duration': 300,  # 5분
                'severity': 'warning',
                'message': 'CPU usage above {threshold}% for {duration} seconds'
            },
            'memory_low': {
                'metric': 'available_memory_mb',
                'threshold': 1024,
                'operator': '<',
                'duration': 180,
                'severity': 'critical',
                'message': 'Available memory below {threshold}MB'
            },
            'long_running_query': {
                'metric': 'query_duration_ms',
                'threshold': 30000,  # 30초
                'severity': 'warning',
                'message': 'Query running longer than {threshold}ms'
            },
            'blocked_processes': {
                'metric': 'blocked_process_count',
                'threshold': 5,
                'duration': 60,
                'severity': 'critical',
                'message': 'More than {threshold} blocked processes'
            }
        }
    
    async def check_alerts(self):
        """알림 규칙 확인"""
        current_time = datetime.utcnow()
        
        for rule_name, rule in self.alert_rules.items():
            # 최근 메트릭 조회
            query = f"""
            SELECT MEAN("{rule['metric']}") 
            FROM system_metrics 
            WHERE time > now() - {rule.get('duration', 60)}s 
            GROUP BY time(30s)
            """
            
            result = self.influx_client.query(query)
            
            if self._evaluate_alert_condition(result, rule):
                await self._trigger_alert(rule_name, rule, current_time)
            else:
                # 알림 해제
                if rule_name in self.alert_history:
                    await self._clear_alert(rule_name)
    
    async def _trigger_alert(self, rule_name, rule, timestamp):
        """알림 발생"""
        if rule_name not in self.alert_history:
            self.alert_history[rule_name] = {
                'first_triggered': timestamp,
                'count': 0
            }
        
        self.alert_history[rule_name]['count'] += 1
        self.alert_history[rule_name]['last_triggered'] = timestamp
        
        # 알림 전송
        await self._send_alert_notification(rule_name, rule)
    
    async def _send_alert_notification(self, rule_name, rule):
        """알림 전송"""
        # 이메일 알림
        if self.config.get('email_alerts_enabled'):
            msg = MIMEMultipart()
            msg['From'] = self.config['email']['from']
            msg['To'] = ', '.join(self.config['email']['to'])
            msg['Subject'] = f"[{rule['severity'].upper()}] Database Alert: {rule_name}"
            
            body = f"""
            Alert: {rule_name}
            Severity: {rule['severity']}
            Message: {rule['message'].format(**rule)}
            Time: {datetime.utcnow().isoformat()}
            Server: {self.config['server_name']}
            Environment: {self.config['environment']}
            
            Alert History:
            First Triggered: {self.alert_history[rule_name]['first_triggered']}
            Trigger Count: {self.alert_history[rule_name]['count']}
            """
            
            msg.attach(MIMEText(body, 'plain'))
            
            # SMTP 서버로 전송
            with smtplib.SMTP(self.config['email']['smtp_server']) as server:
                server.send_message(msg)
        
        # Slack 알림
        if self.config.get('slack_alerts_enabled'):
            webhook_url = self.config['slack']['webhook_url']
            
            slack_message = {
                'text': f"Database Alert: {rule_name}",
                'attachments': [{
                    'color': 'danger' if rule['severity'] == 'critical' else 'warning',
                    'fields': [
                        {'title': 'Severity', 'value': rule['severity'], 'short': True},
                        {'title': 'Server', 'value': self.config['server_name'], 'short': True},
                        {'title': 'Message', 'value': rule['message'].format(**rule)},
                        {'title': 'Time', 'value': datetime.utcnow().isoformat()}
                    ]
                }]
            }
            
            async with aiohttp.ClientSession() as session:
                await session.post(webhook_url, json=slack_message)
```

## 4. 성능 문제 진단 및 해결

### 4.1 자동 성능 진단 시스템

```python
class PerformanceDiagnosticSystem:
    """성능 진단 시스템"""
    
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.diagnostics = []
        
    def run_full_diagnostics(self):
        """전체 진단 실행"""
        self.diagnostics = []
        
        # 진단 항목들
        self.check_cpu_pressure()
        self.check_memory_pressure()
        self.check_io_bottlenecks()
        self.check_blocking_chains()
        self.check_deadlocks()
        self.check_query_performance()
        self.check_index_fragmentation()
        self.check_statistics_staleness()
        
        return self.generate_report()
    
    def check_cpu_pressure(self):
        """CPU 압박 확인"""
        cpu_check_query = """
        WITH CPU_Usage AS (
            SELECT 
                SQLProcessUtilization,
                SystemIdle,
                100 - SystemIdle - SQLProcessUtilization AS OtherProcessUtilization,
                DATEADD(ms, -1 * (cpu_ticks / (cpu_ticks/ms_ticks)), GETDATE()) AS EventTime
            FROM (
                SELECT 
                    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
                    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
                    timestamp
                FROM (
                    SELECT CONVERT(XML, record) AS record, timestamp
                    FROM sys.dm_os_ring_buffers
                    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
                    AND record LIKE '%<SystemHealth>%'
                ) AS x
            ) AS y
            CROSS JOIN sys.dm_os_sys_info
        )
        SELECT 
            AVG(SQLProcessUtilization) AS avg_sql_cpu,
            MAX(SQLProcessUtilization) AS max_sql_cpu,
            AVG(SystemIdle) AS avg_idle
        FROM CPU_Usage
        WHERE EventTime > DATEADD(MINUTE, -5, GETDATE())
        """
        
        # 진단 결과 분석
        result = self._execute_query(cpu_check_query)
        
        if result and result[0]['avg_sql_cpu'] > 80:
            self.diagnostics.append({
                'category': 'CPU',
                'severity': 'High',
                'issue': 'High CPU Usage',
                'details': f"Average SQL CPU: {result[0]['avg_sql_cpu']}%",
                'recommendations': [
                    'Identify and optimize CPU-intensive queries',
                    'Consider query parallelism settings (MAXDOP)',
                    'Review index strategy to reduce CPU overhead',
                    'Consider scaling up CPU resources'
                ]
            })
    
    def check_blocking_chains(self):
        """블로킹 체인 확인"""
        blocking_query = """
        WITH BlockingChain AS (
            SELECT 
                s1.session_id AS blocked_session,
                s1.blocking_session_id AS blocking_session,
                s1.wait_type,
                s1.wait_time,
                s1.wait_resource,
                t1.text AS blocked_query,
                t2.text AS blocking_query,
                s1.program_name AS blocked_program,
                s2.program_name AS blocking_program,
                s1.login_name AS blocked_login,
                s2.login_name AS blocking_login,
                0 AS level
            FROM sys.dm_exec_sessions s1
            JOIN sys.dm_exec_requests r1 ON s1.session_id = r1.session_id
            CROSS APPLY sys.dm_exec_sql_text(r1.sql_handle) t1
            LEFT JOIN sys.dm_exec_sessions s2 ON s1.blocking_session_id = s2.session_id
            LEFT JOIN sys.dm_exec_requests r2 ON s2.session_id = r2.session_id
            OUTER APPLY sys.dm_exec_sql_text(r2.sql_handle) t2
            WHERE s1.blocking_session_id > 0
            
            UNION ALL
            
            SELECT 
                s.session_id,
                s.blocking_session_id,
                s.wait_type,
                s.wait_time,
                s.wait_resource,
                t.text,
                NULL,
                s.program_name,
                NULL,
                s.login_name,
                NULL,
                bc.level + 1
            FROM sys.dm_exec_sessions s
            JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
            CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
            JOIN BlockingChain bc ON s.blocking_session_id = bc.blocked_session
            WHERE bc.level < 10  -- 최대 10 레벨까지
        )
        SELECT * FROM BlockingChain
        ORDER BY level, blocking_session, blocked_session
        """
        
        blocking_chains = self._execute_query(blocking_query)
        
        if blocking_chains:
            self.diagnostics.append({
                'category': 'Blocking',
                'severity': 'High',
                'issue': 'Active Blocking Chains Detected',
                'details': f"Found {len(blocking_chains)} blocked sessions",
                'recommendations': [
                    'Review and optimize long-running transactions',
                    'Consider READ_COMMITTED_SNAPSHOT isolation level',
                    'Implement query timeout settings',
                    'Review lock escalation settings'
                ],
                'affected_sessions': blocking_chains
            })
    
    def generate_report(self):
        """진단 리포트 생성"""
        report = {
            'timestamp': datetime.utcnow().isoformat(),
            'server': self.connection_string.split(';')[0],
            'total_issues': len(self.diagnostics),
            'critical_issues': len([d for d in self.diagnostics if d['severity'] == 'Critical']),
            'high_issues': len([d for d in self.diagnostics if d['severity'] == 'High']),
            'diagnostics': self.diagnostics,
            'summary': self._generate_summary()
        }
        
        return report
    
    def _generate_summary(self):
        """요약 생성"""
        if not self.diagnostics:
            return "No significant performance issues detected."
        
        summary_parts = []
        
        # 카테고리별 집계
        categories = {}
        for diag in self.diagnostics:
            cat = diag['category']
            if cat not in categories:
                categories[cat] = []
            categories[cat].append(diag)
        
        for cat, issues in categories.items():
            summary_parts.append(
                f"{cat}: {len(issues)} issue(s) found"
            )
        
        return "; ".join(summary_parts)
```

## 마무리

효과적인 데이터베이스 모니터링과 성능 진단은 안정적인 시스템 운영의 핵심입니다. 자동화된 모니터링 시스템을 구축하고, 주요 메트릭을 지속적으로 추적하며, 문제를 사전에 감지하고 해결하는 것이 중요합니다. 정기적인 성능 분석과 최적화를 통해 데이터베이스 시스템의 성능을 최상의 상태로 유지할 수 있습니다.

다음 장에서는 데이터베이스 고가용성과 재해 복구 전략에 대해 학습하겠습니다.