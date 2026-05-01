# 6장. 그래프 알고리즘으로 인사이트 발굴하기 — Neo4j GDS 활용

당신이 HR 데이터 분석을 맡게 되었다고 상상해보자. 경영진이 이런 질문을 던진다. "우리 조직에서 실제로 긴밀하게 협업하는 그룹이 어디인가?", "이 사람이 퇴사하면 정보 흐름에 얼마나 큰 구멍이 뚫리는가?", "스킬 셋이 비슷한 직원을 찾아서 프로젝트에 배정하고 싶은데 어떻게 해야 하는가?" 전통적인 관계형 데이터베이스로는 이런 질문에 답하기가 난감하다. SQL로 조직도의 깊이를 따라 재귀 쿼리를 돌릴 수는 있지만, "비공식 협업 구조"나 "정보 흐름의 병목"을 찾아내는 건 전혀 다른 차원의 문제다.

5장에서 완성한 지식그래프를 떠올려보자. 직원, 부서, 정책, 복리후생이 관계로 촘촘하게 엮여 있다. 이 구조 위에 그래프 알고리즘을 돌리면 눈에 보이지 않던 패턴이 드러난다. 공식 조직도에는 없지만 실제로 긴밀한 커뮤니티, 조직의 정보 허브 역할을 하는 핵심 인물, 두 부서 사이의 최단 정보 전달 경로 — 이런 인사이트가 알고리즘 몇 줄로 나온다.

Neo4j GDS(Graph Data Science) 라이브러리가 바로 이 일을 해준다. 함께 살펴보자.

## GDS 라이브러리 — 그래프 분석의 도구 상자

GDS는 Neo4j 위에서 돌아가는 그래프 분석 라이브러리다. 60개 이상의 그래프 알고리즘을 제공하며, 대부분 세 단계로 실행된다. 그래프 프로젝션(projection), 알고리즘 실행, 결과 활용이다.

왜 "그래프 프로젝션"이라는 별도 단계가 필요할까? GDS는 알고리즘을 효율적으로 실행하기 위해 Neo4j의 원본 데이터를 메모리에 최적화된 형태로 복사한다. 이 복사본이 프로젝션이다. 원본 그래프에서 분석에 필요한 노드와 관계만 골라서 가져오므로, 불필요한 데이터가 분석을 방해하지 않는다.

### GDS 설치와 설정

Neo4j Desktop이라면 Plugins 탭에서 GDS를 설치할 수 있다. AuraDB를 쓴다면 AuraDS(Data Science) 인스턴스를 선택하면 GDS가 기본 탑재되어 있다.

```cypher
// GDS 설치 확인
RETURN gds.version() AS version
```

### 그래프 프로젝션 생성

HR 지식그래프에서 분석에 쓸 프로젝션을 만들어보자.

```cypher
// HR 조직 분석용 그래프 프로젝션
CALL gds.graph.project(
  'hr-org-graph',
  ['Employee', 'Department', 'Policy'],
  {
    BELONGS_TO: { orientation: 'UNDIRECTED' },
    REPORTS_TO: { orientation: 'NATURAL' },
    COLLABORATES_WITH: { orientation: 'UNDIRECTED' },
    APPLIES_TO: { orientation: 'UNDIRECTED' }
  }
)
YIELD graphName, nodeCount, relationshipCount
RETURN graphName, nodeCount, relationshipCount
```

여기서 `orientation` 설정이 중요하다. `REPORTS_TO`는 방향이 의미를 가지므로 `NATURAL`(원래 방향 유지)로 두지만, `COLLABORATES_WITH`는 양방향이므로 `UNDIRECTED`로 설정한다. 이 설정을 잘못하면 알고리즘 결과가 완전히 달라질 수 있으니 주의하자.

프로젝션이 만들어졌는지 확인한다.

```cypher
// 프로젝션 목록 확인
CALL gds.graph.list()
YIELD graphName, nodeCount, relationshipCount, creationTime
RETURN *
```

이제 알고리즘을 하나씩 돌려보자.

## 커뮤니티 탐지 — 보이지 않는 협업 구조 발견

조직도에는 인사팀, 개발팀, 재무팀이라는 공식 구분만 있다. 하지만 실제 업무에서는 부서를 넘나드는 비공식 그룹이 존재한다. 인사팀의 김 주임과 개발팀의 박 선임이 매주 채용 관련 미팅을 하고, 재무팀의 최 과장이 개발팀 예산 관련으로 수시로 소통한다. 이런 비공식 연결이 조직의 실제 동작 방식이다.

Louvain 알고리즘은 그래프에서 이런 커뮤니티를 자동으로 탐지한다. 내부 연결이 밀집된 노드 그룹을 찾아내는 것이다.

### Louvain 알고리즘 실행

먼저 예상 결과를 확인하는 `stream` 모드로 실행해보자.

```cypher
// Louvain 커뮤니티 탐지 (stream 모드 — 결과 미리보기)
CALL gds.louvain.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO']
})
YIELD nodeId, communityId
WITH gds.util.asNode(nodeId) AS employee, communityId
RETURN communityId,
       collect(employee.name) AS members,
       count(*) AS memberCount
ORDER BY memberCount DESC
```

결과가 흥미롭다. 공식 부서와 다른 그룹이 보이는가? 개발팀과 기획팀 직원이 하나의 커뮤니티로 묶여 있다거나, 인사팀 내에서도 두 개의 하위 커뮤니티가 나뉘어 있을 수 있다. 이것이 "공식 조직도에는 없는 실제 협업 구조"다.

결과에 만족했다면 `mutate` 모드로 프로젝션에 결과를 기록하자.

```cypher
// Louvain 결과를 프로젝션에 기록
CALL gds.louvain.mutate('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO'],
  mutateProperty: 'communityId'
})
YIELD communityCount, modularity, modularityScore
RETURN communityCount, modularity, modularityScore
```

`modularity` 값이 0.3 이상이면 커뮤니티 구조가 의미 있다고 판단하는 것이 일반적이다. 0에 가까우면 무작위 연결과 큰 차이가 없다는 뜻이니, 데이터의 관계가 충분히 풍부한지 점검해볼 필요가 있다.

물론 Louvain만이 유일한 선택은 아니다. Leiden 알고리즘은 Louvain의 개선 버전으로, 잘못 분류된 노드를 더 적극적으로 재배치한다. 커뮤니티 품질이 더 높아지는 경향이 있으니, 두 알고리즘의 결과를 비교해보는 것도 좋다.

```cypher
// Leiden 알고리즘으로 비교
CALL gds.leiden.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO']
})
YIELD nodeId, communityId
WITH gds.util.asNode(nodeId) AS employee, communityId
RETURN communityId, collect(employee.name) AS members, count(*) AS size
ORDER BY size DESC
```

## 중심성 분석 — 조직의 핵심 인물은 누구인가

"이 사람이 퇴사하면 어떻게 되는가?" — 이 질문에 데이터로 답하려면 중심성(centrality) 분석이 필요하다. 중심성은 그래프에서 특정 노드가 얼마나 중요한 위치에 있는지를 수치로 나타낸다.

### PageRank — 영향력 측정

PageRank는 구글이 웹 페이지의 중요도를 매기기 위해 만든 알고리즘이다. "중요한 페이지로부터 링크를 많이 받는 페이지가 중요하다"는 원리인데, 이를 조직 네트워크에 적용하면 "중요한 사람들과 많이 연결된 사람이 영향력이 크다"가 된다.

```cypher
// PageRank로 직원 영향력 측정
CALL gds.pageRank.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO'],
  maxIterations: 20,
  dampingFactor: 0.85
})
YIELD nodeId, score
WITH gds.util.asNode(nodeId) AS employee, score
RETURN employee.name AS name,
       employee.employeeId AS id,
       round(score, 4) AS pageRank
ORDER BY pageRank DESC
LIMIT 10
```

상위 10명이 조직에서 가장 영향력 있는 인물이다. 그런데 여기서 한 가지 의문이 생긴다. PageRank 점수가 높다고 반드시 핵심 인물인 걸까? 단순히 연결이 많은 것과, 정보 흐름에서 중요한 위치에 있는 것은 다른 문제다.

### Betweenness Centrality — 정보 흐름의 병목

Betweenness Centrality는 다른 관점으로 중요도를 측정한다. 그래프에서 두 노드 사이의 최단 경로 위에 해당 노드가 얼마나 자주 등장하는지를 계산한다. 다시 말해, "이 사람을 거치지 않으면 정보가 전달되지 않는 경우가 얼마나 많은가"를 수치화한 것이다.

```cypher
// Betweenness Centrality로 정보 흐름 병목 식별
CALL gds.betweenness.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO']
})
YIELD nodeId, score
WITH gds.util.asNode(nodeId) AS employee, score
RETURN employee.name AS name,
       employee.employeeId AS id,
       round(score, 2) AS betweenness
ORDER BY betweenness DESC
LIMIT 10
```

PageRank와 Betweenness의 결과를 나란히 놓고 보면 재미있는 패턴이 보인다. PageRank는 높은데 Betweenness가 낮은 사람은 "존경받지만 대체 가능한" 인물이다. 반대로 PageRank는 보통인데 Betweenness가 높은 사람은 "눈에 잘 띄지 않지만 이 사람 없으면 소통이 끊기는" 인물이다. 후자가 조직 리스크 관점에서 더 위험한 경우가 많다.

두 지표를 함께 조회해보자.

```cypher
// PageRank와 Betweenness를 함께 비교
CALL gds.pageRank.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO']
})
YIELD nodeId, score AS pageRank
WITH nodeId, pageRank
CALL gds.betweenness.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO']
})
YIELD nodeId AS bNodeId, score AS betweenness
WHERE nodeId = bNodeId
WITH gds.util.asNode(nodeId) AS emp, pageRank, betweenness
RETURN emp.name AS name,
       round(pageRank, 4) AS influence,
       round(betweenness, 2) AS bridging
ORDER BY betweenness DESC
LIMIT 10
```

이 결과를 보고 "아, 박지민 선임이 개발팀과 기획팀 사이에서 다리 역할을 하고 있었구나" 같은 인사이트가 나올 수 있다. 이런 분석은 순수하게 데이터가 말해주는 것이지, 누군가의 주관적 판단이 아니다.

## 경로 탐색 — 두 부서 사이의 정보 전달 경로

"개발팀에서 결정한 기술 스택 변경이 재무팀까지 어떤 경로로 전달되는가?" 이런 질문에 최단 경로 알고리즘이 답한다.

```cypher
// 두 부서 사이의 최단 경로
MATCH (start:Employee {name: '김철수'}), (end:Employee {name: '최재훈'})
CALL gds.shortestPath.dijkstra.stream('hr-org-graph', {
  sourceNode: start,
  targetNode: end,
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO']
})
YIELD path
RETURN [n IN nodes(path) | n.name] AS route,
       length(path) AS hops
```

경로가 너무 길다면? 다섯 홉 이상 거쳐야 정보가 전달된다면, 그 두 부서 사이에 직접적인 소통 채널을 만드는 것을 고려해볼 수 있다. 그래프 분석이 조직 설계에 대한 시사점까지 제공하는 셈이다.

모든 직원 쌍의 최단 경로 길이 분포를 보면 조직 전체의 소통 효율성을 가늠할 수 있다.

```cypher
// 전체 직원 쌍의 평균 최단 경로 (조직 소통 밀도 지표)
CALL gds.allShortestPaths.stream('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH']
})
YIELD sourceNode, targetNode, distance
WHERE sourceNode < targetNode  // 중복 제거
WITH distance, count(*) AS pathCount
RETURN distance AS hops, pathCount
ORDER BY hops
```

평균 경로 길이가 3 이하라면 소통이 꽤 밀접한 조직이다. 5 이상이면 부서 간 사일로(silo)가 심각할 수 있다. 이 수치 하나로 "우리 조직의 소통 건강도"에 대한 실마리를 잡을 수 있다는 게 흥미롭지 않은가?

## 유사도 분석 — 스킬 기반 직원 매칭

프로젝트 배정이나 부서 이동을 결정할 때, "이 직원과 스킬 셋이 비슷한 사람이 누구인가?"라는 질문이 자주 나온다. Node Similarity 알고리즘이 이 질문에 답한다.

먼저 직원과 스킬의 관계가 지식그래프에 있어야 한다. 5장에서 넣은 데이터에 스킬 정보를 추가해보자.

```cypher
// 스킬 노드와 직원-스킬 관계 추가 (예시)
MERGE (s1:Skill {name: 'Python'})
MERGE (s2:Skill {name: 'Cypher'})
MERGE (s3:Skill {name: 'Machine Learning'})
MERGE (s4:Skill {name: 'Data Analysis'})
MERGE (s5:Skill {name: 'Project Management'})

WITH 1 AS dummy
MATCH (e:Employee {name: '박지민'})
MATCH (s:Skill) WHERE s.name IN ['Python', 'Cypher', 'Machine Learning']
MERGE (e)-[:HAS_SKILL]->(s)
```

스킬 데이터가 준비되었으면 유사도를 계산한다.

```cypher
// 스킬 기반 직원 유사도 분석을 위한 프로젝션
CALL gds.graph.project(
  'skill-similarity-graph',
  ['Employee', 'Skill'],
  { HAS_SKILL: { orientation: 'UNDIRECTED' } }
)
```

```cypher
// Node Similarity 실행
CALL gds.nodeSimilarity.stream('skill-similarity-graph', {
  nodeLabels: ['Employee'],
  topK: 5
})
YIELD node1, node2, similarity
WITH gds.util.asNode(node1) AS emp1,
     gds.util.asNode(node2) AS emp2,
     similarity
WHERE similarity > 0.5
RETURN emp1.name AS employee1,
       emp2.name AS employee2,
       round(similarity, 3) AS skillSimilarity
ORDER BY skillSimilarity DESC
```

유사도가 0.8 이상이면 스킬 셋이 매우 비슷한 직원 쌍이다. 프로젝트에 특정 스킬이 필요할 때 이 결과를 참고하면, "박지민 선임이 바쁘면 이 과장에게 맡겨도 되겠다"는 판단이 데이터에 근거해서 나온다.

## GDS 결과를 지식그래프에 기록하기 — Write-Back

지금까지 알고리즘 결과를 화면에서 보기만 했다. 하지만 진짜 가치는 이 결과를 지식그래프에 다시 기록해서, 나중에 다른 쿼리나 애플리케이션에서 활용할 수 있게 만드는 데 있다. 특히 8장에서 만들 HR안내봇이 "우리 팀의 핵심 인물이 누구야?"라는 질문에 답하려면, 분석 결과가 그래프 속에 있어야 한다.

이것을 Write-Back이라 부른다. `write` 모드로 알고리즘을 실행하면 결과가 Neo4j 원본 그래프의 노드 속성으로 저장된다.

```cypher
// PageRank 결과를 노드 속성으로 기록
CALL gds.pageRank.write('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO'],
  writeProperty: 'pageRank'
})
YIELD nodePropertiesWritten, ranIterations
RETURN nodePropertiesWritten, ranIterations
```

```cypher
// Betweenness Centrality 결과도 기록
CALL gds.betweenness.write('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO'],
  writeProperty: 'betweenness'
})
YIELD nodePropertiesWritten
RETURN nodePropertiesWritten
```

```cypher
// 커뮤니티 ID도 기록
CALL gds.louvain.write('hr-org-graph', {
  nodeLabels: ['Employee'],
  relationshipTypes: ['COLLABORATES_WITH', 'REPORTS_TO'],
  writeProperty: 'communityId'
})
YIELD communityCount, nodePropertiesWritten
RETURN communityCount, nodePropertiesWritten
```

이제 각 Employee 노드에 `pageRank`, `betweenness`, `communityId` 속성이 추가되었다. 확인해보자.

```cypher
// Write-Back 결과 확인
MATCH (e:Employee)
RETURN e.name,
       e.pageRank,
       e.betweenness,
       e.communityId
ORDER BY e.pageRank DESC
LIMIT 10
```

이 데이터는 8장의 HR안내봇에서 직접 활용된다. "우리 팀에서 가장 영향력 있는 사람은 누구야?"라고 물으면, 봇이 `pageRank` 속성을 기준으로 답변하고, "개발팀과 실제로 긴밀한 부서가 어디야?"라고 물으면 `communityId`를 기반으로 같은 커뮤니티에 속한 다른 부서 직원들을 찾아준다.

Write-Back 후에는 프로젝션을 정리해주는 게 좋다. 프로젝션은 메모리를 점유하므로, 분석이 끝나면 삭제한다.

```cypher
// 프로젝션 정리
CALL gds.graph.drop('hr-org-graph')
CALL gds.graph.drop('skill-similarity-graph')
```

> **기술 리더 의사결정 박스: 그래프 분석을 도입할 것인가?**
>
> | 기준 | GDS Community Edition | GDS Enterprise Edition |
> |------|----------------------|----------------------|
> | **비용** | 무료 | 유료 라이선스 |
> | **알고리즘** | 주요 알고리즘 포함 | 전체 알고리즘 + 최적화 버전 |
> | **성능** | 중소 규모 적합 | 대규모 그래프 최적화 |
> | **메모리 관리** | 기본 | 고급 메모리 추정·관리 |
> | **프로덕션 지원** | 커뮤니티 지원 | 공식 기술 지원 |
>
> **분석 빈도별 접근:**
> - **일회성 분석:** Jupyter 노트북 + GDS Community로 충분하다
> - **주간/월간 배치:** 스케줄러(cron, Airflow)로 분석 → Write-Back 파이프라인 자동화
> - **준실시간:** 데이터 변경 이벤트 시 자동 재분석 — Enterprise 권장
>
> HR안내봇 수준이라면 Community Edition으로 시작하되, 조직 규모가 수천 명 이상이거나 분석 주기가 짧다면 Enterprise를 검토하는 편이 낫다.

## 마무리

이번 장에서 우리는 지식그래프에 분석의 눈을 더했다. Louvain으로 비공식 커뮤니티를 발견했고, PageRank와 Betweenness로 핵심 인물을 식별했다. 최단 경로로 부서 간 소통 경로를 분석했고, Node Similarity로 스킬 기반 직원 매칭까지 해봤다. 무엇보다, 이 결과를 Write-Back으로 지식그래프에 기록해서 다른 곳에서 쓸 수 있게 만들었다.

기억해두자. 그래프 알고리즘은 도구일 뿐이다. 알고리즘이 내놓은 숫자 자체보다, 그 숫자가 비즈니스 맥락에서 무엇을 의미하는지 해석하는 것이 더 중요하다. PageRank 1위가 반드시 최고의 직원은 아니며, 커뮤니티가 나뉜다고 반드시 문제인 것도 아니다. 데이터는 질문에 답하되, 판단은 사람이 해야 한다.

다음 장에서는 이 지식그래프에 AI를 연결한다. LLM이 그래프를 탐색하며 자연어 질문에 답하는 GraphRAG 아키텍처를 설계해보자. 6장에서 Write-Back한 분석 결과가 7장과 8장에서 어떻게 활용되는지, 흐름을 이어서 확인하게 될 것이다.

---

**HR안내봇 진행도**

| 항목 | 상태 |
|------|------|
| 커뮤니티 탐지 | 완료 — Louvain으로 비공식 협업 그룹 식별 |
| 핵심 인물 식별 | 완료 — PageRank + Betweenness로 상위 인물 3명 식별 |
| 경로 분석 | 완료 — 부서 간 최단 소통 경로 분석 |
| 유사도 분석 | 완료 — 스킬 기반 직원 매칭 |
| Write-Back | 완료 — pageRank, betweenness, communityId 노드 속성으로 기록 |
| 다음 단계 | 7장에서 GraphRAG 아키텍처 설계 → 8장에서 통합 구현 |
