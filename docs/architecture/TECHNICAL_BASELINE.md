# 기술 기준선

상태: Accepted  
기준일: 2026-07-27  
목적: 개발 작업마다 다른 프레임워크와 패턴이 섞이지 않게 첫 구현 기준을 고정한다.

## 1. 선택한 기준선

| 영역 | 선택 | 사용 범위 |
|---|---|---|
| 언어 | TypeScript strict | web, worker, 도메인, 계약 |
| 런타임 | 구현 시점의 Node.js Active LTS 고정 | web, worker, tooling |
| 패키지 관리 | pnpm workspace | 단일 lockfile과 패키지 경계 |
| 모노레포 작업 | Turborepo | build, test, lint, typecheck 캐시 |
| 웹 | Next.js App Router | UI, BFF, 짧은 request/response |
| 런타임 검증 | Zod | 환경변수, API·이벤트·adapter 응답 |
| 데이터베이스 | PostgreSQL | 단일 업무 기준 데이터 |
| SQL 접근 | Drizzle ORM + 명시적 SQL | 타입 안전 쿼리와 마이그레이션 |
| 비동기 작업 | PostgreSQL-backed queue adapter | 첫 규모의 worker 작업 |
| 이벤트 전달 | Transactional outbox | 모듈 간 신뢰 가능한 후속 처리 |
| 파일 | S3-compatible object storage | 원문, 첨부, 추출물, 증빙 |
| API 형식 | JSON over HTTPS, schema-first | web BFF와 모듈 외부 계약 |
| 관측성 | OpenTelemetry + 오류추적 adapter | trace, metric, error |
| 단위·통합 테스트 | Vitest + Testcontainers | domain, DB, adapter |
| 브라우저 테스트 | Playwright | 핵심 고객 흐름 |
| CI | GitHub Actions | 품질·보안·마이그레이션 게이트 |
| 로컬 환경 | Docker Compose | PostgreSQL, object storage, worker 의존성 |

버전은 package.json의 packageManager, lockfile, 런타임 버전 파일에 정확히 고정한다. Dependabot 또는 Renovate가 PR을 만들 수는 있지만 메이저 버전을 자동 병합하지 않는다.

## 2. 왜 TypeScript 한 언어인가

초기에는 작은 팀과 Codex가 web, worker, 계약, 도메인 규칙을 함께 관리한다. 동일한 타입과 테스트 도구를 공유하면 경계 계약이 어긋나는 위험이 줄어든다. 다만 HWP 같은 문서 처리에서 Python 또는 외부 서비스가 명확한 우위를 보이면 DocumentExtractor port 뒤의 별도 프로세스로 추가한다. 그 경우에도 핵심 eligibility 도메인을 옮기지 않는다.

## 3. Next.js 사용 경계

Next.js는 화면과 BFF다. 다음을 route handler나 server action 안에 직접 넣지 않는다.

- 공식 API DTO 변환
- 자격 판정 규칙
- 장시간 첨부파일 추출
- queue consumer
- 결제 webhook의 비즈니스 처리 전체
- 다른 모듈 테이블을 직접 조합하는 임시 query

route는 입력 검증, 인증 컨텍스트 생성, application use case 호출, response mapping까지만 담당한다. 시간이 오래 걸리는 작업은 job을 생성하고 상태 URL을 반환한다.

## 4. PostgreSQL과 Drizzle 경계

- 각 모듈이 자신의 schema 선언과 repository를 소유한다.
- platform/database가 connection, transaction, migration 실행을 제공한다.
- 다른 모듈의 Drizzle table object를 import하지 않는다.
- 복잡한 쿼리는 억지 ORM 표현보다 검토 가능한 명시적 SQL을 사용한다.
- 모든 마이그레이션은 순번, 소유 모듈, 롤아웃 단계가 드러나야 한다.
- 앱 시작 시 자동 마이그레이션하지 않는다. 배포 파이프라인의 별도 단계에서 실행한다.
- production query에는 timeout과 식별 가능한 query name을 둔다.
- money, timestamp, JSON schema, enum의 저장 규칙을 공통 기준으로 검사한다.

PostgreSQL을 검색엔진·queue·analytics 용도로 무한 확장하지 않는다. 실제 부하와 운영지표가 기준을 넘으면 해당 adapter 뒤에서 분리한다.

## 5. Queue 기준

첫 버전은 운영 구성요소를 줄이기 위해 PostgreSQL-backed queue를 사용한다. 라이브러리 API는 worker 코드 전체에 노출하지 않고 JobQueue port 뒤에 둔다.

작업 envelope 최소 필드:

~~~text
job_id
job_type
schema_version
idempotency_key
correlation_id
organization_id? 
subject_id
attempt
created_at
not_before
payload
~~~

모든 handler는 중복 실행을 견뎌야 한다. 최대 재시도 후 dead-letter 상태와 운영자 재처리 수단을 제공한다. 문서 본문과 비밀정보는 payload에 넣지 않고 object key 또는 entity ID만 전달한다.

다음 조건이 지속되면 별도 managed queue를 검토한다.

- PostgreSQL 업무 쿼리에 queue 부하가 영향을 준다.
- 우선순위·예약·처리량 요구를 현재 adapter로 충족하지 못한다.
- worker가 여러 지역 또는 독립 서비스로 분리된다.
- dead-letter와 재처리 운영비가 공급자 서비스보다 커진다.

## 6. 인증과 권한

특정 인증 공급자의 user object를 도메인에 저장하지 않는다. identity-access가 외부 subject를 내부 user_id에 연결한다.

- 외부 인증은 OIDC 또는 검증 가능한 서버 SDK를 사용한다.
- authorization은 서버의 policy에서 결정한다.
- UI에서 버튼을 숨기는 것은 보안 경계가 아니다.
- organization owner, admin, member, reviewer, support 역할을 capability로 변환한다.
- 결제 entitlement와 사용자 role은 분리한다.
- 운영자 impersonation이 필요하면 시간 제한, 목적, 승인, 감사가 있는 별도 기능으로 만든다.

공급자 선정은 실제 법인 요금, 국내 사용자 로그인 방식, B2B 조직관리 요구를 비교한 별도 ADR로 확정한다.

## 7. 결제

commerce 모듈은 provider-neutral order, payment, refund, entitlement를 소유한다. 국내 결제대행사 SDK와 webhook은 payments adapter에 둔다.

- 주문 금액은 서버에서 offer version으로 계산한다.
- 클라이언트 금액을 신뢰하지 않는다.
- webhook 서명과 중복 event를 검증한다.
- 결제 승인과 entitlement 부여를 같은 함수 호출 성공으로 가정하지 않는다.
- outbox와 보상 작업으로 불일치를 복구한다.
- 카드정보는 저장하지 않는다.
- 세금계산서·현금영수증·환불정책은 출시 전 실제 PG·회계 운영과 검증한다.

PG 공급자와 계약방식은 수수료·정산·B2B 증빙 요구 확인 후 별도 ADR로 고정한다.

## 8. 파일과 문서

- 업로드와 공식 첨부는 object storage에 불변 원본으로 저장한다.
- presigned URL은 짧은 만료시간과 정확한 object 권한을 갖는다.
- 파일명은 표시용이며 object key로 사용하지 않는다.
- 브라우저가 보낸 MIME을 신뢰하지 않는다.
- 추출기는 별도 sandbox 또는 제한된 worker에서 실행한다.
- 원문, 정규화본, 추출 block, evidence span을 구분한다.
- 삭제는 DB와 object storage를 함께 추적하는 보상 가능한 작업으로 수행한다.

## 9. AI와 규칙 엔진

AI SDK는 integrations/ai에만 둔다. core module은 provider-neutral structured request와 result를 사용한다.

- 출력 schema 검증 실패는 재시도 또는 needs_review다.
- temperature, prompt, model, provider, token usage를 run metadata로 기록한다.
- 원문 전체를 무조건 전송하지 않고 필요한 구간과 데이터 분류 정책을 적용한다.
- 고객 파일이 모델 학습에 사용되는지 공급자 계약과 설정을 확인한다.
- AI가 만든 후보를 deterministic validation과 evidence 연결 없이 finding으로 승격하지 않는다.
- 모델 교체는 shadow comparison과 골든셋을 통과해야 한다.

## 10. 관측성과 오류 모델

공통 오류는 다음 범주로 제한한다.

- validation
- unauthorized
- forbidden
- not_found
- conflict
- rate_limited
- external_dependency
- retryable
- unsupported
- internal

내부 stack과 민감정보는 고객 응답에 포함하지 않는다. 모든 오류는 stable error code, trace_id, 안전한 message를 갖는다.

OpenTelemetry context는 web → outbox → queue → worker → external adapter까지 이어간다. 오류추적 공급자는 adapter로 감싸되 구조화 로그와 핵심 메트릭은 공급자를 바꿔도 유지한다.

## 11. 환경과 배포

최소 환경:

- local
- test
- staging
- production

staging은 production과 같은 마이그레이션·queue·object storage 경계를 사용하고 데이터만 합성·익명화한다. 실제 고객 문서를 staging에 복제하지 않는다.

배포 순서:

1. 정적 검사와 테스트
2. build artifact 생성과 SBOM
3. 마이그레이션 expand 단계
4. web·worker 호환 버전 배포
5. smoke test
6. 기능 플래그 제한 활성화
7. 메트릭 관찰
8. 점진 확대
9. 이후 contract 마이그레이션

worker와 web은 같은 commit SHA와 contract version을 표시한다. 배포가 섞여도 event와 DB schema가 호환되도록 한다.

## 12. 코드 품질 설정

초기 scaffold에 다음을 반드시 넣는다.

- TypeScript strict, noUncheckedIndexedAccess
- ESLint import boundary 규칙
- package exports
- circular dependency 검사
- format과 lint의 CI 고정
- secret scanning
- dependency vulnerability scan
- migration verification
- unit, integration, architecture, E2E test 분리
- coverage 수치보다 중요 도메인 분기 테스트 강제
- CODEOWNERS 또는 승인 규칙
- PR template과 ADR template

## 13. 아직 선택하지 않는 것

다음은 실제 요구나 공급자 계약 전까지 확정하지 않는다.

- 클라우드 사업자
- 인증 공급자
- 결제대행사
- managed queue
- 별도 검색엔진
- 데이터웨어하우스
- AI 모델과 OCR 공급자
- 이메일·문자 공급자

미정은 “매번 아무거나 선택”을 뜻하지 않는다. 반드시 기존 port를 구현하고 비교 근거를 ADR로 남긴 뒤 production에 넣는다.
