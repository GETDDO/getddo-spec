# 응모권 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [응모권 규칙](../02-domain/ticket.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

근거: [응모권 규칙][ticket]. 권한 U, 주 담당 윤태형.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| T01 | `GET /tickets/me` | 없음 | 200 `MyTickets` | 공통 |
| T02 | `GET /tickets/histories/me` | `cursor,size,operationType,from,to` | 200 `Cursor<TicketHistory>` | 공통, 400 조회 조건 오류 |

| DTO | 필드 |
| --- | --- |
| `MyTickets` | `availableCount:long`, `countByGrade:{BRONZE,SILVER,GOLD:long}`, `holdings:TicketHolding[]`, `serverTime:instant` |
| `TicketHolding` | `grade:BRONZE/SILVER/GOLD`, `expiresAt:instant`, `count:long` |
| `TicketHistory` | `id:UUID`, `ticketId:UUID`, `operationType:GRANT/USE/REFUND/EXPIRE/CORRECTION`, `grade:BRONZE/SILVER/GOLD`, `status:AVAILABLE/RETURNED/SPENT/EXPIRED`, `expiresAt:instant`, `reason:string`, `createdAt:instant`, `eventId:UUID?`, `eventEntryId:UUID?`, `missionId:UUID?`, `gameId:UUID?`, `attendanceDate:date?`, `originalUseHistoryId:UUID?`, `correctedHistoryId:UUID?` |

응모권은 한 장이 한 행이다. `TicketHistory`는 응모권 한 장의 처리 한 번이며 수량 필드는 없다. 한 응모에 응모권 여러 장을 쓰면 장마다 `USE` 이력이 생기고 같은 `eventEntryId`를 가진다. `status`·`expiresAt`은 그 처리가 끝난 직후의 값이다. `originalUseHistoryId`는 `REFUND`가 되돌리는 원본 `USE` 이력, `correctedHistoryId`는 `CORRECTION`이 정정하는 과거 이력이다. `CORRECTION`은 이력 유형을 표현하며 임의 수량 수정 API를 제공한다는 뜻이 아니다.

T01의 `availableCount`는 `AVAILABLE`·`RETURNED` 상태이고 `expiresAt`이 서버 시각보다 뒤인 응모권만 센다. 등급과 관계없이 한 장은 1이다. `holdings`는 등급과 만료 시각이 같은 응모권의 묶음이며 만료가 임박한 순서다. `countByGrade`는 장수가 0인 등급도 포함하고, 합계는 `availableCount`와 같다. 지급분은 지급 KST 월의 다음 달 1일 자정에, 반환분은 반환 KST 월의 다다음 달 1일 자정에 만료한다. 예를 들어 9월 지급분 만료는 `2026-09-30T15:00:00Z`, 9월 반환분 만료는 `2026-10-31T15:00:00Z`다. 자동 만료 처리가 지연되어도 이미 만료된 응모권을 사용 가능으로 표시하지 않는다.

T02는 `createdAt DESC, id DESC` 순서의 커서 페이지다. `from`·`to`는 시간대(`Z` 또는 오프셋)를 포함한 시각이어야 하고 기간은 `[from, to)`다. `totalElements`는 커서와 관계없이 조회 조건에 맞는 전체 이력 수다. `size` 범위와 커서 규칙은 [공통 API 계약](common.md)을 따른다.

응모권 수량 변경·보상 청구·사용자 반환 요청 API는 없다. 보상과 반환은 원본 업무 처리에서 수행한다. 차감 우선순위를 새 정책으로 추가하지 않는다.

## 관리자 응모권 조회

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AO03 | `GET /admin/ticket-grants` | `page,size,userId,keyword,missionId,from,to` | 200 `Page<TicketGrant>` | 공통 |
| AO04 | `GET /admin/users/{userId}/tickets/consistency` | 없음 | 200 `TicketConsistency` | 공통 |
| AO05 | `GET /admin/ticket-histories` | `page,size,userId,eventId,missionId,operationType,from,to` | 200 `Page<AdminTicketHistory>` | 공통 |
| AO08 | `GET /admin/ticket-refunds` | `page,size,eventId,userId,status,from,to` | 200 `Page<TicketRefund>` | 공통 |

| DTO | 필드 |
| --- | --- |
| `TicketGrant` | `historyId:UUID`, `ticketId:UUID`, `userId:UUID`, `userName:string`, `grade:BRONZE/SILVER/GOLD`, `sourceType:ATTENDANCE/MISSION/GAME`, `sourceId:UUID`, `missionId:UUID?`, `grantedAt:instant`, `expiresAt:instant` |
| `AdminTicketHistory` | TicketHistory 전체 + `userId:UUID` |
| `TicketConsistency` | `userId:UUID`, `availableCount:long`, `isConsistent:boolean`, `inconsistencies:{ticketId:UUID,ticketVersion:long,latestHistoryVersion:long}[]`, `checkedAt:instant` |
| `TicketRefund` | `id:UUID`, `spendHistoryId:UUID`, `eventId:UUID`, `userId:UUID`, `reasonType:EVENT_CANCELED/PARTICIPANT_EXCLUDED`, `status:PENDING/PROCESSING/COMPLETED/FAILED`, `targetExpiresAt:instant?`, `attemptCount:int`, `createdAt:instant`, `completedAt:instant?` |

`keyword`는 사용자 이름 검색으로 제안하며 정확한 사용자 ID는 `userId`로 필터링한다. AO03의 `sourceId`는 출석·미션 제출·게임 플레이 원본 ID로 산출한다. 응모권 지급 이력과 현재 상태의 정합성 확인을 함께 제공한다. `isConsistent`는 같은 기준 시점에 응모권의 현재 버전·상태와 마지막 이력의 버전·상태를 비교한 결과여야 하며 단순히 항상 true로 반환하지 않는다. 불일치한 응모권만 `inconsistencies`에 담고, 없으면 `isConsistent`가 true다. `TicketGrant`는 응모권 한 장의 지급 이력이므로 한 번의 보상으로 여러 장을 지급하면 장수만큼 행이 나온다.

AO08은 처리 이력 조회다. 이를 근거로 이벤트 취소 완료 후 반환 배치를 별도로 기다리는 정상 흐름을 만들지 않는다. 사용자 정보 수정, 임의 지급·회수·수량 조정 경로는 없다.

[ticket]: ../02-domain/ticket.md

## 등급 응모권 정책 반영 후속 — 2026-10-06

게임·미션 보상은 [등급 응모권 규칙](../02-domain/ticket.md#게임미션-보상의-등급-추첨)을 따른다. T01·T02는 등급별 보유량(`countByGrade`, `holdings`)과 이력의 등급(`grade`)을 포함하도록 갱신했다. 지급 결과와 가중치 표현은 아직 확정하지 않았다. 게임·미션 보상의 실제 지급 장수는 1이며 가중치 값을 `ticketCount`나 `quantity`에 넣지 않는다. 미션의 보상 수량을 임의 설정하는 종전 요청 초안도 새 정책에 맞춰 재검토한다. 응모 시 차감은 사용자가 등급별 장수를 고르며(규칙은 [응모권 규칙](../02-domain/ticket.md#응모권-경제운영)), 요청·응답 필드는 [응모 API](entry.md#응모-요청응답)를 따른다. 미결정 항목은 [등급 응모권 후속 사항](../00-requirements/pending-decisions.md#등급-응모권)을 따른다.

이번 버전의 신규 처리 유형에서 REVOKE는 제외한다. 기존 저장 이력의 조회 호환 처리는 담당자가 별도 확인하며, 이번 명세 변경으로 기존 이력을 삭제하지 않는다.
