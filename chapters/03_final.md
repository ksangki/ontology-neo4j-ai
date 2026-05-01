# 3장. Neo4j 시작하기 — 그래프 데이터베이스에 첫 데이터 넣기

2장에서 우리는 종이 위에 HR 온톨로지를 스케치했다. 직원, 부서, 정책, 그리고 그 사이의 관계. 설계도는 완성되었는데, 이제 이 설계도를 실제로 구현할 도구가 필요하다. 도면만 보고 있으면 건물이 지어지지 않는 것처럼.

"데이터베이스에 넣으면 되는 거 아닌가?"라고 생각할 수 있다. 맞다. 그런데 **어떤** 데이터베이스에 넣느냐가 중요하다. 우리가 다루는 데이터의 핵심은 **관계**다. 직원과 부서의 소속 관계, 정책 간의 영향 관계, 직원과 매니저의 보고 관계. 이 관계를 자연스럽게 저장하고 탐색할 수 있는 데이터베이스가 필요하다.

관계형 데이터베이스(RDBMS)로도 관계를 표현할 수 있다. 외래 키와 JOIN을 쓰면 된다. 하지만 "김철수의 매니저의 매니저가 관리하는 정책 중 육아휴직과 관련된 것"을 쿼리하려면 JOIN이 서너 번 중첩되어야 한다. 쿼리가 복잡해지고, 성능도 떨어진다. 관계의 깊이가 깊어질수록 상황은 더 난감해진다.

**그래프 데이터베이스**는 이 문제를 위해 태어났다. 노드와 관계를 일급 시민(first-class citizen)으로 다루어, 관계 탐색이 데이터베이스의 핵심 연산이 된다. 그중에서도 **Neo4j**가 가장 널리 쓰이고, 생태계가 풍부하다. 함께 시작해보자.

## Neo4j 설치와 환경 설정

Neo4j를 시작하는 방법은 크게 두 가지다. 로컬에 직접 설치하는 방법과, 클라우드 서비스를 이용하는 방법이다.

### 경로 A: 로컬 설치 — Neo4j Desktop

로컬 환경에서 자유롭게 실험하고 싶다면 Neo4j Desktop을 설치하는 편이 낫다.

1. [neo4j.com/download](https://neo4j.com/download/)에서 Neo4j Desktop을 다운로드한다
2. 설치 후 실행하면 프로젝트를 생성할 수 있다
3. 프로젝트 안에서 "Add Database"를 클릭하고 로컬 DBMS를 생성한다
4. 데이터베이스를 시작하면 Neo4j Browser가 `http://localhost:7474`에서 열린다

Neo4j Desktop은 데이터베이스 관리, 플러그인 설치, 백업까지 GUI로 제공한다. 학습 단계에서는 이것이 가장 편하다.

### 경로 B: 클라우드 — AuraDB Free Tier

설치 없이 바로 시작하고 싶다면 Neo4j AuraDB의 무료 티어를 쓸 수 있다.

1. [neo4j.com/cloud/aura-free](https://neo4j.com/cloud/aura-free/)에서 계정을 만든다
2. Free Instance를 생성한다 (노드 20만 개, 관계 40만 개까지 무료)
3. 생성된 인스턴스의 Connection URI와 비밀번호를 기록해둔다
4. 웹 브라우저에서 바로 Neo4j Browser에 접속할 수 있다

AuraDB Free Tier는 설치 부담이 없고, 어디서든 접속할 수 있다는 장점이 있다. 다만 플러그인 설치에 제약이 있어서, 5장에서 Neosemantics(n10s) 플러그인을 사용할 때는 로컬 설치가 필요하다. 지금은 어느 쪽이든 상관없다.

어떤 경로를 선택했든, Neo4j Browser가 열리면 준비 완료다. 화면 상단의 입력창에 Cypher 쿼리를 입력하고 실행하는 방식이다.

## Labeled Property Graph — Neo4j의 데이터 모델

Neo4j에 데이터를 넣기 전에, Neo4j가 데이터를 어떻게 바라보는지 이해해야 한다. Neo4j는 **Labeled Property Graph(LPG)** 모델을 사용한다. 세 가지 핵심 요소가 있다.

### 노드(Node) — 명사

노드는 개체를 표현한다. 2장에서 배운 "인스턴스"에 해당한다. 각 노드에는 하나 이상의 **라벨(Label)**을 붙일 수 있고, **속성(Property)**을 키-값 쌍으로 가질 수 있다.

```
(김철수:Employee {name: "김철수", hireDate: "2020-03-15", age: 35})
(인사팀:Department {name: "인사팀", code: "HR-001"})
```

`김철수`는 `Employee`라는 라벨이 붙은 노드이고, `name`, `hireDate`, `age`라는 속성을 가진다. 라벨은 2장에서 배운 "클래스"에 대응하고, 속성은 "데이터 프로퍼티"에 대응한다.

### 관계(Relationship) — 동사

관계는 두 노드를 연결한다. 항상 **방향**이 있고, **타입(Type)**이 있으며, 노드와 마찬가지로 **속성**을 가질 수 있다.

```
(김철수)-[:BELONGS_TO {since: "2020-03-15"}]->(인사팀)
(김철수)-[:REPORTS_TO]->(박매니저)
```

관계에도 속성을 넣을 수 있다는 점이 중요하다. "김철수가 인사팀에 소속되어 있다"는 사실뿐 아니라, "2020년 3월 15일부터 소속되어 있다"는 맥락까지 관계에 담을 수 있다.

### 속성(Property) — 형용사

속성은 노드와 관계의 메타데이터다. 문자열, 숫자, 날짜, 불리언, 리스트 등의 타입을 지원한다. 2장에서 배운 "데이터 프로퍼티"가 여기에 해당한다.

### 그래프 데이터 모델링 원칙

2장에서 온톨로지의 구성 요소를 배울 때, 이미 모델링의 핵심 원리를 만났다. Neo4j에서도 같은 원리가 적용된다.

- **명사 → 노드:** 직원, 부서, 정책 → 각각 노드로 생성
- **동사 → 관계:** 소속, 보고, 적용 → 각각 관계로 생성
- **형용사/부사 → 속성:** 이름, 날짜, 금액 → 노드나 관계의 속성으로 저장

이 원칙을 기억해두면, 어떤 도메인의 데이터든 그래프로 모델링할 수 있다. "이것이 명사인가, 동사인가, 형용사인가?"라고 물어보면 답이 나온다.

## Cypher 기초 — 그래프를 말하는 언어

Neo4j에서 데이터를 다루는 언어가 **Cypher**다. SQL이 관계형 데이터베이스의 언어라면, Cypher는 그래프 데이터베이스의 언어다. 2011년에 Neo4j 엔지니어가 만들었고, 지금은 ISO 표준화(GQL)가 진행 중이다.

Cypher의 가장 큰 특징은 **ASCII art 스타일의 패턴 매칭**이다.

```cypher
(a)-[:KNOWS]->(b)
```

괄호가 노드, 대괄호가 관계, 화살표가 방향이다. 그래프의 모양을 텍스트로 그리는 셈이다. 직관적이라 처음 보는 사람도 대략 무슨 뜻인지 짐작할 수 있다.

### CREATE — 데이터 생성

노드와 관계를 만드는 것부터 시작하자.

```cypher
// 직원 노드 생성
CREATE (e:Employee {name: "김철수", hireDate: date("2020-03-15"), age: 35})
RETURN e
```

`CREATE` 뒤에 노드 패턴을 적으면 된다. `e`는 이 노드를 참조하기 위한 변수 이름이고, `Employee`는 라벨, 중괄호 안은 속성이다.

```cypher
// 부서 노드 생성
CREATE (d:Department {name: "인사팀", code: "HR-001"})
RETURN d
```

두 노드 사이에 관계를 만들려면 이렇게 한다.

```cypher
// 김철수를 인사팀에 소속시키기
MATCH (e:Employee {name: "김철수"})
MATCH (d:Department {name: "인사팀"})
CREATE (e)-[:BELONGS_TO {since: date("2020-03-15")}]->(d)
```

`MATCH`로 기존 노드를 찾고, `CREATE`로 그 사이에 관계를 만든다. 화살표 방향이 "김철수에서 인사팀으로"임에 주목하자.

### MATCH — 데이터 조회

이미 들어있는 데이터를 찾아보자.

```cypher
// 인사팀 소속 직원 찾기
MATCH (e:Employee)-[:BELONGS_TO]->(d:Department {name: "인사팀"})
RETURN e.name, d.name
```

`MATCH`는 그래프에서 패턴을 찾는 명령이다. "(Employee 노드)가 (BELONGS_TO 관계)로 (인사팀 Department 노드)에 연결되어 있는 패턴"을 찾고, `RETURN`으로 결과를 돌려준다.

### WHERE — 조건 필터

조건을 추가하려면 `WHERE`를 쓴다.

```cypher
// 2021년 이후 입사한 인사팀 직원
MATCH (e:Employee)-[:BELONGS_TO]->(d:Department {name: "인사팀"})
WHERE e.hireDate > date("2021-01-01")
RETURN e.name, e.hireDate
ORDER BY e.hireDate
```

SQL의 WHERE와 같은 역할이다. 비교 연산, 논리 연산(AND, OR, NOT), 문자열 매칭(CONTAINS, STARTS WITH) 등을 지원한다.

### 패턴 매칭의 진짜 힘

Cypher의 진짜 힘은 **복잡한 관계 패턴을 직관적으로 표현**할 수 있다는 데 있다. 예를 하나 살펴보자.

```cypher
// 김철수의 매니저와, 그 매니저가 속한 부서 찾기
MATCH (e:Employee {name: "김철수"})-[:REPORTS_TO]->(m:Employee)-[:BELONGS_TO]->(d:Department)
RETURN e.name AS 직원, m.name AS 매니저, d.name AS 매니저부서
```

한 줄의 MATCH로 두 번의 관계 탐색(홉)을 표현했다. SQL이었다면 JOIN을 두 번 써야 할 것이다. 관계형 데이터베이스에서는 홉이 늘어날수록 쿼리가 급격히 복잡해지지만, Cypher에서는 패턴을 이어 붙이기만 하면 된다.

```cypher
// 김철수의 매니저의 매니저가 관리하는 정책 찾기 (3홉)
MATCH (e:Employee {name: "김철수"})
      -[:REPORTS_TO]->(m1:Employee)
      -[:REPORTS_TO]->(m2:Employee)
      -[:MANAGES]->(p:Policy)
RETURN m2.name AS 상위매니저, p.name AS 정책
```

1장에서 이야기한 "멀티홉 추론"이 바로 이런 것이다. 노드에서 노드로, 관계를 따라 건너가며 답을 찾는 과정. Cypher는 이 과정을 직관적이고 읽기 쉬운 문법으로 표현한다.

### MERGE — 있으면 찾고, 없으면 만들기

실무에서 자주 쓰이는 명령이 `MERGE`다. `MATCH`와 `CREATE`를 합친 것으로, 패턴이 이미 존재하면 그것을 반환하고, 없으면 새로 생성한다.

```cypher
// 부서가 이미 있으면 찾고, 없으면 생성
MERGE (d:Department {name: "인사팀"})
ON CREATE SET d.code = "HR-001", d.createdAt = datetime()
ON MATCH SET d.updatedAt = datetime()
RETURN d
```

`ON CREATE SET`은 새로 생성될 때만, `ON MATCH SET`은 이미 존재할 때만 실행된다. 데이터 중복을 방지하면서 안전하게 데이터를 넣을 수 있어서, 데이터 로딩 스크립트에서 특히 유용하다.

### DELETE — 데이터 삭제

잘못 넣은 데이터를 삭제할 수도 있다.

```cypher
// 특정 직원 삭제 (관계도 함께 삭제)
MATCH (e:Employee {name: "테스트직원"})
DETACH DELETE e
```

`DETACH DELETE`는 노드에 연결된 모든 관계를 먼저 삭제한 뒤 노드를 삭제한다. 관계가 남아있는 노드는 삭제할 수 없기 때문에, 대부분의 경우 `DETACH DELETE`를 쓰는 편이 낫다.

## 실습: HR 샘플 데이터 입력

이론은 충분하다. 이제 실제로 HR 샘플 데이터를 Neo4j에 입력해보자. 직원 10명, 부서 3개, 정책 5개를 넣어 우리 HR안내봇의 데이터 기반을 만들 것이다.

### 부서 생성

```cypher
// 부서 3개 생성
CREATE (hr:Department {name: "인사팀", code: "HR-001"})
CREATE (dev:Department {name: "개발팀", code: "DEV-001"})
CREATE (mkt:Department {name: "마케팅팀", code: "MKT-001"})
RETURN hr, dev, mkt
```

### 직원 생성

```cypher
// 직원 10명 생성
CREATE (e1:Employee {name: "김철수", empId: "EMP-001", hireDate: date("2018-03-15"), type: "정규직"})
CREATE (e2:Employee {name: "이영희", empId: "EMP-002", hireDate: date("2019-07-01"), type: "정규직"})
CREATE (e3:Employee {name: "박매니저", empId: "EMP-003", hireDate: date("2015-01-10"), type: "정규직"})
CREATE (e4:Employee {name: "최개발", empId: "EMP-004", hireDate: date("2020-06-01"), type: "정규직"})
CREATE (e5:Employee {name: "정마케팅", empId: "EMP-005", hireDate: date("2021-02-15"), type: "정규직"})
CREATE (e6:Employee {name: "한인턴", empId: "EMP-006", hireDate: date("2025-01-02"), type: "인턴"})
CREATE (e7:Employee {name: "강시니어", empId: "EMP-007", hireDate: date("2014-05-20"), type: "정규직"})
CREATE (e8:Employee {name: "윤주니어", empId: "EMP-008", hireDate: date("2023-09-01"), type: "계약직"})
CREATE (e9:Employee {name: "송팀장", empId: "EMP-009", hireDate: date("2013-04-01"), type: "정규직"})
CREATE (e10:Employee {name: "임사원", empId: "EMP-010", hireDate: date("2024-11-15"), type: "정규직"})
```

### 정책 생성

```cypher
// 정책 5개 생성
CREATE (p1:Policy {name: "육아휴직정책", type: "LeavePolicy", effectiveDate: date("2024-01-01")})
CREATE (p2:Policy {name: "연차산정기준", type: "LeavePolicy", effectiveDate: date("2024-01-01")})
CREATE (p3:Policy {name: "재택근무정책", type: "WorkPolicy", effectiveDate: date("2025-03-01")})
CREATE (p4:Policy {name: "성과평가기준", type: "EvaluationPolicy", effectiveDate: date("2024-07-01")})
CREATE (p5:Policy {name: "복리후생규정", type: "WelfarePolicy", effectiveDate: date("2024-01-01")})
```

### 관계 생성

```cypher
// 부서 소속 관계
MATCH (e:Employee {name: "김철수"}), (d:Department {name: "인사팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "이영희"}), (d:Department {name: "인사팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "박매니저"}), (d:Department {name: "인사팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "최개발"}), (d:Department {name: "개발팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "강시니어"}), (d:Department {name: "개발팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "윤주니어"}), (d:Department {name: "개발팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "한인턴"}), (d:Department {name: "개발팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "정마케팅"}), (d:Department {name: "마케팅팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "송팀장"}), (d:Department {name: "마케팅팀"})
CREATE (e)-[:BELONGS_TO]->(d);

MATCH (e:Employee {name: "임사원"}), (d:Department {name: "마케팅팀"})
CREATE (e)-[:BELONGS_TO]->(d);
```

```cypher
// 보고 관계
MATCH (e:Employee {name: "김철수"}), (m:Employee {name: "박매니저"})
CREATE (e)-[:REPORTS_TO]->(m);

MATCH (e:Employee {name: "이영희"}), (m:Employee {name: "박매니저"})
CREATE (e)-[:REPORTS_TO]->(m);

MATCH (e:Employee {name: "최개발"}), (m:Employee {name: "강시니어"})
CREATE (e)-[:REPORTS_TO]->(m);

MATCH (e:Employee {name: "한인턴"}), (m:Employee {name: "최개발"})
CREATE (e)-[:REPORTS_TO]->(m);

MATCH (e:Employee {name: "윤주니어"}), (m:Employee {name: "강시니어"})
CREATE (e)-[:REPORTS_TO]->(m);

MATCH (e:Employee {name: "정마케팅"}), (m:Employee {name: "송팀장"})
CREATE (e)-[:REPORTS_TO]->(m);

MATCH (e:Employee {name: "임사원"}), (m:Employee {name: "송팀장"})
CREATE (e)-[:REPORTS_TO]->(m);
```

```cypher
// 정책 적용 관계
MATCH (p:Policy {name: "육아휴직정책"}), (d:Department {name: "인사팀"})
CREATE (p)-[:APPLIES_TO]->(d);
MATCH (p:Policy {name: "육아휴직정책"}), (d:Department {name: "개발팀"})
CREATE (p)-[:APPLIES_TO]->(d);
MATCH (p:Policy {name: "육아휴직정책"}), (d:Department {name: "마케팅팀"})
CREATE (p)-[:APPLIES_TO]->(d);

MATCH (p:Policy {name: "재택근무정책"}), (d:Department {name: "개발팀"})
CREATE (p)-[:APPLIES_TO]->(d);

MATCH (p:Policy {name: "성과평가기준"}), (d:Department {name: "인사팀"})
CREATE (p)-[:APPLIES_TO]->(d);
MATCH (p:Policy {name: "성과평가기준"}), (d:Department {name: "개발팀"})
CREATE (p)-[:APPLIES_TO]->(d);
MATCH (p:Policy {name: "성과평가기준"}), (d:Department {name: "마케팅팀"})
CREATE (p)-[:APPLIES_TO]->(d);

// 정책 간 영향 관계
MATCH (p1:Policy {name: "육아휴직정책"}), (p2:Policy {name: "연차산정기준"})
CREATE (p1)-[:AFFECTS {description: "휴직 기간 연차 미발생"}]->(p2);
```

### 데이터 확인

모든 데이터가 제대로 들어갔는지 확인해보자.

```cypher
// 전체 그래프 시각화
MATCH (n)
OPTIONAL MATCH (n)-[r]->(m)
RETURN n, r, m
```

Neo4j Browser에서 이 쿼리를 실행하면, 노드와 관계가 시각적으로 그려진 그래프를 볼 수 있다. 직원 노드들이 부서 노드에 연결되어 있고, 정책 노드들이 부서에 적용되어 있는 그림이 나타날 것이다.

### 쿼리 실습

몇 가지 실용적인 쿼리를 직접 날려보자.

```cypher
// 인사팀 소속 직원 목록
MATCH (e:Employee)-[:BELONGS_TO]->(d:Department {name: "인사팀"})
RETURN e.name AS 직원명, e.type AS 고용형태, e.hireDate AS 입사일
ORDER BY e.hireDate
```

```cypher
// 김철수의 매니저 찾기
MATCH (e:Employee {name: "김철수"})-[:REPORTS_TO]->(m:Employee)
RETURN m.name AS 매니저
```

```cypher
// 개발팀에 적용되는 정책 목록
MATCH (p:Policy)-[:APPLIES_TO]->(d:Department {name: "개발팀"})
RETURN p.name AS 정책명, p.type AS 정책유형, p.effectiveDate AS 시행일
```

```cypher
// 육아휴직정책이 영향을 미치는 다른 정책 찾기 (멀티홉)
MATCH (p1:Policy {name: "육아휴직정책"})-[a:AFFECTS]->(p2:Policy)
RETURN p1.name AS 원인정책, a.description AS 영향내용, p2.name AS 영향받는정책
```

```cypher
// 각 부서별 직원 수 집계
MATCH (e:Employee)-[:BELONGS_TO]->(d:Department)
RETURN d.name AS 부서명, count(e) AS 직원수
ORDER BY 직원수 DESC
```

```cypher
// 매니저가 없는 직원 찾기 (최상위 관리자)
MATCH (e:Employee)
WHERE NOT (e)-[:REPORTS_TO]->()
RETURN e.name AS 최상위관리자
```

마지막 쿼리 결과에 박매니저, 강시니어, 송팀장이 나올 것이다. 이 세 사람은 REPORTS_TO 관계의 출발점이 되지 않는, 즉 "보고할 상위자가 없는" 직원이다. 이런 패턴을 `WHERE NOT (e)-[:REPORTS_TO]->()`로 간결하게 표현할 수 있다는 것이 Cypher의 매력이다.

## 인덱스와 제약 조건

데이터가 적을 때는 아무 문제 없지만, 데이터가 수천, 수만 건으로 늘어나면 성능이 중요해진다. 인덱스와 제약 조건을 미리 설정해두는 편이 현명하다.

```cypher
// Employee의 empId에 유니크 제약 (중복 방지 + 자동 인덱스)
CREATE CONSTRAINT employee_empid IF NOT EXISTS
FOR (e:Employee) REQUIRE e.empId IS UNIQUE;

// Department의 name에 유니크 제약
CREATE CONSTRAINT department_name IF NOT EXISTS
FOR (d:Department) REQUIRE d.name IS UNIQUE;

// Policy의 name에 유니크 제약
CREATE CONSTRAINT policy_name IF NOT EXISTS
FOR (p:Policy) REQUIRE p.name IS UNIQUE;

// Employee의 name에 인덱스 (검색 성능 향상)
CREATE INDEX employee_name IF NOT EXISTS
FOR (e:Employee) ON (e.name);
```

유니크 제약(UNIQUE constraint)은 두 가지 역할을 한다. 첫째, 같은 값을 가진 노드가 중복 생성되는 것을 방지한다. 둘째, 자동으로 인덱스도 생성해서 조회 성능을 높인다. 일석이조다.

## 데이터 연속성 — 이 데이터의 미래

여기서 한 가지 중요한 이야기를 하고 넘어가야 한다.

지금 입력한 HR 샘플 데이터는 Cypher 학습용 스캐폴딩이다. 라벨 체계나 관계 타입이 아직 온톨로지에 기반하지 않았다. `Employee`라는 라벨, `BELONGS_TO`라는 관계 타입은 우리가 직감적으로 지은 이름이지, 2장에서 설계한 온톨로지의 정식 체계를 따른 것이 아니다.

그래서 이 데이터는 5장에서 **마이그레이션**을 겪게 된다. 4장에서 Protege로 정식 온톨로지를 설계한 뒤, 5장에서 이 샘플 데이터의 라벨, 관계, 속성을 온톨로지 기반 스키마로 재매핑하는 것이다. 마치 실무에서 "레거시 데이터를 새 스키마로 전환하는" 경험을 미리 체득하는 셈이다.

그러니 지금 입력한 데이터를 삭제하지 말자. 이 데이터는 5장에서 다시 만날 레거시 친구다.

> **기술 리더 의사결정 박스: 어떤 그래프 데이터베이스를 선택할 것인가?**
>
> Neo4j만이 유일한 선택지는 아니다. 주요 대안을 비교해보자.
>
> | 기준 | Neo4j | Amazon Neptune | TigerGraph | NebulaGraph |
> |------|-------|---------------|------------|-------------|
> | **데이터 모델** | LPG (Labeled Property Graph) | RDF + LPG 병행 | LPG | LPG |
> | **쿼리 언어** | Cypher | SPARQL + Gremlin + openCypher | GSQL | nGQL |
> | **생태계** | 가장 풍부 (GDS, n10s, GraphRAG 등) | AWS 통합 우수 | 분석 특화 | 오픈소스 |
> | **GDS(그래프 분석)** | 60+ 알고리즘 | 제한적 | 강력 | 제한적 |
> | **온톨로지 지원** | n10s 플러그인 | 네이티브 RDF | 제한적 | 제한적 |
> | **비용** | Community(무료) / Enterprise | AWS 종량제 | Enterprise | 오픈소스(무료) |
> | **적합 시나리오** | GraphRAG, 지식그래프, 범용 | AWS 환경, RDF 중심 | 대규모 분석 | 대규모 그래프 |
>
> **이 책에서 Neo4j를 선택한 이유:**
> 1. GraphRAG 생태계가 가장 성숙하다 (공식 Python 패키지, LangChain 통합)
> 2. GDS 라이브러리로 그래프 알고리즘 실습이 가능하다 (6장)
> 3. Neosemantics(n10s)로 온톨로지를 직접 임포트할 수 있다 (5장)
> 4. 한국어 커뮤니티와 자료가 가장 풍부하다
>
> **판단 기준:** (a) 이미 AWS에 깊이 묶여 있다면 Neptune, (b) 초대규모 분석이 핵심이면 TigerGraph, (c) 지식그래프 + AI 통합이 목표면 Neo4j가 현실적인 선택이다.

## Neo4j를 프로그래밍 언어에서 사용하기

Neo4j Browser에서 직접 Cypher를 입력하는 것도 좋지만, 실제 애플리케이션에서는 프로그래밍 언어에서 Neo4j에 접속해야 한다. Python 예제를 간단히 살펴보자.

```python
from neo4j import GraphDatabase

# Neo4j 연결
driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "your-password")
)

# 인사팀 직원 조회
def get_hr_employees(driver):
    with driver.session() as session:
        result = session.run("""
            MATCH (e:Employee)-[:BELONGS_TO]->(d:Department {name: "인사팀"})
            RETURN e.name AS name, e.hireDate AS hireDate
            ORDER BY e.hireDate
        """)
        for record in result:
            print(f"{record['name']} (입사일: {record['hireDate']})")

get_hr_employees(driver)
driver.close()
```

`neo4j` Python 드라이버를 설치하고(`pip install neo4j`), 연결 URI와 인증 정보를 넘기면 된다. Cypher 쿼리를 문자열로 전달하고 결과를 순회하는 방식이다. 8장에서 HR안내봇의 백엔드를 구현할 때 이 패턴을 본격적으로 활용할 것이다.

## 마무리

이번 장에서 우리는 Neo4j를 설치하고, Labeled Property Graph 모델을 이해하고, Cypher의 기본 문법을 익혔다. 그리고 HR 샘플 데이터를 실제로 Neo4j에 입력하여, "인사팀 소속 직원 목록"부터 "정책 간 영향 관계"까지 다양한 쿼리를 실행해보았다.

기초가 갖춰졌다. 그런데 솔직히, 지금 만든 데이터 구조가 조금 찜찜하지 않은가? `Employee`라는 라벨은 누가 정한 건지, `BELONGS_TO`라는 관계 타입은 표준인지 우리가 지은 건지, 정규직과 계약직을 `type` 속성으로 구분한 것은 클래스로 나눠야 하는 건 아닌지. 지금은 데이터가 10건이라 아무 문제 없지만, 수천 건, 수만 건으로 늘어나면 이런 "대충 정한" 구조가 발목을 잡을 수 있다.

바로 이것이 **온톨로지 설계**가 필요한 이유다. 다음 장에서는 Protege라는 도구로 HR 온톨로지를 정식으로 설계하고, LLM을 활용해 자동 생성하는 방법도 함께 알아보자. 두 가지 길 중 어느 쪽이 우리 프로젝트에 맞는지, 직접 비교해볼 것이다.

---

**HR안내봇 진행도**

| 항목 | 상태 |
|------|------|
| 이번 장에서 한 것 | Neo4j에 HR 샘플 데이터 입력 완료 (직원 10명, 부서 3개, 정책 5개) |
| 산출물 | Neo4j 데이터베이스에 노드 18개, 관계 20+개 적재. 기본 Cypher 쿼리 실행 확인 |
| 핵심 쿼리 | "인사팀 소속 직원 목록", "김철수의 매니저", "정책 간 영향 관계" 조회 가능 |
| 데이터 연속성 | 이 데이터는 5장에서 온톨로지 스키마로 마이그레이션 예정 — 삭제하지 말 것 |
| 다음 장 예고 | 온톨로지 설계 실전 — Protege 수동 설계와 LLM 자동 생성, 두 가지 길 비교 |
