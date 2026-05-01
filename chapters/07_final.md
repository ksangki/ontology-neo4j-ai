# 7장. AI와 지식그래프의 만남 — GraphRAG 아키텍처

누군가 당신에게 이렇게 묻는다고 해보자. "육아휴직 중인 직원이 연차를 사용할 수 있나요? 그리고 그 직원의 매니저가 알아야 할 관련 정책도 함께 알려주세요." 이 질문 하나에 여러 겹의 정보가 얽혀 있다. 육아휴직 규정, 연차 사용 지침, 해당 직원의 매니저 관계, 매니저에게 적용되는 정책 — 최소 서너 홉을 넘나들어야 온전한 답이 나온다.

벡터 RAG로 이 질문에 답하려면 어떻게 될까? 육아휴직 관련 문서 조각과 연차 관련 문서 조각을 유사도 검색으로 가져올 수는 있다. 하지만 "그 직원의 매니저"를 찾고, "매니저에게 적용되는 정책"까지 연결하는 건 벡터 유사도만으로는 난감하다. 텍스트 청크 사이의 관계가 아니라, 엔티티 사이의 구조적 관계를 따라가야 하기 때문이다.

바로 이 지점에서 GraphRAG가 등장한다. 지식그래프의 구조화된 관계를 LLM과 결합해 멀티홉 추론이 가능한 답변을 생성하는 아키텍처다. 원리부터 살펴보고, HR안내봇에 어떤 패턴이 적합한지 함께 결정해보자.

## 벡터 RAG vs GraphRAG — 무엇이 다른가

먼저 벡터 RAG의 동작 방식을 간단히 짚어보자.

벡터 RAG는 문서를 청크(chunk)로 나누고, 각 청크를 벡터(임베딩)로 변환한다. 사용자의 질문도 벡터로 바꿔서, 가장 유사한 청크 몇 개를 찾아 LLM에게 "이 맥락을 참고해서 답변해"라고 넘긴다. 단순하고 효과적이다. 정보가 하나의 문서 안에 집중되어 있을 때, 즉 단일 홉 질문에는 잘 작동한다.

그런데 찜찜한 부분이 있다. 벡터 유사도는 "비슷한 단어가 포함된 텍스트"를 찾는 것이지, "논리적으로 연결된 정보"를 찾는 것이 아니다. "김철수의 매니저"를 알려면 김철수 노드에서 REPORTS_TO 관계를 따라가야 하는데, 벡터 검색은 이런 구조적 탐색을 할 수 없다.

GraphRAG는 이 한계를 지식그래프로 보완한다. 질문에서 엔티티를 추출하고, 지식그래프에서 그 엔티티의 관계를 탐색해 맥락을 수집한 뒤, LLM에게 전달한다. 관계의 사슬을 따라가므로 멀티홉 추론이 자연스럽다.

성능 차이는 벤치마크로 확인할 수 있다. Microsoft의 연구에서 계층적 커뮤니티 기반 GraphRAG는 86%의 정확도를 달성했는데, 같은 데이터셋에서 기존 벡터 RAG는 32%에 머물렀다. FalkorDB의 2025년 벤치마크에서도 GraphRAG가 90% 이상의 정확도를 보인 반면, 벡터 RAG는 56.2%였다. 관계 기반 질문에서 격차가 특히 두드러진다.

물론 벡터 RAG가 항상 열등한 것은 아니다. 단일 문서 내 정보 검색, 의미적 유사도 기반 탐색에서는 벡터 RAG가 여전히 효율적이다. 그렇다면 어떻게 해야 할까? 둘을 결합하는 것이다.

## 4가지 통합 패턴 — 어떤 방식으로 연결할 것인가

LLM과 지식그래프를 연결하는 방식은 크게 네 가지로 나뉜다. 각 패턴의 동작 원리와 장단점을 비교해보자.

### 패턴 1: Text-to-Cypher

가장 직관적인 방식이다. 사용자의 자연어 질문을 LLM이 Cypher 쿼리로 변환하고, 그 쿼리를 Neo4j에서 실행해 결과를 돌려준다.

```
[사용자] "인사팀 소속 직원 중 PageRank가 가장 높은 사람은?"
    ↓ LLM이 Cypher로 변환
[Cypher] MATCH (e:Employee)-[:BELONGS_TO]->(d:Department {name: '인사팀'})
         RETURN e.name, e.pageRank ORDER BY e.pageRank DESC LIMIT 1
    ↓ Neo4j 실행
[결과] 김철수 (pageRank: 0.4521)
    ↓ LLM이 자연어로 정리
[답변] "인사팀에서 PageRank가 가장 높은 직원은 김철수님입니다."
```

**장점:** 정확한 구조적 쿼리가 가능하다. 수치 필터, 정렬, 집계 같은 연산에 강하다.

**단점:** LLM이 생성한 Cypher가 틀릴 수 있다. 스키마를 모르면 존재하지 않는 라벨이나 관계를 만들어내기도 한다. 복잡한 질문일수록 오류율이 올라가서 찜찜하다.

**적합한 경우:** 질문 패턴이 비교적 정형화되어 있고, 정확한 수치 기반 답변이 필요할 때.

### 패턴 2: Graph-Enhanced RAG

벡터 RAG의 검색 단계를 그래프 탐색으로 보강하는 방식이다.

```
[사용자] "육아휴직 중 연차 사용이 가능한가요?"
    ↓ 엔티티 추출
[엔티티] 육아휴직, 연차
    ↓ 그래프 탐색 (엔티티 주변 관계 수집)
[맥락] 육아휴직 규정 → 적용 대상 → 정규직 직원
       연차 사용 지침 → 예외 조항 → 휴직 중 연차 소멸 방지
       육아휴직 규정 -[:RELATED_TO]-> 연차 사용 지침
    ↓ + 벡터 검색으로 관련 텍스트 보충
    ↓ LLM에게 통합 맥락 전달
[답변] 구조적 관계 + 텍스트 맥락이 결합된 풍부한 답변
```

**장점:** 그래프의 구조적 맥락과 벡터의 의미적 맥락을 함께 활용한다. 멀티홉 추론이 가능하면서도 텍스트 기반 보충 설명까지 붙일 수 있다.

**단점:** 엔티티 추출 정확도에 의존한다. 질문에서 엔티티를 잘못 뽑으면 엉뚱한 맥락이 수집된다.

**적합한 경우:** 관계 기반 추론과 텍스트 검색이 모두 필요한 복합 질문.

### 패턴 3: LLM 기반 그래프 구축

방향이 반대다. LLM이 비정형 텍스트에서 엔티티와 관계를 추출해 지식그래프를 자동으로 구축한다.

```
[입력] HR 정책 문서 (PDF, 워드)
    ↓ LLM이 엔티티/관계 추출
[추출] (육아휴직 규정)-[:적용_대상]->(정규직 직원)
       (육아휴직 규정)-[:기간]->(최대 1년)
       (육아휴직 규정)-[:조건]->(근속 1년 이상)
    ↓ Neo4j에 적재
[결과] 지식그래프 자동 확장
```

4장에서 LLM으로 온톨로지를 생성했던 것과 비슷하지만, 여기서는 인스턴스 수준의 데이터를 자동으로 추출한다는 점이 다르다. Neo4j의 LLM Knowledge Graph Builder가 바로 이 패턴을 구현한 도구다.

**장점:** 비정형 데이터를 구조화하는 노력을 크게 줄인다.

**단점:** 추출 정확도가 완벽하지 않다. 도메인 전문가의 검증이 필수이며, 잘못 추출된 관계가 그래프를 오염시킬 수 있다.

**적합한 경우:** 새로운 문서가 지속적으로 유입되고, 수동 구조화가 병목인 상황.

### 패턴 4: Agentic GraphRAG

가장 진보된 패턴이다. AI 에이전트가 지식그래프를 영속 메모리(persistent memory)로 활용하며, 반복적으로 탐색하고 추론한다.

```
[에이전트] 질문 분석 → "이 질문에 답하려면 3단계 탐색이 필요하다"
    ↓ 1단계: 직원 정보 탐색
    ↓ 2단계: 관련 정책 수집
    ↓ 3단계: 매니저 관계까지 확장
    ↓ 중간 결과를 그래프에 캐시
[답변] 다단계 추론이 반영된 포괄적 답변
```

에이전트는 한 번의 그래프 탐색으로 부족하면 추가 탐색을 스스로 결정한다. 작업 수행 중 새로운 지식을 그래프에 축적하고, 다음 작업에서 참조하기도 한다.

**장점:** 복잡한 다단계 질문에 강하다. 자율적 탐색으로 사람이 예측하지 못한 연결고리를 찾기도 한다.

**단점:** 구현 복잡도가 높고, 에이전트의 행동을 제어하기 어렵다. 비용도 가장 많이 든다.

**적합한 경우:** 매우 복잡한 분석 질문, 장기적 맥락 유지가 필요한 대화형 시스템.

### 4가지 패턴 비교

| 기준 | Text-to-Cypher | Graph-Enhanced RAG | LLM 그래프 구축 | Agentic GraphRAG |
|------|---------------|-------------------|----------------|-----------------|
| **구현 복잡도** | 낮음 | 중간 | 중간 | 높음 |
| **멀티홉 추론** | 가능 (Cypher 정확도 의존) | 자연스러움 | 해당 없음 | 매우 강함 |
| **답변 풍부도** | 구조적 데이터 중심 | 구조 + 텍스트 | 해당 없음 | 가장 풍부 |
| **비용** | LLM 1회 호출 | LLM 1~2회 + 그래프 탐색 | LLM 다수 호출 | LLM 다수 호출 |
| **유지보수** | 스키마 변경 시 프롬프트 수정 | 리트리버 로직 관리 | 추출 품질 관리 | 에이전트 행동 관리 |
| **적합 시나리오** | 정형 질문, 수치 조회 | 복합 질문, HR안내봇 | 문서 자동 구조화 | 복잡한 분석 대화 |

> **기술 리더 의사결정 박스: 어떤 GraphRAG 패턴을 선택할 것인가?**
>
> 선택의 핵심은 **질문의 복잡도**와 **팀의 역량**이다.
>
> - **질문이 대부분 단일 홉이고 정형적이다** → Text-to-Cypher로 시작하자. 구현이 가장 빠르다.
> - **멀티홉 질문이 많고, 텍스트 맥락도 중요하다** → Graph-Enhanced RAG. 대부분의 엔터프라이즈 시나리오에 적합하다.
> - **비정형 문서가 지속적으로 유입된다** → LLM 그래프 구축을 파이프라인에 추가하자.
> - **고도로 복잡한 분석 질문을 다뤄야 한다** → Agentic GraphRAG. 단, 팀에 AI 엔지니어링 역량이 필요하다.
>
> **권장 조합:** Graph-Enhanced RAG를 기본으로 하되, 정형 질문에는 Text-to-Cypher를 보조로, 문서 유입에는 LLM 그래프 구축을 추가하는 하이브리드 접근이 바람직하다.

## 하이브리드 RAG 아키텍처 설계

HR안내봇에는 다양한 유형의 질문이 들어온다. "김철수의 이메일 주소"처럼 단순 조회도 있고, "육아휴직 중 연차 사용 가능 여부"처럼 정책 해석이 필요한 것도 있고, "우리 팀에서 가장 영향력 있는 사람"처럼 GDS 분석 결과를 활용해야 하는 것도 있다. 하나의 패턴으로 모든 질문을 처리하기는 어렵다.

그래서 하이브리드 RAG 아키텍처를 설계한다. 핵심은 **쿼리 라우터(Query Router)**다. 질문 유형을 분류하고, 적절한 리트리버로 분배하는 역할이다.

```
[사용자 질문]
    ↓
[쿼리 라우터] ← LLM이 질문 유형 분류
    ├─ 구조적 질문 → [그래프 리트리버] → Cypher 쿼리로 Neo4j 탐색
    ├─ 텍스트 질문 → [벡터 리트리버] → 임베딩 유사도 검색
    ├─ 분석 질문  → [GDS 메타데이터 리트리버] → Write-Back된 속성 조회
    └─ 복합 질문  → [하이브리드 리트리버] → 위 세 가지 결과 통합
    ↓
[맥락 통합기] — 수집된 맥락을 정리·중복 제거
    ↓
[LLM 답변 생성] — 통합 맥락 기반 자연어 답변
    ↓
[사용자에게 응답]
```

각 구성 요소를 좀 더 살펴보자.

### 쿼리 라우터

질문을 분류하는 첫 관문이다. LLM에게 분류 기준을 프롬프트로 제공한다.

```python
ROUTER_PROMPT = """
다음 질문을 아래 유형 중 하나로 분류하세요.

유형:
- STRUCTURAL: 조직 구조, 직원 정보, 부서 관계 등 그래프 탐색이 필요한 질문
  예: "김철수의 매니저는 누구인가?", "인사팀 소속 직원은?"
- TEXTUAL: 정책 내용, 규정 해석 등 문서 검색이 필요한 질문
  예: "육아휴직 신청 절차는?", "연차 소멸 규정은?"
- ANALYTICAL: GDS 분석 결과 활용이 필요한 질문
  예: "가장 영향력 있는 직원은?", "비공식 협업 그룹은?"
- HYBRID: 위 유형이 복합된 질문
  예: "육아휴직 중인 직원의 매니저가 알아야 할 정책은?"

질문: {question}
유형:
"""
```

### 그래프 리트리버

구조적 질문을 처리한다. 질문에서 엔티티를 추출하고, 그 엔티티를 중심으로 지식그래프를 탐색한다.

```python
from neo4j import GraphDatabase

class GraphRetriever:
    def __init__(self, driver):
        self.driver = driver

    def retrieve(self, entities, depth=2):
        """엔티티 주변 N홉까지 탐색하여 맥락 수집"""
        context = []
        with self.driver.session() as session:
            for entity in entities:
                result = session.run("""
                    MATCH (n)
                    WHERE n.name = $name OR n.employeeId = $name
                    CALL apoc.path.subgraphAll(n, {
                        maxLevel: $depth,
                        relationshipFilter: 'BELONGS_TO|REPORTS_TO|APPLIES_TO|HAS_POSITION'
                    })
                    YIELD nodes, relationships
                    RETURN nodes, relationships
                """, name=entity, depth=depth)

                for record in result:
                    nodes = record["nodes"]
                    rels = record["relationships"]
                    context.append(self._format_subgraph(nodes, rels))

        return "\n".join(context)

    def _format_subgraph(self, nodes, relationships):
        """서브그래프를 LLM이 이해할 수 있는 텍스트로 변환"""
        lines = []
        for rel in relationships:
            start_name = rel.start_node.get("name", "unknown")
            end_name = rel.end_node.get("name", "unknown")
            rel_type = rel.type
            lines.append(f"{start_name} --[{rel_type}]--> {end_name}")
        return "\n".join(lines)
```

### 벡터 리트리버

텍스트 질문을 처리한다. HR 정책 문서를 청크로 나누고 임베딩해서, 유사도 기반으로 검색한다. Neo4j 자체도 벡터 인덱스를 지원하므로, 별도의 벡터 DB 없이 Neo4j 안에서 벡터 검색을 할 수 있다.

```python
class VectorRetriever:
    def __init__(self, driver):
        self.driver = driver

    def retrieve(self, question, top_k=5):
        """벡터 유사도 기반 문서 청크 검색"""
        with self.driver.session() as session:
            result = session.run("""
                // Neo4j 벡터 인덱스 검색
                CALL db.index.vector.queryNodes(
                    'policy-embeddings',
                    $top_k,
                    $question_embedding
                )
                YIELD node, score
                RETURN node.title AS title,
                       node.content AS content,
                       score
                ORDER BY score DESC
            """, top_k=top_k,
                question_embedding=self._embed(question))

            return [{"title": r["title"],
                     "content": r["content"],
                     "score": r["score"]}
                    for r in result]

    def _embed(self, text):
        """텍스트를 벡터로 변환 (임베딩 모델 호출)"""
        # OpenAI, Sentence-Transformers 등 활용
        pass
```

### GDS 메타데이터 리트리버

6장에서 Write-Back한 분석 결과를 활용한다. `pageRank`, `betweenness`, `communityId` 같은 속성을 직접 조회한다.

```python
class GDSMetadataRetriever:
    def __init__(self, driver):
        self.driver = driver

    def retrieve(self, query_type, params=None):
        """GDS 분석 결과 기반 조회"""
        queries = {
            "key_person": """
                MATCH (e:Employee)-[:BELONGS_TO]->(d:Department)
                WHERE d.name = $department
                RETURN e.name AS name,
                       e.pageRank AS influence,
                       e.betweenness AS bridging
                ORDER BY e.pageRank DESC
                LIMIT 3
            """,
            "community_members": """
                MATCH (e:Employee {communityId: $communityId})
                RETURN e.name AS name,
                       e.communityId AS community
                ORDER BY e.name
            """,
            "similar_skills": """
                MATCH (e:Employee {name: $name})-[:HAS_SKILL]->(s:Skill)
                       <-[:HAS_SKILL]-(similar:Employee)
                WHERE similar <> e
                WITH similar, count(s) AS sharedSkills
                RETURN similar.name AS name, sharedSkills
                ORDER BY sharedSkills DESC
                LIMIT 5
            """
        }
        with self.driver.session() as session:
            result = session.run(queries[query_type], **(params or {}))
            return [dict(record) for record in result]
```

## Neo4j GraphRAG Python 패키지

Neo4j에서는 공식 GraphRAG 파이썬 패키지를 제공한다. 위에서 설계한 아키텍처의 많은 부분을 이 패키지가 이미 구현해두었다.

```python
# 설치
# pip install neo4j-graphrag

from neo4j_graphrag.retrievers import (
    VectorRetriever,
    VectorCypherRetriever,
    HybridRetriever
)
from neo4j_graphrag.generation import GraphRAG
from neo4j_graphrag.llm import OpenAILLM
```

패키지의 핵심 컴포넌트를 살펴보자.

- **VectorRetriever:** Neo4j 벡터 인덱스를 활용한 유사도 검색
- **VectorCypherRetriever:** 벡터 검색 결과를 Cypher로 보강 — 벡터로 찾은 노드의 주변 관계를 추가 탐색
- **HybridRetriever:** 벡터 검색 + 그래프 탐색 결합
- **GraphRAG:** 리트리버와 LLM을 연결하는 파이프라인

```python
from neo4j import GraphDatabase
from neo4j_graphrag.retrievers import VectorCypherRetriever
from neo4j_graphrag.generation import GraphRAG
from neo4j_graphrag.llm import OpenAILLM

# Neo4j 연결
driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password")
)

# LLM 설정
llm = OpenAILLM(model_name="gpt-4o", api_key="...")

# 벡터 + Cypher 하이브리드 리트리버
retriever = VectorCypherRetriever(
    driver=driver,
    index_name="policy-embeddings",
    # 벡터 검색으로 찾은 노드에서 추가 그래프 탐색
    retrieval_query="""
        MATCH (node)-[:APPLIES_TO]->(target)
        OPTIONAL MATCH (node)-[:RELATED_TO]->(related)
        RETURN node.title AS title,
               node.content AS content,
               collect(DISTINCT target.name) AS appliesTo,
               collect(DISTINCT related.title) AS relatedPolicies,
               score
    """
)

# GraphRAG 파이프라인 구성
rag = GraphRAG(
    retriever=retriever,
    llm=llm
)

# 질문에 답하기
response = rag.search(
    query_text="육아휴직 중 연차 사용이 가능한가요?"
)
print(response.answer)
```

`VectorCypherRetriever`가 핵심이다. 벡터 검색으로 관련 문서를 찾은 뒤, `retrieval_query`에 정의된 Cypher를 실행해서 그 문서와 연결된 그래프 맥락까지 함께 가져온다. 벡터의 의미적 검색과 그래프의 구조적 탐색이 하나의 리트리버 안에서 결합되는 셈이다.

## HR안내봇에 적합한 패턴 선택

그렇다면 우리 HR안내봇에는 어떤 조합이 맞을까? 1장에서 정의한 질문 시나리오 20개를 유형별로 분류해보면 답이 나온다.

- **구조적 질문 (7개):** "인사팀 직원 목록", "김철수의 매니저", "개발팀 적용 정책" 등
- **텍스트 질문 (6개):** "육아휴직 신청 절차", "연차 소멸 규정", "재택근무 조건" 등
- **분석 질문 (3개):** "핵심 인물", "비공식 협업 그룹", "스킬 유사 직원" 등
- **복합 질문 (4개):** "육아휴직 중인 직원의 매니저가 알아야 할 정책" 등

세 가지 유형이 고르게 섞여 있으므로, **Graph-Enhanced RAG를 기본으로 하되 GDS 메타데이터 리트리버를 추가**하는 조합이 적합하다. 정형 질문에는 Text-to-Cypher를 보조로 쓰면 정확도를 높일 수 있다.

아키텍처를 정리하면 이렇다.

```
[사용자 질문]
    ↓
[쿼리 라우터 (LLM)]
    ├─ STRUCTURAL → Graph Retriever (Cypher)
    ├─ TEXTUAL → Vector Retriever (임베딩 검색)
    ├─ ANALYTICAL → GDS Metadata Retriever
    └─ HYBRID → Graph + Vector + GDS 통합
    ↓
[맥락 통합기]
    ↓
[LLM 답변 생성]
    ↓
[응답]
```

### PoC 연결 테스트

본격적인 구현은 8장에서 하지만, 기본 연결이 되는지 PoC(Proof of Concept)를 확인해보자.

```python
# PoC: Neo4j 연결 + LLM 연동 기본 테스트
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password")
)

# 1. Neo4j 연결 확인
with driver.session() as session:
    result = session.run("MATCH (e:Employee) RETURN count(e) AS total")
    print(f"총 직원 수: {result.single()['total']}")

# 2. 간단한 그래프 탐색 테스트
with driver.session() as session:
    result = session.run("""
        MATCH (e:Employee {name: '김철수'})-[:REPORTS_TO]->(m:Employee)
        MATCH (m)-[:BELONGS_TO]->(d:Department)
        RETURN e.name AS employee, m.name AS manager, d.name AS department
    """)
    for record in result:
        print(f"{record['employee']}의 매니저: {record['manager']} ({record['department']})")

# 3. LLM 연동 테스트 (간단한 답변 생성)
from neo4j_graphrag.llm import OpenAILLM

llm = OpenAILLM(model_name="gpt-4o", api_key="...")
response = llm.invoke("""
다음 정보를 바탕으로 질문에 답하세요.

정보:
- 김철수는 인사팀 소속이다
- 김철수의 매니저는 박영수이다
- 인사팀에는 육아휴직 규정, 연차 사용 지침이 적용된다

질문: 김철수의 매니저가 관리하는 팀에 적용되는 정책은?
""")
print(response.content)

driver.close()
```

이 PoC가 성공적으로 돌아간다면, Neo4j에서 데이터를 가져오고 LLM으로 답변을 생성하는 기본 파이프라인이 작동하는 것이다. 8장에서는 이 기본 연결 위에 쿼리 라우터, 하이브리드 리트리버, 맥락 통합기를 얹어 완전한 HR안내봇을 만든다.

## 마무리

이번 장에서 우리는 LLM과 지식그래프를 연결하는 원리를 살펴보았다. 벡터 RAG의 한계를 인식하고, 4가지 통합 패턴을 비교했으며, HR안내봇에 적합한 Graph-Enhanced RAG + GDS 메타데이터 조합을 선택했다. 하이브리드 RAG 아키텍처를 설계하고, PoC로 기본 연결을 확인했다.

기억해두자. GraphRAG는 만능이 아니다. 벡터 RAG가 더 효율적인 질문도 분명히 있다. 핵심은 "질문의 성격에 따라 적절한 도구를 선택하는 것"이다. 쿼리 라우터가 바로 그 판단을 자동화하는 장치다.

다음 장이 이 책의 클라이맥스다. 지금까지 쌓아 온 모든 것 — 온톨로지, 지식그래프, GDS 분석, GraphRAG 아키텍처 — 을 하나로 조합해 실제로 동작하는 HR안내봇을 완성한다. 설계도만 그려왔던 여정이 드디어 동작하는 프로토타입으로 결실을 맺는 순간이다. 함께 만들어보자.

---

**HR안내봇 진행도**

| 항목 | 상태 |
|------|------|
| GraphRAG 패턴 선택 | 완료 — Graph-Enhanced RAG + GDS 메타데이터 + Text-to-Cypher 보조 |
| 아키텍처 설계 | 완료 — 쿼리 라우터 + 하이브리드 리트리버 + 맥락 통합기 |
| PoC 연결 | 완료 — Neo4j 연결, 그래프 탐색, LLM 답변 생성 확인 |
| Neo4j GraphRAG 패키지 | 확인 — VectorCypherRetriever 활용 방안 결정 |
| 다음 단계 | 8장에서 전체 파이프라인 통합 구현 |
