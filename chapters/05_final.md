# 5장. 온톨로지를 Neo4j에 심기 — 레거시 데이터 마이그레이션과 지식그래프 구축

4장에서 공들여 설계한 OWL 파일이 손에 들려 있다. Protege에서 직접 다듬었든, LLM에게 초안을 맡겼든, 어쨌든 HR 도메인의 클래스 12개와 관계 15개가 깔끔하게 정의된 온톨로지가 완성되었다. 그런데 이 파일을 두고 가만히 생각해보자. 설계도만으로 건물에 사람이 살 수 있을까? 당연히 아니다. 설계도대로 건물을 짓고, 기존 세입자들을 새 건물로 이사시켜야 한다.

3장에서 Neo4j에 넣어두었던 HR 샘플 데이터를 기억하는가? 직원 10명, 부서 3개, 정책 5개 — Cypher를 익히기 위해 급하게 넣은 데이터였다. 라벨도 대충 붙였고, 관계 이름도 제각각이었다. 솔직히 말하면, 그때는 그게 최선이었다. 하지만 이제 온톨로지라는 설계도가 생겼으니, 이 데이터를 온톨로지 기반 스키마로 전환해야 한다. 실무에서 레거시 시스템을 새 아키텍처로 이전하는 작업과 본질적으로 같은 일이다.

이번 장에서 할 일은 크게 세 가지다. 첫째, Neosemantics(n10s) 플러그인으로 OWL 온톨로지를 Neo4j에 임포트한다. 둘째, 3장의 샘플 데이터를 온톨로지 스키마에 맞춰 마이그레이션한다. 셋째, 추가 데이터를 적재해 100건 규모의 지식그래프를 완성한다. 하나씩 살펴보자.

## Neosemantics(n10s) — Neo4j에 시맨틱 레이어 얹기

Neo4j는 Labeled Property Graph(LPG) 모델을 쓴다. 온톨로지는 RDF/OWL 세계에 살고 있다. 이 둘은 태생이 다르다. LPG는 속성이 풍부한 노드와 관계를 다루고, RDF는 주어-술어-목적어 트리플로 모든 것을 표현한다. 그렇다면 이 두 세계를 어떻게 연결할까?

Neosemantics, 줄여서 n10s라 부르는 플러그인이 바로 그 다리 역할을 한다. Neo4j Labs에서 공식으로 관리하는 이 플러그인은 OWL, RDFS, SKOS 온톨로지를 Neo4j 그래프로 임포트하고, 반대로 Neo4j 데이터를 RDF로 내보내는 일까지 해준다.

### n10s 설치

Neo4j Desktop을 쓴다면 설치는 간단하다. 프로젝트의 Plugins 탭에서 n10s를 찾아 Install 버튼을 누르면 된다. 수동 설치가 필요한 환경이라면 JAR 파일을 직접 내려받아 plugins 디렉터리에 넣는다.

```cypher
// n10s가 제대로 설치되었는지 확인
RETURN n10s.version() AS version
```

설치 확인이 되었다면, 가장 먼저 해야 할 일은 그래프 설정 초기화다. n10s는 RDF 데이터를 어떤 방식으로 LPG에 매핑할지 결정하는 설정이 필요하다.

```cypher
// n10s 설정 초기화
CALL n10s.graphconfig.init({
  handleVocabUris: 'MAP',
  handleMultival: 'ARRAY',
  handleRDFTypes: 'LABELS'
})
```

각 옵션이 하는 일을 살펴보자.

- `handleVocabUris: 'MAP'` — 긴 URI를 짧은 이름으로 매핑한다. `http://example.org/hr#Employee`가 `Employee`라는 라벨이 된다. 이게 없으면 노드 라벨이 끔찍하게 긴 URI가 되어 Cypher 쿼리를 쓸 때마다 난감해진다.
- `handleMultival: 'ARRAY'` — 하나의 속성에 여러 값이 있으면 배열로 저장한다.
- `handleRDFTypes: 'LABELS'` — RDF의 `rdf:type`을 Neo4j 노드 라벨로 변환한다. 이것이 핵심이다. 온톨로지의 클래스가 곧 Neo4j의 라벨이 된다.

여기서 한 가지 주의할 점이 있다. `graphconfig.init`은 데이터가 비어 있는 상태에서만 실행할 수 있다. 이미 데이터가 들어 있다면 설정을 변경할 수 없다. 나중에 설정을 바꾸려면 데이터를 먼저 정리해야 한다는 뜻이니, 처음 설정할 때 신중하게 결정하는 편이 낫다.

그 다음으로, URI 접두사 매핑을 설정한다.

```cypher
// 네임스페이스 접두사 매핑
CALL n10s.nsprefixes.add('hr', 'http://example.org/hr#');
CALL n10s.nsprefixes.add('org', 'http://www.w3.org/ns/org#');
CALL n10s.nsprefixes.add('foaf', 'http://xmlns.com/foaf/0.1/');
```

이렇게 하면 `http://example.org/hr#Employee`를 Cypher에서 `hr__Employee` 대신 깔끔한 `Employee`로 쓸 수 있다.

### OWL 온톨로지 임포트

설정이 끝났으니 이제 진짜 임포트를 해보자. 4장에서 내보낸 OWL/Turtle 파일을 Neo4j에 불러온다.

```cypher
// 온톨로지 파일 임포트 (로컬 파일 경로)
CALL n10s.onto.import.fetch(
  'file:///path/to/hr-ontology.ttl',
  'Turtle'
)
```

URL로도 가져올 수 있다. GitHub에 온톨로지 파일을 올려두었다면 이렇게 한다.

```cypher
// 원격 URL에서 온톨로지 임포트
CALL n10s.onto.import.fetch(
  'https://raw.githubusercontent.com/your-repo/hr-ontology.ttl',
  'Turtle'
)
```

임포트가 성공하면 Neo4j에 어떤 일이 벌어질까? 온톨로지의 각 클래스가 `Class` 라벨을 가진 노드로 생성되고, 클래스 간 계층 관계(`rdfs:subClassOf`)가 `SCO` 관계로 매핑된다. 오브젝트 프로퍼티와 데이터 프로퍼티는 각각 `ObjectProperty`, `DatatypeProperty` 라벨의 노드가 된다.

임포트 결과를 확인해보자.

```cypher
// 임포트된 클래스 목록 확인
MATCH (c:Class)
RETURN c.name AS className, c.uri AS uri
ORDER BY c.name
```

```cypher
// 클래스 계층 구조 확인
MATCH (sub:Class)-[:SCO]->(super:Class)
RETURN sub.name AS subClass, super.name AS superClass
```

```cypher
// 오브젝트 프로퍼티(관계) 확인
MATCH (op:ObjectProperty)
OPTIONAL MATCH (op)-[:DOMAIN]->(d:Class)
OPTIONAL MATCH (op)-[:RANGE]->(r:Class)
RETURN op.name AS property, d.name AS domain, r.name AS range
```

결과를 보면 4장에서 설계한 HR 온톨로지의 뼈대가 고스란히 Neo4j 안에 들어와 있다. Employee, Department, Position, Policy, Benefit 같은 클래스들과, belongsTo, manages, appliesTo 같은 관계들이 모두 보인다. 설계도가 Neo4j라는 건물의 기초 구조로 변환된 셈이다.

물론 지금 상태에서는 스키마 뼈대만 있다. 실제 직원 데이터, 부서 데이터 같은 인스턴스는 아직 없다. 그렇다면 이제 3장에서 넣어두었던 데이터를 이 스키마 위로 옮기는 작업을 시작해보자.

## 3장 데이터 마이그레이션 — 레거시를 온톨로지로

3장에서 우리가 만들었던 데이터를 떠올려보자. 대략 이런 모양이었다.

```cypher
// 3장에서 만든 데이터 (비정형 LPG)
(:Person {name: '김철수', employee_id: 'E001', hire_date: '2020-03-15'})
  -[:WORKS_IN]->(:Dept {name: '인사팀', code: 'HR'})

(:Person {name: '이영희', employee_id: 'E002'})
  -[:REPORTS_TO]->(:Person {name: '김철수'})

(:Rule {title: '육아휴직 규정', category: '복리후생'})
```

문제가 보이는가? `Person`이라는 라벨은 온톨로지에서 `Employee`다. `Dept`는 `Department`여야 한다. `Rule`은 `Policy`다. 관계 이름도 다르다. `WORKS_IN`은 `BELONGS_TO`로, `REPORTS_TO`는 `REPORTS_TO` 그대로 쓸 수도 있지만 온톨로지에 정의된 이름과 맞추는 편이 낫다. 이런 불일치가 실무에서 레거시 마이그레이션이 번거로운 이유다.

마이그레이션 전략은 크게 세 단계로 나눈다. 라벨 재매핑, 관계 재매핑, 속성 정규화다.

### 1단계: 라벨 재매핑

기존 라벨을 온톨로지의 클래스 이름으로 바꾼다. Cypher의 `SET`과 `REMOVE`를 조합하면 된다.

```cypher
// Person → Employee 라벨 변환
MATCH (p:Person)
SET p:Employee
REMOVE p:Person
RETURN count(p) AS migratedEmployees
```

```cypher
// Dept → Department 라벨 변환
MATCH (d:Dept)
SET d:Department
REMOVE d:Dept
RETURN count(d) AS migratedDepartments
```

```cypher
// Rule → Policy 라벨 변환
MATCH (r:Rule)
SET r:Policy
REMOVE r:Rule
RETURN count(r) AS migratedPolicies
```

간단해 보이지만, 실무에서는 여기서 난감한 상황이 생긴다. 만약 하나의 노드에 여러 라벨이 붙어 있다면? 예를 들어 `(:Person:Manager)` 같은 경우, `Person`을 `Employee`로 바꾸되 `Manager`는 유지해야 할 수도 있다. 온톨로지에서 Manager가 Employee의 하위 클래스라면, 라벨을 둘 다 붙이는 게 맞을 수도 있고, Manager만 남기는 게 맞을 수도 있다. 이런 판단은 온톨로지 설계를 다시 들여다봐야 한다.

```cypher
// Manager가 Employee의 하위 클래스인 경우
MATCH (p:Person:Manager)
SET p:Employee:Manager
REMOVE p:Person
RETURN count(p) AS migratedManagers
```

### 2단계: 관계 재매핑

관계 이름을 바꾸는 건 라벨보다 조금 더 번거롭다. Cypher에서 기존 관계의 타입을 직접 변경하는 문법이 없기 때문이다. 새 관계를 만들고, 원래 관계를 지우는 방식으로 처리한다.

```cypher
// WORKS_IN → BELONGS_TO 관계 변환
MATCH (e:Employee)-[old:WORKS_IN]->(d:Department)
CREATE (e)-[new:BELONGS_TO]->(d)
DELETE old
RETURN count(new) AS migratedRelations
```

```cypher
// REPORTS_TO 관계를 온톨로지 정의에 맞게 재설정
// 매니저를 별도 노드가 아닌 Employee의 역할로 처리하는 경우
MATCH (e:Employee)-[old:REPORTS_TO]->(m:Employee)
CREATE (e)-[new:REPORTS_TO {since: old.since}]->(m)
DELETE old
RETURN count(new) AS migratedReports
```

관계에 속성이 있었다면 새 관계에 복사하는 것을 잊지 말자. 위 예시에서 `since` 속성을 옮긴 것처럼, 기존 관계의 모든 속성을 새 관계로 이전해야 데이터 손실을 막을 수 있다. 이 과정을 빼먹으면 나중에 "어? 이 데이터 어디 갔지?" 하며 아찔해지는 순간이 온다.

### 3단계: 속성 정규화

온톨로지에서 정의한 데이터 프로퍼티와 기존 데이터의 속성 이름이 다를 수 있다. 또한 데이터 타입도 맞춰야 한다.

```cypher
// employee_id → employeeId (카멜케이스로 통일)
MATCH (e:Employee)
WHERE e.employee_id IS NOT NULL
SET e.employeeId = e.employee_id
REMOVE e.employee_id
RETURN count(e) AS normalized
```

```cypher
// hire_date 문자열을 Neo4j date 타입으로 변환
MATCH (e:Employee)
WHERE e.hire_date IS NOT NULL
SET e.hireDate = date(e.hire_date)
REMOVE e.hire_date
RETURN count(e) AS dateConverted
```

```cypher
// Policy의 category 속성을 별도 노드(BenefitCategory)로 분리
MATCH (p:Policy)
WHERE p.category IS NOT NULL
MERGE (c:BenefitCategory {name: p.category})
CREATE (p)-[:CATEGORIZED_AS]->(c)
REMOVE p.category
RETURN count(p) AS categorized
```

마지막 예시가 특히 흥미롭다. 3장에서는 `category`를 단순 문자열 속성으로 넣었지만, 온톨로지 관점에서 보면 카테고리는 독립된 개념(클래스)이다. 이렇게 속성을 노드로 승격시키는 것은 그래프 모델링에서 자주 하는 리팩토링이다. 속성으로 남겨두면 "같은 카테고리에 속하는 정책들"을 찾기가 번거롭지만, 노드로 분리하면 관계를 따라가는 것만으로 쉽게 조회할 수 있다.

### 마이그레이션 검증

마이그레이션이 끝났다고 안심하기엔 이르다. 데이터가 제대로 옮겨졌는지 검증해야 한다.

```cypher
// 마이그레이션 전후 노드 수 비교
MATCH (e:Employee) RETURN 'Employee' AS label, count(e) AS count
UNION ALL
MATCH (d:Department) RETURN 'Department' AS label, count(d) AS count
UNION ALL
MATCH (p:Policy) RETURN 'Policy' AS label, count(p) AS count
```

```cypher
// 고아 노드 확인 (관계가 하나도 없는 노드)
MATCH (n)
WHERE NOT (n)--()
RETURN labels(n) AS labels, n.name AS name
```

```cypher
// 온톨로지 스키마와 실제 데이터의 라벨 일치 확인
MATCH (c:Class)
WITH collect(c.name) AS ontologyClasses
CALL db.labels() YIELD label
WHERE label IN ontologyClasses
RETURN label, 'EXISTS' AS status
```

고아 노드가 발견된다면 마이그레이션 과정에서 관계 연결이 빠진 것이다. 빠뜨린 관계를 추가해주자. 온톨로지에 정의되지 않은 라벨이 남아 있다면, 변환 스크립트에서 누락된 매핑이 있다는 뜻이다.

> **기술 리더 의사결정 박스: 기존 시스템 데이터를 어떻게 전환할 것인가?**
>
> | 전략 | 설명 | 리스크 | 소요 기간 | 적합한 경우 |
> |------|------|--------|-----------|-------------|
> | **빅뱅 마이그레이션** | 한 번에 전체 데이터를 새 스키마로 전환 | 높음 — 실패 시 롤백 복잡 | 짧음 (1~2주) | 데이터 규모 작고, 다운타임 허용 가능할 때 |
> | **점진적 이중 운영** | 구 스키마와 신 스키마를 병행 운영하며 단계적 전환 | 낮음 — 언제든 중단 가능 | 길음 (1~3개월) | 서비스 중단 불가, 데이터 정합성 검증이 중요할 때 |
> | **ETL 파이프라인** | 별도 파이프라인으로 구 데이터를 추출-변환-적재 | 중간 — 파이프라인 자체의 복잡도 | 중간 (2~4주) | 대규모 데이터, 반복 가능한 프로세스가 필요할 때 |
>
> 이 책의 HR안내봇 규모라면 빅뱅 마이그레이션으로 충분하다. 하지만 실제 엔터프라이즈 환경에서는 점진적 이중 운영이 안전한 선택인 경우가 많다. 특히 기존 시스템을 당장 끌 수 없는 상황이라면, 이중 운영하면서 신규 데이터는 새 스키마로, 기존 데이터는 배치로 마이그레이션하는 전략이 바람직하다.

## 인스턴스 데이터 확장 적재

3장의 10명 데이터를 마이그레이션했으니, 이제 HR안내봇이 제대로 동작할 수 있도록 데이터를 더 넣어보자. 직원 100명, 부서 8개, 직급 5단계, 정책 15개, 복리후생 항목 10개 규모를 목표로 한다.

대량 데이터 적재에는 여러 방법이 있지만, 여기서는 `LOAD CSV`와 Cypher `MERGE` 패턴을 결합하는 방식을 살펴보자.

### CSV 데이터 준비

```csv
// employees.csv
employeeId,name,email,hireDate,departmentCode,positionTitle,managerId
E001,김철수,cs.kim@example.com,2020-03-15,HR,팀장,
E002,이영희,yh.lee@example.com,2021-06-01,HR,주임,E001
E003,박지민,jm.park@example.com,2019-11-20,DEV,시니어 개발자,E010
...
```

```csv
// departments.csv
code,name,description,headCount
HR,인사팀,인사 관리 및 채용,15
DEV,개발팀,소프트웨어 개발,35
FIN,재무팀,재무 회계 관리,10
...
```

```csv
// policies.csv
policyId,title,category,effectiveDate,description
P001,육아휴직 규정,복리후생,2024-01-01,1년 이상 근속자 대상 최대 1년
P002,연차 사용 지침,근무,2024-03-01,입사 1년 미만 월 1일 발생
P003,재택근무 정책,근무,2025-01-01,주 2회 재택근무 가능
...
```

### LOAD CSV로 적재

```cypher
// 부서 데이터 적재
LOAD CSV WITH HEADERS FROM 'file:///departments.csv' AS row
MERGE (d:Department {code: row.code})
SET d.name = row.name,
    d.description = row.description,
    d.headCount = toInteger(row.headCount)
RETURN count(d) AS departments
```

```cypher
// 직급 데이터 적재
LOAD CSV WITH HEADERS FROM 'file:///positions.csv' AS row
MERGE (p:Position {title: row.title})
SET p.level = toInteger(row.level),
    p.description = row.description
RETURN count(p) AS positions
```

```cypher
// 직원 데이터 적재 + 부서/직급 연결
LOAD CSV WITH HEADERS FROM 'file:///employees.csv' AS row
MERGE (e:Employee {employeeId: row.employeeId})
SET e.name = row.name,
    e.email = row.email,
    e.hireDate = date(row.hireDate)
WITH e, row
MATCH (d:Department {code: row.departmentCode})
MERGE (e)-[:BELONGS_TO]->(d)
WITH e, row
MATCH (p:Position {title: row.positionTitle})
MERGE (e)-[:HAS_POSITION]->(p)
RETURN count(e) AS employees
```

```cypher
// 매니저 관계 설정 (직원 데이터가 모두 들어간 뒤 실행)
LOAD CSV WITH HEADERS FROM 'file:///employees.csv' AS row
WHERE row.managerId IS NOT NULL AND row.managerId <> ''
MATCH (e:Employee {employeeId: row.employeeId})
MATCH (m:Employee {employeeId: row.managerId})
MERGE (e)-[:REPORTS_TO]->(m)
RETURN count(*) AS managerRelations
```

```cypher
// 정책 데이터 적재 + 카테고리 연결
LOAD CSV WITH HEADERS FROM 'file:///policies.csv' AS row
MERGE (p:Policy {policyId: row.policyId})
SET p.title = row.title,
    p.effectiveDate = date(row.effectiveDate),
    p.description = row.description
WITH p, row
MERGE (c:BenefitCategory {name: row.category})
MERGE (p)-[:CATEGORIZED_AS]->(c)
RETURN count(p) AS policies
```

여기서 `CREATE` 대신 `MERGE`를 쓰는 이유가 궁금할 수 있다. `CREATE`는 매번 새 노드를 만들지만, `MERGE`는 이미 같은 키를 가진 노드가 있으면 기존 노드를 재사용한다. 데이터를 반복 적재할 때 중복이 생기지 않으니, 실무에서는 `MERGE`를 기본으로 쓰는 편이 낫다.

### 정책-부서-직급 간 관계 설정

HR 온톨로지의 핵심은 정책이 어떤 대상에게 적용되는지를 표현하는 것이다. 예를 들어 "육아휴직 규정은 모든 정규직 직원에게 적용된다"거나 "재택근무 정책은 개발팀과 기획팀에만 적용된다" 같은 관계다.

```cypher
// 정책-부서 적용 관계
MATCH (p:Policy {policyId: 'P003'})  // 재택근무 정책
MATCH (d:Department) WHERE d.code IN ['DEV', 'PLAN']
MERGE (p)-[:APPLIES_TO]->(d)
RETURN p.title, collect(d.name) AS applicableDepts
```

```cypher
// 정책-직급 적용 관계 (직급 3 이상에게만 적용되는 정책)
MATCH (p:Policy {policyId: 'P005'})  // 임원 복리후생
MATCH (pos:Position) WHERE pos.level >= 3
MERGE (p)-[:APPLIES_TO]->(pos)
RETURN p.title, collect(pos.title) AS applicablePositions
```

```cypher
// 복리후생 항목과 직원 연결
MATCH (b:Benefit {benefitId: 'B001'})  // 건강검진
MATCH (e:Employee)
MERGE (e)-[:ENTITLED_TO]->(b)
RETURN b.name, count(e) AS eligibleCount
```

## 온톨로지 기반 데이터 검증

데이터를 넣었다고 끝이 아니다. 온톨로지에 정의한 규칙을 데이터가 제대로 따르고 있는지 검증해야 한다. 온톨로지의 공리(Axiom)가 바로 이 역할을 한다.

예를 들어, 4장에서 "모든 Employee는 정확히 하나의 Department에 속해야 한다"는 제약을 정의했다면, 이를 Cypher로 검증할 수 있다.

```cypher
// 부서가 없는 직원 찾기 (제약 위반)
MATCH (e:Employee)
WHERE NOT (e)-[:BELONGS_TO]->(:Department)
RETURN e.employeeId, e.name AS orphanEmployee
```

```cypher
// 두 개 이상의 부서에 속한 직원 찾기 (카디널리티 위반)
MATCH (e:Employee)-[:BELONGS_TO]->(d:Department)
WITH e, count(d) AS deptCount
WHERE deptCount > 1
RETURN e.employeeId, e.name, deptCount
```

```cypher
// 매니저 순환 참조 확인 (A가 B에게 보고하고, B가 A에게 보고하는 경우)
MATCH path = (e:Employee)-[:REPORTS_TO*2..5]->(e)
RETURN [n IN nodes(path) | n.name] AS circularChain
```

이런 검증 쿼리들은 데이터 적재 파이프라인의 마지막 단계에 넣어두는 게 좋다. 새 데이터가 들어올 때마다 자동으로 검증하면, 잘못된 데이터가 지식그래프에 섞여 들어오는 것을 사전에 막을 수 있다. 8장에서 HR안내봇이 엉뚱한 답변을 내놓는 원인 중 상당수가 바로 이런 데이터 품질 문제에서 비롯된다는 사실을 기억해두자.

n10s에는 온톨로지 기반으로 데이터를 검증하는 기능도 내장되어 있다.

```cypher
// n10s 온톨로지 기반 SHACL 검증 (SHACL shapes가 정의된 경우)
CALL n10s.validation.shacl.import.fetch(
  'file:///hr-shapes.ttl',
  'Turtle'
)
```

```cypher
// 검증 실행
CALL n10s.validation.shacl.validate()
YIELD focusNode, resultPath, resultMessage
RETURN focusNode, resultPath, resultMessage
```

SHACL(Shapes Constraint Language)은 RDF 데이터의 형태를 정의하고 검증하는 W3C 표준이다. 온톨로지가 "어떤 데이터가 존재할 수 있는가"를 정의한다면, SHACL은 "데이터가 이 모양을 만족하는가"를 검사한다. 둘을 함께 쓰면 데이터 품질에 대한 확신이 훨씬 높아진다.

## 트러블슈팅 — 임포트와 마이그레이션의 흔한 함정

실제로 이 과정을 따라하다 보면 여러 가지 오류를 만나게 된다. 자주 발생하는 문제들과 해결법을 정리했다.

### URI 충돌

```
Neo.ClientError.Procedure.ProcedureCallFailed:
Failed to invoke procedure `n10s.onto.import.fetch`:
Caused by: n10s.RDFImportException: Multiple definitions for...
```

같은 URI를 가진 엔티티가 온톨로지 파일에 중복 정의되어 있을 때 발생한다. 온톨로지 파일을 점검해서 중복을 제거하자. Protege에서 내보낸 파일은 보통 깨끗하지만, LLM이 생성한 파일에서는 종종 이런 문제가 생긴다.

### 인코딩 문제

한국어가 포함된 온톨로지 파일을 임포트할 때 인코딩 오류가 날 수 있다. 파일이 반드시 UTF-8로 저장되어 있는지 확인하자. 특히 윈도우 환경에서 작업했다면 BOM(Byte Order Mark)이 붙어 있을 수 있는데, 이것이 파싱 오류의 원인이 되기도 한다.

### 대량 데이터 적재 시 메모리 부족

`LOAD CSV`로 만 건 이상의 데이터를 한 번에 넣으면 메모리가 부족해질 수 있다. 이때는 `USING PERIODIC COMMIT`(Neo4j 4.x) 또는 `:auto` 트랜잭션(Neo4j 5.x)을 사용한다.

```cypher
// Neo4j 5.x에서 대량 적재
:auto LOAD CSV WITH HEADERS FROM 'file:///large-employees.csv' AS row
CALL {
  WITH row
  MERGE (e:Employee {employeeId: row.employeeId})
  SET e.name = row.name, e.email = row.email
} IN TRANSACTIONS OF 500 ROWS
```

500건씩 나눠서 트랜잭션을 처리하므로 메모리 부담이 크게 줄어든다.

### 관계 타입 변경 시 속성 누락

앞서 언급했듯이, 관계를 재생성할 때 기존 속성을 복사하지 않으면 데이터가 사라진다. 마이그레이션 스크립트에서 이 부분을 꼼꼼히 점검하자. 관계에 속성이 많다면 `properties()` 함수로 한 번에 복사하는 방법도 있다.

```cypher
// 관계의 모든 속성을 안전하게 복사
MATCH (a:Employee)-[old:WORKS_IN]->(b:Department)
CREATE (a)-[new:BELONGS_TO]->(b)
SET new = properties(old)
DELETE old
RETURN count(new) AS migrated
```

`SET new = properties(old)`가 핵심이다. 이 한 줄이 기존 관계의 모든 속성을 새 관계로 복사해준다.

## 지식그래프 완성 확인

모든 작업이 끝났으면, 지식그래프의 전체 모습을 확인해보자.

```cypher
// 지식그래프 통계 요약
CALL apoc.meta.stats() YIELD labels, relTypes
RETURN labels, relTypes
```

```cypher
// 멀티홉 쿼리 테스트: "김철수의 매니저는 누구이며, 그 매니저가 관리하는 정책은?"
MATCH (e:Employee {name: '김철수'})-[:REPORTS_TO]->(m:Employee)
MATCH (m)-[:MANAGES]->(d:Department)<-[:APPLIES_TO]-(p:Policy)
RETURN e.name AS employee,
       m.name AS manager,
       d.name AS department,
       collect(p.title) AS relatedPolicies
```

```cypher
// 2홉 탐색: "인사팀 소속 직원들이 받을 수 있는 복리후생은?"
MATCH (d:Department {name: '인사팀'})<-[:BELONGS_TO]-(e:Employee)
MATCH (e)-[:ENTITLED_TO]->(b:Benefit)
RETURN d.name AS department,
       collect(DISTINCT e.name) AS employees,
       collect(DISTINCT b.name) AS benefits
```

이 쿼리들이 정상적으로 결과를 돌려준다면, 3장의 단순한 데이터가 온톨로지 기반의 지식그래프로 성공적으로 변환된 것이다. 3장에서는 "인사팀 소속 직원 목록"을 겨우 조회하는 수준이었지만, 이제는 "김철수의 매니저가 관리하는 정책"이라는 멀티홉 질문에도 답할 수 있다. 이것이 바로 온톨로지가 가져다주는 구조적 힘이다.

## 마무리

이번 장에서 우리는 설계도를 현실로 옮기는 작업을 했다. n10s로 온톨로지를 임포트하고, 3장의 레거시 데이터를 새 스키마로 마이그레이션하고, 추가 데이터를 적재해 100건 규모의 지식그래프를 완성했다. 검증 쿼리로 데이터 품질도 확인했다.

기억해두자. 온톨로지를 Neo4j에 심는 것은 일회성 작업이 아니다. 새로운 HR 정책이 추가되고, 조직 개편이 일어나면 온톨로지와 데이터를 함께 업데이트해야 한다. 9장에서 이 진화 관리를 다룰 예정이니 지금은 "지식그래프는 살아 있는 유기체"라는 감각만 가져가면 된다.

다음 장에서는 이 지식그래프에 그래프 알고리즘을 돌려보자. Louvain으로 비공식 커뮤니티를 발견하고, PageRank로 조직의 핵심 인물을 찾아낸다. 데이터가 들어 있는 지식그래프에 분석의 눈을 더하면 어떤 인사이트가 나오는지, 함께 확인해보자.

---

**HR안내봇 진행도**

| 항목 | 상태 |
|------|------|
| 3장 샘플 데이터 마이그레이션 | 완료 — 라벨, 관계, 속성 모두 온톨로지 스키마로 전환 |
| 온톨로지 임포트 | 완료 — n10s로 OWL 파일 임포트, 스키마 구조 확인 |
| 데이터 규모 | 직원 100명, 부서 8개, 정책 15개, 복리후생 10개 |
| 멀티홉 쿼리 | 가능 — "김철수의 매니저가 관리하는 정책" 조회 성공 |
| 다음 단계 | 6장에서 GDS 알고리즘으로 그래프 분석 |
