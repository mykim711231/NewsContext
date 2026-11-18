# 실행 구조 설계: 공유 상태 + 노드·엣지 + 도구 + 하네스

> **상태: 제안 설계(미검증).** 라이브러리 설치·실행 검증은 하지 않았습니다. 단계명·상태값은 [README 단계명·상태값 정본](../README.md)을 따르며, 이 문서는 그 이름 위에 상태 필드·분기·도구·하네스를 정의합니다. 충돌하면 README가 우선합니다.

## 1. 네 요소의 역할 분담

| 요소 | 담당 | 담당하지 않는 것 |
|---|---|---|
| 공유 상태(LangGraph state) | 실행 중 ID·버전·결과·오류·예산 카운터 | 원문·모델·DB 객체, 업무 정본 데이터 |
| 노드·엣지 | 업무 함수 호출, 상태 변경분 반환, 조건 분기·재시도·interrupt | 업무 로직 자체(순수 함수에 위임) |
| 도구(Tools) | Agent가 선택할 수 있는 읽기 조회 | KG 쓰기·삭제·승인·공개·임의 SQL·외부 웹 |
| 하네스 | 모델·도구 호출 전후의 강제 검사(예산·허용 도구·인용·승인 해시·실행 추적) | 모델의 판단 대체 |

정본 데이터는 PostgreSQL에 있고 상태는 그 ID만 참조합니다. 체크포인트와 업무 쓰기는 별도 트랜잭션입니다([README](../README.md) 체크포인트 절).

## 2. 상태 스키마

그래프는 세 개이며 thread를 공유하지 않습니다(Part A 온톨로지, Part A KG, Part B). 상태는 `TypedDict`, 값은 직렬화 가능한 ID·버전·결과·오류만 둡니다. "reducer"는 병렬 노드가 같은 키를 쓸 때만 지정합니다.

### 2.1 공통 필드(세 그래프)

| 필드 | 쓰는 노드 | reducer |
|---|---|---|
| job_id, thread_id | 시작 시 1회 | 없음 |
| execution_token | 작업 선점 시 | 없음 |
| errors | 실패한 모든 노드 | 있음(append) |
| step_trace | 하네스 래퍼 | 있음(append) |

### 2.2 Part A-KG

| 필드 | 쓰는 노드 | 비고 |
|---|---|---|
| article_version_id | load_article_version | |
| contract_version, ontology_version | load_approved_contract | 이후 변경 불가 |
| topic_decision | filter_topic | included / excluded |
| segment_ids | split_article | 문단 병렬 추출 시에만 Send 도입 |
| candidate_batch_id | extract_candidates | 후보 본문은 DB, 상태에는 ID |
| validation_result | validate_candidates, revalidate_references | 오류 목록 ID·통과 여부 |
| canonical_map_id | canonicalize_entities, resolve_event_candidates | |
| review_decision | decide_review | load / review / hold / fail |
| graph_batch_id | build_graph_batch | 멱등 키 |
| batch_status | write_graph_batch, verify_graph_batch | |
| snapshot_id | freeze_kg_manifest, publish_kg_snapshot | |

### 2.3 Part A-온톨로지

| 필드 | 쓰는 노드 |
|---|---|
| question_set_id | define_questions |
| sample_set_id | select_samples |
| schema_draft_id | propose_ontology, normalize_schema |
| schema_validation | validate_schema |
| review_decision, approved_hash | review_ontology |
| contract_version | approve_contract |

### 2.4 Part B

| 필드 | 쓰는 노드 | 비고 |
|---|---|---|
| research_question, cutoff_at | create_research_job | 이후 변경 불가 |
| contract_version, ontology_version, kg_snapshot_id, kg_manifest_hash | validate_part_a_handoff, freeze_kg_snapshot_and_scope | 이후 변경 불가 |
| search_index_manifest_id | freeze_kg_snapshot_and_scope | 임베딩 사용 시 |
| sub_questions | plan_questions | id·status(answered / partial / insufficient / not_supported) |
| evidence_ids | retrieve_evidence | 병렬 조회 시 reducer(중복 제거) |
| coverage_result | assess_coverage | 부족 질문 ID 목록 |
| budget | 하네스 | hop·기사 수·도구 횟수·토큰·시간의 한도와 사용량 |
| claim_comparison_id | compare_claims | |
| outline_id, draft_id | build_report_outline, draft_report | |
| validation_result, repair_count | validate_citations_and_claims | repair 상한은 계약에서 지정 |
| report_version_id, approved_hash | save_draft, human_review | |

고정 필드(cutoff, 계약·온톨로지·snapshot 버전)는 시작 노드가 한 번 쓰고 이후 노드가 덮어쓰면 하네스가 거부합니다.

## 3. 노드·엣지

### 3.1 Part A-KG

| 노드 | 다음 | 분기·실패 경로 |
|---|---|---|
| filter_topic | split_article | excluded이면 skipped로 종료 |
| validate_candidates | load_aliases | 오류는 제한 횟수 재추출, 초과 시 failed |
| decide_review | build_graph_batch | review면 interrupt(승인 해시 검증 후 재개), hold면 보류로 종료 |
| write_graph_batch | verify_graph_batch | 한 트랜잭션. 중단되면 롤백 후 lease 만료 뒤 같은 멱등 키로 재실행 |
| verify_graph_batch | freeze_kg_manifest | 불일치는 failed |
| validate_report_readiness | publish_kg_snapshot | 미달이면 snapshot을 failed로 표시 |

### 3.2 Part B

| 노드 | 다음 | 분기·실패 경로 |
|---|---|---|
| validate_part_a_handoff | freeze_kg_snapshot_and_scope | ready snapshot이 없으면 waiting_part_a |
| assess_coverage | compare_claims | 부족하고 예산이 남으면 retrieve_evidence로 되돌아가 추가 조회 |
| assess_coverage | request_part_a_enrichment | KG 자체 부족이면 waiting_part_a. 보완 후에는 새 snapshot을 고정한 새 실행 |
| assess_coverage | compare_claims | 예산 소진 시 한계를 기록하고 진행 |
| validate_citations_and_claims | save_draft | 통과 |
| validate_citations_and_claims | draft_report | 수정 가능하면 repair_count 한도 내 재작성 후 재검증 |
| human_review | publish_version | 승인(해시 일치 시만) |
| human_review | draft_report | needs_revision |
| publish_version | render_markdown | 결정적 renderer |

### 3.3 재개 규칙
- 검토 노드는 interrupt 이후 처음부터 다시 실행될 수 있으므로 interrupt 앞의 코드는 부작용이 없어야 합니다.
- 중단 신호를 일반 예외 처리로 삼키지 않습니다.
- 같은 `thread_id`로 재개하고 thread_id↔job ID 매핑을 보존합니다.

## 4. 도구 레지스트리(Part B 추가 근거 탐색)

모든 도구는 읽기 전용이며 snapshot·cutoff·search_index_manifest 범위를 **도구 코드가 DB 조건으로 강제**합니다. 모델이 범위 인자를 넘겨도 무시합니다.

| 도구 | 입력 | 출력 | 한도 |
|---|---|---|---|
| search_by_period | 기간, 개체 ID(선택) | claim·evidence ID 목록 | 최대 건수 |
| lookup_entity | 개체 ID 또는 별칭 | 개체·관련 claim ID | |
| find_paths | 시작·끝 개체 ID | 경로(고정 매개변수 SQL 또는 깊이 제한 재귀 CTE) | hop 상한 |
| get_article | article_version_id | 원문 구간·메타 | 구간 길이 |
| get_evidence | evidence_id | 근거 구간·원문 해시 | |
| semantic_search(선택) | 질의 문장 | 후보 ID | pgvector, 동일 snapshot·cutoff·profile |

금지: 임의 SQL, 외부 웹 검색, KG 쓰기, 삭제, 승인, 최종 공개. 허용 도구 목록은 분석 계약 버전에서 읽고, Agent가 넓히지 못합니다. 도구가 반환한 ID는 하네스가 snapshot 소속을 재확인한 뒤에만 evidence_ids에 반영합니다.

### 4.1 Agent 위치 결정(제안)

`create_agent`로 만든 제한된 Agent를 **`retrieve_evidence` 노드 안에서 호출하는 하위 실행**으로 둡니다. 노드가 입력(미해결 sub_question·남은 예산·허용 도구)을 넘기고 Agent의 결과를 `evidence_ids`·`search_trace`·sub_question 상태 변경분으로 변환해 반환합니다.

- 이유: Agent 루프를 상위 StateGraph에 펼치면 상태·체크포인트가 도구 호출 단위로 쌓여 재개 규칙이 복잡해지고, 예산 강제 지점이 분산됩니다.
- 대안: 상위 StateGraph에 tool-call 루프를 직접 둡니다. 도구 호출마다 체크포인트와 interrupt가 가능하지만 상태가 커집니다.
- 이 결정은 구현 전에 소규모로 재개·예산 초과 동작을 확인한 뒤 확정합니다(**미검증**).
- Part A 추출은 `with_structured_output` 고정 경로이며 Agent가 아닙니다.

도구 미지원 모델은 실행 전 검사에서 거부하거나 명시적 고정 워크플로로 선택하며 조용한 전환은 없습니다.

## 5. 하네스

하네스는 노드·모델·도구 호출을 감싸는 래퍼로 구현하며 모델 출력과 무관하게 강제됩니다.

| 시점 | 검사 | 위반 시 |
|---|---|---|
| 실행 전 | 모델의 도구·구조화 출력 지원, 계약·snapshot 상태(ready), 고정 필드 존재 | 실행 거부 |
| 모델 호출 전 | 토큰·시간 예산 잔량 | 한계 기록 후 다음 단계 또는 failed |
| 도구 호출 전 | 도구가 계약의 허용 목록에 있음, hop·기사 수·호출 횟수 한도 | 호출 거부, 거부 기록 |
| 도구 호출 후 | 반환 ID의 snapshot·cutoff 소속 | 해당 결과 폐기 |
| 모델 호출 후 | 출력 스키마(Pydantic), 고정 필드 덮어쓰기 여부 | 제한 재시도 또는 failed |
| 초안 후 | 문장별 인용 존재, 인용 원문 구간의 의미 지지(별도 검사), 한계 표기 | repair 또는 needs_revision |
| 승인 시 | 승인 해시(보고서 content·인용 manifest·계약·온톨로지·snapshot)와 대상 일치 | 승인 거부 |
| 매 단계 | search_trace·사용량·모델/프롬프트 버전 기록 | 기록 실패는 failed |

예산은 `budget` 상태 필드에 누적하고, 하네스만 증가시킵니다. 노드와 Agent는 읽을 수만 있습니다.

## 6. 미결정·미검증

- Agent 위치(4.1)와 재개 동작은 구현 전 소규모 확인이 필요합니다.
- processing_jobs 상태값은 아직 정본에 없습니다.
- sequences 10의 공개 보고서 철회가 사전 dump보다 앞서는지 순서를 확인해야 합니다.
- 상태 필드 이름·reducer는 구현 시 조정될 수 있으며 라이브러리 동작은 검증하지 않았습니다.
