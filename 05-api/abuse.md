# 어뷰징 차단 내역 API 초안

- 상태: 이번 정책 변경 검토 완료 (2026-10-11) / 상세 API 계약은 담당자 검토 대기
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [응모·어뷰징 규칙](../02-domain/entry.md)
- 갱신일: 2026-10-11 — 지급 전 검증·차단 기록 중심 정책 반영

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다. 차단 적용 대상과 판정 기준은 [후속 확인](../00-requirements/pending-decisions.md#지급-전-차단-전환의-후속-확인)이며 이번 초안으로 확정하지 않는다.

권한 A. 석종수는 차단 기록·관리자 조회, 전민규는 게임 결과 검증·점수·보상 자격, 윤태형은 실제 응모권 지급을 담당한다. 관련 요청·검증 근거 연결은 공동 경계이므로 구현 전 계약을 확인한다. 알림 계약은 이번 변경 범위에서 제외한다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AR01 | `GET /admin/abuse-cases` | `page,size,userId,eventId,sourceType,from,to` 제안 | 200 `Page<AbuseCaseSummary>` | 공통 |
| AR02 | `GET /admin/abuse-cases/{caseId}` | 없음 | 200 `AbuseCaseDetail` | 공통 |
| AR03 | 기존 참여 허용·추첨 제외 결정 초안은 구현 범위에서 제외 | 사후 부정 확인 계약은 후속 검토 | — | — |
| AR04 | 이번 버전 제외 | 회수 대상 조회 | — | — |
| AR05 | 이번 버전 제외 | 회수 결정·처리 | — | — |
| AR06 | 기존 검토 결정 이력 조회 초안은 후속 재검토 | 사후 처리 계약·과거 이력 호환성 확인 | — | — |
| AR07 | `POST /admin/abuse-cases/{caseId}/game-play-invalidations` 기존 초안 유지 | 필수 `gamePlayId:UUID`, `reason:string` | 200 `GameInvalidationResult` | 409 부정 미확정·연결 불일치 |

## 차단 내역 조회

관리자는 차단 사유·발생 시각·관련 요청·검증 근거를 조회한다. 차단 기록을 조회하거나 저장하는 동작이 참여 허용, 추첨 제외, 응모권 반환을 수행하지 않는다. 차단된 지급 요청에는 응모권을 지급하지 않으며 해당 사용자의 다른 정상 응모는 추첨에 포함한다.

`AbuseCaseSummary`의 기존 제안 필드 `id`, `userId`, `eventId?`, `sourceType`, `sourceId?`, `detectionType`, `occurredAt`, `detectedAt`은 차단 기록 계약에 맞는지 검토한다. `AbuseCaseDetail`은 차단 사유·관련 요청·검증 근거를 관리자 전용으로 제공한다. 원문 근거를 일반 사용자에게 그대로 전달하지 않는다. `sourceType` 후보가 새 차단 적용 범위를 확정하지 않는다.

기존 `reviewStatus:PENDING/ALLOWED/CONFIRMED` 필터·DTO는 신규 차단 내역 조회의 필수 계약으로 사용하지 않는다. 기록의 탐지/차단 구분, 검색·정렬·필드 타입과 저장 계약은 GD-141 설계에서 확인한다. 기존 검토자·시각·사유와 판단 이력은 이번 문서 변경을 이유로 삭제하거나 덮어쓰지 않는다.

## 참여 허용·추첨 제외 초안 제외

기존 AR03의 `ALLOW/CONFIRM`, `excludeFromEvent`, `userNoticeReason`을 이용한 참여 허용·이벤트 추첨 제외 결정은 제공하지 않는다. 관련 `eligibilityStatus`, `refundedTicketCount` 등 제외·반환 결과 DTO도 신규 계약에서 제외한다. AR03·AR06 번호는 기존 참조 보존을 위해 남기며 다른 기능의 번호로 재사용하지 않는다. 지급 후 부정 확인·점수 처리에 필요한 판단과 이력 계약은 후속 확인하며 삭제나 폐지로 임의 확정하지 않는다.

## 응모권 회수 제외

AR04·AR05는 이번 버전에서 제외한다. 회수 API·DTO·처리를 제공하지 않는다. 이벤트 취소에 따른 반환은 유지하며 추첨 제외에 따른 반환은 제공하지 않는다.

## 부정 게임 점수 무효화 — 기존 정책 유지, 후속 확인

지급 후 발견한 부정의 점수 무효화·당첨 취소·재추첨 유지 여부는 후속 확인한다. 결정 전에는 AR07의 기존 초안과 사후 처리 정책을 유지하며 지급 전 차단 기능의 필수 선행으로 묶지 않는다.

`GameInvalidationResult`: `gamePlayId:UUID`, `status:INVALID`, `statistics:MyGameStatistics`, `processedAt:instant`. 부정이 확인되고 연결된 플레이일 때만 무효화한다. 최고점·누적점수·횟수의 정확한 재계산과 최초/변경 결과 이력을 보존하며 응모권 회수는 수행하지 않는다. 실제 확인 근거와 이력·재조회·복구 계약은 담당자 확인 후 구현한다.

결정 기록: [ADR-018](../03-decisions/018-pre-issuance-abuse-blocking-and-no-draw-exclusion.md).
