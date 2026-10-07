# 응모권 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [응모권 규칙](../02-domain/ticket.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

근거: [응모권 규칙][ticket]. 권한 U, 주 담당 윤태형.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| T01 | `GET /tickets/wallets/me` | 없음 | 200 `MyWallets` | 공통 |
| T02 | `GET /tickets/ledger/me` | `cursor,size,transactionType,from,to` | 200 `Cursor<TicketTransaction>` | 공통 |

| DTO | 필드 |
| --- | --- |
| `MyWallets` | `availableBalance:long`, `wallets:TicketWallet[]`, `serverTime:instant` |
| `TicketWallet` | `id:UUID`, `expiryMonth:month`, `validFrom:instant`, `expiresAt:instant`, `balance:long`, `status:ACTIVE/EXPIRED` |
| `TicketTransaction` | `id:UUID`, `walletId:UUID`, `transactionType:GRANT/SPEND/REFUND/EXPIRE/CORRECTION`, `quantity:long`, `balanceAfter:long`, `reason:string`, `createdAt:instant`, `expiresAt:instant?`, `eventId:UUID?`, `eventEntryId:UUID?`, `missionId:UUID?`, `gameId:UUID?`, `attendanceDate:date?`, `relatedLedgerId:UUID?`, `refundOfId:UUID?` |

`quantity`는 지급·반환이 양수, 차감·만료가 음수다. `balanceAfter`는 해당 지갑의 처리 직후 잔액이며 전체 지갑 합계가 아니다. `CORRECTION`은 이력 유형을 표현하며 임의 잔액 수정 API를 제공한다는 뜻이 아니다.

`availableBalance`에는 서버 시각에 사용 가능한 잔여량만 포함한다. 지급분은 지급 KST 월의 다음 달 1일 자정에, 반환분은 반환 KST 월의 다다음 달 1일 자정에 만료한다. 예를 들어 9월 지급분 만료는 `2026-09-30T15:00:00Z`, 9월 반환분 만료는 `2026-10-31T15:00:00Z`다. 자동 만료 처리가 지연되어도 이미 만료된 잔액을 사용 가능으로 표시하지 않는다.

지갑 잔액 변경·보상 청구·사용자 반환 요청 API는 없다. 보상과 반환은 원본 업무 처리에서 수행한다. 차감 우선순위를 새 정책으로 추가하지 않는다.

## 관리자 응모권 조회

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AO03 | `GET /admin/ticket-grants` | `page,size,userId,keyword,missionId,from,to` | 200 `Page<TicketGrant>` | 공통 |
| AO04 | `GET /admin/users/{userId}/ticket-wallets` | 없음 | 200 `WalletReconciliation` | 공통 |
| AO05 | `GET /admin/ticket-ledger` | `page,size,userId,eventId,missionId,transactionType,from,to` | 200 `Page<AdminTicketTransaction>` | 공통 |
| AO08 | `GET /admin/ticket-refunds` | `page,size,eventId,userId,status,from,to` | 200 `Page<TicketRefund>` | 공통 |

| DTO | 필드 |
| --- | --- |
| `TicketGrant` | `ledgerId:UUID`, `userId:UUID`, `userName:string`, `quantity:long`, `sourceType:ATTENDANCE/MISSION/GAME`, `sourceId:UUID`, `missionId:UUID?`, `grantedAt:instant`, `expiresAt:instant` |
| `AdminTicketTransaction` | TicketTransaction 전체 + `userId:UUID`, `actorId:UUID?`, `recoveryTargetId:UUID?` |
| `WalletReconciliation` | `userId:UUID`, `availableBalance:long`, `isConsistent:boolean`, `wallets:{walletId:UUID,balance:long,ledgerBalance:long,isConsistent:boolean}[]`, `checkedAt:instant` |
| `TicketRefund` | `id:UUID`, `spendLedgerId:UUID`, `eventId:UUID`, `userId:UUID`, `reasonType:EVENT_CANCELED/PARTICIPANT_EXCLUDED`, `status:PENDING/PROCESSING/COMPLETED/FAILED`, `targetExpiresAt:instant?`, `attemptCount:int`, `createdAt:instant`, `completedAt:instant?` |

`keyword`는 사용자 이름 검색으로 제안하며 정확한 사용자 ID는 `userId`로 필터링한다. AO03의 `sourceId`는 출석·미션 제출·게임 플레이 원본 ID로 산출한다. 응모권 지급량·이력과 잔액 정합성 확인을 함께 제공한다. `isConsistent`는 같은 기준 시점의 원장 합계와 지갑 잔액 비교 결과여야 하며 단순히 항상 true로 반환하지 않는다.

AO08은 처리 이력 조회다. 이를 근거로 이벤트 취소 완료 후 반환 배치를 별도로 기다리는 정상 흐름을 만들지 않는다. 사용자 정보 수정, 임의 지급·회수·잔액 조정 경로는 없다.

[ticket]: ../02-domain/ticket.md

## 등급 응모권 정책 반영 후속 — 2026-10-06

게임·미션 보상은 [등급 응모권 규칙](../02-domain/ticket.md#게임미션-보상의-등급-추첨)을 따른다. 기존 DTO 표는 등급별 보유량·지급 결과·가중치를 표현하지 못하는 부분이 있어 구현 전 담당자와 갱신해야 한다. 게임·미션 보상의 실제 지급 장수는 1이며 가중치 값을 `ticketCount`나 `quantity`에 넣지 않는다. 미션의 보상 수량을 임의 설정하는 종전 요청 초안도 새 정책에 맞춰 재검토한다. 이번 정책 문서 수정만으로 새로운 공개 필드·차감 등급 선택 계약을 확정하지 않는다. 미결정 항목은 [등급 응모권 후속 사항](../00-requirements/pending-decisions.md#등급-응모권)을 따른다.

이번 버전의 신규 처리 유형에서 REVOKE는 제외한다. 기존 저장 이력의 조회 호환 처리는 담당자가 별도 확인하며, 이번 명세 변경으로 기존 이력을 삭제하지 않는다.
