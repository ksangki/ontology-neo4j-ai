# 8장. HR안내봇 만들기 — 관통 프로젝트 통합 구현

드디어 이 순간이 왔다. 1장에서 "기존 방식으로는 왜 한계가 있는가"를 고민했던 그때부터, 2장에서 온톨로지 개념을 익히고, 3장에서 Neo4j에 첫 데이터를 넣고, 4장에서 온톨로지를 설계하고, 5장에서 지식그래프를 구축하고, 6장에서 그래프 알고리즘으로 인사이트를 발굴하고, 7장에서 GraphRAG 아키텍처를 설계했다. 일곱 개 장에 걸쳐 하나씩 쌓아 올린 벽돌들이 이번 장에서 하나의 건물이 된다.

당신이 HR 부서의 기술 리더라고 상상해보자. 경영진에게 "AI 기반 HR안내봇을 만들겠다"고 발표한 지 한 달이 지났다. 팀원들은 온톨로지도 설계했고, 지식그래프도 구축했고, 그래프 분석까지 돌려봤다. 이제 남은 건 단 하나 — 실제로 동작하는 봇을 보여주는 것이다. 오늘 그것을 만든다.

## 전체 아키텍처 — 지금까지의 모든 것을 하나로

먼저 전체 그림을 잡아보자. 각 장에서 만든 것들이 어떻게 맞물리는지 한눈에 보는 것이 중요하다.

```
┌─────────────────────────────────────────────────────┐
│                    HR안내봇 전체 아키텍처                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [사용자 질문]                                        │
│       ↓                                             │
│  ┌──────────────┐                                   │
│  │ 질문 분류기    │ ← LLM (질문 유형 판별)               │
│  └──────┬───────┘                                   │
│         ↓                                           │
│  ┌──────────────┐                                   │
│  │ 엔티티 추출기  │ ← LLM (질문에서 개체명 추출)          │
│  └──────┬───────┘                                   │
│         ↓                                           │
│  ┌──────────────────────────────────────┐           │
│  │         하이브리드 리트리버              │           │
│  │  ┌────────────┐  ┌────────────┐      │           │
│  │  │그래프 리트리버│  │벡터 리트리버 │      │           │
│  │  │ (5장 KG)   │  │(정책 문서)  │      │           │
│  │  └────────────┘  └────────────┘      │           │
│  │  ┌────────────┐  ┌────────────┐      │           │
│  │  │GDS 메타데이터│  │Text-to-    │      │           │
│  │  │ (6장 분석)  │  │Cypher 보조 │      │           │
│  │  └────────────┘  └────────────┘      │           │
│  └──────────────────┬───────────────────┘           │
│                     ↓                               │
│  ┌──────────────┐                                   │
│  │ 맥락 통합기    │ ← 수집된 정보 정리·중복 제거           │
│  └──────┬───────┘                                   │
│         ↓                                           │
│  ┌──────────────┐                                   │
│  │ 답변 생성기    │ ← LLM (맥락 기반 자연어 답변)         │
│  └──────┬───────┘                                   │
│         ↓                                           │
│  [사용자에게 응답]                                      │
│                                                     │
│  ─── 데이터 레이어 ───                                 │
│  Neo4j: 온톨로지(4장) + 지식그래프(5장) + GDS(6장)       │
│  벡터 인덱스: HR 정책 문서 임베딩                         │
└─────────────────────────────────────────────────────┘
```

복잡해 보이지만 본질은 단순하다. 질문이 들어오면 분류하고, 적절한 도구로 맥락을 수집하고, LLM이 답변을 만든다. 이 흐름을 파이썬 코드로 하나씩 구현해보자.

## 프로젝트 설정

먼저 필요한 패키지를 설치한다.

```bash
pip install neo4j neo4j-graphrag openai python-dotenv
```

프로젝트 구조는 이렇게 잡는다.

```
hr-chatbot/
├── config.py          # 설정 (Neo4j 연결, LLM API 키)
├── classifier.py      # 질문 분류기
├── entity_extractor.py # 엔티티 추출기
├── retrievers/
│   ├── graph_retriever.py
│   ├── vector_retriever.py
│   ├── gds_retriever.py
│   └── hybrid_retriever.py
├── context_builder.py  # 맥락 통합기
├── generator.py        # 답변 생성기
├── chatbot.py          # 메인 파이프라인
└── test_scenarios.py   # 시나리오 테스트
```

### 설정 파일

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

NEO4J_URI = os.getenv("NEO4J_URI", "bolt://localhost:7687")
NEO4J_USER = os.getenv("NEO4J_USER", "neo4j")
NEO4J_PASSWORD = os.getenv("NEO4J_PASSWORD")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
OPENAI_MODEL = os.getenv("OPENAI_MODEL", "gpt-4o")

# 벡터 인덱스 이름 (5장에서 생성)
VECTOR_INDEX_NAME = "policy-embeddings"
# 임베딩 모델
EMBEDDING_MODEL = "text-embedding-3-small"
```

## 질문 분류기 — 어떤 도구로 답할 것인가

7장에서 설계한 쿼리 라우터를 구현한다. 질문이 들어오면 LLM이 유형을 판별한다.

```python
# classifier.py
from openai import OpenAI
from config import OPENAI_API_KEY, OPENAI_MODEL

client = OpenAI(api_key=OPENAI_API_KEY)

CLASSIFICATION_PROMPT = """당신은 HR안내봇의 질문 분류기입니다.
사용자 질문을 다음 유형 중 하나로 분류하세요.

유형:
- STRUCTURAL: 조직 구조, 직원 정보, 부서 관계 등 그래프 데이터 조회
  예: "김철수의 매니저는?", "인사팀 소속 직원은?", "개발팀에 적용되는 정책은?"
- TEXTUAL: HR 정책 내용, 규정 해석, 절차 안내 등 문서 검색
  예: "육아휴직 신청 절차는?", "연차 소멸 규정 알려줘", "재택근무 조건은?"
- ANALYTICAL: 그래프 분석 결과 활용 (영향력, 커뮤니티, 유사도 등)
  예: "가장 영향력 있는 직원은?", "실제 협업 그룹은?", "비슷한 스킬 가진 사람은?"
- HYBRID: 위 유형이 2개 이상 결합된 복합 질문
  예: "육아휴직 중인 직원의 매니저가 알아야 할 정책은?"

반드시 유형 이름만 답하세요.

질문: {question}"""


def classify_question(question: str) -> str:
    """질문 유형을 분류하여 반환한다."""
    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=[
            {"role": "system", "content": "질문 유형만 정확히 답하세요."},
            {"role": "user", "content": CLASSIFICATION_PROMPT.format(question=question)}
        ],
        temperature=0,
        max_tokens=20
    )
    result = response.choices[0].message.content.strip().upper()

    valid_types = {"STRUCTURAL", "TEXTUAL", "ANALYTICAL", "HYBRID"}
    if result not in valid_types:
        return "HYBRID"  # 판별 불확실하면 HYBRID로 (모든 리트리버 동원)
    return result
```

분류가 불확실할 때 `HYBRID`로 폴백하는 것이 핵심이다. 잘못 분류해서 필요한 맥락을 빠뜨리는 것보다, 조금 과하게 수집하는 편이 낫다. 물론 HYBRID는 비용이 더 들지만, 사용자 경험 관점에서는 빈약한 답변보다 풍부한 답변이 낫다.

## 엔티티 추출기 — 질문에서 핵심 개체 뽑아내기

그래프 리트리버가 동작하려면 질문에서 엔티티를 먼저 추출해야 한다. "김철수의 매니저는?"에서 "김철수"를, "인사팀에 적용되는 정책은?"에서 "인사팀"을 뽑아내는 작업이다.

```python
# entity_extractor.py
import json
from openai import OpenAI
from config import OPENAI_API_KEY, OPENAI_MODEL

client = OpenAI(api_key=OPENAI_API_KEY)

EXTRACTION_PROMPT = """다음 질문에서 HR 도메인 관련 개체(엔티티)를 추출하세요.

추출할 엔티티 유형:
- EMPLOYEE: 직원 이름 또는 사번
- DEPARTMENT: 부서 이름
- POLICY: 정책/규정 이름
- POSITION: 직급/직위
- BENEFIT: 복리후생 항목
- SKILL: 기술/스킬

JSON 형식으로 답하세요.
예시: {{"entities": [{{"type": "EMPLOYEE", "value": "김철수"}}, {{"type": "DEPARTMENT", "value": "인사팀"}}]}}

질문: {question}"""


def extract_entities(question: str) -> list:
    """질문에서 엔티티를 추출한다."""
    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=[
            {"role": "system", "content": "JSON으로만 답하세요."},
            {"role": "user", "content": EXTRACTION_PROMPT.format(question=question)}
        ],
        temperature=0,
        max_tokens=200,
        response_format={"type": "json_object"}
    )
    try:
        result = json.loads(response.choices[0].message.content)
        return result.get("entities", [])
    except (json.JSONDecodeError, KeyError):
        return []
```

`response_format={"type": "json_object"}`를 지정하면 LLM이 반드시 유효한 JSON을 반환한다. 파싱 실패를 줄이는 간단하지만 효과적인 방법이다. 그래도 만약의 경우를 대비해 `try-except`를 걸어두었다. 외부 API를 호출하는 코드에서 예외 처리를 빼먹으면 나중에 프로덕션에서 끔찍한 일이 벌어질 수 있다.

## 리트리버 구현 — 네 가지 도구

### 그래프 리트리버

지식그래프에서 엔티티 주변의 관계를 탐색한다. 5장에서 구축한 온톨로지 기반 지식그래프를 직접 조회한다.

```python
# retrievers/graph_retriever.py
from neo4j import GraphDatabase
from config import NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD


class GraphRetriever:
    def __init__(self):
        self.driver = GraphDatabase.driver(
            NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD)
        )

    def retrieve(self, entities: list, depth: int = 2) -> str:
        """엔티티 주변 N홉까지 관계를 탐색하여 텍스트 맥락으로 반환한다."""
        contexts = []

        with self.driver.session() as session:
            for entity in entities:
                entity_type = entity["type"]
                entity_value = entity["value"]

                if entity_type == "EMPLOYEE":
                    context = self._retrieve_employee(session, entity_value, depth)
                elif entity_type == "DEPARTMENT":
                    context = self._retrieve_department(session, entity_value)
                elif entity_type == "POLICY":
                    context = self._retrieve_policy(session, entity_value)
                else:
                    context = self._retrieve_generic(session, entity_value, depth)

                if context:
                    contexts.append(context)

        return "\n\n".join(contexts)

    def _retrieve_employee(self, session, name: str, depth: int) -> str:
        """직원 중심 서브그래프 탐색"""
        result = session.run("""
            MATCH (e:Employee)
            WHERE e.name = $name OR e.employeeId = $name
            OPTIONAL MATCH (e)-[:BELONGS_TO]->(d:Department)
            OPTIONAL MATCH (e)-[:HAS_POSITION]->(p:Position)
            OPTIONAL MATCH (e)-[:REPORTS_TO]->(m:Employee)
            OPTIONAL MATCH (e)-[:ENTITLED_TO]->(b:Benefit)
            OPTIONAL MATCH (e)-[:HAS_SKILL]->(s:Skill)
            OPTIONAL MATCH (d)<-[:APPLIES_TO]-(pol:Policy)
            RETURN e.name AS name,
                   e.employeeId AS id,
                   e.email AS email,
                   e.hireDate AS hireDate,
                   e.pageRank AS pageRank,
                   e.communityId AS communityId,
                   d.name AS department,
                   p.title AS position,
                   m.name AS manager,
                   collect(DISTINCT b.name) AS benefits,
                   collect(DISTINCT s.name) AS skills,
                   collect(DISTINCT pol.title) AS deptPolicies
        """, name=name)

        record = result.single()
        if not record:
            return ""

        lines = [f"직원 정보: {record['name']}"]
        if record["id"]:
            lines.append(f"  사번: {record['id']}")
        if record["department"]:
            lines.append(f"  소속: {record['department']}")
        if record["position"]:
            lines.append(f"  직급: {record['position']}")
        if record["manager"]:
            lines.append(f"  매니저: {record['manager']}")
        if record["email"]:
            lines.append(f"  이메일: {record['email']}")
        if record["hireDate"]:
            lines.append(f"  입사일: {record['hireDate']}")
        if record["benefits"]:
            lines.append(f"  복리후생: {', '.join(record['benefits'])}")
        if record["skills"]:
            lines.append(f"  스킬: {', '.join(record['skills'])}")
        if record["deptPolicies"]:
            lines.append(f"  부서 적용 정책: {', '.join(record['deptPolicies'])}")

        return "\n".join(lines)

    def _retrieve_department(self, session, name: str) -> str:
        """부서 중심 서브그래프 탐색"""
        result = session.run("""
            MATCH (d:Department)
            WHERE d.name = $name OR d.code = $name
            OPTIONAL MATCH (e:Employee)-[:BELONGS_TO]->(d)
            OPTIONAL MATCH (pol:Policy)-[:APPLIES_TO]->(d)
            RETURN d.name AS department,
                   d.code AS code,
                   collect(DISTINCT e.name) AS employees,
                   collect(DISTINCT pol.title) AS policies
        """, name=name)

        record = result.single()
        if not record:
            return ""

        lines = [f"부서 정보: {record['department']} ({record['code']})"]
        if record["employees"]:
            lines.append(f"  소속 직원: {', '.join(record['employees'][:10])}")
            if len(record["employees"]) > 10:
                lines.append(f"  (외 {len(record['employees']) - 10}명)")
        if record["policies"]:
            lines.append(f"  적용 정책: {', '.join(record['policies'])}")

        return "\n".join(lines)

    def _retrieve_policy(self, session, title: str) -> str:
        """정책 중심 서브그래프 탐색"""
        result = session.run("""
            MATCH (p:Policy)
            WHERE p.title CONTAINS $title
            OPTIONAL MATCH (p)-[:APPLIES_TO]->(target)
            OPTIONAL MATCH (p)-[:CATEGORIZED_AS]->(c:BenefitCategory)
            OPTIONAL MATCH (p)-[:RELATED_TO]->(related:Policy)
            RETURN p.title AS title,
                   p.description AS description,
                   p.effectiveDate AS effectiveDate,
                   c.name AS category,
                   collect(DISTINCT labels(target)[0] + ': ' + target.name) AS appliesTo,
                   collect(DISTINCT related.title) AS relatedPolicies
        """, title=title)

        record = result.single()
        if not record:
            return ""

        lines = [f"정책 정보: {record['title']}"]
        if record["description"]:
            lines.append(f"  설명: {record['description']}")
        if record["category"]:
            lines.append(f"  카테고리: {record['category']}")
        if record["effectiveDate"]:
            lines.append(f"  시행일: {record['effectiveDate']}")
        if record["appliesTo"]:
            lines.append(f"  적용 대상: {', '.join(record['appliesTo'])}")
        if record["relatedPolicies"]:
            lines.append(f"  관련 정책: {', '.join(record['relatedPolicies'])}")

        return "\n".join(lines)

    def _retrieve_generic(self, session, value: str, depth: int) -> str:
        """범용 검색 — 이름으로 아무 노드나 찾아서 주변 탐색"""
        result = session.run("""
            MATCH (n)
            WHERE n.name CONTAINS $value OR n.title CONTAINS $value
            WITH n LIMIT 3
            OPTIONAL MATCH (n)-[r]-(connected)
            RETURN labels(n) AS nodeLabels,
                   n.name AS nodeName,
                   type(r) AS relType,
                   labels(connected) AS connLabels,
                   connected.name AS connName
            LIMIT 20
        """, value=value)

        lines = []
        for record in result:
            node = record["nodeName"] or "unknown"
            rel = record["relType"] or "관련"
            conn = record["connName"] or "unknown"
            lines.append(f"  {node} --[{rel}]--> {conn}")

        return "\n".join(lines) if lines else ""

    def close(self):
        self.driver.close()
```

`_retrieve_employee` 메서드를 자세히 살펴보자. 한 번의 Cypher 쿼리로 직원의 부서, 직급, 매니저, 복리후생, 스킬, 그리고 부서에 적용되는 정책까지 모두 가져온다. 7장에서 설계한 "엔티티 주변 N홉 탐색"을 하나의 OPTIONAL MATCH 체인으로 구현한 것이다. OPTIONAL MATCH를 쓴 이유는, 연결이 없는 경우에도 쿼리 자체가 실패하지 않도록 하기 위해서다.

### 벡터 리트리버

정책 문서의 텍스트 내용을 검색한다. Neo4j의 벡터 인덱스를 활용한다.

```python
# retrievers/vector_retriever.py
from neo4j import GraphDatabase
from openai import OpenAI
from config import (
    NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD,
    OPENAI_API_KEY, VECTOR_INDEX_NAME, EMBEDDING_MODEL
)


class VectorRetriever:
    def __init__(self):
        self.driver = GraphDatabase.driver(
            NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD)
        )
        self.openai = OpenAI(api_key=OPENAI_API_KEY)

    def retrieve(self, question: str, top_k: int = 5) -> str:
        """벡터 유사도 기반으로 관련 정책 문서 청크를 검색한다."""
        question_embedding = self._embed(question)

        with self.driver.session() as session:
            result = session.run("""
                CALL db.index.vector.queryNodes(
                    $index_name,
                    $top_k,
                    $embedding
                )
                YIELD node, score
                RETURN node.title AS title,
                       node.content AS content,
                       node.chunkId AS chunkId,
                       score
                ORDER BY score DESC
            """, index_name=VECTOR_INDEX_NAME,
                top_k=top_k,
                embedding=question_embedding)

            chunks = []
            for record in result:
                chunks.append(
                    f"[{record['title']}] (유사도: {record['score']:.3f})\n"
                    f"{record['content']}"
                )

            return "\n\n---\n\n".join(chunks) if chunks else ""

    def _embed(self, text: str) -> list:
        """텍스트를 벡터 임베딩으로 변환한다."""
        response = self.openai.embeddings.create(
            model=EMBEDDING_MODEL,
            input=text
        )
        return response.data[0].embedding

    def close(self):
        self.driver.close()
```

벡터 리트리버가 동작하려면, 사전에 정책 문서를 청크로 나누고 임베딩을 생성해서 Neo4j에 저장해두어야 한다. 이 준비 작업을 살펴보자.

```python
# 정책 문서 임베딩 사전 준비 (한 번만 실행)
def prepare_policy_embeddings(driver, openai_client):
    """정책 문서를 청크로 나누고 벡터 인덱스에 저장한다."""

    # 1. 벡터 인덱스 생성
    with driver.session() as session:
        session.run("""
            CREATE VECTOR INDEX `policy-embeddings` IF NOT EXISTS
            FOR (c:PolicyChunk)
            ON (c.embedding)
            OPTIONS {
                indexConfig: {
                    `vector.dimensions`: 1536,
                    `vector.similarity_function`: 'cosine'
                }
            }
        """)

    # 2. 정책 문서를 청크로 분할하고 임베딩 생성
    with driver.session() as session:
        policies = session.run("""
            MATCH (p:Policy)
            WHERE p.fullText IS NOT NULL
            RETURN p.policyId AS id, p.title AS title, p.fullText AS text
        """)

        for policy in policies:
            chunks = split_into_chunks(policy["text"], chunk_size=500, overlap=50)

            for i, chunk in enumerate(chunks):
                embedding = openai_client.embeddings.create(
                    model="text-embedding-3-small",
                    input=chunk
                ).data[0].embedding

                session.run("""
                    CREATE (c:PolicyChunk {
                        chunkId: $chunk_id,
                        title: $title,
                        content: $content,
                        embedding: $embedding
                    })
                    WITH c
                    MATCH (p:Policy {policyId: $policy_id})
                    CREATE (c)-[:CHUNK_OF]->(p)
                """, chunk_id=f"{policy['id']}_chunk_{i}",
                    title=policy["title"],
                    content=chunk,
                    embedding=embedding,
                    policy_id=policy["id"])


def split_into_chunks(text: str, chunk_size: int = 500, overlap: int = 50) -> list:
    """텍스트를 겹침이 있는 청크로 분할한다."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```

정책 문서 청크(`PolicyChunk`)를 원래 정책 노드(`Policy`)에 `CHUNK_OF` 관계로 연결한 부분이 중요하다. 이렇게 하면 벡터 검색으로 관련 청크를 찾은 뒤, 그 청크가 속한 정책 노드로 올라가서 구조적 맥락(적용 대상, 관련 정책 등)까지 함께 가져올 수 있다. 벡터와 그래프가 자연스럽게 연결되는 지점이다.

### GDS 메타데이터 리트리버

6장에서 Write-Back한 분석 결과를 조회한다.

```python
# retrievers/gds_retriever.py
from neo4j import GraphDatabase
from config import NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD


class GDSMetadataRetriever:
    def __init__(self):
        self.driver = GraphDatabase.driver(
            NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD)
        )

    def retrieve(self, query_intent: str, params: dict = None) -> str:
        """GDS 분석 결과(Write-Back 속성)를 조회한다."""
        params = params or {}

        handlers = {
            "key_person": self._key_person,
            "community": self._community,
            "similar_skills": self._similar_skills,
            "bridge_person": self._bridge_person,
        }

        handler = handlers.get(query_intent)
        if handler:
            return handler(params)
        return ""

    def _key_person(self, params: dict) -> str:
        """특정 부서 또는 전체 조직에서 영향력 높은 인물 조회"""
        with self.driver.session() as session:
            if "department" in params:
                result = session.run("""
                    MATCH (e:Employee)-[:BELONGS_TO]->(d:Department {name: $dept})
                    WHERE e.pageRank IS NOT NULL
                    RETURN e.name AS name,
                           e.pageRank AS influence,
                           e.betweenness AS bridging,
                           d.name AS department
                    ORDER BY e.pageRank DESC
                    LIMIT 5
                """, dept=params["department"])
            else:
                result = session.run("""
                    MATCH (e:Employee)
                    WHERE e.pageRank IS NOT NULL
                    RETURN e.name AS name,
                           e.pageRank AS influence,
                           e.betweenness AS bridging
                    ORDER BY e.pageRank DESC
                    LIMIT 5
                """)

            lines = ["영향력 상위 직원:"]
            for record in result:
                name = record["name"]
                influence = round(record["influence"], 4)
                bridging = round(record["bridging"], 2) if record["bridging"] else "N/A"
                lines.append(f"  {name}: 영향력 {influence}, 매개 중심성 {bridging}")

            return "\n".join(lines) if len(lines) > 1 else ""

    def _community(self, params: dict) -> str:
        """커뮤니티 소속 직원 조회"""
        with self.driver.session() as session:
            if "employee_name" in params:
                # 특정 직원과 같은 커뮤니티 멤버 조회
                result = session.run("""
                    MATCH (target:Employee {name: $name})
                    WITH target.communityId AS cid
                    MATCH (e:Employee {communityId: cid})
                    OPTIONAL MATCH (e)-[:BELONGS_TO]->(d:Department)
                    RETURN e.name AS name, d.name AS department, cid AS communityId
                    ORDER BY e.name
                """, name=params["employee_name"])
            else:
                # 전체 커뮤니티 요약
                result = session.run("""
                    MATCH (e:Employee)
                    WHERE e.communityId IS NOT NULL
                    OPTIONAL MATCH (e)-[:BELONGS_TO]->(d:Department)
                    RETURN e.communityId AS communityId,
                           collect(e.name) AS members,
                           collect(DISTINCT d.name) AS departments,
                           count(e) AS size
                    ORDER BY size DESC
                """)

            lines = ["커뮤니티 분석 결과:"]
            for record in result:
                if "communityId" in record.keys() and "members" in record.keys():
                    lines.append(
                        f"  커뮤니티 {record['communityId']}: "
                        f"{', '.join(record['members'][:5])} "
                        f"(부서: {', '.join(record['departments'])})"
                    )
                else:
                    lines.append(
                        f"  {record['name']} ({record['department']})"
                    )

            return "\n".join(lines) if len(lines) > 1 else ""

    def _similar_skills(self, params: dict) -> str:
        """스킬 유사도 기반 직원 매칭"""
        with self.driver.session() as session:
            result = session.run("""
                MATCH (e:Employee {name: $name})-[:HAS_SKILL]->(s:Skill)
                       <-[:HAS_SKILL]-(similar:Employee)
                WHERE similar <> e
                WITH similar, collect(s.name) AS sharedSkills, count(s) AS cnt
                RETURN similar.name AS name,
                       sharedSkills,
                       cnt AS sharedCount
                ORDER BY cnt DESC
                LIMIT 5
            """, name=params.get("employee_name", ""))

            lines = [f"'{params.get('employee_name', '')}' 님과 스킬이 유사한 직원:"]
            for record in result:
                lines.append(
                    f"  {record['name']}: "
                    f"공통 스킬 {record['sharedCount']}개 "
                    f"({', '.join(record['sharedSkills'])})"
                )

            return "\n".join(lines) if len(lines) > 1 else ""

    def _bridge_person(self, params: dict) -> str:
        """매개 중심성(Betweenness) 높은 인물 — 정보 흐름의 다리"""
        with self.driver.session() as session:
            result = session.run("""
                MATCH (e:Employee)
                WHERE e.betweenness IS NOT NULL
                OPTIONAL MATCH (e)-[:BELONGS_TO]->(d:Department)
                RETURN e.name AS name,
                       e.betweenness AS betweenness,
                       d.name AS department
                ORDER BY e.betweenness DESC
                LIMIT 5
            """)

            lines = ["정보 흐름 핵심 인물 (매개 중심성 상위):"]
            for record in result:
                lines.append(
                    f"  {record['name']} ({record['department']}): "
                    f"매개 중심성 {round(record['betweenness'], 2)}"
                )

            return "\n".join(lines) if len(lines) > 1 else ""

    def close(self):
        self.driver.close()
```

### 하이브리드 리트리버 — 네 도구의 지휘자

질문 유형에 따라 적절한 리트리버를 조합해서 호출한다.

```python
# retrievers/hybrid_retriever.py
from retrievers.graph_retriever import GraphRetriever
from retrievers.vector_retriever import VectorRetriever
from retrievers.gds_retriever import GDSMetadataRetriever


class HybridRetriever:
    def __init__(self):
        self.graph = GraphRetriever()
        self.vector = VectorRetriever()
        self.gds = GDSMetadataRetriever()

    def retrieve(self, question: str, question_type: str,
                 entities: list, gds_intent: str = None,
                 gds_params: dict = None) -> dict:
        """질문 유형에 따라 적절한 리트리버를 조합하여 맥락을 수집한다."""
        context = {
            "graph": "",
            "vector": "",
            "gds": ""
        }

        if question_type == "STRUCTURAL":
            context["graph"] = self.graph.retrieve(entities)

        elif question_type == "TEXTUAL":
            context["vector"] = self.vector.retrieve(question)

        elif question_type == "ANALYTICAL":
            if gds_intent:
                context["gds"] = self.gds.retrieve(gds_intent, gds_params)
            # 분석 질문이라도 그래프 맥락이 있으면 보충
            if entities:
                context["graph"] = self.graph.retrieve(entities)

        elif question_type == "HYBRID":
            # 모든 리트리버 동원
            if entities:
                context["graph"] = self.graph.retrieve(entities)
            context["vector"] = self.vector.retrieve(question)
            if gds_intent:
                context["gds"] = self.gds.retrieve(gds_intent, gds_params)

        return context

    def close(self):
        self.graph.close()
        self.vector.close()
        self.gds.close()
```

HYBRID 유형일 때 모든 리트리버를 동원하는 것이 보이는가? 비용은 좀 더 들지만, 복합 질문에 대해 빈약한 답변을 내놓는 것보다 낫다. 실무에서는 비용과 품질의 균형을 잡기 위해, 자주 들어오는 질문 유형의 비율을 모니터링하고 리트리버 호출 전략을 조정하는 편이 바람직하다.

## 맥락 통합기 — 수집된 정보를 정리하기

여러 리트리버에서 가져온 맥락을 하나의 깔끔한 텍스트로 만든다.

```python
# context_builder.py

def build_context(retrieval_result: dict) -> str:
    """리트리버들의 결과를 하나의 맥락 문자열로 통합한다."""
    sections = []

    if retrieval_result.get("graph"):
        sections.append(
            "=== 조직 구조 정보 (지식그래프) ===\n"
            + retrieval_result["graph"]
        )

    if retrieval_result.get("vector"):
        sections.append(
            "=== 관련 정책 문서 ===\n"
            + retrieval_result["vector"]
        )

    if retrieval_result.get("gds"):
        sections.append(
            "=== 그래프 분석 인사이트 ===\n"
            + retrieval_result["gds"]
        )

    if not sections:
        return "관련 정보를 찾지 못했습니다."

    return "\n\n".join(sections)
```

단순해 보이지만, 섹션을 명확하게 구분해주는 것이 LLM의 답변 품질에 의미 있는 영향을 준다. LLM이 "이 정보는 조직 구조에서 왔고, 저 정보는 정책 문서에서 왔다"는 것을 인지하면, 답변에서 정보의 출처를 더 정확하게 구분한다.

## 답변 생성기 — LLM이 최종 답변을 만든다

수집된 맥락을 바탕으로 자연어 답변을 생성한다.

```python
# generator.py
from openai import OpenAI
from config import OPENAI_API_KEY, OPENAI_MODEL

client = OpenAI(api_key=OPENAI_API_KEY)

GENERATION_PROMPT = """당신은 기업의 HR안내봇입니다.
아래 제공된 정보를 바탕으로 질문에 정확하고 친절하게 답변하세요.

규칙:
1. 제공된 정보에 없는 내용은 추측하지 마세요. "해당 정보를 찾지 못했습니다"라고 안내하세요.
2. 직원 이름, 부서명, 정책명 등 고유명사는 정확하게 사용하세요.
3. 관련 정책이 있으면 정책 이름과 핵심 내용을 함께 안내하세요.
4. 추가로 확인이 필요한 사항이 있으면 안내하세요.
5. 답변은 한국어로, 존댓말을 사용하세요.

제공된 정보:
{context}

질문: {question}

답변:"""


def generate_answer(question: str, context: str) -> str:
    """맥락 기반으로 자연어 답변을 생성한다."""
    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=[
            {
                "role": "system",
                "content": "당신은 정확하고 친절한 HR안내봇입니다."
            },
            {
                "role": "user",
                "content": GENERATION_PROMPT.format(
                    context=context,
                    question=question
                )
            }
        ],
        temperature=0.3,
        max_tokens=1000
    )
    return response.choices[0].message.content
```

`temperature`를 0.3으로 낮게 설정한 것에 주목하자. HR안내봇은 창의적 답변보다 정확한 답변이 중요하다. 정책 해석을 창의적으로 하면 난감한 결과가 나올 수 있다. 단, 너무 0에 가까우면 답변이 딱딱해지므로, 0.2~0.4 사이에서 테스트하며 조정하는 게 좋다.

## 메인 파이프라인 — 모든 것의 조합

이제 모든 컴포넌트를 하나로 엮는다.

```python
# chatbot.py
from classifier import classify_question
from entity_extractor import extract_entities
from retrievers.hybrid_retriever import HybridRetriever
from context_builder import build_context
from generator import generate_answer


class HRChatbot:
    def __init__(self):
        self.retriever = HybridRetriever()

    def answer(self, question: str) -> dict:
        """사용자 질문에 대한 전체 파이프라인을 실행한다."""

        # 1단계: 질문 분류
        question_type = classify_question(question)

        # 2단계: 엔티티 추출
        entities = extract_entities(question)

        # 3단계: GDS 의도 판별 (분석 질문인 경우)
        gds_intent, gds_params = self._detect_gds_intent(
            question, question_type, entities
        )

        # 4단계: 하이브리드 리트리버로 맥락 수집
        retrieval_result = self.retriever.retrieve(
            question=question,
            question_type=question_type,
            entities=entities,
            gds_intent=gds_intent,
            gds_params=gds_params
        )

        # 5단계: 맥락 통합
        context = build_context(retrieval_result)

        # 6단계: 답변 생성
        answer = generate_answer(question, context)

        return {
            "answer": answer,
            "question_type": question_type,
            "entities": entities,
            "context_sources": {
                k: bool(v) for k, v in retrieval_result.items()
            }
        }

    def _detect_gds_intent(self, question: str, q_type: str,
                           entities: list) -> tuple:
        """분석 관련 질문의 GDS 의도를 판별한다."""
        if q_type not in ("ANALYTICAL", "HYBRID"):
            return None, None

        question_lower = question.lower()

        # 키워드 기반 간단 판별 (프로덕션에서는 LLM 분류 권장)
        if any(kw in question_lower for kw in ["영향력", "핵심 인물", "중요한 사람", "key person"]):
            dept = next(
                (e["value"] for e in entities if e["type"] == "DEPARTMENT"),
                None
            )
            return "key_person", {"department": dept} if dept else {}

        if any(kw in question_lower for kw in ["커뮤니티", "협업 그룹", "비공식"]):
            emp = next(
                (e["value"] for e in entities if e["type"] == "EMPLOYEE"),
                None
            )
            return "community", {"employee_name": emp} if emp else {}

        if any(kw in question_lower for kw in ["유사", "비슷한 스킬", "스킬 매칭"]):
            emp = next(
                (e["value"] for e in entities if e["type"] == "EMPLOYEE"),
                None
            )
            if emp:
                return "similar_skills", {"employee_name": emp}

        if any(kw in question_lower for kw in ["다리 역할", "매개", "정보 흐름"]):
            return "bridge_person", {}

        return None, None

    def close(self):
        self.retriever.close()


# 실행
if __name__ == "__main__":
    bot = HRChatbot()

    questions = [
        "김철수의 매니저는 누구인가요?",
        "육아휴직 신청 절차를 알려주세요.",
        "우리 조직에서 가장 영향력 있는 직원은 누구인가요?",
        "육아휴직 중인 직원의 매니저가 알아야 할 관련 정책은 무엇인가요?",
    ]

    for q in questions:
        print(f"\n질문: {q}")
        result = bot.answer(q)
        print(f"유형: {result['question_type']}")
        print(f"엔티티: {result['entities']}")
        print(f"맥락 출처: {result['context_sources']}")
        print(f"답변: {result['answer']}")
        print("-" * 60)

    bot.close()
```

6단계 파이프라인이 완성되었다. 질문이 들어오면 분류하고, 엔티티를 뽑고, GDS 의도를 판별하고, 하이브리드 리트리버로 맥락을 모으고, 통합하고, 답변을 생성한다. 반환값에 `question_type`, `entities`, `context_sources`를 포함시킨 것은 디버깅과 품질 분석을 위해서다. 봇이 엉뚱한 답을 내놓을 때, 어디서 문제가 생겼는지 추적하려면 이 메타데이터가 필수다.

## 엔드투엔드 시나리오 테스트

1장에서 정의한 20개 질문 시나리오로 봇을 검증해보자. 질문 유형별로 대표적인 시나리오를 뽑아 테스트한다.

```python
# test_scenarios.py
from chatbot import HRChatbot

SCENARIOS = [
    # 구조적 질문
    {"question": "김철수는 어느 부서 소속인가요?",
     "expected_type": "STRUCTURAL",
     "expected_keywords": ["인사팀"]},

    {"question": "인사팀 소속 직원 목록을 알려주세요.",
     "expected_type": "STRUCTURAL",
     "expected_keywords": ["김철수"]},

    {"question": "김철수의 매니저는 누구이며, 그 매니저가 관리하는 부서는?",
     "expected_type": "STRUCTURAL",
     "expected_keywords": ["매니저"]},

    {"question": "개발팀에 적용되는 정책은 무엇인가요?",
     "expected_type": "STRUCTURAL",
     "expected_keywords": ["정책"]},

    # 텍스트 질문
    {"question": "육아휴직 신청 절차를 알려주세요.",
     "expected_type": "TEXTUAL",
     "expected_keywords": ["육아휴직", "신청"]},

    {"question": "연차 소멸 방지 규정은 어떻게 되나요?",
     "expected_type": "TEXTUAL",
     "expected_keywords": ["연차"]},

    {"question": "재택근무는 어떤 조건에서 가능한가요?",
     "expected_type": "TEXTUAL",
     "expected_keywords": ["재택근무"]},

    # 분석 질문
    {"question": "우리 조직에서 가장 영향력 있는 직원 3명은?",
     "expected_type": "ANALYTICAL",
     "expected_keywords": ["영향력"]},

    {"question": "인사팀과 실제로 긴밀하게 협업하는 비공식 그룹은?",
     "expected_type": "ANALYTICAL",
     "expected_keywords": ["커뮤니티", "협업"]},

    {"question": "박지민 선임과 스킬이 비슷한 직원을 추천해주세요.",
     "expected_type": "ANALYTICAL",
     "expected_keywords": ["스킬", "유사"]},

    # 복합 질문
    {"question": "육아휴직 중인 직원의 매니저가 알아야 할 관련 정책은?",
     "expected_type": "HYBRID",
     "expected_keywords": ["육아휴직", "매니저", "정책"]},

    {"question": "김철수의 매니저는 누구이며, 그 매니저가 관리하는 정책 중 복리후생 관련은?",
     "expected_type": "HYBRID",
     "expected_keywords": ["매니저", "복리후생"]},

    {"question": "개발팀의 핵심 인물이 퇴사하면 어떤 정책적 영향이 있을까요?",
     "expected_type": "HYBRID",
     "expected_keywords": ["핵심 인물", "개발팀"]},
]


def run_tests():
    """시나리오 테스트를 실행하고 결과를 보고한다."""
    bot = HRChatbot()
    results = []

    for i, scenario in enumerate(SCENARIOS, 1):
        question = scenario["question"]
        expected_type = scenario["expected_type"]
        expected_keywords = scenario["expected_keywords"]

        print(f"\n{'='*60}")
        print(f"시나리오 {i}: {question}")
        print(f"기대 유형: {expected_type}")

        result = bot.answer(question)

        # 유형 분류 정확도
        type_correct = result["question_type"] == expected_type

        # 답변에 핵심 키워드 포함 여부
        answer_lower = result["answer"].lower()
        keyword_hits = sum(
            1 for kw in expected_keywords
            if kw.lower() in answer_lower
        )
        keyword_score = keyword_hits / len(expected_keywords)

        status = "PASS" if type_correct and keyword_score >= 0.5 else "FAIL"

        print(f"실제 유형: {result['question_type']} {'(O)' if type_correct else '(X)'}")
        print(f"키워드 매칭: {keyword_hits}/{len(expected_keywords)}")
        print(f"답변 (앞 200자): {result['answer'][:200]}...")
        print(f"판정: {status}")

        results.append({
            "scenario": i,
            "status": status,
            "type_correct": type_correct,
            "keyword_score": keyword_score
        })

    # 종합 보고
    print(f"\n{'='*60}")
    print("종합 결과")
    print(f"{'='*60}")
    total = len(results)
    passed = sum(1 for r in results if r["status"] == "PASS")
    type_accuracy = sum(1 for r in results if r["type_correct"]) / total
    avg_keyword = sum(r["keyword_score"] for r in results) / total

    print(f"통과: {passed}/{total} ({passed/total*100:.1f}%)")
    print(f"유형 분류 정확도: {type_accuracy*100:.1f}%")
    print(f"키워드 매칭 평균: {avg_keyword*100:.1f}%")

    bot.close()


if __name__ == "__main__":
    run_tests()
```

이 테스트는 완벽한 자동 평가는 아니다. 키워드 매칭은 답변 품질을 대략적으로만 측정한다. 프로덕션에서는 LLM을 평가자로 쓰는 "LLM-as-Judge" 방식이나, 사람이 직접 평가하는 골드 셋 방식을 병행하는 편이 낫다. 하지만 프로토타입 단계에서 빠르게 문제를 잡아내기에는 이 정도로도 충분하다.

### 시나리오별 응답 품질 분석

테스트를 돌려보면 몇 가지 패턴이 보일 것이다.

**잘 동작하는 경우:**
- 단일 엔티티 구조적 질문 ("김철수의 부서는?") — 엔티티 추출이 쉽고, 그래프 탐색 결과가 명확하다
- 단일 정책 텍스트 질문 ("육아휴직 신청 절차는?") — 벡터 검색이 관련 문서를 잘 찾는다

**개선이 필요한 경우:**
- 멀티홉 복합 질문 — 질문에 포함된 엔티티가 많을수록 그래프 탐색 범위가 넓어지고, 맥락이 너무 길어져 LLM이 핵심을 놓칠 수 있다. 맥락에 관련도 점수를 매겨 상위 N개만 전달하는 전략이 필요하다.
- GDS 분석 결과 해석 — "핵심 인물이 퇴사하면 어떤 영향이 있을까?"처럼 데이터를 해석하는 질문은, 분석 결과와 정책 맥락을 함께 제공해야 의미 있는 답변이 나온다.

> **기술 리더 의사결정 박스: 프로토타입에서 어디까지 검증할 것인가?**
>
> | 검증 항목 | MVP 기준선 | 측정 방법 |
> |-----------|-----------|----------|
> | **질문 분류 정확도** | 80% 이상 | 테스트 시나리오 20개 기준 유형 일치율 |
> | **답변 관련성** | 70% 이상 | 키워드 매칭 + 사람 평가 샘플링 |
> | **멀티홉 정확도** | 60% 이상 | 2홉 이상 질문 중 정확한 관계 추론 비율 |
> | **응답 시간** | 5초 이내 | 질문~답변 종단 시간 |
> | **할루시네이션** | 5% 미만 | 존재하지 않는 정보를 생성한 비율 |
>
> **Go/No-Go 판단:**
> - 위 기준 4개 이상 충족 → **Go** — 다음 단계(9장 품질 관리, 10장 프로덕션)로 진행
> - 2~3개 충족 → **조건부 Go** — 미충족 항목 집중 개선 후 재검증
> - 1개 이하 → **No-Go** — 아키텍처 또는 데이터 품질 근본적 재검토
>
> 프로토타입 단계에서 완벽을 추구할 필요는 없다. 핵심은 "이 접근 방식이 작동하는가"를 입증하는 것이다. 수치가 낮더라도 개선 방향이 명확하다면 그것 자체가 프로토타입의 성과다.

## 실제 동작 확인

코드가 완성되었으니, 실제로 몇 가지 질문을 던져보자.

```python
bot = HRChatbot()

# 시나리오 1: 구조적 질문
result = bot.answer("김철수의 매니저는 누구이며, 그 매니저가 관리하는 정책은?")
print(result["answer"])
# 예상 답변:
# "김철수님의 매니저는 박영수님입니다. 박영수님이 관리하는 인사팀에는
#  육아휴직 규정, 연차 사용 지침, 성과평가 정책이 적용되고 있습니다."

# 시나리오 2: 분석 질문
result = bot.answer("우리 조직에서 가장 영향력 있는 직원 3명을 알려주세요.")
print(result["answer"])
# 예상 답변:
# "그래프 분석 결과, 조직 내 영향력(PageRank) 상위 3명은 다음과 같습니다:
#  1. 박영수 (영향력: 0.4521, 인사팀)
#  2. 최재훈 (영향력: 0.3892, 개발팀)
#  3. 이미경 (영향력: 0.3156, 기획팀)"

# 시나리오 3: 복합 질문 (클라이맥스!)
result = bot.answer(
    "육아휴직 중인 직원의 매니저가 알아야 할 관련 정책은 무엇이며, "
    "해당 매니저가 조직 내에서 얼마나 중요한 위치에 있는지도 알려주세요."
)
print(result["answer"])
# 예상 답변:
# "육아휴직 관련하여 매니저가 알아야 할 정책은 다음과 같습니다:
#  1. 육아휴직 규정: 근속 1년 이상 정규직 대상, 최대 1년 ...
#  2. 연차 사용 지침: 휴직 중 연차 소멸 방지 조항 ...
#  3. 업무 인수인계 가이드: ...
#
#  참고로, 해당 매니저의 조직 내 위치를 분석한 결과:
#  - 영향력(PageRank): 0.4521 (상위 5% 이내)
#  - 매개 중심성: 48.3 (부서 간 정보 흐름에서 핵심 역할)
#  이 매니저가 부재할 경우 정보 전달에 영향이 있을 수 있으니,
#  대행자 지정을 사전에 해두시는 것이 좋겠습니다."

bot.close()
```

세 번째 시나리오가 이 책의 클라이맥스다. 하나의 질문에 대해 정책 문서 검색(벡터 리트리버), 조직 구조 탐색(그래프 리트리버), 그래프 분석 결과 활용(GDS 리트리버)이 모두 동작한다. 1장부터 7장까지 쌓아 온 모든 요소가 한 번의 질문에 결합되어 답변을 만들어낸다.

이 답변이 벡터 RAG만으로 가능했을까? 불가능하다. 매니저 관계는 그래프 탐색 없이 알 수 없고, 매니저의 조직 내 중요도는 GDS 분석 없이 알 수 없다. 정책 내용은 벡터 검색 없이 가져올 수 없다. 세 가지가 결합되어야 비로소 완전한 답변이 나온다. 이것이 온톨로지 기반 지식그래프 시스템의 힘이다.

## 마무리

이번 장에서 우리는 드디어 HR안내봇을 완성했다. 질문 분류기, 엔티티 추출기, 네 종류의 리트리버, 맥락 통합기, 답변 생성기 — 여섯 개의 컴포넌트가 파이프라인으로 연결되어, 구조적 질문, 텍스트 질문, 분석 질문, 복합 질문에 모두 대응하는 봇이 만들어졌다.

기억해두자. 이 프로토타입은 시작이지 끝이 아니다. 분류기의 정확도를 높여야 하고, 엔티티 추출의 견고함을 강화해야 하고, 맥락 길이를 최적화해야 하고, 답변의 할루시네이션을 줄여야 한다. 하지만 중요한 것은, "이 접근 방식이 작동한다"는 것을 우리가 직접 확인했다는 사실이다.

다음 장에서는 이 프로토타입의 품질을 체계적으로 관리하는 방법을 다룬다. 온톨로지가 변경되면 어떻게 대응하는지, 답변 정확도를 어떻게 지속적으로 측정하는지, 지식그래프를 어떻게 진화시키는지 — 만든 것을 유지하고 키우는 이야기다.

---

**HR안내봇 진행도**

| 항목 | 상태 |
|------|------|
| 질문 분류기 | 완료 — LLM 기반 4유형 분류 (STRUCTURAL, TEXTUAL, ANALYTICAL, HYBRID) |
| 엔티티 추출기 | 완료 — LLM 기반 6유형 엔티티 추출 (JSON 출력) |
| 그래프 리트리버 | 완료 — 직원/부서/정책별 특화 쿼리 + 범용 탐색 |
| 벡터 리트리버 | 완료 — Neo4j 벡터 인덱스 + OpenAI 임베딩 |
| GDS 메타데이터 리트리버 | 완료 — PageRank, Betweenness, Community, Skill Similarity 조회 |
| 하이브리드 리트리버 | 완료 — 유형별 리트리버 조합 전략 |
| 맥락 통합기 | 완료 — 섹션 구분 + 중복 제거 |
| 답변 생성기 | 완료 — LLM 기반 자연어 답변 (temperature 0.3) |
| 시나리오 테스트 | 완료 — 13개 시나리오 (구조적 4, 텍스트 3, 분석 3, 복합 3) |
| 프로토타입 수준 | 동작 확인 — 멀티홉 + GDS + 벡터 통합 답변 생성 성공 |
| 다음 단계 | 9장 품질 관리, 10장 프로덕션 |
