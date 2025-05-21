# 데이터베이스 자동화와 DevOps

## 개요

데이터베이스 DevOps는 개발과 운영의 경계를 허물고, 지속적인 통합과 배포(CI/CD)를 데이터베이스 영역으로 확장하는 것입니다. 이 장에서는 데이터베이스 버전 관리, 자동화된 배포 파이프라인, 인프라스트럭처 코드(IaC), 그리고 데이터베이스 테스트 자동화에 대해 학습합니다.

## 1. 데이터베이스 버전 관리

### 1.1 마이그레이션 기반 버전 관리

```python
import hashlib
import os
from datetime import datetime
import json
import sqlalchemy
from sqlalchemy import create_engine, text

class DatabaseMigrationSystem:
    """데이터베이스 마이그레이션 시스템"""
    
    def __init__(self, connection_string, migrations_path='./migrations'):
        self.engine = create_engine(connection_string)
        self.migrations_path = migrations_path
        self.migration_table = 'schema_migrations'
        self._ensure_migration_table()
        
    def _ensure_migration_table(self):
        """마이그레이션 테이블 생성"""
        create_table_sql = f"""
        CREATE TABLE IF NOT EXISTS {self.migration_table} (
            version VARCHAR(255) NOT NULL PRIMARY KEY,
            migration_name VARCHAR(500) NOT NULL,
            applied_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            checksum VARCHAR(64) NOT NULL,
            execution_time_ms INT,
            applied_by VARCHAR(255),
            success BOOLEAN DEFAULT TRUE,
            error_message TEXT
        );
        """
        
        with self.engine.connect() as conn:
            conn.execute(text(create_table_sql))
            conn.commit()
    
    def generate_migration(self, name, description=""):
        """새 마이그레이션 파일 생성"""
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        version = f"V{timestamp}__{name.replace(' ', '_')}"
        
        migration_template = f'''-- Migration: {version}
-- Description: {description}
-- Author: {os.environ.get('USER', 'unknown')}
-- Date: {datetime.now().isoformat()}

-- ===== UP MIGRATION =====
-- TODO: Add your forward migration SQL here

-- Example:
-- CREATE TABLE users (
--     id SERIAL PRIMARY KEY,
--     username VARCHAR(255) NOT NULL UNIQUE,
--     email VARCHAR(255) NOT NULL,
--     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
-- );

-- ===== DOWN MIGRATION =====
-- TODO: Add your rollback SQL here

-- Example:
-- DROP TABLE IF EXISTS users;
'''
        
        up_file = os.path.join(self.migrations_path, f"{version}.up.sql")
        down_file = os.path.join(self.migrations_path, f"{version}.down.sql")
        
        # UP 마이그레이션
        with open(up_file, 'w') as f:
            f.write(migration_template)
            
        # DOWN 마이그레이션 (롤백용)
        rollback_template = f'''-- Rollback Migration: {version}
-- Description: Rollback for {description}

-- TODO: Add rollback SQL here
'''
        
        with open(down_file, 'w') as f:
            f.write(rollback_template)
            
        return version
    
    def get_pending_migrations(self):
        """적용되지 않은 마이그레이션 목록 조회"""
        # 적용된 마이그레이션 조회
        with self.engine.connect() as conn:
            result = conn.execute(
                text(f"SELECT version FROM {self.migration_table} WHERE success = TRUE")
            )
            applied_versions = {row[0] for row in result}
        
        # 파일 시스템에서 마이그레이션 파일 검색
        all_migrations = []
        for filename in sorted(os.listdir(self.migrations_path)):
            if filename.endswith('.up.sql'):
                version = filename.replace('.up.sql', '')
                if version not in applied_versions:
                    all_migrations.append({
                        'version': version,
                        'filename': filename,
                        'path': os.path.join(self.migrations_path, filename)
                    })
        
        return all_migrations
    
    def apply_migration(self, migration):
        """단일 마이그레이션 적용"""
        start_time = datetime.now()
        
        try:
            # 마이그레이션 파일 읽기
            with open(migration['path'], 'r') as f:
                migration_sql = f.read()
            
            # 체크섬 계산
            checksum = hashlib.sha256(migration_sql.encode()).hexdigest()
            
            # 트랜잭션 내에서 실행
            with self.engine.begin() as conn:
                # 마이그레이션 실행
                conn.execute(text(migration_sql))
                
                # 성공 기록
                execution_time = int((datetime.now() - start_time).total_seconds() * 1000)
                
                conn.execute(text(f"""
                    INSERT INTO {self.migration_table} 
                    (version, migration_name, checksum, execution_time_ms, applied_by)
                    VALUES (:version, :name, :checksum, :exec_time, :user)
                """), {
                    'version': migration['version'],
                    'name': migration['filename'],
                    'checksum': checksum,
                    'exec_time': execution_time,
                    'user': os.environ.get('USER', 'system')
                })
                
            print(f"✓ Applied migration: {migration['version']}")
            return True
            
        except Exception as e:
            # 실패 기록
            with self.engine.connect() as conn:
                conn.execute(text(f"""
                    INSERT INTO {self.migration_table} 
                    (version, migration_name, checksum, applied_by, success, error_message)
                    VALUES (:version, :name, :checksum, :user, FALSE, :error)
                """), {
                    'version': migration['version'],
                    'name': migration['filename'],
                    'checksum': 'N/A',
                    'user': os.environ.get('USER', 'system'),
                    'error': str(e)
                })
                conn.commit()
                
            print(f"✗ Failed to apply migration: {migration['version']}")
            print(f"  Error: {e}")
            raise
    
    def migrate(self, target_version=None):
        """데이터베이스 마이그레이션 실행"""
        pending = self.get_pending_migrations()
        
        if not pending:
            print("Database is up to date!")
            return
        
        print(f"Found {len(pending)} pending migrations")
        
        for migration in pending:
            if target_version and migration['version'] > target_version:
                break
                
            self.apply_migration(migration)
        
        print("Migration completed successfully!")
    
    def rollback(self, target_version):
        """특정 버전으로 롤백"""
        # 현재 적용된 마이그레이션을 역순으로 조회
        with self.engine.connect() as conn:
            result = conn.execute(text(f"""
                SELECT version, migration_name 
                FROM {self.migration_table} 
                WHERE success = TRUE AND version > :target
                ORDER BY version DESC
            """), {'target': target_version})
            
            migrations_to_rollback = list(result)
        
        for version, name in migrations_to_rollback:
            down_file = os.path.join(
                self.migrations_path, 
                f"{version}.down.sql"
            )
            
            if os.path.exists(down_file):
                print(f"Rolling back: {version}")
                
                with open(down_file, 'r') as f:
                    rollback_sql = f.read()
                
                try:
                    with self.engine.begin() as conn:
                        conn.execute(text(rollback_sql))
                        
                        # 마이그레이션 기록 삭제
                        conn.execute(text(f"""
                            DELETE FROM {self.migration_table} 
                            WHERE version = :version
                        """), {'version': version})
                        
                    print(f"✓ Rolled back: {version}")
                    
                except Exception as e:
                    print(f"✗ Rollback failed for {version}: {e}")
                    raise
            else:
                print(f"⚠ No rollback script found for {version}")
```

### 1.2 스키마 비교 및 동기화

```python
class SchemaComparator:
    """데이터베이스 스키마 비교기"""
    
    def __init__(self, source_conn, target_conn):
        self.source_engine = create_engine(source_conn)
        self.target_engine = create_engine(target_conn)
        
    def compare_schemas(self):
        """스키마 비교"""
        source_schema = self.extract_schema(self.source_engine)
        target_schema = self.extract_schema(self.target_engine)
        
        differences = {
            'tables': {
                'only_in_source': [],
                'only_in_target': [],
                'modified': []
            },
            'procedures': {
                'only_in_source': [],
                'only_in_target': [],
                'modified': []
            },
            'indexes': {
                'missing': [],
                'extra': [],
                'different': []
            }
        }
        
        # 테이블 비교
        source_tables = set(source_schema['tables'].keys())
        target_tables = set(target_schema['tables'].keys())
        
        differences['tables']['only_in_source'] = list(source_tables - target_tables)
        differences['tables']['only_in_target'] = list(target_tables - source_tables)
        
        # 공통 테이블의 구조 비교
        for table in source_tables & target_tables:
            source_def = source_schema['tables'][table]
            target_def = target_schema['tables'][table]
            
            if self.table_structure_differs(source_def, target_def):
                differences['tables']['modified'].append({
                    'table': table,
                    'source': source_def,
                    'target': target_def,
                    'differences': self.get_table_differences(source_def, target_def)
                })
        
        return differences
    
    def extract_schema(self, engine):
        """스키마 정보 추출"""
        schema = {
            'tables': {},
            'procedures': {},
            'indexes': {},
            'constraints': {}
        }
        
        with engine.connect() as conn:
            # 테이블 정보
            tables_query = """
            SELECT 
                t.table_name,
                c.column_name,
                c.data_type,
                c.is_nullable,
                c.column_default,
                c.character_maximum_length,
                c.numeric_precision,
                c.numeric_scale
            FROM information_schema.tables t
            JOIN information_schema.columns c 
                ON t.table_name = c.table_name
            WHERE t.table_schema = 'public'
                AND t.table_type = 'BASE TABLE'
            ORDER BY t.table_name, c.ordinal_position
            """
            
            result = conn.execute(text(tables_query))
            
            for row in result:
                table_name = row[0]
                if table_name not in schema['tables']:
                    schema['tables'][table_name] = {'columns': {}}
                
                schema['tables'][table_name]['columns'][row[1]] = {
                    'data_type': row[2],
                    'nullable': row[3] == 'YES',
                    'default': row[4],
                    'max_length': row[5],
                    'precision': row[6],
                    'scale': row[7]
                }
            
            # 인덱스 정보
            index_query = """
            SELECT 
                schemaname,
                tablename,
                indexname,
                indexdef
            FROM pg_indexes
            WHERE schemaname = 'public'
            """
            
            result = conn.execute(text(index_query))
            
            for row in result:
                table_name = row[1]
                index_name = row[2]
                
                if table_name not in schema['indexes']:
                    schema['indexes'][table_name] = {}
                    
                schema['indexes'][table_name][index_name] = {
                    'definition': row[3]
                }
        
        return schema
    
    def generate_sync_script(self, differences):
        """동기화 스크립트 생성"""
        sync_script = []
        
        # 새 테이블 생성
        for table in differences['tables']['only_in_source']:
            source_def = self.extract_table_definition(self.source_engine, table)
            sync_script.append(f"-- Create table {table}")
            sync_script.append(source_def)
            sync_script.append("")
        
        # 수정된 테이블 업데이트
        for modified in differences['tables']['modified']:
            table = modified['table']
            sync_script.append(f"-- Modify table {table}")
            
            for diff in modified['differences']:
                if diff['type'] == 'column_added':
                    col = diff['column']
                    col_def = diff['definition']
                    sync_script.append(
                        f"ALTER TABLE {table} ADD COLUMN {col} {col_def};"
                    )
                elif diff['type'] == 'column_removed':
                    sync_script.append(
                        f"ALTER TABLE {table} DROP COLUMN {diff['column']};"
                    )
                elif diff['type'] == 'column_modified':
                    # 주의: 일부 변경은 데이터 손실을 일으킬 수 있음
                    sync_script.append(
                        f"-- WARNING: Column modification may cause data loss"
                    )
                    sync_script.append(
                        f"ALTER TABLE {table} ALTER COLUMN {diff['column']} "
                        f"TYPE {diff['new_type']};"
                    )
            
            sync_script.append("")
        
        return '\n'.join(sync_script)
```

## 2. CI/CD 파이프라인

### 2.1 데이터베이스 CI/CD 파이프라인

```yaml
# .gitlab-ci.yml - GitLab CI/CD 파이프라인
stages:
  - validate
  - test
  - build
  - deploy-dev
  - deploy-staging
  - deploy-production

variables:
  POSTGRES_DB: testdb
  POSTGRES_USER: testuser
  POSTGRES_PASSWORD: testpass

# 스키마 검증
validate-schema:
  stage: validate
  image: postgres:14
  script:
    - |
      # SQL 구문 검증
      for file in migrations/*.sql; do
        echo "Validating $file"
        psql -h postgres -U $POSTGRES_USER -d $POSTGRES_DB -f "$file" --variable ON_ERROR_STOP=1 --dry-run
      done
    - |
      # 명명 규칙 검증
      python scripts/validate_naming_conventions.py migrations/

# 단위 테스트
unit-tests:
  stage: test
  image: python:3.9
  services:
    - postgres:14
  before_script:
    - pip install -r requirements.txt
    - python scripts/setup_test_db.py
  script:
    - pytest tests/unit/ -v --cov=database --cov-report=xml
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# 통합 테스트
integration-tests:
  stage: test
  image: python:3.9
  services:
    - postgres:14
    - redis:6
  script:
    - |
      # 마이그레이션 적용
      python migrate.py up
      
      # 통합 테스트 실행
      pytest tests/integration/ -v
      
      # 성능 벤치마크
      python scripts/run_benchmarks.py --baseline

# 보안 스캔
security-scan:
  stage: test
  image: aquasec/trivy
  script:
    - |
      # SQL 인젝션 취약점 스캔
      trivy fs --security-checks vuln,config migrations/
      
      # 민감한 정보 스캔
      python scripts/scan_sensitive_data.py

# 개발 환경 배포
deploy-dev:
  stage: deploy-dev
  image: python:3.9
  environment:
    name: development
    url: https://dev-db.company.com
  script:
    - |
      # 백업 생성
      python scripts/backup_database.py --env dev
      
      # 마이그레이션 적용
      python migrate.py up --env dev
      
      # 스키마 검증
      python scripts/verify_schema.py --env dev
      
      # 헬스 체크
      python scripts/health_check.py --env dev
  only:
    - develop

# 스테이징 환경 배포
deploy-staging:
  stage: deploy-staging
  image: python:3.9
  environment:
    name: staging
    url: https://staging-db.company.com
  script:
    - |
      # 프로덕션 데이터 복사 (마스킹 적용)
      python scripts/copy_prod_to_staging.py --mask-sensitive-data
      
      # 마이그레이션 적용
      python migrate.py up --env staging
      
      # 성능 테스트
      python scripts/load_test.py --env staging --duration 300
      
      # 롤백 테스트
      python scripts/test_rollback.py --env staging
  only:
    - main
  when: manual

# 프로덕션 배포
deploy-production:
  stage: deploy-production
  image: python:3.9
  environment:
    name: production
    url: https://db.company.com
  script:
    - |
      # 사전 체크
      python scripts/pre_deployment_check.py --env production
      
      # 백업 생성
      python scripts/backup_database.py --env production --type full
      
      # Blue-Green 배포
      python scripts/blue_green_deploy.py --env production
      
      # 포스트 배포 검증
      python scripts/post_deployment_validation.py --env production
  only:
    - main
  when: manual
  allow_failure: false
```

### 2.2 자동화된 데이터베이스 테스트

```python
import pytest
import asyncio
from sqlalchemy import create_engine, text
import pandas as pd
from faker import Faker
import random

class DatabaseTestFramework:
    """데이터베이스 테스트 프레임워크"""
    
    def __init__(self, connection_string):
        self.engine = create_engine(connection_string)
        self.faker = Faker()
        
    @pytest.fixture
    def test_database(self):
        """테스트 데이터베이스 픽스처"""
        # 테스트 데이터베이스 생성
        test_db_name = f"test_{random.randint(1000, 9999)}"
        
        with self.engine.connect() as conn:
            conn.execute(text(f"CREATE DATABASE {test_db_name}"))
            conn.commit()
        
        test_engine = create_engine(
            self.connection_string.replace('/postgres', f'/{test_db_name}')
        )
        
        yield test_engine
        
        # 정리
        test_engine.dispose()
        with self.engine.connect() as conn:
            conn.execute(text(f"DROP DATABASE {test_db_name}"))
            conn.commit()
    
    def test_migration_idempotency(self, test_database):
        """마이그레이션 멱등성 테스트"""
        migration_system = DatabaseMigrationSystem(test_database.url)
        
        # 첫 번째 실행
        migration_system.migrate()
        first_schema = self.get_schema_hash(test_database)
        
        # 두 번째 실행 (아무 변화 없어야 함)
        migration_system.migrate()
        second_schema = self.get_schema_hash(test_database)
        
        assert first_schema == second_schema, "Migration is not idempotent"
    
    def test_rollback_safety(self, test_database):
        """롤백 안전성 테스트"""
        migration_system = DatabaseMigrationSystem(test_database.url)
        
        # 데이터 삽입
        self.insert_test_data(test_database)
        initial_data = self.get_data_snapshot(test_database)
        
        # 마이그레이션 적용
        migration_system.migrate()
        
        # 롤백
        migration_system.rollback('V00000000__initial')
        
        # 재적용
        migration_system.migrate()
        
        # 데이터 무결성 확인
        final_data = self.get_data_snapshot(test_database)
        assert initial_data == final_data, "Data integrity lost during rollback"
    
    def test_concurrent_access(self, test_database):
        """동시성 테스트"""
        async def concurrent_operation(operation_id):
            async with test_database.connect() as conn:
                # 트랜잭션 시작
                trans = await conn.begin()
                
                try:
                    # 동시 삽입
                    await conn.execute(text("""
                        INSERT INTO concurrent_test (id, value, created_by)
                        VALUES (:id, :value, :created_by)
                    """), {
                        'id': operation_id,
                        'value': self.faker.text(),
                        'created_by': f'worker_{operation_id}'
                    })
                    
                    # 랜덤 지연
                    await asyncio.sleep(random.uniform(0.1, 0.5))
                    
                    # 업데이트
                    await conn.execute(text("""
                        UPDATE concurrent_test 
                        SET value = :new_value, updated_at = NOW()
                        WHERE id = :id
                    """), {
                        'id': operation_id,
                        'new_value': self.faker.text()
                    })
                    
                    await trans.commit()
                    
                except Exception as e:
                    await trans.rollback()
                    raise
        
        # 테이블 생성
        with test_database.connect() as conn:
            conn.execute(text("""
                CREATE TABLE concurrent_test (
                    id INT PRIMARY KEY,
                    value TEXT,
                    created_by VARCHAR(50),
                    created_at TIMESTAMP DEFAULT NOW(),
                    updated_at TIMESTAMP
                )
            """))
            conn.commit()
        
        # 동시 실행
        async def run_concurrent_test():
            tasks = [concurrent_operation(i) for i in range(100)]
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            # 오류 확인
            errors = [r for r in results if isinstance(r, Exception)]
            assert len(errors) == 0, f"Concurrent operations failed: {errors}"
        
        asyncio.run(run_concurrent_test())
    
    def test_performance_regression(self, test_database):
        """성능 회귀 테스트"""
        # 테스트 데이터 생성
        self.generate_large_dataset(test_database, rows=100000)
        
        # 벤치마크 쿼리들
        benchmark_queries = [
            {
                'name': 'simple_select',
                'query': 'SELECT * FROM users WHERE created_at > NOW() - INTERVAL \'1 day\'',
                'baseline_ms': 50
            },
            {
                'name': 'complex_join',
                'query': '''
                    SELECT u.*, COUNT(o.id) as order_count
                    FROM users u
                    LEFT JOIN orders o ON u.id = o.user_id
                    WHERE u.created_at > NOW() - INTERVAL \'30 days\'
                    GROUP BY u.id
                    HAVING COUNT(o.id) > 5
                ''',
                'baseline_ms': 200
            },
            {
                'name': 'aggregation',
                'query': '''
                    SELECT 
                        DATE_TRUNC(\'day\', created_at) as day,
                        COUNT(*) as user_count,
                        AVG(age) as avg_age
                    FROM users
                    GROUP BY DATE_TRUNC(\'day\', created_at)
                    ORDER BY day DESC
                    LIMIT 30
                ''',
                'baseline_ms': 100
            }
        ]
        
        results = []
        
        with test_database.connect() as conn:
            for benchmark in benchmark_queries:
                # 캐시 정리
                conn.execute(text("DISCARD ALL"))
                
                # 실행 시간 측정
                start_time = datetime.now()
                conn.execute(text(benchmark['query']))
                execution_time = (datetime.now() - start_time).total_seconds() * 1000
                
                # 기준선 대비 확인
                regression_ratio = execution_time / benchmark['baseline_ms']
                
                results.append({
                    'query': benchmark['name'],
                    'execution_ms': execution_time,
                    'baseline_ms': benchmark['baseline_ms'],
                    'regression_ratio': regression_ratio,
                    'passed': regression_ratio < 1.2  # 20% 이내 허용
                })
        
        # 성능 리포트 생성
        df = pd.DataFrame(results)
        print("\nPerformance Test Results:")
        print(df.to_string(index=False))
        
        # 실패한 테스트 확인
        failed_tests = df[~df['passed']]
        assert len(failed_tests) == 0, f"Performance regression detected:\n{failed_tests}"
```

## 3. 인프라스트럭처 코드 (IaC)

### 3.1 Terraform을 사용한 데이터베이스 인프라

```hcl
# main.tf - 데이터베이스 인프라 정의
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
  
  backend "s3" {
    bucket = "terraform-state-db"
    key    = "database/terraform.tfstate"
    region = "us-east-1"
  }
}

# RDS 클러스터
resource "aws_rds_cluster" "main" {
  cluster_identifier     = "${var.environment}-database-cluster"
  engine                = "aurora-postgresql"
  engine_version        = "14.6"
  database_name         = var.database_name
  master_username       = var.master_username
  master_password       = random_password.master.result
  
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.main.name
  db_subnet_group_name            = aws_db_subnet_group.main.name
  vpc_security_group_ids          = [aws_security_group.rds.id]
  
  backup_retention_period = var.backup_retention_days
  preferred_backup_window = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  
  enabled_cloudwatch_logs_exports = [
    "postgresql"
  ]
  
  deletion_protection = var.environment == "production"
  skip_final_snapshot = var.environment != "production"
  
  tags = local.common_tags
}

# RDS 인스턴스
resource "aws_rds_cluster_instance" "main" {
  count              = var.instance_count
  identifier         = "${var.environment}-database-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = var.instance_class
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version
  
  performance_insights_enabled = true
  monitoring_interval         = 60
  monitoring_role_arn        = aws_iam_role.rds_monitoring.arn
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.environment}-database-instance-${count.index + 1}"
    }
  )
}

# 파라미터 그룹
resource "aws_rds_cluster_parameter_group" "main" {
  name        = "${var.environment}-cluster-params"
  family      = "aurora-postgresql14"
  description = "Custom parameter group for ${var.environment}"
  
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,pg_hint_plan,auto_explain"
  }
  
  parameter {
    name  = "log_statement"
    value = "all"
  }
  
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # 1초 이상 쿼리 로깅
  }
  
  parameter {
    name  = "auto_explain.log_min_duration"
    value = "1000"
  }
  
  tags = local.common_tags
}

# 자동 스케일링
resource "aws_appautoscaling_target" "replicas" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "cluster:${aws_rds_cluster.main.cluster_identifier}"
  scalable_dimension = "rds:cluster:ReadReplicaCount"
  service_namespace  = "rds"
}

resource "aws_appautoscaling_policy" "replicas" {
  name               = "${var.environment}-replica-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.replicas.resource_id
  scalable_dimension = aws_appautoscaling_target.replicas.scalable_dimension
  service_namespace  = aws_appautoscaling_target.replicas.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "RDSReaderAverageCPUUtilization"
    }
    
    target_value = 70.0
  }
}

# 모니터링 대시보드
resource "aws_cloudwatch_dashboard" "database" {
  dashboard_name = "${var.environment}-database-dashboard"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/RDS", "CPUUtilization", "DBClusterIdentifier", aws_rds_cluster.main.id],
            [".", "DatabaseConnections", ".", "."],
            [".", "CommitLatency", ".", "."],
            [".", "ReadLatency", ".", "."]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "Database Performance Metrics"
        }
      },
      {
        type   = "metric"
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/RDS", "FreeableMemory", "DBClusterIdentifier", aws_rds_cluster.main.id],
            [".", "SwapUsage", ".", "."],
            [".", "DiskQueueDepth", ".", "."]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "Resource Utilization"
        }
      }
    ]
  })
}

# 출력 변수
output "cluster_endpoint" {
  value       = aws_rds_cluster.main.endpoint
  description = "RDS cluster endpoint"
}

output "reader_endpoint" {
  value       = aws_rds_cluster.main.reader_endpoint
  description = "RDS cluster reader endpoint"
}

output "database_name" {
  value       = aws_rds_cluster.main.database_name
  description = "Database name"
}
```

### 3.2 Ansible을 사용한 구성 관리

```yaml
# playbook-database-setup.yml
---
- name: Database Setup and Configuration
  hosts: database_servers
  become: yes
  vars:
    postgres_version: 14
    postgres_data_dir: /var/lib/postgresql/14/main
    postgres_config_dir: /etc/postgresql/14/main
    
  tasks:
    - name: Install PostgreSQL
      apt:
        name: 
          - postgresql-{{ postgres_version }}
          - postgresql-client-{{ postgres_version }}
          - postgresql-contrib-{{ postgres_version }}
          - python3-psycopg2
        state: present
        
    - name: Configure PostgreSQL
      template:
        src: postgresql.conf.j2
        dest: "{{ postgres_config_dir }}/postgresql.conf"
        owner: postgres
        group: postgres
        mode: '0644'
      notify: restart postgresql
      
    - name: Configure pg_hba.conf
      template:
        src: pg_hba.conf.j2
        dest: "{{ postgres_config_dir }}/pg_hba.conf"
        owner: postgres
        group: postgres
        mode: '0640'
      notify: reload postgresql
      
    - name: Create application database
      postgresql_db:
        name: "{{ app_database }}"
        encoding: UTF-8
        lc_collate: en_US.UTF-8
        lc_ctype: en_US.UTF-8
        template: template0
      become_user: postgres
      
    - name: Create application user
      postgresql_user:
        name: "{{ app_user }}"
        password: "{{ app_password }}"
        db: "{{ app_database }}"
        priv: ALL
        state: present
      become_user: postgres
      
    - name: Install monitoring extensions
      postgresql_ext:
        name: "{{ item }}"
        db: "{{ app_database }}"
      loop:
        - pg_stat_statements
        - pg_buffercache
        - pg_prewarm
      become_user: postgres
      
    - name: Setup backup script
      template:
        src: backup.sh.j2
        dest: /usr/local/bin/pg_backup.sh
        mode: '0755'
        
    - name: Schedule backup cron job
      cron:
        name: "PostgreSQL backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/pg_backup.sh"
        user: postgres
        
    - name: Configure log rotation
      template:
        src: postgresql-logrotate.j2
        dest: /etc/logrotate.d/postgresql
        
    - name: Setup monitoring
      include_tasks: monitoring.yml
      
  handlers:
    - name: restart postgresql
      systemd:
        name: postgresql
        state: restarted
        
    - name: reload postgresql
      systemd:
        name: postgresql
        state: reloaded
```

## 4. 데이터베이스 자동화 도구

### 4.1 자동화된 성능 튜닝

```python
class AutomatedDatabaseTuner:
    """자동화된 데이터베이스 튜너"""
    
    def __init__(self, connection_string):
        self.engine = create_engine(connection_string)
        self.metrics_history = []
        self.tuning_history = []
        
    async def auto_tune(self):
        """자동 튜닝 실행"""
        # 현재 메트릭 수집
        current_metrics = await self.collect_metrics()
        self.metrics_history.append(current_metrics)
        
        # 워크로드 분석
        workload_profile = self.analyze_workload()
        
        # 최적화 권장사항 생성
        recommendations = self.generate_recommendations(
            current_metrics, 
            workload_profile
        )
        
        # 권장사항 적용
        for recommendation in recommendations:
            if recommendation['auto_apply'] and recommendation['risk'] == 'low':
                await self.apply_recommendation(recommendation)
        
        return recommendations
    
    def analyze_workload(self):
        """워크로드 분석"""
        with self.engine.connect() as conn:
            # 쿼리 패턴 분석
            query_stats = conn.execute(text("""
                SELECT 
                    query,
                    calls,
                    total_time,
                    mean_time,
                    rows,
                    100.0 * total_time / SUM(total_time) OVER () AS percentage
                FROM pg_stat_statements
                WHERE query NOT LIKE '%pg_stat_statements%'
                ORDER BY total_time DESC
                LIMIT 100
            """)).fetchall()
            
            # 워크로드 분류
            workload_type = self.classify_workload(query_stats)
            
            return {
                'type': workload_type,
                'top_queries': query_stats[:10],
                'read_write_ratio': self.calculate_read_write_ratio(query_stats),
                'avg_query_complexity': self.calculate_query_complexity(query_stats)
            }
    
    def generate_recommendations(self, metrics, workload):
        """최적화 권장사항 생성"""
        recommendations = []
        
        # 메모리 최적화
        if metrics['cache_hit_ratio'] < 0.95:
            recommendations.append({
                'type': 'memory',
                'parameter': 'shared_buffers',
                'current_value': metrics['shared_buffers'],
                'recommended_value': self.calculate_optimal_shared_buffers(metrics),
                'reason': 'Low cache hit ratio',
                'risk': 'low',
                'auto_apply': True
            })
        
        # 연결 풀 최적화
        if metrics['connection_usage'] > 0.8:
            recommendations.append({
                'type': 'connection',
                'parameter': 'max_connections',
                'current_value': metrics['max_connections'],
                'recommended_value': int(metrics['max_connections'] * 1.5),
                'reason': 'High connection usage',
                'risk': 'medium',
                'auto_apply': False
            })
        
        # 워크로드별 최적화
        if workload['type'] == 'OLTP':
            recommendations.extend(self.generate_oltp_recommendations(metrics))
        elif workload['type'] == 'OLAP':
            recommendations.extend(self.generate_olap_recommendations(metrics))
        
        return recommendations
    
    async def apply_recommendation(self, recommendation):
        """권장사항 적용"""
        try:
            parameter = recommendation['parameter']
            new_value = recommendation['recommended_value']
            
            # 파라미터 변경
            with self.engine.connect() as conn:
                conn.execute(text(f"""
                    ALTER SYSTEM SET {parameter} = '{new_value}'
                """))
                conn.commit()
            
            # 변경 기록
            self.tuning_history.append({
                'timestamp': datetime.now(),
                'parameter': parameter,
                'old_value': recommendation['current_value'],
                'new_value': new_value,
                'reason': recommendation['reason']
            })
            
            # 일부 파라미터는 재시작 필요
            if self.requires_restart(parameter):
                print(f"Parameter {parameter} requires database restart")
            else:
                # 설정 리로드
                with self.engine.connect() as conn:
                    conn.execute(text("SELECT pg_reload_conf()"))
            
            return True
            
        except Exception as e:
            print(f"Failed to apply recommendation: {e}")
            return False
```

## 5. 모니터링 및 알림 자동화

### 5.1 지능형 알림 시스템

```python
class IntelligentAlertingSystem:
    """지능형 알림 시스템"""
    
    def __init__(self, config):
        self.config = config
        self.alert_history = []
        self.suppression_rules = []
        self.ml_model = self.load_anomaly_model()
        
    async def process_metrics(self, metrics):
        """메트릭 처리 및 알림 생성"""
        # 이상 탐지
        anomalies = self.detect_anomalies(metrics)
        
        # 알림 생성
        alerts = []
        for anomaly in anomalies:
            if not self.should_suppress(anomaly):
                alert = self.create_alert(anomaly, metrics)
                alerts.append(alert)
        
        # 알림 상관관계 분석
        correlated_alerts = self.correlate_alerts(alerts)
        
        # 알림 전송
        for alert_group in correlated_alerts:
            await self.send_alert(alert_group)
        
        # 학습
        self.update_ml_model(metrics, anomalies)
        
        return correlated_alerts
    
    def detect_anomalies(self, metrics):
        """이상 탐지"""
        anomalies = []
        
        # 규칙 기반 탐지
        for rule in self.config['alert_rules']:
            if self.evaluate_rule(rule, metrics):
                anomalies.append({
                    'type': 'rule_based',
                    'rule': rule,
                    'severity': rule['severity'],
                    'value': metrics.get(rule['metric'])
                })
        
        # ML 기반 탐지
        ml_anomalies = self.ml_model.predict(metrics)
        anomalies.extend(ml_anomalies)
        
        return anomalies
    
    def should_suppress(self, anomaly):
        """알림 억제 규칙 확인"""
        # 최근 동일 알림 확인
        recent_similar = [
            alert for alert in self.alert_history
            if alert['type'] == anomaly['type'] 
            and (datetime.now() - alert['timestamp']).seconds < 300
        ]
        
        if len(recent_similar) > 3:
            return True  # 5분 내 3회 이상 발생 시 억제
        
        # 유지보수 창 확인
        if self.in_maintenance_window():
            return True
        
        # 커스텀 억제 규칙
        for rule in self.suppression_rules:
            if rule['condition'](anomaly):
                return True
        
        return False
    
    def correlate_alerts(self, alerts):
        """알림 상관관계 분석"""
        alert_groups = []
        
        # 시간적 근접성과 인과관계로 그룹화
        for alert in alerts:
            grouped = False
            
            for group in alert_groups:
                if self.are_related(alert, group):
                    group.append(alert)
                    grouped = True
                    break
            
            if not grouped:
                alert_groups.append([alert])
        
        # 루트 원인 분석
        for group in alert_groups:
            root_cause = self.identify_root_cause(group)
            for alert in group:
                alert['root_cause'] = root_cause
        
        return alert_groups
    
    async def send_alert(self, alert_group):
        """알림 전송"""
        # 알림 우선순위 결정
        priority = max(alert['severity'] for alert in alert_group)
        
        # 수신자 결정
        recipients = self.determine_recipients(priority, alert_group)
        
        # 알림 포맷팅
        message = self.format_alert_message(alert_group)
        
        # 채널별 전송
        for recipient in recipients:
            if recipient['channel'] == 'email':
                await self.send_email(recipient['address'], message)
            elif recipient['channel'] == 'slack':
                await self.send_slack(recipient['webhook'], message)
            elif recipient['channel'] == 'pagerduty':
                await self.send_pagerduty(recipient['key'], message, priority)
        
        # 기록
        self.alert_history.extend(alert_group)
```

## 마무리

데이터베이스 DevOps는 개발과 운영의 효율성을 크게 향상시킵니다. 버전 관리, CI/CD 파이프라인, 인프라스트럭처 코드, 자동화된 테스트와 모니터링을 통해 안정적이고 신속한 데이터베이스 변경 관리가 가능합니다. 지속적인 개선과 자동화를 통해 데이터베이스 운영의 품질을 높일 수 있습니다.

다음 장에서는 빅데이터와 분석 데이터베이스에 대해 학습하겠습니다.