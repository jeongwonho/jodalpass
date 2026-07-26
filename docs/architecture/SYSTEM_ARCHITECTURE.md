# 시스템 아키텍처

상태: Accepted  
기준일: 2026-07-27  
적용 범위: 첫 유료 버전부터 반복입찰·낙찰 후 업무 확장까지

## 1. 이 설계가 해결하는 문제

조달패스는 단순 공고 검색기가 아니다. 외부 공공데이터, 첨부문서, 회사 자격, 해석 규칙, 사람 검수, 결제와 입찰 진행상태가 함께 움직이는 업무 시스템이다. 초기에 화면 중심으로 구현하면 다음 변경에서 쉽게 꼬인다.

- 용역 외에 물품·공사를 추가할 때 조건문이 전 화면에 퍼진다.
- 나라장터 API가 바뀌면 UI와 데이터베이스까지 함께 수정해야 한다.
- 정정공고가 기존 분석 결과를 조용히 덮어쓴다.
- AI 모델을 바꾸면 과거 판정의 근거를 재현할 수 없다.
- 구독 상품을 추가할 때 핵심 업무 로직에 가격 조건이 침투한다.
- 운영자 보정이 원본 데이터와 뒤섞여 무엇이 사실이었는지 알 수 없어진다.

따라서 목표는 기능을 미리 많이 만드는 것이 아니라, 변화가 들어오는 지점을 분리하고 각 데이터의 소유자를 명확히 하는 것이다.

## 2. 핵심 결정

1. 첫 구조는 마이크로서비스가 아닌 모듈형 모놀리스다.
2. 웹 요청과 장시간 처리는 분리하여 web과 worker 두 실행 단위로 배포한다.
3. 각 업무 모듈은 자신의 테이블과 공개 계약을 소유한다.
4. 외부 API 응답은 내부 도메인 모델로 변환하며 화면에 직접 노출하지 않는다.
5. 공고 원문과 정정 이력은 불변 버전으로 보존한다.
6. 참가조건 판정은 근거가 있는 규칙을 중심으로 하고 AI는 추출·분류 보조 역할로 제한한다.
7. 업종 확장은 핵심 코드의 조건문이 아니라 버전이 있는 규칙팩으로 수행한다.
8. 모듈 간 후속 처리는 트랜잭션 아웃박스와 멱등 이벤트로 연결한다.
9. 모든 고객 데이터에는 organization_id를 포함하고 권한·감사로그를 기본 기능으로 둔다.
10. 인프라 공급자는 포트로 감싸 교체할 수 있게 하되, 실제 고객 가치와 무관한 과도한 추상화는 만들지 않는다.

## 3. 시스템 형태

~~~mermaid
flowchart LR
    U["고객 브라우저"] --> W["Web · UI/BFF"]
    O["운영자"] --> W
    W --> APP["애플리케이션 모듈"]
    APP --> DB[("PostgreSQL")]
    APP --> OBJ[("Object Storage")]
    APP --> OUT["Outbox"]
    OUT --> Q["Job Queue"]
    Q --> WK["Worker"]
    WK --> APP
    WK --> G2B["나라장터·공공데이터 Adapter"]
    WK --> DOC["문서 추출 Adapter"]
    WK --> AI["AI Provider Adapter"]
    APP --> PAY["결제 Adapter"]
    APP --> MSG["알림 Adapter"]
    DB --> OBS["로그·메트릭·감사"]
    WK --> OBS
    W --> OBS
~~~

### 배포 단위

- apps/web: 고객 UI, 인증된 운영 화면, BFF/API, 짧은 동기 명령과 조회
- apps/worker: 공고 수집, 첨부파일 처리, 재분석, 결과 추적, 알림 등 장시간·재시도 작업
- PostgreSQL: 업무 상태의 단일 기준
- Object Storage: 원문 응답, 첨부파일, 추출 산출물, 제출 증빙
- Job Queue: 비동기 작업 전달. 특정 공급자 API는 모듈 밖으로 노출하지 않는다.

운영 화면은 초기에는 web 안에 두되 /ops 경로, 별도 권한, 별도 감사 이벤트를 강제한다. 운영팀과 배포주기가 독립되거나 보안상 격리가 필요해질 때 apps/ops로 분리한다.

## 4. 모듈 경계

| 모듈 | 책임 | 소유 데이터 | 외부에 공개하는 것 |
|---|---|---|---|
| identity-access | 사용자, 세션, 조직 멤버십, 역할 | user, membership, role_grant | 현재 주체와 권한 확인 |
| organization-profile | 회사 기본정보, 업종, 면허, 실적, 서류 메타데이터 | organization, company_profile_version, credential, company_fact | 특정 시점의 회사 사실 스냅샷 |
| procurement-catalog | 공고 수집, 정정·취소, 공식 메타데이터, 원문 버전 | tender, tender_version, source_snapshot, attachment | 표준화된 공고 버전과 변경 이벤트 |
| document-intelligence | 파일 검사, 텍스트·표 추출, 페이지·문단 위치 | document, extraction_run, document_block, evidence_span | 근거 위치가 있는 추출 결과 |
| eligibility | 요구조건 구조화, 회사사실 대조, 판정, 검수 | requirement, analysis_run, finding, finding_evidence, review | 버전 고정 진단 결과 |
| bid-workspace | 고객별 공고 진행, 할 일, 체크리스트, 제출·결과 상태 | bid_case, task, checklist_item, submission_proof, result | 현재 단계와 다음 행동 |
| commerce | 상품, 주문, 결제, 환불, 이용권 | offer, order, payment, refund, entitlement | 기능 사용 가능 여부 |
| notification | 발송 선호, 템플릿, 발송 시도 | notification_preference, message, delivery_attempt | 알림 요청과 상태 |
| operations | 재처리, 사람 검수, 고객지원, 사고·정정 | support_case, manual_action, incident | 권한이 검증된 운영 명령 |
| audit-analytics | 감사 이벤트와 제품 이벤트 수집 | audit_event, product_event | 변경 추적과 집계용 스트림 |

### 경계 규칙

- 다른 모듈의 내부 파일을 import하지 않는다. 각 모듈의 public API나 계약 이벤트만 사용한다.
- 다른 모듈이 소유한 테이블을 직접 읽거나 수정하지 않는다.
- apps는 조립과 전달만 담당하고 업무 규칙을 갖지 않는다.
- domain은 Next.js, ORM, HTTP, 큐, 결제 SDK를 import하지 않는다.
- application은 유스케이스를 조정하고 외부 시스템은 port 인터페이스로 요청한다.
- adapters는 port를 구현한다. 외부 DTO는 adapter를 벗어나지 못한다.
- shared에는 ID, 시간, 금액, 오류형 같은 작은 공통 타입만 둔다. 업무 로직이나 만능 utils를 넣지 않는다.
- 순환 의존은 허용하지 않는다.

기본 의존 방향은 다음과 같다.

~~~text
apps
  → module public API / contracts
    → application
      → domain
      → ports
adapters
  → ports 구현

domain → 외부 프레임워크 의존 금지
module A → module B 내부 구현 의존 금지
~~~

## 5. 저장소 구조

~~~text
jodalpass/
├─ apps/
│  ├─ web/
│  └─ worker/
├─ packages/
│  ├─ modules/
│  │  ├─ identity-access/
│  │  ├─ organization-profile/
│  │  ├─ procurement-catalog/
│  │  ├─ document-intelligence/
│  │  ├─ eligibility/
│  │  ├─ bid-workspace/
│  │  ├─ commerce/
│  │  ├─ notification/
│  │  └─ operations/
│  ├─ contracts/
│  │  ├─ api/
│  │  └─ events/
│  ├─ integrations/
│  │  ├─ g2b/
│  │  ├─ public-data/
│  │  ├─ payments/
│  │  ├─ messaging/
│  │  └─ ai/
│  ├─ rule-packs/
│  │  └─ general-services-it-marketing/
│  ├─ platform/
│  │  ├─ database/
│  │  ├─ queue/
│  │  ├─ object-storage/
│  │  ├─ observability/
│  │  └─ security/
│  └─ ui/
├─ tests/
│  ├─ architecture/
│  ├─ contract/
│  ├─ integration/
│  ├─ e2e/
│  └─ golden/
├─ infra/
├─ docs/
│  ├─ architecture/
│  └─ adr/
└─ tooling/
~~~

각 모듈 내부는 기능 중심으로 다음 형태를 유지한다.

~~~text
module-name/
├─ src/
│  ├─ domain/
│  ├─ application/
│  ├─ ports/
│  ├─ adapters/
│  └─ public.ts
├─ migrations/
└─ tests/
~~~

public.ts에 노출하지 않은 코드는 다른 패키지에서 가져갈 수 없도록 package exports와 정적 의존성 검사로 막는다.

## 6. 핵심 데이터 모델

### 식별과 버전

모든 주요 ID는 내부의 불투명 ID를 사용한다. 나라장터 공고번호, 차수, 사업자등록번호 같은 외부 식별자는 별도 열에 저장한다.

- tender: 한 공고 계열의 안정된 식별자
- tender_version: 최초공고·정정공고·취소 등 특정 시점의 버전
- company_profile_version: 판정 당시 회사정보 스냅샷
- rule_pack_version: 판정에 사용한 규칙 묶음
- analysis_run: 위 세 버전과 추출기·모델 버전을 고정한 실행 기록

완료된 버전과 분석은 수정하지 않는다. 새 정보가 들어오면 새 버전을 만들고 이전 결과를 superseded 상태로 연결한다.

### 분석 재현성

analysis_run에는 최소한 다음을 기록한다.

- tender_version_id
- company_profile_version_id
- rule_pack_id와 version
- extraction_pipeline_version
- model_provider와 model_version, 사용한 경우의 prompt_version
- started_at, completed_at
- input_hash와 output_hash
- 판정 상태
- 생성한 finding 목록
- 운영자 검수와 수정 이력

판정 한 건은 단순 boolean이 아니다.

~~~text
finding
- requirement_code
- outcome: met | unmet | needs_review | not_applicable
- severity: blocking | warning | info
- normalized_value
- explanation
- confidence
- rule_id + rule_version
- evidence_span_ids[]
- company_fact_ids[]
~~~

최종 참가판정은 eligible, ineligible, needs_review 세 상태를 사용한다. 미해석·누락·상충 조건이 하나라도 차단 가능성을 가지면 eligible로 올리지 않는다.

## 7. 주요 상태기계

### 공고

~~~text
discovered → active → amended → active
                    ↘ cancelled
active → closed
~~~

정정공고 수집 시 기존 tender_version을 수정하지 않고 새 버전을 만든다. 해당 버전을 참조한 완료 분석은 superseded가 되고 재분석 작업이 생성된다.

### 분석

~~~text
queued
→ collecting
→ extracting
→ evaluating
→ needs_human_review | completed
→ failed_retryable | failed_terminal
→ superseded
~~~

각 전이는 상태 조건, 실행 주체, 발생 이벤트, 재시도 가능 여부를 갖는다. 같은 작업이 두 번 실행돼도 동일 결과를 내도록 idempotency_key를 사용한다.

### 고객 입찰 케이스

~~~text
draft
→ diagnosing
→ action_required | not_eligible | ready_to_prepare
→ preparing
→ ready_to_submit
→ submitted
→ result_pending
→ won | lost | cancelled | expired
~~~

분석 상태와 고객의 업무 상태를 하나의 열에 섞지 않는다. 참여 가능 여부가 바뀌어도 고객이 이미 수행한 작업과 제출 증빙은 보존해야 하기 때문이다.

## 8. 동기와 비동기 경계

### 동기로 처리

- 인증과 권한 확인
- 회사 프로필의 짧은 수정
- 이미 처리된 공고·분석·할 일 조회
- 주문 생성과 결제 승인 결과 접수
- 사용자의 작업 완료 체크

### 비동기로 처리

- 나라장터 API 호출과 변경 감지
- 첨부파일 다운로드, 악성파일 검사, 텍스트·표 추출
- 규칙 실행과 AI 보조 추출
- 정정공고 재분석
- 낙찰·계약 결과 추적
- 이메일·문자·푸시 발송
- 통계 집계와 대량 내보내기

사용자에게는 처리 중 상태, 예상되는 다음 단계, 실패 사유와 재시도 상태를 보여준다. HTTP 요청이 끝날 때까지 분석 전체를 붙잡아 두지 않는다.

## 9. 이벤트와 일관성

한 데이터베이스 트랜잭션 안에서 업무 변경과 outbox_event를 함께 저장한다. worker가 이를 전달하고 소비자는 event_id와 idempotency_key로 중복을 제거한다.

초기 핵심 이벤트:

- tender.version.discovered
- tender.version.amended
- tender.cancelled
- document.extraction.completed
- company.profile.versioned
- eligibility.analysis.completed
- eligibility.analysis.superseded
- eligibility.review.requested
- bid_case.task.completed
- bid_case.submission.confirmed
- commerce.entitlement.changed
- payment.refunded

이벤트에는 민감한 문서 본문이나 사업자등록번호 원문을 넣지 않는다. ID와 최소 메타데이터만 보내고 권한 있는 소비자가 자신의 인터페이스를 통해 조회한다.

## 10. 보안과 테넌트 격리

- 모든 고객 소유 레코드는 organization_id를 갖는다.
- 요청 시작 시 actor_id, organization_id, role을 확정하며 클라이언트가 보낸 조직 ID를 그대로 신뢰하지 않는다.
- 테넌트 필터는 저장소 계층과 데이터베이스 정책으로 이중 적용한다.
- 운영자 조회와 수정은 목적·사유를 입력하고 별도 audit_event를 남긴다.
- 사업자등록번호, 담당자 정보, 고객 문서는 민감도에 따라 암호화·마스킹한다.
- 공동인증서, 비밀번호, 카드번호는 저장하지 않는다.
- 파일은 업로드와 외부 다운로드 모두 유형·크기·해시·악성 여부를 확인한 뒤 처리한다.
- 공개 로그·분석 이벤트에는 개인정보와 문서 본문을 넣지 않는다.

## 11. 관측성과 운영

모든 요청과 비동기 작업은 trace_id, actor_id, organization_id, tender_id, analysis_run_id 중 해당 식별자를 구조화 로그에 넣는다. 단, 개인정보 원문은 제외한다.

필수 지표:

- 소스별 API 성공률·지연·제한 응답
- 공고 수집 지연과 정정 감지 지연
- 파일 유형별 추출 성공률
- 단계별 분석 실패율과 재시도 횟수
- needs_review 비율과 사람 처리시간
- 중대한 허위 참가 가능 판정
- 결제·환불·이용권 불일치
- 큐 적체 시간
- 고객별 첫 입찰 완주율

운영자는 특정 공고 버전과 분석 실행을 찾아 입력, 규칙, 근거, 실패 단계, 재처리 이력을 볼 수 있어야 한다. 재처리는 기존 결과를 덮어쓰지 않고 새 run을 만든다.

## 12. 품질 방어선

CI에서 다음을 자동 검사한다.

- 모듈 간 금지 import와 순환 의존
- public API 밖의 deep import
- 데이터베이스 마이그레이션의 전진·롤백 또는 보상 절차
- 외부 API adapter 계약 테스트
- organization_id 누락과 테넌트 격리 테스트
- 이벤트 스키마 호환성
- 규칙팩 스키마와 골든셋 회귀
- 정정공고가 이전 분석을 만료시키는 E2E
- 결제·환불·이용권 일관성
- 접근성·모바일·핵심 성능 예산

## 13. 개발 순서

### 0단계: 구조 방어선

- monorepo와 package exports
- 모듈 의존성 검사
- 환경설정 검증
- CI, 마이그레이션, 로그, 오류추적
- 조직·사용자·권한과 감사로그
- queue, object storage, outbox 인터페이스

### 1단계: 공고 원문 수집 세로 기능

공고 URL 또는 번호 입력부터 공식 API 수집, raw snapshot 저장, tender_version 생성, 기본정보 화면 표시까지 완성한다. 정정·취소·중복·API 장애를 포함한다.

### 2단계: 첨부문서와 근거

첨부파일 다운로드, 검사, 추출, 페이지·문단 근거 표시, 추출 실패 운영자 재처리까지 완성한다.

### 3단계: 회사 프로필

공식 업체정보 조회와 고객 확인을 결합해 versioned company facts를 만든다. 출처, 확인자, 유효기간, 증빙을 기록한다.

### 4단계: 참가조건 진단

첫 규칙팩을 적용해 requirement와 finding을 만들고 회사사실과 대조한다. needs_review, 사람 검수, 정정 후 재분석까지 포함한다.

### 5단계: 첫 입찰 완주

결제·이용권, 개인화 할 일, 문서함, 제출 체크, 증빙, 결과 추적을 한 흐름으로 연결한다.

각 단계는 데모 화면이 아니라 실제 공고 기반 수용 기준, 실패 흐름, 운영 화면, 자동 테스트까지 완료해야 다음 단계로 간다.

## 14. 나중에 분리할 기준

다음 중 하나가 실제로 발생할 때만 모듈을 별도 서비스로 추출한다.

- 독립적으로 3배 이상 다른 확장 특성이 지속된다.
- 장애를 격리하지 않으면 핵심 결제·업무 흐름이 반복적으로 영향을 받는다.
- 별도 팀이 독립 배포와 데이터 소유권을 가져야 한다.
- 보안·규제상 프로세스 또는 네트워크 분리가 필요하다.
- 측정된 성능 한계가 현재 구조로 해결되지 않는다.

추출 순서는 document-intelligence worker, notification, analytics 가능성이 높다. 그러나 실제 지표 없이 미리 분리하지 않는다. 현재 모듈 경계와 공개 계약을 지키면 이 이동은 업무 로직을 다시 쓰는 작업이 아니라 배포 경계를 바꾸는 작업이 된다.

## 15. 변경 불가 원칙

다음은 제품 방향이 바뀌어도 유지한다.

- 원문과 과거 결과를 덮어쓰지 않는다.
- 근거 없는 참가 가능 판정을 만들지 않는다.
- 외부 API DTO를 도메인과 화면으로 누출하지 않는다.
- 다른 모듈의 데이터베이스를 몰래 읽지 않는다.
- 업종별 조건문을 핵심 흐름에 추가하지 않는다.
- 결제 플랜 이름을 업무 규칙 안에 하드코딩하지 않는다.
- 비동기 작업은 멱등성과 재시도를 설계하지 않고 추가하지 않는다.
- 운영자의 수동 수정도 감사와 원본 보존 없이 수행하지 않는다.
