# 어뷰징·검토 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [어뷰징·검토 규칙](../02-domain/entry.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

근거: [응모·어뷰징 규칙][entry], [응모권 규칙][ticket]. 권한 A. 석종수(검토 결정)·전민규(게임 점수·통계 무효화와 알림)·윤태형(응모 제외에 따른 응모권 반환) 공동 경계다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AR01 | `GET /admin/abuse-cases` | `page,size,userId,eventId,sourceType,reviewStatus,from,to` | 200 `Page<AbuseCaseSummary>` | 공통 |
| AR02 | `GET /admin/abuse-cases/{caseId}` | 없음 | 200 `AbuseCaseDetail` | 공통 |
| AR03 | `POST /admin/abuse-cases/{caseId}/decisions` | `ReviewDecisionRequest` | 201 `ReviewDecisionResult` | 409 검토 상태/대상 충돌 |
| AR04 | 이번 버전 제외 | 회수 대상 조회 | — | — |
| AR05 | 이번 버전 제외 | 회수 결정·처리 | — | — |
| AR06 | `GET /admin/abuse-cases/{caseId}/decisions` | `page,size` | 200 `Page<ReviewDecisionHistory>` | 공통 |
| AR07 | `POST /admin/abuse-cases/{caseId}/game-play-invalidations` | 필수 `gamePlayId:UUID`, `reason:string` | 200 `GameInvalidationResult` | 409 부정 미확정·연결 불일치 |

AR01의 기본 정렬은 `detectedAt DESC, id DESC`다. 다른 목록은 [공통 목록 조회](common.md#목록-조회)의 기본 정렬을 따른다.

`AbuseCaseSummary`: `id:UUID`, `userId:UUID`, `eventId:UUID?`, `sourceType:ENTRY/ATTENDANCE/MISSION/GAME/REQUEST`, `sourceId:UUID?`, `detectionType:string`, `occurredAt:instant`, `detectedAt:instant`, `reviewStatus:PENDING/ALLOWED/CONFIRMED`.

`AbuseCaseDetail`: Summary 전체 + `detectionReason:string`, `evidence:object?`, `requestId:string?`, `reviewedBy:UUID?`, `reviewedAt:instant?`, `reviewReason:string?`. 이 근거는 관리자 전용이며 사용자에게 그대로 전달하지 않는다.

## 검토와 참여 제외

`ReviewDecisionRequest`: 필수 `decision:ALLOW/CONFIRM`, `reason:string`; 선택 `excludeFromEvent:boolean`(기본 false), `userNoticeReason:string`. 이벤트 제외 요청이면 `userNoticeReason`을 필수로 제안한다.

`ReviewDecisionResult`: `caseId:UUID`, `reviewStatus:ALLOWED/CONFIRMED`, `reviewedBy:UUID`, `reviewedAt:instant`, `participantId:UUID?`, `eligibilityStatus:ELIGIBLE/EXCLUDED/null`, `refundedTicketCount:long`.

탐지만으로 제외하지 않는다. ALLOW는 검토 결과이며 새 응모 접수를 생성하지 않는다. CONFIRM도 무조건 모든 이벤트에서 제외하는 동작이 아니다. `excludeFromEvent=true`는 해당 검토의 이벤트·응모자 연결이 존재할 때만 처리한다. 지급 단계 검토는 이벤트 연결 없이 가능하다.

제외 시 해당 이벤트의 원본 차감분을 반환한다. 반환분을 다시 회수하지 않는다. 정상 지급분을 대체 차감하지 않는다. 검토 결정 변경 시 기존 판단 이력을 보존해야 하므로 이력을 삭제하거나 덮어쓰지 않는다. 확정 판단 번복 후 제외·반환·점수의 복구 범위는 **검토 변경 후 복구 계약**이며 확정 전 복구 API를 약속하지 않는다.

`ReviewDecisionHistory`: `id:UUID`, `caseId:UUID`, `beforeStatus:PENDING/ALLOWED/CONFIRMED`, `afterStatus:ALLOWED/CONFIRMED`, `reason:string`, `actorId:UUID`, `createdAt:instant`.

## 응모권 회수 제외

AR04·AR05는 이번 버전에서 제외한다. 번호는 기존 초안 참조를 위해 유지하며 회수 API·DTO·처리와 회수 알림은 제공하지 않는다. 이벤트 제외 시 반환과 당첨 취소·재추첨은 기존 정책을 따른다.

## 부정 게임 점수 무효화

`GameInvalidationResult`: `gamePlayId:UUID`, `status:INVALID`, `statistics:MyGameStatistics`, `processedAt:instant`. 검토에서 부정이 확인되고 연결된 플레이일 때만 무효화하며, 최고점·누적점수·횟수의 정확한 재계산과 최초/변경 결과 이력을 보존한다. 응모권 회수는 수행하지 않는다. 이력 보존 및 이미 내려준 결과 재조회 의미는 **검토 변경 후 복구 계약·백엔드 이력 저장 설계** 확인 후 구현한다.

[entry]: ../02-domain/entry.md
[ticket]: ../02-domain/ticket.md
