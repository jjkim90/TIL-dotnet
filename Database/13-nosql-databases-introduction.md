# NoSQL 데이터베이스 소개

## 개요

NoSQL(Not Only SQL) 데이터베이스는 전통적인 관계형 데이터베이스의 한계를 극복하기 위해 등장한 다양한 데이터 저장 기술의 총칭입니다. 이 장에서는 NoSQL의 개념, 주요 유형별 특징, CAP 이론, 그리고 각 유형의 대표적인 데이터베이스와 사용 사례를 학습합니다.

## 1. NoSQL 기본 개념

### 1.1 NoSQL의 특징과 RDBMS와의 비교

```python
# NoSQL vs RDBMS 특성 비교
import pandas as pd
import matplotlib.pyplot as plt

# 비교 데이터
comparison_data = {
    'Feature': ['Schema', 'Scalability', 'ACID', 'Query Language', 
                'Data Model', 'Consistency', 'Use Case'],
    'RDBMS': ['Fixed Schema', 'Vertical (Scale-up)', 'Full ACID', 
              'SQL', 'Tables/Relations', 'Strong Consistency', 
              'Complex Transactions'],
    'NoSQL': ['Schema-less/Flexible', 'Horizontal (Scale-out)', 
              'BASE/Eventual', 'Various', 'Document/KV/Graph/Column', 
              'Eventual Consistency', 'Big Data/Real-time']
}

df = pd.DataFrame(comparison_data)
print(df.to_string(index=False))

# CAP 이론 시각화
def visualize_cap_theorem():
    fig, ax = plt.subplots(figsize=(10, 8))
    
    # CAP 삼각형
    triangle = plt.Polygon([(0.5, 0.866), (0, 0), (1, 0)], 
                          fill=False, edgecolor='black', linewidth=2)
    ax.add_patch(triangle)
    
    # 레이블
    ax.text(0.5, 0.9, 'Consistency', ha='center', fontsize=12, weight='bold')
    ax.text(-0.1, -0.1, 'Availability', ha='center', fontsize=12, weight='bold')
    ax.text(1.1, -0.1, 'Partition\nTolerance', ha='center', fontsize=12, weight='bold')
    
    # 데이터베이스 위치
    databases = {
        'RDBMS': (0.25, 0.433, 'CA'),
        'MongoDB': (0.75, 0.433, 'CP'),
        'Cassandra': (0.5, 0.1, 'AP'),
        'Redis': (0.3, 0.35, 'CA/CP'),
        'DynamoDB': (0.7, 0.35, 'CP/AP')
    }
    
    for db, (x, y, cap) in databases.items():
        ax.scatter(x, y, s=100)
        ax.text(x, y-0.05, f'{db}\n({cap})', ha='center', fontsize=9)
    
    ax.set_xlim(-0.3, 1.3)
    ax.set_ylim(-0.3, 1.1)
    ax.axis('off')
    plt.title('CAP Theorem and Database Systems')
    plt.show()

visualize_cap_theorem()
```

### 1.2 BASE vs ACID

```python
# BASE 속성 구현 예제
import time
import threading
from datetime import datetime

class EventuallyConsistentStore:
    def __init__(self):
        self.primary_store = {}
        self.replica_stores = [{}, {}, {}]  # 3개의 복제본
        self.replication_queue = []
        self.lock = threading.Lock()
    
    def write(self, key, value):
        """Basically Available - 쓰기는 항상 가능"""
        timestamp = datetime.now()
        
        # 주 저장소에 즉시 쓰기
        with self.lock:
            self.primary_store[key] = {
                'value': value,
                'timestamp': timestamp,
                'version': self.primary_store.get(key, {}).get('version', 0) + 1
            }
            
            # 비동기 복제를 위해 큐에 추가
            self.replication_queue.append({
                'key': key,
                'value': value,
                'timestamp': timestamp,
                'replicated': []
            })
        
        # 백그라운드에서 복제 시작
        threading.Thread(target=self._replicate_async).start()
        
        return True
    
    def read(self, key, consistency_level='eventual'):
        """Soft state - 읽기 일관성 수준 선택"""
        if consistency_level == 'strong':
            # 모든 복제본에서 읽어서 최신 버전 반환
            values = [self.primary_store.get(key)]
            for replica in self.replica_stores:
                values.append(replica.get(key))
            
            # 가장 최신 버전 찾기
            latest = None
            for val in values:
                if val and (not latest or val['timestamp'] > latest['timestamp']):
                    latest = val
            
            return latest['value'] if latest else None
            
        elif consistency_level == 'eventual':
            # 가장 가까운 복제본에서 읽기
            return self.primary_store.get(key, {}).get('value')
    
    def _replicate_async(self):
        """Eventually consistent - 비동기 복제"""
        time.sleep(0.1)  # 네트워크 지연 시뮬레이션
        
        with self.lock:
            while self.replication_queue:
                item = self.replication_queue.pop(0)
                
                # 각 복제본에 전파
                for i, replica in enumerate(self.replica_stores):
                    if i not in item['replicated']:
                        replica[item['key']] = {
                            'value': item['value'],
                            'timestamp': item['timestamp']
                        }
                        item['replicated'].append(i)
                
                # 모든 복제본에 전파되지 않았으면 다시 큐에 추가
                if len(item['replicated']) < len(self.replica_stores):
                    self.replication_queue.append(item)

# 사용 예제
store = EventuallyConsistentStore()

# 쓰기 작업
store.write('user:1', {'name': 'John', 'age': 30})
store.write('user:2', {'name': 'Jane', 'age': 25})

# 즉시 읽기 (eventual consistency)
print("Immediate read:", store.read('user:1'))

# 잠시 대기 후 읽기
time.sleep(0.5)
print("After replication:", store.read('user:1', 'strong'))
```

## 2. 키-값 저장소

### 2.1 Redis 예제

```python
import redis
import json
import time

# Redis 연결
r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

# 기본 키-값 작업
def basic_key_value_operations():
    # 문자열 저장
    r.set('user:1000', 'John Doe')
    print(f"User 1000: {r.get('user:1000')}")
    
    # TTL 설정
    r.setex('session:abc123', 3600, 'user:1000')  # 1시간 만료
    print(f"TTL: {r.ttl('session:abc123')} seconds")
    
    # 원자적 증가
    r.set('counter:pageviews', 0)
    for _ in range(5):
        r.incr('counter:pageviews')
    print(f"Page views: {r.get('counter:pageviews')}")
    
    # 해시 사용
    user_data = {
        'name': 'Jane Smith',
        'email': 'jane@example.com',
        'age': '28',
        'last_login': str(int(time.time()))
    }
    r.hset('user:1001', mapping=user_data)
    print(f"User 1001: {r.hgetall('user:1001')}")
    
    # 리스트 작업
    r.lpush('queue:tasks', 'task1', 'task2', 'task3')
    print(f"Next task: {r.rpop('queue:tasks')}")
    
    # 집합 작업
    r.sadd('tags:python', 'user:1000', 'user:1001', 'user:1002')
    r.sadd('tags:javascript', 'user:1001', 'user:1003')
    
    # 교집합 찾기
    common_users = r.sinter('tags:python', 'tags:javascript')
    print(f"Users with both tags: {common_users}")
    
    # 정렬된 집합 (점수 기반)
    r.zadd('leaderboard', {'player1': 100, 'player2': 85, 'player3': 95})
    top_players = r.zrevrange('leaderboard', 0, 2, withscores=True)
    print(f"Top players: {top_players}")

# 캐싱 패턴
def caching_pattern(user_id):
    cache_key = f'user:{user_id}:profile'
    
    # 캐시 확인
    cached_data = r.get(cache_key)
    if cached_data:
        print("Cache hit!")
        return json.loads(cached_data)
    
    # 캐시 미스 - DB에서 로드 (시뮬레이션)
    print("Cache miss - loading from DB")
    user_data = {
        'id': user_id,
        'name': f'User {user_id}',
        'created_at': str(datetime.now())
    }
    
    # 캐시에 저장 (1시간 TTL)
    r.setex(cache_key, 3600, json.dumps(user_data))
    
    return user_data

# Pub/Sub 패턴
def pubsub_example():
    # Publisher
    def publisher():
        for i in range(5):
            message = {'event': 'user_action', 'data': f'action_{i}'}
            r.publish('events:user', json.dumps(message))
            time.sleep(1)
    
    # Subscriber
    def subscriber():
        pubsub = r.pubsub()
        pubsub.subscribe('events:user')
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                print(f"Received: {data}")
    
    # 실행 (실제로는 별도 프로세스에서)
    import threading
    threading.Thread(target=subscriber).start()
    time.sleep(0.5)
    threading.Thread(target=publisher).start()

# 분산 락
def distributed_lock_example():
    lock_key = 'resource:lock'
    lock_value = str(time.time())
    
    # 락 획득 시도
    if r.set(lock_key, lock_value, nx=True, ex=10):  # 10초 만료
        try:
            print("Lock acquired, processing...")
            # 크리티컬 섹션
            time.sleep(2)
            print("Processing complete")
        finally:
            # 락 해제 (자신이 설정한 락인지 확인)
            if r.get(lock_key) == lock_value:
                r.delete(lock_key)
                print("Lock released")
    else:
        print("Could not acquire lock")

# 실행
basic_key_value_operations()
print("\n--- Caching Example ---")
print(caching_pattern(1000))
print(caching_pattern(1000))  # 캐시에서 로드
```

### 2.2 Memcached와 분산 캐싱

```python
import memcache
import hashlib
import json

class DistributedCache:
    def __init__(self, servers):
        self.mc = memcache.Client(servers, debug=0)
        self.servers = servers
    
    def consistent_hash(self, key):
        """일관된 해싱으로 서버 선택"""
        hash_value = int(hashlib.md5(key.encode()).hexdigest(), 16)
        return self.servers[hash_value % len(self.servers)]
    
    def get(self, key):
        return self.mc.get(key)
    
    def set(self, key, value, expiry=0):
        return self.mc.set(key, value, time=expiry)
    
    def add(self, key, value, expiry=0):
        """키가 없을 때만 추가"""
        return self.mc.add(key, value, time=expiry)
    
    def increment(self, key, delta=1):
        """원자적 증가"""
        return self.mc.incr(key, delta)
    
    def get_multi(self, keys):
        """여러 키 한번에 가져오기"""
        return self.mc.get_multi(keys)
    
    def cache_aside_pattern(self, key, loader_func, expiry=3600):
        """Cache-Aside 패턴"""
        # 캐시에서 확인
        value = self.get(key)
        if value is not None:
            return value
        
        # 캐시 미스 - 데이터 로드
        value = loader_func()
        
        # 캐시에 저장
        self.set(key, value, expiry)
        
        return value
    
    def write_through_pattern(self, key, value, writer_func, expiry=3600):
        """Write-Through 패턴"""
        # 데이터베이스에 먼저 쓰기
        writer_func(value)
        
        # 캐시 업데이트
        self.set(key, value, expiry)
        
        return True
    
    def write_behind_pattern(self, key, value, expiry=3600):
        """Write-Behind 패턴 (비동기)"""
        # 캐시에 먼저 쓰기
        self.set(key, value, expiry)
        
        # 백그라운드에서 DB 업데이트 (큐 사용)
        self.mc.set(f'writeback:{key}', value, time=0)
        
        return True

# 사용 예제
cache = DistributedCache(['127.0.0.1:11211', '127.0.0.1:11212'])

# 세션 관리
session_data = {
    'user_id': 1000,
    'username': 'john_doe',
    'roles': ['user', 'admin'],
    'last_activity': time.time()
}
cache.set('session:abc123', json.dumps(session_data), expiry=1800)

# 카운터
cache.set('stats:pageviews', 0)
for _ in range(10):
    cache.increment('stats:pageviews')

print(f"Page views: {cache.get('stats:pageviews')}")
```

## 3. 문서 저장소

### 3.1 MongoDB 예제

```python
from pymongo import MongoClient, ASCENDING, DESCENDING
from datetime import datetime
import bson

# MongoDB 연결
client = MongoClient('mongodb://localhost:27017/')
db = client['ecommerce']

# 컬렉션
users = db['users']
products = db['products']
orders = db['orders']

# 문서 저장소 기본 작업
def document_store_operations():
    # 사용자 문서 삽입
    user_doc = {
        'username': 'john_doe',
        'email': 'john@example.com',
        'profile': {
            'first_name': 'John',
            'last_name': 'Doe',
            'age': 30,
            'interests': ['programming', 'music', 'travel']
        },
        'addresses': [
            {
                'type': 'home',
                'street': '123 Main St',
                'city': 'Seoul',
                'country': 'Korea',
                'primary': True
            },
            {
                'type': 'work',
                'street': '456 Office Blvd',
                'city': 'Seoul',
                'country': 'Korea'
            }
        ],
        'created_at': datetime.utcnow(),
        'last_login': datetime.utcnow()
    }
    
    user_id = users.insert_one(user_doc).inserted_id
    print(f"Inserted user with ID: {user_id}")
    
    # 중첩된 문서 쿼리
    # 특정 도시에 사는 사용자 찾기
    seoul_users = users.find({'addresses.city': 'Seoul'})
    
    # 배열 요소 쿼리
    # 프로그래밍에 관심있는 사용자
    programmers = users.find({'profile.interests': 'programming'})
    
    # 복잡한 쿼리
    complex_query = users.find({
        '$and': [
            {'profile.age': {'$gte': 25, '$lte': 35}},
            {'addresses': {'$elemMatch': {
                'type': 'home',
                'city': 'Seoul'
            }}},
            {'profile.interests': {'$in': ['programming', 'technology']}}
        ]
    })
    
    # 부분 업데이트
    users.update_one(
        {'_id': user_id},
        {
            '$set': {
                'profile.last_name': 'Smith',
                'last_login': datetime.utcnow()
            },
            '$push': {
                'profile.interests': 'blockchain'
            },
            '$inc': {
                'login_count': 1
            }
        }
    )
    
    # 배열 업데이트
    users.update_one(
        {'_id': user_id, 'addresses.type': 'home'},
        {'$set': {'addresses.$.primary': False}}
    )

# 집계 파이프라인
def aggregation_examples():
    # 상품 판매 분석
    pipeline = [
        # 주문 펼치기
        {'$unwind': '$items'},
        
        # 상품 정보 조인
        {'$lookup': {
            'from': 'products',
            'localField': 'items.product_id',
            'foreignField': '_id',
            'as': 'product_info'
        }},
        
        # 카테고리별 그룹화
        {'$group': {
            '_id': '$product_info.category',
            'total_sales': {'$sum': {'$multiply': ['$items.quantity', '$items.price']}},
            'order_count': {'$sum': 1},
            'avg_order_value': {'$avg': {'$multiply': ['$items.quantity', '$items.price']}}
        }},
        
        # 정렬
        {'$sort': {'total_sales': -1}},
        
        # 상위 5개만
        {'$limit': 5}
    ]
    
    results = orders.aggregate(pipeline)
    
    # 시계열 분석
    time_series_pipeline = [
        {'$match': {
            'created_at': {
                '$gte': datetime(2024, 1, 1),
                '$lt': datetime(2024, 12, 31)
            }
        }},
        
        {'$group': {
            '_id': {
                'year': {'$year': '$created_at'},
                'month': {'$month': '$created_at'},
                'day': {'$dayOfMonth': '$created_at'}
            },
            'daily_revenue': {'$sum': '$total_amount'},
            'order_count': {'$sum': 1}
        }},
        
        {'$sort': {'_id.year': 1, '_id.month': 1, '_id.day': 1}}
    ]
    
    daily_stats = orders.aggregate(time_series_pipeline)

# 인덱싱 전략
def indexing_strategies():
    # 단일 필드 인덱스
    users.create_index('email', unique=True)
    users.create_index('username', unique=True)
    
    # 복합 인덱스
    orders.create_index([
        ('user_id', ASCENDING),
        ('created_at', DESCENDING)
    ])
    
    # 다중 키 인덱스 (배열)
    users.create_index('profile.interests')
    
    # 텍스트 인덱스
    products.create_index([('name', 'text'), ('description', 'text')])
    
    # 지리공간 인덱스
    stores = db['stores']
    stores.create_index([('location', '2dsphere')])
    
    # TTL 인덱스 (자동 삭제)
    sessions = db['sessions']
    sessions.create_index('expires_at', expireAfterSeconds=0)
    
    # 부분 인덱스
    orders.create_index(
        'total_amount',
        partialFilterExpression={'total_amount': {'$gt': 100}}
    )

# 트랜잭션 예제
def transaction_example():
    with client.start_session() as session:
        with session.start_transaction():
            try:
                # 재고 감소
                products.update_one(
                    {'_id': product_id, 'stock': {'$gte': quantity}},
                    {'$inc': {'stock': -quantity}},
                    session=session
                )
                
                # 주문 생성
                order = {
                    'user_id': user_id,
                    'items': [{'product_id': product_id, 'quantity': quantity}],
                    'total_amount': total,
                    'status': 'pending',
                    'created_at': datetime.utcnow()
                }
                orders.insert_one(order, session=session)
                
                # 사용자 포인트 차감
                users.update_one(
                    {'_id': user_id, 'points': {'$gte': points_to_use}},
                    {'$inc': {'points': -points_to_use}},
                    session=session
                )
                
                # 모든 작업 성공 시 커밋
                session.commit_transaction()
                
            except Exception as e:
                # 실패 시 롤백
                session.abort_transaction()
                raise e

# 변경 스트림 (Change Streams)
def change_streams_example():
    # 실시간 변경 감지
    with orders.watch() as stream:
        for change in stream:
            if change['operationType'] == 'insert':
                print(f"New order: {change['fullDocument']}")
                # 실시간 알림, 분석 등 처리
            elif change['operationType'] == 'update':
                print(f"Order updated: {change['documentKey']}")
```

### 3.2 CouchDB와 동기화

```python
import couchdb
import json
from datetime import datetime

# CouchDB 연결
couch = couchdb.Server('http://admin:password@localhost:5984/')

# 데이터베이스 생성/연결
try:
    db = couch.create('myapp')
except:
    db = couch['myapp']

# CouchDB 문서 작업
def couchdb_operations():
    # 문서 저장
    doc = {
        '_id': 'user:1000',
        'type': 'user',
        'name': 'John Doe',
        'email': 'john@example.com',
        'created_at': datetime.utcnow().isoformat(),
        'preferences': {
            'theme': 'dark',
            'language': 'en'
        }
    }
    
    db.save(doc)
    
    # 문서 조회
    user = db['user:1000']
    print(f"User: {user['name']}")
    
    # 문서 업데이트 (낙관적 동시성 제어)
    user['last_login'] = datetime.utcnow().isoformat()
    db.save(user)  # _rev 자동 처리
    
    # 뷰 생성
    design_doc = {
        '_id': '_design/users',
        'views': {
            'by_email': {
                'map': '''
                    function(doc) {
                        if (doc.type === 'user' && doc.email) {
                            emit(doc.email, doc.name);
                        }
                    }
                '''
            },
            'by_created_date': {
                'map': '''
                    function(doc) {
                        if (doc.type === 'user' && doc.created_at) {
                            emit(doc.created_at, {
                                name: doc.name,
                                email: doc.email
                            });
                        }
                    }
                '''
            }
        }
    }
    
    db.save(design_doc)
    
    # 뷰 쿼리
    for row in db.view('users/by_email'):
        print(f"{row.key}: {row.value}")

# 복제와 동기화
def replication_example():
    # 로컬에서 원격으로 복제
    source_db = couch['myapp']
    target_url = 'http://remote-server:5984/myapp'
    
    couch.replicate(source_db, target_url, continuous=True)
    
    # 양방향 동기화
    couch.replicate(source_db, target_url, continuous=True)
    couch.replicate(target_url, source_db, continuous=True)
    
    # 필터링된 복제
    filter_func = '''
    function(doc, req) {
        return doc.type === 'user' && doc.active === true;
    }
    '''
    
    couch.replicate(
        source_db, 
        target_url,
        filter='users/active_only',
        continuous=True
    )

# 충돌 해결
def conflict_resolution():
    doc_id = 'user:1000'
    
    # 충돌 확인
    doc = db[doc_id]
    if '_conflicts' in doc:
        conflicts = doc['_conflicts']
        
        # 각 충돌 버전 조회
        versions = [doc]
        for rev in conflicts:
            versions.append(db.get(doc_id, rev=rev))
        
        # 충돌 해결 로직 (예: 타임스탬프 기반)
        winner = max(versions, key=lambda v: v.get('updated_at', ''))
        
        # 패배한 버전들 삭제
        for version in versions:
            if version['_rev'] != winner['_rev']:
                db.delete(version)
        
        print(f"Resolved conflict, winner: {winner['_rev']}")
```

## 4. 컬럼 패밀리 저장소

### 4.1 Cassandra 예제

```python
from cassandra.cluster import Cluster
from cassandra.auth import PlainTextAuthProvider
from cassandra.policies import DCAwareRoundRobinPolicy
from datetime import datetime
import uuid

# Cassandra 연결
auth_provider = PlainTextAuthProvider(username='cassandra', password='cassandra')
cluster = Cluster(
    ['127.0.0.1'],
    auth_provider=auth_provider,
    load_balancing_policy=DCAwareRoundRobinPolicy(local_dc='datacenter1')
)
session = cluster.connect()

# 키스페이스와 테이블 생성
def setup_schema():
    # 키스페이스 생성
    session.execute("""
        CREATE KEYSPACE IF NOT EXISTS ecommerce
        WITH replication = {
            'class': 'NetworkTopologyStrategy',
            'datacenter1': 3
        }
        AND durable_writes = true
    """)
    
    session.set_keyspace('ecommerce')
    
    # 시계열 데이터 테이블
    session.execute("""
        CREATE TABLE IF NOT EXISTS sensor_data (
            sensor_id text,
            date date,
            time timestamp,
            temperature double,
            humidity double,
            pressure double,
            PRIMARY KEY ((sensor_id, date), time)
        ) WITH CLUSTERING ORDER BY (time DESC)
        AND compression = {'class': 'LZ4Compressor'}
        AND compaction = {
            'class': 'TimeWindowCompactionStrategy',
            'compaction_window_unit': 'HOURS',
            'compaction_window_size': 1
        }
    """)
    
    # 와이드 로우 테이블
    session.execute("""
        CREATE TABLE IF NOT EXISTS user_activities (
            user_id uuid,
            activity_date date,
            activity_time timestamp,
            activity_type text,
            details text,
            ip_address inet,
            user_agent text,
            PRIMARY KEY ((user_id, activity_date), activity_time, activity_type)
        ) WITH CLUSTERING ORDER BY (activity_time DESC)
    """)
    
    # 카운터 테이블
    session.execute("""
        CREATE TABLE IF NOT EXISTS page_views (
            page_url text,
            view_date date,
            view_count counter,
            PRIMARY KEY ((page_url), view_date)
        )
    """)

# 데이터 모델링 패턴
def data_modeling_patterns():
    # 1. 시계열 데이터 삽입
    prepared = session.prepare("""
        INSERT INTO sensor_data (sensor_id, date, time, temperature, humidity, pressure)
        VALUES (?, ?, ?, ?, ?, ?)
    """)
    
    # 배치 삽입
    from cassandra.query import BatchStatement
    batch = BatchStatement()
    
    sensor_id = 'sensor-001'
    today = datetime.utcnow().date()
    
    for i in range(100):
        time = datetime.utcnow()
        batch.add(prepared, (
            sensor_id,
            today,
            time,
            20.0 + i * 0.1,  # temperature
            60.0 + i * 0.05,  # humidity
            1013.25           # pressure
        ))
    
    session.execute(batch)
    
    # 2. 시간 범위 쿼리
    results = session.execute("""
        SELECT * FROM sensor_data
        WHERE sensor_id = %s 
        AND date = %s
        AND time >= %s
        AND time <= %s
    """, (sensor_id, today, start_time, end_time))
    
    # 3. 와이드 로우 활용
    user_id = uuid.uuid4()
    
    # 사용자 활동 기록
    session.execute("""
        INSERT INTO user_activities 
        (user_id, activity_date, activity_time, activity_type, details, ip_address, user_agent)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
    """, (
        user_id,
        datetime.utcnow().date(),
        datetime.utcnow(),
        'login',
        json.dumps({'method': 'oauth', 'provider': 'google'}),
        '192.168.1.100',
        'Mozilla/5.0...'
    ))
    
    # 4. 카운터 업데이트
    session.execute("""
        UPDATE page_views
        SET view_count = view_count + 1
        WHERE page_url = %s AND view_date = %s
    """, ('/home', datetime.utcnow().date()))

# 성능 최적화 패턴
def performance_patterns():
    # 1. 비정규화를 통한 조인 회피
    session.execute("""
        CREATE TABLE IF NOT EXISTS orders_by_user (
            user_id uuid,
            order_id uuid,
            order_date timestamp,
            total_amount decimal,
            -- 비정규화된 사용자 정보
            user_name text,
            user_email text,
            -- 비정규화된 상품 정보
            product_names list<text>,
            product_prices list<decimal>,
            PRIMARY KEY ((user_id), order_date, order_id)
        ) WITH CLUSTERING ORDER BY (order_date DESC)
    """)
    
    # 2. 매터리얼라이즈드 뷰
    session.execute("""
        CREATE MATERIALIZED VIEW IF NOT EXISTS orders_by_date AS
        SELECT * FROM orders_by_user
        WHERE order_date IS NOT NULL 
        AND order_id IS NOT NULL
        AND user_id IS NOT NULL
        PRIMARY KEY ((order_date), user_id, order_id)
        WITH CLUSTERING ORDER BY (user_id ASC)
    """)
    
    # 3. 세컨더리 인덱스 (신중히 사용)
    session.execute("""
        CREATE INDEX IF NOT EXISTS idx_user_email 
        ON users (email)
    """)
    
    # 4. SASI 인덱스 (검색용)
    session.execute("""
        CREATE CUSTOM INDEX IF NOT EXISTS idx_product_name
        ON products (name)
        USING 'org.apache.cassandra.index.sasi.SASIIndex'
        WITH OPTIONS = {
            'mode': 'CONTAINS',
            'analyzer_class': 'org.apache.cassandra.index.sasi.analyzer.StandardAnalyzer'
        }
    """)

# 일관성 수준 제어
def consistency_examples():
    from cassandra import ConsistencyLevel
    
    # 강한 일관성 (모든 복제본에서 읽기/쓰기)
    session.default_consistency_level = ConsistencyLevel.ALL
    
    # 쿼럼 일관성 (과반수)
    session.default_consistency_level = ConsistencyLevel.QUORUM
    
    # 로컬 쿼럼 (같은 데이터센터 내 과반수)
    session.default_consistency_level = ConsistencyLevel.LOCAL_QUORUM
    
    # 쿼리별 일관성 수준 설정
    from cassandra.query import SimpleStatement
    
    query = SimpleStatement(
        "SELECT * FROM users WHERE user_id = %s",
        consistency_level=ConsistencyLevel.ONE
    )
    
    session.execute(query, (user_id,))
    
    # 조건부 업데이트 (경량 트랜잭션)
    result = session.execute("""
        UPDATE users
        SET email = %s
        WHERE user_id = %s
        IF email = %s
    """, (new_email, user_id, old_email))
    
    if result.one().applied:
        print("Email updated successfully")
    else:
        print("Update failed - email has changed")
```

### 4.2 HBase 예제

```python
import happybase
import struct
from datetime import datetime

# HBase 연결
connection = happybase.Connection('localhost', port=9090)

# 테이블 생성
def create_tables():
    # 사용자 테이블
    connection.create_table(
        'users',
        {
            'profile': dict(max_versions=5),
            'activity': dict(max_versions=10, compression='SNAPPY'),
            'preferences': dict()
        }
    )
    
    # 시계열 데이터 테이블
    connection.create_table(
        'timeseries',
        {
            'data': dict(
                max_versions=1,
                compression='LZ4',
                bloom_filter_type='ROW'
            )
        }
    )

# 데이터 작업
def hbase_operations():
    users_table = connection.table('users')
    
    # 행 키 설계 (사용자 ID + 타임스탬프)
    row_key = b'user:1000'
    
    # 데이터 삽입
    users_table.put(row_key, {
        b'profile:name': b'John Doe',
        b'profile:email': b'john@example.com',
        b'profile:age': struct.pack('>I', 30),  # 정수를 바이트로
        b'activity:last_login': datetime.utcnow().isoformat().encode(),
        b'preferences:theme': b'dark',
        b'preferences:language': b'en'
    })
    
    # 단일 행 조회
    row = users_table.row(row_key)
    print(f"User: {row[b'profile:name'].decode()}")
    
    # 특정 컬럼 패밀리만 조회
    row = users_table.row(row_key, columns=[b'profile'])
    
    # 스캔 (범위 조회)
    for key, data in users_table.scan(row_start=b'user:1000', row_stop=b'user:2000'):
        print(f"Key: {key}, Name: {data.get(b'profile:name', b'').decode()}")
    
    # 필터 사용
    for key, data in users_table.scan(
        filter=b"SingleColumnValueFilter('profile', 'age', >, 'binary:25')"
    ):
        print(f"Adult user: {key}")
    
    # 원자적 증가
    users_table.counter_inc(row_key, b'activity:login_count', 1)
    
    # 조건부 업데이트
    if users_table.row(row_key, columns=[b'profile:email'])[b'profile:email'] == b'john@example.com':
        users_table.put(row_key, {b'profile:email_verified': b'true'})

# 시계열 데이터 패턴
def timeseries_pattern():
    ts_table = connection.table('timeseries')
    
    # 행 키: 센서ID + 역순 타임스탬프
    def make_row_key(sensor_id, timestamp):
        # 타임스탬프를 역순으로 (최신 데이터가 먼저 오도록)
        reverse_ts = 9999999999 - int(timestamp.timestamp())
        return f"{sensor_id}:{reverse_ts}".encode()
    
    # 데이터 삽입
    sensor_id = 'sensor-001'
    now = datetime.utcnow()
    
    row_key = make_row_key(sensor_id, now)
    ts_table.put(row_key, {
        b'data:temperature': struct.pack('>f', 23.5),
        b'data:humidity': struct.pack('>f', 65.0),
        b'data:timestamp': now.isoformat().encode()
    })
    
    # 최근 N개 데이터 조회
    recent_data = []
    for key, data in ts_table.scan(
        row_start=f"{sensor_id}:".encode(),
        row_stop=f"{sensor_id}:~".encode(),
        limit=100
    ):
        temp = struct.unpack('>f', data[b'data:temperature'])[0]
        humid = struct.unpack('>f', data[b'data:humidity'])[0]
        recent_data.append({'temperature': temp, 'humidity': humid})
    
    return recent_data

# 배치 작업
def batch_operations():
    users_table = connection.table('users')
    
    # 배치 삽입
    with users_table.batch() as batch:
        for i in range(1000, 2000):
            batch.put(f'user:{i}'.encode(), {
                b'profile:name': f'User {i}'.encode(),
                b'profile:created': datetime.utcnow().isoformat().encode()
            })
    
    # 배치 삭제
    with users_table.batch() as batch:
        for i in range(1500, 1600):
            batch.delete(f'user:{i}'.encode())

# Coprocessor 시뮬레이션 (서버 사이드 처리)
def aggregation_pattern():
    # HBase는 기본적으로 집계를 지원하지 않으므로
    # 클라이언트 사이드에서 처리하거나 Coprocessor 사용
    
    users_table = connection.table('users')
    
    # 나이별 사용자 수 집계
    age_distribution = {}
    
    for key, data in users_table.scan(columns=[b'profile:age']):
        if b'profile:age' in data:
            age = struct.unpack('>I', data[b'profile:age'])[0]
            age_group = age // 10 * 10  # 10대, 20대, 30대...
            age_distribution[age_group] = age_distribution.get(age_group, 0) + 1
    
    return age_distribution
```

## 5. 그래프 데이터베이스

### 5.1 Neo4j 예제

```python
from neo4j import GraphDatabase
import json

class Neo4jConnection:
    def __init__(self, uri, user, password):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
    
    def close(self):
        self.driver.close()
    
    def execute_query(self, query, parameters=None):
        with self.driver.session() as session:
            result = session.run(query, parameters)
            return [record for record in result]

# 연결 설정
neo4j = Neo4jConnection("bolt://localhost:7687", "neo4j", "password")

# 소셜 네트워크 모델링
def social_network_example():
    # 사용자 노드 생성
    neo4j.execute_query("""
        CREATE (u1:User {id: 1, name: 'Alice', age: 30, city: 'Seoul'})
        CREATE (u2:User {id: 2, name: 'Bob', age: 28, city: 'Seoul'})
        CREATE (u3:User {id: 3, name: 'Charlie', age: 32, city: 'Busan'})
        CREATE (u4:User {id: 4, name: 'David', age: 25, city: 'Seoul'})
        CREATE (u5:User {id: 5, name: 'Eve', age: 29, city: 'Incheon'})
        
        // 친구 관계
        CREATE (u1)-[:FRIEND {since: date('2020-01-15')}]->(u2)
        CREATE (u2)-[:FRIEND {since: date('2020-01-15')}]->(u1)
        CREATE (u1)-[:FRIEND {since: date('2019-06-20')}]->(u3)
        CREATE (u3)-[:FRIEND {since: date('2019-06-20')}]->(u1)
        CREATE (u2)-[:FRIEND {since: date('2021-03-10')}]->(u4)
        CREATE (u4)-[:FRIEND {since: date('2021-03-10')}]->(u2)
        
        // 팔로우 관계
        CREATE (u1)-[:FOLLOWS {since: date('2022-01-01')}]->(u5)
        CREATE (u2)-[:FOLLOWS {since: date('2022-02-15')}]->(u5)
        CREATE (u3)-[:FOLLOWS {since: date('2022-03-20')}]->(u5)
        
        // 게시물
        CREATE (p1:Post {id: 1, content: 'Hello Neo4j!', created_at: datetime()})
        CREATE (p2:Post {id: 2, content: 'Graph databases are awesome', created_at: datetime()})
        CREATE (u1)-[:PUBLISHED]->(p1)
        CREATE (u5)-[:PUBLISHED]->(p2)
        
        // 좋아요
        CREATE (u2)-[:LIKES {at: datetime()}]->(p1)
        CREATE (u3)-[:LIKES {at: datetime()}]->(p1)
        CREATE (u1)-[:LIKES {at: datetime()}]->(p2)
    """)
    
    # 친구 추천 (친구의 친구)
    friends_of_friends = neo4j.execute_query("""
        MATCH (user:User {name: $username})-[:FRIEND]->(friend)-[:FRIEND]->(fof)
        WHERE NOT (user)-[:FRIEND]-(fof) AND user <> fof
        RETURN DISTINCT fof.name AS recommended_friend, 
               COUNT(friend) AS mutual_friends
        ORDER BY mutual_friends DESC
        LIMIT 5
    """, {'username': 'Alice'})
    
    # 최단 경로 찾기
    shortest_path = neo4j.execute_query("""
        MATCH path = shortestPath(
            (start:User {name: $start})-[*]-(end:User {name: $end})
        )
        RETURN [node IN nodes(path) | node.name] AS path,
               length(path) AS distance
    """, {'start': 'Alice', 'end': 'David'})
    
    # 영향력 분석 (PageRank)
    neo4j.execute_query("""
        CALL gds.pageRank.stream('social-network', {
            relationshipTypes: ['FOLLOWS']
        })
        YIELD nodeId, score
        MATCH (u:User) WHERE id(u) = nodeId
        RETURN u.name AS user, score
        ORDER BY score DESC
    """)

# 추천 시스템
def recommendation_system():
    # 협업 필터링
    neo4j.execute_query("""
        // 상품 구매 데이터 생성
        CREATE (p1:Product {id: 1, name: 'Laptop', category: 'Electronics'})
        CREATE (p2:Product {id: 2, name: 'Headphones', category: 'Electronics'})
        CREATE (p3:Product {id: 3, name: 'Book', category: 'Books'})
        CREATE (p4:Product {id: 4, name: 'Coffee Maker', category: 'Appliances'})
        
        MATCH (u:User {id: 1}), (p:Product {id: 1})
        CREATE (u)-[:PURCHASED {rating: 5, date: date()}]->(p)
        
        MATCH (u:User {id: 1}), (p:Product {id: 2})
        CREATE (u)-[:PURCHASED {rating: 4, date: date()}]->(p)
        
        MATCH (u:User {id: 2}), (p:Product {id: 1})
        CREATE (u)-[:PURCHASED {rating: 5, date: date()}]->(p)
        
        MATCH (u:User {id: 2}), (p:Product {id: 4})
        CREATE (u)-[:PURCHASED {rating: 4, date: date()}]->(p)
    """)
    
    # 유사한 사용자 기반 추천
    recommendations = neo4j.execute_query("""
        MATCH (target:User {id: $userId})-[:PURCHASED]->(p:Product)<-[:PURCHASED]-(other:User)
        MATCH (other)-[:PURCHASED]->(rec:Product)
        WHERE NOT (target)-[:PURCHASED]->(rec)
        RETURN rec.name AS product, 
               COUNT(DISTINCT other) AS recommended_by,
               AVG(r.rating) AS avg_rating
        ORDER BY recommended_by DESC, avg_rating DESC
        LIMIT 10
    """, {'userId': 1})
    
    # 콘텐츠 기반 추천
    content_recommendations = neo4j.execute_query("""
        MATCH (u:User {id: $userId})-[:PURCHASED]->(p:Product)
        MATCH (p)-[:HAS_TAG]->(tag:Tag)<-[:HAS_TAG]-(rec:Product)
        WHERE NOT (u)-[:PURCHASED]->(rec)
        RETURN rec.name AS product,
               COUNT(DISTINCT tag) AS common_tags,
               COLLECT(DISTINCT tag.name) AS tags
        ORDER BY common_tags DESC
        LIMIT 10
    """, {'userId': 1})

# 지식 그래프
def knowledge_graph_example():
    # 엔티티와 관계 생성
    neo4j.execute_query("""
        // 회사
        CREATE (c1:Company {name: 'TechCorp', founded: 2010, industry: 'Technology'})
        CREATE (c2:Company {name: 'DataSoft', founded: 2015, industry: 'Software'})
        
        // 사람
        CREATE (p1:Person {name: 'John Smith', title: 'CEO'})
        CREATE (p2:Person {name: 'Jane Doe', title: 'CTO'})
        CREATE (p3:Person {name: 'Bob Johnson', title: 'Developer'})
        
        // 기술
        CREATE (t1:Technology {name: 'Machine Learning'})
        CREATE (t2:Technology {name: 'Blockchain'})
        CREATE (t3:Technology {name: 'Cloud Computing'})
        
        // 관계
        CREATE (p1)-[:WORKS_AT {since: 2010, role: 'CEO'}]->(c1)
        CREATE (p2)-[:WORKS_AT {since: 2015, role: 'CTO'}]->(c1)
        CREATE (p3)-[:WORKS_AT {since: 2018, role: 'Developer'}]->(c2)
        
        CREATE (c1)-[:USES]->(t1)
        CREATE (c1)-[:USES]->(t3)
        CREATE (c2)-[:USES]->(t2)
        CREATE (c2)-[:USES]->(t3)
        
        CREATE (c1)-[:PARTNERS_WITH {since: 2020}]->(c2)
        
        CREATE (p2)-[:EXPERT_IN]->(t1)
        CREATE (p3)-[:EXPERT_IN]->(t2)
    """)
    
    # 복잡한 쿼리 - 기술 전문가 찾기
    experts = neo4j.execute_query("""
        MATCH (tech:Technology {name: $technology})<-[:USES]-(company:Company)
              <-[:WORKS_AT]-(person:Person)
        OPTIONAL MATCH (person)-[:EXPERT_IN]->(tech)
        RETURN person.name AS name,
               person.title AS title,
               company.name AS company,
               CASE WHEN (person)-[:EXPERT_IN]->(tech) THEN true ELSE false END AS is_expert
        ORDER BY is_expert DESC
    """, {'technology': 'Machine Learning'})
    
    # 네트워크 분석
    network_analysis = neo4j.execute_query("""
        CALL gds.graph.project(
            'company-network',
            ['Company', 'Person', 'Technology'],
            ['WORKS_AT', 'USES', 'PARTNERS_WITH', 'EXPERT_IN']
        )
        
        CALL gds.betweenness.stream('company-network')
        YIELD nodeId, score
        RETURN gds.util.asNode(nodeId).name AS entity, score
        ORDER BY score DESC
        LIMIT 10
    """)

# 실시간 그래프 업데이트
def real_time_updates():
    # 트랜잭션 사용
    def add_friendship(user1_id, user2_id):
        with neo4j.driver.session() as session:
            with session.begin_transaction() as tx:
                # 양방향 친구 관계 생성
                tx.run("""
                    MATCH (u1:User {id: $user1}), (u2:User {id: $user2})
                    CREATE (u1)-[:FRIEND {since: date()}]->(u2)
                    CREATE (u2)-[:FRIEND {since: date()}]->(u1)
                """, user1=user1_id, user2=user2_id)
                
                # 친구 수 업데이트
                tx.run("""
                    MATCH (u:User {id: $user1})
                    SET u.friend_count = SIZE((u)-[:FRIEND]-())
                """, user1=user1_id)
                
                tx.run("""
                    MATCH (u:User {id: $user2})
                    SET u.friend_count = SIZE((u)-[:FRIEND]-())
                """, user2=user2_id)
                
                tx.commit()
```

## 6. NoSQL 선택 가이드

### 6.1 사용 사례별 추천

```python
def nosql_selection_guide():
    use_cases = {
        "세션 관리": {
            "추천": ["Redis", "Memcached"],
            "이유": "빠른 읽기/쓰기, TTL 지원, 메모리 기반",
            "예제": """
            # Redis
            session_data = {'user_id': 1234, 'logged_in': True}
            redis_client.setex(f'session:{session_id}', 3600, json.dumps(session_data))
            """
        },
        
        "실시간 분석": {
            "추천": ["Cassandra", "HBase"],
            "이유": "시계열 데이터 최적화, 높은 쓰기 처리량",
            "예제": """
            # Cassandra
            INSERT INTO metrics (metric_id, timestamp, value)
            VALUES (?, ?, ?)
            USING TTL 86400
            """
        },
        
        "콘텐츠 관리": {
            "추천": ["MongoDB", "CouchDB"],
            "이유": "유연한 스키마, 풍부한 쿼리, 문서 저장",
            "예제": """
            # MongoDB
            articles.insert_one({
                'title': 'NoSQL Guide',
                'content': '...',
                'tags': ['database', 'nosql'],
                'metadata': {...}
            })
            """
        },
        
        "소셜 네트워크": {
            "추천": ["Neo4j", "ArangoDB"],
            "이유": "관계 중심 모델, 그래프 순회, 경로 찾기",
            "예제": """
            # Neo4j
            MATCH (a:User)-[:FRIEND*2..3]-(b:User)
            WHERE a.id = 123
            RETURN DISTINCT b
            """
        },
        
        "카탈로그/인벤토리": {
            "추천": ["MongoDB", "DynamoDB"],
            "이유": "다양한 속성, 계층 구조, 검색",
            "예제": """
            # MongoDB
            products.find({
                'category': 'electronics',
                'price': {'$lt': 1000},
                'features.wireless': true
            })
            """
        },
        
        "로그 저장": {
            "추천": ["Elasticsearch", "Cassandra"],
            "이유": "대용량 쓰기, 시간 기반 파티셔닝, 검색",
            "예제": """
            # Elasticsearch
            PUT /logs-2024.03.15/_doc/
            {
                "timestamp": "2024-03-15T10:15:30Z",
                "level": "ERROR",
                "message": "...",
                "metadata": {...}
            }
            """
        }
    }
    
    return use_cases

# 성능 벤치마크 시뮬레이션
def performance_comparison():
    import time
    import random
    
    results = {
        "Redis": {"write": [], "read": []},
        "MongoDB": {"write": [], "read": []},
        "Cassandra": {"write": [], "read": []}
    }
    
    # 시뮬레이션 (실제로는 각 DB에 연결하여 테스트)
    for db in results.keys():
        for operation in ["write", "read"]:
            for _ in range(100):
                # 실제 작업 대신 랜덤 지연 시뮬레이션
                if db == "Redis":
                    latency = random.uniform(0.1, 0.5)  # ms
                elif db == "MongoDB":
                    latency = random.uniform(1, 5)
                else:  # Cassandra
                    latency = random.uniform(2, 8)
                
                if operation == "write" and db == "Cassandra":
                    latency *= 0.7  # Cassandra는 쓰기가 더 빠름
                
                results[db][operation].append(latency)
    
    # 결과 분석
    for db, ops in results.items():
        print(f"\n{db}:")
        for op, latencies in ops.items():
            avg = sum(latencies) / len(latencies)
            print(f"  {op}: {avg:.2f}ms (avg)")
```

## 마무리

NoSQL 데이터베이스는 각각 특정 사용 사례에 최적화되어 있으며, 전통적인 RDBMS의 한계를 극복하는 다양한 접근 방식을 제공합니다. 키-값 저장소는 캐싱과 세션 관리에, 문서 저장소는 유연한 데이터 모델이 필요한 경우에, 컬럼 패밀리는 대규모 시계열 데이터에, 그래프 데이터베이스는 복잡한 관계 분석에 적합합니다. 다음 장에서는 이러한 다양한 데이터베이스 간의 연동과 통합 방법을 학습하겠습니다.