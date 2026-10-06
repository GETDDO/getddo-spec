# 알림 API 계약

- 상태: 사용자 알림 N01~N03 확정 / 관리자 알림 AN01~AN05 검토 대기
- 사용자 알림 확정일: 2026-09-30
- 확정 근거: 사용자의 현재 구현 기준 N01~N03 및 spec 확정 지시 (GD-65)
- 공통 계약: [공통 API 계약](common.md) — 문서 전체는 초안이며 N01~N03에 적용하는 계약은 아래에 명시한다.
- 도메인 규칙: [알림 규칙](../02-domain/notification.md)
- 주 담당: 전민규 — 사용자 알림·읽음 처리, 비동기 생성·모의 발송·재시도, 관리자 알림 운영

확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다. 관리자 알림의 경로·메서드·DTO·업무 오류 코드는 담당자 확인 전 제안이다.

## 사용자 알림 — N01~N03 확정

### 경로·사용자 문맥·응답

- 기본 경로는 `/api/v1`이다. 아래 표에는 실제 호출 경로를 기재한다.
- USER와 ADMIN 모두 자신의 알림을 조회·읽음 처리할 수 있다. 관리자가 다른 사용자의 알림을 대신 읽음 처리할 수 없다.
- `X-User-ID`는 필수이며 등록된 사용자 UUID다. `X-User-Membership`은 선택이며 `excellent`, `vip`, `vvip` 중 하나다. 전달하면 DB 멤버십과 일치해야 하며 사용자 정보는 변경하지 않는다.
- 사용자 헤더는 [시연용 사용자 문맥](common.md#시연용-사용자-헤더--사용자-요청을-반영한-권고안)이다. 개발·통제된 시연 환경에 한정하며 신원 인증으로 사용하지 않는다.
- 사용자 문맥의 공통 처리 위치·연결 방식과 미확정 사용자 상태 처리는 [담당자 후속 작업](../00-requirements/pending-decisions.md#사용자-문맥과-알림-api)을 따른다.
- 요청·응답은 JSON이며 성공 HTTP 상태는 세 API 모두 200이다. N02·N03 요청 본문은 없다.
- 모든 응답은 `success:boolean`, `code:string`, `message:string`, `data` 봉투를 사용한다. 성공은 `success=true`, `code=SUCCESS`, `message=성공했습니다.`다. 실패는 실제 오류 HTTP 상태와 `success=false`, 해당 오류의 코드·메시지, `data=null`을 반환한다.
- 아래 응답 타입은 `data` 내부다. 필드명은 camelCase이며 모든 명시 필드를 반환한다. `?` 필드는 값이 없을 때 null이며 빈 목록은 `[]`이다. UUID는 문자열, `createdAt`은 UTC ISO-8601 `Z`, `long`은 JSON 정수다.

| ID | 권한 | 메서드·경로 | 요청 | 성공 `data` | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| N01 | U/A | `GET /api/v1/notifications/me` | 선택 쿼리 `cursor,size,isRead` | `Cursor<Notification>` | 조회 조건·사용자 문맥 오류 |
| N02 | U/A | `PUT /api/v1/notifications/{notificationId}/read` | 알림 UUID, 본문 없음 | `NotificationReadResult` | 타인 소유·없는 알림 모두 404 |
| N03 | U/A | `PUT /api/v1/notifications/me/read-all` | 본문 없음 | `{updatedCount:long}` | 사용자 문맥 오류 |

### N01 — 내 알림 목록

| 쿼리 | 형식·기본값 | 처리 |
| --- | --- | --- |
| `cursor` | 선택 문자열, 최초 조회는 생략 | 이전 응답의 `nextCursor`를 그대로 전달한다. 클라이언트가 해석하거나 생성하지 않는다. |
| `size` | 정수, 기본 20, 범위 1~100 | 한 번에 반환할 최대 건수 |
| `isRead` | 선택 boolean | 생략하면 전체, true면 읽음, false면 미읽음만 조회 |

- 정렬은 `createdAt DESC, id DESC`다. 같은 생성 시각에도 ID를 기준으로 순서를 고정한다.
- `Cursor<Notification>`은 `items:Notification[]`, `nextCursor:string?`, `totalElements:long`이다. 다음 목록이 없으면 `nextCursor=null`이다.
- `totalElements`는 본인 및 읽음 필터 조건에 맞는 전체 건수이며 커서 이후 남은 건수가 아니다.
- `Notification`은 `id:UUID`, `title:string`, `body:string`, `createdAt:instant`, `isRead:boolean`, `eventId:UUID?`, `linkUrl:string?`이다.
- `eventId`와 `linkUrl`은 관련 정보가 없으면 null이다. 이 계약에서 특정 프론트엔드 화면 경로를 새로 정하지 않으며 응답의 링크 값을 사용한다.
- 목록 조회는 읽음 상태를 변경하지 않는다.

### N02 — 개별 읽음

- 성공 응답 `NotificationReadResult`는 `id:UUID`, `isRead:true`다.
- 본인 알림만 처리한다. 타인 소유 알림과 없는 알림은 모두 HTTP 404, `NOTIFICATION-002`로 반환하여 존재 여부를 구분해 노출하지 않는다.
- 이미 읽은 본인 알림에 재요청해도 HTTP 200과 같은 읽음 응답을 반환한다.
- 사용자가 알림을 클릭하면 N02를 호출한다. 읽음 시각은 저장하거나 반환하지 않는다.

### N03 — 전체 읽음

- 본인의 미읽음 알림 전체를 변경한다. 현재 화면의 목록·페이지·읽음 필터에 한정하지 않는다.
- `updatedCount`는 실제 변경 건수이며 이미 읽은 알림은 제외한다. 처리할 미읽음이 없거나 완료 후 재요청하면 0이다.
- 다른 사용자의 알림과 모의 발송 상태는 변경하지 않는다. 모의 발송 성공만으로 읽음 처리하지 않는다.

### 오류 계약

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON-005` | UUID·쿼리 타입 해석 실패 또는 선택 멤버십 형식 오류 |
| 400 | `NOTIFICATION-001` | size 범위 위반 또는 커서 형식 오류 |
| 401 | `USER_CONTEXT_REQUIRED` | 필수 `X-User-ID` 누락 |
| 401 | `USER_CONTEXT_INVALID` | 미등록 사용자 ID |
| 409 | `USER_MEMBERSHIP_MISMATCH` | 선택 멤버십과 DB 값 불일치 |
| 404 | `NOTIFICATION-002` | N02의 알림이 없거나 본인 소유가 아님 |
| 500 | `COMMON-001` | 예상하지 못한 서버 오류 |

그 밖의 표준 HTTP·입력 오류는 [공통 오류](common.md#오류)의 현행 응답 형식을 따른다. 공통 사용자 처리로 통합할 때에도 확정된 N01~N03 공개 계약과의 호환성을 확인한다.

## 관리자 알림 운영

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AN01 | `GET /admin/notification-jobs` | `page,size,status,notificationType,eventId,from,to` | 200 `Page<NotificationJob>` | 공통 |
| AN02 | `GET /admin/notification-jobs/{jobId}` | 없음 | 200 `NotificationJobDetail` | 공통 |
| AN03 | `POST /admin/notification-jobs/{jobId}/retry` | 본문 없음 | 202 `NotificationJob` | 409 재처리 불가 상태 |
| AN04 | `GET /admin/notifications` | `page,size,userId,eventId,jobId,mockDeliveryStatus,from,to` | 200 `Page<AdminNotification>` | 공통 |
| AN05 | `POST /admin/notifications/{notificationId}/delivery-retries` | 본문 없음 | 202 `AdminNotification` | 409 재처리 불가 상태 |

`NotificationJob`: `id:UUID`, `eventId:UUID?`, `publicationId:UUID?`, `targetUserId:UUID?`, `notificationType:NotificationType`, `status:PENDING/PROCESSING/COMPLETED/FAILED`, `attemptCount:int`, `scheduledAt:instant`, `nextAttemptAt:instant?`, `createdAt:instant`, `completedAt:instant?`.

`NotificationType`: `EVENT_START`, `RESULT_PUBLISHED`, `ENTRY_EXCLUDED`, `TICKET_REVOKED`, `EVENT_SCHEDULE_CHANGED`, `EVENT_CANCELED`, `RESULT_CHANGED`.

`NotificationJobDetail`: NotificationJob 전체 + `lastError:string?`, `sourceJobId:UUID?`, `lastProcessedUserId:UUID?`, `leaseUntil:instant?`. `AdminNotification`: Notification 전체 + `jobId:UUID`, `userId:UUID`, `mockDeliveryStatus:PENDING/SENT/FAILED`, `mockSentAt:instant?`, `deliveryAttemptCount:int`, `nextDeliveryAttemptAt:instant?`, `lastDeliveryError:string?`.

AN03은 생성 작업 실패, AN05는 개별 모의 발송 실패의 재처리다. 재시도로 동일 알림을 복제하거나 읽음 상태를 변경하지 않는다. 최종 실패 여부·허용 재시도 상태의 구체 판정은 담당자의 운영 설정으로 정의한다. 관리자 목록 조회가 타인의 알림 읽음을 대신 처리하지 않는다.
