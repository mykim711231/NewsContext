# NewsContext 필요 기술 스택 심층 분석

검토일: 2026-11-13. 상태: 설계 검토이며 설치·벤치마크·실서비스 연결 결과가 아닙니다. 공식 문서의 현재 기능과 프로젝트 요구사항을 비교한 권고입니다. 정확한 패키지 버전은 통합 검증 후 lock 파일로 확정합니다.

최종 출력은 [근거 기반 심층 리서치 보고서](report-output.md)로 확정했습니다. Part A가 생성·검증한 KG와 여섯 분석 축이 보고서의 근거를 공급합니다. Part B의 보고서 작성 Agent는 현재 LangChain·LangGraph 구성 안에 추가하고, 구조화 보고서를 정본으로 Markdown을 생성합니다. 문장별 인용 검증·버전·검토/공개 상태가 추가되며 PDF 라이브러리는 MVP 필수가 아닙니다.

## 1. 판단 기준과 결론

목표는 **경제뉴스만으로 주요 이슈와 기사에 보도된 단기 시장 반응을 추출하고, 기업·업종·사건의 관계를 원문 근거와 함께 탐색하는 포트폴리오**입니다. 실제 시장 수익률 계산이나 매매 예측이 목적이 아닙니다. 기사에 수치가 있으면 원문과 함께 추출하지만, 없는 가격·거래량·수익률을 생성하지 않습니다.

프로세스는 **Crawler(뉴스 원문) → Part A(LLM 온톨로지·KG 구축/검증) → Part B(KG를 활용한 심층 보고서)**로 확정합니다. Part A는 분석 계약·온톨로지를 승인하고 원문에서 추출·검증·정합화·적재한 `kg_snapshot_id`를 발행합니다. Part B는 `analysis_contract_version`, `ontology_version`, `kg_snapshot_id`와 보고서 brief를 입력으로 사용합니다. 앱 안에 전문가 검색·전문가 카탈로그·팀 기획 시스템을 새로 만들지 않습니다. 전문 에이전트 검토는 이번 기획·설계 문서의 검토 방법이며 제품 기능과 구분합니다.

> **저장소 기준(2026-11-17 정리):** 로컬 본체 우선 결정(2026-11-13)에 따라 MVP의 정본·작업 큐·체크포인트는 **로컬 PostgreSQL**, 그래프도 같은 PostgreSQL의 그래프 테이블입니다(2026-11-17 Neo4j 제외). Supabase·Render는 `APP_MODE=public_demo`(향후)의 export 대상으로만 남습니다([README 저장소 기준](../README.md)). 아래 7·11·12절의 Supabase 관련 한도·휴면 논의는 공개 데모 기준입니다.

공개 데모(Render, 향후)는 조회만 수행하고 데이터 생성·관리·백업은 로컬에서 실행합니다. 최신 사용자 지시에 따라 임베딩은 필요한 기능에 조건부 도입하며 정본인 로컬 PostgreSQL의 pgvector를 우선 사용합니다. 업무 스키마에는 FK를 생성하지 않고 프로그램에서 참조를 검사합니다.

논문·소스 기반 설계는 [참조 구현 계획](reference-implementation-plan.md), 실제 원본 파일·함수·license 확인은 [소스 감사](source-audit.md), vector profile·SQL·검색·차원/용량·재색인은 [임베딩 설계](embedding-retrieval.md)를 기준으로 보완합니다. 직접 원형 실행과 최신 LangChain/LangGraph 변형 구현은 별도 환경·상태로 관리합니다.

**현재 선택은 적합하나 데이터 계약·저장 경계·재현성이 기술 추가보다 우선**입니다. PostgreSQL 한 곳에 정본과 그래프 테이블을 두어 저장소 간 정합성 부담을 없애고, 그래프 DB는 측정 근거가 생길 때까지 보류합니다(8절).

| 영역 | 권고 | 우선도 | 주요 비용·위험 |
|---|---|---|---|
| Python·uv | 유지; 검증한 Python minor 및 uv 버전 고정 | 필수 | 개발 PC·CI·이미지 버전 불일치 |
| requests·BeautifulSoup4 | 유지; 초기 수집은 동기·순차 | 필수 | HTML 변경·수집 허용 범위·시간 정보 누락 |
| Pydantic·pydantic-settings | 유지; 데이터 계약과 실행 모드별 설정 검증 | 필수 | JSON 형식 통과를 사실 검증으로 오인 |
| LangChain | 구조화 추출과 제한된 보고서 근거 탐색 Agent | 필수 | 공급자별 출력·도구·재시도 차이 |
| LangGraph·PostgresSaver | 유지; 검토·재개·예산 분기에 사용 | 필수 | 체크포인트 증가·외부 쓰기 재실행 |
| 로컬 PostgreSQL·psycopg 3 (공개 데모·향후: Supabase Free) | 유지; 업무 정본·작업 큐·체크포인트 저장 | 필수 | 로컬 디스크·백업 / 공개 데모는 연결 수·용량·휴면 |
| 그래프 저장: PostgreSQL 그래프 테이블·고정 SQL (Neo4j·AuraDB 보류) | 변경(2026-11-17); 근거 연결과 제한된 관계 탐색 | 필수(모델), DB는 PostgreSQL | SQL 질의 복잡도·응답 시간 측정 |
| FastAPI·Uvicorn | 유지; 조회 서버와 로컬 관리 모드 분리 | 필수 | 동기 I/O 혼용·권한 경계 |
| React·TypeScript·Vite | 유지; PWA는 설치·정적 캐시에 한정 | 필수 | Python+JS 두 생태계 관리 |
| Docker·GitHub Actions·Render | GitHub Actions는 MVP; Docker(사전 승인 필요)·Render는 공개 데모(향후), 공개 조회 이미지 최소화 | 배포 필수 | 이미지 비대화·콜드 스타트·검사 누락 |
| pg_dump·pg_restore·로컬 스케줄러 | 유지; UI 수동 실행은 동일 함수 재사용 | 운영 필수 | PC 종료·불완전 파일·복원 미검증 |
| 전용 그래프 UI·검색 확장·외부 추적 | 실제 사용성·성능 문제 발생 시 도입 | 선택 | 프런트 복잡도·저장 공간·원문 외부 전송 |

## 2. Python과 함수형 처리

업무 로직은 타입을 가진 함수로 연결합니다. 순수 변환과 외부 I/O를 분리하면 모델·DB를 교체하고 fixture로 재현하기 쉽습니다. 불변 데이터 모델은 선택적으로 사용하되 모든 함수를 복사 비용이 큰 완전 불변 구조로 강제하지 않습니다.

```text
Crawler: fetch → parse → normalize → hash → save_version
Part A: question_types → ontology/contract → approve → extract → validate/resolve → KG → publish_snapshot
Part B: brief → pin_contract_and_snapshot → retrieve/compare → draft → validate → review/publish
```

파싱 함수에 DB 연결을 전달하거나 validator 안에서 LLM을 재호출하지 않습니다. 재시도와 수정 호출은 조립 계층에서 보이도록 둡니다. 업무 실패는 `ValidationResult`, `Skipped`, `NeedsReview`처럼 분류된 결과로 반환하고 인프라 오류는 명시적 예외로 전달합니다. 파일 경로·현재 시각·실행 ID는 실행 경계에서 결정합니다.

설정은 pydantic-settings로 실행 모드에 맞춰 검증합니다. `public_demo`에 LLM 키가 없다는 이유로 시작을 막아서는 안 되며, `local_admin`에서 선택한 공급자만 검증해야 합니다. `.env` 로딩은 pydantic-settings에 모으고 별도의 dotenv 로딩이 중복으로 환경변수를 덮어쓰지 않도록 합니다.

## 3. Crawler: requests와 BeautifulSoup4

참조 코드의 순차 수집은 MVP에 적합합니다. 병목은 CPU보다 출처의 요청 간격·응답 시간이므로 초기 asyncio 전환보다 신뢰할 수 있는 파싱·종료 조건이 중요합니다.

필수 저장 정보는 기사 키·정규 URL·제목·원문·발행 시각·수정 시각(있을 때)·수집 시각·원문 해시·정규화 버전입니다. 본문 offset 기준 텍스트를 고정하고 기사 수정 시 새 버전을 만듭니다. 발행 시각을 모르면 수집 시각으로 대체하지 않습니다.

원문 없는 제목·이미지 설명을 완전한 기사로 처리하지 않습니다. 빈 페이지·반복 링크·최대 페이지로 순회를 종료하고 HTTP 오류와 본문 파싱 오류를 구별합니다. 고정 HTML fixture로 출처별 파서 회귀 검사를 합니다. 수집·공개 재배포의 허용 범위를 출처별로 확인하고 포트폴리오 공개 화면에는 허용 범위의 인용·출처 링크를 제공합니다.

Playwright 기반 수집은 JS 렌더링이 필수이고 허용된 출처에서만 별도 검토합니다. 현재 정적 HTML 수집에 브라우저 자동화를 추가하는 것은 운영 부담이 더 큽니다.

## 4. 가장 중요한 기술: 뉴스 분석 데이터 계약

추출 계약은 금리 사건에 한정하지 않고 이슈·보도된 시장 반응·귀속된 인과 설명을 표현합니다. 실제 구현 전 Part A에서 표본·반례로 확정할 최소 계약은 다음과 같습니다.

| 정보 | 최소 필드 | 검증 기준 |
|---|---|---|
| 이슈·사건 | issue_type, description, event_time | 사실·발표·계획·전망을 구분 |
| 관련 대상 | entity_id, entity_type, mention | 기업·기관·업종·지수 이름과 원문 언급 |
| 시장 반응 보도 | target_id, direction, metric, value, unit, reported_period | 상승/하락/혼조/변화 없음/알 수 없음; 기사에 있을 때만 수치 저장 |
| 주장 성격 | observed_report / forecast / interpretation | 이미 보도된 움직임과 전망·해석 구분 |
| 시간 | published_at, event_time, reaction_time, collected_at | 상대 시점은 기준과 시간대가 있을 때만 정규화 |
| 연결 | relation_type, attribution | 기사에서 제시한 배경과 시스템의 연결 후보 구분 |
| 근거 | field_path, version_id, quote, start, end | 해당 원문 버전의 구간과 의미 지지 확인 |

기사 한 건에 여러 이슈와 반응이 있으면 일대일로 강제하지 않습니다. “금리 우려에도 상승”·“상승 전망이나 당일 하락” 같은 문장에서 관계 범위를 구분해야 합니다. 원인 관계는 기사에 귀속된 설명으로 저장하고 실제 인과효과로 승격하지 않습니다. 기사 수는 독립 근거 수와 같지 않으므로 전재·유사 기사 중복도 표시합니다.

경제뉴스 감성 점수 하나로 시장 반응을 대신하는 방식은 권장하지 않습니다. 기업에 긍정적인 기사와 해당 종목의 당일 상승은 다른 정보입니다. 단기의 초기 정의는 기사에 명시된 당일·최근 거래일·기간 표현이며 가격 데이터가 없는 상태에서 T+1·T+3 반응을 계산하지 않습니다.

## 5. LangChain: 고정 추출과 보고서 Agent의 경계

Part A 기사 추출은 정해진 입력과 스키마가 있으므로 `init_chat_model`로 만든 모델의 `with_structured_output` 경로가 적합합니다. Part B 보고서의 추가 근거 탐색에는 `create_agent` 기반의 제한된 도구 선택을 사용합니다. 기간·개체·경로·기사·근거 조회만 허용하고 임의 SQL, 외부 웹 검색, KG 갱신, 삭제·승인·최종 공개는 허용하지 않습니다. 도구 미지원 모델은 조용히 다른 모델로 전환하지 않고 해당 기능의 실행 전 검사에서 거부하거나 명시적인 고정 워크플로로 선택해야 합니다. Pydantic 검증은 출력 모양을 검사하는 단계이고 의미·근거·단위 검증은 별도로 수행합니다. [모델 공식 문서](https://docs.langchain.com/oss/python/langchain/models)

Agent는 질문을 좁히고 필요한 도구를 선택하며 충분하지 않은 자료를 발견할 수 있습니다. corpus·cutoff·허용 도구·예산을 확대하거나 승인된 분석 계약을 고치는 권한은 갖지 않습니다. 단일 Agent와 역할별 프롬프트부터 검증하고, 전문가 수나 호출 수 증가를 품질의 증거로 사용하지 않습니다.

공급자별 adapter를 두고 모델 초기화·사용량·오류 분류를 공통 출력으로 변환합니다. OpenAI·Anthropic·DeepSeek·Ollama의 구조화 출력과 도구 지원을 동일하게 가정하지 않습니다. 설정된 모델 식별자는 실제 접속 및 소규모 계약 검사로 확인해야 하며 이 검토에서 모델 가용성을 인증하지 않습니다.

공급자 네 개의 전환 구조는 유지하되 첫 완료 기준은 기본 공급자 하나의 종단 동작으로 잡습니다. 다른 공급자는 같은 20~40건 평가셋에서 검증한 뒤 지원 상태를 표시합니다. 자동 fallback은 비용·재현성을 바꾸므로 사용하지 않습니다. Ollama는 로컬 구동이 아니라 월 구독 API 키로 사용합니다(2026-11-13 사용자 결정). 로컬 모델 크기·양자화·하드웨어 기록은 필요 없고, 대신 구독 플랜의 요청 한도·모델 목록·이용 조건을 확인해 처리 이력에 모델 식별자를 기록합니다.

입력 해시·schema/prompt/extractor 버전·모델 설정별 결과 캐시는 재처리 비용을 줄입니다. 동일 입력이라도 LLM 출력이 항상 같지는 않으므로 재평가는 별도 실행 ID로 구분합니다. 비용은 관측 토큰과 해당 실행 시 가격 정보로 계산한 추정치로 표시하고 정확한 청구액처럼 표현하지 않습니다.

## 6. LangGraph와 PostgreSQL 체크포인트

단순 함수 호출만으로 수집은 구현할 수 있습니다. LangGraph는 Part A의 온톨로지 승인·KG 추출/검증/적재와 Part B의 근거 탐색·작성·검토·실패 재개를 각각 관리합니다. A의 KG 작업 thread와 B의 보고서 작업 thread를 구분하고 같은 실행을 두 프로세스가 동시에 소비하지 못하도록 합니다. StateGraph는 일반 함수 노드를 상태와 edge로 연결하므로 함수 중심 설계와 맞습니다. [Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)

노드 경계를 모든 미세 함수로 나누면 상태와 체크포인트가 늘어납니다. 한 노드는 재실행하기 적합한 업무 단위로 묶고 내부 순수 함수는 일반 호출합니다. 원문 전체·모든 중간 후보를 매 단계 상태에 중복 보관하지 않고 불변 version_id·result_id를 중심으로 전달합니다. 재개 시에도 동일 원문·프롬프트·코드 계약이 확보되어야 합니다.

`langgraph-checkpoint-postgres`를 사용해 로컬 PostgreSQL의 별도 `langgraph` 스키마에 저장합니다. 패키지의 setup·schema/search_path·connection 설정은 설치 버전으로 검증하고 라이브러리 테이블에 수동 SQL을 임의로 추가하지 않습니다. 공식 checkpoint 저장과 외부 업무 쓰기는 자동으로 한 트랜잭션이 되지 않습니다. 멱등 적재와 작업 완료 대조가 필요합니다. [체크포인터](https://docs.langchain.com/oss/python/langgraph/checkpointers)

승인 대상 해시와 schema 버전을 확인하고 interrupt 노드 앞에서 중복 외부 쓰기를 피합니다. 진행 중·검토 대기는 보존하고 완료 thread 정리는 지원 API·검증된 절차로 수행합니다. 코드 변경 후 과거 checkpoint를 무조건 재개하지 말고 실행 버전 호환 여부를 검사해 재개 또는 새 작업으로 분기합니다.

Part A는 원문 version/hash 목록인 corpus manifest·추출/적재 batch·코드/모델/프롬프트를 고정합니다. Part B의 재개 계약은 `analysis_contract_version`, `ontology_version`, `kg_snapshot_id`, brief 해시, 코드·프롬프트·모델 버전을 함께 고정합니다. checkpoint에는 큰 원문·보고서 사본을 매번 넣지 않고 그 버전의 정본 ID를 넣습니다. 체크포인트에 참조된 정본을 먼저 삭제하면 재개가 깨지므로 삭제 정책도 함께 관리합니다. 같은 thread의 동시 재개를 job lease와 실행 토큰으로 차단하고, lease가 만료된 이전 Worker의 늦은 완료 쓰기는 현재 토큰 비교로 거부합니다.

## 7. PostgreSQL(로컬 정본, 공개 데모는 Supabase)과 FK 없는 무결성

선택 검색을 위해 vector extension과 명시적 embedding 테이블을 추가할 수 있습니다. LangChain embedding adapter로 벡터를 만들고 psycopg SQL/custom retriever로 조회합니다. 자동 FK·관리 테이블을 만드는 wrapper는 검토 전 도입하지 않습니다. 체크포인트·원문·vector가 같은 Free 용량을 사용하므로 profile별 행·인덱스·복원 의존성을 함께 측정합니다. product default로 외부 Chroma/FAISS 서비스를 추가하지 않습니다.

MVP는 로컬 PostgreSQL을 쓰고, 공개 데모(향후)의 Supabase는 관리형 PostgreSQL 저장소로만 사용합니다. 현재 Auth·Storage·Realtime까지 도입할 근거는 약하며 프런트는 FastAPI만 호출합니다. psycopg 3와 명시적 SQL이 PK·인덱스·트랜잭션을 설명하기 쉽고 현재 함수 중심 설계에 적합합니다. SQLAlchemy ORM은 필수로 추가하지 않습니다.

FK 미사용은 구현 부담을 DB에서 프로그램으로 옮기는 선택입니다. 저장 함수의 공통 부모 행 잠금·참조 검사·상태 전이를 모든 API·CLI·Worker에 적용해야 합니다. 관계 없는 `exists` 조회와 나중의 insert를 분리하면 경합이 생길 수 있습니다. 신규 부모 insert 경쟁은 고유 인덱스와 충돌 후 재조회로 처리합니다. deadlock은 일정한 잠금 순서와 제한된 재시도로 관리합니다.

업무 transaction은 짧게 유지합니다. LLM·HTTP 응답을 기다리는 동안 row lock이나 DB 연결을 점유하지 않습니다. psycopg connection context의 commit/rollback을 명시적으로 사용하고 작은 connection pool로 시작합니다. 로컬 Worker·관리 API·체크포인터의 합산 연결 수를 관리합니다. 공개 데모(향후)에서는 앱 pool과 Supabase pooler의 역할이 다르며 Render 연결도 합산합니다. [psycopg pool](https://www.psycopg.org/psycopg3/docs/advanced/pool.html)

작업 상태와 보고서 상태는 분리합니다. 작업은 queued/running/waiting_review/waiting_part_a/completed/failed/cancelled, 보고서 버전은 draft/needs_revision/approved/published/withdrawn으로 관리합니다(정본: [README 단계명·상태값](../README.md)). 코드에서 허용 전이를 검사하고 중복 요청 키·보고서 content_hash·인용 manifest를 승인에 묶습니다. `published` 포인터 갱신은 PostgreSQL의 한 트랜잭션으로 처리하며 독자에게 서로 다른 버전의 본문·인용·다운로드가 섞이지 않도록 `report_version_id`를 응답과 다운로드에 고정합니다.

필수 인덱스는 기사 식별·원문 버전·프로젝트/상태/생성시각별 작업·참조 ID입니다. 사용하지 않는 JSONB 전체 GIN 인덱스를 미리 추가하지 않습니다. 키워드는 초기 ILIKE와 구조화 필터로 시작하고 느려지면 `pg_trgm` GIN을 측정합니다. 이는 임베딩이 아니라 문자 기반 인덱스이며 짧은 검색어·한국어 표현별 효율을 확인해야 합니다. [PostgreSQL pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)

로컬 정본은 디스크 여유로 관리합니다. 공개 데모(향후) export는 Supabase 공식 Free 제공 DB 크기 500MB를 예산 기준으로 보고 원문 버전·인덱스·추출 결과·체크포인트를 합산합니다. 실제 disk 할당과 요금제 DB 크기 지표는 구분해서 모니터링합니다. 예를 들어 본문 20KB×5,000건은 단일 본문만 약 100MB인 계산이며 중간 결과와 인덱스를 포함한 실측 용량이 아닙니다. checkpoint는 작은 표본에서 먼저 측정해야 합니다. [Supabase 요금제](https://supabase.com/pricing)

## 8. 그래프 저장: PostgreSQL 테이블 (Neo4j 보류, 2026-11-17)

> 2026-11-17 결정: MVP는 별도 그래프 DB(Neo4j Community·AuraDB)를 쓰지 않고 로컬 PostgreSQL `newscontext` 스키마의 그래프 테이블(노드·관계·snapshot membership)에 지식그래프를 둡니다. 공개 데모(향후)도 Supabase 한 곳에 export합니다. 상세와 재도입 조건은 [README 그래프 저장 방침](../README.md)에 있습니다.

기본 검색·기사 목록은 PostgreSQL만으로 충분합니다. 그래프의 역할은 “같은 이슈에 연결된 여러 기업·업종·보도 근거”를 탐색하는 관계 질의입니다. 이 질의는 지식그래프 **모델**이 필요하다는 근거이지 별도 그래프 **DB**가 필요하다는 근거는 아닙니다.

시연할 핵심 질의를 먼저 고정합니다: 특정 이슈의 관련 기업·업종, 동일 대상의 보도 반응 변화, 동일 주장에 대한 근거 기사, 충돌하는 관측/전망입니다. 제한된 1~2 hop·기간·노드 수 상한과 목록형 대체 화면을 사용합니다. 이 범위는 조인 또는 깊이 상한을 둔 재귀 CTE로 표현할 수 있으므로 SQL로 먼저 구현하고 응답 시간을 측정합니다. 초기부터 전체 그래프를 브라우저에 전달하지 않습니다.

원문·검증 결과·적재 상태와 그래프가 한 PostgreSQL에 있으므로 그래프 쓰기와 작업 상태를 한 트랜잭션으로 묶을 수 있고 이중 쓰기·저장소 간 대조·별도 그래프 백업이 필요 없습니다. 그래프 접근은 도구 함수(이웃·경로·주장/근거 조회) 뒤에 두어, 아래 재도입 조건이 충족되면 호출부를 바꾸지 않고 Neo4j를 파생 조회 저장소로 추가할 수 있게 합니다.

Part A의 `kg_snapshot_id`는 DB의 물리 복사를 뜻하지 않습니다. PostgreSQL의 불변 manifest에 corpus·claim/evidence 버전·batch·온톨로지·계약·내용 해시를 기록하고, 그래프 조회는 snapshot membership 테이블에 포함된 버전만 선택합니다. 공유 Entity는 안정 식별에 사용하되 이름·소속·사건 속성의 변경 가능한 최신값을 과거 사실처럼 사용하지 않습니다. 기존 주장을 덮어쓰기보다 새 주장 버전·snapshot membership을 추가하고 이전 snapshot이 참조하는 근거를 유지합니다.

Part A가 적재 완료·참조·근거 대조를 통과한 snapshot만 ready로 발행하고 Part B는 ready snapshot만 읽습니다. 보고서 도구가 최신 그래프 전체를 조회해 과거 snapshot 밖 근거를 섞지 않도록 snapshot 필터를 공통 적용합니다. Part B에 필요한 근거가 없으면 A 재구축 요청 또는 한계 표시로 처리하며 자체 추출·KG 쓰기는 하지 않습니다. 이미 공개한 보고서의 출처 ID가 바뀌지 않아야 합니다.

**Neo4j 재도입 조건(측정으로 확인될 때만):** (a) 위 핵심 질의가 SQL로 지나치게 복잡하거나 목표 응답 시간을 넘는 경우, (b) G4의 ToG식 탐색처럼 3 hop 이상 가변 경로 탐색이 필요해진 경우. 재도입해도 PostgreSQL이 정본이고 Neo4j는 재구축 가능한 파생 저장소입니다. 공개 데모의 그래프 행 수는 Supabase Free 500MB 예산(7절)에 포함해 측정합니다.

## 9. FastAPI와 로컬 Worker

단일 저장소에서 실행 진입점을 분리합니다: 공개 조회 앱, 로컬 관리 앱, 로컬 Worker, 백업 CLI. 공개 앱 시작 경로에서는 모델 초기화·LangGraph setup·작업 큐 처리·마이그레이션을 import 부작용으로 실행하지 않습니다. 의존성은 core/api와 processing 그룹으로 나누고 Render 이미지에는 조회에 필요한 부분만 설치합니다.

동기 psycopg를 쓴다면 우선 FastAPI의 일반 `def` endpoint와 맞춥니다. `async def` 안에서 동기 DB 호출을 직접 실행해 event loop를 막지 않습니다. 비동기로 옮길 때는 DB·드라이버·checkpoint까지 경로를 맞춥니다. [FastAPI 동기·비동기](https://fastapi.tiangolo.com/async/)

Worker는 PostgreSQL queue와 lease·heartbeat로 충분합니다. 작업 선점에 row lock·SKIP LOCKED를 사용하고 종료 후 재실행을 고려합니다. Redis·Celery·Kafka는 로컬 단일 Worker 규모에서 필수 아닙니다. 중지 요청은 안전한 단위 사이에서 확인하고 미응답 프로세스는 lease 만료 후 회수합니다.

공개 앱은 읽기 전용 DB 역할과 demo project 범위, 제한된 pagination·쿼리 시간·최대 탐색 깊이를 적용합니다. 조회 실패를 실제 이슈가 없다는 빈 결과로 표시하지 않습니다. 운영 health와 외부 DB 상태는 구분합니다. 인증·삭제·복원 기능은 local_admin에만 존재해야 합니다.

## 10. React PWA

React·TypeScript·Vite는 표·필터·타임라인·후보 비교·단계별 관리에 적합합니다. PWA는 PC·모바일 진입을 돕고 분석 로직은 Python에 둡니다. 초기에는 React Router·fetch·로컬 state·일반 CSS로 충분하며 Redux·Next.js·대형 UI 라이브러리를 일괄 추가하지 않습니다. 서버 상태 캐시가 복잡해지면 별도 라이브러리를 검토합니다. [Vite](https://vite.dev/guide/)

graph UI는 선택 항목입니다. 이슈 중심 목록·타임라인·근거 패널로 먼저 효용을 검증하고 관계 탐색이 필요할 때 Cytoscape.js 같은 전용 라이브러리를 비교합니다. 선택 시 대량 노드 처리보다 소수 관계의 가독성·선택·근거 연결을 우선합니다.

주 화면은 보고서 목록→본문→문장별 근거 패널→동일 버전 Markdown입니다. 타임라인·그래프는 보고서의 설명을 돕는 보조 화면입니다. 초안 미리보기는 local_admin에만 제공하고 본문 Markdown은 허용된 렌더링·링크 정책으로 처리합니다. 기사 원문이나 모델 출력에 포함된 HTML·명령문을 실행하지 않습니다.

Service Worker는 정적 파일만 캐시하고 API·관리 데이터·비밀값은 캐시하지 않습니다. 설치된 앱도 DB가 휴면이면 라이브 데이터를 읽을 수 없습니다. 오프라인에서는 입력 편집 손실 방지와 연결 안내만 제공하고 승인·삭제·재적재 요청을 큐에 쌓지 않습니다. API 계약 변경 시 업데이트 안내가 필요합니다.

로컬 관리에서는 HttpOnly 세션·CSRF와 서버 인증을 적용하고 공개 빌드에는 관리 라우트를 숨기는 것에 더해 API 미등록을 적용합니다. 비밀값을 VITE_* 환경변수로 전달하지 않습니다. 같은 origin 배포는 CORS·인증 경계를 단순하게 합니다.

## 11. Render·GitHub·Docker

MVP는 GitHub 형상관리·Actions 검증까지만 구성합니다. 이하 Render 내용은 공개 데모(향후) 기준입니다. Render Free는 공개 조회 데모에 적합하고 생성 작업에는 맞지 않습니다. 15분 무요청 시 휴면하며 재시작·휴면 때 로컬 파일 변경분이 사라집니다. 읽기 API의 DB 오류·기동 지연·재시도 UI를 준비합니다. [Render Free](https://render.com/docs/free)

Docker 다단계 빌드에서 React 정적 파일을 만들고 최종 이미지에는 최소 Python API·정적 파일만 남깁니다. 공개 서버에는 Ollama·크롤러 브라우저·모델 키가 필요 없습니다. 처음에는 단일 Uvicorn 프로세스로 메모리와 응답을 측정하고 worker 수를 늘려 메모리를 불필요하게 복제하지 않습니다.

GitHub PR·main 검사 후 Render의 After CI Checks Pass로 배포합니다. CI는 Python·React·PWA·공개 쓰기 차단을 확인하고 운영 DB를 변경하지 않습니다. 필수 검사가 skip돼 통과하는 빈 workflow는 금지합니다. CI 성공은 Render 빌드·기동·조회 성공과 다르므로 배포 후 커밋 SHA·검색·근거 조회를 확인합니다.

## 12. 백업·복원·무료 서비스 장애

pg_dump custom format으로 newscontext와 langgraph를 같은 snapshot에 저장하고 로컬 영속 경로와 별도 저장소에 보관합니다. 로컬 작업 스케줄러는 PC가 켜져 있어야 하므로 누락 실행과 마지막 성공 시각을 확인합니다. GitHub Actions artifact나 Render 디스크를 유일한 백업으로 사용하지 않습니다.

관리 화면의 백업·복원 기능은 후속 구현에서 CLI와 같은 함수를 호출하게 하고 목록·실행 이력은 로컬에서도 읽을 수 있어야 합니다. 대상 PostgreSQL이 접속 불가한 상태에서는 백업 실행이 실패하므로 DB 복구와 dump 복원을 구분합니다. 복원은 격리 검증 → 대상 확인 → 현재 상태 추가 백업 → Worker 정지 → 복원 → 참조 검사(그래프 테이블 포함) 순서입니다. 백업 대상은 로컬 PostgreSQL(정본)이며 Supabase는 로컬에서 재export하므로 백업하지 않습니다.

(공개 데모·향후) Supabase의 7일 저활동 정책은 즉시 삭제가 아닌 일시 중지이며 현재 공식 재개 기간은 중지 후 1년입니다. 이 경우 Dashboard Resume를 먼저 수행하며 dump를 자동 덮어쓰지 않습니다. 모든 접속 실패를 휴면으로 판정하지 않습니다. [휴면 정책](https://supabase.com/docs/guides/platform/free-project-pausing)

백업은 목록 확인만으로 완료하지 않고 격리 DB에 실제 복원해야 합니다. checkpoint 재개는 대응 코드·schema 버전과 맞춰 검증합니다. 그래프 테이블은 같은 dump에 포함되어 시점이 일치하며, manifest 대조 실패 시를 위한 재구축 경로는 유지합니다.

개발 초기화는 조회 차단/공개 보고서 철회→신규 작업 차단→실행 토큰 무효화·Worker 종료 확인→checkpoint 정리→관련 결과/인용/비공유 그래프 삭제→Part A 계약 삭제→참조 검사 순서로 설계합니다. 장시간 DB row lock을 잡은 채 Worker 종료를 기다리지 않고 영속 reset 작업과 프로젝트 fence 상태를 사용합니다. 독립 저장소 사이 단계별 실패를 기록해 재개하며, 완료 전까지 해당 범위의 재생성을 시작하지 않습니다. 원문·감사·백업 이력과 다른 보고서가 참조하는 공유 근거는 보존합니다.

## 13. 검증·관측·포트폴리오 증거

테스트의 중심은 모델 응답의 모양보다 잘못된 주장을 얼마나 차단하는가입니다. 기사 원문·정답 이슈·대상·반응·근거를 고정하고 같은 사건/전재 기사를 개발·보류셋 양쪽에 넣지 않습니다.

| 검증 | 꼭 포함할 사례 |
|---|---|
| 파서 | 빈 본문·중복 링크·수정 기사·발행 시각 누락 |
| 의미 추출 | 전망과 관측 혼재·부정·상승에도/하락에도·기사 내 다중 이슈 |
| 근거 | 반복 인용·문단 offset·원문 수정 후 이전 근거 |
| 무결성 | 부모 삭제와 자식 생성 경합·고유키 충돌·작업 상태 전이 |
| 재개 | LLM 직후 중단·그래프 반영 직후 중단·승인 후보 변경 |
| 운영 | DB 접속 실패·중지 후 재개·로컬 PC 종료·dump 복원 |
| 공개 화면 | write API 미등록·API 비캐시·오류와 빈 결과 구분 |
| 보고서 | 승인 해시 변경·초안 비공개·본문/인용/다운로드 동일 버전·지원 안 되는 질문 |
| 초기화 | 늦게 끝난 Worker 쓰기 거부·checkpoint 참조 정리·공유 근거 보존·철회 후 공개 차단 |

측정은 이슈/대상/반응/근거별 precision·recall, 잘못된 병합, 검토 비율, 기사당 토큰·시간·추정 비용으로 나눕니다. 뉴스만 사용하는 결과를 실제 가격 예측 정확도로 평가하지 않습니다. LLM 평가는 보조로 사용하고 보류 표본을 사람이 확인합니다.

로그에는 run_id·job_id·thread_id·article_version·schema/prompt/model 버전을 연결합니다. 원문 전체·키·연결 URL은 로그에서 제외합니다. LangSmith 등 외부 추적은 원문 전송 범위·비용을 확인한 후 선택적으로 도입하며 MVP는 구조화 로그·DB 실행 지표로 시작합니다.

## 13-1. 구현 충분성 점검 (2026-11-13)

결론: **핵심 파이프라인(수집→추출→KG→보고서)을 만들 기술은 충분하나, 아래 공백은 구현 전에 메워야 합니다.** 웹 검색 요약 기준이며 설치·실행 검증은 아닙니다.

| 공백 | 내용 | 조치 |
|---|---|---|
| 한국어 처리 | 형태소·문장 분리 도구가 스택에 없음. PostgreSQL 기본 전문 검색은 한국어를 지원하지 않음 | **기본선: pg_trgm(내장 3-gram) + ILIKE.** 조사 붙은 어절 검색이 부족하면 pg_bigm(2-gram, 1글자부터 검색) 후보이나 Supabase 프로젝트에서 `pg_available_extensions` 조회 결과 0행(2026-11-13) → 공개 데모는 pg_trgm 유지, 로컬 PostgreSQL은 별도 선택. mecab-ko+textsearch_ko는 네이티브 컴파일이 필요해 관리형 DB에서 부적합(로컬 PostgreSQL에서만 선택 가능). kiwipiepy는 선택 사항이며 Apache-2.0 확인(0.24 이상, NOTICE·모델 패키지 표기는 배포 전 확인)([source-audit §3-1](source-audit.md)) |
| 한국어 평가 | 금융 뉴스용 공개 한국어 NER·관계·이벤트 데이터셋을 찾지 못함 | 자체 20~40건 평가셋이 유일한 근거. 일정에 라벨링 시간 포함 |
| 본문 추출 | requests+BS4만으로 출처별 파서를 직접 유지 | trafilatura와 고정 HTML로 비교 후 채택 판단 |
| 로컬 Neo4j Community | (2026-11-17 보류) GPLv3, 사용자 DB 1개, dump는 서버 정지 상태에서만 가능 | 재도입 시에만 해당. 현재 그래프는 PostgreSQL dump에 포함 |
| 로컬 구동 방식 | Docker Compose 등은 새 의존성이므로 설치 전 승인 필요([corpus-feasibility §8.2](corpus-feasibility.md)) | 승인 후 결정. 승인 시 이미지는 `latest`가 아닌 고정 태그(예: 2026.07.1 Community), 볼륨은 named volume 또는 Windows 경로(`$HOME` 사용 불가), 마운트 폴더는 사전 생성. 비밀번호 변경은 데이터 볼륨이 남아 있으면 반영되지 않을 수 있음(2019년 자료) |
| 환경 분리 | 로컬/공개 연결 프로필, 공개용 export 단계가 스택·환경변수에 미반영 | 실행 모드 `APP_MODE`(`local_admin`/`public_demo`)는 README·ci-cd에 이미 있음. 남은 것: 로컬 PostgreSQL 연결값을 `local_admin` 프로필에 명시하고 공개용 export 단계(manifest·행 상한 검사, 그래프 테이블 포함)를 설정 계약에 추가 |
| 중복 기사 | 전재·근접 중복 판정 도구 미지정 | 후보: datasketch(MinHash-LSH, MIT는 2차 자료 기준), text-dedup(NEWS-COPY 벤치마크 보고), Gaoya(Rust, MIT). 뉴스 근접 중복 기준 예시는 trigram Jaccard ≥ 0.8. 임계값은 자체 평가셋으로 결정 |
| 그래프 라이브러리 | neo4j-graphrag-python: 2026-11-13 도입 결정 → 2026-11-17 Neo4j 제외로 **보류** | Neo4j 재도입(8절 조건 a·b) 시 다시 판단([source-audit §3-1](source-audit.md)) |
| 버전 고정 | langgraph-checkpoint-postgres 3.1.1(미러 확인), neo4j-graphrag 1.14.1/1.18.0 상충(보류 중이므로 해당 없음) | 첫 통합 시 lock으로 확정 |
| Windows 운영 | 로컬 스케줄러·pg_dump 클라이언트·UTF-8 경로 동작은 미검증 | 첫 백업 스크립트 구현 시 실측 |

(2026-11-17) Neo4j는 보류되어 버전 확인 대상에서 제외합니다. 재도입 시 그 시점의 릴리스를 확인합니다.

## 14. 구현 우선순위와 보완할 문서

1. 뉴스만 사용하는 출력 계약을 먼저 확정하고 기사 20~40건을 수동 검토합니다.
2. Crawler·원문 버전·논리 참조 저장을 구현합니다.
3. 기본 LLM 공급자 하나에서 Part A 분석 계약·KG snapshot과 Part B 보고서 최소 경로를 완성합니다.
4. 로컬 PostgreSQL checkpoint·중단/재개·그래프 테이블 멱등 적재와 핵심 그래프 질의의 SQL 응답 시간을 검증합니다.
5. 공개 조회 API와 PWA로 보고서→주장→원문 근거→동일 버전 Markdown 경로를 시연합니다.
6. 로컬 관리자·삭제 재생성·백업 복원·CI/CD를 연결합니다.

금리 변경은 경제 이슈 계약을 검증하는 예시 중 하나입니다. 구현 착수 시 계약에 지원하는 질문·미지원 질문과 보고서 절별 완료 기준을 명시하고 샘플 보고서부터 검증합니다. 관리자 백업·복원·분석 Agent를 포함한 모든 기능은 현재 설계이며 실제 구현이 아닙니다. 무료 서비스 접근성·LLM 모델 가용성·패키지 호환성은 첫 통합 실험에서 다시 확인합니다.

완료된 포트폴리오는 서비스 URL뿐 아니라 아키텍처·실제 평가 보고서·실패 사례·복구 시연·배포 이력으로 설명합니다. 특히 근거 기반 추출, FK 없이 관리한 무결성, 중단 복구, 개발 초기화, 조회용 그래프 재구축을 설계 판단과 실험 결과로 연결하는 것이 핵심입니다.
