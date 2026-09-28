# 알림 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [알림 규칙](../02-domain/notification.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 사용자 알림

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| N01 | U/A | `GET /notifications/me` | `cursor,size,isRead` | 200 `Cursor<Notification>` | 공통 |
| N02 | U/A | `PUT /notifications/{notificationId}/read` | 본문 없음 | 200 `NotificationReadResult` | 타인 소유 403/404 |
| N03 | U/A | `PUT /notifications/me/read-all` | 본문 없음 | 200 `{updatedCount:long}` | 공통 |

`Notification`: `id:UUID`, `title:string`, `body:string`, `createdAt:instant`, `isRead:boolean`, `eventId:UUID?`, `linkUrl:string?`.

`NotificationReadResult`: `id:UUID`, `isRead:true`. 읽음 시각 필드는 없다. N01만 호출하면 읽음 상태가 바뀌지 않으며 클릭 시 N02를 호출한다. N03은 처리 대상 본인의 미읽음 알림만 변경하고 실제 변경 건수를 반환한다. 모의 발송 성공 여부와 읽음 상태를 분리한다.

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
