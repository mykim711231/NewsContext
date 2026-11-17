# 논문·GitHub 참조 정의와 LangChain/LangGraph 구현 계획

확인일: 2026-11-13. 상태: 논문 메타데이터·방법론과 공개 저장소의 정적 검토. 이 문서의 함수·실험·일정은 구현 계획이며 원본 코드 실행·성능 재현·라이선스 법률 검토를 완료한 것이 아닙니다. 참고 링크를 나열하는 데 그치지 않고 적용 요소→구현 위치→검증→채택 조건을 정의합니다.

실제 확인한 파일·함수·의존성·라이선스와 확인 한계는 [소스 감사 기록](source-audit.md)에 구분했습니다. Multi-Agent와 OntoExtend 등 논문 본문이 연결한 추가 저자 저장소도 기록합니다.

## 1. 참조 원칙과 링크 교정

Part A는 온톨로지·KG 구축, Part B는 고정된 KG를 활용한 심층 보고서입니다. 사용자가 제시한 학습 순서는 보존하되 논문이 원래 수행하는 과제와 우리 시스템에서 적용할 단계를 구분합니다. EventKG의 모델과 뉴스 인과 주장 추출은 Part A에도 적용하고 Part B에서는 그 결과를 사용합니다.

- **AutoSchemaKG:** 제공된 `2604.11755`는 [블랙홀 관련 논문](https://arxiv.org/abs/2604.11755)입니다. 저장소가 연결하는 원 논문은 [2505.23628](https://arxiv.org/abs/2505.23628), 최초 공개 2025년입니다. 2026 학회 자료와 preprint 연도를 구분합니다.
- **Text2KGBench:** 제공된 `amazon-science/text2kgbench`는 이번 확인에서 열리지 않았습니다. 논문이 연결하는 [cenguix/Text2KGBench](https://github.com/cenguix/Text2KGBench)를 참조 대상으로 사용합니다. 원 URL의 삭제/이전 이유는 확정하지 않습니다.
- **ToG:** 제공된 [IDEA-FinAI/ToG](https://github.com/IDEA-FinAI/ToG)는 확인 시 [DataArcTech/ToG](https://github.com/DataArcTech/ToG)로 연결됩니다. 구현 착수 시 저장소와 commit을 다시 고정합니다.
- **TLogic:** preprint 2021년, AAAI 발표 2022년을 구분합니다. **AnoT:** 2024 preprint와 저장소의 2025 학회 표기를 구분합니다.
- **시계열 인과추론 survey와 Tigramite:** 관련 논문과 구현 라이브러리의 연결이며 해당 survey의 동일 공식 구현으로 표시하지 않습니다.

임베딩은 필요한 기능에 조건부 도입합니다. 검색·정합화 후보 생성에는 활용할 수 있지만 유사도를 근거·개체 동일성·인과효과의 증명으로 사용하지 않습니다. 저장·모델·비용·snapshot 정책은 [임베딩 설계](embedding-retrieval.md)를 따릅니다.

구현 순서는 임베딩 없는 종단 기준선이 먼저입니다. 2년·3개 신문사 후보 corpus의 접근성·선별 기사·보고서 성과를 측정하고 동의 표현 검색 누락 등 필요성이 확인된 EXP만 실행합니다. 참조 논문이 embedding을 쓴다는 사실만으로 제품에 바로 추가하지 않습니다. [규모·성과 검토](corpus-feasibility.md)

## 2. Part A 논문별 적용 정의

| ID·학습 순서 | 논문·저장소 | 참고할 요소 | NewsContext 변형·구현 위치 | 채택 수준·검증 |
|---|---|---|---|---|
| A01·1주 | [LLMs4OL, 2023](https://arxiv.org/abs/2307.16648) · [GitHub](https://github.com/HamedBabaei/LLMs4OL) | 용어 타입·계층·비계층 관계를 나눈 과제 | Part A `ontology/design.py`: 뉴스 도메인 타입/관계 후보를 별도 구조화 출력으로 생성 | 핵심 과제 정의 참조. 원 도메인 점수를 한국어 뉴스 점수로 사용하지 않음 |
| A02·2주 | [Multi-Agent Ontology Generation, 2026](https://arxiv.org/abs/2604.23090) | Domain Expert→Manager→Coder→QA의 산출물 중심 설계와 선행 기획 | Part A 후보 설계→검증→수정 루프를 StateGraph 노드로 분리, 단일 호출 baseline과 비교 | 역할 분리 비교 실험. 원형 OWL/SPARQL·vector RAG 평가와 우리 property graph 평가 구분 |
| A03·2주 | [AutoSchemaKG, 2025 preprint](https://arxiv.org/abs/2505.23628) · [GitHub](https://github.com/HKUST-KnowComp/AutoSchemaKG) | entity/event 추출과 conceptualization에 의한 스키마 유도 | `ontology/design.py`, `extraction/extractor.py`: 승인 스키마 밖 후보를 별도 큐에 축적·개념화 | 후보 발견 적용. 동적 스키마를 운영 그래프에 무승인 자동 반영하지 않음 |
| A04·3주 | [CQ4OE, 2026](https://arxiv.org/abs/2609.26029) · [GitHub](https://github.com/oeg-upm/cq4oe-benchmark) | CQ와 타입·속성·공리의 provenance, 용어/구조 평가 분리 | `analysis/validation.py`, `evaluation/`: 질문→필요 타입/관계→실행 질의→정답/한계의 추적표 | 평가 설계 핵심. SQL 그래프 조회 기반 변형은 OWL benchmark 공식 재현으로 부르지 않음 |
| A05·4주 | [EDC, 2024](https://arxiv.org/abs/2404.03868) · [GitHub](https://github.com/clear-nus/edc) | Extract→Define→Canonicalize와 관련 스키마 검색 | `extraction/`, `ontology/canonicalization.py`: 추출·관계 정의·표준화 분리, 필요시 embedding top-k | 우선 참조. 고정 승인 스키마 baseline과 개방 후보+정합화 비교; 근거·시간·양태 추가 |
| A06·4주 | [KGGen, 2025](https://arxiv.org/abs/2502.09956) · [GitHub](https://github.com/stair-lab/kg-gen) | graph aggregation·개체/관계 cluster로 희소성 완화, MINE 평가 | `ontology/canonicalization.py`: 병합 후보 생성과 검토, `evaluation/` 유용성 과제 | 비교 실험. 과잉 병합·원문 근거 유실·시간 충돌을 별도 평가 |
| A07·5주 | [OntoExtend, 2026](https://arxiv.org/abs/2607.17963) · [저자 GitHub](https://github.com/dersuchendee/OntoExtend) | CQ→관련 기존 온톨로지 검색→확장 후보→통합 | Part A 스키마 검색에 임베딩 조건부 도입, fragment 생성·검증·승인 후 새 버전 | 요구사항 기반 확장 참조. 원형 OWL/SHACL·FAISS 검색과 pgvector property graph adapter 변형을 구분 |
| A08·5주 | [Text2KGBench, 2023](https://arxiv.org/abs/2308.02357) · [확인된 GitHub](https://github.com/cenguix/Text2KGBench) | 온톨로지 준수·추출 품질·환각 평가 | `evaluation/`: 타입·domain/range·근거 충실도·부정/전망 검사 | 평가 핵심. 외부 영문 정답 데이터와 자체 한국어 뉴스 정답셋을 분리 |

## 3. Part B 논문별 적용 정의

| ID·학습 순서 | 논문·저장소 | 원래 과제·참고 요소 | 우리 적용 위치·변형 | 결정 |
|---|---|---|---|---|
| B01·6주 | [EventKG, 2018](https://arxiv.org/abs/1804.04526) | 사건 중심 temporal KG·시간 관계 모델 | Part A Event/Time 계약·manifest, Part B 사건 타임라인 | 데이터 모델 핵심 참조; EventKG 데이터 자체를 뉴스 원문 대신 사용하지 않음 |
| B02·7주 | [AnoT, 2024](https://arxiv.org/abs/2408.00872) · [GitHub](https://github.com/zjs123/ANoT) | rule graph 기반 online anomaly detection | 로컬 분석 실험에서 비정상/새 관계의 검토 우선순위·설명 경로 생성 | 2차 실험. anomaly를 사실 오류·허위 뉴스·시장 급변과 동일시하지 않음 |
| B03·7주 | [Bayesian Online CPD, 2007](https://arxiv.org/abs/0710.3742) | 수치 관측열의 run-length posterior·변화점 | 뉴스량·보도 반응 비율 등 명시적으로 집계한 **보도 지표**만 대상으로 실험 | 2차 실험. 충분한 기간·노출량·계절성·결측 모델 필요; 시장 가격 변화점이라고 표현하지 않음 |
| B04·8주 | [TLogic, 2021/2022](https://arxiv.org/abs/2112.08025) · [GitHub](https://github.com/liu-yushan/TLogic) | temporal random walk·시간 논리 규칙·link forecasting | 과거 시점 규칙의 설명 경로·가설 생성 참고, 보고서 derived/hypothesis로 구분 | 2차 실험. 미래 예측 과제 재현은 별도이며 규칙 confidence를 인과 확률로 표시하지 않음 |
| B05·8주 | [TGB 2.0, 2024](https://arxiv.org/abs/2406.09639) · [GitHub](https://github.com/shenyangHuang/TGB) | 시간 그래프 link prediction benchmark·재현 평가 | 시간 분할·음성 샘플·baseline 원칙만 현재 평가에 참고 | 방법론 참조. 대규모 원 데이터·예측 모델은 무료 DB/보고서 MVP에 적재하지 않음 |
| B06·9주 | [GraphRAG, 2024](https://arxiv.org/abs/2404.16130) · [GitHub](https://github.com/microsoft/graphrag) | KG와 커뮤니티 요약·local/global 검색·종합 | Part A KG 기반 로컬 분석 산출물/커뮤니티 요약, Part B 절별 근거→부분 답변→보고서 종합 | 핵심 아이디어 적용·검색 실험. 완제품 인덱서/스토리지를 중복 도입하지 않음 |
| B07·9주 | [Think-on-Graph, 2023](https://arxiv.org/abs/2307.07697) · [제공 GitHub](https://github.com/IDEA-FinAI/ToG) | LLM이 KG의 entity/relation 경로를 선택·확장·판정 | Part B `research/tools.py`, `agent.py`: snapshot 제한 고정 SQL 그래프 도구·budgeted 경로 탐색(3 hop 이상 가변 경로가 필요하면 README의 Neo4j 재도입 조건 b 검토) | 우선 참조. Freebase/SPARQL 구현은 PostgreSQL 그래프 도구 함수로 adapter 변경 |
| B08·10주 | [Causal KG from News, 2023](https://arxiv.org/abs/2305.09359) | 인과 표현 추출·argument clustering·KG 연결 | 추출·cluster는 Part A, Part B는 근거에 귀속된 원인 설명 비교 | 우선 방법론 참조. reported cause만 표시; cluster 연결을 인과효과로 승격하지 않음 |
| B09·10주 | [Causal Inference survey, 2021](https://arxiv.org/abs/2102.05829) · [관련 Tigramite](https://github.com/jakobrunge/tigramite) | 시계열 causal discovery·effect estimation의 조건·방법 | 보고서 한계와 후속 연구 설계 | 보류. 뉴스만으로 실제 시장 개입 효과를 추정하지 않으며 런타임 의존성 제외 |
| B10·11주 | [MIRAI, 2024](https://arxiv.org/abs/2407.01231) · [GitHub](https://github.com/yecchen/MIRAI) | 뉴스·이벤트 API를 사용하는 temporal forecasting Agent 평가 | Part B `search_trace`, cutoff·툴 실행·예산·시간 누수 검증 | 평가/도구 경계 참조. 국제관계 forecasting 점수를 보고서 품질로 대체하지 않음 |

## 4. 참조 방식과 코드 재사용

**방법론 재설계**, **adapter 기반 부분 재사용**, **격리된 원형 재현**을 구분합니다. 기본은 방법론·출력 계약·평가 원칙을 읽고 최신 LangChain/LangGraph와 NewsContext의 저장소를 대상으로 재구성하는 방식입니다. 모든 원본 repo를 requirements에 추가하거나 원본 pipeline을 중첩 호출하지 않습니다.

외부 코드를 복사하거나 설치하기 전 source URL·정확 commit/tag·파일 경로·코드/데이터/모델 license·NOTICE·의존성·실행 조건을 기록합니다. 공개 저장소 또는 논문 라이선스만으로 코드·데이터 전체의 재사용을 허용한다고 판단하지 않습니다. 미확인 항목은 코드 재사용 보류로 남깁니다. 원형 실험은 별도 의존성 환경에서 수행하고 공개 Render 이미지에 연구용 GPU·대형 모델·외부 benchmark를 포함하지 않습니다.

구현 단계에 `configs/references.yaml`을 생성하여 `reference_id`, paper_url/version, repo_url, commit_sha, inspected_paths, license_status, adopted_elements, excluded_elements, embedding_requirement, target_modules, experiment_id, verification_status를 관리합니다. 현재 문서 확인은 commit 고정·실행 성공을 대신하지 않습니다.

## 5. 최신 LangChain/LangGraph로의 연결

| 프로세스 | LangChain 역할 | LangGraph/프로그램 역할 | 주요 참조 |
|---|---|---|---|
| 온톨로지 초안 | 모델 구조화 출력·관련 스키마 검색 | 후보 상태·검증·제한 수정·사람 승인 | A01/A02/A03/A07 |
| 기사 KG | Extract/Define 구조화 호출·embedding 후보 검색 | 근거·시간·타입 검사·정합화 검토·멱등 적재 | A05/A06/A08/B08 |
| 계약 평가 | 필요시 보조 LLM 판정 | CQ→질의·정답 비교·domain/range·실행 로그 | A04/A08 |
| KG 인계 | 모델 호출 불필요 | manifest hash·snapshot ready 검사 | B01, 자체 저장 계약 |
| 보고서 근거 탐색 | `create_agent` 또는 tool-call 모델과 제한된 읽기 도구 | StateGraph의 예산·경로·충분성·중단/재개 | B06/B07/B10 |
| 보고서 작성 | 근거 묶음→절별 구조화 출력→종합 | 문장별 인용·양태·상충·cutoff 검증·검토/공개 | B06/B08, 자체 보고서 계약 |

고정 추출은 `init_chat_model`와 `with_structured_output`을 기본으로 하고, 자율 도구 선택이 필요한 경로에 `create_agent`를 사용합니다. 모델 객체의 기능·provider-native 구조화 출력/ToolStrategy는 공급자별 계약 검사로 확인합니다. 원본 저장소의 오래된 SDK import·model ID를 복사하지 않습니다. [LangChain](https://docs.langchain.com/oss/python/langchain/overview), [구조화 출력](https://docs.langchain.com/oss/python/langchain/structured-output), [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)

## 6. 비교 실험과 도입 게이트

| 실험 | 비교 | 채택 판단 |
|---|---|---|
| EXP-A1 | 단일 후보 생성 vs 산출물 중심 설계/검증 역할 분리 | CQ 충족·구조 오류·검토 수정률·비용 |
| EXP-A2 | 승인 스키마 추출 vs EDC식 정의/정합화 보강 | 근거 지지 precision/recall·미지원 관계·환각 |
| EXP-A3 | 별칭/키워드 후보 vs embedding 후보 vs KGGen식 후보 cluster | 동일 대상 recall·잘못된 병합·시간 충돌·비용 |
| EXP-A4 | 스키마 재생성 vs OntoExtend식 기존 구조 재사용·확장 | 기존 CQ 회귀·새 CQ 충족·불필요 타입·변경 크기 |
| EXP-B1 | 단순 요약 vs 고정 KG 경로 검색 vs ToG식 제한 탐색 | 질문 충족·원문 의미 지지·상충 포괄·검색 비용 |
| EXP-B2 | 키워드/graph vs embedding hybrid·community 부분 요약 | 근거 recall@k·최종 주장 precision·시간 누수·지연·DB 용량 |
| EXP-B3 | 고정 규칙 vs TLogic/AnoT·뉴스 집계 CPD | 해석 가능성·거짓 경보·시간 분할·baseline 대비 개선 |

동일 corpus snapshot·개발/보류 분할·기본 모델·전체 예산을 고정하고 방법만 바꿉니다. 생성 변동은 반복 실행으로 기록하며 과잉 병합과 인과 과장을 평균 점수로 숨기지 않습니다. 임베딩·모델 변경은 별도 변수입니다. 초기 수치 목표는 파일럿 후 확정하고 외부 논문의 수치를 우리 결과로 기재하지 않습니다.

## 7. 학습 일정과 구현 일정의 관계

제공된 1~11주는 **논문 읽기·원형 이해 순서**로 채택합니다. 기존 6주 개발 초안은 핵심 MVP 범위이며 논문 18편의 전체 재현까지 6주에 완료한다는 약속이 아닙니다. 각 주의 산출물은 읽기 요약이 아니라 source audit·적용/제외 판단·최소 실험 설계·작은 fixture입니다.

개발 게이트는 G1(CQ·온톨로지) → G2(추출·근거·정합화) → G3(snapshot 인계) → G4(보고서 경로 탐색·작성) → G5(검색/역할 개선 비교) → G6(시간 규칙·이상/변화 탐색 실험)입니다. G1~G4를 먼저 완성하고 예측·causal inference 원형 재현은 후속 연구로 분리합니다.

## 8. 완료 상태의 표현

현재는 논문·README·일부 핵심 파일을 확인한 **참조 계획 단계**입니다. `읽음`, `소스 확인`, `license 확인`, `commit 고정`, `설치`, `원형 실행`, `변형 구현`, `평가 통과`를 별도 상태로 관리합니다. 실제로 수행하지 않은 단계는 완료로 표시하지 않습니다. 코드·데이터·모델의 허용 범위를 검토하고 같은 실험을 재현할 수 있을 때만 구현 참조가 완료된 것으로 봅니다.
