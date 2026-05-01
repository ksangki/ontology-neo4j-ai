# 4장. 온톨로지 설계 실전 — 수동 설계와 AI 자동 생성, 두 가지 길

HR 도메인의 핵심 개념을 종이 위에 스케치했고(2장), Neo4j에 샘플 데이터도 넣어보았다(3장). 그런데 3장 마지막에 던진 질문이 여전히 남아 있다. `Employee`라는 라벨은 누가 정한 건가? `BELONGS_TO`라는 관계 타입은 표준인가, 우리가 임의로 지은 건가? 정규직과 계약직을 속성 하나로 구분한 것이 정말 최선인가?

이 찜찜함의 정체는 명확하다. **설계도 없이 건물을 짓기 시작한 것**이다. 3장의 데이터는 학습용 스캐폴딩이었으니 괜찮지만, 실제 프로젝트에서 이렇게 시작하면 나중에 구조 변경이 끔찍한 비용을 초래한다.

이번 장에서는 HR 온톨로지를 **정식으로** 설계한다. 그런데 설계하는 방법이 하나가 아니다. 전통적인 방법, 즉 온톨로지 편집기를 사용해 수동으로 설계하는 **경로 A**가 있고, LLM에게 도메인 문서를 입력하여 자동 생성하는 **경로 B**가 있다. 두 경로를 모두 따라간 뒤, 어느 쪽이 우리 프로젝트에 맞는지 비교해보자.

## 온톨로지 개발 8단계 프로세스

어떤 경로를 택하든, 온톨로지 설계의 기본 프로세스를 알아두는 편이 낫다. 온톨로지 엔지니어링에서 널리 쓰이는 8단계 프로세스를 살펴보자.

**1단계: 도메인 범위 정의.** 이 온톨로지가 답해야 하는 질문은 무엇인가? HR안내봇의 경우, 1장에서 정의한 20개 질문 시나리오가 범위를 결정한다. "육아휴직 중 연차는 어떻게 되나요?", "인사팀 소속 직원 목록을 알려주세요" 같은 질문에 답하는 데 필요한 개념과 관계가 온톨로지의 범위다.

**2단계: 기존 온톨로지 조사.** 바퀴를 다시 발명할 필요는 없다. Schema.org, Dublin Core, FOAF 같은 표준 온톨로지에서 재사용할 수 있는 부분이 있는지 살펴본다. HR 도메인에서는 Schema.org의 `Person`, `Organization` 클래스를 참고할 수 있다.

**3단계: 핵심 용어 열거.** 도메인의 주요 명사와 동사를 수집한다. HR 도메인이라면: 직원, 부서, 직급, 정책, 복리후생, 소속, 보고, 적용, 관리, 수혜 등.

**4단계: 클래스 계층 설계.** 수집한 명사를 클래스로 만들고, 상위-하위 관계를 정의한다. 2장에서 스케치한 클래스 계층을 정교화하는 단계다.

**5단계: 프로퍼티와 관계 정의.** 각 클래스의 속성과 클래스 간 관계를 구체적으로 정의한다.

**6단계: 제약 조건 추가.** 카디널리티, 필수 관계, 값 범위 등의 공리를 추가한다.

**7단계: 인스턴스 생성 및 검증.** 실제 데이터를 온톨로지에 맞추어 넣어보며, 설계가 현실과 맞는지 검증한다.

**8단계: 반복 개선.** 실사용 피드백으로 지속적으로 보완한다.

이 8단계가 선형적으로 진행되는 것은 아니다. 4단계에서 클래스를 설계하다가 3단계로 돌아가 누락된 용어를 추가하는 일은 흔하다. 온톨로지 설계는 본질적으로 반복적(iterative)이다.

이제 이 프로세스를 경로 A(수동 설계)와 경로 B(AI 자동 생성)로 각각 실행해보자.

## 경로 A: Protege로 수동 설계

### Protege란?

Protege는 Stanford 대학이 개발한 오픈소스 온톨로지 편집기다. 온톨로지 설계의 사실상 표준 도구로, OWL 2를 완전히 지원하고, 추론기(Reasoner) 통합, 시각적 편집, 플러그인 아키텍처를 갖추고 있다.

[protege.stanford.edu](https://protege.stanford.edu/)에서 무료로 다운로드할 수 있다. 설치 후 실행하면 빈 온톨로지가 열리고, 여기에 클래스, 프로퍼티, 관계, 제약 조건을 하나씩 추가해나간다.

### 클래스 계층 만들기

Protege의 "Classes" 탭에서 클래스 계층을 생성한다. 2장에서 스케치한 구조를 정식으로 옮겨보자.

```
owl:Thing
├── Agent
│   └── Person
│       └── Employee
│           ├── FullTimeEmployee
│           └── ContractEmployee
├── OrganizationalUnit
│   └── Department
├── Position
│   ├── ManagerPosition
│   └── StaffPosition
├── Policy
│   ├── LeavePolicy
│   ├── SalaryPolicy
│   ├── WorkPolicy
│   └── WelfarePolicy
└── Benefit
    ├── HealthBenefit
    ├── EducationBenefit
    └── WelfareBenefit
```

몇 가지 변화가 눈에 띈다. 2장 스케치에서는 `Manager`를 `Position`의 하위로 두었는데, 여기서는 `ManagerPosition`으로 이름을 바꾸었다. "매니저"가 사람인지 직급인지 모호하지 않도록 하기 위해서다. `Agent`라는 상위 클래스도 추가했는데, 나중에 외부 시스템이나 AI 에이전트를 추가할 때 확장 포인트가 된다.

Protege에서 클래스를 추가하는 것은 직관적이다. 상위 클래스를 선택하고 "Add subclass" 버튼을 누르면 된다.

### 오브젝트 프로퍼티(관계) 정의

"Object Properties" 탭에서 클래스 간 관계를 정의한다.

```
belongsTo
  - Domain: Employee
  - Range: Department
  - Inverse: hasMember
  - Characteristics: Functional (직원은 하나의 부서에만 소속)

reportsTo
  - Domain: Employee
  - Range: Employee
  - Characteristics: Asymmetric, Irreflexive

hasPosition
  - Domain: Employee
  - Range: Position
  - Characteristics: Functional

appliesTo
  - Domain: Policy
  - Range: Employee ∪ Department

affectsPolicy
  - Domain: Policy
  - Range: Policy

receivesBenefit
  - Domain: Employee
  - Range: Benefit

managedBy
  - Domain: Department
  - Range: Employee
```

각 프로퍼티에 **Domain(출발 클래스)**과 **Range(도착 클래스)**를 지정한다. `belongsTo`의 Domain은 `Employee`이고 Range는 `Department`이니, "직원이 부서에 소속된다"를 의미한다.

**Functional** 특성을 지정하면, 하나의 직원은 최대 하나의 부서에만 소속될 수 있다는 제약이 걸린다. **Asymmetric**은 "A가 B에게 보고하면, B는 A에게 보고할 수 없다"를 보장한다. **Irreflexive**는 "자기 자신에게 보고할 수 없다"를 보장한다. 이런 특성들이 2장에서 배운 "공리"를 구현하는 방법이다.

### 데이터 프로퍼티 정의

"Data Properties" 탭에서 속성을 정의한다.

```
employeeId
  - Domain: Employee
  - Range: xsd:string
  - Characteristics: Functional

name
  - Domain: Agent
  - Range: xsd:string

hireDate
  - Domain: Employee
  - Range: xsd:date

effectiveDate
  - Domain: Policy
  - Range: xsd:date

departmentCode
  - Domain: Department
  - Range: xsd:string
  - Characteristics: Functional
```

데이터 프로퍼티의 Range는 문자열(`xsd:string`), 날짜(`xsd:date`), 숫자(`xsd:integer`) 같은 데이터 타입이다.

### 제약 조건(공리) 추가

클래스의 "Description" 패널에서 추가적인 제약을 설정할 수 있다.

**Employee 클래스:**
```
Employee SubClassOf belongsTo exactly 1 Department
Employee SubClassOf hasPosition exactly 1 Position
Employee SubClassOf name exactly 1 xsd:string
Employee SubClassOf employeeId exactly 1 xsd:string
```

이것은 "모든 직원은 정확히 하나의 부서에 소속되어야 하고, 정확히 하나의 직급을 가져야 하며, 이름과 사번은 각각 하나씩 있어야 한다"는 의미다.

**FullTimeEmployee 클래스:**
```
FullTimeEmployee EquivalentTo Employee and (employmentType value "정규직")
```

이 등가 공리(equivalent class)는 "정규직 직원은 employmentType이 '정규직'인 직원과 동일하다"를 정의한다. 추론기가 이 규칙을 사용해 자동 분류를 수행할 수 있다.

### 추론기로 일관성 검사

Protege에 내장된 추론기(HermiT 또는 Pellet)를 실행하면, 온톨로지의 **논리적 일관성**을 자동으로 검사한다.

- 모순된 클래스가 없는가? (어떤 인스턴스도 속할 수 없는 클래스)
- 불필요한 중복이 없는가? (사실상 같은 클래스가 다른 이름으로 존재)
- 제약 조건이 서로 충돌하지 않는가?

추론기를 돌려보면 가끔 예상치 못한 문제가 발견된다. 예를 들어, `FullTimeEmployee`와 `ContractEmployee`가 서로 배타적이라고 정의하지 않으면, 추론기가 "한 직원이 동시에 정규직이면서 계약직일 수 있다"는 결론을 내릴 수 있다. 이런 문제를 설계 단계에서 잡아내는 것이 추론기의 가치다.

```
FullTimeEmployee DisjointWith ContractEmployee
```

이 한 줄을 추가하면, 한 인스턴스가 두 클래스에 동시에 속하는 것이 불가능해진다.

### OWL 파일로 내보내기

설계가 완성되면 File > Save As에서 OWL/Turtle 형식으로 내보낸다.

```turtle
@prefix hr: <http://example.org/hr-ontology#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

hr:Employee a owl:Class ;
    rdfs:subClassOf hr:Person ;
    rdfs:subClassOf [
        a owl:Restriction ;
        owl:onProperty hr:belongsTo ;
        owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
        owl:onClass hr:Department
    ] .

hr:FullTimeEmployee a owl:Class ;
    rdfs:subClassOf hr:Employee ;
    owl:disjointWith hr:ContractEmployee .

hr:belongsTo a owl:ObjectProperty, owl:FunctionalProperty ;
    rdfs:domain hr:Employee ;
    rdfs:range hr:Department ;
    owl:inverseOf hr:hasMember .

hr:reportsTo a owl:ObjectProperty, owl:AsymmetricProperty, owl:IrreflexiveProperty ;
    rdfs:domain hr:Employee ;
    rdfs:range hr:Employee .
```

이 Turtle 파일이 5장에서 Neo4j에 임포트할 온톨로지의 정식 산출물이다.

## 경로 B: LLM으로 자동 생성

경로 A는 정교하지만, 솔직히 번거롭다. Protege를 설치하고, OWL 문법을 이해하고, 클래스와 관계를 하나씩 수동으로 추가해야 한다. 소규모 온톨로지라면 몇 시간이면 되지만, 클래스가 수십 개, 관계가 수백 개인 복잡한 도메인이라면 몇 주가 걸릴 수도 있다.

그렇다면 이 과정을 AI에게 맡기면 어떨까? 2025~2026년의 LLM은 도메인 문서를 입력받아 온톨로지 초안을 자동으로 생성할 수 있는 수준에 도달했다.

### LLM에게 온톨로지 초안 요청하기

가장 직접적인 방법은 LLM에게 도메인 문서와 요구사항을 주고 온톨로지를 생성해달라고 요청하는 것이다.

```
프롬프트 예시:

다음 HR 도메인 정보를 바탕으로 OWL 온톨로지를 Turtle 형식으로 생성해주세요.

[도메인 설명]
- 회사에는 직원(Employee), 부서(Department), 직급(Position)이 있습니다
- 직원은 정규직(FullTime)과 계약직(Contract)으로 나뉩니다
- 직원은 하나의 부서에 소속되며, 하나의 직급을 가집니다
- 직원은 다른 직원에게 보고합니다 (매니저-부하 관계)
- 회사에는 정책(Policy)이 있으며, 정책은 직원이나 부서에 적용됩니다
- 정책 간에는 영향 관계가 있습니다 (예: 육아휴직이 연차 산정에 영향)
- 복리후생(Benefit)이 있으며, 직원이 수혜합니다

[요구사항]
- OWL DL 수준의 표현력
- 카디널리티 제약, 불리언 특성(Asymmetric, Functional 등) 포함
- 데이터 프로퍼티(이름, 날짜, ID 등) 포함
- Turtle 형식으로 출력
```

LLM은 이 프롬프트를 받고 몇 초 만에 Turtle 형식의 온톨로지를 생성한다. 경로 A에서 몇 시간 걸리는 작업이 몇 분으로 줄어드는 셈이다.

### Neo4j LLM Knowledge Graph Builder 활용

더 체계적인 방법도 있다. Neo4j가 제공하는 오픈소스 도구인 **LLM Knowledge Graph Builder**는 비정형 텍스트에서 엔티티와 관계를 자동으로 추출하여 지식그래프를 구축한다.

이 도구의 워크플로우는 이렇다.

1. HR 정책 문서(PDF, 텍스트)를 입력한다
2. LLM이 문서에서 엔티티(직원, 부서, 정책 등)를 추출한다
3. 엔티티 간 관계(소속, 적용, 영향 등)를 식별한다
4. 추출된 엔티티와 관계를 Neo4j 지식그래프로 구축한다
5. 커뮤니티 요약 기능으로 관련 엔티티를 자동 그룹핑한다

이 과정에서 온톨로지가 명시적으로 생성되지는 않지만, 추출된 엔티티 유형과 관계 유형이 사실상 암묵적 온톨로지(implicit ontology) 역할을 한다. 이 암묵적 구조를 Turtle 파일로 정리하면 정식 온톨로지가 된다.

### LLM 생성 결과의 한계

LLM이 생성한 온톨로지가 완벽할까? 솔직히 그렇지 않다. 몇 가지 전형적인 문제가 있다.

**첫째, 일관성 부족.** 같은 프롬프트를 두 번 던져도 다른 결과가 나올 수 있다. 어떤 때는 `belongsTo`를, 어떤 때는 `memberOf`를 관계 이름으로 쓴다.

**둘째, 도메인 깊이 부족.** LLM은 HR 도메인의 일반적인 구조는 잘 잡아내지만, "육아휴직 기간 중 연차 미발생"처럼 특정 조직의 구체적인 규칙은 놓칠 수 있다. 프롬프트에 포함되지 않은 도메인 지식은 반영되지 않는다.

**셋째, 공리 설계의 취약성.** LLM이 클래스와 관계는 비교적 잘 생성하지만, 정교한 공리(카디널리티 제약, 배타 클래스 등)는 누락하거나 잘못 정의하는 경우가 있다.

이 때문에 **"AI가 초안을 잡고, 전문가가 정제하는" 협업 모델**이 현재로서는 최적이다. LLM이 80%를 빠르게 만들어주면, 도메인 전문가가 나머지 20%를 검증하고 보완한다.

### 검증 체크리스트

LLM이 생성한 온톨로지를 검증할 때 확인해야 할 항목이다.

- [ ] 모든 핵심 클래스가 포함되어 있는가?
- [ ] 클래스 계층이 논리적인가? (하위 클래스가 상위 클래스의 특수화인가?)
- [ ] 관계의 Domain과 Range가 정확한가?
- [ ] 필요한 카디널리티 제약이 있는가? (예: 직원은 부서 하나에만 소속)
- [ ] 배타적 클래스(Disjoint Class)가 설정되어 있는가? (예: 정규직 vs 계약직)
- [ ] 데이터 프로퍼티의 타입이 적절한가? (날짜에 xsd:date, 이름에 xsd:string)
- [ ] 우리의 질문 시나리오에 답하는 데 필요한 관계가 모두 존재하는가?

이 체크리스트를 통과하면, LLM이 생성한 온톨로지도 Protege에서 수동 설계한 것과 동등한 품질에 도달할 수 있다.

## 두 경로 비교

경로 A와 경로 B를 나란히 놓고 비교해보자.

| 기준 | 경로 A: Protege 수동 설계 | 경로 B: LLM 자동 생성 + 정제 |
|------|--------------------------|----------------------------|
| **소요 시간** | 수 시간 ~ 수일 | 수 분 ~ 수 시간 (정제 포함) |
| **정교함** | 높음 — 공리까지 세밀하게 제어 | 중간 — 공리 품질은 검증 필요 |
| **재현성** | 높음 — 같은 과정을 반복하면 같은 결과 | 낮음 — LLM 출력이 매번 다를 수 있음 |
| **도메인 적합성** | 높음 — 도메인 전문가가 직접 설계 | 중간 — 프롬프트에 포함된 정보에 의존 |
| **학습 곡선** | 가파름 — Protege + OWL 학습 필요 | 완만 — 프롬프트 작성 능력이면 충분 |
| **확장성** | 낮음 — 수동 작업은 규모에 비례하여 증가 | 높음 — 대규모 문서도 빠르게 처리 |
| **적합 상황** | 핵심 도메인, 높은 품질 요구, 규제 환경 | 빠른 PoC, 대규모 도메인, 초기 탐색 |

어떤 것이 "더 좋다"가 아니라, **상황에 따라 최적이 다르다**는 것이 핵심이다.

규제가 엄격한 금융이나 의료 도메인에서, 온톨로지의 모든 공리가 법적 요건을 반영해야 한다면, 경로 A가 안전하다. 반면 빠르게 PoC를 만들어 이해관계자에게 보여주고, 피드백을 받아 반복 개선해야 한다면, 경로 B로 시작하는 것이 효율적이다.

그리고 실무에서 가장 많이 쓰이는 방식은 사실 **하이브리드**다. 경로 B로 초안을 빠르게 잡고, 경로 A(Protege)에서 정제하는 것이다. Juan Sequeda의 교훈을 다시 기억해두자. **"충분히 좋은 온톨로지가 완벽한 온톨로지보다 낫다."** 완벽을 추구하다 프로젝트가 멈추는 것보다, "충분히 좋은" 온톨로지로 시작하여 운영하면서 개선하는 것이 현실적이다.

> **기술 리더 의사결정 박스: 온톨로지 설계를 팀에 어떻게 도입할 것인가?**
>
> | 접근법 | 비용 | 일정 | 품질 | 적합 상황 |
> |--------|------|------|------|----------|
> | **(a) 수동 설계 전담 인력** | 높음 (온톨로지 엔지니어 채용/교육) | 느림 (2~4주+) | 최고 | 규제 환경, 핵심 비즈니스 온톨로지 |
> | **(b) LLM 자동 생성 + 전문가 검수** | 낮음 (기존 인력 활용) | 빠름 (2~5일) | 높음 (검수 품질에 의존) | PoC, 빠른 검증, 대규모 도메인 |
> | **(c) 하이브리드** | 중간 | 보통 (1~2주) | 높음 | 대부분의 프로젝트에 권장 |
>
> **권장:** 대부분의 기술 리더에게는 **(c) 하이브리드**가 가장 현실적이다. LLM으로 초안을 잡아 시간을 벌고, 도메인 전문가가 핵심 공리와 제약 조건을 검증/보완한다. 팀에 온톨로지 전문가가 없어도 시작할 수 있고, 나중에 전문성이 쌓이면 (a)로 전환할 수 있다.

## 기존 표준 온톨로지 재활용

처음부터 모든 것을 만들 필요는 없다. 이미 잘 설계된 표준 온톨로지를 재활용하면 시간을 절약하고 상호운용성도 높일 수 있다.

**Schema.org:** Google, Microsoft, Yahoo가 공동으로 관리하는 어휘. `Person`, `Organization`, `Place` 같은 범용 클래스를 제공한다. HR 온톨로지의 `Employee`를 Schema.org의 `Person`의 하위 클래스로 정의하면, 외부 시스템과의 데이터 교환이 수월해진다.

**Dublin Core:** 메타데이터 표준. `title`, `creator`, `date` 같은 기본 메타데이터 속성을 제공한다. 정책 문서의 메타데이터를 표현할 때 활용할 수 있다.

**FOAF (Friend of a Friend):** 사람과 사회적 관계를 기술하는 온톨로지. `Person`, `knows`, `member` 같은 클래스와 관계를 제공한다. 조직 내 인적 네트워크를 표현할 때 참고할 만하다.

재활용할 때 주의할 점이 하나 있다. 표준 온톨로지를 **그대로** 쓰는 것이 아니라, **참조하고 확장**하는 것이다. Schema.org의 `Person`을 가져오되, HR 도메인에 특화된 속성과 관계는 우리가 추가한다. 이렇게 하면 표준의 이점(상호운용성)과 도메인 특화의 이점(정확성)을 모두 취할 수 있다.

## HR 온톨로지 완성

경로 A와 경로 B를 모두 거쳐, 우리의 HR 온톨로지 최종 사양을 정리하자.

### 클래스 (12개)

```
owl:Thing
├── Agent
│   └── Person
│       └── Employee
│           ├── FullTimeEmployee (disjoint with ContractEmployee)
│           └── ContractEmployee (disjoint with FullTimeEmployee)
├── OrganizationalUnit
│   └── Department
├── Position
│   ├── ManagerPosition
│   └── StaffPosition
├── Policy
│   ├── LeavePolicy
│   ├── SalaryPolicy
│   ├── WorkPolicy
│   └── WelfarePolicy
└── Benefit
    ├── HealthBenefit
    └── EducationBenefit
```

### 오브젝트 프로퍼티(관계, 15개)

| 관계 | Domain | Range | 특성 |
|------|--------|-------|------|
| belongsTo | Employee | Department | Functional, InverseFunctional |
| hasMember | Department | Employee | inverse of belongsTo |
| reportsTo | Employee | Employee | Asymmetric, Irreflexive |
| hasDirectReport | Employee | Employee | inverse of reportsTo |
| hasPosition | Employee | Position | Functional |
| appliesTo | Policy | Employee | - |
| appliesToDepartment | Policy | Department | - |
| affectsPolicy | Policy | Policy | - |
| affectedByPolicy | Policy | Policy | inverse of affectsPolicy |
| receivesBenefit | Employee | Benefit | - |
| offersBenefit | Department | Benefit | - |
| managedBy | Department | Employee | Functional |
| manages | Employee | Department | inverse of managedBy |
| hasSubPolicy | Policy | Policy | Transitive |
| relatedPolicy | Policy | Policy | Symmetric |

### 데이터 프로퍼티 (주요)

| 속성 | Domain | Range |
|------|--------|-------|
| employeeId | Employee | xsd:string |
| name | Agent | xsd:string |
| hireDate | Employee | xsd:date |
| email | Employee | xsd:string |
| employmentType | Employee | xsd:string |
| departmentCode | Department | xsd:string |
| policyName | Policy | xsd:string |
| effectiveDate | Policy | xsd:date |
| description | Policy | xsd:string |
| benefitName | Benefit | xsd:string |

### 핵심 공리 (8개)

1. Employee SubClassOf belongsTo exactly 1 Department
2. Employee SubClassOf hasPosition exactly 1 Position
3. Employee SubClassOf employeeId exactly 1 xsd:string
4. FullTimeEmployee DisjointWith ContractEmployee
5. ManagerPosition DisjointWith StaffPosition
6. Department SubClassOf managedBy max 1 Employee
7. reportsTo: Asymmetric, Irreflexive
8. belongsTo: Functional

이것이 HR안내봇의 온톨로지 설계도다. 클래스 12개, 관계 15개, 공리 8개. 과하게 복잡하지도, 지나치게 단순하지도 않은, "충분히 좋은" 온톨로지다.

## Turtle 파일 — 최종 산출물

이 온톨로지의 핵심 부분을 Turtle 형식으로 정리하면 아래와 같다.

```turtle
@prefix hr: <http://example.org/hr-ontology#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

# --- Ontology Declaration ---
<http://example.org/hr-ontology> a owl:Ontology ;
    rdfs:label "HR Domain Ontology"@ko ;
    rdfs:comment "HR안내봇을 위한 인사 도메인 온톨로지"@ko .

# --- Classes ---
hr:Agent a owl:Class .
hr:Person a owl:Class ; rdfs:subClassOf hr:Agent .
hr:Employee a owl:Class ;
    rdfs:subClassOf hr:Person ;
    rdfs:subClassOf [
        a owl:Restriction ;
        owl:onProperty hr:belongsTo ;
        owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
        owl:onClass hr:Department
    ] .

hr:FullTimeEmployee a owl:Class ;
    rdfs:subClassOf hr:Employee ;
    owl:disjointWith hr:ContractEmployee .

hr:ContractEmployee a owl:Class ;
    rdfs:subClassOf hr:Employee .

hr:Department a owl:Class ;
    rdfs:subClassOf hr:OrganizationalUnit .

hr:Policy a owl:Class .
hr:LeavePolicy a owl:Class ; rdfs:subClassOf hr:Policy .
hr:WorkPolicy a owl:Class ; rdfs:subClassOf hr:Policy .
hr:WelfarePolicy a owl:Class ; rdfs:subClassOf hr:Policy .

hr:Benefit a owl:Class .
hr:HealthBenefit a owl:Class ; rdfs:subClassOf hr:Benefit .
hr:EducationBenefit a owl:Class ; rdfs:subClassOf hr:Benefit .

# --- Object Properties ---
hr:belongsTo a owl:ObjectProperty, owl:FunctionalProperty ;
    rdfs:domain hr:Employee ;
    rdfs:range hr:Department ;
    owl:inverseOf hr:hasMember .

hr:reportsTo a owl:ObjectProperty, owl:AsymmetricProperty, owl:IrreflexiveProperty ;
    rdfs:domain hr:Employee ;
    rdfs:range hr:Employee .

hr:affectsPolicy a owl:ObjectProperty ;
    rdfs:domain hr:Policy ;
    rdfs:range hr:Policy .

hr:appliesTo a owl:ObjectProperty ;
    rdfs:domain hr:Policy ;
    rdfs:range hr:Employee .

# --- Data Properties ---
hr:employeeId a owl:DatatypeProperty, owl:FunctionalProperty ;
    rdfs:domain hr:Employee ;
    rdfs:range xsd:string .

hr:name a owl:DatatypeProperty ;
    rdfs:domain hr:Agent ;
    rdfs:range xsd:string .

hr:hireDate a owl:DatatypeProperty ;
    rdfs:domain hr:Employee ;
    rdfs:range xsd:date .

hr:effectiveDate a owl:DatatypeProperty ;
    rdfs:domain hr:Policy ;
    rdfs:range xsd:date .
```

이 Turtle 파일을 `hr-ontology.ttl`로 저장한다. 5장에서 Neosemantics(n10s) 플러그인을 사용해 이 파일을 Neo4j에 임포트하고, 3장의 샘플 데이터를 이 스키마에 맞추어 마이그레이션할 것이다.

## 마무리

이번 장에서 우리는 온톨로지를 설계하는 두 가지 길을 모두 걸어보았다. Protege로 수동 설계하는 경로 A에서는 클래스 계층, 오브젝트 프로퍼티, 데이터 프로퍼티, 제약 조건을 직접 정의하며 온톨로지 설계의 전 과정을 체험했다. LLM으로 자동 생성하는 경로 B에서는 빠르게 초안을 잡고 전문가가 정제하는 협업 모델을 살펴보았다.

두 경로 모두 장단점이 있고, 실무에서는 대부분 하이브리드로 접근한다. "충분히 좋은 온톨로지가 완벽한 온톨로지보다 낫다"는 교훈을 기억해두자.

이제 설계도(온톨로지)가 완성되었다. 다음 할 일은 이 설계도를 Neo4j에 심고, 3장에서 입력한 샘플 데이터를 이 설계도에 맞추어 리모델링하는 것이다. 건물의 골조를 바꾸는 작업이니 긴장감이 있겠지만, 차근차근 함께 해보자.

---

**HR안내봇 진행도**

| 항목 | 상태 |
|------|------|
| 이번 장에서 한 것 | HR 온톨로지 완성 (클래스 12개, 관계 15개, 공리 8개) |
| 산출물 | Protege 수동 설계 결과 + LLM 자동 생성 결과 비교. OWL/Turtle 파일(`hr-ontology.ttl`) 내보내기 완료 |
| 경로 비교 | 수동 설계는 정교하지만 느림, LLM 자동 생성은 빠르지만 검증 필요. 하이브리드 추천 |
| 다음 장 예고 | 온톨로지를 Neo4j에 임포트하고, 3장 샘플 데이터를 온톨로지 스키마로 마이그레이션 |
