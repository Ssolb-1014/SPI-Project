# PRD: 그래프 구축 (C-01 ~ C-05)

## 1. 배경 및 목적

제조 기업의 비정형 문서(장비 매뉴얼, SOP, 이슈 보고서, 이력 기록)로부터 **지식 그래프를 자동 구축**하여, GraphRAG 기반 검색·질의의 핵심 데이터 레이어를 제공한다.

수집/전처리 파트(D-01~D-03)의 출력물인 **청크(Chunk)와 메타데이터**를 입력으로 받아,
그래프 탐색기·노드 상세 패널·온톨로지 관리 등의 관리 UI와 하이브리드 검색에서 소비 가능한 그래프 데이터를 생성한다.

### 1-1. 범위

**In-Scope**

- 온톨로지 스키마 관리 (C-01)
- LLM 기반 엔티티/관계 추출 (C-02)
- 엔티티 해소 — Entity Resolution (C-03)
- 커뮤니티 감지 (C-04)
- 요약/설명 생성 및 벡터화 (C-05)

**Out-of-Scope**

- 그래프 탐색기 / 노드 상세 패널 UI
- 원본 추적 UI
- 검색 엔진 (하이브리드/로컬/글로벌 검색)

---

## 2. 목표 및 성공 지표

| 지표 | 목표값 | 측정 방법 |
| --- | --- | --- |
| 엔티티 추출 Precision | ≥ 85% (MVP) | 수동 레이블 100건 대비 |
| 엔티티 추출 Recall | ≥ 75% (MVP) | 수동 레이블 100건 대비 |
| 엔티티 해소 정확도 | ≥ 90% | 동의어 쌍 50건 대비 |
| 그래프 구축 처리량 | 100페이지 문서 30분 이내 | 파이프라인 end-to-end |
| 요약 품질 (인간 평가) | 4.0/5.0 이상 | 제조 도메인 전문가 평가 |

---

## 3. 시스템 아키텍처

### 3-1 파이프라인 흐름

정책서 기준 문서 처리 파이프라인은 **파싱→청킹→OCR→추출→적재** 5단계로 구성된다.
본 PRD(그래프 구축)는 이 중 **추출→적재** 단계를 상세히 다룬다.

```
[전처리 출력]                 [C-01~C-05 그래프 구축 (추출→적재)]              [소비자]

청크(Chunks)   ──┐
메타데이터       ──┤
                ├──▶ C-01 온톨로지 스키마 로드
                │        │
                │        ▼
                ├──▶ C-02 엔티티/관계 추출 (LLM)
                │        │
                │        ▼
                ├──▶ C-03 실체 연결 (Resolution)
                │        │
                │        ├──▶ C-05 요약 및 설명 생성 + 벡터화
                │        │
                │        └──▶ C-04 커뮤니티 감지 (배치)
                │                  │
                │                  └──▶ C-05 커뮤니티 요약 생성
                │
                └──▶ Neo4j + Vector Store ──▶ 그래프 탐색기, 검색
```

> **파이프라인 모니터 연동**: 각 문서의 파이프라인 진행을 파싱→청킹→추출→적재 4단계 컬러 dot으로 시각화한다 — 완료(녹), 진행 중(파랑), 검수 대기(노랑), 실패(빨강).

### 3-2 기술 스택 (권장)

| 레이어 | 기술 | 선택 근거 |
| --- | --- | --- |
| 그래프 DB | Neo4j (Community/Enterprise) | Cypher 쿼리, APOC 플러그인, GDS 라이브러리 |
| 벡터 스토어 | Neo4j Vector Index 또는 별도 Qdrant/Weaviate | 요약 벡터 저장 |
| LLM | OpenAI GPT-4o / Claude Sonnet | 엔티티 추출, 요약 생성 |
| 임베딩 모델 | text-embedding-3-large 또는 multilingual-e5-large | 한국어 지원 필수 |
| 파이프라인 | Python (LangChain/LlamaIndex 또는 커스텀) |  |
| 커뮤니티 감지 | Neo4j GDS (Leiden / Louvain) | 그래프 네이티브 알고리즘 |

---

## 4. 데이터 모델

### 4-1 제조 도메인 온톨로지 스키마

#### 1) 엔티티 유형 (Node Labels)

| 엔티티 유형 | 설명 | 필수 속성 | 예시 |
| --- | --- | --- | --- |
| `Equipment` | 설비, 장비, 기계 | equipment_id, name, model, manufacturer, status | 인쇄기_01, CNC선반_A3 |
| `Process` | 공정, 작업 절차 | process_id, name, sequence_order | 인쇄공정, 도금공정, 열처리 |
| `Part` | 부품, 자재 | part_id, name, specification | 잉크카트리지, 볼트M8 |
| `Defect` | 불량, 결함 유형 | defect_id, name, severity | 인쇄불량, 치수초과, 크랙 |
| `Parameter` | 공정 파라미터 | param_id, name, unit, range_min, range_max | 온도(℃), 압력(bar), 속도(rpm) |
| `Document` | 원본 문서 | doc_id, title, doc_type, created_at | 장비매뉴얼_프린터_01 |
| `Person` | 담당자, 작성자 | person_id, name, department, role | 박QA팀장(품질팀) |
| `Issue` | 이슈, 사건 | issue_id, title, occurred_at, status | 인쇄 불량 발생 |
| `SOP` | 표준작업절차 | sop_id, title, version, effective_date | SOP_인쇄공정_v2.1 |
| `History` | 이력, 기록 | history_id, event_type, timestamp | 히스토리_001 (2025-09-12) |
| `Material` | 원자재 | material_id, name, grade | SUS304, AL6061 |
| `Location` | 공장, 라인, 구역 | location_id, name, type | A동_2층_인쇄라인 |


> **그래프 탐색기 필터**: 전체 타입 / 장비 / 공정 / 원자재 / 이슈 — 4개 필터 + 전체를 기본 제공한다. 규정·작업자는 전체 보기에서 표시된다.

#### 2) 관계 유형 (Relationship Types)

6개 엔티티 유형 간의 관계를 정의한다. 관계는 단방향(A→B) 또는 양방향(A↔B)을 지원한다.

| 관계 유형 | 소스 → 타겟 | 속성 | 설명 |
| --- | --- | --- | --- |
| `CONTAINS` | 공정 → 장비 | - | 공정에 장비가 포함됨 |
| `USES` | 공정 → 원자재 | quantity | 공정에서 부품/자재를 사용 |
| `CAUSES` | 이슈 → 이슈 | probability | 원인-결과 관계 (불량 간, 이슈 간) |
| `TRIGGERS` | 장비 → 이슈 | condition | 장비 파라미터 이탈이 불량 유발 |
| `REFERENCES` | 규정 → 규정 | ref_type | 문서 간 참조 |
| `DESCRIBES` | 규정 → 장비 | - | 문서/SOP가 장비를 설명 |
| `ASSIGNED_TO` | 이슈 → 작업자 | - | 이슈 담당자 |
| `RESOLVED_BY` | 이슈 → 규정 | - | SOP/규정으로 이슈 해결 |
| `PARENT_OF` | 이슈 → 이슈 | - | 상위 이슈에 속한 하위 이슈/이력 |
| `FOLLOWS` | 공정 → 공정 | - | 공정 순서 |
| `RECORDED_IN` | 이슈 → 규정 | - | 이슈가 문서에 기록됨 |
| `PERFORMED_BY` | 공정 → 작업자 | - | 공정/작업의 수행자 |
| `MADE_OF` | 원자재 → 원자재 | - | 부품의 재질 (볼트M8 → SUS304) |

#### 3) 동적 스키마 확장 규칙

- 온톨로지 관리 탭에서 엔티티 타입(색상, 이름, 설명)을 추가·편집·삭제 가능. 해당 타입에 속하는 노드가 1건 이상인 경우 삭제 시 컨펌 표시
- 관계 타입(이름, 방향, 설명)을 추가·편집 가능. 방향은 단방향(A→B)과 양방향(A↔B) 지원
- 새 유형 추가 시 필수 속성 최소 세트(`name`, `id`)를 강제
- 스키마 버전 관리: 변경 시 버전 increment, 이전 버전 그래프와의 호환성 유지
- 스키마 변경은 **"draft → publish"** 2단계로 운영

### 4-2. Neo4j 그래프 DB 스키마

#### 1) 노드 공통 속성

```
모든 노드 공통:
  - _id: String (UUID, 시스템 생성)
  - _label: String (엔티티 유형 — 장비/공정/원자재/이슈/규정/작업자)
  - _name: String (대표 명칭)
  - _aliases: List<String> (동의어, 약어 목록)
  - _source_doc_id: String (출처 문서 ID)
  - _source_chunk_ids: List<String> (출처 청크 ID 목록)
  - _description: String (자연어 설명, C-05에서 생성)
  - _description_embedding: List<Float> (설명 벡터, C-05에서 생성)
  - _community_id: String (소속 커뮤니티 ID, C-04에서 생성)
  - _confidence: Float (추출 신뢰도, 0.0~1.0)
  - _security_level: String (보안등급 — 일반/대외비/기밀/극비, 출처 문서에서 상속)
  - _created_at: DateTime
  - _updated_at: DateTime
  - _created_by: String (system / user_id)
  - _version: Integer
```

> **보안등급 상속**: 문서에 설정된 보안등급은 해당 문서에서 추출된 그래프 노드에 자동 상속된다. 기밀 이상 등급 문서의 노드는 L2 이하 등급의 검색 결과에서 완전 제외된다.

#### 2) 엣지 공통 속성

```
모든 관계 공통:
  - _id: String (UUID)
  - _type: String (관계 유형)
  - _weight: Float (관계 강도, 기본 1.0)
  - _source_chunk_ids: List<String> (근거 청크)
  - _description: String (관계 설명)
  - _confidence: Float (추출 신뢰도)
  - _created_at: DateTime
  - _created_by: String
```

#### 3) 인덱스 설계

```cypher
-- 고유성 제약조건
CREATE CONSTRAINT entity_id_unique FOR (n:Entity) REQUIRE n._id IS UNIQUE;

-- 검색용 인덱스
CREATE INDEX entity_name_idx FOR (n:Entity) ON (n._name);
CREATE INDEX entity_source_doc_idx FOR (n:Entity) ON (n._source_doc_id);

-- 전문 검색 인덱스
CREATE FULLTEXT INDEX entity_name_fulltext FOR (n:Entity) ON EACH [n._name, n._aliases];

-- 벡터 인덱스 (Neo4j 5.x+)
CREATE VECTOR INDEX entity_description_vector
FOR (n:Entity) ON (n._description_embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}};
```

#### 4) 커뮤니티 메타 노드

```
(:Community {
  community_id: String,
  level: Integer,        -- 계층 수준 (0=최하위)
  title: String,         -- 커뮤니티 제목
  summary: String,       -- 커뮤니티 요약
  summary_embedding: List<Float>,
  member_count: Integer,
  rank: Float
})
```

#### 5) 멀티테넌시 (도메인 격리)

- **전략**: Neo4j 데이터베이스 분리(Enterprise) 또는 `_domain_id` 속성 기반 필터링(Community)
- 도메인 관리 탭에서 워크스페이스 생성 시 그래프 격리 공간 자동 할당
- 모든 데이터(문서, 그래프, 검색 결과)는 현재 선택된 도메인 워크스페이스 범위 내에서만 조회·조작. 교차 도메인 접근은 RBAC 정책에 따름
- 도메인 전용 동의어 사전과 공통 사전을 모두 적용. 충돌 시 도메인 전용 사전 우선

---

## 5. 기능 상세 명세

### 5.1 C-01: 온톨로지 정의

#### 기능 설명

제조 도메인의 엔티티 유형(6개: 장비/공정/원자재/이슈/규정/작업자)과 관계 유형을 정의하고 관리하는 **스키마 레지스트리**. C-02 추출 시 LLM 프롬프트에 주입되어 추출 대상을 제한하고, 그래프 DB의 노드/엣지 Label로 매핑된다. 온톨로지 관리 탭에서 추출 현황 대시보드(추출 엔티티 수, 관계 수, 평균 신뢰도, 마지막 추출 시간, 처리 문서, 추출 모델)를 확인할 수 있다.

#### 상세 TASK

| Task ID | Task명 | 설명 | 산출물 |
| --- | --- | --- | --- | --- |
| C-01-T01 | 기본 온톨로지 데이터 모델 설계 | EntityType, RelationType 엔티티를 정의하는 DB 스키마 설계 (RDB 또는 Document Store) | ERD, DDL |
| C-01-T02 | 제조 도메인 시드 데이터 작성 | 4.1절의 엔티티/관계 유형을 초기 데이터로 작성 | JSON/YAML seed 파일 |
| C-01-T03 | 스키마 CRUD API 개발 | EntityType, RelationType에 대한 생성/조회/수정/삭제 REST API | API 엔드포인트 4개 |
| C-01-T04 | 스키마 버전 관리 구현 | draft/published 상태 관리, 버전 히스토리, 롤백 기능 | 버전 관리 로직 | 3일 |
| C-01-T05 | 스키마 유효성 검증 로직 | 순환 관계 방지, 필수 속성 검증, 명명 규칙 검증 | Validator 모듈 |
| C-01-T06 | 스키마→프롬프트 직렬화 | 온톨로지 스키마를 C-02 LLM 프롬프트에 주입할 수 있는 텍스트 포맷으로 변환 | Serializer 모듈 |
| C-01-T07 | 도메인별 스키마 격리 | 멀티테넌시: 도메인마다 독립 스키마, DS-01과 연동 | 도메인 필터링 로직 |
| C-01-T08 | 단위 테스트 및 통합 테스트 | 스키마 CRUD, 버전 관리, 직렬화 테스트 | 테스트 코드 |

#### 입출력 명세

- **입력**: 관리자의 스키마 정의 요청 (REST API)
- **출력**: 직렬화된 온톨로지 스키마 (C-02 프롬프트 주입용)

#### 스키마 시드 데이터 예시

```yaml
# ontology_schema_v1.yaml
version: 1
domain: "printing_factory"
entity_types:
  - name: Equipment
    display_name: "장비"
    description: "제조 설비 및 장비. 공정 파라미터를 하위 속성으로 포함."
    required_properties:
      - { name: equipment_id, type: string }
      - { name: name, type: string }
    optional_properties:
      - { name: model, type: string }
      - { name: manufacturer, type: string }
      - { name: status, type: enum, values: [active, inactive, maintenance] }
      - { name: location, type: string }
      - { name: parameters, type: json }
  - name: Issue
    display_name: "이슈"
    description: "이슈, 불량, 결함, 이력 기록"
    required_properties:
      - { name: issue_id, type: string }
      - { name: title, type: string }
    optional_properties:
      - { name: severity, type: enum, values: [critical, major, minor] }
      - { name: occurred_at, type: datetime }
      - { name: status, type: enum, values: [open, in_progress, resolved, closed] }
      - { name: event_type, type: string }

relationship_types:
  - name: CAUSES
    description: "원인-결과 관계"
    source_types: [Issue]
    target_types: [Issue]
    properties:
      - { name: probability, type: float, required: false }
    direction: unidirectional
  - name: TRIGGERS
    description: "장비 파라미터 이탈이 이슈 유발"
    source_types: [Equipment]
    target_types: [Issue]
    properties:
      - { name: condition, type: string, required: false }
    direction: unidirectional
```

---

### 5.2 C-02: 엔티티/관계 추출

#### 기능 설명

D-02(의미론적 청킹)의 출력인 텍스트 청크를 입력으로 받아, LLM을 통해 온톨로지 스키마에 정의된 엔티티와 관계를 추출한다. Microsoft GraphRAG의 추출 패턴을 참고하되 **제조 도메인에 최적화**한다.

#### 상세 TASK

| Task ID | Task명 | 설명 | 산출물 |
| --- | --- | --- | --- |
| C-02-T01 | 추출 파이프라인 아키텍처 설계 | 청크 입력 → 프롬프트 조합 → LLM 호출 → 파싱 → 저장 흐름 설계 | 아키텍처 다이어그램 |
| C-02-T02 | 엔티티 추출 프롬프트 설계 | 제조 도메인 컨텍스트 포함, 온톨로지 스키마 주입, Few-shot 예시 포함 | 프롬프트 템플릿 |
| C-02-T03 | 관계 추출 프롬프트 설계 | 추출된 엔티티 목록을 컨텍스트로 주입, 관계 유형 제한, 방향성 명시 | 프롬프트 템플릿 |
| C-02-T04 | LLM 응답 파서 구현 | JSON/구조화 출력 파싱, 스키마 미준수 응답 재시도, 폴백 처리 | Parser 모듈 |
| C-02-T05 | 글리닝(Gleaning) 구현 | 동일 청크에 대해 2회차 추출 수행: "누락된 엔티티가 있는지 다시 확인" | Gleaning 로직 |
| C-02-T06 | 청크-엔티티 출처 매핑 | 추출된 엔티티/관계에 source_chunk_id를 태깅 (M-03 원본 추적의 기반) | Provenance 모듈 |
| C-02-T07 | Neo4j 적재 모듈 | 추출 결과를 Neo4j에 MERGE 기반으로 적재 (중복 방지) | Loader 모듈 |
| C-02-T08 | 배치 처리 및 병렬화 | 다수 청크의 비동기 병렬 LLM 호출, rate limiting, 재시도 | Batch Processor |
| C-02-T09 | 신뢰도 점수 산출 | LLM 응답 확신도 또는 반복 추출 일치율 기반 confidence 계산 | Scoring 모듈 |
| C-02-T10 | 추출 품질 평가 도구 | 수동 레이블 데이터 대비 Precision/Recall 계산 스크립트 | 평가 스크립트 |
| C-02-T11 | 단위/통합 테스트 | 프롬프트 응답 파싱, Neo4j 적재, 배치 처리 테스트 | 테스트 코드 |

#### 처리 흐름 상세

```
1. 청크 큐에서 미처리 청크를 배치 단위(N=10)로 가져옴
2. 각 청크에 대해:
   a. C-01에서 현재 published 온톨로지 스키마 로드
   b. 엔티티 추출 프롬프트 조합 (시스템 프롬프트 + 스키마 + 청크 텍스트)
   c. LLM 호출 → 구조화된 JSON 응답 수신
   d. 응답 파싱 및 유효성 검증
   e. (선택) Gleaning: 누락 확인 2차 프롬프트
   f. 관계 추출 프롬프트 조합 (추출된 엔티티 목록 포함)
   g. LLM 호출 → 관계 JSON 응답
   h. 신뢰도 점수 부여
   i. 출처(source_chunk_id) 태깅
3. 추출 결과를 Neo4j에 MERGE로 적재
4. 처리 상태를 파이프라인 상태 관리에 보고 (파싱→청킹→추출→적재 단계 진행 상태)
```

#### 입출력 명세

**입력**:

```json
{
  "chunk_id": "uuid",
  "doc_id": "uuid",
  "chunk_index": 0,
  "text": "청크 텍스트 내용",
  "token_count": 512,
  "metadata": {
    "doc_type": "equipment_manual",
    "created_at": "2024-01-15",
    "author": "김OO",
    "department": "품질관리팀"
  }
}
```

**출력**:

```json
{
  "entities": [
    {
      "entity_id": "uuid",
      "type": "Equipment",
      "name": "인쇄기_01",
      "properties": { "model": "HP LaserJet Pro", "parameters": {"이송속도": {"unit": "mm/min", "value": "150"}} },
      "confidence": 0.95,
      "source_chunk_ids": ["chunk-uuid-1"]
    },
    {
      "entity_id": "uuid",
      "type": "Issue",
      "name": "치수 초과 불량",
      "properties": { "severity": "major", "event_type": "defect" },
      "confidence": 0.85,
      "source_chunk_ids": ["chunk-uuid-1"]
    }
  ],
  "relationships": [
    {
      "rel_id": "uuid",
      "type": "TRIGGERS",
      "source_entity_id": "entity-uuid-1",
      "target_entity_id": "entity-uuid-2",
      "properties": { "condition": "이송속도 150mm/min 설정" },
      "confidence": 0.88,
      "source_chunk_ids": ["chunk-uuid-1"]
    }
  ]
}
```

---

### 5.3 C-03: 실체 연결 (Entity Resolution)

#### 기능 설명

서로 다른 청크 또는 문서에서 추출된 **동일 대상을 하나의 노드로 통합**한다. 제조 현장에서는 같은 장비를 "프린터_01", "PT-01", "1번 인쇄기", "Printer Unit 1" 등으로 부르는 경우가 빈번하며, 이를 해소하지 않으면 그래프가 분절된다.

#### 상세 TASK

| Task ID | Task명 | 설명 | 산출물 |
| --- | --- | --- | --- |
| C-03-T01 | 후보 쌍 생성 (Blocking) | 동일 유형 노드 간 이름 유사도 기반 후보 쌍 생성. 전수 비교 방지를 위한 블로킹 전략 (LSH, 접두사) | Blocker 모듈 |
| C-03-T02 | 규칙 기반 매칭 | 정규화(대소문자, 공백, 특수문자), 약어 확장, 코드명 패턴 매칭 | Rule Matcher |
| C-03-T03 | 임베딩 기반 매칭 | 엔티티명 + 컨텍스트의 임베딩 유사도 비교, 임계값 설정 | Embedding Matcher |
| C-03-T04 | LLM 기반 매칭 (선택) | 규칙/임베딩으로 판단 불가한 경계 사례에 대해 LLM으로 최종 판단 | LLM Matcher |
| C-03-T05 | 노드 병합 로직 | 병합 시 속성 통합 전략(우선순위 규칙), 엣지 재연결, alias 목록 갱신 | Merger 모듈 |
| C-03-T06 | 동의어 사전 연동 | 사전/매핑 탭에서 등록된 동의어 사전을 매칭에 활용. 대표어와 동의어를 동일 엔티티로 취급 | Dictionary Loader |
| C-03-T07 | 병합 이력 기록 | 어떤 노드가 어떤 노드로 병합되었는지 추적. 변경 이력 탭에 "실체 병합" 이벤트로 기록 | Audit Log |
| C-03-T08 | 수동 병합/분리 API | 관리자가 노드 상세 패널에서 수동으로 노드 병합/분리 시 호출하는 API | API 엔드포인트 |
| C-03-T09 | 테스트 | 매칭 정확도 평가, 병합 무결성 테스트 | 테스트 코드 |

#### 매칭 전략 우선순위

```
1단계: 정확 매칭 (정규화 후 동일 문자열) ──▶ 자동 병합
2단계: 동의어 사전 매칭 (사전/매핑 탭에 등록된 매핑) ──▶ 자동 병합
3단계: 규칙 기반 (약어 패턴, 코드명 패턴) ──▶ confidence ≥ 0.9 → 자동, 미만 → 후보 제시
4단계: 임베딩 유사도 (cosine ≥ 0.92) ──▶ 후보 제시, 관리자 확인
5단계: LLM 판단 (경계 사례) ──▶ 후보 제시, 관리자 확인
```

#### 제조 도메인 특수 패턴

| 패턴 | 예시 | 처리 방법 |
| --- | --- | --- |
| 장비 코드 ↔ 풀네임 | PT-01 ↔ 프린터_01 | 코드명 매핑 테이블 |
| 한영 혼용 | 인쇄기 ↔ Printer | 번역 사전 |
| 약어 | SOP ↔ 표준작업절차서 | 약어 사전 |
| 모델번호 포함/미포함 | HP LaserJet Pro ↔ 레이저젯 | 임베딩 유사도 |
| 위치 접미어 | A동_프린터 ↔ 프린터(A동) | 정규화 규칙 |

#### 오류 플래그 연동 (정책서 반영)

그래프 무결성 오류를 감지하여 노드 상세 패널에 표시한다. 각 오류에 **수정 / 연결 / 삭제 / 무시** 버튼을 제공한다. "무시" 선택 시 해당 오류를 목록에서 숨긴다.

- 병합 시 엣지 방향 무결성 검증 (예: 이슈 간 시간 순서 역전 감지)
- 오류 감지 시 `error_flag` 속성에 기록
- 오류 유형:
  - `circular_reference` — 순환 참조 (A→B→C→A)
  - `direction_error` — 연결 방향 오류
  - `duplicate_edge` — 중복 엣지
  - `orphan_node` — 고립 노드 (연결된 엣지가 없는 노드)

---

### 5.4 C-04: 커뮤니티 감지


#### 기능 설명

그래프 내에서 밀접하게 연결된 노드 그룹(커뮤니티)을 자동으로 감지한다. 커뮤니티는 "인쇄 공정 관련 클러스터", "품질 불량 원인 클러스터" 등 상위 주제를 형성하며, **글로벌 검색**에서 전체 맥락 파악에 활용된다.

#### 상세 TASK

| Task ID | Task명 | 설명 | 산출물 |
| --- | --- | --- | --- |
| C-04-T01 | Neo4j GDS 프로젝션 설정 | 커뮤니티 감지 대상 노드/엣지의 서브그래프를 GDS 프로젝션으로 생성 | GDS 설정 스크립트 |
| C-04-T02 | Leiden 알고리즘 적용 | Neo4j GDS의 Leiden 알고리즘 실행, 해상도(resolution) 파라미터 튜닝 | 커뮤니티 감지 모듈 |
| C-04-T03 | 계층적 커뮤니티 구축 | 여러 해상도 수준에서 커뮤니티를 감지하여 계층 구조(Level 0~N) 생성 | Hierarchical 모듈 |
| C-04-T04 | 커뮤니티 메타 노드 생성 | Community 노드를 생성하고 BELONGS_TO 관계로 멤버 노드 연결 | Community 적재 로직 |
| C-04-T05 | 커뮤니티 요약 생성 | 각 커뮤니티의 멤버 노드/엣지 정보를 기반으로 C-05를 호출하여 요약 생성 | C-05 연동 로직 |
| C-04-T06 | 스케줄링 및 재실행 | 그래프 변경 시 커뮤니티 재감지 트리거, 배치 스케줄러 | Scheduler |
| C-04-T07 | 테스트 | 커뮤니티 품질 평가 (Modularity 점수), 계층 구조 무결성 | 테스트 코드 |

#### 파라미터 가이드

| 파라미터 | 설명 | 권장 초기값 | 비고 |
| --- | --- | --- | --- |
| resolution | 커뮤니티 세분화 정도 | 1.0 (기본) | 높을수록 작은 커뮤니티 |
| max_levels | 계층 최대 깊이 | 3 | 제조 도메인: 공장 > 공정 > 세부 |
| min_community_size | 최소 커뮤니티 크기 | 3 | 노드 3개 미만은 병합 |

---

### 5.5 C-05: 요약 및 설명 생성

#### 기능 설명

그래프의 모든 노드, 엣지, 커뮤니티에 대해 **자연어 설명(description)**을 LLM으로 생성하고, 이를 **임베딩하여 벡터 인덱스에 저장**한다. 하이브리드 검색에서 벡터 유사도 검색의 대상이 된다.

#### 상세 TASK

| Task ID | Task명 | 설명 | 산출물 |
| --- | --- | --- | --- |
| C-05-T01 | 노드 요약 프롬프트 설계 | 노드의 타입, 속성, 연결된 엣지 정보를 컨텍스트로 자연어 설명 생성 | 프롬프트 템플릿 |
| C-05-T02 | 엣지 요약 프롬프트 설계 | 소스/타겟 노드 정보와 관계 속성을 컨텍스트로 관계 설명 생성 | 프롬프트 템플릿 |
| C-05-T03 | 커뮤니티 요약 프롬프트 설계 | 커뮤니티 멤버 노드/엣지의 설명을 종합하여 상위 주제 요약 생성 | 프롬프트 템플릿 |
| C-05-T04 | 요약 생성 파이프라인 | 노드 → 엣지 → 커뮤니티 순서로 요약 생성 (의존 관계 준수) | Pipeline 모듈 |
| C-05-T05 | 임베딩 생성 및 저장 | 생성된 요약 텍스트를 임베딩 모델로 벡터화, Neo4j 벡터 인덱스에 저장 | Embedder 모듈 |
| C-05-T06 | 증분 업데이트 | 노드/엣지 변경 시 해당 요약만 재생성 (전체 재생성 방지) | Incremental 로직 |
| C-05-T07 | 배치 처리 및 비용 최적화 | 대량 노드에 대한 배치 LLM 호출, 토큰 사용량 추적 (운영 모니터링 KPI 연계) | Batch + Tracking |
| C-05-T08 | 테스트 | 요약 품질 평가 (인간 평가 샘플링), 벡터 검색 정확도 | 테스트 코드 |

#### 요약 품질 기준

| 기준 | 설명 | 검증 방법 |
| --- | --- | --- |
| 정확성 | 요약 내용이 그래프 데이터와 일치 | 사실 관계 대조 |
| 완전성 | 주요 속성과 관계가 누락 없이 포함 | 속성 커버리지 체크 |
| 간결성 | 불필요한 반복 없이 핵심만 | 토큰 수 상한 (노드: 100토큰, 커뮤니티: 300토큰) |
| 검색 적합성 | 관련 질의에 벡터 유사도가 높게 나오는지 | 검색 히트율 테스트 |

---

## 6. LLM 프롬프트 설계 가이드라인

### 6-1. 공통 원칙

| 원칙 | 설명 |
| --- | --- |
| 구조화 출력 강제 | 모든 LLM 응답은 JSON 형식으로 요청. function calling 또는 structured output 활용 |
| 온톨로지 스키마 주입 | 추출/요약 시 C-01의 스키마를 시스템 프롬프트에 포함하여 할루시네이션 방지 |
| 한국어 우선 | 제조 현장 용어는 한국어 기준, 영문 병기 허용 |
| Few-shot 예시 | 제조 도메인 실제 문서에서 추출한 예시 3~5개를 포함 |
| 청크 단위 처리 | 하나의 프롬프트에 하나의 청크만 처리 (컨텍스트 오염 방지) |
| Idempotency | 동일 입력에 대해 가능한 동일 출력을 위해 temperature=0 또는 낮은 값 사용 |

### 6-2. 엔티티 추출 프롬프트 템플릿

```
[System Prompt]
당신은 제조 도메인 전문 지식 그래프 엔지니어입니다.
주어진 텍스트에서 아래 온톨로지 스키마에 정의된 엔티티를 추출하세요.

## 온톨로지 스키마 (허용된 엔티티 유형)
{C-01에서 직렬화된 스키마 주입}

## 추출 규칙
1. 텍스트에 명시적으로 언급된 엔티티만 추출합니다.
2. 추론이나 상식에 의한 엔티티는 추출하지 않습니다.
3. 각 엔티티에 대해 유형(type), 이름(name), 속성(properties)을 반환합니다.
4. 동일 엔티티가 여러 번 언급되면 한 번만 추출합니다.
5. 장비 코드, 모델명, 약어도 name에 포함합니다.
6. 확신이 낮은 경우에도 추출하되, confidence를 낮게 설정합니다.

## 출력 형식
{
  "entities": [
    {
      "name": "엔티티 이름",
      "type": "EntityType (스키마에 정의된 유형)",
      "properties": {"key": "value"},
      "confidence": 0.0~1.0
    }
  ]
}

## 예시
입력: "CNC선반 A3(모델: FANUC α-DiA)의 이송속도를 150mm/min으로 설정했으나
      치수 초과 불량이 발생했다."
출력:
{
  "entities": [
    {"name": "CNC선반 A3", "type": "Equipment",
     "properties": {"model": "FANUC α-DiA", "parameters": {"이송속도": {"unit": "mm/min", "value": "150"}}}, "confidence": 0.95},
    {"name": "치수 초과 불량", "type": "Issue",
     "properties": {"severity": "major", "event_type": "defect"}, "confidence": 0.85}
  ]
}

[User Prompt]
다음 텍스트에서 엔티티를 추출하세요:
---
{청크 텍스트}
---
```

### 6-3. 관계 추출 프롬프트 템플릿

```
[System Prompt]
당신은 제조 도메인 전문 지식 그래프 엔지니어입니다.
주어진 텍스트와 이미 추출된 엔티티 목록을 바탕으로, 엔티티 간의 관계를 추출하세요.

## 허용된 관계 유형
{C-01에서 직렬화된 관계 스키마 주입}

## 추출 규칙
1. 반드시 아래 엔티티 목록에 있는 엔티티 간의 관계만 추출합니다.
2. 관계의 방향(source → target)을 정확히 지정합니다.
3. 텍스트에 명시적으로 드러나거나 강하게 암시된 관계만 추출합니다.
4. 관계 유형은 스키마에 정의된 것만 사용합니다.

## 추출된 엔티티 목록
{C-02 엔티티 추출 결과 주입}

## 출력 형식
{
  "relationships": [
    {
      "source": "소스 엔티티 name",
      "target": "타겟 엔티티 name",
      "type": "RelationType",
      "description": "관계에 대한 한 줄 설명",
      "properties": {"key": "value"},
      "confidence": 0.0~1.0
    }
  ]
}

[User Prompt]
다음 텍스트에서 엔티티 간 관계를 추출하세요:
---
{청크 텍스트}
---
```

### 6-4. Gleaning (2차 추출) 프롬프트

```
[User Prompt]
이전 추출 결과를 검토합니다.
아래는 동일 텍스트에서 이미 추출된 엔티티/관계입니다:

{1차 추출 결과}

누락된 엔티티나 관계가 있다면 추가로 추출하세요.
없다면 빈 리스트를 반환하세요.
---
{청크 텍스트}
---
```

### 6-5. 요약 생성 프롬프트 (노드)

```
[System Prompt]
주어진 그래프 노드 정보를 바탕으로, 이 엔티티에 대한 간결하고 정확한
자연어 설명을 제조 도메인 전문가가 이해할 수 있는 수준으로 생성하세요.
- 100토큰 이내로 작성합니다.
- 엔티티의 유형, 주요 속성, 핵심 관계를 포함합니다.
- 검색에 활용되므로 관련 키워드를 자연스럽게 포함합니다.

[User Prompt]
노드 정보:
- 이름: {name}
- 유형: {type}
- 속성: {properties}
- 연결된 관계:
  {edge_list: [{type, direction, connected_node_name}]}

이 엔티티에 대한 설명을 생성하세요.
```

### 6-6. 프롬프트 관리 지침

| 항목 | 지침 |
| --- | --- |
| 버전 관리 | 모든 프롬프트 템플릿은 Git 관리, 변경 시 평가 수행 |
| 평가 데이터셋 | 제조 문서 최소 20건(다양한 문서 유형 포함)으로 구성 |
| A/B 테스트 | 프롬프트 변경 시 A/B 비교 테스트(에이전트 좌우 배치, 동시 전송, 성능 지표 비교)와 연계하여 품질 비교 |
| 토큰 예산 | 엔티티 추출: ~2000 input + ~500 output / 청크 |
| 엔티티 유형 제한 | 6개 타입(장비/공정/원자재/이슈/규정/작업자)만 추출 대상으로 제한. 온톨로지 관리 탭에서 타입 추가 시 프롬프트 자동 갱신 |
| 모델 폴백 | GPT-4o 실패 시 Claude Sonnet으로 폴백, 응답 형식 호환 보장 |
| Temperature | 추출: 0.0, 요약: 0.3 |

---

## 7. API 명세

### 7-1. 내부 파이프라인 인터페이스

| API | 메서드 | 설명 | 선행 | 후행 |
| --- | --- | --- | --- | --- |
| `POST /api/v1/ontology/schema` | REST | 온톨로지 스키마 CRUD | - | C-02 |
| `GET /api/v1/ontology/schema/{domain_id}/published` | REST | 발행된 스키마 조회 | C-01 | C-02 |
| `POST /api/v1/graph/extract` | REST/Queue | 청크 배치 추출 요청 | D-02, C-01 | C-03 |
| `POST /api/v1/graph/resolve` | REST/Queue | 엔티티 해소 실행 | C-02 | C-05 |
| `POST /api/v1/graph/detect-communities` | REST/Queue | 커뮤니티 감지 실행 | C-03 | C-05 |
| `POST /api/v1/graph/generate-summaries` | REST/Queue | 요약 생성 및 벡터화 | C-03 | R-01 |
| `GET /api/v1/graph/nodes` | REST | 노드 목록 조회 (필터/페이징) | - | M-01, M-02 |
| `GET /api/v1/graph/nodes/{id}/provenance` | REST | 노드의 출처 청크 추적 | C-02 | M-03 |

### 7-2. 선행 파트로부터의 입력 계약

D-02 의미론적 청킹 출력 → 5.2절 입출력 명세의 "입력" 참조

---

## 8. 비기능 요구사항

| 항목 | 요구사항 | 비고 |
| --- | --- | --- |
| 성능 | 100페이지 문서 그래프 구축: 30분 이내 | LLM API 병렬 호출 |
| 확장성 | 10,000 노드, 50,000 엣지 규모까지 MVP 지원 | Neo4j 인덱스 최적화 |
| 재현성 | 동일 입력 → 동일 그래프 (temperature=0) | 결정적 파이프라인 |
| 추적성 | 모든 노드/엣지에 출처 청크 ID 기록 | 원본 추적 기반 |
| 보안 | 노드에 출처 문서의 보안등급(_security_level) 자동 상속 | RBAC 정책 연계 |
| 비용 | 문서 1건당 LLM 토큰 비용 $2 이내 | 운영 모니터링 KPI 연계 |
| 에러 처리 | LLM 호출 실패 시 3회 재시도 후 큐에 재적재 | 파이프라인 상태 관리 |
| 로깅 | 각 단계별 처리 시간, 토큰 사용량, 에러 로그 | 구조화된 로깅 |

---

## 9. 의존성 및 연동

### 9-1. 선행 의존성 (Upstream)

| 의존 대상 | 필요 시점 | 필요 항목 | 대안(Mock) |
| --- | --- | --- | --- |
| D-01 멀티 포맷 파싱 | C-02 시작 전 | 파싱된 텍스트 | 샘플 텍스트 파일 |
| D-02 의미론적 청킹 | C-02 시작 전 | 청크 데이터 | 고정 길이 청킹 폴백 |
| D-03 메타데이터 추출 | C-02 시작 전 | 문서 메타데이터 | 수동 메타데이터 |

### 9-2. 후행 소비자 (Downstream)

| 소비자 | 필요 항목 | 제공 시점 |
| --- | --- | --- |
| 그래프 탐색기 탭 | 노드/엣지 데이터 (Neo4j). 6개 타입별 색상 렌더링, 드래그/줌/패닝, 2D/3D 모드 전환 | C-02 완료 후 |
| 노드 상세 패널 | 노드/엣지 CRUD API. 속성 편집, 엣지 관리, 오류 플래그 표시 | C-02 완료 후 |
| 원본 추적 | source_chunk_ids 매핑. 노드 클릭 시 출처 문서/페이지/청크 표시 | C-02 완료 후 |
| 사전/매핑 탭 | 동의어 사전 ↔ C-03 양방향. 불용어 목록, 엔티티 매핑 규칙(정규식) 참조 | C-03 완료 후 |
| 변경 이력 탭 | 노드 추가/수정/삭제, 엣지 추가/삭제, 실체 병합 이벤트 기록. 롤백·스냅샷 지원 | 모든 C-0x |
| 추출 현황 대시보드 | 추출 엔티티 수, 관계 수, 평균 신뢰도, 마지막 추출 시간, 처리 문서(완료/전체), 추출 모델 | C-02 완료 후 |
| 하이브리드 검색 | 벡터 인덱스 + 그래프 구조 | C-05 완료 후 |
| 글로벌 검색 | 커뮤니티 요약 벡터 | C-04 + C-05 완료 후 |

### 9-3. 기능 간 내부 의존성

```
C-01 (온톨로지 정의) ──────────────────────────┐
     │                                     │
     ▼                                     │
C-02 (엔티티/관계 추출) ── 스키마 참조 ───────────┘
     │
     ▼
C-03 (실체 연결) ── C-02 결과 필요
     │
     ├──▶ C-05 (요약 생성) ── C-03 이후 정제된 그래프 대상
     │
     └──▶ C-04 (커뮤니티 감지) ── C-03 이후 통합된 그래프 대상
              │
              └──▶ C-05 (커뮤니티 요약) ── C-04 결과로 커뮤니티 요약 추가 생성
```

---

## 10. 리스크 및 완화 방안

| 리스크 | 영향도 | 발생 확률 | 완화 방안 |
| --- | --- | --- | --- |
| LLM 추출 품질 부족 | 높음 | 중 | Gleaning, Few-shot 튜닝, 평가 데이터셋 확보 |
| 한국어 제조 용어 인식 실패 | 높음 | 중 | 제조 용어 사전 구축, 프롬프트에 도메인 용어집 주입 |
| LLM API 비용 초과 | 중 | 중 | 운영 모니터링 KPI 카드(일일 토큰 사용량/월간 비용) 활용, sLLM 검토, 캐싱 |
| Neo4j 대규모 그래프 성능 | 중 | 낮 | 인덱스 최적화, 파티셔닝, 쿼리 프로파일링 |
| 엔티티 해소 오류 (과잉/과소 병합) | 높음 | 중 | 단계적 자동화(자동→후보 제시→수동), 노드 상세 패널의 속성 편집/엣지 관리 기능 활용 |
| D-01~D-03 지연으로 C-02 시작 불가 | 중 | 중 | Mock 데이터로 선행 개발, 인터페이스 계약 선확정 |

---

## 용어집

| 용어 | 정의 |
| --- | --- |
| 온톨로지 | 엔티티 유형과 관계 유형을 정의하는 도메인 스키마 |
| Gleaning | 1차 추출 후 누락 확인을 위해 2차 추출을 수행하는 기법 |
| Entity Resolution | 동일 대상을 지칭하는 여러 표현을 하나로 통합하는 과정 |
| Community Detection | 밀접하게 연결된 노드 그룹을 자동 식별하는 그래프 알고리즘 |
| Leiden Algorithm | 커뮤니티 감지 알고리즘 (Louvain의 개선 버전) |
| GraphRAG | 그래프 구조를 활용한 검색 증강 생성 |
| MERGE (Cypher) | Neo4j에서 기존 데이터 있으면 매칭, 없으면 생성하는 쿼리 |
| Blocking | 전수 비교 방지를 위해 후보 쌍을 제한하는 전처리 기법 |
