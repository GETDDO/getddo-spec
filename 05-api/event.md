# 이벤트 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [이벤트 규칙](../02-domain/event.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 사용자 이벤트 조회

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| E01 | R | `GET /events` | `page,size,status,eventType,membershipRule` | 200 `Page<EventSummary>` | 공통 |
| E02 | R | `GET /events/{eventId}` | 없음 | 200 `EventDetail` | 공통 |

| DTO | 필드 |
| --- | --- |
| `EventSummary` | `id:UUID`, `title:string`, `imageUrl:string?`, `eventType:NO_TICKET/TICKET`, `weightingEnabled:boolean`, `membershipRule:excellent/vip/vvip`, `startsAt:instant`, `endsAt:instant`, `status:EventStatus`, `publicationScheduledAt:instant`, `serverTime:instant` |
| `EventDetail` | EventSummary 전체 + `description:string`, `maxTicketsPerUser:int?`, `prizes:Prize[]` |
| `Prize` | `id:UUID`, `rank:int`, `name:string`, `description:string?`, `imageUrl:string?`, `winnerCount:int` |

`EventStatus`는 관리자 상태 값 `SCHEDULED`, `OPEN`, `CLOSED`, `DRAW_CONFIRMED`, `PUBLISHED`, `SUSPENDED`, `CANCELED`, `REDRAWING`, `NO_ENTRANTS`, `NO_ELIGIBLE_ENTRANTS`에 대응한다. 관리자에게는 운영 상태를 제공하지만 사용자 목록·상세에서 `REDRAWING`을 그대로 노출하지 않는다. 이미 발표된 이벤트는 공개 상태를 유지하고 최초 발표 전에는 기존 대기 상태로 표현하도록 제안한다. 공개 결과에는 별도 공개 여부를 사용하며 이벤트 상태만으로 당첨 정보를 노출하지 않는다. 구체적인 중간 결과 표시 범위는 [결과 조회](drawing.md#공개-결과와-본인-결과)의 공개 중간 상태를 따른다.

## 관리자 이벤트 등록·운영

권한 A. 이벤트 CRUD는 박지훈, 상태 운영·배너는 전민규, 반환은 윤태형, 추첨 차단은 석종수와 협의한다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AE01 | `GET /admin/events` | `page,size,keyword,status,eventType,from,to` | 200 `Page<AdminEvent>` | 공통 |
| AE02 | `GET /admin/events/{eventId}` | 없음 | 200 `AdminEvent` | 공통 |
| AE03 | `POST /admin/events` | `EventWriteRequest` | 201 `AdminEvent` | 400 기간·경품·조건 오류 |
| AE04 | `PUT /admin/events/{eventId}` | `EventWriteRequest` | 200 `AdminEvent` | 409 수정 불가 상태·시작 앞당김 |
| AE05 | `DELETE /admin/events/{eventId}` | 없음 | 200 `null` | 409 시작/응모 이력 존재, 삭제 이벤트의 알림 처리 계약 |
| AE06 | `POST /admin/events/{eventId}/suspend` | `ReasonRequest` | 200 `EventOperationResult` | 409 허용되지 않은 상태 |
| AE07 | `POST /admin/events/{eventId}/resume` | `ReasonRequest` | 200 `EventOperationResult` | 409 중단 아님·마감·취소 |
| AE08 | `POST /admin/events/{eventId}/cancel` | `ReasonRequest` | 200 `EventOperationResult` | 반환 실패 시 성공 금지 |

## 이벤트·경품 동시 등록 및 수정

| `EventWriteRequest` 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| title | string | 필수 | 1~200자 |
| description | string | 필수 | 공백만 입력 불가 제안 |
| imageKey | string/null | 선택 | 최대 500자, 이미지 처리 방식은 담당 기능 계약에 따름 |
| eventType | NO_TICKET/TICKET | 필수 | 이벤트 유형 |
| weightingEnabled | boolean | 필수 | NO_TICKET은 false |
| maxTicketsPerUser | int/null | 필수 | 미사용 null, 사용·미가중치 1, 사용·일반 가중치 5, 사용·월말 소진용 null 제안. 월말 운영 허용 조건은 담당자 계약에 따름 |
| membershipRule | excellent/vip/vvip | 필수 | 최소 등급 |
| startsAt / endsAt | instant | 필수 | 종료 > 시작, 명시적 오프셋 |
| prizes | PrizeWrite[] | 필수 | 비어 있지 않음, 등수 중복 금지 |

`PrizeWrite`: 선택 `id:UUID`(수정 시 기존 경품), 필수 `rank:int(≥1)`, `name:string(1~200)`, `winnerCount:int(≥1)`, 선택 `description:string/null`, `imageKey:string/null(≤500)`.

```json
{
  "title": "가을 경품 이벤트",
  "description": "유효한 응모권으로 응모하는 이벤트",
  "imageKey": null,
  "eventType": "TICKET",
  "weightingEnabled": true,
  "maxTicketsPerUser": 5,
  "membershipRule": "vip",
  "startsAt": "2026-09-29T18:00:00+09:00",
  "endsAt": "2026-09-29T18:20:00+09:00",
  "prizes": [
    { "rank": 1, "name": "경품 A", "description": null, "imageKey": null, "winnerCount": 1 },
    { "rank": 2, "name": "경품 B", "description": null, "imageKey": null, "winnerCount": 3 }
  ]
}
```

생성자는 헤더에서 조회하고 초기 상태는 서버가 정한다. 이벤트와 경품은 한 트랜잭션으로 등록한다. 독립적인 경품 생성 후 이벤트 등록을 완료하는 API로 쪼개지 않는다. 한 등수에 한 경품 종류와 별도 당첨 인원을 설정한다.

PUT은 수정 가능한 설정 전체를 전달한다. 기존 경품은 ID를 유지하고 다른 이벤트의 경품 ID는 거절한다. 경품 목록 교체도 이벤트와 함께 원자적으로 처리한다. 경품 개수·등수 상한을 임의로 3등까지로 제한하지 않는다.

월말 소진용 이벤트는 별도 구분 필드 없이 `eventType=TICKET`, `weightingEnabled=true`, `maxTicketsPerUser=null` 조합으로 표현한다. `NO_TICKET`의 `maxTicketsPerUser=null`과는 `eventType`으로 구분한다. 서버는 이벤트 유형·가중치 여부·상한값 조합을 검증한다. 월말 소진용 이벤트의 지정 주체와 허용 조건은 담당자가 정한다.

`AdminEvent`: EventDetail 전체 + `imageKey:string?`, `createdBy:UUID`, `createdAt:instant`, `updatedAt:instant`, `suspendedFromStatus:SCHEDULED/OPEN/null`, `suspendedAt:instant?`, `canceledAt:instant?`, `prizeImages:{prizeId:UUID,imageKey:string?}[]`. 관리자에게 운영 상태를 보여준다.

수정은 시작 전 **SCHEDULED**에서만 허용하고, 현재 설정된 시작 시각보다 앞당길 수 없다. 중단·취소 상태는 시작 전이어도 수정할 수 없다. 진행·마감 후에는 제목·이미지도 수정할 수 없다. 수정 시 시작 알림 예약과 필요한 변경 알림을 연동한다.

삭제는 시작 전이고 응모 이력이 없을 때만 가능하다. 배너는 함께 삭제한다. 이미 생성된 알림 전체를 삭제한다는 정책은 최신 ADR에서 철회되었으므로 임의로 삭제하거나 반드시 보존한다고 확정하지 않는다. 삭제 이벤트의 기존 알림 처리 방식은 담당자 계약에 따른다.

## 상태 운영

`ReasonRequest`: 필수 `reason:string`, 공백만 입력 불가. `EventOperationResult`: `eventId:UUID`, `previousStatus:EventStatus`, `status:EventStatus`, `refundedTicketCount:long`, `processedAt:instant`.

| 작업 | 처리 계약 |
| --- | --- |
| 중단 | SCHEDULED/OPEN은 SUSPENDED, 반환 0. CLOSED에 중단 요청하면 즉시 CANCELED와 반환을 함께 처리 |
| 재개 | 마감 전 SUSPENDED만 허용. 현재 시각이 시작 전이면 SCHEDULED, 이후면 OPEN. 기존 응모·마감·차감 유지 |
| 취소 | 상태와 관계없이 허용. 당첨 발표 이후·대상 없는 종료 상태 포함. 취소와 대상 차감분 반환이 함께 완료되어야 200 |
| 중단 중 마감 | 내부 처리로 취소·반환. 이후 재개 불가 |

AE08을 반환 작업 접수만 한 202로 완료 처리하지 않는다. 실패는 취소/반환 모두 완료된 것으로 응답하지 않고 같은 원본 차감에 중복 반환하지 않는다. AE06~AE08 성공 뒤 반환하는 수량은 해당 운영 처리의 확정 결과로 제안한다.
