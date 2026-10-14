# 추첨·결과 API 초안

- 상태: 정책 변경 검토 완료 / 최초 재개 GD-87 및 개별 취소·재추첨 GD-138 일부 계약 확정(2026-10-12), GD-169 선택 범위 확정(2026-10-13), 사용자 결과 조회 GD-140 계약 채택(2026-10-14), 나머지 상세 API 계약은 담당자 검토 대기
- 갱신일: 2026-10-14 — GD-140 계약·마스킹과 자동 추첨/수동 최초 공개 분리 반영
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [추첨·결과 규칙](../02-domain/drawing.md)

아래 사용자 결과 조회 GD-140, GD-87·GD-138 및 2026-10-13 GD-169 연동 계약 절에서 승인한 항목 외의 경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다. 이번 갱신은 API 초안과 정책 결정을 동기화하며 구현 완료를 뜻하지 않는다. 기본 경로 `/api/v1`은 생략한다.

## 사용자 결과 조회

2026-10-14 사용자 확인으로 E08·E09의 아래 경로·권한·성공 응답·DTO·상태값·정렬 초안을 채택한다. 기존 공통 오류 계약을 따르며, 다른 관리자 API의 미확정 부분까지 채택한 것은 아니다. 구현 완료를 뜻하지 않는다.

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| E08 | R | `GET /events/{eventId}/results` | 없음 | 200 `PublicResults` | 공통 |
| E09 | U | `GET /events/{eventId}/my-result` | 없음 | 200 `MyResult` | 공통 |

## 공개 결과와 본인 결과

`PublicResults`:

- `eventId:UUID`, `isPublished:boolean`, `publishedAt:instant?`, `revision:int?`, `updatedAt:instant?`, `serverTime:instant`
- `displayStatus:WAITING/PUBLISHED/CANCELED/NO_ENTRANTS` — 공개용 상태값
- `prizes:PublishedPrize[]`; `PublishedPrize`는 `prizeId:UUID`, `rank:int`, `name:string`, `winnerCount:int`, `unfilledCount:int`, `winners:MaskedWinner[]`
- `MaskedWinner`: `maskedName:string`, `maskedPhoneNum:string?`. 원본 개인정보·사용자 ID·후보 가중치·검토 사유·이메일을 포함하지 않는다. 이름과 전화번호는 [결과 발표의 마스킹 규칙](../02-domain/drawing.md#결과-발표)을 따른다. `maskedPhoneNum`은 전화번호 끝 4자리만 제공한다. 미등록 값의 별도 대체 문구는 이번 범위에서 다루지 않는다.

발표 전에는 `isPublished=false`, `publishedAt=null`, `revision=null`, `updatedAt=null`, `prizes=[]`다. `publicationScheduledAt`은 이번 사용자 계약에서 제외한다. 마감 +5분은 자동 추첨 기준이며 공개 예정 시각이 아니다. 고정된 발표 예정 시각·카운트다운·새 예정 시각을 응답하지 않고 서버의 실제 공개 여부에 따라 발표 대기를 표시한다.

정상 최초 발표는 마감 + 5분에 별도 관리자 승인 없이 자동으로 진행한다. 최초 공개 전 취소·재추첨이 있으면 관리자 확인까지 끝난 새 결과와 유지되는 기존 결과를 합쳐 최초 명단을 공개한다. 예정 시각까지 준비·확인이 끝나지 않으면 일부 명단을 먼저 공개하지 않고 최초 발표를 지연한다. 준비·확인 완료 후 서버가 최초 공개를 진행하며 예정 시각 전에는 공개하지 않는다. 시스템 장애·정합성 오류의 나머지 복구 정책은 [미결정 사항](../00-requirements/pending-decisions.md)을 따른다.

공개된 `prizes`는 등수 내림차순으로 전달해 하위 등수부터 연출할 수 있게 한다. 실제 선정은 상위 등수부터 진행한다. 후보 부족으로 확정된 미충원은 `unfilledCount`로 표현하며, 진행 중인 재추첨이나 관리자 확인 대기를 미충원으로 처리하지 않는다. 관리자 미확인 결과는 노출하지 않는다. 발표 후 취소·재추첨을 이번 운영에서 사용하지 않는 전제로 취소자 임시 표시는 이번 GD-140에서 구현하지 않는다. 기존 관리자 기능의 삭제·발표 후 호출 차단을 의미하지 않는다.

`MyResult`: `eventId:UUID`, `result:NOT_ENTERED/PENDING/WON/LOST/CANCELED`, `prize:Prize?`, `isPublished:boolean`, `revision:int?`, `serverTime:instant`.

`result`는 API 전용 상태값이다. 최초 발표가 지연돼도 공개 전 당첨 여부는 `PENDING`, `prize=null`로 유지한다. 발표 후 당첨 취소·재추첨 중 본인 중간 상태는 이번 GD-140 범위에서 제외한다. 기존 초안의 상태값은 유지하되, 개별 당첨 취소의 새로운 중간 응답 의미를 이번 결정으로 추가하지 않는다. 실제 운영에서 기존 관리자 기능을 사용하려면 중간 표시 계약을 먼저 보완한다.

## 관리자 추첨·공개 명단

근거: [추첨 규칙](../02-domain/drawing.md). 권한 A, 주 담당 석종수. 공개·알림·감사는 관련 담당과 협의한다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AD01 | `GET /admin/events/{eventId}/draws` | `page,size` | 200 `Page<DrawRun>` | 공통 |
| AD02 | `GET /admin/draws/{drawId}` | 없음 | 200 `DrawRunDetail` | 공통 |
| AD03 | `GET /admin/draws/{drawId}/participants` | `page,size` | 200 `Page<DrawCandidate>` | 공통 |
| AD04 | `POST /admin/draws/{drawId}/checks` | 본문 없음 | 201 `DrawVerification` | 409 미확정 실행 |
| AD05 | `POST /admin/wins/{winId}/cancellations` | 멱등 헤더, `AwardCancellationRequest` | 202 `AwardCancellationResult`; 이미 확정된 동일 처리 200 | 409 취소 불가·키 충돌 |
| AD06 | `POST /admin/draws/{drawId}/publication` | 본문 없음 | 200 `PublicationUpdateResult` | 409 재추첨 아님·미확정·이벤트 취소·최초 발표 전 |
| AD07 | `GET /admin/events/{eventId}/result-changes` | `page,size` | 200 `Page<PublicListChange>` | 공통 |
| AD08 | `GET /admin/draws/{drawId}/checks` | `page,size` | 200 `Page<DrawVerification>` | 공통 |
| AD09 | `POST /admin/cancellations/{cancellationId}/redraws` | 본문 없음 | 202 `DrawExecutionResponse`; 이미 확정된 동일 실행 200 | 409 실행·재개 불가 |

AD01~AD07의 업무 ID를 유지하고 검증 이력·실행 시작 및 재개는 AD08~AD09로 추가했다. 이전 경로의 별칭 API를 제공하는 것으로 해석하지 않는다. 기존 초안의 `draw-runs`, `verifications`, `cancel-and-redraw`, `public-list-updates`, `public-list-changes`, `results/me` 경로를 위 제안 경로로 갱신했다. 사용자 2개·관리자 9개 구성이다. 요청·응답·업무 오류의 추가 세부사항은 구현 전 확인한다.

## 최초 추첨 실패 재개 — GD-87 일부 계약 확정

2026-10-12 사용자 확인으로 아래 최초 재개 경로·입력·권한·성공 및 상태 충돌 코드만 확정했다. AD01~AD09의 상세 계약·업무 ID 배정·응답 DTO의 전체 필드와 오류 코드 사전은 이 확인으로 일괄 확정하지 않는다.

| 항목 | 확인한 계약 |
| --- | --- |
| 메서드·경로 | `POST /admin/draws/{runId}/resume` (공통 기본 경로 `/api/v1`) |
| 입력 | 최초 추첨 실행 ID(UUID), 본문 없음 |
| 권한 | 활성 관리자만 허용 |
| 대상 | 실패한 기존 INITIAL 실행의 재개. 새 최초 실행·취소·재추첨을 생성하지 않음 |
| 성공 | 저장된 CONFIRMED 결과 및 재개 중 확정된 NO_ENTRIES 정상 종료는 200, 진행 중 또는 재개 후 미확정 상태는 202 |
| 거절 | 유효 사용자 문맥의 비관리자 403, REPLACEMENT·재개 불가 상태·실행 불가 이벤트 409 |

진행 중이면 기존 상태, CONFIRMED이면 저장된 결과를 반환하며 다시 선정하지 않는다. 입력 확정 후 저장된 스냅샷을 재사용한다. `NO_ENTRIES`·과거 `NO_CANDIDATES`는 재개하지 않는다. 재개 결과가 NO_ENTRIES이면 200으로 정상 종료하며 조회를 끝낸다. 202는 선정·결과 확정 성공을 보장하지 않으므로 같은 실행을 조회한다.

최초 재개 진입점은 GD-87, 당첨 취소·재추첨 시작 및 재개 AD05·AD09는 GD-138 책임이다. 일반 자동 실행이 운영자 확인 대기를 해제하지 않는 원칙을 유지한다. 실제 처리자·시작 및 재개 행위는 감사 이력과 연결하고 취소 처리자로 대체하지 않는다.

이 절은 확인된 외부 계약을 기록하며 구현·운영 적용 완료를 뜻하지 않는다. 실패 상세 조회는 GD-137과 연결하고 별도 중복 조회 경로를 추가하지 않는다. 응답 DTO 필드와 재시도 예약·운영자 대기의 저장 방식은 백엔드 구현 계약에서 관리하며 실행 검증 결과와 구분한다.

## 개별 취소·재추첨 — GD-138 일부 계약 확정

2026-10-12 사용자 승인 범위는 아래 항목이다. AD09 응답은 기존 메타데이터 전용 `DrawRun` 초안에서 GD-87의 `DrawExecutionResponse` 재사용으로 변경해 확정 결과 재반환을 응답에서도 확인한다. 프론트엔드는 AD09의 `runId`, `status`, `confirmation`을 사용한다. 다른 조회·공개 API의 `DrawRun` 계약은 이 승인으로 변경하지 않는다.

| 항목 | 확인한 계약 |
| --- | --- |
| AD05 경로 | `POST /admin/wins/{winId}/cancellations` |
| AD05 입력 | UUID `Idempotency-Key` 필수, `reason` 필수·양끝 공백 제거 후 1~1000자 |
| AD05 응답 | `AwardCancellationResult`의 cancellationId, canceledDrawResultId, replacementDrawRunId, status, canceledAt |
| AD09 경로·입력 | `POST /admin/cancellations/{cancellationId}/redraws`, 본문 없음. 기존 취소에 연결된 같은 실행만 사용 |
| AD09 응답 | `ResponseEnvelope<DrawExecutionResponse>`; runId:UUID, status, confirmation:null 또는 저장된 확정 결과 |
| 권한·성공 | 활성 관리자만 허용, 저장된 CONFIRMED 결과는 200, 미확정은 202 |
| 거절 | 없는 대상 404, 취소 불가·멱등 키/본문 충돌 409. 형식·사유 검증 실패는 400, 유효 비관리자 문맥은 403 |
| 동시 취소 | 같은 이벤트에 미확정 REPLACEMENT가 있으면 다른 당첨의 새 취소는 409. 기존 취소 재요청·AD09 시작/재개는 허용 |

`confirmation`은 `confirmedAt:instant`와 저장된 `results` 목록이다. 결과 항목은 `resultId:UUID`, `prizeId:UUID`, `rank:int`, `slotNumber:int`, `selectionOrder:int?`, `candidateId:UUID?`, `userId:UUID?`, `resultType:SELECTED/UNFILLED`를 제공한다. CONFIRMED 재요청은 같은 저장 결과를 반환하며 재선정하지 않는다. 실패 원문·스택 트레이스를 응답에 추가하지 않는다.

2026-10-13 사용자 결정으로 취소 멱등 키의 전역 범위와 `CANCEL-001`~`CANCEL-004`의 의미·HTTP 상태를 아래 GD-169 계약 절에서 확정했다. 다른 오류 코드 사전 전체는 확정하지 않는다. 전체 당첨 취소·전체 재추첨, 공개 전환과 어뷰징 근거 연결도 이번 API 승인으로 확정하지 않는다.

## 실행·후보·결과·검증 응답

| DTO | 필드 |
| --- | --- |
| `DrawRun` | `id:UUID`, `eventId:UUID`, `runNumber:int`, `executionType:AUTO/MANUAL`, `status:PREPARING/READY/RUNNING/CONFIRMED/FAILED/NO_ENTRIES`, `originalDrawId:UUID?`, `startedAt:instant?`, `confirmedAt:instant?`, `createdAt:instant` |
| `DrawRunDetail` | DrawRun 전체 + `algorithmVersion:string?`, `rulesSnapshot:object?`, `snapshotFixedAt:instant?`, `cancellations:CancellationSummary[]`, `results:DrawResult[]`, `failureCount:int`, `lastFailureCode:string?`, `lastFailureMessage:string?`, `lastFailedAt:instant?`, `lastFailureTraceId:string?` |
| `CancellationSummary` | `id:UUID`, `canceledDrawResultId:UUID`, `canceledDrawRunId:UUID`, `canceledBy:UUID`, `reason:string`, `canceledAt:instant` |
| `DrawCandidate` | `id:UUID`, `participantId:UUID`, `userId:UUID`, `ticketCount:long`, `weight:long`, `entrySnapshot:object`, `eligibilitySnapshot:object` |
| `DrawResult` | `id:UUID`, `prizeId:UUID`, `prizeRank:int`, `slotNumber:int`, `selectionOrder:int?`, `resultType:SELECTED/UNFILLED`, `candidateId:UUID?`, `userId:UUID?`, `wasPublished:boolean`, `isCanceled:boolean` |
| `DrawVerification` | `id:UUID`, `drawRunId:UUID`, `passed:boolean`, `checks:{code:string,passed:boolean,message:string}[]`, `verifiedAt:instant`, `verifiedBy:UUID` |

- AD03은 요청 실행에 실제로 사용한 후보 명단을 반환한다. 재추첨 후보의 ID·수량·가중치·두 스냅샷은 최초 후보 정보를 재사용하며 매 실행의 새 후보 상세 정보로 해석하지 않는다. 원래 응모 자격·가중치를 유지하고 기존 당첨자·취소자를 제거한다. 어뷰징 차단 기록을 이유로 다른 정상 응모를 제거하지 않는다. 실제 내부 조회 경로는 백엔드 설계 문서에서 관리한다.
- AD02의 `cancellations`는 최초 실행에서 빈 목록, 재추첨에서 한 건 이상이다. 취소별 원래 결과와 그 결과를 만든 실행 ID를 제공한다. `previousDrawId`는 제거하고 시간순 실행은 `runNumber`로 구분한다. `originalDrawId`는 이벤트의 최초 실행을 뜻하며 시간상 직전 실행이나 취소 원본 실행과 혼동하지 않는다.
- 취소 처리자를 실행자 `executedBy` 하나로 치환하지 않고, 취소 사유를 실행 사유 `reason` 하나로 임의 합치지 않는다. 실행 상세에는 연결된 취소 근거를 목록으로 제공한다. 다른 관리자의 시작·재개 행위는 별도 감사 이력 연계 대상이며 취소 기록으로 대신하지 않는다.
- `lastFailureMessage`는 `lastFailureCode`를 관리자용 문구로 변환한 값이다. 기존 `lastFailureReason`은 대체하고 원문 예외·스택 트레이스를 응답하지 않는다. 필요하면 접근 권한이 있는 진단 기록을 `lastFailureTraceId`로 연계한다. 구체적인 코드·문구 사전과 추적 연계는 구현 후속이다.
- 기존 `publicationStatus`는 제거한다. `wasPublished`는 해당 결과가 실제 공개 명단에 반영된 이력, `isCanceled`는 당첨 취소 여부다. 공개 후 취소된 결과는 둘 다 true일 수 있다. `wasPublished`는 현재 명단 포함 여부가 아니며 실행 전체가 공개됐다는 이유만으로 최초 공개 전에 취소된 개별 결과를 true로 판정하지 않는다. 미충원 결과는 당첨 취소 대상이 아니므로 `isCanceled=false`다.
- 실행별 명단·응모권 수·가중치·경품 조건을 고정하고 같은 실행 재시도에는 고정 명단·조건을 사용한다. 이미 확정된 실행은 저장 결과를 반환한다. AD04는 정합성 검증이며 난수를 다시 돌려 같은 당첨자를 재현하는 작업이 아니다.

가중치 적용 이벤트의 등급 응모권 가중치는 실제 차감한 등급별 장수에 브론즈 1·실버 3·골드 5를 곱해 합산한다. `ticketCount`는 실제 차감 장수이고 `weight`와 구분하며 미사용·미적용 이벤트의 후보 가중치는 1이다. 등급별 값 외의 추가 상한·배율은 없다. 기준은 [추첨 도메인 규칙](../02-domain/drawing.md#가중치-및-중복-당첨)을 따른다.

## 당첨 취소와 즉시 재추첨

`AwardCancellationRequest`: 필수 `reason:string`. 관리자가 사유를 기록해 개별 당첨을 취소할 수 있으며 어뷰징·자격 미달 확인을 필수 조건으로 요구하지 않는다. 부정 미확정을 일반 관리자 취소의 거절 조건으로 사용하지 않는다. `abuseCaseId`의 선택적 요청·저장 연계는 후속 확인이며 필수 계약으로 채택하지 않는다. 사유 길이·권한·HTTP 상태는 위 GD-138 일부 계약 확정 절을 따른다. 취소 오류 코드와 전역 멱등 키 범위는 아래 GD-169 확정 계약을 따른다.

`AwardCancellationResult`: `cancellationId:UUID`, `canceledDrawResultId:UUID`, `replacementDrawRunId:UUID`, `status:PREPARING/READY/RUNNING/CONFIRMED/FAILED`, `canceledAt:instant`.

AD05는 당첨 한 건의 취소와 재추첨 작업 등록을 함께 보장한다. 등록 실패 시 취소도 확정되지 않는다. 성공 응답에는 연결된 실행 ID가 있고 서버는 등록된 작업을 즉시 시작한다. 등록 성공 뒤 실제 선정이 실패하면 같은 실행을 재개한다. 취소만 확정하고 프론트엔드의 다음 호출을 기다리는 흐름으로 변경하지 않는다.

AD09는 해당 취소에 이미 연결된 실행의 시작·재개 용도다. AD05의 서버 실행과 요청이 겹쳐도 새 실행을 중복 생성하지 않는다. 진행 중이면 기존 진행 상태를 반환하고 완료됐으면 저장 결과를 반환한다. 확정된 실행을 다시 선정하지 않는다.

실행 하나에서 취소 근거를 여러 건 조회할 수 있다는 관계는 일괄 취소 요청 기능의 채택을 뜻하지 않는다. 취소·재추첨·공개 요청 재시도는 같은 처리를 중복 반영하지 않는다. 유지되는 당첨자·경품은 그대로 두고 빈 자리만 보충한다.

## 공개 명단 반영과 변경 이력

`PublicationUpdateResult`: `eventId:UUID`, `publicationId:UUID`, `drawRunId:UUID`, `revision:int`, `publishedAt:instant`, `updatedAt:instant`, `updatedBy:UUID`.

`PublicListChange`: `id:UUID`, `publicationId:UUID`, `revision:int`, `drawRunId:UUID`, `reason:string`, `confirmedBy:UUID`, `confirmedAt:instant`, `updatedAt:instant`, `changes:{prizeId:UUID,slotNumber:int,beforeUserId:UUID?,afterUserId:UUID?}[]`.

기존 AD06 구현은 최초 발표 이후 재추첨 확인·공개 명단 갱신용이다. 2026-10-14 결정에서는 확인과 공개를 분리하고 최초 INITIAL 확인·공개도 제공한다. GD-199에서 기존 경로의 대상 확장·분리를 우선 검토하며 세부 계약을 보완한다. 미확정·미확인 결과·취소 이벤트는 공개할 수 없고 반복 공개는 revision과 알림을 중복 생성하지 않는다. AD07은 실제 공개 버전 간 변경 이력을 조회한다.

최초 INITIAL 및 발표 전 REPLACEMENT 결과에 관리자 확인이 필요하다. 확인 저장은 공개를 수행하지 않는다. 확인된 최종 명단은 별도 관리자 공개 요청으로 공개한다. 기존 /result-confirmation·/publication 경로를 확장하는 방안은 GD-199에서 입력·대상·응답·업무 오류와 함께 확인하며 이번 정책 변경을 코드 구현 완료로 해석하지 않는다.

관리자 미확인 결과는 일반 사용자·본인 결과·알림에 노출하지 않는다. 최초 공개 알림은 응모자 전원에게 실제 공개에 연결해 생성한다. 최초 발표 후 명단 갱신은 취소자·새 당첨자에게만 안내하며 전체 재발표 알림을 보내지 않는다.

## 변경 범위와 후속 계약

- 변경 전: 실행자·사유·이전 실행을 단일 필드로 표현하고 공개와 취소를 하나의 상태로 표현했다. 변경 후: 취소별 근거·원래 결과·실행 목록과 공개 이력 여부·취소 여부를 구분한다.
- 변경 전: 최초 공개 전 취소·재추첨의 확인·최초 명단 구성은 미확정이었다. 변경 후: 확인된 최종 명단을 공개하고 준비·확인이 늦으면 최초 발표를 지연한다. 정상 자동 최초 발표는 유지한다.
- 아래 GD-169 절에서 확정한 종료 응답·전역 멱등 키·취소 코드·GD-139 확인 경로와 조회/공개 정보 제공 방향을 따른다. 나머지 DTO·추가 조회 경로·지연 안내·감사 세부 연동·어뷰징 근거 연결은 [후속 사항](../00-requirements/pending-decisions.md)을 따른다.
- 내부 스키마·트랜잭션·Query·검증은 [백엔드 DB 문서](https://github.com/GETDDO/getddo-be/tree/dev/docs/03-database)와 [백엔드 ADR](https://github.com/GETDDO/getddo-be/tree/dev/docs/04-decisions)에서 관리한다.

## 2026-10-11 추첨 제외 폐지

- 신규 응답은 어뷰징 추첨 제외 결과·제외 사유와 전원 제외 종료를 제공하지 않는다. 응모자 없음 종료는 유지한다.
- 과거 제외 상태·스냅샷·응답을 소급 삭제하거나 재작성하지 않는다. 기존 데이터가 있는 경우의 조회 호환성과 공개 API 전환은 구현 전 담당자와 검토한다.
- 지급 후 당첨 취소·재추첨과 마감 후 5분 일정은 후속 결정 전까지 유지한다.
- 결정 기록: [ADR-018](../03-decisions/018-pre-issuance-abuse-blocking-and-no-draw-exclusion.md).

## 2026-10-12 일반 관리자 취소·재추첨 정책 반영

- AD05는 관리자 사유에 따른 개별 당첨 취소와 빈 자리 재추첨 등록에 사용한다. AD09는 이미 연결된 동일 실행의 시작·재개이며 새 전체 재추첨을 생성하지 않는다.
- 유지 당첨자·경품·최초 가중치, 취소와 등록의 원자성 및 확정 결과 불변을 유지한다. 당첨 취소·재추첨을 이유로 응모권을 반환하거나 회수하지 않는다.
- 전체 당첨 취소·처음부터 전체 재추첨은 추후 추가 예정이며 신규 경로·DTO를 이번 초안에 추가하지 않는다.
- 기존 상세 API 초안의 검토 상태는 유지하며 정책 변경만 반영한다. 결정 기록: [ADR-019](../03-decisions/019-admin-award-cancellation-and-partial-redraw.md).

## 2026-10-13 GD-169 연동 계약 확정

사용자가 항목 1~7의 추천안 A를 선택하고 항목 8은 기존 이벤트 상태 변경 코드만 재사용하도록 확정했다. 이번 기록은 계약·구현 방향의 승인이고 코드 구현 완료가 아니다.

### 정상 종료·취소 멱등·오류

- 최초 재개 중 응모자 없음으로 NO_ENTRIES가 확정되면 HTTP 200을 반환하고 클라이언트는 같은 실행의 진행 조회를 종료한다. 당첨 결과 유무는 실행 상태로 구분한다. 이미 NO_ENTRIES로 종료된 실행을 새로 재개하는 행위는 허용하지 않는다. 과거 NO_CANDIDATES의 응답 호환성을 이번 결정으로 변경하지 않는다.
- AD05 Idempotency-Key는 전역 범위다. 같은 키·같은 관리자·같은 당첨 대상·같은 정규화 사유의 재요청은 기존 취소·연결 실행을 재사용한다. 다른 대상·내용·관리자가 같은 키를 재사용하면 409/CANCEL-003을 반환한다.
- UUID 키 형식과 사유 1~1000자·공백 정규화, 신규 취소 중복·진행 중 재추첨의 기존 제한은 유지한다.

| 코드 | HTTP | 의미 |
| --- | --- | --- |
| CANCEL-001 | 404 | 당첨 또는 취소 기록을 찾을 수 없음 |
| CANCEL-002 | 409 | 현재 당첨을 취소할 수 없음 또는 실행 연결 등 취소 업무 조건 불충족 |
| CANCEL-003 | 409 | 같은 멱등 키가 다른 취소 요청에 사용됨 |
| CANCEL-004 | 400 | 취소 대상·멱등 키·필수 사유 입력이 유효하지 않음 |

코드와 HTTP·업무 의미를 계약으로 확정한다. 프레임워크의 UUID·본문 형식 검증과 기존 공통 권한/실행 오류까지 이 네 코드로 치환하지 않는다. 사용자에게 노출되는 문구는 의미를 유지하고 원문 예외를 포함하지 않는다.

### GD-139 승인된 확인·공개 API 동기화

- 기존 AD06은 본문 없이 연결 취소 사유를 사용해 최초 공개 후 REPLACEMENT 확인·명단 반영을 수행했다. 2026-10-14 이후 확인/공개 분리·최초 공개 지원을 GD-199에서 보완하며, 이 절의 과거 최초 공개 후 전용 제한을 새 흐름의 최종 계약으로 사용하지 않는다.
- 최초 공개 전 확인은 POST /admin/draws/{runId}/result-confirmation, 본문 없음, 활성 관리자 전용이다. 성공은 200이며 최초 공개 후 새로운 사전 확인은 거절하고 기존 확인의 재요청은 허용한다.
- 확인 응답은 eventId, confirmationId, runId, confirmedBy, confirmedAt, isPublished, publicationId, revision, publishedAt을 제공한다. 실제 미공개이면 publicationId·revision·publishedAt은 null이다. 실제 공개 전 결과를 공개로 가장하지 않는다.
- AD07 페이지는 page 기본 1, size 기본 20·허용 1~100이며 실제 공개 버전의 최신순이다. 추가 경로·업무 번호·모든 오류 문자열을 이번 동기화로 확정하지 않는다.

### 공개 버전에 반영된 실행별 근거

- 복수 재추첨을 함께 공개하면 반영된 실행마다 연결 취소 사유 목록·확인자·확인 시각을 목록으로 제공한다. 과거 유지 결과의 실행 근거와 이번 갱신의 계기 실행을 구분한다. 최초 INITIAL 결과도 관리자 확인을 요구한다. 2026-10-14 결정 이후 확인 저장은 공개하지 않으며 공개 요청을 별도로 처리한다.
- 기존 단일 reason·confirmedBy·confirmedAt·drawRunId는 계기 실행의 정보로 유지한다. 실행별 목록을 추가하고 기존 필드의 의미를 바꾸지 않는다.
- 실행 식별자, 연결 취소 식별자/사유, 확인 기록 식별자/처리자/시각을 대응시켜 사유와 확인자를 임의로 합치지 않는다. 목록 필드 이름·순서·DTO 매핑은 GD-169 구현 시 기존 응답과 맞춰 기록한다. 이 목록은 관리자 전용이다.

### 관리자 상세 요약과 전체 이력

- 실행 상세에는 최근 실패 코드·관리자용 문구·시각, 실패 횟수, 재시도 예약·운영자 대기 상태, 연결 취소 근거 요약을 제공한다. 실제 시작·재개 처리자와 취소 처리자를 구분한다.
- 전체 실패·감사 이력은 별도 페이징 조회로 제공하고 기존 감사 조회 기능을 우선 재사용한다. 실패 상세의 중복 조회 API를 만들지 않는다.
- 원문 예외·스택 트레이스·불필요한 개인정보를 노출하지 않는다. 구체적인 추가 경로·페이지 상세·문구 사전·감사 연계는 GD-169에서 기존 계약 확인 후 기록한다. 요약/페이징 분리 선택만으로 새 경로를 임의 확정하지 않는다.

### 알림과 이벤트 상태 경계

실제 공개 저장과 함께 전달할 기록을 보존하고 커밋 후 기존 알림 기능으로 전달한다. 실패 시 재전달하고 공개 식별자로 중복을 방지한다. 수신 대상의 기존 최초 발표/명단 갱신 규칙을 유지한다. 알림 생성·발송 자체는 알림 담당 영역이며 이번 변경은 추첨 측 호출 연결이다.

이벤트 상태 변경은 기존 코드·호출 계약이 있으면 재사용한다. 없다면 GD-169에서 신규 상태 변경 기능·API를 만들지 않고 미연결 사항으로 기록한다. 기존 상태 코드 확인은 새 기능 추가 승인으로 해석하지 않는다.

## 2026-10-14 관리자 수동 최초 공개

- 최초 자동 추첨 결과를 관리자 조회에서 검토하고, 필요 시 기존 AD05·AD09로 개별 당첨 취소·빈 자리 재추첨을 처리한다. 정상 응모자 후보 제외는 추가하지 않는다.
- 관리자 확인과 수동 공개는 별도 동작이다. 최초 결과도 확인해야 하며 명단이 변경되면 변경된 결과를 확인한 뒤 공개한다.
- E08·E09는 실제 공개 전 WAITING·PENDING과 비공개 필드를 유지하고 실제 공개 버전만 제공한다. 발표 예정 시각을 삭제하고 화면 문구는 '응모 마감 · 결과 발표 대기'를 사용한다.
- 스케줄러의 자동 공개·공개 재시도는 없다. 공개 요청 재시도는 같은 명단·버전의 중복을 방지한다. 공개 이후 알림 전달 재시도는 별개이며 기존 담당 기능을 유지한다.
- 최초 확인·수동 공개의 API 입력·응답·오류·경로 확장 세부사항은 GD-199에서 보완한다. 기존 AD06과 사전 확인 API의 자동 반영 구현을 그대로 승인한 것이 아니다.
- 정책 원본: [추첨 규칙](../02-domain/drawing.md#결과-발표), [ADR-022](../03-decisions/022-manual-initial-publication.md). 과거 자동 최초 공개 문구는 이 결정으로 대체한다.
