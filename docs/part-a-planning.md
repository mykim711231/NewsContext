# Part A 기획·설계: LLM 기반 온톨로지와 지식그래프

상태: 구현 전 설계. 이 문서는 전문 에이전트의 기획·방법론·아키텍처 검토를 반영하는 문서이며, 실행 앱의 전문가 검색·팀 편성 시스템을 정의하지 않습니다.

## 1. 3단계 책임과 목표

| 단계 | 책임 | 산출물 |
|---|---|---|
| Crawler | 경제뉴스 수집·파싱·정규화·불변 원문 버전 저장 | 기사·원문 버전·시간·수집 이력 |
| Part A | LLM 온톨로지 설계와 KG 추출·검증·정합화·적재 | 승인 온톨로지·분석 계약·검증된 KG와 manifest |
| Part B | Part A 자료로 근거 탐색·분석·심층 리서치 보고서 작성 | 문장별 출처가 연결된 보고서·Markdown |

Part A는 Part B가 보고서를 작성할 수 있는 지식 기반을 만드는 단계입니다. Part B는 KG를 새로 추출·적재하거나 운영 온톨로지를 자동 수정하지 않습니다. 부족한 자료는 Part A 보완 요청으로 반환하고, 승인된 새 manifest로 새 보고서 작업을 시작합니다.

## 2. 요구사항과 업무 질문

경제뉴스만으로 이슈·기업·기관·업종·사건·기사에 보도된 반응을 표현하고 가격 원천은 수집하지 않습니다. 임베딩은 스키마·동일 대상 후보 검색에 조건부 도입하며 pgvector와 프로그램 검증을 사용합니다. [검색 설계](embedding-retrieval.md). 초기 업무 질문은 다음과 같습니다.

- 어떤 이슈가 어떤 기업·업종과 언제 연결되어 보도되었는가?
- 기사에 관측된 움직임·전망·해석이 어떻게 표현되었는가?
- 사건 전후 어떤 후속 보도·상충 주장이 존재하는가?
- 기사에서 누가 무엇을 원인·배경으로 설명하는가?
- 어떤 원문 구간이 각 주장을 지지하며 무엇을 확인할 수 없는가?

시간·맥락·보도된 인과 설명을 우선 지원하고 규칙 도출·관계 가설은 후속 기능으로 구분합니다. 실제 개입 효과 추정은 뉴스만으로 지원하지 않습니다.

## 3. Part A 내부 흐름

구현 참조는 [논문·GitHub 계획](reference-implementation-plan.md)의 A01~A08이며 실제 파일·license 확인은 [소스 감사](source-audit.md)를 따릅니다. LLMs4OL의 타입/계층/관계 분해, EDC의 Extract/Define/Canonicalize, CQ4OE·Text2KGBench의 분리 평가를 기본으로 둡니다. Multi-Agent·AutoSchemaKG·KGGen·OntoExtend는 후보 생성·역할 분리·정합화·기존 스키마 확장의 비교 실험입니다. OWL/SHACL 원형과 우리 property graph 계약을 동일하게 취급하지 않습니다.

### A1. 온톨로지·분석 계약 설계

`define_questions → select_samples → propose_ontology → normalize_schema → validate_schema → review_ontology → approve_contract`

LLM은 표본·업무 질문으로 타입·관계·속성·제약 후보를 생성합니다. 프로그램은 타입·참조·관계 domain/range·단위·필수 근거·질문 표현 가능성을 검증하고 관리자가 승인합니다. 승인된 기존 스키마를 재사용할 수 있으며 매 기사마다 스키마를 재설계하지 않습니다.

`ontology_version`은 데이터 모델 버전, `analysis_contract_version`은 질문 범위·근거 정책·보고서 절·허용 도구·예산·평가를 포함하는 버전입니다. 승인 시 두 버전과 내용 해시를 고정합니다. 새 후보와 운영 승인을 구분합니다.

### A2. 지식 추출·검증

`load_article_version → load_approved_contract → filter_topic → split_article → extract_candidates → validate_candidates`

고정 구조 추출에는 LangChain 모델의 구조화 출력을 우선 사용합니다. Pydantic 형식 검증과 실제 근거·단위·시간 의미 검사는 별개입니다. 추출 결과는 reported/derived/hypothesis, 관측 보도/전망/해석, 긍정/부정/불확실성을 보존합니다. 알 수 없는 값은 null과 unresolved로 남깁니다.

문단의 offset을 원문 전체 offset으로 변환하고 특정 normalization_version의 `[start,end)`와 인용이 일치하는지 검사합니다. 기사 지시문을 시스템 명령으로 실행하지 않습니다. 근거 없는 후보는 KG 확정 결과에 넣지 않습니다.

### A3. 정합화·검토·적재

선택 임베딩이 활성화되면 기존 별칭/키워드와 관계 정의 vector top-k로 후보를 찾고 EDC식 LLM 선택/거절 및 시간·domain/range·근거 검증을 수행합니다. AutoSchemaKG/KGGen의 병합 결과를 그대로 확정하지 않습니다. 후보 집합이 작은 경우 메모리 검색과 pgvector를 비교하고 전체 모델·FAISS stack을 무조건 도입하지 않습니다.

`load_aliases → canonicalize_entities → resolve_event_candidates → revalidate_references → review_if_needed → build_graph_batch → write_graph_batch`

별칭·기간·참여자·수치를 비교하되 동명 대상과 모호한 사건은 보류합니다. 문서별 Claim과 공통 Event는 분리하고 다른 기사에 같은 문장이 있다는 이유만으로 독립 검증 사실로 승격하지 않습니다. 기사에 귀속된 원인 설명은 REPORTED_CAUSE로 표현합니다.

로컬 PostgreSQL은 원문·검증 결과·적재 입력의 정본이며 지식그래프도 같은 DB의 그래프 테이블(노드·관계·snapshot membership)에 둡니다([README 그래프 저장 방침](../README.md), 2026-11-17). 논리 참조·소속·삭제 제한은 프로그램이 검사하고 FK를 생성하지 않습니다. 그래프 쓰기는 고유 ID·멱등 키로 제어하고 업무 상태와 같은 트랜잭션에 묶습니다. checkpoint와 업무 쓰기의 자동 원자성은 가정하지 않습니다.

### A4. Part B 인계 자료 확정

`verify_graph_batch → freeze_kg_manifest → validate_report_readiness → publish_kg_snapshot`

인계는 `analysis_contract_version`, `ontology_version`, `kg_snapshot_id`와 corpus manifest·적재 버전·근거 상태를 포함합니다. snapshot은 그래프 DB의 물리 스냅샷이 아니라 **보고서 입력을 고정하는 논리 manifest**입니다. 각 구성요소의 ID·해시·버전을 PostgreSQL에 보관하며 조회는 snapshot의 허용 범위에 한정합니다. 입력을 재현할 수 없는 최신 속성 덮어쓰기는 피합니다.

상태는 building/validating/ready/withdrawn/failed로 두고 ready 자료만 Part B가 사용합니다. 공개 보고서가 참조한 원문·Claim·근거는 다른 프로젝트의 정정 작업으로 조용히 삭제되지 않도록 합니다. 정정 시 새 snapshot과 보고서 버전을 생성하고 기존 결과의 공개 유효성은 별도 검토합니다.

## 4. 최소 모델과 저장 계약

| 데이터 | 역할 |
|---|---|
| ArticleVersion·Document | 불변 원문·해시·발행/수집/사건 시각의 근거 |
| Entity·Event·Issue | 대상·사건·이슈의 정규 식별 |
| Claim·Evidence | 원문에 귀속된 주장·필드별 인용·양태·상태 |
| AnalysisContract·OntologyVersion | 추출·분석·보고서 작성 계약 |
| KGSnapshot·Manifest | 보고서에 사용 가능한 KG·원문·검증 결과 집합 |

PostgreSQL에는 analysis_contract_versions/reviews, ontology_versions/reviews, processing_jobs, extraction_results, kg_snapshots, kg_snapshot_items를 저장합니다. 참조 컬럼에는 일반/고유 인덱스를 두고 PK·프로젝트 소속·버전 존재를 공통 함수에서 검사합니다. Agent 역할 카탈로그나 ResearchPlan 저장소는 도입하지 않습니다.

## 5. LangGraph와 Agent 경계

A1 후보 생성·A2 구조화 추출은 LLM을 사용하는 단계이며 자동으로 자율 Agent라 부르지 않습니다. 등록된 도구 중 추가 표본·개체 조회나 근거 확인을 모델이 선택할 때에만 제한된 Agent로 정의합니다. 함수 파이프라인과 StateGraph가 순서·조건 분기·예산·검토 대기·재개를 제어합니다.

동일 작업의 thread_id와 입력 해시·코드/모델/프롬프트 버전을 보존합니다. 체크포인트는 로컬 PostgreSQL의 langgraph 스키마에 저장합니다. 승인 대상이 바뀌면 재검토하고 외부 쓰기는 재실행에도 중복되지 않도록 합니다. 프로그램 삭제·승인·공개는 Agent의 자유 도구로 제공하지 않습니다.

## 6. 관리자 기능과 API

Part A 화면은 온톨로지/분석 계약 설계, 표본 비교, 추출 실행, 근거 오류, 개체/사건 정합화 검토, 적재 상태, snapshot 확정, 개발 초기화로 구성합니다. Part B는 ready snapshot 선택과 보고서 생성·검토·공개에 집중합니다.

| API 초안 (로컬 관리자 전용) | 목적 |
|---|---|
| POST /admin/analysis-contracts | 질문 범위·근거·평가 계약 후보 등록 |
| POST /admin/analysis-contracts/{id}/approve | 해시·참조 온톨로지 확인 후 승인 |
| POST /processing-jobs | 원문·승인 계약 기준 KG 생성 작업 등록 |
| GET /processing-jobs/{id} | 추출·검증·정합화·적재 상태 |
| POST /admin/kg-snapshots | 특정 적재 결과·원문으로 manifest 생성·검증 |
| GET /admin/kg-snapshots/{id} | 상태·버전·보고서 영향 조회 |
| POST /research-jobs | Part B에서 승인 계약·ready snapshot·개별 brief로 보고서 생성 |

관리 API는 `local_admin`에서만 등록하며 Render 공개 데모(향후)에는 등록하지 않습니다. 공개 보고서는 검토된 인용과 방법 요약만 제공하며 내부 프롬프트·비밀값은 노출하지 않습니다.

## 7. 개발 삭제·재작업·복구

선택 범위 또는 프로젝트의 Part A 후보·승인 계약·추출·KG·snapshot을 개발 모드에서 삭제하고 재생성할 수 있습니다. 먼저 연결된 Part B 보고서·작업·공개 영향과 공유 노드 범위를 계산합니다. 관련 공개 보고서를 철회하고 A/B 작업을 중지한 뒤 자식부터 정리합니다. Crawler 원문과 삭제 범위 밖 공유 근거·소스 설정은 보존합니다.

복구는 로컬 dump의 업무/체크포인트 스키마와 원문·검증 결과로 수행합니다. 그래프 테이블은 dump와 함께 복원되며 manifest와 대조해 불일치 시 재구축합니다. 정리 중 실패는 재개 가능해야 하고 삭제 완료 전에 자동 재생성을 시작하지 않습니다. 상세 잠금·checkpoint 삭제·공개 철회는 시퀀스와 백업 설계를 따릅니다.

## 8. 개발 순서와 통과 기준

A1 최소 온톨로지·분석 계약 → 고정 표본 추출 → 근거 검증 → 정합화 → 멱등 적재 → ready snapshot → Part B의 보고서 한 편 순서로 구현합니다. 실패 재개·초기화·복원은 실제 표본으로 검증합니다.

필수 검증은 미승인 계약 차단, 인용 offset·의미 지지, 부정/전망 구분, 날짜·단위, 잘못된 병합, 재실행 중복, 동시 저장/삭제, snapshot 재현, ready 이전 B 진입 차단, 원문 정정 영향, 공개 보고서 참조 무결성입니다. 정량 품질은 사람 정답 표본과 baseline 비교로 평가하고 현재 수치 성능을 주장하지 않습니다.

관련 문서: [보고서 출력](report-output.md), [분석 방법론](research-approach.md), [기술 스택](tech-stack-analysis.md), [시퀀스](sequences.md), [백업](backup-restore.md), [CI/CD](ci-cd.md).
