# 조건부 임베딩·하이브리드 검색 설계

상태: 기획. 사용자 지시에 따라 기존 임베딩 제외 방침을 변경하여 **필요한 기능에 임베딩을 허용**합니다. 현재 모델 선정·extension 설치·벡터 생성·검색 품질 측정은 수행하지 않았습니다.

최신 실행 순서는 **임베딩 없이 2년·3매체 후보 corpus의 표본과 선별 KG/보고서 성과 확인 → 실패 원인 분석 → 필요한 구간만 임베딩 실험 → 채택/보류**입니다. 아래는 도입 필요성이 확인됐을 때 사용할 설계이며 처음부터 pgvector·embedding 모델을 설치해야 한다는 뜻이 아닙니다. 자세한 규모·성과 검토는 [corpus 타당성 계획](corpus-feasibility.md)을 따릅니다.

## 1. 사용 위치

| 구간 | 임베딩 용도 | 지켜야 할 경계 |
|---|---|---|
| Part A 온톨로지 | CQ와 관련된 기존 class/relation fragment 검색 | 유사 스키마를 후보로만 사용; 새 계약 승인 필요 |
| Part A 개체/관계 정합화 | 별칭·표현이 다른 mention의 top-k 후보 생성 | 자동 병합 금지; ID·타입·시간·원문 근거 재검증 |
| Part B 보고서 | 질문과 관련된 고정 snapshot 내 기사 chunk·주장·요약 검색 | cutoff·프로젝트·원문/주장 버전·snapshot membership 필터 적용 |

기본 경로는 키워드·별칭·Cypher이며 검색 누락이 확인된 경우 의미 검색과 결합해 비교합니다. 임베딩은 판단의 근거를 대신하지 않고 검색 후보를 찾는 보조 정보입니다. KG를 변경하거나 시장 인과효과를 추정하는 기능이 아닙니다.

## 2. 저장소와 패키지

우선 **기존 Supabase PostgreSQL의 pgvector**를 사용해 별도 벡터 서버의 운영을 피합니다. vector extension을 준비하고 업무 스키마의 embedding 테이블을 명시적 SQL 마이그레이션으로 생성합니다. 기본 PK·인덱스와 프로그램 논리 참조 원칙을 유지합니다. [Supabase pgvector](https://supabase.com/docs/guides/database/extensions/pgvector)

LangChain embedding interface와 선택 공급자 adapter로 벡터를 생성하고 psycopg 저장·조회 함수 또는 custom retriever/tool로 연결합니다. `langchain-postgres`의 편의 vectorstore는 자동 생성 테이블·FK·마이그레이션·버전 필터가 프로젝트 원칙에 맞는지 확인하기 전 바로 도입하지 않습니다. [LangChain PGVector](https://docs.langchain.com/oss/python/integrations/vectorstores/pgvector), [Embedding integrations](https://docs.langchain.com/oss/python/integrations/embeddings)

FAISS 등 원 연구 구현의 저장소는 격리 원형 실험에서 비교할 수 있으나 제품 기본 구성과 섞지 않습니다. 대형 embedding 모델은 로컬 작업 환경에서만 실행하고 Render 공개 조회 이미지에 모델 가중치를 포함하지 않습니다.

## 3. 모델과 버전 계약

대화 LLM 공급자와 embedding 공급자는 별도 설정입니다. Anthropic·DeepSeek 등의 채팅 설정에서 embedding 지원을 추정하지 않습니다. 초기 후보는 API 제공 embedding 또는 한국어 품질을 확인한 로컬 모델이며, 모델 ID·revision·라이선스·차원·거리 함수·정규화·chunk 버전을 평가 후 고정합니다.

같은 vector column/index에는 동일한 모델·차원·정규화 공간의 벡터만 넣습니다. 모델이 달라지면 새 embedding profile/table과 재색인 작업을 만들고 기존 벡터와 혼합 검색하지 않습니다. `EMBEDDING_DIMENSIONS`는 실제 출력·column 타입과 대조합니다. API 키는 local_admin에서만 검증하고 public_demo에서는 embedding 설정을 요구하지 않습니다.

`embedding_profiles`, `text_chunks`, `text_embeddings`를 업무 스키마에 두고 source_type·source_version_id·content_hash·chunk_offset·profile_version·vector·status를 기록합니다. `UNIQUE(source_type, source_version_id, chunk_hash, profile_version)`로 타입 간 ID 충돌과 재색인 중복을 막습니다. 각 source_type별 부모 존재·프로젝트/버전 소속 검증 함수를 적용하고 chunk 텍스트·offset·해시는 불변으로 보존합니다. 삭제와 삽입은 같은 부모 잠금·reset generation/실행 토큰 규칙을 사용하여 초기화 뒤 늦게 도착한 embedding 저장을 차단합니다.

KG manifest와 검색 인덱스 manifest는 분리합니다. Part A의 `kg_snapshot_id`는 사실·원문·근거 집합을 고정하고, 별도 `search_index_manifest_id`는 그 KG/승인 스키마에 대한 chunk 목록·profile·index 버전·상태를 기록합니다. 뒤늦은 재색인이 기존 KG snapshot을 수정하지 않습니다. Part B는 embedding 활성 시 ready인 검색 manifest와 profile을 실행/보고서 생성 이력에 고정합니다. 같은 KG라도 검색 profile이 바뀌면 새 검색 manifest·새 보고서 실행으로 비교하며 미완료 index를 사용하지 않습니다.

## 4. 검색 흐름과 필터

```text
질문·승인 계약·snapshot 검증
 → 구조화 필터·키워드/별칭·Cypher 후보
 → embedding 활성 시 질문 벡터·동일 profile 검색 (허용 ID·시간 조건 DB 사전 필터)
 → snapshot/cutoff/프로젝트 소속 재검증
 → 중복 제거·순위 결합 → 원문/Claim/Evidence 재조회
 → 인용·의미 지지·상충 검증 → 보고서 작성
```

벡터 유사도는 확률로 표시하지 않습니다. DB 쿼리에 project·source_type·source_version membership·cutoff·profile·ready 검색 manifest를 조건으로 넣고 반환 후 다시 소속을 검증합니다. 전체 데이터의 top-k를 먼저 받은 뒤 금지 자료만 버리는 방식으로 권한 경계를 구현하지 않습니다. 필터링이 강한 소규모 데이터에서는 정확 거리 검색으로 시작하고 필요 시 HNSW/IVFFlat을 실측 비교합니다. ANN의 필터 조건 때문에 recall이 줄어들 수 있으므로 top-k 확대·필터 전용 쿼리·정확 검색 기준 결과와 비교합니다. [벡터 인덱스 안내](https://supabase.com/docs/guides/ai/vector-indexes)

public_demo는 사전 생성한 보고서 조회만 하므로 방문자 요청에 embedding API를 호출하지 않습니다. 향후 공개 의미 검색을 추가하는 것은 별도 비용·보안·캐시 설계가 필요한 범위 변경입니다.

## 5. 용량·백업·재색인

원문·checkpoint와 같은 Free DB 공간을 사용합니다. 예를 들어 float32 1,536차원×10,000 chunk의 벡터 payload만 약 61.44MB(약58.6MiB)이며 행·메타데이터·중복 profile·인덱스 공간은 별도입니다. 차원·기사 수·chunk 길이·중복 버전·index를 파일럿으로 측정하고 한도 접근 시 전체 원문 자동 embedding을 멈춥니다.

초기에는 핵심 스키마·선택 뉴스 범위만 색인합니다. 임베딩 변경은 독립 job으로 실행하고 완료한 profile만 검색에 노출합니다. text chunk는 원문 offset과 해시를 보존하고 원문 정정 시 새 버전에만 생성합니다.

업무 스키마 dump는 chunk·profile·vector 행과 인덱스 정의를 포함하며 복원 환경에는 호환 pgvector extension이 먼저 필요합니다. 모델 가중치·API·외부 artifact는 dump에 포함되지 않습니다. profile의 재생성 설정과 원문으로 재색인할 수 있어야 합니다. 삭제 시 원문·Claim·snapshot·보고서 의존성을 대조하고 공유되지 않은 embedding만 제거합니다.

## 6. 활성화 기준

기본 feature flag는 false로 두고 schema retrieval·entity 후보·report retrieval를 각각 평가합니다. 자체 한국어 보류셋에서 lexical/graph baseline 대비 근거 recall·질문 충족이 개선되고 잘못된 병합·최종 주장 오류가 악화되지 않는지 확인합니다. 함께 토큰·embedding 비용·p95 지연·DB 크기를 기록합니다. 게이트를 통과한 해당 기능만 켜고 실패 시 baseline을 유지합니다. 이 기본 비활성은 임베딩 금지가 아니라 미검증 기능의 선택적 도입 정책입니다.
