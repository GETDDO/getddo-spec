# 응모 API

- 상태: 사용자 API(E03~E07) 구현 반영, 관리자 조회(AO06·AO07) 구현(백엔드 GD-218). 정책 변경 검토 완료 (2026-10-11)
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [응모 규칙](../02-domain/entry.md)

사용자 API는 구현 결과를 반영했다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 사용자 응모·현황

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| E03 | U | `GET /events/{eventId}/statistics` | 없음 | 200 `EntryStatistics` | 공통 |
| E04 | U | `POST /events/{eventId}/entries` | 멱등 헤더, `EntryRequest` | 201 `EntryReceipt`; 접수 성공 건의 동일 요청 200 | 403 자격/관리자, 409 기간·취소·상한·잔액·중복, 429; 거절 건의 동일 요청은 최초 4xx·오류 코드 |
| E05 | U | `GET /events/{eventId}/entries/me` | `page,size` | 200 `Page<EntryReceipt>` | 공통 |
| E06 | U | `GET /users/me/entries` | `page,size,eventId,status,from,to` | 200 `Page<EntryReceipt>` | 공통 |
| E07 | U | `GET /events/{eventId}/eligibility` | 없음 | 200 `EntryEligibility` | 공통 |

| DTO | 필드 |
| --- | --- |
| `EntryStatistics` | `eventId:UUID`, `participantCount:long`, `totalSpentTicketCount:long`, `mySpentTicketCount:long`, `serverTime:instant` |
| `EntryEligibility` | `eventId:UUID`, `canEnter:boolean`, `reasons:string[]`, `usedTicketCount:long`, `remainingTicketLimit:long?`, `availableTicketBalance:long`, `serverTime:instant` |

`participantCount`는 접수 완료된 응모자의 중복 제거 수다. 추가 응모 건수나 유효 추첨 후보 수와 다르다. 차감 합계는 접수 완료 건의 차감량이며 반환을 빼서 순사용량으로 바꾸지 않는다. `remainingTicketLimit=null`은 승인된 월말 소진용 이벤트의 수량 상한 없음이다. 잔액과 별개이며 `null`을 잔액 무제한으로 해석하지 않는다. 미사용 이벤트는 수량 관련 값이 0이고 1회 제한은 `canEnter`로 확인한다. `availableTicketBalance`는 이 이벤트에서 실제로 쓸 수 있는 응모권 장수다. 가중치 미적용 이벤트는 브론즈만 쓸 수 있으므로 사용 가능한 브론즈 장수이고, 브론즈가 없으면 `canEnter=false`와 그 사유를 `reasons`에 담는다. 등급별 보유 장수는 T01의 `countByGrade`를 쓴다.

갱신은 E03 재조회 방식으로 제안한다. 성공 응모 직후 E03·E07·T01을 재조회한다. 폴링 간격은 구현 담당자 설정이며 SSE·WebSocket을 필수 계약으로 추가하지 않는다. 당첨 확률은 반환하지 않는다.

## 응모 요청·응답

`EntryRequest`의 유일한 필드는 `tickets`다. 사용자가 쓸 응모권의 등급별 장수를 담은 맵이며 키는 `BRONZE`·`SILVER`·`GOLD`·`PLATINUM`, 값은 0 이상의 정수다. 없는 등급은 0으로 본다. 요청 합계는 서버가 계산하고 별도의 합계 필드는 받지 않는다. 등급별 장수는 보유 조회 T01의 `countByGrade`에서 고르며 그 등급의 현재 사용 가능한 장수를 넘을 수 없다. 같은 등급 안에서는 만료가 이른 응모권부터 차감한다.

| 이벤트 | 허용 요청 | 서버 검증 |
| --- | --- | --- |
| 미사용 | `tickets` 생략 또는 `{}` (합계 `0`) | 사용자당 1회, 경품 선택 없음 |
| 사용·가중치 미적용 | `{"BRONZE": 1}`만 허용 | 사용자당 1회. 브론즈 외 등급이 하나라도 있으면 거절 |
| 사용·일반 가중치 | 합계 `1..5` | 기존 누적 사용량 + 요청 합계 ≤ 5 |
| 사용·월말 소진용 | 합계가 양의 정수 | `TICKET`·가중치 적용·`maxTicketsPerUser=null` 조합과 등급별 유효 보유량; 운영 허용 조건은 월말 소진용 이벤트 운영 조건 |

```json
{ "tickets": { "GOLD": 2, "SILVER": 5 } }
```

가중치 미적용 이벤트는 등급이 추첨에 쓰이지 않으므로 비싼 등급을 실수로 쓰지 않도록 브론즈 응모권만 쓸 수 있다. 이 이벤트에 응모하려면 사용 가능한 브론즈가 1장 이상이어야 하며, 브론즈가 없는 사용자는 다른 등급이 있어도 응모할 수 없다. 이 경우는 일반 보유량 부족과 구분되는 안내(브론즈 응모권이 필요하다는 메시지)를 반환한다.

`EntryReceipt`: `id:UUID`, `eventId:UUID`, `eventTitle:string`, `requestedTicketCount:int`, `requestedTicketsByGrade:{BRONZE,SILVER,GOLD,PLATINUM:int}`, `deductedTicketCount:int`, `deductedTicketsByGrade:{BRONZE,SILVER,GOLD,PLATINUM:int}`, `status:ACCEPTED/REJECTED`, `requestedAt:instant`, `acceptedAt:instant?`, `rejectionCode:string?`, `rejectionReason:string?`.

`requestedTicketCount`와 `deductedTicketCount`는 등급별 장수의 합계이며, 등급별 맵은 장수가 0인 등급도 포함한다. 거절 건의 `deductedTicketsByGrade`는 모두 0이다. 같은 멱등 키로 등급별 장수가 다른 요청이 오면 409로 거절한다.

신규 접수는 멤버십·역할·기간·상태·보유량·누적 상한을 검증하고 응모 기록과 차감을 함께 확정한다. `acceptedAt < endsAt`이어야 하며 사용 응모권의 만료 전에도 확정되어야 한다. 도착 시각만으로 마감 전 응모를 인정하지 않는다. 거절은 해당 4xx 봉투로 반환하고 저장된 업무 거절 이력은 E05/E06에서 조회한다. 형식 오류·인증 실패의 조회 이력은 제공하지 않는다.

**현행 구현 범위(백엔드 GD-134, GD-145, GD-200)**: 응모 ID는 `Idempotency-Key` 헤더의 UUID이며 응모 요청 결과(접수 또는 거절)의 `id`와 같다. 접수된 응모는 `status=ACCEPTED`이고 `requestedAt`은 `acceptedAt`과 같으며 `rejectionCode`·`rejectionReason`은 `null`이다. 업무 거절 중 `ENTRY-002·003·005·006·007·008`과 `TICKET-006`은 응모 처리와 별도로 이력에 저장하고, E05/E06에서 `status=REJECTED`, 차감 장수 0, `acceptedAt=null`, `rejectionCode`·`rejectionReason`으로 조회한다. 거절 건의 `requestedAt`은 거절을 기록한 시각이고 `requestedTicketsByGrade`는 사용자가 요청한 값이다. 요청 형식 오류(`ENTRY-001`), 없는 이벤트(`ENTRY-004`), 같은 키의 다른 요청(`ENTRY-009`), 인증 실패와 요청 속도 제한(429)은 저장하지 않는다. 같은 `Idempotency-Key`로 거절된 요청을 같은 내용으로 다시 보내면 마감·보유량 등 상황이 바뀌어도 최초와 같은 4xx와 오류 코드를 돌려주고, 내용이 다르면 `ENTRY-009`다. E05는 `GET /events/{eventId}/entries/me`(없는 이벤트는 404 `ENTRY-004`), E06은 `GET /users/me/entries`이며 `from`·`to`는 시간대를 포함한 ISO-8601 시각으로 `requestedAt` 기준 `[from, to)`를 거른다. 정렬은 `requestedAt` 내림차순, 같으면 `id` 내림차순이다. 알 수 없는 `status`나 시각 형식은 400 `COMMON-005`, `from`이 `to`보다 늦거나 같으면 400 `COMMON-002`다. E03은 `participantCount`를 접수 완료된 응모자의 중복 제거 수로, `totalSpentTicketCount`를 그 응모자들의 누적 차감 수량 합계(반환을 빼지 않음)로, `mySpentTicketCount`를 요청 사용자의 누적 차감 수량(응모하지 않았으면 0, 관리자도 0)으로 계산하며 거절 이력은 포함하지 않는다. 없거나 삭제된 이벤트는 404 `ENTRY-004`다. 마감 판정은 응모 트랜잭션을 커밋하기 직전에 마지막으로 하며, 그 시각이 마감 전이어야 접수가 확정된다. 이 확인과 DB가 커밋을 확정하는 순간 사이에는 상한이 보장되지 않는 간격이 남는다. 응모 요청 속도 제한(429)은 공통 오류 규칙(`common.md`)을 따른다(GD-146). 자격 조회 `reasons`는 위 응모 오류의 상수 이름(`ADMIN_ENTRY_FORBIDDEN`, `ENTRY_MEMBERSHIP_NOT_MET`, `EVENT_NOT_OPEN`, `ALREADY_ENTERED`, `TICKET_LIMIT_EXCEEDED`, `BRONZE_REQUIRED`)과 보유 부족 `INSUFFICIENT_TICKETS`다.

자격 사전 조회 E07은 응모 보장이 아니다. E04에서 다시 검증한다. 일반 추가 응모와 동일 요청 재전송을 구분하며 사용자가 `prizeId`, 가중치, 접수 확정 시각을 지정할 수 없다.

## 이벤트 목록의 내 응모 여부 (이벤트 계약에서 처리)

프론트 요청 R7에 따라 이벤트 목록(E01)의 `EventSummary`에 `myEntered:boolean`을 두는 일은 이벤트 도메인이 맡는다(백엔드 GD-226). 응모 도메인은 별도 기능이나 HTTP 필드를 제공하지 않는다. 판정 기준은 응모 계약과 같다: 요청 사용자가 해당 이벤트에 접수 완료된 응모를 가졌을 때만 `true`이고, 거절된 요청만 있거나 관리자·다른 사용자의 응모는 `false`다. 한 번 접수된 응모는 이벤트가 취소되어 응모권이 반환돼도 기록으로 남으므로 `true`가 유지된다. 이 값은 추가 응모 가능 여부가 아니며 자격은 E07이 판단한다.

## 관리자 응모·응모자 조회

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AO06 | `GET /admin/entries` | `page,size,userId,eventId,status,from,to` | 200 `Page<AdminEntry>` | 공통 |
| AO07 | `GET /admin/events/{eventId}/participants` | `page,size` | 200 `Page<AdminParticipant>` | 공통 |

| DTO | 필드 |
| --- | --- |
| `AdminEntry` | EntryReceipt 전체 + `userId:UUID`, `participantId:UUID?` |
| `AdminParticipant` | `id:UUID`, `eventId:UUID`, `userId:UUID`, `usedTicketCount:long`, `createdAt:instant` |

**현행 구현 범위(백엔드 GD-218)**: 두 경로는 활성 관리자만 호출할 수 있고 아니면 403 `ENTRY-010`(`ADMIN_REQUIRED`)이다. AO06은 접수와 거절 이력을 모두 `createdAt`(`requestedAt`) 내림차순, 같으면 `id` 내림차순으로 돌려주며 `from`·`to`는 E06과 같이 `[from, to)`다. 거절 건의 `participantId`는 `null`이다. AO07은 최초 응모 시각 내림차순, 같으면 `id` 내림차순이고, 없는 이벤트는 404 `ENTRY-004`이며 삭제된 이벤트의 응모자는 조회할 수 있다. `size`는 1~100이다.

## 2026-10-11 추첨 제외 폐지

관리자 참여 허용·추첨 제외 결정은 제공하지 않는다. AO07의 신규 필터·DTO에 제외 상태·사유·시각을 포함하지 않는다. 정상 접수된 유효 응모를 차단 기록으로 추첨에서 제외하지 않는다. 기존 데이터에 제외 정보가 있는 경우의 조회 호환성은 구현 전 담당자와 확인하며 저장 이력을 삭제하지 않는다.

## 2026-10-17 플래티넘 등급 계약 반영

위 등급 목록에는 플래티넘(`PLATINUM`) 추가 정책을 반영했다. 플래티넘 지원의 서비스 적용 시점과 구현·프론트엔드 연동 검증은 후속 사항이며, 기존의 구현 반영 표기를 플래티넘 지원 완료로 해석하지 않는다. 확률·가중치는 [응모권 규칙](../02-domain/ticket.md#게임미션-보상의-등급-추첨), 전환 후속은 [등급 응모권 후속 사항](../00-requirements/pending-decisions.md#등급-응모권)을 따른다.
