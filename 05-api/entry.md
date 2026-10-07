# 응모 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [응모 규칙](../02-domain/entry.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

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

`participantCount`는 접수 완료된 응모자의 중복 제거 수다. 추가 응모 건수나 유효 추첨 후보 수와 다르다. 차감 합계는 접수 완료 건의 차감량이며 반환을 빼서 순사용량으로 바꾸지 않는다. `remainingTicketLimit=null`은 승인된 월말 소진용 이벤트의 수량 상한 없음이다. 잔액과 별개이며 `null`을 잔액 무제한으로 해석하지 않는다. 미사용 이벤트는 수량 관련 값이 0이고 1회 제한은 `canEnter`로 확인한다.

갱신은 E03 재조회 방식으로 제안한다. 성공 응모 직후 E03·E07·T01을 재조회한다. 폴링 간격은 구현 담당자 설정이며 SSE·WebSocket을 필수 계약으로 추가하지 않는다. 당첨 확률은 반환하지 않는다.

## 응모 요청·응답

`EntryRequest`의 유일한 필드는 필수 `ticketCount:int`다.

| 이벤트 | 허용 요청 | 서버 검증 |
| --- | --- | --- |
| 미사용 | `0` | 사용자당 1회, 경품 선택 없음 |
| 사용·가중치 미적용 | `1` | 사용자당 1회 |
| 사용·일반 가중치 | `1..5` | 기존 누적 사용량 + 요청량 ≤ 5 |
| 사용·월말 소진용 | 양의 정수 | `TICKET`·가중치 적용·`maxTicketsPerUser=null` 조합과 유효 잔액; 운영 허용 조건은 월말 소진용 이벤트 운영 조건 |

```json
{ "ticketCount": 2 }
```

`EntryReceipt`: `id:UUID`, `eventId:UUID`, `eventTitle:string`, `requestedTicketCount:int`, `deductedTicketCount:int`, `status:ACCEPTED/REJECTED`, `requestedAt:instant`, `acceptedAt:instant?`, `rejectionCode:string?`, `rejectionReason:string?`.

신규 접수는 멤버십·역할·기간·상태·보유량·누적 상한을 검증하고 응모 기록과 차감을 함께 확정한다. `acceptedAt < endsAt`이어야 하며 사용 응모권의 만료 전에도 확정되어야 한다. 도착 시각만으로 마감 전 응모를 인정하지 않는다. 거절은 해당 4xx 봉투로 반환하고 저장된 업무 거절 이력은 E05/E06에서 조회한다. 형식 오류·인증 실패의 조회 이력은 제공하지 않는다.

자격 사전 조회 E07은 응모 보장이 아니다. E04에서 다시 검증한다. 일반 추가 응모와 동일 요청 재전송을 구분하며 사용자가 `prizeId`, 가중치, 접수 확정 시각을 지정할 수 없다.

## 관리자 응모·응모자 조회

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AO06 | `GET /admin/entries` | `page,size,userId,eventId,status,from,to` | 200 `Page<AdminEntry>` | 공통 |
| AO07 | `GET /admin/events/{eventId}/participants` | `page,size,eligibilityStatus` | 200 `Page<AdminParticipant>` | 공통 |

| DTO | 필드 |
| --- | --- |
| `AdminEntry` | EntryReceipt 전체 + `userId:UUID`, `participantId:UUID?` |
| `AdminParticipant` | `id:UUID`, `eventId:UUID`, `userId:UUID`, `usedTicketCount:long`, `eligibilityStatus:ELIGIBLE/EXCLUDED`, `exclusionReasonCode:ABUSE/INELIGIBLE/null`, `exclusionReason:string?`, `excludedAt:instant?`, `createdAt:instant` |
