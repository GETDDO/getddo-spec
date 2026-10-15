# 감사 로그 API

- 상태: AU01/AU02 상세 계약 사용자 확인 완료 (2026-10-15)
- 공통 계약: [공통 API 계약](common.md)

2026-10-15 사용자는 아래 상세 계약과 확인된 변경 필드 공개, spec main 게시·Jira 작업 생성·구현을 확인했다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AU01 | `GET /admin/audit-logs` | `page,size,actorId,action,targetType,targetId,from,to` | 200 `Page<AuditLogSummary>` | 공통 |
| AU02 | `GET /admin/audit-logs/{auditLogId}` | 없음 | 200 `AuditLogDetail` | 공통 |

`AuditLogSummary`: `id:UUID`, `actorId:UUID?`, `action:string`, `targetType:string`, `targetId:UUID`, `reason:string?`, `requestId:string?`, `createdAt:instant`.

`AuditLogDetail`: Summary 전체 + `beforeData:object?`, `afterData:object?`. 상세 데이터는 관리자가 검토할 변경 근거에 한정하고 비밀값·무관한 개인정보를 기록하거나 응답하지 않는다. 감사 로그 수정·삭제 API는 제공하지 않는다.


## 목록·상세 계약

- 실제 경로는 `/api/v1/admin/audit-logs`와 `/api/v1/admin/audit-logs/{auditLogId}`다. 공통 DB 사용자 문맥의 ACTIVE ADMIN만 조회한다. 선택 멤버십 헤더도 기존 공통 대조를 따른다.
- page 기본 1·1 이상, size 기본 20·1~100이다. 정렬은 createdAt DESC·id DESC, totalElements는 필터 전체 건수다. 빈 결과·범위 밖 페이지는 빈 items로 반환한다.
- actorId·targetId는 정확한 UUID 검색, action·targetType은 코드 일치 검색이며 모든 전달 조건을 AND로 적용한다. 문자열 필터는 양끝 공백을 제거하고 빈 값은 해당 필터 없음으로 처리한다. 길이는 기존 기록 코드와 같은 최대 60자다. action/targetType의 현재 코드 예시는 아래 공개 필드 표를 따르며 목록 검색에서 알려지지 않은 코드를 임의로 거절하지 않는다.
- 기간은 createdAt의 UTC `[from,to)`다. 양끝은 선택이며 둘 다 있으면 from < to다. MySQL DATETIME(6)에서 다룰 수 있는 UTC 범위를 벗어난 조건은 거절한다. FE의 KST 날짜 입력은 UTC 경계로 바꿔 전달한다.
- 성공은 HTTP 200·기존 응답 봉투이며 목록은 Page<AuditLogSummary>, 상세는 AuditLogDetail이다. 시스템 기록 actorId와 저장되지 않은 reason/requestId는 null을 유지한다.
- UUID·시각·숫자 타입 해석 오류는 400 COMMON-005다. 페이지/크기·문자열 길이·기간 조건 위반은 400 AUDIT-001(`감사 로그 조회 조건이 올바르지 않습니다.`), 호출자 자격 위반은 403 AUDIT-002(`활성 관리자만 감사 로그를 조회할 수 있습니다.`), 없는 상세는 404 AUDIT-003(`감사 로그를 찾을 수 없습니다.`)이다. 기존 사용자 식별·헤더 대조 오류는 그대로 유지한다.

## 변경 전후 정보의 공개 범위

변경 전: 전후 JSON 공개 필드가 초안에서 구체화되지 않았다. 변경 후: 사용자가 **현재 확인된 업무 필드만 공개**하는 방식을 선택했다. 저장 원본은 변경하지 않고 조회 응답에서 필드를 선택한다. 아래 작업·대상 조합에 실제 존재하며 예상 형식에 맞는 필드만 공개한다.

| action | targetType | beforeData/afterData 공개 가능 필드 |
| --- | --- | --- |
| EVENT_CANCEL | EVENT | status,canceledAt,refundedTicketCount |
| EVENT_DELETE | EVENT | deletedAt |
| NOTIFICATION_JOB_RETRY | NOTIFICATION_JOB | status,attemptCount,attemptBase |
| NOTIFICATION_DELIVERY_RETRY | NOTIFICATION | status,attemptCount,attemptBase |
| DRAW_EXECUTE / DRAW_RESUME | DRAW_RUN | status,attemptId |
| AWARD_CANCEL | DRAW_RESULT | isCanceled,cancellationId,replacementRunId |

- 상태는 해당 업무의 기존 상태 코드, 횟수는 음수가 아닌 정수, 반환 수량은 음수가 아닌 정수 수량, 시각은 instant, ID는 UUID, isCanceled는 boolean으로 해석 가능한 값만 공개한다. 허용하지 않은 키·중첩 객체·예상 형식이 아닌 값은 제외한다.
- 전후 데이터 자체의 null과 허용된 키의 JSON null은 보존한다. 알려지지 않은 작업/대상 조합 또는 공개 가능한 필드가 없는 데이터는 null로 응답한다. 새 업무 필드의 공개는 담당 기능과 확인한 뒤 확장한다.
- 연락처·토큰·원문 요청·임의 JSON 전체를 공개하지 않는다. 상세가 존재해도 일부 변경 데이터가 공개되지 않을 수 있으며 이를 원본 삭제로 해석하지 않는다.

## 담당 경계와 연결

- A의 기록 기반과 각 업무의 실제 감사 기록을 재사용한다. C는 새 읽기 계약·조회 구현을 담당하고 기존 기록·인증 동작을 변경하지 않는다. FE의 실제 화면 연결은 담당자와 진행한다.
- 기존 FE의 `/admin/audit-logs` 호출은 baseURL에 따른 `/v1` 접두사 확인이 필요하다. 과거 action/targetType 선택지도 실제 기록 코드에 맞춰 연결한다. 기존 시연 데이터나 화면을 계약 채택만으로 수정하지 않는다.
- 감사 기록은 업무 트랜잭션과 함께 롤백된다. 이 조회는 보존된 감사 기록을 읽으며, 롤백된 실패 시도의 전체 이력을 대신 제공하지 않는다. 추첨 실패 내역의 추가 조회 계약은 별도 후속 사항을 따른다.
