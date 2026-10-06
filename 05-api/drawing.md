# 추첨·결과 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 갱신일: 2026-10-05 — GD-93 추첨 ERD 검토와 사용자 결정 반영
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [추첨·결과 규칙](../02-domain/drawing.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다. 이번 갱신은 API 초안과 정책 결정을 동기화하며 구현 완료를 뜻하지 않는다. 기본 경로 `/api/v1`은 생략한다.

## 사용자 결과 조회

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| E08 | R | `GET /events/{eventId}/results` | 없음 | 200 `PublicResults` | 공통 |
| E09 | U | `GET /events/{eventId}/my-result` | 없음 | 200 `MyResult` | 공통 |

## 공개 결과와 본인 결과

`PublicResults`:

- `eventId:UUID`, `isPublished:boolean`, `publicationScheduledAt:instant`, `publishedAt:instant?`, `revision:int?`, `updatedAt:instant?`, `serverTime:instant`
- `displayStatus:WAITING/PUBLISHED/CANCELED/NO_ENTRANTS/NO_ELIGIBLE_ENTRANTS` — 공개용 제안 값
- `prizes:PublishedPrize[]`; `PublishedPrize`는 `prizeId:UUID`, `rank:int`, `name:string`, `winnerCount:int`, `unfilledCount:int`, `winners:MaskedWinner[]`
- `MaskedWinner`: `maskedName:string`, `maskedPhoneNum:string?`, `maskedEmail:string?`. 원본 개인정보·사용자 ID·후보 가중치·검토 사유를 포함하지 않는다. 세부 마스킹 규칙은 담당자 계약에 따른다.

발표 전에는 `isPublished=false`, `publishedAt=null`, `revision=null`, `updatedAt=null`, `prizes=[]`다. 예정 시각이 지났다는 사실만으로 결과를 공개하지 않는다. `publicationScheduledAt`은 원래 예정 시각인 마감 + 5분이며 실제 공개 시각과 구분한다. 새 발표 예정 시각이나 지연 중 구체적인 화면 문구는 임의로 만들지 않는다.

정상 최초 발표는 마감 + 5분에 별도 관리자 승인 없이 자동으로 진행한다. 최초 공개 전 취소·재추첨이 있으면 관리자 확인까지 끝난 새 결과와 유지되는 기존 결과를 합쳐 최초 명단을 공개한다. 예정 시각까지 준비·확인이 끝나지 않으면 일부 명단을 먼저 공개하지 않고 최초 발표를 지연한다. 준비·확인 완료 후 서버가 최초 공개를 진행하며 예정 시각 전에는 공개하지 않는다. 시스템 장애·정합성 오류의 나머지 복구 정책은 [미결정 사항](../00-requirements/pending-decisions.md)을 따른다.

공개된 `prizes`는 등수 내림차순으로 전달해 하위 등수부터 연출할 수 있게 제안한다. 실제 선정은 상위 등수부터 진행한다. 후보 부족으로 확정된 미충원은 `unfilledCount`로 표현하며, 진행 중인 재추첨이나 관리자 확인 대기를 미충원으로 처리하지 않는다. 관리자 미확인 결과는 노출하지 않는다. 최초 발표 이후 취소부터 공개 갱신 전까지의 취소자 임시 표시는 후속 계약이다.

`MyResult`: `eventId:UUID`, `result:NOT_ENTERED/PENDING/WON/LOST/EXCLUDED/CANCELED`, `prize:Prize?`, `exclusionReason:string?`, `isPublished:boolean`, `revision:int?`, `serverTime:instant`.

`result`는 API 전용 제안 값이다. 본인 제외는 일반 낙첨과 구분한다. 최초 발표가 지연돼도 공개 전 당첨 여부는 `PENDING`, `prize=null`로 유지한다. 제외 사유는 본인에게만 공개 가능한 문구로 제공한다. 발표 후 당첨 취소·재추첨 중 본인 표시의 중간 상태는 후속 계약이다.

## 관리자 추첨·공개 명단

근거: [추첨 규칙](../02-domain/drawing.md). 권한 A, 주 담당 석종수. 공개·알림·감사는 관련 담당과 협의한다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AD01 | `GET /admin/events/{eventId}/draws` | `page,size` | 200 `Page<DrawRun>` | 공통 |
| AD02 | `GET /admin/draws/{drawId}` | 없음 | 200 `DrawRunDetail` | 공통 |
| AD03 | `GET /admin/draws/{drawId}/participants` | `page,size` | 200 `Page<DrawCandidate>` | 공통 |
| AD04 | `POST /admin/draws/{drawId}/checks` | 본문 없음 | 201 `DrawVerification` | 409 미확정 실행 |
| AD05 | `POST /admin/wins/{winId}/cancellations` | 멱등 헤더, `AwardCancellationRequest` | 202 `AwardCancellationResult`; 이미 확정된 동일 처리 200 | 409 취소 불가·부정 미확정·키 충돌 |
| AD06 | `POST /admin/draws/{drawId}/publication` | `ReasonRequest` | 200 `PublicationUpdateResult` | 409 재추첨 아님·미확정·이벤트 취소·최초 발표 전 |
| AD07 | `GET /admin/events/{eventId}/result-changes` | `page,size` | 200 `Page<PublicListChange>` | 공통 |
| AD08 | `GET /admin/draws/{drawId}/checks` | `page,size` | 200 `Page<DrawVerification>` | 공통 |
| AD09 | `POST /admin/cancellations/{cancellationId}/redraws` | 본문 없음 | 202 `DrawRun`; 이미 확정된 동일 실행 200 | 409 실행·재개 불가 |

AD01~AD07의 업무 ID를 유지하고 검증 이력·실행 시작 및 재개는 AD08~AD09로 추가했다. 이전 경로의 별칭 API를 제공하는 것으로 해석하지 않는다. 기존 초안의 `draw-runs`, `verifications`, `cancel-and-redraw`, `public-list-updates`, `public-list-changes`, `results/me` 경로를 위 제안 경로로 갱신했다. 사용자 2개·관리자 9개 구성이다. 요청·응답·업무 오류의 추가 세부사항은 구현 전 확인한다.

## 실행·후보·결과·검증 응답

| DTO | 필드 |
| --- | --- |
| `DrawRun` | `id:UUID`, `eventId:UUID`, `runNumber:int`, `executionType:AUTO/MANUAL`, `status:PREPARING/READY/RUNNING/CONFIRMED/FAILED`, `originalDrawId:UUID?`, `startedAt:instant?`, `confirmedAt:instant?`, `createdAt:instant` |
| `DrawRunDetail` | DrawRun 전체 + `algorithmVersion:string?`, `rulesSnapshot:object?`, `snapshotFixedAt:instant?`, `cancellations:CancellationSummary[]`, `results:DrawResult[]`, `failureCount:int`, `lastFailureCode:string?`, `lastFailureMessage:string?`, `lastFailedAt:instant?`, `lastFailureTraceId:string?` |
| `CancellationSummary` | `id:UUID`, `canceledDrawResultId:UUID`, `canceledDrawRunId:UUID`, `canceledBy:UUID`, `reason:string`, `canceledAt:instant` |
| `DrawCandidate` | `id:UUID`, `participantId:UUID`, `userId:UUID`, `ticketCount:long`, `weight:long`, `entrySnapshot:object`, `eligibilitySnapshot:object` |
| `DrawResult` | `id:UUID`, `prizeId:UUID`, `prizeRank:int`, `slotNumber:int`, `selectionOrder:int?`, `resultType:SELECTED/UNFILLED`, `candidateId:UUID?`, `userId:UUID?`, `wasPublished:boolean`, `isCanceled:boolean` |
| `DrawVerification` | `id:UUID`, `drawRunId:UUID`, `passed:boolean`, `checks:{code:string,passed:boolean,message:string}[]`, `verifiedAt:instant`, `verifiedBy:UUID` |

- AD03은 요청 실행에 실제로 사용한 후보 명단을 반환한다. 재추첨 후보의 ID·수량·가중치·두 스냅샷은 최초 후보 정보를 재사용하며 매 실행의 새 후보 상세 정보로 해석하지 않는다. 원래 응모 자격·가중치를 유지하고 기존 당첨자·취소자 및 이후 제외 확정자를 제거한다. 실제 내부 조회 경로는 백엔드 설계 문서에서 관리한다.
- AD02의 `cancellations`는 최초 실행에서 빈 목록, 재추첨에서 한 건 이상이다. 취소별 원래 결과와 그 결과를 만든 실행 ID를 제공한다. `previousDrawId`는 제거하고 시간순 실행은 `runNumber`로 구분한다. `originalDrawId`는 이벤트의 최초 실행을 뜻하며 시간상 직전 실행이나 취소 원본 실행과 혼동하지 않는다.
- 취소 처리자를 실행자 `executedBy` 하나로 치환하지 않고, 취소 사유를 실행 사유 `reason` 하나로 임의 합치지 않는다. 실행 상세에는 연결된 취소 근거를 목록으로 제공한다. 다른 관리자의 시작·재개 행위는 별도 감사 이력 연계 대상이며 취소 기록으로 대신하지 않는다.
- `lastFailureMessage`는 `lastFailureCode`를 관리자용 문구로 변환한 값이다. 기존 `lastFailureReason`은 대체하고 원문 예외·스택 트레이스를 응답하지 않는다. 필요하면 접근 권한이 있는 진단 기록을 `lastFailureTraceId`로 연계한다. 구체적인 코드·문구 사전과 추적 연계는 구현 후속이다.
- 기존 `publicationStatus`는 제거한다. `wasPublished`는 해당 결과가 실제 공개 명단에 반영된 이력, `isCanceled`는 당첨 취소 여부다. 공개 후 취소된 결과는 둘 다 true일 수 있다. `wasPublished`는 현재 명단 포함 여부가 아니며 실행 전체가 공개됐다는 이유만으로 최초 공개 전에 취소된 개별 결과를 true로 판정하지 않는다. 미충원 결과는 당첨 취소 대상이 아니므로 `isCanceled=false`다.
- 실행별 명단·응모권 수·가중치·경품 조건을 고정하고 같은 실행 재시도에는 고정 명단·조건을 사용한다. 이미 확정된 실행은 저장 결과를 반환한다. AD04는 정합성 검증이며 난수를 다시 돌려 같은 당첨자를 재현하는 작업이 아니다.

가중치 적용 이벤트의 등급 응모권 가중치는 실제 차감한 등급별 장수에 브론즈 1·실버 3·골드 5를 곱해 합산한다. `ticketCount`는 실제 차감 장수이고 `weight`와 구분하며 미사용·미적용 이벤트의 후보 가중치는 1이다. 등급별 값 외의 추가 상한·배율은 없다. 기준은 [추첨 도메인 규칙](../02-domain/drawing.md#가중치-및-중복-당첨)을 따른다.

## 당첨 취소와 즉시 재추첨

`AwardCancellationRequest`: 필수 `reason:string`. `abuseCaseId`의 요청·저장 연계는 어뷰징 영역 검토 이후 재논의하며 이번 갱신에서는 채택하지 않는다. 자격 미달·부정의 확인 근거가 필요하다는 기존 정책을 제거하는 것은 아니다.

`AwardCancellationResult`: `cancellationId:UUID`, `canceledDrawResultId:UUID`, `replacementDrawRunId:UUID`, `status:PREPARING/READY/RUNNING/CONFIRMED/FAILED`, `canceledAt:instant`.

AD05는 당첨 한 건의 취소와 재추첨 작업 등록을 함께 보장한다. 등록 실패 시 취소도 확정되지 않는다. 성공 응답에는 연결된 실행 ID가 있고 서버는 등록된 작업을 즉시 시작한다. 등록 성공 뒤 실제 선정이 실패하면 같은 실행을 재개한다. 취소만 확정하고 프론트엔드의 다음 호출을 기다리는 흐름으로 변경하지 않는다.

AD09는 해당 취소에 이미 연결된 실행의 시작·재개 용도다. AD05의 서버 실행과 요청이 겹쳐도 새 실행을 중복 생성하지 않는다. 진행 중이면 기존 진행 상태를 반환하고 완료됐으면 저장 결과를 반환한다. 확정된 실행을 다시 선정하지 않는다.

실행 하나에서 취소 근거를 여러 건 조회할 수 있다는 관계는 일괄 취소 요청 기능의 채택을 뜻하지 않는다. 취소·재추첨·공개 요청 재시도는 같은 처리를 중복 반영하지 않는다. 유지되는 당첨자·경품은 그대로 두고 빈 자리만 보충한다.

## 공개 명단 반영과 변경 이력

`PublicationUpdateResult`: `eventId:UUID`, `publicationId:UUID`, `drawRunId:UUID`, `revision:int`, `publishedAt:instant`, `updatedAt:instant`, `updatedBy:UUID`.

`PublicListChange`: `id:UUID`, `publicationId:UUID`, `revision:int`, `drawRunId:UUID`, `reason:string`, `confirmedBy:UUID`, `confirmedAt:instant`, `updatedAt:instant`, `changes:{prizeId:UUID,slotNumber:int,beforeUserId:UUID?,afterUserId:UUID?}[]`.

AD06은 최초 발표 이후 재추첨 결과 확인·공개 명단 갱신용이다. 최초 추첨 자체에 대한 승인 API로 사용하지 않는다. 미확정 결과·취소 이벤트는 반영할 수 없고, 동일 실행의 재반영은 revision과 알림을 중복 생성하지 않는다. AD07은 공개 버전 간 변경 대상·전후 정보·확인자·시각·사유를 조회한다.

최초 공개 전 재추첨 결과도 관리자 확인이 필요하다. 확인된 새 결과와 유지되는 기존 결과를 합쳐 최초 공개하며, 확인이 늦으면 실제 최초 발표를 지연한다. 최초 공개 전 확인을 수행하는 구체적인 API 경로·응답은 후속 계약으로 남기고 AD06이 이미 이를 제공한다고 가정하지 않는다. 최초 공개에 여러 실행의 결과를 연결하고 확인 이력을 보존하는 내부 저장 설계도 보완이 필요하다.

관리자 미확인 결과는 일반 사용자·본인 결과·알림에 노출하지 않는다. 최초 공개 알림은 응모자 전원에게 실제 공개에 연결해 생성한다. 최초 발표 후 명단 갱신은 취소자·새 당첨자에게만 안내하며 전체 재발표 알림을 보내지 않는다.

## 변경 범위와 후속 계약

- 변경 전: 실행자·사유·이전 실행을 단일 필드로 표현하고 공개와 취소를 하나의 상태로 표현했다. 변경 후: 취소별 근거·원래 결과·실행 목록과 공개 이력 여부·취소 여부를 구분한다.
- 변경 전: 최초 공개 전 취소·재추첨의 확인·최초 명단 구성은 미확정이었다. 변경 후: 확인된 최종 명단을 공개하고 준비·확인이 늦으면 최초 발표를 지연한다. 정상 자동 최초 발표는 유지한다.
- 경로·신규 DTO의 세부 검증·오류 코드·멱등 처리, 최초 공개 전 확인 API와 지연 안내, 감사 연동·어뷰징 근거 연결은 [후속 사항](../00-requirements/pending-decisions.md)을 따른다.
- 내부 스키마·트랜잭션·Query·검증은 [백엔드 DB 문서](https://github.com/GETDDO/getddo-be/tree/dev/docs/03-database)와 [백엔드 ADR](https://github.com/GETDDO/getddo-be/tree/dev/docs/04-decisions)에서 관리한다.
