# 어뷰징·검토 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [어뷰징·검토 규칙](../02-domain/entry.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

근거: [응모·어뷰징 규칙][entry], [부정 획득 회수 규칙][ticket]. 권한 A. 석종수(검토)·윤태형(점수·응모권)·전민규(운영 API·알림) 공동 경계다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AR01 | `GET /admin/abuse-cases` | `page,size,userId,eventId,sourceType,reviewStatus,from,to` | 200 `Page<AbuseCaseSummary>` | 공통 |
| AR02 | `GET /admin/abuse-cases/{caseId}` | 없음 | 200 `AbuseCaseDetail` | 공통 |
| AR03 | `POST /admin/abuse-cases/{caseId}/decisions` | `ReviewDecisionRequest` | 201 `ReviewDecisionResult` | 409 검토 상태/대상 충돌 |
| AR04 | `GET /admin/abuse-cases/{caseId}/recovery-targets` | `page,size` | 200 `Page<RecoveryTarget>` | 공통 |
| AR05 | `POST /admin/abuse-cases/{caseId}/recovery-targets` | `RecoveryDecisionRequest` | 201 `RecoveryDecisionResult`; 기존 결정 재사용 200 | 409 부정 미확정·원본/수량 충돌 |
| AR06 | `GET /admin/abuse-cases/{caseId}/decisions` | `page,size` | 200 `Page<ReviewDecisionHistory>` | 공통 |
| AR07 | `POST /admin/abuse-cases/{caseId}/game-play-invalidations` | 필수 `gamePlayId:UUID`, `reason:string` | 200 `GameInvalidationResult` | 409 부정 미확정·연결 불일치 |

AR01의 기본 정렬은 `detectedAt DESC, id DESC`다. 다른 목록은 [공통 목록 조회](common.md#목록-조회)의 기본 정렬을 따른다.

`AbuseCaseSummary`: `id:UUID`, `userId:UUID`, `eventId:UUID?`, `sourceType:ENTRY/ATTENDANCE/MISSION/GAME/REQUEST`, `sourceId:UUID?`, `detectionType:string`, `occurredAt:instant`, `detectedAt:instant`, `reviewStatus:PENDING/ALLOWED/CONFIRMED`.

`AbuseCaseDetail`: Summary 전체 + `detectionReason:string`, `evidence:object?`, `requestId:string?`, `reviewedBy:UUID?`, `reviewedAt:instant?`, `reviewReason:string?`. 이 근거는 관리자 전용이며 사용자에게 그대로 전달하지 않는다.

## 검토와 참여 제외

`ReviewDecisionRequest`: 필수 `decision:ALLOW/CONFIRM`, `reason:string`; 선택 `excludeFromEvent:boolean`(기본 false), `userNoticeReason:string`. 이벤트 제외 요청이면 `userNoticeReason`을 필수로 제안한다.

`ReviewDecisionResult`: `caseId:UUID`, `reviewStatus:ALLOWED/CONFIRMED`, `reviewedBy:UUID`, `reviewedAt:instant`, `participantId:UUID?`, `eligibilityStatus:ELIGIBLE/EXCLUDED/null`, `refundedTicketCount:long`.

탐지만으로 제외·회수하지 않는다. ALLOW는 검토 결과이며 새 응모 접수를 생성하지 않는다. CONFIRM도 무조건 모든 이벤트에서 제외하는 동작이 아니다. `excludeFromEvent=true`는 해당 검토의 이벤트·응모자 연결이 존재할 때만 처리한다. 지급 단계 검토는 이벤트 연결 없이 가능하다.

제외 시 해당 이벤트의 원본 차감분을 반환한다. 반환 대상에 부정 지급분이 있으면 이미 확정된 회수 결정과 연결하여 반환·회수를 함께 처리한다. 정상 지급분을 대체 차감하지 않는다. 검토 결정 변경 시 기존 판단 이력을 보존해야 하므로 이력을 삭제하거나 덮어쓰지 않는다. 확정 판단 번복 후 제외·반환·점수의 복구 범위는 **검토 변경 후 복구 계약**이며 확정 전 복구 API를 약속하지 않는다.

`ReviewDecisionHistory`: `id:UUID`, `caseId:UUID`, `beforeStatus:PENDING/ALLOWED/CONFIRMED`, `afterStatus:ALLOWED/CONFIRMED`, `reason:string`, `actorId:UUID`, `createdAt:instant`.

## 부정 획득 원본 지급분 회수

`RecoveryDecisionRequest`: 필수 `reason:string`, `targets:RecoveryTargetWrite[]`.

`RecoveryTargetWrite`: 필수 `originalGrantId:UUID`, `targetQuantity:long(>0)`; 선택 `supersedesTargetId:UUID`(이전 결정을 대체할 때).

`RecoveryTarget`: `id:UUID`, `caseId:UUID`, `originalGrantId:UUID`, `targetQuantity:long`, `recoveredQuantity:long`, `unrecoveredQuantity:long`, `status:ACTIVE/SUPERSEDED`, `supersedesTargetId:UUID?`, `reason:string`, `decidedBy:UUID`, `createdAt:instant`.

`RecoveryDecisionResult`: `caseId:UUID`, `targets:RecoveryTarget[]`, `recoveredTicketCount:long`. `unrecoveredQuantity`는 아직 실제 회수로 연결되지 않은 대상량이며 이미 만료된 수량까지 모두 현재 회수 가능하다는 뜻이 아니다. 수치는 원본·배분·REVOKE 이력에서 산출한다.

한 요청에서 같은 사용자의 여러 지급 건을 지정한다. 다른 사용자의 원본·GRANT가 아닌 원본·원본 수량을 넘는 결정을 거절한다. 같은 지급 건의 다른 검토 결정을 단순 합산하지 않는다. 이미 회수·만료·차감된 수량과 배분을 확인하고 당장 회수할 수 없어도 결정은 보존한다. 잔액을 음수로 만들거나 다른 정상 지급분으로 채우지 않는다. 실제 회수 원장에는 회수 대상 결정 ID를 연결한다.

AR05의 의미는 자유로운 수동 회수가 아니라 **부정이 확정된 검토에 대한 원본 지정 회수 결정 및 가능한 수량 처리**다. 당첨자라면 당첨 취소·재추첨 흐름 AD05도 이어져야 한다.

## 부정 게임 점수 무효화

`GameInvalidationResult`: `gamePlayId:UUID`, `status:INVALID`, `statistics:MyGameStatistics`, `processedAt:instant`. 검토에서 부정이 확인되고 연결된 플레이일 때만 무효화하며, 최고점·누적점수·횟수의 정확한 재계산과 최초/변경 결과 이력을 보존한다. 부정 보상 회수 결정은 AR05와 연결한다. 이력 보존 및 이미 내려준 결과 재조회 의미는 **검토 변경 후 복구 계약·백엔드 이력 저장 설계** 확인 후 구현한다.

[entry]: ../02-domain/entry.md
[ticket]: ../02-domain/ticket.md
