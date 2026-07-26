# 확장 개발 프로토콜

상태: Required  
적용 대상: 사람과 Codex가 수행하는 모든 기능·연동·데이터 변경

## 1. 목적

이 문서는 “새 기능을 빨리 추가했지만 기존 기능과 데이터가 꼬이는 상황”을 막는 변경 계약이다. 기능 개발자는 코드를 만들기 전에 변경이 어느 경계에 속하는지 결정하고, 계약·마이그레이션·실패 흐름·운영 방법을 함께 제출해야 한다.

## 2. 변경을 시작하기 전 분류

새 요청은 먼저 아래 중 하나로 분류한다.

| 변경 종류 | 기본 확장 지점 | 핵심 모듈 변경 여부 |
|---|---|---|
| 새 조달 업종·사업유형 | packages/rule-packs의 새 규칙팩 | 원칙적으로 없음 |
| 새 공공데이터 API | packages/integrations의 새 adapter | canonical contract가 부족할 때만 |
| 새 문서 형식 | DocumentExtractor adapter | document-intelligence 공개 계약은 유지 |
| 새 AI 공급자·모델 | AI Provider adapter와 실행 설정 | eligibility 결정 규칙은 유지 |
| 새 결제상품·할인 | commerce의 offer·entitlement 설정 | bid-workspace 로직 변경 금지 |
| 새 알림 채널 | notification delivery adapter | 발송 요청 계약은 유지 |
| 새 고객 역할 | identity-access policy | UI 조건문만으로 권한 구현 금지 |
| 새 입찰 단계 | bid-workspace 상태기계 | ADR과 기존 데이터 마이그레이션 필요 |
| 낙찰 후 기능 | 새 bounded module 우선 검토 | 기존 bid-workspace에 무조건 추가 금지 |
| 성능 개선 | 측정된 병목의 adapter·query | 도메인 의미 변경 금지 |

어느 칸에도 맞지 않으면 새 모듈이 필요한지 ADR을 먼저 작성한다.

## 3. 기능 변경 패킷

코드 전에 다음 문서를 이슈 또는 docs/features에 작성한다.

~~~text
작업명:
고객과 실제 상황:
끝났을 때 고객이 얻는 결과:
소유 모듈:
사용하는 공개 계약:
새로 추가할 계약:
데이터 소유자:
공식 데이터 출처:
상태 전이:
동기/비동기 경계:
권한과 개인정보:
정상 흐름:
실패·중단·재시도·취소 흐름:
멱등성 키:
수용 기준:
골든셋·테스트:
로그·메트릭·운영 화면:
마이그레이션·백필:
기능 플래그와 롤아웃:
롤백 또는 보상 방법:
범위에서 제외:
~~~

화면 목록만 있는 작업은 구현 준비가 완료되지 않은 것으로 본다.

## 4. 의존성 결정 절차

1. 데이터와 업무 규칙을 실제로 소유할 모듈을 하나 선택한다.
2. 다른 모듈 정보가 필요하면 해당 모듈의 public query나 event를 사용한다.
3. 해당 계약이 없다면 필요한 최소 계약을 먼저 설계한다.
4. 편의를 위해 다른 모듈 테이블을 join하거나 내부 파일을 import하지 않는다.
5. 두 모듈이 서로 호출해야 한다면 순환 의존을 만들지 말고 이벤트 또는 상위 orchestration use case로 이동한다.
6. 공통화는 두 개 구현이 비슷해 보인다는 이유가 아니라 같은 의미와 변경주기를 가질 때만 한다.

새 코드가 shared/utils로 들어가야만 구현될 수 있다면 모듈 소유권을 다시 검토한다.

## 5. 데이터베이스 변경 규칙

### 소유권

- 마이그레이션은 테이블 소유 모듈에 둔다.
- 다른 모듈의 열을 직접 추가하지 않는다.
- 고객 소유 데이터에는 organization_id, created_at, updated_at을 기본으로 둔다.
- 외부 식별자에는 공급자와 업무구분을 포함한 unique constraint를 설계한다.
- 비동기 명령은 idempotency_key의 uniqueness로 중복을 막는다.
- 금액은 정수 최소 화폐단위와 currency로 저장한다.
- 시각은 UTC로 저장하고 공고 원문 시간대와 표시 시간대를 별도로 다룬다.

### Expand → Migrate → Contract

호환되지 않는 스키마 변경은 한 번의 배포로 끝내지 않는다.

1. Expand: 새 열·테이블을 nullable 또는 기본값과 함께 추가한다.
2. Dual write/read: 구·신 구조를 함께 지원한다.
3. Migrate: 재시작 가능한 백필 작업으로 기존 데이터를 변환한다.
4. Verify: 누락·중복·해시·건수를 검증한다.
5. Switch: 읽기를 새 구조로 전환하고 관측한다.
6. Contract: 충분한 안정화 기간 뒤 구 구조를 제거한다.

열 rename, type 축소, NOT NULL 추가, 테이블 삭제는 바로 수행하지 않는다. 대규모 백필은 요청 처리 프로세스가 아니라 worker에서 청크와 체크포인트를 사용한다.

## 6. API와 이벤트 호환성

- API request와 response는 schema로 검증한다.
- 공개 필드는 삭제하거나 의미를 바꾸지 않고 새 필드를 추가하는 방식을 우선한다.
- 이벤트에는 event_id, event_type, schema_version, occurred_at, producer, correlation_id를 둔다.
- 소비자는 알 수 없는 추가 필드를 무시할 수 있어야 한다.
- 필드 삭제나 의미 변경이 필요하면 새 이벤트 버전을 병행 발행한다.
- 이벤트 consumer는 중복·역순·지연 수신을 견딘다.
- 이벤트에 문서 본문, 인증정보, 전체 사업자등록번호를 넣지 않는다.
- UI는 DB schema가 아니라 API contract에 의존한다.

## 7. 업종 규칙팩 추가 방식

새 업종은 core 코드에 if category === ... 형태로 추가하지 않는다. 규칙팩은 다음을 포함한다.

~~~text
rule-pack/
├─ manifest.yaml
├─ facts.schema.json
├─ requirements.schema.json
├─ rules/
├─ copy/
│  └─ ko-KR.json
├─ mappings/
│  └─ official-codes.yaml
├─ fixtures/
└─ golden/
~~~

manifest 최소 항목:

- pack_id와 semantic version
- 지원 업무구분과 업종코드
- 필요한 회사 fact
- 필요한 공고·문서 fact
- 지원하는 requirement type
- 미지원·사람검수 조건
- 적용 시작일과 종료일
- 규칙 작성자·검수자
- 골든셋 버전
- 변경 로그

규칙팩 변경은 과거 analysis_run을 바꾸지 않는다. 새 버전은 재분석 대상으로 명시된 공고에만 적용한다. major 변경은 판정 의미 변경, minor는 새 규칙 추가, patch는 의미를 바꾸지 않는 오류 수정으로 관리한다.

## 8. 기능 플래그와 롤아웃

모든 고위험 기능은 다음 범위로 켤 수 있어야 한다.

- 전체 off
- 내부 운영자
- 지정 조직
- 지정 업종 또는 공고
- 일정 비율
- 전체 on

플래그 이름을 도메인 로직 안에 직접 흩뿌리지 않고 application boundary에서 capability를 결정한다. 결제 권한과 실험 플래그를 같은 개념으로 사용하지 않는다.

롤아웃 순서:

1. shadow: 결과를 저장하되 고객에게 노출하지 않음
2. internal: 내부 골든셋과 운영자 사용
3. pilot: 지정 유료고객
4. limited: 트래픽 일부
5. general: 전체
6. cleanup: 안정화 후 플래그 제거

허위 참가 가능 판정, 테넌트 누출, 중복결제처럼 중대한 문제가 발생하면 즉시 kill switch를 내리고 영향 분석을 시작한다.

## 9. 테스트 포트폴리오

기능별 최소 테스트:

- domain unit: 상태전이와 불변조건
- module integration: 실제 DB와 transaction
- adapter contract: 공식 API fixture와 오류
- event contract: producer·consumer schema
- security: 역할과 조직 격리
- idempotency: 같은 명령·이벤트 중복 실행
- migration: 빈 DB와 기존 fixture DB 모두 업그레이드
- golden: 실제 검수 공고 회귀
- E2E: 사용자가 약속한 결과까지
- operations: 실패 작업 조회·재처리·감사

스냅샷 테스트만으로 업무 판정을 검증하지 않는다. 중요한 규칙은 입력 사실, 적용 규칙, 기대 finding, 근거까지 명시한다.

## 10. 관측성 계약

새 비동기 작업은 다음을 제공한다.

- 시작·완료·실패 구조화 로그
- correlation_id와 업무 엔티티 ID
- 처리시간 histogram
- 성공·재시도·terminal failure counter
- queue age
- 운영자가 볼 수 있는 마지막 오류와 재실행 버튼
- 사용자에게 보일 안전한 오류 메시지
- runbook 링크

새 퍼널 기능은 product event를 추가하되 이벤트 정의, 발생 시점, 중복 기준, 개인정보 금지를 함께 문서화한다.

## 11. Codex 작업 단위

한 번의 작업은 하나의 완결된 세로 기능 또는 하나의 구조적 기반으로 제한한다.

좋은 예:

- 공고번호 입력 → 공식 API 응답 보존 → canonical tender_version 생성 → 오류·중복·정정 계약 테스트
- HWPX 첨부 수집 → 악성검사 → 텍스트·표 추출 → evidence_span 저장 → 재처리 화면
- 회사 업종 fact → 특정 규칙팩 대조 → 근거 포함 finding → needs_review 흐름

나쁜 예:

- 조달 SaaS 전체 구현
- AI 붙이기
- 관리자 페이지 만들기
- 결제와 구독 기능 알아서 완성
- 물품·공사·용역 모두 지원

Codex 요청에는 수정 허용 경로와 금지 경로를 명시한다. 구현 후에는 변경 파일, 계약 변경, 마이그레이션, 테스트 결과, 알려진 제한을 반드시 보고하게 한다.

## 12. PR 승인 게이트

- [ ] 소유 모듈이 하나로 명확하다.
- [ ] 금지 import와 순환 의존 검사를 통과한다.
- [ ] 다른 모듈 테이블을 직접 읽지 않는다.
- [ ] 외부 DTO가 adapter 밖으로 나오지 않는다.
- [ ] 스키마 변경이 backward compatible하다.
- [ ] 정정·중복·재시도에 멱등하다.
- [ ] 개인정보·권한·감사 요구를 충족한다.
- [ ] 실패한 작업을 운영자가 찾고 복구할 수 있다.
- [ ] 기존 골든셋과 신규 사례가 통과한다.
- [ ] 기능 플래그와 중단 방법이 있다.
- [ ] 사용자 문구가 기능 한계를 숨기지 않는다.
- [ ] 관련 아키텍처·운영 문서가 갱신됐다.

## 13. 금지 패턴

- 페이지 컴포넌트에서 공식 API 직접 호출
- 한 서비스 파일에 수집·AI·판정·결제·알림 혼합
- isEligible boolean 하나로 전체 판단 저장
- 최신 공고로 기존 레코드 update
- 운영자가 DB를 직접 수정해 고객 결과 교정
- planName 문자열로 기능 접근 제어
- 고객별로 같은 공고와 첨부 원문 중복 저장
- queue 작업에 재시도만 있고 멱등성 없음
- 로그에 문서 본문·사업자번호·API 키 출력
- 테스트를 통과시키기 위한 실제 규칙 약화
- 제품 범위를 넓히면서 골든셋과 지원문구를 그대로 둠
