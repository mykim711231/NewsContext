# 참조 GitHub 소스 감사 기록

확인일: 2026-11-13. 전문 검토 에이전트가 공개 README·일부 핵심 파일·라이선스 표시를 읽고 교차 확인했습니다. clone·설치·실행·재현은 수행하지 않았습니다. 아래 경로는 확인 당시 branch 기준이며 실제 구현 착수 시 commit SHA와 파일 내용을 다시 고정해야 합니다. 라이선스 표시 확인은 코드·데이터·모델 전체의 재사용 권한 확정이나 법률 검토가 아닙니다.

적용/제외·평가·학습 계획의 기준 문서는 [참조 구현 계획](reference-implementation-plan.md)입니다. 아래에는 원본 식별자와 NewsContext의 새 함수 이름을 혼동하지 않도록 실제 확인한 소스 경계를 기록합니다.

## 1. Part A 소스

| 원본·확인 경로 | 실제 역할·함수 | 재사용 경계·license 상태 |
|---|---|---|
| [LLMs4OL TaskA/B/C](https://github.com/HamedBabaei/LLMs4OL) | task별 configuration/datahandler/src·test.py·evaluator.py; 타입·계층·비계층 관계 평가 | MIT LICENSE 확인. TaskB/build_hierarchy.py는 기준 계층 준비 코드이며 LLM 생성 함수로 오인하지 않음 |
| [Multi-Agent ontology 저자 코드](https://github.com/brains-group/towards_automated_ontology_generation) | agents/cq_generator.py, multi_agent.py, ontology_fixer.py, eval.py, vector_eval.py가 README에 안내됨 | 코드 LICENSE 별도 확인 못함. 논문 CC BY 4.0을 코드 라이선스로 간주하지 않음; 역할/산출물 방법론 우선 참조 |
| [AutoSchemaKG triple_extraction.py](https://github.com/HKUST-KnowComp/AutoSchemaKG/blob/main/atlas_rag/kg_construction/triple_extraction.py) | DatasetProcessor.create_sample_chunks; KnowledgeGraphExtractor.process_stage/run_extraction/generate_concept_csv_temp/compute_kg_embedding/create_faiss_index | MIT 확인. 추출과 embedding/FAISS 단계는 분리; ATLAS 대규모 데이터·무거운 전체 stack은 도입하지 않음 |
| [AutoSchemaKG concept_generation.py](https://github.com/HKUST-KnowComp/AutoSchemaKG/blob/main/atlas_rag/kg_construction/concept_generation.py) | build_batched_entities, build_batched_relations, generate_concept | 개념 후보 생성 참조. 승인 ontology 자동 교체 금지 |
| [CQ4OE 사용 절차](https://github.com/oeg-upm/cq4oe-benchmark/blob/main/usage_instructions.md) | CQ2Term/scripts/eval_cq_terms.py, run_all_cq2term.py; CQ2Onto/scripts의 concept/property/triple/axioms/hierarchy | Apache-2.0 확인. semantic metric·HermiT 등 평가 의존성은 별도; runner 이름 차이는 commit 고정 후 확인 |
| [EDC extract.py](https://github.com/clear-nus/edc/blob/main/edc/extract.py) · [schema_definition.py](https://github.com/clear-nus/edc/blob/main/edc/schema_definition.py) | Extractor.extract, SchemaDefiner.define_schema | MIT 확인. 현재 graph 계약의 근거·시간·양태를 추가해야 함 |
| [EDC schema_canonicalization.py](https://github.com/clear-nus/edc/blob/main/edc/schema_canonicalization.py) | SchemaCanonicalizer.retrieve_similar_relations, llm_verify, canonicalize; SchemaRetriever·edc_framework가 조립 | embedding top-k→LLM 선택/거절. 관계 정합화 중심이며 entity coreference를 자동 해결한다고 주장하지 않음 |
| [KGGen kg_gen.py](https://github.com/stair-lab/kg-gen/blob/main/src/kg_gen/kg_gen.py) | KGGen.generate/aggregate/deduplicate/generate_embeddings/retrieve/retrieve_relevant_nodes/retrieve_context; get_entities/get_relations/run_deduplication 호출 | pyproject MIT 선언, 루트 LICENSE 본문 미확인. DSPy/LiteLLM·SEMHash·sentence-transformers 의존, 전체 통합보다 비교 adapter |
| [OntoExtend 저자 저장소](https://github.com/dersuchendee/OntoExtend) | README의 OntoExtend Script/rag-ontoextend.py, embedder.py, llm_service.py; 검색→fragment→통합 | README Apache-2.0 선언. sentence embedding·FAISS 기반 원형과 pgvector adapter 변형 구분; 실제 복사 전 LICENSE/NOTICE 확인 |
| [Text2KGBench run_eval.py](https://github.com/cenguix/Text2KGBench/blob/main/src/evaluation/run_eval.py) | calculate_precision_recall_f1, get_subject_object_hallucinations, get_ontology_conformance, normalize_triple | 코드 Apache-2.0 확인. 데이터 README badge/본문의 CC BY vs ShareAlike 문구 상충은 별도 확인 필요 |

Text2KGBench의 해당 conformance 함수는 관계 라벨 membership 검사이며 전체 domain/range 논리 검증이 아닙니다. 영어 PorterStemmer·문자열 기반 환각 검사를 한국어 원문 의미 지지 검사로 대체하지 않습니다. CQ4OE의 OWL 공리·HermiT 평가도 Neo4j property graph의 JSON schema 검증과 동일하지 않습니다.

## 2. Part B 및 교차 적용 소스

| 원본·확인 경로 | 실제 역할·함수 | 재사용 경계·license 상태 |
|---|---|---|
| [EventKG 저자 pipeline](https://github.com/sgottsch/eventkg) | 기존 KG·반구조 자료에서 event/time/provenance를 통합하는 extraction pipeline | MIT 표시, dataset은 별도 CC BY 4.0 안내. 뉴스 LLM 추출기 자체가 아님 |
| [ANoT anomaly_detector.py](https://github.com/zjs123/ANoT/blob/main/anomaly_detector.py) · model_updater.py | AnomalyDetector.conceptual_score/score_blame_edge/update; ModelUpdater.update_model/update_temporal_model | LICENSE 확인 못함. NumPy/SciPy/NetworkX rule graph; embedding 필수 아님, 코드 복사 보류 |
| [TLogic rule_learning.py](https://github.com/liu-yushan/TLogic/blob/main/mycode/rule_learning.py) · rule_application.py | Rule_Learner.create_rule/estimate_confidence; filter_rules/get_window_edges/match_body_relations | Apache-2.0. pyproject Python >=3.9,<3.11 제약 확인; NewsContext 3.11+와 별도 재현 환경 필요 |
| [TGB evaluate.py](https://github.com/shenyangHuang/TGB/blob/main/tgb/linkproppred/evaluate.py) · negative_sampler.py | Evaluator·NegativeEdgeSampler, 시간 순서와 링크 평가 | 코드 MIT, 데이터별 별도. 대규모 benchmark를 제품 DB로 가져오지 않음 |
| [GraphRAG community reports](https://github.com/microsoft/graphrag/blob/main/packages/graphrag/graphrag/index/workflows/create_community_reports.py) | run_workflow/create_community_reports·context build·summarize_communities | MIT. 확인 당시 packages 모노레포·README maintenance mode; 변경 이력 고정 필요 |
| [GraphRAG global search](https://github.com/microsoft/graphrag/blob/main/packages/graphrag/graphrag/query/structured_search/global_search/search.py) | GlobalSearch.search·_map_response_single_batch·_reduce_response | community report map/reduce 자체는 vector 검색 필수가 아님 |
| [GraphRAG local mixed context](https://github.com/microsoft/graphrag/blob/main/packages/graphrag/graphrag/query/structured_search/local_search/mixed_context.py) | LocalSearchMixedContext는 entity_text_embeddings·text_embedder 입력 사용 | 원본 local search를 embedding 없이 그대로 사용한다고 쓰지 않음 |
| [ToG freebase_func.py](https://github.com/DataArcTech/ToG/blob/main/ToG/freebase_func.py) | relation_search_prune/entity_search/entity_score/entity_prune/reasoning/generate_answer; width/depth 제한 | README Apache-2.0 선언, 독립 LICENSE 본문 미확인. 원 SPARQL은 snapshot 제한 Cypher로 교체 |
| [Tigramite pcmci.py](https://github.com/jakobrunge/tigramite/blob/master/tigramite/pcmci.py) · causal_effects.py | PCMCI.run_pcmci/run_pcmciplus; CausalEffects | GPL-3.0. 숫자 시계열·가정 필요, 현재 runtime 제외 |
| [MIRAI api_implementation.py](https://github.com/yecchen/MIRAI/blob/master/APIs/api_implementation.py) | Date/DateRange/set_default_end_date/count_events/get_events; get_embedding·BM25·embedding 파일 | root LICENSE 미확인·README research purposes only. langchain0.2.3/openai0.27.4 등 원 requirements 그대로 병합하지 않음 |

ToG pruning은 LLM/BM25/sentencebert 선택이 가능하지만 원 파일은 sentence_transformers를 상단 import합니다. 선택 알고리즘과 설치 의존성을 구분합니다. MIRAI의 전역 DEFAULT_END_DATE는 여러 작업에 공유하지 않고 요청별 불변 context로 바꾸며 원본의 임의 코드 실행 경로 대신 허용된 tools만 제공합니다.

## 3. 논문만 또는 기반 코드만 확인된 항목

- [뉴스 causal KG 논문](https://arxiv.org/abs/2305.09359)은 패턴·BERT 기반 causal 추출과 embedding argument clustering을 결합합니다. [UniCausal](https://github.com/tanfiona/UniCausal)·[CauseNet](https://github.com/causenet-org/CIKM-20)은 연결된 기반 코드이며 전체 논문의 공식 pipeline 저장소로 단정하지 않습니다. 우리 구현은 LLM 근거 추출+선택 cluster의 별도 변형입니다.
- [BOCPD](https://arxiv.org/abs/0710.3742)는 run-length·hazard·likelihood 기반 알고리즘입니다. 이번 본문 확인에서 원저자 GitHub를 확정하지 않았으며 별도 구현체를 저자 공식 코드로 표시하지 않습니다.
- [시계열 causal survey](https://arxiv.org/abs/2102.05829)는 방법·평가의 개요입니다. Tigramite와 동일한 논문의 단일 구현이라고 연결하지 않습니다.

## 3-1. 추가 필요 후보 (2026-11-13 웹 검색 확인)

기존 감사는 Part A/B 논문 구현 중심이어서 한국어 처리·본문 추출·그래프 라이브러리·로컬 Neo4j 제약이 비어 있었습니다. 아래는 **검색 결과 요약만 확인**한 후보이며 저장소·LICENSE 파일을 직접 읽거나 설치·실행하지 않았습니다. 도입 전 §4 체크포인트를 따릅니다.

| 후보 | 필요한 이유 | 확인된 것 / 미확인 |
|---|---|---|
| [neo4j-graphrag-python](https://github.com/neo4j/neo4j-graphrag-python) | 공식 Neo4j GraphRAG 패키지. retriever·KG 적재 구현 참조 | PyPI 메타데이터 Apache-2.0, Python 3.10~3.14. 버전은 자료마다 1.14.1·1.18.0로 상충해 설치 시 확인. **2026-11-13 사용자 결정: 도입(의존성 채택)으로 정함.** 설치·동작은 아직 미검증이므로 첫 통합 때 `uv.lock`으로 버전을 고정하고, Neo4j Community·AuraDB 양쪽 연결, pydantic·langchain 버전 충돌, 자체 계약(snapshot·근거 수준)과의 맞물림을 확인함. 계약과 맞지 않는 부분은 라이브러리를 감싸 쓰고 계약을 바꾸지 않음 |
| [trafilatura](https://github.com/adbar/trafilatura) | requests+BS4 파서와 비교할 본문 추출기 후보 | 1.8.0 이후 Apache-2.0(이전은 GPLv3+). 한국어 뉴스 성능 근거는 못 찾음 → 출처별 고정 HTML로 직접 비교 |
| [kiwipiepy](https://github.com/bab2min/kiwipiepy) | 한국어 형태소·문장 분리(검색 보조·offset 검증·중복 판정 전처리) | 2026-11-13 확인: 저장소 LICENSE는 Apache-2.0, PyPI 0.24.1의 `license_expression`도 Apache-2.0, README는 모델 파일도 같은 라이선스라고 명시. 0.23.2 이하는 LGPL이므로 0.24 이상만 사용. NOTICE 확인: 모델 파일도 Apache-2.0, `kiwipiepy_model` 0.24.0 PyPI도 Apache-2.0, 모델은 사전·n-gram 통계이며 원 말뭉치를 복원할 수 없다고 명시. 재배포 시 NOTICE의 저작권·라이선스 고지 유지 의무(Eigen MPL-2.0 등 제3자 구성요소 포함). 미확인: 학습 말뭉치(세종·모두의 말뭉치) 제공처 조건이 Apache-2.0 배포를 허용하는지는 NOTICE가 밝히지 않음. 2026-11-13 검색으로도 약관 원문을 찾지 못함(모두의 말뭉치는 온라인 약정서 승인 방식, 목적은 연구·기술 개발로 보도됨, 상업적 이용·모델 재배포 허용 여부는 근거 없음). 약관 원문 또는 국립국어원 문의로만 확정 가능 → 이 프로젝트는 Kiwi를 사용만 하고 모델·말뭉치를 재배포하지 않으며, 상업 서비스화 시 재확인 |
| [KLUE](https://github.com/KLUE-benchmark/KLUE) | 한국어 NER·관계추출 일반 벤치마크 | 금융 뉴스 이슈·반응 추출용이 아니고 이벤트 과제 없음. 사전 점검용 참조에 한정 |
| FinNewsBench | 한국 금융 뉴스 구조화 추출 벤치마크 | AI 생성 기사 기반으로 보고됨, GitHub 링크 미확인. 실제 뉴스 성능 근거로 쓰지 않음 |
| [langchain-postgres](https://github.com/langchain-ai/langchain-postgres) | pgvector 연동 참조 | `PGVector`는 v0.0.14부터 deprecated, `PGVectorStore` 권장. wrapper는 도입하지 않고 psycopg 직접 SQL 유지 |
| [langgraph checkpoint-postgres](https://github.com/langchain-ai/langgraph) | PostgresSaver 버전 고정 | 서드파티 미러에서 3.1.1 확인. 공식 릴리스 페이지로 재확인 필요 |
| [datasketch](https://github.com/ekzhu/datasketch) | 전재·근접 중복 기사 판정(MinHash-LSH) | 2026-11-13 확인: 저장소 LICENSE와 PyPI 2.0.0(Python ≥3.9) 모두 MIT. 뉴스 근접 중복 정의 예: trigram Jaccard ≥ 0.8 |
| [text-dedup](https://github.com/ChenghaoMou/text-dedup) | 중복 제거 방법 비교 참조 | NEWS-COPY 벤치마크에서 MinHash 결과 보고. 의존성 규모·라이선스 직접 확인 전에는 방법 참조에 한정 |
| Gaoya | Rust 기반 MinHash LSH | MIT로 보고됨. Rust 빌드 의존성이 생기므로 MVP에서는 제외 |
| pg_bigm / pg_trgm / mecab-ko+textsearch_ko | PostgreSQL 한국어 검색 | 기본 FTS는 한국어 미지원. pg_trgm은 내장 3-gram, pg_bigm은 2-gram으로 조사 붙은 어절에 유리(AWS 한국어 블로그), mecab-ko는 네이티브 컴파일 필요. **Supabase pg_bigm: 사용자 프로젝트에서 `pg_available_extensions` 조회 결과 0행(2026-11-13) → 사용 불가로 확정.** 공식 문서에 pg_trgm·PGroonga는 있음(PGroonga의 한국어 지원은 미확인). 실제 프로젝트에서 `select * from pg_available_extensions where name = 'pg_bigm';`로 확인 |
| Neo4j Community 제약 | 로컬 그래프 저장소 | 공식 문서 기준 사용자 DB 1개, dump는 오프라인 전용. 설치·동작 검증은 하지 않음 |

**확인하지 못한 것:** 한국 금융 뉴스의 NER·관계·이벤트를 함께 다루는 공개 데이터셋은 찾지 못했습니다. 한국어 추출 성능은 자체 20~40건 평가셋으로만 판단하며 외부 점수로 대신하지 않습니다. 중복 판정 후보는 위 표에 두었으나 어느 것도 채택하지 않았고, 임계값은 자체 평가셋으로 정합니다. kiwipiepy는 저장소 LICENSE·PyPI·README로 Apache-2.0을 확인했으나 datasketch도 저장소 LICENSE·PyPI로 MIT를 확인했습니다.

## 4. 코드 도입 전 체크포인트

원본 파일/commit·license/NOTICE·데이터·모델 허용 범위 → 별도 환경 설치 → 고정 fixture 원형 실행 → 입력/출력 비교 → LangChain/LangGraph adapter 구현 → 자체 뉴스 보류셋 평가 순서로 진행합니다. 라이선스 미확인 코드는 승인된 복사 대상으로 취급하지 않고 방법론만 참고한 독립 구현을 우선합니다. 원본 결과·벤치마크 점수를 우리 보고서 품질이나 한국어 추출 성능으로 재사용하지 않습니다.
