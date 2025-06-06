# 클라우드 데이터베이스

## 개요

클라우드 데이터베이스는 클라우드 환경에서 실행되는 데이터베이스 서비스로, 전통적인 온프레미스 데이터베이스와 비교하여 많은 이점을 제공합니다. 이 장에서는 주요 클라우드 데이터베이스 서비스들의 특징과 .NET 애플리케이션에서의 활용 방법을 살펴봅니다.

## 클라우드 데이터베이스의 장점

### 1. 확장성
```csharp
// Azure SQL Database 탄력적 풀 구성
public class ElasticPoolConfiguration
{
    public async Task ConfigureElasticPoolAsync()
    {
        var elasticPool = new ElasticPool
        {
            Name = "ProductionPool",
            Edition = "Standard",
            Dtu = 1200,
            DatabaseDtuMin = 10,
            DatabaseDtuMax = 100,
            StorageMB = 1228800 // 1.2TB
        };
        
        // 데이터베이스를 탄력적 풀에 추가
        await AddDatabaseToPoolAsync("CustomerDB", elasticPool);
        await AddDatabaseToPoolAsync("OrderDB", elasticPool);
        await AddDatabaseToPoolAsync("InventoryDB", elasticPool);
    }
}
```

### 2. 고가용성
```csharp
// Azure SQL Database 지역 복제 설정
public class GeoReplicationSetup
{
    private readonly string _connectionString;
    
    public async Task SetupGeoReplicationAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        // 보조 데이터베이스 생성
        var command = new SqlCommand(@"
            ALTER DATABASE [ProductionDB]
            ADD SECONDARY ON SERVER 'secondary-server.database.windows.net'
            WITH (SERVICE_OBJECTIVE = 'P2', ALLOW_CONNECTIONS = ALL);
        ", connection);
        
        await command.ExecuteNonQueryAsync();
    }
    
    // 장애 조치(Failover) 구현
    public async Task<bool> PerformFailoverAsync()
    {
        try
        {
            using var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync();
            
            var command = new SqlCommand(@"
                ALTER DATABASE [ProductionDB]
                FAILOVER
            ", connection);
            
            await command.ExecuteNonQueryAsync();
            return true;
        }
        catch (SqlException ex)
        {
            // 장애 조치 실패 처리
            LogError($"Failover failed: {ex.Message}");
            return false;
        }
    }
}
```

## Azure SQL Database

### 연결 및 구성
```csharp
public class AzureSqlConfiguration
{
    // 연결 문자열 구성
    public string GetConnectionString()
    {
        return new SqlConnectionStringBuilder
        {
            DataSource = "myserver.database.windows.net",
            InitialCatalog = "MyDatabase",
            UserID = "sqladmin",
            Password = "MyPassword123!",
            Encrypt = true,
            TrustServerCertificate = false,
            ConnectTimeout = 30,
            MultipleActiveResultSets = true
        }.ConnectionString;
    }
    
    // Azure AD 인증 사용
    public async Task<SqlConnection> GetConnectionWithAzureADAsync()
    {
        var connectionString = "Server=myserver.database.windows.net;" +
                              "Database=MyDatabase;" +
                              "Authentication=Active Directory Interactive;";
        
        var connection = new SqlConnection(connectionString);
        await connection.OpenAsync();
        return connection;
    }
}

// Entity Framework Core 구성
public class AzureSqlDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(
            "Server=myserver.database.windows.net;Database=MyDatabase;",
            options => options
                .EnableRetryOnFailure(
                    maxRetryCount: 5,
                    maxRetryDelay: TimeSpan.FromSeconds(30),
                    errorNumbersToAdd: null)
                .CommandTimeout(60));
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 클러스터드 컬럼스토어 인덱스 사용
        modelBuilder.Entity<OrderHistory>()
            .HasAnnotation("SqlServer:Clustered", false)
            .HasAnnotation("SqlServer:ColumnstoreIndex", true);
    }
}
```

### 성능 최적화
```csharp
public class AzureSqlPerformanceOptimization
{
    private readonly string _connectionString;
    
    // 쿼리 성능 개선도 추천 활용
    public async Task<List<IndexRecommendation>> GetIndexRecommendationsAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        var command = new SqlCommand(@"
            SELECT 
                d.statement as [Table],
                d.equality_columns,
                d.inequality_columns,
                d.included_columns,
                s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans) AS Score
            FROM sys.dm_db_missing_index_details d
            INNER JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
            INNER JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
            WHERE d.database_id = DB_ID()
            ORDER BY Score DESC", connection);
        
        var recommendations = new List<IndexRecommendation>();
        using var reader = await command.ExecuteReaderAsync();
        
        while (await reader.ReadAsync())
        {
            recommendations.Add(new IndexRecommendation
            {
                TableName = reader.GetString(0),
                EqualityColumns = reader.GetString(1),
                InequalityColumns = reader.IsDBNull(2) ? null : reader.GetString(2),
                IncludedColumns = reader.IsDBNull(3) ? null : reader.GetString(3),
                ImpactScore = reader.GetDouble(4)
            });
        }
        
        return recommendations;
    }
    
    // 자동 튜닝 활성화
    public async Task EnableAutomaticTuningAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        var command = new SqlCommand(@"
            ALTER DATABASE CURRENT
            SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON);
        ", connection);
        
        await command.ExecuteNonQueryAsync();
    }
}
```

## Azure Cosmos DB

### 기본 설정과 사용
```csharp
public class CosmosDbService
{
    private readonly CosmosClient _cosmosClient;
    private readonly Container _container;
    
    public CosmosDbService(string connectionString, string databaseId, string containerId)
    {
        _cosmosClient = new CosmosClient(connectionString, new CosmosClientOptions
        {
            SerializerOptions = new CosmosSerializationOptions
            {
                PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
            },
            ConnectionMode = ConnectionMode.Direct,
            MaxRetryAttemptsOnRateLimitedRequests = 9,
            MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30)
        });
        
        _container = _cosmosClient.GetContainer(databaseId, containerId);
    }
    
    // 문서 생성
    public async Task<T> CreateItemAsync<T>(T item, string partitionKey)
    {
        try
        {
            var response = await _container.CreateItemAsync(
                item,
                new PartitionKey(partitionKey),
                new ItemRequestOptions
                {
                    ConsistencyLevel = ConsistencyLevel.Session
                });
            
            Console.WriteLine($"Created item with RU charge: {response.RequestCharge}");
            return response.Resource;
        }
        catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.Conflict)
        {
            Console.WriteLine("Item already exists");
            throw;
        }
    }
    
    // 쿼리 실행
    public async Task<List<T>> QueryItemsAsync<T>(string sqlQuery, Dictionary<string, object> parameters = null)
    {
        var queryDefinition = new QueryDefinition(sqlQuery);
        
        if (parameters != null)
        {
            foreach (var param in parameters)
            {
                queryDefinition.WithParameter($"@{param.Key}", param.Value);
            }
        }
        
        var queryIterator = _container.GetItemQueryIterator<T>(
            queryDefinition,
            requestOptions: new QueryRequestOptions
            {
                MaxConcurrency = 4,
                MaxItemCount = 100
            });
        
        var results = new List<T>();
        double totalRU = 0;
        
        while (queryIterator.HasMoreResults)
        {
            var response = await queryIterator.ReadNextAsync();
            results.AddRange(response);
            totalRU += response.RequestCharge;
        }
        
        Console.WriteLine($"Query consumed {totalRU} RUs");
        return results;
    }
}

// 변경 피드 처리
public class ChangeFeedProcessor
{
    private readonly Container _monitoredContainer;
    private readonly Container _leaseContainer;
    
    public async Task StartChangeFeedProcessorAsync()
    {
        var changeFeedProcessor = _monitoredContainer
            .GetChangeFeedProcessorBuilder<Product>(
                processorName: "productChangeFeed",
                onChangesDelegate: HandleChangesAsync)
            .WithInstanceName("instance1")
            .WithLeaseContainer(_leaseContainer)
            .WithStartTime(DateTime.UtcNow.AddHours(-1))
            .Build();
        
        await changeFeedProcessor.StartAsync();
    }
    
    private async Task HandleChangesAsync(
        ChangeFeedProcessorContext context,
        IReadOnlyCollection<Product> changes,
        CancellationToken cancellationToken)
    {
        foreach (var change in changes)
        {
            Console.WriteLine($"Detected change for product: {change.Id}");
            
            // 변경 사항 처리 (예: 캐시 무효화, 이벤트 발행 등)
            await ProcessProductChangeAsync(change);
        }
    }
}
```

### 파티션 전략
```csharp
public class PartitioningStrategy
{
    // 계층적 파티션 키 설계
    public class HierarchicalPartitionKey
    {
        public string GetPartitionKey(string tenantId, string category, string subcategory)
        {
            // 복합 파티션 키 생성
            return $"{tenantId}#{category}#{subcategory}";
        }
        
        public async Task<Container> CreateContainerWithHierarchicalPartitionAsync(
            Database database,
            string containerId)
        {
            var containerProperties = new ContainerProperties
            {
                Id = containerId,
                PartitionKeyPaths = new List<string> { "/tenantId", "/category", "/subcategory" },
                DefaultTimeToLive = 3600 // 1시간 TTL
            };
            
            // 인덱싱 정책 구성
            containerProperties.IndexingPolicy = new IndexingPolicy
            {
                Automatic = true,
                IndexingMode = IndexingMode.Consistent,
                IncludedPaths = { new IncludedPath { Path = "/*" } },
                ExcludedPaths = { new ExcludedPath { Path = "/metadata/*" } }
            };
            
            // 처리량 자동 스케일링 설정
            var throughputProperties = ThroughputProperties.CreateAutoscaleThroughput(4000);
            
            return await database.CreateContainerAsync(containerProperties, throughputProperties);
        }
    }
    
    // 파티션 키 값 분산 모니터링
    public async Task<Dictionary<string, long>> GetPartitionDistributionAsync()
    {
        var query = new QueryDefinition(
            "SELECT c.partitionKey, COUNT(1) as count FROM c GROUP BY c.partitionKey");
        
        var distribution = new Dictionary<string, long>();
        var iterator = _container.GetItemQueryIterator<dynamic>(query);
        
        while (iterator.HasMoreResults)
        {
            var response = await iterator.ReadNextAsync();
            foreach (var item in response)
            {
                distribution[item.partitionKey] = item.count;
            }
        }
        
        return distribution;
    }
}
```

## AWS RDS

### RDS 인스턴스 연결
```csharp
public class AwsRdsService
{
    private readonly string _connectionString;
    
    public AwsRdsService()
    {
        // RDS MySQL 연결
        _connectionString = new MySqlConnectionStringBuilder
        {
            Server = "myinstance.c1234567890.us-east-1.rds.amazonaws.com",
            Port = 3306,
            Database = "mydb",
            UserID = "admin",
            Password = "MyPassword123!",
            SslMode = MySqlSslMode.Required,
            ConnectionTimeout = 30,
            DefaultCommandTimeout = 60
        }.ConnectionString;
    }
    
    // IAM 데이터베이스 인증 사용
    public async Task<MySqlConnection> GetConnectionWithIAMAsync()
    {
        var awsCredentials = new EnvironmentVariablesAWSCredentials();
        var token = await GenerateRDSAuthTokenAsync(awsCredentials);
        
        var connectionString = new MySqlConnectionStringBuilder
        {
            Server = "myinstance.c1234567890.us-east-1.rds.amazonaws.com",
            Port = 3306,
            Database = "mydb",
            UserID = "iam_user",
            Password = token,
            SslMode = MySqlSslMode.Required
        }.ConnectionString;
        
        var connection = new MySqlConnection(connectionString);
        await connection.OpenAsync();
        return connection;
    }
    
    // 읽기 전용 복제본 활용
    public class ReadWriteSplitService
    {
        private readonly string _writeConnectionString;
        private readonly List<string> _readConnectionStrings;
        private int _currentReadIndex = 0;
        
        public ReadWriteSplitService()
        {
            _writeConnectionString = "Server=master.rds.amazonaws.com;...";
            _readConnectionStrings = new List<string>
            {
                "Server=read-replica-1.rds.amazonaws.com;...",
                "Server=read-replica-2.rds.amazonaws.com;...",
                "Server=read-replica-3.rds.amazonaws.com;..."
            };
        }
        
        public async Task<T> ExecuteReadAsync<T>(Func<MySqlConnection, Task<T>> operation)
        {
            // 라운드 로빈 방식으로 읽기 복제본 선택
            var connectionString = _readConnectionStrings[_currentReadIndex];
            _currentReadIndex = (_currentReadIndex + 1) % _readConnectionStrings.Count;
            
            using var connection = new MySqlConnection(connectionString);
            await connection.OpenAsync();
            return await operation(connection);
        }
        
        public async Task<T> ExecuteWriteAsync<T>(Func<MySqlConnection, Task<T>> operation)
        {
            using var connection = new MySqlConnection(_writeConnectionString);
            await connection.OpenAsync();
            return await operation(connection);
        }
    }
}
```

### RDS 프록시 활용
```csharp
public class RdsProxyConfiguration
{
    private readonly string _proxyEndpoint = "myproxy.proxy-c1234567890.us-east-1.rds.amazonaws.com";
    
    public string GetProxyConnectionString()
    {
        return new MySqlConnectionStringBuilder
        {
            Server = _proxyEndpoint,
            Port = 3306,
            Database = "mydb",
            UserID = "proxyuser",
            Password = "ProxyPassword123!",
            SslMode = MySqlSslMode.Required,
            Pooling = true,
            MinimumPoolSize = 5,
            MaximumPoolSize = 100
        }.ConnectionString;
    }
    
    // 연결 풀 모니터링
    public class ConnectionPoolMonitor
    {
        private readonly Timer _timer;
        private readonly MySqlConnection _connection;
        
        public ConnectionPoolMonitor(string connectionString)
        {
            _connection = new MySqlConnection(connectionString);
            _timer = new Timer(MonitorConnectionPool, null, TimeSpan.Zero, TimeSpan.FromMinutes(1));
        }
        
        private void MonitorConnectionPool(object state)
        {
            var poolingInfo = MySqlConnection.ClearAllPools();
            
            Console.WriteLine($"Active Connections: {poolingInfo.ActiveConnections}");
            Console.WriteLine($"Available Connections: {poolingInfo.AvailableConnections}");
            Console.WriteLine($"Total Connections: {poolingInfo.TotalConnections}");
            
            // CloudWatch 메트릭으로 전송
            SendMetricToCloudWatch("ConnectionPool.Active", poolingInfo.ActiveConnections);
            SendMetricToCloudWatch("ConnectionPool.Available", poolingInfo.AvailableConnections);
        }
    }
}
```

## Google Cloud SQL

### Cloud SQL 연결 구성
```csharp
public class GoogleCloudSqlService
{
    private readonly string _connectionString;
    
    // Unix 소켓 연결 (Cloud Run/App Engine)
    public string GetUnixSocketConnection()
    {
        var connectionString = new NpgsqlConnectionStringBuilder
        {
            Host = "/cloudsql/project:region:instance",
            Database = "mydatabase",
            Username = "postgres",
            Password = "MyPassword123!",
            SslMode = SslMode.Disable,
            Pooling = true,
            MinPoolSize = 0,
            MaxPoolSize = 100
        }.ConnectionString;
        
        return connectionString;
    }
    
    // Cloud SQL 프록시를 통한 연결
    public string GetProxyConnection()
    {
        return new NpgsqlConnectionStringBuilder
        {
            Host = "127.0.0.1",
            Port = 5432,
            Database = "mydatabase",
            Username = "postgres",
            Password = "MyPassword123!",
            SslMode = SslMode.Require,
            TrustServerCertificate = true
        }.ConnectionString;
    }
    
    // Private IP 연결
    public async Task<NpgsqlConnection> GetPrivateIpConnectionAsync()
    {
        var connectionString = new NpgsqlConnectionStringBuilder
        {
            Host = "10.20.30.40", // Private IP
            Port = 5432,
            Database = "mydatabase",
            Username = "postgres",
            Password = "MyPassword123!",
            SslMode = SslMode.Require,
            RootCertificate = "/path/to/server-ca.pem",
            ClientCertificate = "/path/to/client-cert.pem",
            ClientCertificateKey = "/path/to/client-key.pem"
        }.ConnectionString;
        
        var connection = new NpgsqlConnection(connectionString);
        await connection.OpenAsync();
        return connection;
    }
}

// Cloud SQL 자동 백업 및 복원
public class CloudSqlBackupService
{
    private readonly CloudSqlAdminService _adminService;
    
    public async Task<BackupRun> CreateOnDemandBackupAsync(string project, string instance)
    {
        var request = new SqlBackupRunsInsertRequest
        {
            Project = project,
            Instance = instance,
            Body = new BackupRun
            {
                Description = $"On-demand backup {DateTime.UtcNow:yyyy-MM-dd}",
                Type = "ON_DEMAND"
            }
        };
        
        var response = await _adminService.BackupRuns.Insert(request).ExecuteAsync();
        return response;
    }
    
    public async Task RestoreFromBackupAsync(string project, string instance, long backupId)
    {
        var request = new InstancesRestoreBackupRequest
        {
            Project = project,
            Instance = instance,
            Body = new RestoreBackupContext
            {
                BackupRunId = backupId,
                TargetInstanceId = instance
            }
        };
        
        await _adminService.Instances.RestoreBackup(request).ExecuteAsync();
    }
}
```

## 서버리스 데이터베이스

### Azure Cosmos DB Serverless
```csharp
public class CosmosDbServerlessService
{
    private readonly CosmosClient _client;
    private readonly Container _container;
    
    public CosmosDbServerlessService()
    {
        // 서버리스 모드 구성
        _client = new CosmosClient(
            connectionString: Environment.GetEnvironmentVariable("COSMOS_CONNECTION_STRING"),
            clientOptions: new CosmosClientOptions
            {
                ConsistencyLevel = ConsistencyLevel.Eventual,
                MaxRetryAttemptsOnRateLimitedRequests = 3,
                MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(10)
            });
        
        var database = _client.GetDatabase("ServerlessDB");
        _container = database.GetContainer("Items");
    }
    
    // 비용 최적화된 배치 작업
    public async Task<BulkOperationResponse> BulkInsertOptimizedAsync<T>(List<T> items)
    {
        var bulkOperations = new List<Task<ResponseMessage>>();
        var batchSize = 100; // 작은 배치로 나누어 처리
        
        for (int i = 0; i < items.Count; i += batchSize)
        {
            var batch = items.Skip(i).Take(batchSize);
            
            foreach (var item in batch)
            {
                bulkOperations.Add(_container.CreateItemStreamAsync(
                    JsonSerializer.SerializeToUtf8Bytes(item),
                    new PartitionKey(GetPartitionKey(item))));
            }
            
            // 배치 실행 및 대기
            if (bulkOperations.Count >= batchSize)
            {
                await ProcessBatchAsync(bulkOperations);
                bulkOperations.Clear();
                
                // 처리량 제한 방지를 위한 대기
                await Task.Delay(100);
            }
        }
        
        // 남은 항목 처리
        if (bulkOperations.Any())
        {
            await ProcessBatchAsync(bulkOperations);
        }
        
        return new BulkOperationResponse { Success = true };
    }
}
```

### AWS Aurora Serverless v2
```csharp
public class AuroraServerlessService
{
    private readonly string _clusterEndpoint;
    private readonly IAmazonRDS _rdsClient;
    
    public AuroraServerlessService()
    {
        _clusterEndpoint = "myserverlesscluster.cluster-123456.us-east-1.rds.amazonaws.com";
        _rdsClient = new AmazonRDSClient();
    }
    
    // Data API 사용 (HTTP 엔드포인트)
    public async Task<ExecuteStatementResponse> ExecuteStatementAsync(string sql, List<SqlParameter> parameters = null)
    {
        var request = new ExecuteStatementRequest
        {
            ResourceArn = "arn:aws:rds:us-east-1:123456789012:cluster:myserverlesscluster",
            SecretArn = "arn:aws:secretsmanager:us-east-1:123456789012:secret:rds-secret",
            Database = "mydb",
            Sql = sql
        };
        
        if (parameters != null)
        {
            request.Parameters = parameters.Select(p => new SqlParameter
            {
                Name = p.Name,
                Value = new Field { StringValue = p.Value?.ToString() }
            }).ToList();
        }
        
        var dataClient = new AmazonRDSDataServiceClient();
        return await dataClient.ExecuteStatementAsync(request);
    }
    
    // 자동 일시 정지/재개 모니터링
    public async Task<ClusterStatus> MonitorClusterStatusAsync()
    {
        var describeRequest = new DescribeDBClustersRequest
        {
            DBClusterIdentifier = "myserverlesscluster"
        };
        
        var response = await _rdsClient.DescribeDBClustersAsync(describeRequest);
        var cluster = response.DBClusters.FirstOrDefault();
        
        return new ClusterStatus
        {
            Status = cluster.Status,
            Capacity = cluster.ServerlessV2ScalingConfiguration.MinCapacity,
            IsPaused = cluster.Status == "stopped"
        };
    }
}
```

## 마이그레이션 전략

### 온프레미스에서 클라우드로
```csharp
public class DatabaseMigrationService
{
    // AWS Database Migration Service 활용
    public async Task MigrateToCloudAsync(MigrationConfig config)
    {
        var dmsClient = new AmazonDatabaseMigrationServiceClient();
        
        // 복제 인스턴스 생성
        var replicationInstance = await CreateReplicationInstanceAsync(dmsClient, config);
        
        // 소스 및 대상 엔드포인트 생성
        var sourceEndpoint = await CreateSourceEndpointAsync(dmsClient, config.Source);
        var targetEndpoint = await CreateTargetEndpointAsync(dmsClient, config.Target);
        
        // 마이그레이션 태스크 생성
        var migrationTask = new CreateReplicationTaskRequest
        {
            ReplicationTaskIdentifier = $"migration-{DateTime.UtcNow:yyyyMMddHHmmss}",
            SourceEndpointArn = sourceEndpoint.EndpointArn,
            TargetEndpointArn = targetEndpoint.EndpointArn,
            ReplicationInstanceArn = replicationInstance.ReplicationInstanceArn,
            MigrationType = MigrationType.FullLoadAndCdc,
            TableMappings = GetTableMappings(config),
            ReplicationTaskSettings = GetTaskSettings(config)
        };
        
        await dmsClient.CreateReplicationTaskAsync(migrationTask);
    }
    
    // 변경 데이터 캡처(CDC) 구성
    private string GetTableMappings(MigrationConfig config)
    {
        var mappings = new
        {
            rules = new[]
            {
                new
                {
                    @object_locator = new
                    {
                        schema_name = config.SourceSchema,
                        table_name = "%"
                    },
                    rule_type = "selection",
                    rule_id = "1",
                    rule_name = "1",
                    rule_action = "include"
                },
                new
                {
                    @object_locator = new
                    {
                        schema_name = config.SourceSchema,
                        table_name = "%"
                    },
                    rule_type = "transformation",
                    rule_id = "2",
                    rule_name = "2",
                    rule_action = "rename",
                    rule_target = "schema",
                    value = config.TargetSchema
                }
            }
        };
        
        return JsonSerializer.Serialize(mappings);
    }
}

// 무중단 마이그레이션 전략
public class ZeroDowntimeMigration
{
    private readonly ILogger<ZeroDowntimeMigration> _logger;
    
    public async Task ExecuteMigrationAsync()
    {
        // 1단계: 읽기 전용 복제본 설정
        await SetupReadReplicaAsync();
        
        // 2단계: 애플리케이션을 읽기 전용 모드로 전환
        await SwitchToReadOnlyModeAsync();
        
        // 3단계: 최종 데이터 동기화
        await FinalDataSyncAsync();
        
        // 4단계: DNS 전환
        await UpdateDnsToPointToCloudAsync();
        
        // 5단계: 애플리케이션 정상 모드 복원
        await RestoreNormalModeAsync();
        
        // 6단계: 이전 데이터베이스 연결 정리
        await CleanupOldConnectionsAsync();
    }
    
    private async Task SwitchToReadOnlyModeAsync()
    {
        // 애플리케이션 설정 업데이트
        Environment.SetEnvironmentVariable("DATABASE_MODE", "READONLY");
        
        // 활성 쓰기 트랜잭션 대기
        await WaitForActiveTransactionsAsync();
        
        _logger.LogInformation("Application switched to read-only mode");
    }
}
```

## 모니터링 및 관리

### 통합 모니터링
```csharp
public class CloudDatabaseMonitoring
{
    private readonly IMetricsPublisher _metricsPublisher;
    
    // 멀티 클라우드 메트릭 수집
    public async Task CollectAndPublishMetricsAsync()
    {
        var tasks = new List<Task>
        {
            CollectAzureSqlMetricsAsync(),
            CollectCosmosDbMetricsAsync(),
            CollectAwsRdsMetricsAsync(),
            CollectGoogleCloudSqlMetricsAsync()
        };
        
        await Task.WhenAll(tasks);
    }
    
    private async Task CollectAzureSqlMetricsAsync()
    {
        using var connection = new SqlConnection(_azureSqlConnectionString);
        await connection.OpenAsync();
        
        // DTU 사용률 확인
        var dtuCommand = new SqlCommand(@"
            SELECT TOP 1 
                end_time,
                avg_cpu_percent,
                avg_data_io_percent,
                avg_log_write_percent,
                avg_memory_usage_percent,
                dtu_used = GREATEST(avg_cpu_percent, avg_data_io_percent, avg_log_write_percent)
            FROM sys.dm_db_resource_stats
            ORDER BY end_time DESC", connection);
        
        using var reader = await dtuCommand.ExecuteReaderAsync();
        if (await reader.ReadAsync())
        {
            await _metricsPublisher.PublishAsync(new Metric
            {
                Name = "AzureSQL.DTU.Usage",
                Value = reader.GetDouble(5),
                Tags = new Dictionary<string, string>
                {
                    ["database"] = connection.Database,
                    ["metric_type"] = "dtu"
                }
            });
        }
    }
    
    // 비용 최적화 추천
    public async Task<List<CostOptimizationRecommendation>> GetCostOptimizationRecommendationsAsync()
    {
        var recommendations = new List<CostOptimizationRecommendation>();
        
        // 사용률이 낮은 리소스 확인
        var underutilizedResources = await FindUnderutilizedResourcesAsync();
        foreach (var resource in underutilizedResources)
        {
            recommendations.Add(new CostOptimizationRecommendation
            {
                ResourceType = resource.Type,
                ResourceName = resource.Name,
                CurrentCost = resource.MonthlyCost,
                RecommendedAction = "Downgrade or switch to serverless",
                PotentialSavings = resource.MonthlyCost * 0.4m
            });
        }
        
        // 예약 용량 추천
        var reservationCandidates = await FindReservationCandidatesAsync();
        foreach (var candidate in reservationCandidates)
        {
            recommendations.Add(new CostOptimizationRecommendation
            {
                ResourceType = candidate.Type,
                ResourceName = candidate.Name,
                CurrentCost = candidate.MonthlyCost,
                RecommendedAction = "Purchase reserved capacity",
                PotentialSavings = candidate.MonthlyCost * 0.3m
            });
        }
        
        return recommendations;
    }
}

// 알림 및 자동 대응
public class AutomatedResponseService
{
    private readonly Dictionary<string, Func<Alert, Task>> _responseHandlers;
    
    public AutomatedResponseService()
    {
        _responseHandlers = new Dictionary<string, Func<Alert, Task>>
        {
            ["HighDTU"] = HandleHighDtuAsync,
            ["StorageFull"] = HandleStorageFullAsync,
            ["ConnectionLimit"] = HandleConnectionLimitAsync,
            ["QueryTimeout"] = HandleQueryTimeoutAsync
        };
    }
    
    private async Task HandleHighDtuAsync(Alert alert)
    {
        // DTU가 90% 이상일 때 자동 스케일 업
        if (alert.Value > 90)
        {
            await ScaleDatabaseTierAsync(alert.ResourceId, ScaleDirection.Up);
            
            // 일정 시간 후 사용률 확인하여 다시 스케일 다운
            _ = Task.Run(async () =>
            {
                await Task.Delay(TimeSpan.FromHours(1));
                var currentUsage = await GetCurrentDtuUsageAsync(alert.ResourceId);
                if (currentUsage < 50)
                {
                    await ScaleDatabaseTierAsync(alert.ResourceId, ScaleDirection.Down);
                }
            });
        }
    }
}
```

## 보안 모범 사례

### 암호화 및 접근 제어
```csharp
public class CloudDatabaseSecurity
{
    // 투명한 데이터 암호화(TDE) 구성
    public async Task EnableTransparentDataEncryptionAsync(string databaseId)
    {
        // Azure SQL Database
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        var command = new SqlCommand(@"
            ALTER DATABASE CURRENT
            SET ENCRYPTION ON;
        ", connection);
        
        await command.ExecuteNonQueryAsync();
    }
    
    // 고객 관리 키(CMK) 구성
    public async Task ConfigureCustomerManagedKeyAsync()
    {
        // Azure Key Vault 통합
        var keyVaultClient = new KeyClient(
            new Uri("https://myvault.vault.azure.net/"),
            new DefaultAzureCredential());
        
        // 키 생성 또는 가져오기
        var key = await keyVaultClient.GetKeyAsync("database-encryption-key");
        
        // SQL Database에 CMK 설정
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        var command = new SqlCommand($@"
            ALTER DATABASE CURRENT
            SET ENCRYPTION ON
            WITH (
                ENCRYPTION_TYPE = CUSTOMER_MANAGED_KEY,
                KEY_STORE_PROVIDER = 'AzureKeyVault',
                KEY_PATH = '{key.Value.Id}'
            );
        ", connection);
        
        await command.ExecuteNonQueryAsync();
    }
    
    // 동적 데이터 마스킹
    public async Task ConfigureDynamicDataMaskingAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        // 이메일 마스킹
        var emailMask = new SqlCommand(@"
            ALTER TABLE Customers
            ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
        ", connection);
        await emailMask.ExecuteNonQueryAsync();
        
        // 신용카드 마스킹
        var creditCardMask = new SqlCommand(@"
            ALTER TABLE Payments
            ALTER COLUMN CreditCardNumber ADD MASKED WITH (FUNCTION = 'partial(0,XXXX-XXXX-XXXX-,4)');
        ", connection);
        await creditCardMask.ExecuteNonQueryAsync();
    }
}
```

## 마무리

클라우드 데이터베이스는 현대 애플리케이션 개발에서 필수적인 요소가 되었습니다. 각 클라우드 제공업체의 서비스는 고유한 특징과 장점을 가지고 있으며, 애플리케이션의 요구사항에 따라 적절한 서비스를 선택하는 것이 중요합니다.

주요 고려사항:
- **비용 최적화**: 사용량 기반 가격 모델을 이해하고 적절한 최적화 전략 수립
- **성능**: 자동 스케일링, 캐싱, 읽기 복제본 활용
- **보안**: 암호화, 접근 제어, 규정 준수
- **가용성**: 지역 복제, 자동 백업, 재해 복구 계획
- **마이그레이션**: 무중단 마이그레이션 전략과 도구 활용

클라우드 데이터베이스를 효과적으로 활용하면 인프라 관리 부담을 줄이고 애플리케이션 개발에 집중할 수 있습니다.