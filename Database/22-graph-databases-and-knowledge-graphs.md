# 그래프 데이터베이스와 지식 그래프

## 개요

그래프 데이터베이스는 노드(정점)와 엣지(간선)로 구성된 그래프 구조로 데이터를 저장하고 쿼리하는 데이터베이스입니다. 복잡한 관계를 효율적으로 표현하고 탐색할 수 있어 소셜 네트워크, 추천 시스템, 지식 그래프, 사기 탐지 등 다양한 분야에서 활용됩니다. 이 장에서는 주요 그래프 데이터베이스와 지식 그래프 구축 방법을 학습합니다.

## 1. Neo4j 그래프 데이터베이스

### 1.1 Neo4j 기본 개념과 Cypher 쿼리

```python
from neo4j import GraphDatabase
import pandas as pd
from datetime import datetime
import networkx as nx
from typing import List, Dict, Any

class Neo4jGraphDB:
    """Neo4j 그래프 데이터베이스 관리"""
    
    def __init__(self, uri="bolt://localhost:7687", user="neo4j", password="password"):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
        
    def close(self):
        self.driver.close()
        
    def create_social_network_schema(self):
        """소셜 네트워크 스키마 생성"""
        with self.driver.session() as session:
            # 제약조건 생성
            constraints = [
                "CREATE CONSTRAINT user_id IF NOT EXISTS ON (u:User) ASSERT u.id IS UNIQUE",
                "CREATE CONSTRAINT post_id IF NOT EXISTS ON (p:Post) ASSERT p.id IS UNIQUE",
                "CREATE CONSTRAINT tag_name IF NOT EXISTS ON (t:Tag) ASSERT t.name IS UNIQUE"
            ]
            
            for constraint in constraints:
                session.run(constraint)
            
            # 인덱스 생성
            indexes = [
                "CREATE INDEX user_email IF NOT EXISTS FOR (u:User) ON (u.email)",
                "CREATE INDEX post_timestamp IF NOT EXISTS FOR (p:Post) ON (p.timestamp)",
                "CREATE INDEX user_location IF NOT EXISTS FOR (u:User) ON (u.location)"
            ]
            
            for index in indexes:
                session.run(index)
            
            # 전문 검색 인덱스
            session.run("""
                CREATE FULLTEXT INDEX post_content IF NOT EXISTS 
                FOR (p:Post) ON EACH [p.content, p.title]
            """)
    
    def build_social_graph(self):
        """소셜 그래프 구축"""
        with self.driver.session() as session:
            # 사용자 생성
            session.run("""
                UNWIND $users AS user
                MERGE (u:User {id: user.id})
                SET u.name = user.name,
                    u.email = user.email,
                    u.created_at = datetime(user.created_at),
                    u.location = point({latitude: user.lat, longitude: user.lon}),
                    u.interests = user.interests
            """, users=self.generate_sample_users())
            
            # 팔로우 관계 생성
            session.run("""
                MATCH (u1:User), (u2:User)
                WHERE u1.id < u2.id AND rand() < 0.1
                CREATE (u1)-[:FOLLOWS {since: datetime()}]->(u2)
            """)
            
            # 친구 관계 생성 (양방향)
            session.run("""
                MATCH (u1:User), (u2:User)
                WHERE u1.id < u2.id AND rand() < 0.05
                CREATE (u1)-[:FRIEND {since: datetime()}]->(u2)
                CREATE (u2)-[:FRIEND {since: datetime()}]->(u1)
            """)
            
            # 게시물 및 상호작용 생성
            session.run("""
                MATCH (u:User)
                WITH u, range(1, toInteger(rand() * 10)) AS posts
                UNWIND posts AS post
                CREATE (p:Post {
                    id: randomUUID(),
                    content: 'Post content ' + toString(post),
                    timestamp: datetime() - duration({days: toInteger(rand() * 30)}),
                    likes: toInteger(rand() * 100),
                    shares: toInteger(rand() * 20)
                })
                CREATE (u)-[:POSTED]->(p)
                
                // 좋아요 관계
                WITH p, u
                MATCH (other:User)
                WHERE other <> u AND rand() < 0.2
                CREATE (other)-[:LIKES {timestamp: datetime()}]->(p)
                
                // 댓글 관계
                WITH p, u
                MATCH (commenter:User)
                WHERE commenter <> u AND rand() < 0.1
                CREATE (c:Comment {
                    id: randomUUID(),
                    text: 'Great post!',
                    timestamp: datetime()
                })
                CREATE (commenter)-[:COMMENTED]->(c)-[:ON]->(p)
            """)
    
    def friend_recommendation_algorithm(self, user_id: str, limit: int = 10):
        """친구 추천 알고리즘"""
        with self.driver.session() as session:
            # 친구의 친구 기반 추천
            result = session.run("""
                MATCH (user:User {id: $user_id})-[:FRIEND]-(friend:User)
                MATCH (friend)-[:FRIEND]-(foaf:User)
                WHERE NOT (user)-[:FRIEND]-(foaf) AND user <> foaf
                WITH foaf, COUNT(DISTINCT friend) AS mutual_friends
                ORDER BY mutual_friends DESC
                LIMIT $limit
                RETURN foaf.id AS user_id, 
                       foaf.name AS name,
                       mutual_friends,
                       // 공통 관심사 계산
                       size([interest IN foaf.interests WHERE interest IN user.interests]) AS common_interests
            """, user_id=user_id, limit=limit)
            
            recommendations = []
            for record in result:
                recommendations.append({
                    'user_id': record['user_id'],
                    'name': record['name'],
                    'mutual_friends': record['mutual_friends'],
                    'common_interests': record['common_interests'],
                    'score': record['mutual_friends'] * 2 + record['common_interests']
                })
            
            return sorted(recommendations, key=lambda x: x['score'], reverse=True)
    
    def detect_communities(self):
        """커뮤니티 감지"""
        with self.driver.session() as session:
            # Louvain 알고리즘을 사용한 커뮤니티 감지
            session.run("""
                CALL gds.graph.project(
                    'social-network',
                    'User',
                    {
                        FRIEND: {orientation: 'UNDIRECTED'},
                        FOLLOWS: {orientation: 'NATURAL'}
                    }
                )
            """)
            
            # 커뮤니티 감지 실행
            result = session.run("""
                CALL gds.louvain.stream('social-network')
                YIELD nodeId, communityId
                WITH gds.util.asNode(nodeId) AS user, communityId
                WITH communityId, 
                     collect(user) AS members,
                     COUNT(*) AS size
                WHERE size > 5
                RETURN communityId,
                       size,
                       [member IN members | member.name][0..10] AS sample_members
                ORDER BY size DESC
            """)
            
            communities = []
            for record in result:
                communities.append({
                    'id': record['communityId'],
                    'size': record['size'],
                    'sample_members': record['sample_members']
                })
            
            return communities
    
    def shortest_path_analysis(self, start_user: str, end_user: str):
        """최단 경로 분석"""
        with self.driver.session() as session:
            # 다양한 최단 경로 알고리즘
            
            # 1. 단순 최단 경로
            simple_path = session.run("""
                MATCH path = shortestPath(
                    (start:User {id: $start_user})-[*]-(end:User {id: $end_user})
                )
                RETURN length(path) AS distance,
                       [node IN nodes(path) | node.name] AS path_names,
                       [rel IN relationships(path) | type(rel)] AS relationship_types
            """, start_user=start_user, end_user=end_user).single()
            
            # 2. 가중치 기반 최단 경로 (Dijkstra)
            weighted_path = session.run("""
                MATCH (start:User {id: $start_user}), (end:User {id: $end_user})
                CALL gds.shortestPath.dijkstra.stream('social-network', {
                    sourceNode: start,
                    targetNode: end,
                    relationshipWeightProperty: 'weight'
                })
                YIELD nodeIds, costs, totalCost
                RETURN totalCost,
                       [nodeId IN nodeIds | gds.util.asNode(nodeId).name] AS path
            """, start_user=start_user, end_user=end_user).single()
            
            # 3. 모든 최단 경로
            all_paths = session.run("""
                MATCH paths = allShortestPaths(
                    (start:User {id: $start_user})-[*]-(end:User {id: $end_user})
                )
                RETURN COLLECT([node IN nodes(paths) | node.name]) AS all_paths
            """, start_user=start_user, end_user=end_user).single()
            
            return {
                'simple_shortest': {
                    'distance': simple_path['distance'] if simple_path else None,
                    'path': simple_path['path_names'] if simple_path else None
                },
                'weighted_shortest': {
                    'cost': weighted_path['totalCost'] if weighted_path else None,
                    'path': weighted_path['path'] if weighted_path else None
                },
                'all_shortest_paths': all_paths['all_paths'] if all_paths else []
            }
```

### 1.2 고급 그래프 알고리즘

```python
class GraphAlgorithms:
    """고급 그래프 알고리즘"""
    
    def __init__(self, driver):
        self.driver = driver
        
    def pagerank_analysis(self):
        """PageRank 분석"""
        with self.driver.session() as session:
            # PageRank 계산
            result = session.run("""
                CALL gds.pageRank.stream('social-network', {
                    maxIterations: 20,
                    dampingFactor: 0.85
                })
                YIELD nodeId, score
                WITH gds.util.asNode(nodeId) AS user, score
                ORDER BY score DESC
                LIMIT 20
                RETURN user.id AS user_id,
                       user.name AS name,
                       score AS pagerank,
                       size((user)-[:FOLLOWS]-()) AS followers,
                       size((user)-[:POSTED]->()) AS posts
            """)
            
            influential_users = []
            for record in result:
                influential_users.append({
                    'user_id': record['user_id'],
                    'name': record['name'],
                    'pagerank': record['pagerank'],
                    'followers': record['followers'],
                    'posts': record['posts']
                })
            
            return influential_users
    
    def centrality_metrics(self):
        """중심성 메트릭 계산"""
        with self.driver.session() as session:
            # 다양한 중심성 메트릭
            metrics = {}
            
            # 1. 매개 중심성 (Betweenness Centrality)
            betweenness = session.run("""
                CALL gds.betweenness.stream('social-network')
                YIELD nodeId, score
                WITH gds.util.asNode(nodeId) AS user, score
                ORDER BY score DESC
                LIMIT 10
                RETURN user.name AS name, score AS betweenness
            """).data()
            metrics['betweenness'] = betweenness
            
            # 2. 근접 중심성 (Closeness Centrality)
            closeness = session.run("""
                CALL gds.closeness.stream('social-network')
                YIELD nodeId, score
                WITH gds.util.asNode(nodeId) AS user, score
                ORDER BY score DESC
                LIMIT 10
                RETURN user.name AS name, score AS closeness
            """).data()
            metrics['closeness'] = closeness
            
            # 3. 고유벡터 중심성 (Eigenvector Centrality)
            eigenvector = session.run("""
                CALL gds.eigenvector.stream('social-network', {
                    maxIterations: 20
                })
                YIELD nodeId, score
                WITH gds.util.asNode(nodeId) AS user, score
                ORDER BY score DESC
                LIMIT 10
                RETURN user.name AS name, score AS eigenvector
            """).data()
            metrics['eigenvector'] = eigenvector
            
            return metrics
    
    def fraud_detection_pattern(self):
        """사기 탐지 패턴"""
        with self.driver.session() as session:
            # 의심스러운 패턴 탐지
            
            # 1. 순환 거래 탐지
            circular_transactions = session.run("""
                MATCH path = (u:User)-[:TRANSFERRED_TO*3..5]->(u)
                WHERE ALL(r IN relationships(path) WHERE r.amount > 1000)
                WITH path, 
                     [node IN nodes(path) | node.name] AS participants,
                     REDUCE(total = 0, r IN relationships(path) | total + r.amount) AS total_amount
                RETURN participants[0..-1] AS circle,
                       total_amount,
                       length(path) - 1 AS circle_size
                ORDER BY total_amount DESC
            """).data()
            
            # 2. 이상 거래 패턴
            anomalous_patterns = session.run("""
                MATCH (u:User)
                WITH u,
                     [(u)-[t:TRANSFERRED_TO]->() | t.amount] AS sent_amounts,
                     [(u)<-[t:TRANSFERRED_TO]-() | t.amount] AS received_amounts
                WITH u,
                     REDUCE(s = 0, x IN sent_amounts | s + x) AS total_sent,
                     REDUCE(s = 0, x IN received_amounts | s + x) AS total_received,
                     size(sent_amounts) AS sent_count,
                     size(received_amounts) AS received_count
                WHERE total_sent > 10000 OR total_received > 10000
                WITH u, total_sent, total_received, sent_count, received_count,
                     CASE 
                        WHEN received_count = 0 THEN 999999
                        ELSE toFloat(sent_count) / received_count 
                     END AS ratio
                WHERE ratio > 10 OR ratio < 0.1
                RETURN u.name AS user,
                       total_sent,
                       total_received,
                       sent_count,
                       received_count,
                       ratio
                ORDER BY ratio DESC
            """).data()
            
            # 3. 빠른 네트워크 성장 (잠재적 피라미드)
            rapid_growth = session.run("""
                MATCH (u:User)
                WHERE u.created_at > datetime() - duration({days: 7})
                WITH u
                MATCH (u)-[:RECRUITED]->(recruited:User)
                WITH u, COUNT(recruited) AS direct_recruits
                MATCH path = (u)-[:RECRUITED*1..3]->(:User)
                WITH u, direct_recruits, COUNT(DISTINCT path) AS network_size
                WHERE network_size > 20
                RETURN u.name AS user,
                       direct_recruits,
                       network_size,
                       u.created_at AS join_date
                ORDER BY network_size DESC
            """).data()
            
            return {
                'circular_transactions': circular_transactions,
                'anomalous_patterns': anomalous_patterns,
                'rapid_growth': rapid_growth
            }
```

## 2. 지식 그래프 구축

### 2.1 온톨로지 기반 지식 그래프

```python
from rdflib import Graph, Namespace, Literal, URIRef
from rdflib.namespace import RDF, RDFS, OWL, FOAF, XSD
import spacy
from transformers import pipeline

class KnowledgeGraphBuilder:
    """지식 그래프 구축기"""
    
    def __init__(self):
        self.graph = Graph()
        self.nlp = spacy.load("en_core_web_lg")
        self.relation_extractor = pipeline("text2text-generation", 
                                         model="Babelscape/rebel-large")
        
        # 네임스페이스 정의
        self.EX = Namespace("http://example.org/")
        self.graph.bind("ex", self.EX)
        self.graph.bind("foaf", FOAF)
        self.graph.bind("owl", OWL)
        
    def define_ontology(self):
        """온톨로지 정의"""
        # 클래스 정의
        self.graph.add((self.EX.Person, RDF.type, OWL.Class))
        self.graph.add((self.EX.Organization, RDF.type, OWL.Class))
        self.graph.add((self.EX.Location, RDF.type, OWL.Class))
        self.graph.add((self.EX.Event, RDF.type, OWL.Class))
        self.graph.add((self.EX.Product, RDF.type, OWL.Class))
        
        # 속성 정의
        self.graph.add((self.EX.worksFor, RDF.type, OWL.ObjectProperty))
        self.graph.add((self.EX.worksFor, RDFS.domain, self.EX.Person))
        self.graph.add((self.EX.worksFor, RDFS.range, self.EX.Organization))
        
        self.graph.add((self.EX.locatedIn, RDF.type, OWL.ObjectProperty))
        self.graph.add((self.EX.locatedIn, RDFS.range, self.EX.Location))
        
        self.graph.add((self.EX.participatedIn, RDF.type, OWL.ObjectProperty))
        self.graph.add((self.EX.participatedIn, RDFS.domain, self.EX.Person))
        self.graph.add((self.EX.participatedIn, RDFS.range, self.EX.Event))
        
        # 데이터 속성
        self.graph.add((self.EX.foundedYear, RDF.type, OWL.DatatypeProperty))
        self.graph.add((self.EX.foundedYear, RDFS.domain, self.EX.Organization))
        self.graph.add((self.EX.foundedYear, RDFS.range, XSD.integer))
        
        # 추론 규칙
        self.graph.add((self.EX.colleague, RDF.type, OWL.SymmetricProperty))
        self.graph.add((self.EX.colleague, OWL.propertyChainAxiom, 
                       [self.EX.worksFor, OWL.inverseOf(self.EX.worksFor)]))
    
    def extract_entities_relations(self, text):
        """텍스트에서 엔티티와 관계 추출"""
        # NER을 사용한 엔티티 추출
        doc = self.nlp(text)
        entities = []
        
        for ent in doc.ents:
            entity_type = self.map_spacy_to_ontology(ent.label_)
            if entity_type:
                entities.append({
                    'text': ent.text,
                    'type': entity_type,
                    'start': ent.start_char,
                    'end': ent.end_char
                })
        
        # 관계 추출
        relations = self.relation_extractor(text)
        
        # 지식 그래프에 추가
        for entity in entities:
            entity_uri = URIRef(f"{self.EX}{entity['text'].replace(' ', '_')}")
            self.graph.add((entity_uri, RDF.type, entity['type']))
            self.graph.add((entity_uri, RDFS.label, Literal(entity['text'])))
        
        return entities, relations
    
    def map_spacy_to_ontology(self, spacy_label):
        """SpaCy 레이블을 온톨로지 클래스로 매핑"""
        mapping = {
            'PERSON': self.EX.Person,
            'ORG': self.EX.Organization,
            'GPE': self.EX.Location,
            'LOC': self.EX.Location,
            'EVENT': self.EX.Event,
            'PRODUCT': self.EX.Product
        }
        return mapping.get(spacy_label)
    
    def build_from_structured_data(self, data_source):
        """구조화된 데이터에서 지식 그래프 구축"""
        # CSV, JSON, 데이터베이스 등에서 데이터 로드
        import pandas as pd
        
        if isinstance(data_source, str) and data_source.endswith('.csv'):
            df = pd.read_csv(data_source)
        elif isinstance(data_source, pd.DataFrame):
            df = data_source
        else:
            raise ValueError("Unsupported data source")
        
        # 엔티티 생성
        for _, row in df.iterrows():
            if 'person_name' in df.columns:
                person_uri = URIRef(f"{self.EX}{row['person_name'].replace(' ', '_')}")
                self.graph.add((person_uri, RDF.type, self.EX.Person))
                self.graph.add((person_uri, FOAF.name, Literal(row['person_name'])))
                
                if 'organization' in df.columns:
                    org_uri = URIRef(f"{self.EX}{row['organization'].replace(' ', '_')}")
                    self.graph.add((org_uri, RDF.type, self.EX.Organization))
                    self.graph.add((person_uri, self.EX.worksFor, org_uri))
    
    def query_knowledge_graph(self, sparql_query):
        """SPARQL 쿼리 실행"""
        results = self.graph.query(sparql_query)
        return list(results)
    
    def semantic_search(self, query_text):
        """의미 기반 검색"""
        # 쿼리 임베딩 생성
        query_embedding = self.get_text_embedding(query_text)
        
        # 모든 엔티티의 설명 임베딩과 비교
        sparql = """
        SELECT ?entity ?label ?description
        WHERE {
            ?entity rdfs:label ?label .
            OPTIONAL { ?entity rdfs:comment ?description }
        }
        """
        
        results = self.graph.query(sparql)
        
        # 유사도 계산
        similar_entities = []
        for row in results:
            entity_text = f"{row.label} {row.description or ''}"
            entity_embedding = self.get_text_embedding(entity_text)
            similarity = self.cosine_similarity(query_embedding, entity_embedding)
            
            if similarity > 0.7:  # 임계값
                similar_entities.append({
                    'entity': str(row.entity),
                    'label': str(row.label),
                    'similarity': similarity
                })
        
        return sorted(similar_entities, key=lambda x: x['similarity'], reverse=True)
```

### 2.2 그래프 임베딩과 추론

```python
import torch
import torch.nn as nn
import dgl
from dgl.nn import GraphConv
import numpy as np

class GraphNeuralNetwork:
    """그래프 신경망 기반 임베딩"""
    
    def __init__(self, num_features, hidden_dim=128, num_classes=10):
        self.model = GCN(num_features, hidden_dim, num_classes)
        self.optimizer = torch.optim.Adam(self.model.parameters(), lr=0.01)
        
    def train_node_embeddings(self, graph, features, labels, epochs=200):
        """노드 임베딩 학습"""
        self.model.train()
        
        for epoch in range(epochs):
            logits = self.model(graph, features)
            loss = nn.functional.cross_entropy(logits, labels)
            
            self.optimizer.zero_grad()
            loss.backward()
            self.optimizer.step()
            
            if epoch % 20 == 0:
                print(f'Epoch {epoch}, Loss: {loss.item():.4f}')
        
        # 학습된 임베딩 추출
        self.model.eval()
        with torch.no_grad():
            embeddings = self.model.get_embeddings(graph, features)
        
        return embeddings
    
    def link_prediction(self, graph, node_embeddings):
        """링크 예측"""
        # 모든 노드 쌍에 대한 점수 계산
        num_nodes = graph.number_of_nodes()
        scores = torch.zeros((num_nodes, num_nodes))
        
        for i in range(num_nodes):
            for j in range(i+1, num_nodes):
                # 내적을 사용한 링크 점수
                score = torch.dot(node_embeddings[i], node_embeddings[j])
                scores[i, j] = score
                scores[j, i] = score
        
        # 상위 k개 예측
        k = 10
        flat_scores = scores.flatten()
        top_k_indices = torch.topk(flat_scores, k).indices
        
        predictions = []
        for idx in top_k_indices:
            i = idx // num_nodes
            j = idx % num_nodes
            if i != j and not graph.has_edge(i, j):
                predictions.append((i.item(), j.item(), scores[i, j].item()))
        
        return predictions

class GCN(nn.Module):
    """Graph Convolutional Network"""
    
    def __init__(self, in_features, hidden_features, out_features):
        super(GCN, self).__init__()
        self.conv1 = GraphConv(in_features, hidden_features)
        self.conv2 = GraphConv(hidden_features, out_features)
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, g, features):
        x = self.conv1(g, features)
        x = torch.relu(x)
        x = self.dropout(x)
        x = self.conv2(g, x)
        return x
    
    def get_embeddings(self, g, features):
        x = self.conv1(g, features)
        x = torch.relu(x)
        return x
```

## 3. 실제 응용 사례

### 3.1 추천 시스템

```python
class GraphBasedRecommender:
    """그래프 기반 추천 시스템"""
    
    def __init__(self, neo4j_driver):
        self.driver = neo4j_driver
        
    def collaborative_filtering(self, user_id, item_type='movie', k=10):
        """협업 필터링 기반 추천"""
        with self.driver.session() as session:
            # 사용자 기반 협업 필터링
            result = session.run("""
                MATCH (u:User {id: $user_id})-[r1:RATED]->(item:<item_type>)<-[r2:RATED]-(other:User)
                WHERE r1.rating >= 4 AND r2.rating >= 4
                WITH other, COUNT(DISTINCT item) AS shared_items
                ORDER BY shared_items DESC
                LIMIT 20
                
                MATCH (other)-[r:RATED]->(rec_item:<item_type>)
                WHERE NOT EXISTS((u)-[:RATED]->(rec_item))
                      AND r.rating >= 4
                WITH rec_item, 
                     AVG(r.rating) AS avg_rating,
                     COUNT(r) AS num_ratings,
                     COLLECT(DISTINCT other.id) AS recommenders
                WHERE num_ratings >= 3
                
                RETURN rec_item.id AS item_id,
                       rec_item.title AS title,
                       avg_rating,
                       num_ratings,
                       size(recommenders) AS recommender_count
                ORDER BY avg_rating DESC, num_ratings DESC
                LIMIT $k
            """.replace('<item_type>', item_type), 
            user_id=user_id, k=k)
            
            return [dict(record) for record in result]
    
    def content_based_filtering(self, user_id, item_type='movie', k=10):
        """콘텐츠 기반 필터링"""
        with self.driver.session() as session:
            # 사용자 선호 프로필 구축
            user_profile = session.run("""
                MATCH (u:User {id: $user_id})-[r:RATED]->(item:<item_type>)
                WHERE r.rating >= 4
                MATCH (item)-[:HAS_GENRE]->(g:Genre)
                WITH g.name AS genre, COUNT(*) AS count
                ORDER BY count DESC
                RETURN COLLECT(genre) AS preferred_genres
            """.replace('<item_type>', item_type), 
            user_id=user_id).single()
            
            if not user_profile:
                return []
            
            preferred_genres = user_profile['preferred_genres'][:5]
            
            # 유사 아이템 찾기
            result = session.run("""
                MATCH (item:<item_type>)-[:HAS_GENRE]->(g:Genre)
                WHERE g.name IN $genres
                      AND NOT EXISTS((:User {id: $user_id})-[:RATED]->(item))
                WITH item, COUNT(DISTINCT g) AS matching_genres
                WHERE matching_genres >= 2
                
                OPTIONAL MATCH (item)<-[r:RATED]-()
                WITH item, 
                     matching_genres,
                     AVG(r.rating) AS avg_rating,
                     COUNT(r) AS num_ratings
                
                RETURN item.id AS item_id,
                       item.title AS title,
                       matching_genres,
                       avg_rating,
                       num_ratings
                ORDER BY matching_genres DESC, avg_rating DESC
                LIMIT $k
            """.replace('<item_type>', item_type),
            genres=preferred_genres, user_id=user_id, k=k)
            
            return [dict(record) for record in result]
    
    def hybrid_recommendation(self, user_id, item_type='movie', k=10):
        """하이브리드 추천 (협업 + 콘텐츠 + 그래프 기반)"""
        with self.driver.session() as session:
            # 다양한 추천 신호 결합
            result = session.run("""
                // 협업 필터링 점수
                MATCH (u:User {id: $user_id})-[:RATED]->(:<item_type>)<-[:RATED]-(similar:User)
                MATCH (similar)-[r:RATED]->(item:<item_type>)
                WHERE NOT EXISTS((u)-[:RATED]->(item)) AND r.rating >= 4
                WITH item, AVG(r.rating) AS collab_score, COUNT(DISTINCT similar) AS num_similar_users
                
                // 콘텐츠 기반 점수
                MATCH (u)-[ur:RATED]->(ui:<item_type>)-[:HAS_GENRE]->(g:Genre)<-[:HAS_GENRE]-(item)
                WHERE ur.rating >= 4
                WITH item, collab_score, num_similar_users,
                     COUNT(DISTINCT g) AS content_score
                
                // 그래프 기반 점수 (PageRank)
                MATCH (item)
                WITH item, collab_score, num_similar_users, content_score,
                     coalesce(item.pagerank, 0) AS popularity_score
                
                // 가중치 결합
                WITH item,
                     (0.4 * collab_score + 
                      0.3 * content_score + 
                      0.2 * popularity_score + 
                      0.1 * log(num_similar_users + 1)) AS final_score,
                     collab_score,
                     content_score,
                     popularity_score
                
                RETURN item.id AS item_id,
                       item.title AS title,
                       final_score,
                       collab_score,
                       content_score,
                       popularity_score
                ORDER BY final_score DESC
                LIMIT $k
            """.replace('<item_type>', item_type),
            user_id=user_id, k=k)
            
            return [dict(record) for record in result]
```

### 3.2 지식 그래프 기반 질의응답 시스템

```python
class KnowledgeGraphQA:
    """지식 그래프 기반 질의응답"""
    
    def __init__(self, knowledge_graph, language_model):
        self.kg = knowledge_graph
        self.lm = language_model
        
    def answer_question(self, question):
        """자연어 질문에 대한 답변"""
        # 1. 질문 분석
        entities = self.extract_entities(question)
        intent = self.classify_intent(question)
        
        # 2. SPARQL 쿼리 생성
        sparql_query = self.generate_sparql(entities, intent)
        
        # 3. 지식 그래프 쿼리
        results = self.kg.query(sparql_query)
        
        # 4. 답변 생성
        answer = self.generate_answer(question, results)
        
        return {
            'question': question,
            'entities': entities,
            'intent': intent,
            'sparql': sparql_query,
            'results': results,
            'answer': answer
        }
    
    def extract_entities(self, question):
        """질문에서 엔티티 추출"""
        doc = self.lm.nlp(question)
        entities = []
        
        for ent in doc.ents:
            # 지식 그래프에서 엔티티 확인
            entity_uri = self.find_entity_in_kg(ent.text)
            if entity_uri:
                entities.append({
                    'text': ent.text,
                    'type': ent.label_,
                    'uri': entity_uri
                })
        
        return entities
    
    def generate_sparql(self, entities, intent):
        """SPARQL 쿼리 생성"""
        if intent == 'definition':
            # "What is X?" 형태의 질문
            return f"""
            SELECT ?description ?type
            WHERE {{
                <{entities[0]['uri']}> rdfs:comment ?description ;
                                       rdf:type ?type .
            }}
            """
        
        elif intent == 'relation':
            # "What is the relationship between X and Y?" 형태
            return f"""
            SELECT ?relation
            WHERE {{
                <{entities[0]['uri']}> ?relation <{entities[1]['uri']}> .
            }}
            """
        
        elif intent == 'property':
            # "What is the Y of X?" 형태
            property_name = self.extract_property(question)
            return f"""
            SELECT ?value
            WHERE {{
                <{entities[0]['uri']}> <{property_name}> ?value .
            }}
            """
        
        else:
            # 기본 쿼리
            return f"""
            SELECT ?p ?o
            WHERE {{
                <{entities[0]['uri']}> ?p ?o .
            }}
            LIMIT 10
            """
    
    def multi_hop_reasoning(self, question):
        """다중 홉 추론"""
        # 예: "Who are the colleagues of John's manager?"
        
        # 1단계: John의 매니저 찾기
        step1 = """
        SELECT ?manager
        WHERE {
            ?john foaf:name "John" .
            ?john ex:hasManager ?manager .
        }
        """
        
        # 2단계: 매니저의 동료 찾기
        step2 = """
        SELECT ?colleague
        WHERE {
            ?manager ex:worksIn ?dept .
            ?colleague ex:worksIn ?dept .
            FILTER(?colleague != ?manager)
        }
        """
        
        # 추론 체인 실행
        manager = self.kg.query(step1)
        if manager:
            colleagues = self.kg.query(step2)
            return colleagues
        
        return None
```

## 4. 그래프 데이터베이스 최적화

### 4.1 인덱싱과 쿼리 최적화

```python
class GraphOptimizer:
    """그래프 데이터베이스 최적화"""
    
    def __init__(self, driver):
        self.driver = driver
        
    def analyze_query_performance(self, cypher_query):
        """쿼리 성능 분석"""
        with self.driver.session() as session:
            # EXPLAIN을 사용한 쿼리 계획 분석
            explain_result = session.run(f"EXPLAIN {cypher_query}")
            
            # PROFILE을 사용한 실제 실행 분석
            profile_result = session.run(f"PROFILE {cypher_query}")
            
            # 성능 메트릭 수집
            stats = profile_result.consume().profile
            
            return {
                'total_db_hits': stats.db_hits,
                'total_rows': stats.rows,
                'compilation_time': stats.time,
                'operator_details': self.extract_operator_details(stats)
            }
    
    def optimize_graph_schema(self):
        """그래프 스키마 최적화"""
        with self.driver.session() as session:
            # 1. 카디널리티 분석
            cardinality_analysis = session.run("""
                MATCH (n)
                WITH labels(n) AS node_labels, COUNT(*) AS count
                UNWIND node_labels AS label
                RETURN label, SUM(count) AS node_count
                ORDER BY node_count DESC
            """).data()
            
            # 2. 관계 분포 분석
            relationship_analysis = session.run("""
                MATCH ()-[r]->()
                RETURN type(r) AS relationship_type, 
                       COUNT(*) AS count
                ORDER BY count DESC
            """).data()
            
            # 3. 속성 사용 분석
            property_analysis = session.run("""
                MATCH (n)
                UNWIND keys(n) AS key
                WITH key, COUNT(*) AS usage_count
                RETURN key AS property, usage_count
                ORDER BY usage_count DESC
            """).data()
            
            # 최적화 권장사항 생성
            recommendations = []
            
            # 고카디널리티 속성에 인덱스 추천
            for prop in property_analysis:
                if prop['usage_count'] > 10000:
                    recommendations.append({
                        'type': 'INDEX',
                        'property': prop['property'],
                        'reason': f"High cardinality property with {prop['usage_count']} uses"
                    })
            
            return {
                'cardinality': cardinality_analysis,
                'relationships': relationship_analysis,
                'properties': property_analysis,
                'recommendations': recommendations
            }
    
    def batch_import_optimization(self, data, batch_size=10000):
        """대량 데이터 임포트 최적화"""
        with self.driver.session() as session:
            # 제약조건 임시 비활성화
            session.run("CALL db.constraints() YIELD name RETURN name")
            
            # UNWIND를 사용한 배치 처리
            for i in range(0, len(data), batch_size):
                batch = data[i:i+batch_size]
                
                session.run("""
                    UNWIND $batch AS row
                    MERGE (n:Entity {id: row.id})
                    SET n += row.properties
                """, batch=batch)
                
                # 주기적 커밋
                if i % (batch_size * 10) == 0:
                    session.run("CALL db.checkpoint()")
```

## 마무리

그래프 데이터베이스와 지식 그래프는 복잡한 관계를 모델링하고 분석하는 강력한 도구입니다. Neo4j와 같은 그래프 데이터베이스를 활용하여 소셜 네트워크 분석, 추천 시스템, 사기 탐지 등 다양한 응용을 구현할 수 있으며, 지식 그래프를 통해 의미 기반 검색과 추론이 가능합니다. 그래프 알고리즘과 머신러닝을 결합하면 더욱 강력한 인사이트를 얻을 수 있습니다.

다음 장에서는 시계열 데이터베이스와 IoT 데이터 처리에 대해 학습하겠습니다.