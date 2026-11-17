# 심층 리서치 보고서 출력 설계

상태: 구현 전 기획. 실제 Agent 실행·보고서 생성은 아직 수행하지 않았습니다.

본 문서의 전문 검토는 기획·설계 문서를 개선하는 개발 활동입니다. 앱에 전문가 검색·에이전트 팀 편성을 추가하는 요구사항은 아닙니다. **Part A는 LLM 온톨로지 설계와 KG 구축·검증**, **Part B는 그 KG를 활용한 심층 보고서 작성**을 담당합니다. Part A가 승인 분석 계약·온톨로지 버전·KG snapshot/manifest를 제공하면 Part B는 읽기 기반 근거 탐색·분석·작성·검토·공개를 수행합니다.

## 1. 산출물과 범위

MVP는 **경제 이슈 하나·분석 기간 하나에 대한 출처 연결형 심층 보고서 한 편**을 완성하는 것입니다. PWA 열람과 Markdown 다운로드를 제공하고 PDF는 후속 확장으로 둡니다. 보고서 생성은 Part B 내부 기능이며 Crawler → Part A → Part B의 3단계를 유지합니다.

경제뉴스만 분석하므로 관측 대상은 기사에 보도된 사건·시장 반응·전망·설명입니다. 실제 가격 데이터가 없는 수익률·예측 정확도·개입 효과는 산출하지 않습니다. 기사에서 추출한 주장, 승인 규칙의 도출, 추가 연결 가설을 구분합니다.

### 입력 계약과 생성 진입 조건

| 입력 | 필수 의미·검증 |
|---|---|
| 연구 브리프 | 질문·이슈/대상·기간·독자·포함/제외 범위 |
| 분석 계약 | 승인된 analysis_contract_version·ontology_version·rule 버전·허용 읽기 도구·평가 기준 |
| KG 인계 | Part A의 kg_snapshot_id·manifest_hash·구축/검증 완료 상태·원문/주장/근거 참조 |
| 시간 정책 | 회고 분석 또는 당시 가용 자료 재현, 발행 cutoff·시스템 가용 cutoff·시간대 |
| 자료 manifest | 불변 기사 version_id·content_hash·선택/제외 이유·전재 그룹 |
| 실행 설정 | prompt/extractor/model/code 버전·검색/수정/시간/토큰 예산 |

브리프와 분석 계약·KG snapshot의 호환성을 먼저 확인합니다. 미승인·철회·누락 계약, 구축/검증 미완료 snapshot이나 미지원 질문은 생성 전 거부하거나 범위 조정을 요청합니다. 기존 snapshot으로 답할 수 있는 새 질문은 Part B 입력으로 처리합니다. 계약 변경·새로운 기사 추출·정합화 수정이 필요하면 Part A가 새 버전·snapshot을 발행해야 합니다. KG는 검색을 돕는 파생 저장소이며 기사 manifest를 대신하지 않습니다.

## 2. 보고서 목차 계약

| 절 | 필수 내용 | 연결 분석 축 |
|---|---|---|
| 분석 질문·범위 | 질문·대상·기간·정보 기준 시점·포함/제외 기준·표본 범위 | 공통 |
| 핵심 발견 | 주요 발견·해석 범위·불확실성 | 공통 |
| 사건 타임라인 | 사건일·발행일·정정·보도 흐름 | Temporal Analysis |
| 맥락과 대상 | 관련 기관·기업·업종·사건과 연결 경로 | Context Awareness |
| 보도된 시장 반응 | 관측 보도/전망/해석 구분·기사 내 수치·단위 | 시간·맥락 |
| 보도된 인과 설명 | 원인/결과 사건·발화자·양태·원문 | Causal Relation Analysis |
| 규칙 도출·추가 발견 | 규칙 ID·입력 근거 또는 가설·미검증 표시 | Logical Reasoning·Knowledge Discovery |
| 상충·누락·한계 | 서로 다른 주장·전재 중복·근거 부족·인과효과 미추정 | 공통·Causal Inference의 범위 제한 |
| 출처·생성 이력 | 문장 인용·URL·원문 버전·모델/프롬프트/온톨로지 버전 | 공통 |

모든 절을 형식상 채우기 위해 내용을 생성하지 않습니다. 해당 근거·규칙이 없으면 확인 불가·미지원·근거 부족으로 명시합니다. 과도하게 긴 보고서보다 질문에 답하는 근거·상충 비교·한계를 우선합니다.

## 3. 생성 파이프라인과 Agent

```text
create_research_job → validate_part_a_handoff → freeze_kg_snapshot_and_scope → plan_questions
  → retrieve_evidence → assess_coverage
  → 부족하면 예산 내 snapshot 내부 추가 읽기 조회
  → KG 자체 부족: request_part_a_enrichment → 보완 대기 또는 한계 기록
  → compare_claims → build_report_outline → draft_report
  → validate_citations_and_claims
  → 수정 가능하면 제한된 repair → 재검증
  → save_draft → human_review → publish_version → render_markdown
```

계획·추가 도구 선택은 Agent가 수행하고 입력 고정·저장·검증·공개는 프로그램이 제어합니다. LangGraph는 상태·분기·수정 횟수·검토 대기·재개를 관리합니다. 승인된 기사/개체/기간/경로/근거 도구를 사용하고 필요시 pgvector 의미 검색을 결합합니다. 임베딩 결과도 동일 snapshot/cutoff/profile 범위로 제한하며 원문 의미 검증을 대체하지 않습니다. 조회 hop·기사 수·도구 횟수·토큰·시간을 제한합니다. [임베딩 설계](embedding-retrieval.md)

방법론은 [참조 구현 계획](reference-implementation-plan.md)의 B06 GraphRAG·B07 ToG·B10 MIRAI를 우선 적용합니다. ToG식 relation/entity 확장·prune를 검증된 읽기 도구로 재구현하고, GraphRAG식 근거 묶음별 부분 분석→상충 비교→종합을 작성 노드로 나눕니다. MIRAI식 cutoff는 요청별 불변 context로 구현하고 원본의 임의 코드 실행·전역 시간 설정·오래된 SDK를 복사하지 않습니다. 원래 forecasting 과제는 보고서 생성과 구분합니다.

임베딩을 쓰는 실행은 KG snapshot과 별도로 `search_index_manifest_id`·profile 버전을 고정하여 생성 이력에 포함합니다. 검색 index의 재생성은 기존 KG manifest를 변경하지 않고 새 검색 manifest·새 보고서 실행으로 연결합니다. embedding 비활성 실행은 lexical/graph 정책 버전을 기록합니다. 후보 쿼리는 snapshot·시간·프로젝트 범위를 DB에서 먼저 제한하고 결과도 재검증합니다.

Part A는 지원 질문 유형·온톨로지·절별 근거·반례·검증 기준을 설계하고, 그 계약으로 원문의 추출·근거 검증·정합화·KG 적재·snapshot 발행을 완료합니다. 매 보고서의 구체적인 연구 질문은 Part B 입력으로 받으며 승인 계약과 완료 snapshot 안에서 수행합니다. 외부 웹 검색·가격 API는 현재 보고서 데이터에 포함하지 않습니다. Part B에는 KG 생성·수정·적재 도구를 제공하지 않습니다.

추가 탐색은 고정 snapshot이 참조하는 원문·KG의 읽기 조회만 수행합니다. 각 하위 질문은 answered/partial/insufficient/not_supported 상태와 근거를 기록하고, 검색을 시도하지 않은 질문을 ‘근거 없음’으로 단정하지 않습니다. snapshot 자체에 정보가 없거나 KG 오류를 발견하면 Part A 보완 요청을 남깁니다. 보완 후에는 새 snapshot을 고정한 새 Part B 실행을 생성하며 기존 실행의 근거를 중간에 교체하지 않습니다. 상충 근거 검색과 전재 중복 점검은 필수 점검 단계로 두되, 검색을 했다는 사실이 반대 근거의 완전한 포괄을 보장하지 않음을 한계에 표시합니다. 예산 소진 시 중간 산출물과 남은 질문을 저장하고 임의로 주제를 확대하지 않습니다.

## 4. 구조화 정본·인용·버전

보고서 정본은 구조화 JSON이고 Markdown은 결정적 renderer로 생성합니다. 핵심 필드는 question, scope, time_policy, cutoff_at, corpus_manifest, analysis_contract_version, ontology_version, kg_snapshot_id, kg_manifest_hash, sections, statements, citations, search_trace, limitations, validation_results, generation_versions입니다. Part A의 KG 구축 작업·적재 버전을 기록해 원문과 파생 결과의 대조 경로도 유지합니다. snapshot은 앱의 불변 manifest이며 DB 물리 백업을 뜻하지 않습니다.

KG 조회는 manifest 소속 주장·관계 ID와 버전 범위로 제한합니다. 그래프 테이블의 최신 노드만 조회한 뒤 과거 snapshot을 사용했다고 표시하지 않습니다. 정정으로 최신 그래프가 바뀌어도 기존 snapshot의 참조와 증거가 재현되어야 하며, 삭제되어 재현할 수 없으면 해당 snapshot·보고서를 사용 불가/철회 상태로 처리합니다. Part B의 분석 결과·규칙 도출·가설은 보고서 결과로 저장하고 원본 KG에 새 사실로 적재하지 않습니다.

주요 사실 문장에는 statement_id와 하나 이상의 citation_id를 연결합니다. 인용은 article_version_id·claim_id·evidence_id·원문 해시·구간·URL까지 추적합니다. 규칙 결과는 규칙 ID와 입력 주장, 가설은 탐색 경로와 미검증 상태를 추가합니다. 인용이 존재하는 것과 해당 문장을 의미적으로 뒷받침하는 것은 별도 검사입니다. 모든 문장에 무의미한 인용을 강제로 붙이지 않고 실질적 주장을 검사합니다.

분석 기간과 정보 기준 시점은 구분합니다. 과거 가용 자료 재현에서는 발행 시점과 시스템 가용 시점 모두 cutoff 조건을 만족하는 불변 원문 버전을 사용합니다. 처음 수집한 과거 기사로 당시 보도를 분석하는 경우에는 ‘현재 수집한 과거 보도의 회고 분석’으로 표시하고 당시 가용 자료를 재현했다고 주장하지 않습니다. 이후 수정 기사·새 근거를 반영하면 새 보고서 버전을 생성합니다. LLM에 원문을 전달할 때 기사 내 지시문을 시스템 지시로 실행하지 않고 데이터로 취급합니다.

작업 상태는 queued/running/waiting_review/waiting_part_a/completed/failed/cancelled, 보고서 버전 상태는 draft/needs_revision/approved/published/withdrawn으로 분리합니다. approved에서 명시적 publish를 거쳐 공개합니다. 공개 버전은 수정하지 않고 새 draft가 검토된 뒤 published 포인터를 바꿉니다. 승인 대상 해시는 보고서 content·인용 manifest·분석 계약 버전·온톨로지 버전·KG snapshot을 포함합니다. 새 보고서가 다른 원문·계약·snapshot을 사용하면 기존 승인을 재사용하지 않습니다. 원문 정정이 발생했다고 과거 보고서의 고정 입력을 덮어쓰지 않으며, 영향 검토 후 구버전 경고·철회 또는 새 버전 발행을 선택합니다.

초안·검토·공개 정보는 로컬 PostgreSQL(정본)에 저장하고 dump에 포함합니다. 공개 데모(Render, 향후)는 export된 published 버전만 조회하며 Markdown 파일을 임시 디스크에 저장하지 않고 저장된 정본으로 응답합니다. 원문 비공개 범위가 있다면 공개 인용은 별도로 필터링하고 내부 원문 전체를 Markdown에 포함하지 않습니다.

## 5. API·화면

| API 초안 | 실행 환경·권한 | 용도 |
|---|---|---|
| POST /research-jobs | 로컬 관리자 | 질문·기간·cutoff·analysis_contract_version·ontology_version·kg_snapshot_id 지정 생성 |
| GET /research-jobs/{id} | 로컬 관리자 | 탐색·작성·검증·비용·검토 상태 |
| POST /admin/research-jobs/{id}/enrichment-requests | 로컬 관리자 | 부족 질문·근거·현재 snapshot을 지정해 Part A 보완 요청 |
| GET /admin/reports/{id}/versions | 로컬 관리자 | 초안·공개 버전·인용·차이 확인 |
| POST /admin/reports/{id}/review | 로컬 관리자 | 대상 버전 해시 지정 승인/수정/거절 |
| POST /admin/reports/{id}/publish | 로컬 관리자 | 승인 버전 명시적 공개 |
| POST /admin/reports/{id}/regenerate | 로컬 관리자 | 선택 입력·현재 코드로 새 초안 생성 |
| GET /reports | 공개 조회 | 공개 보고서 목록·필터 |
| GET /reports/{id} | 공개 조회 | 현재 공개 버전·인용·한계 |
| GET /reports/{id}/markdown | 공개 조회 | 동일 공개 버전 Markdown 다운로드 |

PWA 첫 화면은 공개 보고서 목록·핵심 질문·분석 기간을 보여줍니다. 상세는 핵심 발견→타임라인→맥락/보도 반응→인과 설명→상충/한계→출처 순서로 구성하고 근거 패널로 이동할 수 있게 합니다. 관리자 Part B에는 작성 진행·주요 주장별 근거·오류·수정·공개 기능을 추가합니다.

화면·다운로드 응답에는 report_version_id·content_hash를 함께 포함합니다. 사용자가 열람한 공개 버전과 내려받은 버전이 같도록 다운로드 요청에 버전을 고정하고, 새 공개 포인터로 바뀌었을 때 조용히 다른 버전을 반환하지 않습니다. 철회 버전은 일반 공개 조회·다운로드에서 제외하고 원문 전문·내부 프롬프트·비공개 검토 의견은 공개 출력에 포함하지 않습니다. API 표의 경로는 기능 초안이며 실제 라우트는 `/api` 접두사 규칙에 맞춥니다.

기존 Part A 삭제에 연결된 Part B 초기화는 report job·인용·버전 영향도 포함합니다. 관련 공개 보고서는 우선 withdrawn 처리하여 깨진 출처가 공개되지 않게 하고, 삭제 범위 밖 다른 보고서의 공유 근거는 보존합니다. 단계별 정리·재개·감사 기록은 기존 초기화 원칙을 따릅니다.

## 6. 검증·완료 기준

- 주요 주장마다 연결된 근거가 실제 원문 버전·구간과 일치하고 문장의 의미를 지지하는지 확인합니다.
- 관측 보도와 전망·기자 해석·시스템 규칙 도출·가설을 구분합니다.
- 타임라인의 날짜·시간 정밀도·cutoff를 검사하고 이후 정보를 과거 결론에 누수시키지 않습니다.
- 상충 기사·인용 주체·전재 중복을 숨기지 않으며 근거 부족은 한계에 남깁니다.
- 실제 인과효과·수익률·가격 전망을 데이터 없이 생성하지 않습니다.
- 지원하지 않는 질문·예산 초과·DB 장애 때 부분 초안을 공개하지 않습니다.
- 생성 중단 재개·새 초안 재생성·공개 버전 보존·인용 무효화·동일 버전 다운로드를 검증합니다.
- KG 미완료/철회 snapshot 차단, snapshot 외 주장 유입 차단, Part B의 KG 쓰기 금지, Part A 보완 뒤 새 snapshot·새 실행 연결을 검증합니다.
- 공개 API와 빌드에 초안·관리 기능·LLM 키가 노출되지 않는지 확인합니다.

### 평가와 공개 게이트

| 평가 단위 | 지표·판정 | 공개 차단 조건 |
|---|---|---|
| 인용 연결 | 전체 주요 주장 대비 유효 citation 연결 비율 | 끊어진 원문 ID·해시/구간 불일치·주요 주장 근거 누락 |
| 의미 지지 | 검토한 주요 주장 중 근거가 의미를 지지하는 비율 | 전망을 관측으로 변경·부정 반전·인과효과 단정 |
| 질문 충족 | 하위 질문별 answered/partial/insufficient 상태와 이유 | 미지원 질문을 답한 것처럼 표시 |
| 시간 | cutoff·기간·정밀도와 입력 manifest 대조 | 미래 정보 누수·회고/당시 재현 구분 누락 |
| 상충·자료 범위 | 반대/정정 검색 수행·원문 중복군·제외 이유 | 알려진 상충을 숨김·전재를 독립 근거로 과장 |
| 공개 계약 | 승인 해시·버전·다운로드·공개 인용 범위 | 승인되지 않은 초안·철회 버전·비공개 원문 노출 |

정확한 자동 연결·구간 검사는 공개 대상 전체에 수행하고 의미 지지는 MVP의 공개 보고서 주요 주장을 사람이 전수 검토합니다. 표현 수준·사소한 요약 문장까지 같은 인용 요구를 기계적으로 적용하지 않고 ‘주요 주장’ 판정 기준을 분석 계약에 고정합니다. 거짓인 주장을 실제 사실로 확인한 지표가 아니라 입력 기사에 대한 지지 여부를 측정합니다.

초기 평가는 질문·기간·자료 manifest를 고정한 소규모 보고서 집합으로 시작합니다. 보고서 수·주장 수·검토 수·실패 수를 모두 보고하고 분모가 0이면 N/A로 둡니다. 사람 판정의 불일치는 합의 기준·수정 이유와 함께 보관합니다. 기사 단순 요약 baseline과 같은 모델·입력·예산에서 비교하고 비용·지연·검토 수정량도 함께 제시합니다. 목표 수치는 첫 파일럿 뒤 정하며 현재 수치 성능은 측정하지 않았습니다.
