# 공식 데이터 연동 아키텍처

상태: Accepted  
검증일: 2026-07-27  
목적: 나라장터·공공데이터 변경이 제품 전체로 전파되지 않게 하고, 고객에게 보여준 판단의 당시 근거를 재현한다.

## 1. 확인된 공식 데이터

| 목적 | 공식 서비스 | 제공 내용 | 초기 사용 |
|---|---|---|---|
| 공고 수집 | 조달청 나라장터 입찰공고정보서비스 | 물품·용역·공사·외자 공고 목록·상세, 기초금액, 면허제한, 참가가능지역, 변경이력 | 필수 |
| 회사 정보 | 조달청 나라장터 사용자정보 서비스 | 조달업체 기본정보, 등록업종, 공급물품, 부정당제재업체 정보 | 필수 |
| 업종·법규 | 조달청 나라장터 업종 및 근거법규서비스 | 업종코드, 포함·제한·허용업종, 근거법령 링크 | 필수 |
| 결과 추적 | 조달청 나라장터 낙찰정보서비스 | 개찰·낙찰·순위·예비가격 관련 정보 | 제출 후 |
| 과정 연결 | 조달청 나라장터 계약과정통합공개서비스 | 공고번호 등으로 사전규격·공고·낙찰·계약 진행과정 연결 | 확장 |
| 사전 수요 | 조달청 나라장터 사전규격정보서비스 | 사전규격, 예산, 규격서 파일, 의견 | 리드·확장 |
| 계약 후 | 조달청 나라장터 계약정보서비스 | 계약상세, 변경·삭제 이력, 업무별 계약현황 | 낙찰 후 OS |

공식 문서:

- 입찰공고정보서비스: https://www.data.go.kr/data/15129394/openapi.do
- 사용자정보 서비스: https://www.data.go.kr/data/15129466/openapi.do
- 업종 및 근거법규서비스: https://www.data.go.kr/data/15129467/openapi.do
- 낙찰정보서비스: https://www.data.go.kr/data/15129397/openapi.do
- 계약과정통합공개서비스: https://www.data.go.kr/data/15129459/openapi.do
- 사전규격정보서비스: https://www.data.go.kr/data/15129437/openapi.do
- 계약정보서비스: https://www.data.go.kr/data/15129427/openapi.do

## 2. 검증 결과가 설계에 주는 의미

- 입찰공고 API는 업무구분별 오퍼레이션을 사용해야 한다. 공사·용역·물품을 하나의 외부 호출 형태로 가정하면 안 된다.
- 공고 서비스가 변경이력, 면허제한, 참가가능지역을 제공하므로 공식 구조화 데이터를 우선하고 첨부문서 추출은 보완 근거로 사용한다.
- 사용자정보 API는 업체 기본정보·등록업종·공급물품을 제공하지만 고객이 가진 모든 자격·실적·증빙을 보장하지 않는다. 공식 조회값과 고객 확인값을 서로 다른 출처로 보존해야 한다.
- 업종·근거법규 서비스는 실시간이지만 법령 변경시점과 코드 등록시점에 차이가 있을 수 있다고 공식적으로 안내한다. 따라서 업종코드만으로 법적 결론을 단정하지 않는다.
- 일부 조달 과정에는 사전규격·입찰·낙찰·개찰 정보가 존재하지 않을 수 있다. 데이터 부재를 곧바로 업무 미발생으로 해석하지 않는다.
- 개발계정 기본 트래픽은 서비스에 따라 1,000회 또는 10,000회이며 운영계정은 활용사례 등록 후 증량 신청 구조다. 초기부터 중복 제거, 캐시, 호출 예산과 제한 대응이 필요하다.
- 기존 17종 API가 2024-12-31 중단되고 2025-01-06 대체서비스가 개방된 전례가 있다. 또한 2026년 사용자정보 URL 변경과 기존 URL 90일 유지 공지가 있었다. 엔드포인트와 DTO를 핵심 코드에 직접 넣으면 안 된다.

변경 공지:

- 17종 구 API 중단 및 대체서비스: https://www.data.go.kr/bbs/ntc/selectNotice.do?originId=NOTICE_0000000003985
- 사용자정보 서비스 URL 변경: https://www.data.go.kr/bbs/ntc/selectNotice.do?originId=NOTICE_0000000004468

## 3. Anti-Corruption Layer

외부 서비스별 adapter가 외부 요청·응답을 내부 표준 모델로 변환한다.

~~~text
공식 API DTO
→ 응답 검증
→ raw snapshot 보존
→ source-specific mapper
→ canonical model
→ domain command
~~~

도메인과 UI는 apis.data.go.kr의 필드명, XML 구조, 서비스 URL을 알지 못한다.

초기 포트 예시:

~~~text
ProcurementNoticeSource
- resolveNotice(input)
- fetchNotice(identity)
- listNoticeRevisions(identity)
- fetchQualificationRestrictions(identity)
- fetchAllowedRegions(identity)
- listAttachments(identity)

SupplierRegistrySource
- fetchSupplier(businessIdentity)
- listRegisteredIndustries(supplierIdentity)
- listSuppliedProducts(supplierIdentity)
- findSanctions(supplierIdentity)

AwardSource
- fetchOpeningResult(noticeIdentity)
- fetchAwardResult(noticeIdentity)
~~~

각 반환값에는 source_name, source_record_id, retrieved_at, source_updated_at, schema_version, content_hash를 포함한다.

## 4. 원본과 표준 데이터

### source_snapshot

외부 호출마다 다음 메타데이터를 PostgreSQL에 저장하고 원문 body는 Object Storage에 저장한다.

- source_name과 operation
- request_fingerprint
- HTTP 상태와 공급자 결과코드
- requested_at, received_at
- response_schema_version
- body_object_key
- content_hash
- parser_version
- contains_sensitive_data
- retention_class

API 키, Authorization 헤더, 전체 사업자번호는 저장하지 않는다. 같은 content_hash면 중복 body 저장을 피할 수 있지만 수집 시각 기록은 남긴다.

### canonical model

외부 필드를 내부 표준 용어로 바꾼다.

- NoticeIdentity
- NoticeVersion
- ProcurementCategory
- ProcuringEntity
- Schedule
- Amount
- QualificationRestriction
- RegionRestriction
- AttachmentReference
- SourceProvenance

원본에만 있는 필드는 source_metadata에 제한적으로 보관하되, 업무 로직이 이 bag을 직접 읽는 것을 금지한다. 필요한 필드는 mapper와 canonical schema를 정식으로 변경한다.

## 5. 수집과 변경 감지

### 처음 공고를 추가할 때

1. URL 또는 번호를 파싱한다.
2. 공고 업무구분과 공식 식별자를 판별한다.
3. 올바른 업무별 오퍼레이션을 선택한다.
4. 공고 상세, 변경이력, 제한정보, 첨부 참조를 수집한다.
5. raw snapshot과 해시를 저장한다.
6. canonical validation을 통과한 경우 tender_version을 만든다.
7. 첨부파일 작업과 분석 작업을 큐에 등록한다.

### 감시 주기

고객의 활성 bid_case가 있는 공고를 가장 자주 확인한다.

- 마감 24시간 이내: 10분 이내 변경 확인 목표
- 마감 1~3일: 30분 이내
- 마감 4~14일: 2시간 이내
- 그 외 활성 공고: 12시간 이내
- 종료 공고의 결과: 개찰 예정 전후로 별도 스케줄

이는 제품 목표이며 실제 API 제한과 운영계정 승인량에 맞춰 조정한다. 호출 예산을 넘기면 마감이 임박한 유료 고객 공고를 우선하고 사용자에게 데이터 갱신 시각을 표시한다.

### 변경 판정

외부 updated_at만 신뢰하지 않고 정규화된 핵심 필드와 첨부파일 목록의 content hash를 비교한다. 변경이면 새 tender_version을 만들고 다음을 실행한다.

- 이전 분석 superseded
- 새 분석 예약
- 바뀐 필드와 근거 비교 생성
- 고객 할 일 영향 계산
- 마감·자격·제출서류 변경은 고우선 알림
- 운영자 영향범위 화면에 노출

## 6. 실패와 재시도

오류를 다음으로 구분한다.

| 분류 | 예 | 처리 |
|---|---|---|
| retryable | timeout, 429, 5xx, 공급자 일시 장애 | 지수 백오프와 jitter, 최대 횟수 후 운영 큐 |
| credential | API 키 만료, 권한 거절 | 자동 반복 금지, 즉시 운영 알림 |
| schema_changed | 필수 필드 누락, 예상하지 못한 타입 | raw 보존, adapter 차단, 계약 테스트 실패 |
| not_found | 잘못된 번호, 아직 반영 전 | 짧은 재확인 후 사용자에게 확인 요청 |
| unsupported | 첫 범위 밖 업무구분·문서 형식 | needs_review와 명확한 안내 |
| data_conflict | API와 첨부문서의 마감·조건 불일치 | 자동 eligible 금지, 사람 검수 |
| terminal | 취소 공고, 접근 불가 원문 | 상태 확정과 후속 작업 중지 |

circuit breaker는 소스·오퍼레이션 단위로 둔다. 한 API 장애가 결제, 기존 결과 조회, 사용자 할 일 체크까지 막지 않아야 한다.

## 7. 호출 절약과 성능

- request_fingerprint로 동시에 들어온 동일 요청을 합친다.
- 공고 원문은 전 고객이 공유하고 고객별 분석만 분리한다.
- 공식 updated_at과 ETag/Last-Modified가 신뢰 가능한 경우 조건부 요청을 사용한다.
- source별 token bucket으로 호출 한도를 관리한다.
- 캐시 만료와 마지막 성공 수집시각을 사용자 화면에 표시한다.
- 운영계정 트래픽 증량 신청 전에 예상 피크, 공고당 호출수, 재시도율을 측정한다.
- 전체 검색 수집과 고객이 지정한 공고 감시를 다른 우선순위 큐로 분리한다.

## 8. 첨부파일 처리

1. 허용된 공식 도메인과 리다이렉트 목적지를 검증한다.
2. 최대 크기와 다운로드 시간을 제한한다.
3. 원본 파일의 SHA-256, MIME 추정값, 확장자, 수집 URL과 시각을 기록한다.
4. 악성파일 검사가 통과해야 추출한다.
5. 원본은 불변으로 저장한다.
6. 추출 결과는 extractor_version과 연결한다.
7. 페이지, 표, 셀, 문단 위치를 evidence_span으로 만든다.
8. OCR·파싱 실패는 문서가 없는 것으로 처리하지 않고 needs_review로 보낸다.

HWP, HWPX, PDF, DOCX, XLSX 등 형식별 추출기는 공통 DocumentExtractor 포트를 구현한다. 새 형식 추가가 eligibility 모듈 변경으로 이어져서는 안 된다.

## 9. 데이터 충돌의 우선순위

일률적인 한 줄 우선순위를 두지 않는다. 필드 종류와 출처에 따라 정책을 버전 관리한다.

기본 원칙:

- 공식 구조화 변경이력은 공고 버전 판정에 우선 사용한다.
- 공고문·과업지시서·제안요청서의 명시 조건은 evidence로 보존한다.
- 구조화 필드와 첨부 원문이 충돌하면 최신 수집시각만으로 해결하지 않는다.
- 마감, 참가자격, 지역제한, 면허, 제출서류 충돌은 needs_review다.
- 운영자 결정은 원본을 바꾸지 않고 override 기록으로 추가한다.
- 고객에게는 적용한 출처, 수집시각, 충돌과 확인 필요 상태를 표시한다.

## 10. AI 사용 경계

AI가 해도 되는 일:

- 문단·표에서 후보 요구조건 추출
- 공식 용어를 쉬운 말로 설명할 초안 생성
- 비표준 문서 분류와 중복 후보 탐지
- 사람 검수의 우선순위 제안

AI가 단독으로 하면 안 되는 일:

- 근거 위치 없는 참가 가능 확정
- 미지원 법규의 법적 결론
- 공고 원문과 다른 마감·금액 생성
- 회사의 미확인 자격을 보유로 간주
- 제출 완료를 추정

AI 산출물은 extraction_candidate로 저장하고, 결정 규칙이나 사람 검수를 거쳐야 finding이 된다.

## 11. 계약 테스트

외부 adapter마다 익명화한 fixture와 live smoke test를 둔다.

- 정상 JSON과 XML
- 빈 목록과 누락 가능한 필드
- 업무구분별 응답 차이
- 정정·취소 공고
- 한글·특수문자·시간대
- 결과코드는 성공이지만 body가 비정상인 경우
- 429, 5xx, timeout
- API URL과 스키마 변경 감지

live smoke test는 비밀키가 있는 예약 환경에서만 실행하고 응답 원문을 공개 CI 로그에 남기지 않는다. fixture가 바뀔 때 canonical model의 차이를 사람이 검토한다.

## 12. 운영 체크리스트

- [ ] 데이터 이용 신청과 운영계정 증량 계획
- [ ] 서비스별 담당자·장애 공지 구독
- [ ] API 키 회전과 비밀관리
- [ ] 소스별 요청량·오류·지연 대시보드
- [ ] 마지막 성공 수집시각과 데이터 신선도 알림
- [ ] 스키마 변경 시 adapter만 수정되는지 확인
- [ ] 정정공고 골든셋 회귀
- [ ] 데이터 부재와 업무 부재를 구분하는 문구
- [ ] 고객에게 공식 원문 링크와 수집시각 제공
- [ ] 공급자 장애 시 기능 저하 모드와 지원 안내
