# NewsContext 시퀀스 다이어그램

기획 단계의 목표 호출 흐름입니다. 함수 이름과 API는 구현 계약 초안이며, 대단위 흐름에서 하위 함수 호출로 내려가도록 구성했습니다. Mermaid를 지원하는 Markdown 뷰어에서 확인할 수 있습니다.

최종 산출물은 Part B의 **근거 기반 심층 리서치 보고서**입니다. 아래 기존 KG 생성·탐색 시퀀스는 보고서 근거를 공급하며 보고서 생성·검토·공개 흐름을 13번에 추가했습니다. 공개 사용자는 보고서를 조회하고 생성·재작성은 로컬 관리자와 Worker가 수행합니다.

> **읽는 법(2026-11-17 정리):** 로컬 본체 우선 결정(2026-11-13, [corpus-feasibility §8](corpus-feasibility.md))에 따라 MVP의 정본은 **로컬 PostgreSQL**, 그래프도 **같은 로컬 PostgreSQL의 그래프 테이블**입니다([README 저장소 기준](../README.md), 2026-11-17 Neo4j 제외 결정). 다이어그램 참여자도 이 기준으로 바꿨습니다. `로컬 PostgreSQL`과 `로컬 PostgreSQL 그래프 테이블`이 따로 그려진 곳은 같은 DB의 다른 테이블이며, 그래프 쓰기와 작업 상태 기록은 한 트랜잭션입니다. `Supabase 공개 사본`과 `Render`는 공개 데모(향후) 구성이며, 공개용 export 단계가 로컬 정본에서 선별 적재합니다.

공개 데모(향후)도 그래프를 Supabase PostgreSQL의 그래프 테이블에 두며 AuraDB는 쓰지 않습니다. Python 서버·Worker·초기화 CLI만 DB에 접속하고 PWA는 FastAPI를 통해 접근합니다. DB 초기화는 그래프 테이블 마이그레이션(고유 제약·인덱스)입니다. 개발 삭제·복구는 프로젝트·버전 범위로 한정합니다. `kg_snapshot_id`는 물리 DB 복사본이 아니라 PostgreSQL의 불변 corpus/claim/evidence/batch manifest와 snapshot membership 테이블을 가리킵니다.

아래 추출/정합화는 [EDC·KGGen 등 Part A 참조](reference-implementation-plan.md), 보고서 경로·부분 분석/종합은 ToG·GraphRAG, cutoff 도구는 MIRAI를 참고한 프로젝트용 변형입니다. 필요한 embedding 검색은 별도 로컬 adapter·PostgreSQL pgvector로 수행하며 근거·snapshot 검증을 대체하지 않습니다.

배포 모드는 로컬 관리(`local_admin`, MVP)와 공개 조회 데모(`public_demo`, 향후)로 나눕니다. 아래 수집·Part A·Part B·관리·초기화 시퀀스는 로컬 관리자 API와 로컬 Worker에서만 실행합니다. 공개 데모의 Render에서는 PWA 정적 파일과 읽기 전용 FastAPI만 제공하고 생성·변경 API를 등록하지 않습니다. 모든 Checkpointer는 로컬 PostgreSQL의 `langgraph` 스키마를 가리킵니다.

```mermaid
sequenceDiagram
    actor Visitor as 공개 방문자
    participant Render as Render PWA·조회 API (향후)
    participant Pub as Supabase Free 공개 사본 (향후)
    participant Local as 로컬 관리자·Worker
    participant DB as 로컬 PostgreSQL 정본·그래프 테이블
    participant Backup as 로컬 예약 백업
    participant Disk as 로컬 영속 경로·별도 저장소
    Local->>DB: 원문·분석 계약·추출 결과·그래프 테이블·보고서·체크포인트 저장
    Local->>Pub: 공개 범위 선별 export
    Visitor->>Render: PWA 실행·데모 조회
    Render->>Pub: 데모 범위 고정 읽기 쿼리
    Render-->>Visitor: 공개 보고서·원문 근거·Markdown
    Backup->>DB: newscontext·langgraph 단일 snapshot dump
    DB-->>Backup: 백업 데이터
    Backup->>Disk: 정상 dump·별도 사본 보관
    Note over Render,Disk: Render 로컬 디스크에 체크포인트·dump를 보관하지 않음. 백업 대상은 로컬 정본
```

## 1. 대단위: 뉴스 수집부터 심층 보고서 공개까지

업무 단계는 **Crawler → Part A → Part B**입니다. Part A는 LLM 온톨로지와 분석 계약을 승인하고 기사 추출·근거 검증·정합화·그래프 적재를 수행하여 ready KG snapshot을 발행합니다. Part B는 analysis_contract_version·ontology_version·kg_snapshot_id와 보고서 brief를 입력으로 받습니다. 온톨로지 계약 승인, 추출 주장 검토, 최종 보고서 공개는 별도 결정입니다.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 로컬 관리자
    participant API as 로컬 관리 API
    participant Worker as 로컬 Worker
    participant Source as 뉴스 출처
    participant DB as 로컬 PostgreSQL 정본·실행 상태
    participant A as Part A 온톨로지·KG 구축
    participant B as Part B 보고서 파이프라인
    participant KG as 로컬 PostgreSQL 그래프 테이블
    participant Pub as Supabase 공개 사본 (향후)
    participant Public as Render 공개 조회 API (향후)
    actor Reader as 독자
    Admin->>API: 언론사·기간·페이지 범위 지정
    API->>DB: 수집 작업 등록
    Worker->>DB: 수집 작업 선점
    Worker->>Source: 허용 범위 뉴스 목록·본문 수집
    Source-->>Worker: 기사 HTML
    Worker->>DB: 불변 원문 버전·시각·해시 저장
    Admin->>API: 분석 질문 유형·표본·보고서 요구 지정
    API->>A: 분석 계약 후보 생성·검증 요청
    A->>DB: 표본·기존 승인 온톨로지 조회
    A->>A: 온톨로지·근거/반례·절별 출력·도구/예산 계약 검증
    A->>DB: 계약 후보·해시·검증 결과 저장
    Admin->>API: 대상 해시를 확인해 계약 승인
    API->>DB: 불변 analysis_contract_version 발행
    Admin->>API: Part A 원문 범위·승인 온톨로지로 KG 구축
    API->>DB: KG 작업·corpus manifest 등록
    Worker->>A: 작업 선점·추출/검증/정합화 실행
    opt 추출 주장 검토 필요
        A->>DB: 후보·검토 대기 checkpoint 저장
        Admin->>API: 주장 후보 해시 지정 승인 또는 거절
        API->>DB: 결정·재개 작업 등록
        Worker->>A: 동일 KG 작업 thread로 재개
    end
    A->>KG: 멱등 batch 적재
    A->>DB: 적재 대조·불변 kg_snapshot_id를 ready로 발행
    Admin->>API: 보고서 brief·기간·cutoff·계약·snapshot 지정
    API->>DB: research job 등록
    Worker->>DB: 실행 선점·승인 계약·ready snapshot 고정
    Worker->>B: job_id·실행 토큰·입력 manifest 전달
    B->>DB: 고정 snapshot의 원문·근거 로드
    B->>KG: 고정 snapshot membership으로 관계 조회
    B->>B: 제한된 근거 조회·상충 비교·보고서 초안 작성
    B->>DB: draft·문장별 인용·manifest·검증 결과 저장
    Admin->>API: 보고서 본문/인용 해시 검토·승인
    API->>DB: approved 상태 저장
    Admin->>API: 승인 보고서 버전 공개
    API->>DB: published 포인터 원자적 변경
    opt 공개 데모 (향후)
        Worker->>Pub: published 버전·인용 원문 선별 export
        Reader->>Public: 보고서 목록·상세 조회
        Public->>Pub: 공개 버전·원문 인용만 조회
        Pub-->>Public: 고정 report_version_id의 보고서·출처
        Public-->>Reader: PWA 본문·근거·동일 버전 Markdown
    end
```

지원하지 않는 brief·미승인 계약·검증 실패·초안 상태는 공개로 진행하지 않습니다. 자료가 부족하면 분석 한계를 명시하거나 검토로 보류합니다. 이미 승인된 계약과 ready KG snapshot이 질문·기간을 지원하면 Part A를 반복하지 않고 Part B를 새 brief로 실행할 수 있습니다. Part A는 온톨로지·KG를 구축하는 단계이며 런타임 전문가 검색·팀 구성 기능을 뜻하지 않습니다.

### Part A 하위: 온톨로지·분석 계약 설계·승인

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 관리자
    participant API as 로컬 관리 API
    participant A as Part A 함수 파이프라인
    participant Model as LangChain 구조화 모델
    participant DB as 로컬 PostgreSQL
    Admin->>API: 지원할 질문 유형·표본·보고서 요구 지정
    API->>A: define_questions → select_samples
    A->>DB: 표본 원문 버전·기존 온톨로지 조회
    DB-->>A: 표본·버전·해시
    A->>Model: 온톨로지·근거·보고서 분석 계약 후보 요청
    Model-->>A: 후보·미지원 범위
    A->>A: 타입/시간/양태·반례·출력 절·도구/예산 검증
    A->>DB: ontology_version 참조·계약 후보·해시 저장
    A-->>Admin: 질문별 표현 가능성·오류·반례 결과
    alt 수정 필요
        Admin->>API: 후보 수정 요청
        API->>A: 새 후보 생성·재검증
        A->>DB: 새 후보 해시·검증 이력
    else 승인 요청
        Admin->>API: 후보 해시·검토 사유 제출
        API->>DB: 대상 잠금·검증·권한·현재 해시 확인
        API->>DB: 불변 analysis_contract_version·승인 이력 저장
        API-->>Admin: KG 구축에 사용할 승인 계약·온톨로지 버전
    else 거절
        Admin->>API: 거절·사유
        API->>DB: 거절 이력 저장
    end
```

Part A KG 구축에서 계약 밖 타입·관계가 발견되거나 Part B에서 지원하지 않는 질문이 발견되면 Part A 변경 후보로 보냅니다. 변경은 새 계약·KG snapshot으로 발행하며 기존 보고서 입력은 바꾸지 않습니다. 전문 에이전트의 기획 문서 검토와 제품의 분석 계약 생성은 별개입니다.

## 2. 하위: DB 초기화와 스키마 변경

```mermaid
sequenceDiagram
    autonumber
    actor Operator as 운영자
    participant CLI as 초기화 CLI
    participant Config as load_settings()
    participant Migration as apply_migrations()
    participant DB as 로컬 PostgreSQL
    Operator->>CLI: DB 초기화 명령
    CLI->>Config: 환경변수 로딩·검증
    Config-->>CLI: 접속 설정
    CLI->>Migration: 미적용 버전 확인·실행
    Migration->>DB: migration 이력 조회
    DB-->>Migration: 적용된 버전
    loop 미적용 마이그레이션 순서대로
        Migration->>DB: FK 없이 테이블·PK·UNIQUE INDEX·INDEX 생성 또는 변경
        DB-->>Migration: 결과
        Migration->>DB: 성공 버전 기록
    end
    Note over Migration,DB: 그래프 테이블(node·relation·snapshot membership)과 고유 제약도 같은 마이그레이션으로 생성
    CLI-->>Operator: 적용 버전·결과 출력
    Note over Migration,DB: 일반 마이그레이션은 데이터 보존. 트랜잭션 가능 DDL과 별도 실행이 필요한 DDL을 구분
```

## 3. 하위: 뉴스 수집 함수 파이프라인

```mermaid
sequenceDiagram
    autonumber
    participant Worker as Worker
    participant Run as run_crawl()
    participant Pure as 파싱·정규화 함수
    participant HTTP as fetch_html()
    participant Source as 뉴스 출처
    participant Save as save_article_version()
    participant DB as 로컬 PostgreSQL
    Worker->>Run: 수집 범위·실행 설정
    loop 언론사·날짜·허용 페이지
        Run->>Pure: make_list_url(source, date, page)
        Pure-->>Run: 목록 URL
        Run->>HTTP: URL 요청
        HTTP->>Source: 요청 간격·타임아웃 적용 GET
        Source-->>HTTP: 응답
        alt 정상 HTML
            HTTP-->>Run: 목록 HTML
            Run->>Pure: parse_links → normalize_url
            Pure-->>Run: 중복 제거 기사 URL
            loop 기사 URL
                Run->>HTTP: 본문 요청
                HTTP->>Source: GET article_url
                Source-->>HTTP: 응답
                HTTP-->>Run: 본문 또는 분류된 오류
                alt 본문 수집 성공
                    Run->>Pure: parse_article → normalize_article
                    Pure-->>Run: 기사·발행 시각·본문 해시
                    Run->>Save: 정규화 기사 저장
                    Save->>DB: 기사·버전 트랜잭션
                    DB-->>Save: 신규·동일·변경 결과
                    Save-->>Run: 저장 결과
                else 빈 본문 또는 요청 실패
                    Run->>DB: 기사별 실패·재수집 사유 기록
                end
            end
        else 403 또는 재시도 한도 초과
            HTTP-->>Run: 중단·실패 사유
            Run->>DB: 출처별 중단 지점 저장
            Note over Run,Source: 403 중단, 429는 Retry-After와 예산에 따라 대기·재시도
        end
    end
    Run->>DB: 집계·최종 수집 상태 저장
    Run-->>Worker: 수집 보고서
```

목록이 비었거나 이전 페이지와 같은 URL 집합이 반복되면 페이지 순회를 종료합니다. 한 기사 실패는 기록 후 다음 기사로 진행하고, 출처 단위 접근 제한은 해당 출처 수집을 중단합니다.

## 4. 하위: 중복 검사와 기사 정정 버전 저장

```mermaid
sequenceDiagram
    autonumber
    participant Run as run_crawl()
    participant Save as save_article_version()
    participant DB as 로컬 PostgreSQL
    Run->>Save: NormalizedArticle
    Save->>DB: BEGIN·기사 식별 및 공통 행 잠금
    Save->>Save: 논리 참조 존재·기사 소속 검사
    Save->>DB: 기사 키·원문 해시·정규화 버전 조회
    DB-->>Save: 기존 기사·버전
    alt 신규 기사
        Save->>DB: 기사·첫 원문 버전 저장
        Save->>DB: latest_version_id 설정
    else 동일 내용·정규화 버전
        Save->>DB: 수집 관측 이력만 기록
        Note over Save,DB: 기사·원문 버전을 추가하지 않음
    else 본문 변경
        Save->>DB: 새 원문 버전 저장
        Save->>DB: latest_version_id 변경·재처리 필요 기록
        Note over Save,DB: 이전 원문 보존. 그래프의 이전 주장은 정정 처리 작업에서 상태 갱신
    end
    Save->>DB: COMMIT
    DB-->>Save: 저장 완료
    Save-->>Run: created / unchanged / revised
```

PostgreSQL(로컬 정본)에는 FK를 생성하지 않습니다. 참조 검증과 저장은 같은 연결·트랜잭션에서 실행하고, 부모 삭제 경로도 동일한 행 잠금 규칙을 적용합니다. 신규 기사는 기사 행을 먼저 만든 뒤 원문 버전과 최신 버전 참조를 저장합니다. 고유 인덱스 충돌은 다른 작업의 선행 저장 여부를 다시 조회해 판정합니다. 무결성 오류를 모두 중복으로 처리하지 않습니다. 새 원문 처리 전에는 기존 그래프 결과가 이전 버전 기준임을 표시합니다.

## 5. 하위: Part A 구조화 추출·검증·정합화

```mermaid
sequenceDiagram
    autonumber
    participant Flow as Part A StateGraph
    participant Read as load_article_version()
    participant DB as 로컬 PostgreSQL
    participant Pure as 주제·분할·검증 함수
    participant Extract as extract_candidates()
    participant Model as LangChain 구조화 모델
    participant Resolve as canonicalize_entities()·resolve_event_candidates()
    Flow->>Read: version_id
    Flow->>DB: 승인 analysis_contract_version·실행 manifest 확인
    Read->>DB: 불변 원문 버전 조회
    DB-->>Read: ArticleVersion
    Read-->>Flow: 원문·해시·시간
    Flow->>Pure: filter_topic(article)
    Pure-->>Flow: 대상 여부
    alt 주제 대상
        Flow->>Pure: split_article(article)
        Pure-->>Flow: 문단·원문 기준 offset
        loop 문단별 처리·호출 예산 내
            Flow->>Extract: 문단·온톨로지·프롬프트 버전
            Extract->>Model: with_structured_output 계약으로 invoke
            Model-->>Extract: 구조화 후보·사용량
            Extract-->>Flow: 후보·원문 좌표 매핑
        end
        Flow->>Pure: validate_candidates(batch, article, ontology)
        Pure-->>Flow: 유효 후보·오류
        opt 수정 가능한 오류와 잔여 예산
            Flow->>Extract: repair_candidates(오류, 원문)
            Extract->>Model: 제한된 수정 호출
            Model-->>Extract: 수정 후보
            Extract-->>Flow: 수정 후보
            Flow->>Pure: validate_candidates 재실행
            Pure-->>Flow: 재검증 결과
        end
        Flow->>DB: 기관 별칭·기존 후보 조회
        DB-->>Flow: 정합화 참조 데이터
        Flow->>Resolve: 유효 후보·별칭·참조 데이터
        Resolve-->>Flow: 정합화 후보·모호성
        Flow->>Pure: 정합화 후 참조 재검증 → decide_review
        Pure-->>Flow: 적재 / 검토 / 보류 / 실패
        Flow->>DB: 후보·검증 결과·사용량 저장
    else 주제 제외
        Flow->>DB: skipped 상태·사유 저장
    end
```

원문 인용은 문단의 로컬 offset을 원문 버전 기준 offset으로 바꾼 후 검증합니다. 수정 호출은 루프 정책의 한 회를 표현한 것이며 전체 횟수·토큰·시간 제한을 넘기지 않습니다. 미검증 후보는 적재 분기로 전달하지 않습니다.

이 시퀀스는 Part A의 KG 근거 생성 하위 흐름입니다. 현재 기사 최신값 대신 실행 manifest에 고정된 article_version_id와 계약 버전을 사용합니다. 추출 완료가 보고서 공개 완료를 의미하지 않습니다.

## 6. 하위: Part A 추출 주장 검토와 LangGraph 재개

```mermaid
sequenceDiagram
    autonumber
    participant Flow as Part A StateGraph
    participant DB as 로컬 PostgreSQL
    participant CP as 로컬 PostgreSQL Checkpointer
    participant Worker as Worker
    actor Reviewer as 검토자
    participant API as 검토 API
    Flow->>DB: 검토 항목 멱등 저장(result_id, candidate_hash)
    Flow->>Flow: 별도 review 노드에서 interrupt(payload)
    Flow->>CP: 실행 상태 저장
    Flow-->>Worker: 검토 대기
    Reviewer->>API: 검토 목록·근거 조회
    API->>DB: 후보·원문·검증 오류 조회
    DB-->>API: 검토 자료
    API-->>Reviewer: 후보와 근거 표시
    Reviewer->>API: 승인 또는 거절·사유
    API->>DB: 대상 해시 확인·결정 및 재개 작업 저장
    Worker->>DB: 재개 작업 선점·결정 조회
    Worker->>Flow: invoke(Command(resume=decision), 동일 thread_id)
    Flow->>CP: 저장 상태 로드
    Note over Flow,CP: review 노드는 처음부터 재실행될 수 있음
    Flow->>DB: 현재 job lease·실행 토큰·대상 해시 대조
    Flow->>Flow: 승인 대상 해시·계약·현재 후보 대조
    alt 승인 및 대상 일치·검증 통과
        Flow->>Flow: persist 분기로 이동
    else 거절
        Flow->>DB: rejected 상태 기록
    else 후보·스키마 변경 또는 부적합
        Flow->>DB: 이전 승인 무효·재검토 또는 보류 기록
    end
```

승인은 스키마·근거 검증을 우회하지 않습니다. 검토 항목 생성과 대기 노드를 분리하여 재개 시 불필요한 외부 쓰기를 줄이고, 검토 항목 생성 자체에도 고유 키를 둡니다.

같은 thread를 여러 Worker가 동시에 재개하지 못하도록 실행 lease와 토큰을 확인합니다. 오래된 Worker의 늦은 결과 저장은 토큰 불일치로 거부하며, 후보 승인과 최종 보고서 승인/공개는 서로 대체하지 않습니다.

## 7. 하위: Part A 그래프 적재·snapshot 발행·복구

```mermaid
sequenceDiagram
    autonumber
    participant Worker as Worker
    participant Flow as Part A StateGraph
    participant Pure as build_graph_batch()
    participant Write as write_graph_batch()
    participant DB as 로컬 PostgreSQL 그래프 테이블·작업 상태
    Flow->>Pure: 검증·정합화 완료 후보
    Pure-->>Flow: 안정 ID·매개변수 GraphBatch
    Flow->>Write: GraphBatch·실행 토큰
    Write->>DB: BEGIN
    Write->>DB: 현재 실행 토큰 확인
    Write->>DB: 고유 키로 Document·Entity·Event·Claim·Evidence 노드 upsert
    Write->>DB: 관계 upsert·해당 기사 이전 주장 상태 갱신
    Write->>DB: graph_batch 완료 기록
    alt DB 쓰기 오류 또는 토큰 불일치
        DB-->>Write: 오류
        Write->>DB: ROLLBACK
        Write-->>Flow: 재시도 가능 여부와 실패 정보
        Flow->>DB: 재시도 또는 failed 기록
    else COMMIT 성공
        Write->>DB: COMMIT
        DB-->>Write: 완료
        Write-->>Flow: graph_batch_complete (snapshot 발행 전)
    else COMMIT 전 프로세스 중단
        Note over Worker,DB: 트랜잭션은 롤백됨. lease 만료 후 미완료 작업 재선점
        Worker->>Flow: 동일 작업 재개 또는 복구 실행
        Flow->>Write: 같은 멱등 키로 재적재
        Write->>DB: upsert·완료 기록을 한 트랜잭션으로 재실행
        DB-->>Write: 중복 없는 동일 결과
    end
```

체크포인트 복구 위치에 따라 적재 노드가 생략되거나 재실행될 수 있습니다. 두 경우 모두 외부 저장 결과가 동일하도록 설계합니다. 그래프 테이블 쓰기와 작업 완료 표시는 같은 PostgreSQL 트랜잭션으로 묶고, 재실행에 대비해 멱등 쓰기를 유지합니다. 정정으로 교체하는 주장은 해당 원문 계보에 한정하고, 다른 기사의 유효한 근거를 제거하지 않습니다.

```mermaid
sequenceDiagram
    participant A as Part A snapshot 발행 함수
    participant DB as 로컬 PostgreSQL manifest·batch 상태
    participant KG as 같은 DB의 snapshot membership 테이블
    A->>DB: 대상 batch가 모두 완료인지 확인
    A->>KG: manifest ID·membership·근거 참조 대조
    KG-->>A: 대조 결과
    alt 적재·검증·소속 대조 통과
        A->>DB: 짧은 트랜잭션으로 snapshot manifest 고정·ready 발행
    else 누락·충돌·중단
        A->>DB: building 또는 failed 유지·복구 작업 기록
    end
```

Part B는 ready snapshot만 읽습니다. 과거 snapshot을 사용하는 보고서가 존재하면 해당 membership과 근거 버전을 보존합니다. 정정 시 기존 ready manifest를 수정하지 않고 새 snapshot을 만듭니다. snapshot 범위 밖 최신 관계는 공통 조회 함수가 제외합니다.

## 8. 하위: 검색·타임라인·원문 근거 조회

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant UI as React PWA
    participant API as FastAPI
    participant Query as 조회 함수
    participant KG as 로컬 PostgreSQL 그래프 테이블
    participant DB as 로컬 PostgreSQL 원문
    User->>UI: 기관·기간 선택
    UI->>API: GET /entities/{id}/events
    API->>Query: list_entity_events(id, period, page)
    Query->>KG: 매개변수 조회·활성 주장 필터
    KG-->>Query: 사건·주장·근거 참조
    Query-->>API: 사건 목록·불확실성
    API-->>UI: 응답
    UI-->>User: 타임라인과 기사 연결
    User->>UI: 주장 근거 선택
    UI->>API: GET /claims/{id}/evidence
    API->>Query: get_claim_evidence(claim_id)
    Query->>KG: Evidence·Document 참조 조회
    KG-->>Query: version_id·해시·offset·인용
    Query->>DB: 해당 원문 버전·URL 조회
    DB-->>Query: 불변 원문·해시
    Query->>Query: 해시·인용 구간 일치 검사
    alt 일치
        Query-->>API: 원문 근거·필드·출처
        API-->>UI: 근거 상세
        UI-->>User: 근거 강조·사건일·발행 시각 표시
    else 참조 누락 또는 불일치
        Query-->>API: 근거 불일치 오류
        API-->>UI: 확인 불가 상태
        Note over Query,DB: 최신 원문으로 임의 대체하지 않고 대조·복구 대상으로 기록
    end
```

## 9. 하위: 관리자 조작·인증·작업 등록

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 관리자
    participant UI as PWA 관리자 화면
    participant API as 관리 API
    participant Auth as 인증 의존성
    participant DB as 로컬 PostgreSQL
    participant Worker as Worker
    Admin->>UI: 관리자 토큰 입력·로그인
    UI->>API: POST /auth/login
    API->>Auth: 서버 ADMIN_API_TOKEN 검증
    Auth-->>API: 검증 결과
    API-->>UI: 성공 시 HttpOnly 세션 쿠키·CSRF 토큰
    Admin->>UI: 수집 또는 재처리 요청
    UI->>API: 세션 쿠키·CSRF 토큰·입력·요청 식별 키
    API->>Auth: 세션·CSRF·Origin 확인
    alt 미설정 또는 인증 실패
        Auth-->>API: 거부
        API-->>UI: 인증 실패 응답
    else 인증 성공
        Auth-->>API: 관리자 식별자
        API->>API: 입력 형식 검증
        API->>DB: BEGIN
        API->>DB: 대상 행 잠금·논리 참조·현재 상태·중복 요청 확인
        API->>DB: 작업 등록 또는 허용된 상태 변경
        API->>DB: 관리자 행위·대상·사유 기록
        API->>DB: COMMIT
        API-->>UI: 작업 ID·현재 상태
        Worker->>DB: 작업 선점·처리·heartbeat 갱신
        loop 상태 확인
            UI->>API: 인증된 작업 상세 조회
            API->>DB: 단계·오류·집계 조회
            DB-->>API: 상태 정보
            API-->>UI: 진행·완료·실패·검토 대기
        end
    end
    Note over UI,DB: 화면은 DB에 직접 쓰지 않으며 API 인증·업무 상태 검증을 공통 적용
```

검토 승인·거절은 6번 시퀀스로 연결됩니다. 중지 요청은 작업에 플래그를 기록하고 Worker가 안전한 처리 경계에서 확인합니다. 설정 조회는 비밀값을 반환하지 않습니다.

## 10. 하위: 개발용 Part A 삭제 후 재작업

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 개발 관리자
    participant API as 관리 API
    participant Reset as Part A 초기화 함수
    participant DB as 로컬 PostgreSQL
    participant Worker as Worker
    participant KG as 로컬 PostgreSQL 그래프 테이블
    participant Backup as 로컬 dump 백업
    Admin->>API: 삭제 대상·재생성 옵션으로 미리보기
    API->>API: 관리자 인증·development 환경 확인
    API->>Reset: plan_part_a_reset(scope)
    Reset->>DB: 계약·온톨로지·KG snapshot·연결 작업/보고서/checkpoint 영향 조회
    Reset-->>Admin: 삭제 범위·공유 데이터 영향 표시
    Admin->>API: 대상 확인 후 초기화 실행
    API->>Reset: 미리보기 해시·대상 세대 재검증
    Reset->>DB: 영속 reset 이력·범위 fence·재생성 입력 저장
    Reset->>DB: 연결 공개 보고서 철회·신규 작업 등록 차단
    Reset->>Worker: 대상 실행 중지 요청
    Worker-->>Reset: 실행 종료 확인
    Reset->>Backup: backup_before_reset: 사전 dump 생성
    Backup-->>Reset: dump 경로·해시
    Reset->>Reset: verify_reset_backup: 파일·크기·pg_restore --list 확인
    alt 사전 백업 또는 검증 실패
        Reset->>DB: 초기화 중단 기록·범위 fence 해제
        Reset-->>Admin: 삭제하지 않고 실패 사유 표시
    else 사전 백업 검증 성공
        Reset->>DB: dump 경로·해시를 reset 이력에 기록
    end
    Reset->>DB: 이전 실행 토큰 무효화·checkpoint 참조 정리
    opt 영향 범위의 Part A KG 및 연결 Part B 포함
        Reset->>KG: 대상 주장·근거 제거 및 비공유 노드 정리
        Reset->>DB: 연결 보고서 인용·버전·KG manifest·결과·검토·작업 정리
    end
    Reset->>DB: Part A 검토·계약 후보·승인 버전·비공유 온톨로지 삭제
    Reset->>Reset: 참조·삭제 완료 검증
    Note over Reset,DB: Crawler 원문·관리자 및 초기화 이력은 보존
    opt 재생성 선택 및 초기화 검증 성공
        Reset->>DB: 새 Part A 작업 등록
        Worker->>DB: 재생성 작업 선점
        Worker->>Worker: 현재 코드·프롬프트로 Part A 재실행
    end
    Reset-->>Admin: 초기화 결과·새 작업 ID
```

삭제는 사전 dump 검증이 성공한 뒤에만 시작합니다([백업·복원](backup-restore.md)). 삭제 도중 실패하면 초기화 단계 이력으로 재개하고 재생성을 보류합니다. 연결 Part B가 있는데 함께 정리하지 않는 요청은 거부합니다. 초기화 범위의 새 작업 등록은 잠금 해제 전까지 제한합니다.

범위 fence는 장시간 row lock과 다릅니다. Worker 종료·그래프 정리를 기다리는 동안 DB 트랜잭션을 열어두지 않습니다. 단계별로 짧게 잠금·상태를 확인하며 공개 포인터를 먼저 철회하고 삭제된 근거를 참조하는 보고서가 노출되지 않게 합니다. checkpoint는 지원되는 삭제 절차로 정리하고, 초기화 완료 전 늦게 도착한 작업 쓰기는 실행 토큰과 reset 세대로 거부합니다. 원문·감사 기록과 다른 보고서가 참조하는 공유 근거는 보존합니다.

## 11. 하위: PWA 시작·오프라인·업데이트

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant PWA as 브라우저 또는 설치 앱
    participant SW as Service Worker
    participant Static as 정적 파일 서버
    participant API as FastAPI
    participant Worker as 로컬 Worker
    User->>PWA: 앱 실행
    PWA->>SW: 정적 앱 파일 요청
    alt 캐시된 파일 존재
        SW-->>PWA: 정적 파일
    else 최초 온라인 실행
        SW->>Static: 정적 파일 다운로드
        Static-->>SW: 앱 파일·manifest·아이콘
        SW-->>PWA: 정적 파일
    end
    alt 온라인
        PWA->>API: 작업·조회 데이터 요청 (캐시 제외)
        API-->>PWA: 최신 상태 (no-store)
        opt local_admin 환경에서만 작업 등록
            PWA->>API: 인증된 실행 요청
            API-->>PWA: 작업 ID
            Worker->>Worker: 화면 종료와 독립적으로 작업 처리
        end
    else 오프라인
        PWA-->>User: 연결 안내·변경 기능 비활성화
        Note over PWA,SW: API 응답과 변경 요청을 캐시·자동 재전송하지 않음
    end
    opt 새 앱 버전 발견
        SW->>Static: 새 정적 파일 확인
        Static-->>SW: 새 버전
        SW-->>PWA: 업데이트 가능
        PWA-->>User: 편집 상태 확인·업데이트 안내
        User->>PWA: 업데이트 적용
        PWA->>SW: 새 버전 활성화
        SW->>SW: 이전 정적 캐시 정리
        PWA->>API: 재로그인 필요 여부·최신 상태 조회
    end
```

처음부터 오프라인이고 앱 파일도 캐시되지 않은 경우에는 앱을 시작할 수 없습니다. 설치·일반 브라우저 실행 모두 같은 인증·캐시 정책을 적용합니다.

## 12. 하위: PostgreSQL 정본 정기 dump·별도 보관

> 백업 대상은 로컬 PostgreSQL 정본(그래프 테이블 포함)입니다. Supabase는 공개 데모(향후) export 대상이며 백업하지 않습니다([backup-restore](backup-restore.md)).

```mermaid
sequenceDiagram
    autonumber
    participant Scheduler as 로컬 Windows 작업 스케줄러
    participant Backup as 백업 스크립트
    participant DB as 로컬 PostgreSQL 정본
    participant Disk as 로컬 백업 경로
    participant Copy as 별도 저장소
    Scheduler->>Backup: 매일 03:00 Asia/Seoul 실행
    Backup->>Backup: 설정·접속 대상·중복 실행 확인
    Backup->>DB: pg_dump custom format (newscontext·langgraph)
    DB-->>Backup: 일관된 snapshot의 스키마·데이터
    Backup->>Disk: 임시 파일 저장
    Backup->>Backup: 종료 코드·archive 목록·해시 검사
    alt dump 성공
        Backup->>Disk: 최종 파일명으로 변경
        Backup->>Copy: 별도 사본 생성·해시 대조
        alt 별도 사본 확인 성공
            Backup->>Disk: 성공 기록·보존 기간 지난 사본 정리
            Note over Backup,Copy: 마지막 정상 사본 유지
        else 복사 실패
            Backup->>Disk: 복사 실패 기록·기존 사본 유지
        end
    else dump 실패
        Backup->>Disk: 실패 기록·불완전 파일 구분
    end
```

월 1회 격리 DB에 실제 복원해 행 수·인덱스·논리 참조와 화면 조회를 확인합니다. 클라우드 백업과 별도로 보존하는 dump이며 실제 스케줄은 후속 구현 시 등록합니다.

## 13. 하위: Part B 심층 리서치 보고서 생성·공개

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 로컬 관리자
    participant API as 로컬 관리 API
    participant Worker as 로컬 Worker
    participant Agent as Part B 리서치 Agent
    participant Tools as 기간·경로·근거 조회 도구
    participant DB as 로컬 PostgreSQL 원문·보고서·체크포인트
    participant KG as 로컬 PostgreSQL 그래프 테이블
    participant Check as 보고서 검증 함수
    participant Pub as Supabase 공개 사본 (향후)
    participant Render as 공개 PWA·조회 API (향후)
    actor Reader as 독자
    Admin->>API: 질문·대상·기간·cutoff·계약·ready snapshot 지정
    API->>DB: research job 등록
    API-->>Admin: 작업 ID
    Worker->>DB: 작업 선점·계약/ontology/ready snapshot 고정
    Worker->>Agent: 계획·근거 탐색 시작
    loop 근거 부족 및 잔여 예산 존재
        Agent->>Tools: 질문에 맞는 추가 조회 선택
        Tools->>DB: 선택 snapshot에 포함된 원문 버전·기사 근거 조회
        Tools->>KG: snapshot membership·기간·대상·제한된 경로 탐색
        Tools-->>Agent: 근거·상충 주장·불확실성
    end
    Agent->>Agent: 비교·분석·인용 연결된 초안 작성
    Agent->>Check: 구조화 보고서·인용 manifest
    Check->>DB: 원문·근거 해시 대조
    Check-->>Worker: 검증 결과
    alt 오류 또는 근거 부족
        Worker->>DB: 제한 수정 또는 needs_revision·실패 기록
    else 검증 통과
        Worker->>DB: draft 저장·검토 대기 checkpoint
        Admin->>API: 문장별 근거 검토·버전 해시 지정 승인
        API->>DB: approved·승인 이력
        Admin->>API: 승인 버전 공개
        API->>DB: published 포인터 변경
    end
    opt 공개 데모 (향후)
        Worker->>Pub: published 버전·인용 원문 선별 export
        Reader->>Render: 공개 보고서 열람·Markdown 다운로드
        Render->>Pub: published 포인터를 고정 report_version_id로 해석·해당 인용 조회
        Pub-->>Render: 구조화 보고서·출처
        Render-->>Reader: 보고서·근거 패널·고정 버전 Markdown 링크
    end
```

초안은 공개되지 않으며 배포 서버에서 LLM을 호출하지 않습니다. 재작성은 새 버전으로 처리하고 공개된 버전을 직접 덮어쓰지 않습니다. 결과가 불충분하면 한계를 표시하거나 공개를 보류합니다.

Part B는 KG를 추출·갱신하지 않습니다. 선택 snapshot에 근거가 부족하면 한계를 남기거나 별도 Part A 구축을 요청하고 그 결과는 새 보고서 작업에서 사용합니다. 독자가 조회한 `report_version_id`를 Markdown 링크에도 지정하여 조회 중 공개 포인터가 바뀌어도 본문·인용·다운로드가 같은 버전을 가리키게 합니다.

## 14. 하위: 조건부 임베딩 후보 검색

```mermaid
sequenceDiagram
    autonumber
    participant Flow as 로컬 Part A 또는 Part B
    participant Policy as 검색 정책·profile 검증
    participant Embed as Embedding adapter
    participant PG as 로컬 PostgreSQL pgvector
    participant Graph as 로컬 PostgreSQL 그래프 snapshot 조회
    participant Check as 원문·참조·의미 검증
    Flow->>Policy: 질문·source_type·프로젝트·snapshot/계약
    alt embedding 활성 및 계약 유효
        Policy->>Embed: 동일 model/profile로 질문 벡터 생성
        Embed-->>Policy: 벡터·차원·사용량
        Policy->>PG: project·허용 source version·cutoff·ready index/profile을 사전 필터한 top-k
        PG-->>Policy: 유사 후보 ID·score
    else 비활성 또는 미검증 기능
        Policy->>PG: keyword·별칭·구조화 후보
        PG-->>Policy: 후보 ID
    end
    Policy->>Graph: 필요한 경로·snapshot membership 조회
    Graph-->>Policy: 경로·연결 주장
    Policy->>Check: 중복 제거·후보의 원문/Claim 재조회·검증
    Check-->>Flow: 유효 후보·근거·불확실성
    Note over Flow,Check: 유사도로 자동 병합·사실 확정·snapshot 외 근거 사용 금지
```

Part A 후보 검색은 승인 온톨로지/기사 범위로, Part B는 ready snapshot과 cutoff로 제한합니다. 원본 GraphRAG/ToG/EDC의 vector 도입 여부와 제품 adapter는 [소스 감사](source-audit.md)에서 구분합니다.

검색 index manifest는 KG snapshot과 별도로 생성하고 Part B 실행에 profile과 함께 고정합니다. 재색인·모델 변경은 기존 KG/보고서 입력을 덮어쓰지 않습니다. source_type을 포함한 고유 키와 부모 검증·초기화 fencing으로 vector의 고아 참조·늦은 쓰기를 막습니다.

## 15. 구현 계획과 연결

| 다이어그램 | 대응 기능 | 핵심 검증 |
|---|---|---|
| 1 대단위 | F01~F13 | Crawler→A KG snapshot→B 보고서→공개 근거 연결 |
| 2 DB 초기화 | F01~F02 | 재실행·마이그레이션 실패 시 데이터 보존 |
| 3 수집 | F03 | 페이지 종료·요청 제한·기사별 실패 격리 |
| 4 원문 버전 | F04 | 동일 입력 불변·정정 이력 보존 |
| 5 추출·검증 | F05~F07 | 출력 계약·근거·단위·불확실성 처리 |
| 6 검토·재개 | F08 | 승인 대상 일치·거절 미적재·재시작 복구 |
| 7 적재·복구 | F09 | Part A 부분 실패·멱등성·ready snapshot 대조 |
| 8 조회 | F10~F11 | 정확한 원문 버전·인용 추적 |
| 9 관리자 | F10~F11 | 인증·허용 상태 변경·중복 방지·감사 기록 |
| 10 개발 초기화 | F05·F08~F11 | 사전 dump 검증 실패 시 삭제 중단·승인 버전 삭제·연결 결과 정리·원문 보존·재생성 |
| 11 PWA | F11~F12 | 설치·오프라인 변경 차단·API 비캐시·업데이트·재접속 |
| 12 dump 백업 | F12 | 성공 dump·별도 사본·보존·격리 복원 검증 |
| 13 리서치 보고서 | F13·F10~F12 | 문장별 인용·상충·한계·검토·공개·버전별 다운로드 |
| 14 임베딩 후보 | F14·F07·F13 | profile·차원·snapshot 필터·후보 검색 품질·의미 재검증 |

공식 API 근거: [LangChain 모델](https://docs.langchain.com/oss/python/langchain/models), [LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api), [interrupt와 재개](https://docs.langchain.com/oss/python/langgraph/interrupts). 실제 라이브러리 통합 검증과 구현은 후속 개발 단계에서 수행합니다.
