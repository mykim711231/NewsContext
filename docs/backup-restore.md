# Supabase Free 백업·복원 계획

상태: 운영 설계. 실제 스크립트·예약 작업·DB 접속 및 복원은 아직 실행하지 않았습니다.

## 대상과 주기

- 관계형 DB: Supabase Free PostgreSQL. 업무 테이블은 `newscontext` 스키마에, 체크포인트는 `langgraph` 스키마에 배치합니다.
- 백업: 매일 03:00 Asia/Seoul, 삭제·초기화·스키마 변경 전 추가 실행.
- 실행·보관: 개발자 로컬 PC의 작업 스케줄러·`BACKUP_DIR` 및 별도 장치·저장소 `BACKUP_COPY_DIR`, 기본 14일. Render에서 백업을 실행하거나 dump를 보관하지 않습니다.
- 복원 검증: 월 1회 및 절차 변경 시 격리된 PostgreSQL 환경에서 수행.
- Free 플랫폼 자동 백업에 의존하지 않고 직접 만든 dump를 보관합니다.

백업 대상은 Crawler 원문 버전, Part A의 `analysis_contract_version`·온톨로지·추출/검증/정합화 결과·KG 적재 이력·`kg_snapshot_id` 및 근거 manifest, Part B 작업·보고서 버전·문장별 인용·공개 포인터 및 PostgreSQL 체크포인트입니다. 전문가 에이전트의 문서 검토를 위한 별도 런타임 카탈로그나 ResearchPlan 저장소는 만들지 않습니다. source code·프롬프트·계약 파일은 Git에서 버전 관리하고 백업 manifest에는 해당 커밋·파일 해시를 기록합니다.

실제 예약 작업과 백업/복원 API·화면은 아직 구현되지 않았습니다. DB 스키마가 확정되면 백업 대상 테이블·제외 대상·확장 의존성 목록을 manifest로 고정합니다.

임베딩을 활성화하면 업무 스키마의 embedding_profiles/text_chunks/text_embeddings와 vector 인덱스 정의도 dump 대상입니다. 복원 대상에 호환 pgvector extension을 먼저 준비하고 실제 차원·profile·모델 revision·chunk 해시·snapshot 소속과 검색을 검증합니다. extension 객체 자체와 외부 모델 가중치가 선택 스키마 dump만으로 복제된다고 가정하지 않습니다. 재생성 가능한 검색 파생물이라도 복원 직후 불완전한 index/profile을 정상으로 표시하지 않습니다. [임베딩 설계](embedding-retrieval.md)

검색 index manifest와 KG snapshot은 별도 버전입니다. `search_index_manifest_id`·profile·보고서 실행 연결도 함께 복원하고, index를 재생성하면 새 manifest로 기록합니다. 복원 뒤 KG 내용이 같다는 이유로 다른 embedding profile을 과거 보고서의 검색 입력으로 대체하지 않습니다.

참조 소스의 commit·license 기록과 모델/데이터 사용 범위는 [소스 감사](source-audit.md) 및 Git 설정에서 관리하고 백업 manifest에는 해당 버전·embedding profile을 연결합니다. 원형 재현의 FAISS·실험 파일은 제품 dump와 분리합니다.

## 구현할 백업 절차

1. 백업 실행 계정에서 비밀값을 로딩하고 Supabase 접속 대상·허용 스키마·출력 경로를 확인합니다. 직접 연결 또는 session pooler를 사용하며 transaction pooler는 사용하지 않습니다.
2. 동일 백업의 중복 실행을 잠그고 서버와 호환되는 `pg_dump` 버전을 확인합니다. 접속 비밀번호는 로그·프로세스 명령 인자에 넣지 않고 권한이 제한된 자격 증명 설정으로 전달합니다.
3. `pg_dump`의 custom format, `newscontext`·`langgraph` 스키마 대상으로 임시 `.dump` 파일을 생성합니다. 기본 옵션은 `--format=custom --schema=newscontext --schema=langgraph --no-owner --no-privileges`로 두고 스키마·데이터를 같은 실행에서 보관합니다.
4. 성공 종료·파일 크기·`pg_restore --list` 결과·SHA-256을 확인한 뒤 최종 이름으로 변경합니다. 이 검사는 실제 복원 성공을 보장하지 않습니다.
5. 별도 보관 경로로 복사하고 해시 일치를 확인합니다. dump·메타데이터에 접근 제한과 저장소 암호화를 적용합니다.
6. 성공·실패·별도 복사 결과를 기록하고, 새로운 정상 사본과 별도 사본을 확인한 뒤 보존 기간을 초과한 파일만 정리합니다. 마지막 정상 사본은 유지합니다.
7. OS 스케줄러에는 실행 파일·작업 디렉터리·계정·시간대를 명시합니다. PC가 꺼져 있으면 예정대로 실행되지 않으므로 누락 실행 처리 정책을 설정하고 마지막 성공 시각을 관리자 화면에 표시합니다.

백업 스크립트는 후속 구현에서 `scripts/`에 작성합니다. 스키마에 포함된 객체가 다른 스키마의 함수·확장·역할에 의존한다면 해당 의존성과 생성 절차를 별도 기록합니다. Supabase 관리 스키마를 임의로 백업 범위에 추가하지 않습니다.

`pg_dump`가 제공하는 PostgreSQL snapshot의 일관성과 앱의 업무 단계 완료는 구분합니다. 처리 중인 작업·체크포인트가 서로 다른 transaction에서 기록되므로 같은 dump에도 실행 중 상태가 존재할 수 있습니다. 정기 백업은 이를 허용하고 복원 후 작업 대조로 회복합니다. 개발 초기화·스키마 변경 전 백업은 대상 실행·승인·공개를 멈추고 진행 중 transaction이 정리된 뒤 추가 dump를 만들어 복구 지점을 명확히 합니다. 필수 사전 백업 실패 시 해당 삭제·복원 작업을 진행하지 않습니다.

manifest에는 backup_id, UTC 시각과 표시 시간대, DB/스키마 식별자, 서버·클라이언트 버전, 커밋·분석 계약 버전, archive 형식·크기·SHA-256, 별도 사본 위치·검증 결과, 단계별 상태를 담습니다. 백업 당시의 행 수가 필요하면 같은 snapshot과 맞춘 측정을 사용하고, dump가 끝난 뒤 변경 중인 DB의 단순 count를 정확한 정답으로 기록하지 않습니다. 실패 로그는 URI 비밀번호·원문·키를 제거합니다.

백업은 경로를 고정하고 파일 생성·보존 정리 시 해석된 절대 경로가 허용 백업 루트 안인지 확인합니다. 미완료 파일은 `.dump.partial`로 구분하며 성공 목록에 올리지 않습니다. 별도 복사 실패는 `dump 성공/별도 사본 실패`로 표시하고 전체 백업 성공으로 뭉뚱그리지 않습니다. 보존 정리는 검증된 완료 파일에만 적용하고 마지막 정상 로컬·별도 사본을 유지합니다.

## 로컬 관리자 백업·복원 화면

관리 화면은 백업 목록·마지막 정상 시각·지금 백업·별도 사본 상태·복원 검사·복원 실행 이력을 제공합니다. DB 연결·백업 경로·대상 schema는 운영 설정으로 고정하고 브라우저 입력으로 임의 shell 명령·서버 파일 경로·접속 URL을 실행하지 않습니다. 명령은 검증된 실행 파일과 구조화된 인자 배열로 호출합니다.

백업 index·manifest·복원 journal은 `BACKUP_STATE_DIR`에 DB와 별도로 저장합니다. Supabase가 접속 불가해도 로컬 관리자 로그인·백업 목록·복구 절차는 열려야 하므로 앱 시작을 DB 연결 성공에 종속시키지 않습니다. 인증은 로컬 설정으로 확인하고 감사 기록은 로컬 journal에 먼저 남긴 뒤 DB 복구 후 동기화할 수 있도록 합니다. 로컬 API는 loopback에 바인딩하고 인증·Origin/CSRF 검사를 적용하며 공개 Render에는 이 기능을 등록하지 않습니다.

백업 파일은 관리되는 목록의 ID로 선택하고 SHA-256·manifest·archive 목록을 다시 확인합니다. 임의 업로드 dump를 즉시 실행하지 않습니다. dump 복원은 SQL을 실행할 수 있으므로 소유자가 만든 신뢰된 archive만 사용하고 먼저 격리 환경에서 검증합니다. 복원 API는 즉시 실행 결과 대신 로컬 복원 작업 ID를 반환하며 단계·실패·재실행 가능 여부를 조회합니다.

## 접속 불가·휴면 대응

Supabase 접속 실패만으로 삭제·휴면을 단정하지 않습니다. 네트워크·인증·설정·플랫폼 상태를 구분하고, 일시 중지된 경우 Supabase Dashboard에서 Resume한 뒤 연결을 다시 확인합니다. 단순 Resume에는 dump 재적재가 필요하지 않습니다. 프로젝트 재개와 데이터 손상 복원은 서로 다른 작업입니다. 최신 재개 가능 기간은 [공식 휴면 정책](https://supabase.com/docs/guides/platform/free-project-pausing)을 확인합니다.

원본 DB가 접속 불가하면 새 dump를 만들 수 없으므로 최신 정상 사본을 보존하고 별도 복원 대상에 검사합니다. 복원 대상은 backup의 source와 비교해 호스트·프로젝트·스키마·동작을 명시적으로 확인합니다. 기존 데이터가 있는 대상에 자동 overwrite하지 않습니다. 재개·복원이 끝날 때까지 Worker 선점·보고서 공개를 제한합니다.

## 복원과 확인

1. 먼저 격리된 검증 DB를 생성하고 PostgreSQL 버전·필수 확장·역할을 준비합니다. `newscontext`와 `langgraph`가 모두 비어 있는 격리 대상에 같은 dump를 `pg_restore`로 복원하고 기본적으로 소유자·권한 복원을 제외합니다. 운영 대상 스키마를 검증 준비 과정에서 자동 삭제하지 않습니다.
2. 덤프의 메타데이터와 테이블별 행 수·PK·고유/일반 인덱스·시퀀스를 비교합니다. 업무 스키마에 FK가 생성되지 않았는지도 확인합니다.
3. 프로그램의 참조 대조 함수로 고아 참조·잘못된 기사 최신 버전·작업/결과 연결을 검사합니다.
4. Crawler 기사 조회·Part A 분석 계약/온톨로지 버전·승인 상태·KG 적재 입력·KG snapshot manifest 및 Part B 보고서/인용/공개 버전 관계를 검증합니다. 분석 계약에 연결된 질문 유형·허용 도구·보고서 절·예산/평가가 같은 버전인지 확인합니다. 보고서는 승인된 `analysis_contract_version`·`ontology_version`과 ready 상태의 `kg_snapshot_id`를 참조해야 합니다. building/validating/withdrawn/failed snapshot은 새 보고서 실행 대상으로 복구하지 않습니다.
5. 결과를 복원 검증 보고서로 기록합니다. 운영 복원이 필요할 경우에는 대상 확인·현재 상태 추가 백업·관련 Worker 중지 후 별도 절차로 수행하며 검증 스크립트가 운영 DB를 자동 덮어쓰지 않도록 합니다.

Supabase 전체 프로젝트 이전은 이 업무 스키마 dump와 범위가 다릅니다. 관리 역할·스키마·서비스를 포함하는 이전은 공식 Supabase CLI 백업·복원 절차에 맞춰 별도 수행합니다. Auth·Storage를 향후 도입하면 데이터와 실제 파일의 백업 범위도 추가 정의합니다.

권한은 `--no-owner --no-privileges` 백업으로 복원되지 않으므로 복원 후 역할·schema/view 권한 마이그레이션을 적용하고 Render 읽기 계정이 초안·내부 원문·관리 로그를 읽거나 데이터를 쓸 수 없는지 검사합니다. 프로그램의 논리 참조 검사 전에 공개 서비스를 재개하지 않습니다. FK를 사용하지 않는다는 원칙은 사용자 업무 스키마에 적용하며 Supabase 관리 스키마의 기존 제약을 삭제하지 않습니다. 체크포인터 패키지 생성 스키마도 설치 버전에서 검증하고 충돌을 문서화합니다.

## 다른 저장소와의 관계

LangGraph 체크포인트는 Supabase PostgreSQL의 `langgraph` 스키마에 저장하고 업무 스키마 `newscontext`와 같은 pg_dump 실행에 포함합니다. SQLite 파일은 사용하지 않습니다. 복원 검사에는 checkpoint 테이블·thread_id·승인 대기 재개를 포함하고 실제 외부 쓰기는 격리된 검증 대상에서만 수행합니다.

체크포인트의 존재가 재개 가능성을 보장하지 않습니다. 코드·직렬화 형식·분석 계약·온톨로지·프롬프트·모델·근거 버전 호환성을 먼저 점검합니다. 과거 버전 도구를 사용할 수 없거나 외부 쓰기 이력이 불명확하면 자동 재개하지 않고 새 작업 생성 또는 수동 검토로 전환합니다. 완료 thread 정리는 지원되는 패키지 절차로 수행하고 검토 대기 상태는 보존합니다.

(2026-11-13 로컬/공개 분리 반영: 아래 Aura는 공개 데모 기준이며 로컬 Neo4j Community와 로컬 파일 archive의 백업은 PostgreSQL dump와 별도로 설계합니다. 아직 미설계입니다. [corpus-feasibility §8.2](corpus-feasibility.md)) Neo4j AuraDB Free는 PostgreSQL dump에 포함되지 않습니다. Part A의 KG 재구축에 필요한 원문·승인 온톨로지·검증 결과·snapshot manifest는 PostgreSQL에 저장합니다. Part B는 승인된 KG snapshot을 읽어 보고서를 생성하고 KG 자체를 재구축하지 않습니다. 복원 후 실행 중으로 남은 작업은 실제 로컬 Worker 상태와 대조해 재시도 상태로 전환하고 중복 적재를 멱등 키로 막습니다. 업무 상태와 체크포인트가 별도 트랜잭션으로 기록될 수 있으므로 일치 여부를 확인합니다. PostgreSQL 복원 시점과 Aura 데이터가 다를 수 있으므로 Part A 복구 경로로 그래프 상태를 대조·재구축한 뒤 조회를 정상화합니다. 재구축된 KG가 과거 snapshot과 동일하다고 입증할 수 없다면 새 snapshot으로 등록하고 연결된 보고서를 검토·재작성하기 전까지 공개하지 않습니다.

**로컬 본체 그래프·파일 archive 백업 초안 (2026-11-13, 미구현·미검증):** Neo4j Community는 공식 문서 기준 사용자 DB 1개이며 `neo4j-admin database dump`는 서버를 정지한 상태에서만 실행됩니다. 따라서 로컬 그래프 백업은 `Worker 정지 → Neo4j 정지 → dump → 재기동 → 헬스 확인` 순서의 cold dump로 하고 정지 시간을 실측합니다. 복원은 서버 정지 후 `neo4j-admin database load neo4j --from-path=<dump 폴더> --overwrite-destination=true`이며 로드 후 DB가 offline으로 보이는 사례가 보고되어 있으므로 로그로 상태를 확인합니다. 그래프는 PostgreSQL의 승인 온톨로지·원문·snapshot manifest에서 재구축 가능하므로 dump는 재구축 시간을 줄이는 보조 수단이고 정본은 PostgreSQL입니다. 로컬 파일 archive는 형식·사본 위치가 미정이므로([corpus-feasibility §8.3](corpus-feasibility.md)) 해시 manifest와 별도 디스크 사본 원칙만 정하고 세부는 결정 후 작성합니다. Docker로 구동할 경우 승인 후 named volume 또는 Windows 경로를 사용하며 dump는 Git에서 제외합니다.

일 1회 백업은 마지막 성공 시점 이후 변경분의 유실 가능성을 가집니다. 중요한 개발용 초기화 전 추가 dump를 남기고 복원에 필요한 코드·마이그레이션·프롬프트·온톨로지 설정 버전도 기록합니다. 비밀값과 dump는 Git에서 제외합니다.

운영 복원 journal은 `대상 확인 → 격리 검증 → 공개 유지보수/Worker 정지 → 복원 전 추가 백업(가능할 때) → DB 복원 → 권한/참조/버전 검사 → Aura 대조·재구축 → 공개 보고서 확인 → 정상화`를 기록합니다. 중간 실패 시 이미 복원된 DB에 전체 import를 무조건 반복하지 않고 현재 상태를 검사해 재시도/중단을 선택합니다. 공개 보고서의 원문 근거를 재확인할 수 없으면 철회 또는 비공개 상태로 유지합니다.

완료 기준은 dump 생성뿐 아니라 별도 사본·월간 실제 복원·DB 장애 중 로컬 목록 접근·권한 재적용·체크포인트 호환성·보고서 인용 복구·마지막 성공 기록까지입니다. 정확한 복구 시간은 실제 복원 실험으로 측정하며 현재 보장 수치를 제시하지 않습니다.

참고: [Supabase 백업 정책](https://supabase.com/docs/guides/platform/backups), [CLI 백업·복원](https://supabase.com/docs/guides/platform/migrating-within-supabase/backup-restore), [PostgreSQL 접속](https://supabase.com/docs/guides/database/connecting-to-postgres).
