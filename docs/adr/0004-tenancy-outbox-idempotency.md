# ADR-0004: 테넌트 격리, Outbox와 멱등 작업

상태: Accepted  
결정일: 2026-07-27

## 맥락

한 공고 원문은 여러 고객이 공유할 수 있지만 회사정보, 분석, 문서, 결제와 진행상태는 조직별로 격리해야 한다. 또한 공고 수집과 분석은 외부 API·파일·AI를 거치는 장시간 작업이라 중복 실행과 부분 실패가 불가피하다.

## 결정

- 고객 소유 레코드는 organization_id를 필수로 갖는다.
- 테넌트 범위는 인증 컨텍스트에서 결정하고 저장소 필터와 DB 정책으로 이중 검사한다.
- 공고 원문 같은 공용 데이터와 회사별 데이터를 논리적으로 분리한다.
- 업무 변경과 outbox_event를 같은 DB 트랜잭션에 저장한다.
- consumer는 event_id와 idempotency_key의 unique constraint로 중복을 제거한다.
- 작업은 retryable, credential, schema_changed, unsupported, terminal 오류로 분류한다.
- 운영자 접근·재처리·override는 사유와 audit_event를 남긴다.
- 이벤트 payload에는 민감정보를 넣지 않는다.

## 결과

장점:

- 한 고객의 데이터가 다른 고객에게 노출될 가능성을 구조적으로 줄인다.
- 프로세스 중단 후에도 후속 작업을 유실하지 않는다.
- 중복 webhook·queue delivery·재시도에서 중복결제와 중복분석을 방지한다.
- 장애 영향과 운영자 조치를 추적할 수 있다.

비용:

- 모든 repository와 테스트 fixture가 조직 범위를 다뤄야 한다.
- outbox 전달자, dead-letter 운영, 재처리 UI가 필요하다.
- 공용 공고와 고객별 산출물의 보존·삭제 정책을 따로 관리해야 한다.
