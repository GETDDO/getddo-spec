# 추첨·결과 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [추첨·결과 규칙](../02-domain/drawing.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 사용자 결과 조회

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| E08 | R | `GET /events/{eventId}/results` | 없음 | 200 `PublicResults` | 공통 |
| E09 | U | `GET /events/{eventId}/results/me` | 없음 | 200 `MyResult` | 공통 |

## 공개 결과와 본인 결과

`PublicResults`:

- `eventId:UUID`, `isPublished:boolean`, `publicationScheduledAt:instant`, `publishedAt:instant?`, `revision:int?`, `updatedAt:instant?`, `serverTime:instant`
- `displayStatus:WAITING/PUBLISHED/CANCELED/NO_ENTRANTS/NO_ELIGIBLE_ENTRANTS` — 공개용 제안 값
- `prizes:PublishedPrize[]`; `PublishedPrize`는 `prizeId:UUID`, `rank:int`, `name:string`, `winnerCount:int`, `unfilledCount:int`, `winners:MaskedWinner[]`
- `MaskedWinner`: `maskedName:string`, `maskedPhoneNum:string?`, `maskedEmail:string?`. 원본 개인정보·사용자 ID·후보 가중치·검토 사유를 포함하지 않는다. 세부 마스킹 규칙은 담당자 계약에 따른다.

발표 전에는 `isPublished=false`, `publishedAt=null`, `revision=null`, `updatedAt=null`, `prizes=[]`다. 예정 시각이 지났다는 사실만으로 결과를 공개하지 않는다. 정상 최초 발표는 마감 + 5분에 서버가 수행한다. 장애 시 안내·복구 정책은 발표 실패 처리 계약이며, 준비 실패를 낙첨이나 대상 없음으로 위장하지 않는다.

공개된 `prizes`는 등수 내림차순으로 전달해 하위 등수부터 연출할 수 있게 제안한다. 실제 선정은 상위 등수부터 진행한다. 후보 부족은 `unfilledCount`로 표현한다. 재추첨 중에도 관리자 미확인 결과를 노출하지 않는다. 기존 공개 명단 중 당첨 취소자의 임시 표시 방식은 담당자 계약에 따른다.

`MyResult`: `eventId:UUID`, `result:NOT_ENTERED/PENDING/WON/LOST/EXCLUDED/CANCELED`, `prize:Prize?`, `exclusionReason:string?`, `isPublished:boolean`, `revision:int?`, `serverTime:instant`.

`result`는 API 전용 제안 값이다. 본인 제외는 일반 낙첨과 구분한다. 공개 전 당첨 여부는 `PENDING`, `prize=null`로 유지한다. 제외 사유는 본인에게만 공개 가능한 문구로 제공한다. 당첨 취소·재추첨 후 본인 표시의 중간 상태 역시 담당자 계약을 따른다.

## 관리자 추첨·공개 명단

근거: [추첨 규칙][drawing]. 권한 A, 주 담당 석종수. 공개와 알림은 전민규와 협의한다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AD01 | `GET /admin/events/{eventId}/draw-runs` | `page,size` | 200 `Page<DrawRun>` | 공통 |
| AD02 | `GET /admin/draw-runs/{drawRunId}` | 없음 | 200 `DrawRunDetail` | 공통 |
| AD03 | `GET /admin/draw-runs/{drawRunId}/candidates` | `page,size` | 200 `Page<DrawCandidate>` | 공통 |
| AD04 | `POST /admin/draw-runs/{drawRunId}/verifications` | 본문 없음 | 201 `DrawVerification` | 409 미확정 실행 |
| AD05 | `POST /admin/draw-results/{drawResultId}/cancel-and-redraw` | 멱등 헤더, `AwardCancellationRequest` | 202 `AwardCancellationResult`; 이미 확정된 동일 처리 200 | 409 취소 불가·부정 미확정·키 충돌 |
| AD06 | `POST /admin/draw-runs/{drawRunId}/public-list-updates` | `ReasonRequest` | 200 `PublicationUpdateResult` | 409 재추첨 아님·미확정·이벤트 취소·최초 발표 전 |
| AD07 | `GET /admin/events/{eventId}/public-list-changes` | `page,size` | 200 `Page<PublicListChange>` | 공통 |

## 실행·스냅샷·검증

| DTO | 필드 |
| --- | --- |
| `DrawRun` | `id:UUID`, `eventId:UUID`, `runNumber:int`, `executionType:AUTO/MANUAL`, `status:PREPARING/READY/RUNNING/CONFIRMED/FAILED`, `previousDrawId:UUID?`, `originalDrawId:UUID?`, `executedBy:UUID?`, `reason:string?`, `startedAt:instant?`, `confirmedAt:instant?`, `createdAt:instant` |
| `DrawRunDetail` | DrawRun 전체 + `algorithmVersion:string?`, `rulesSnapshot:object?`, `snapshotFixedAt:instant?`, `results:DrawResult[]`, `failureCount:int`, `lastFailureCode:string?`, `lastFailureReason:string?`, `lastFailedAt:instant?` |
| `DrawCandidate` | `id:UUID`, `participantId:UUID`, `userId:UUID`, `ticketCount:long`, `weight:decimal`, `entrySnapshot:object`, `eligibilitySnapshot:object` |
| `DrawResult` | `id:UUID`, `prizeId:UUID`, `prizeRank:int`, `slotNumber:int`, `selectionOrder:int?`, `resultType:SELECTED/UNFILLED`, `candidateId:UUID?`, `userId:UUID?`, `publicationStatus:PENDING/PUBLISHED/EXCLUDED/null` |
| `DrawVerification` | `id:UUID`, `drawRunId:UUID`, `passed:boolean`, `checks:{code:string,passed:boolean,message:string}[]`, `verifiedAt:instant`, `verifiedBy:UUID` |

실행별 명단·응모권 수·가중치·경품 조건을 고정하고 같은 실행 재시도에는 기존 결과를 사용한다. AD04는 실행 당시 조건과 결과를 대조하는 작업이며 무작위 추첨을 다시 돌려 동일 당첨자를 재현한다는 의미가 아니다.

가중치 환산식·가중치 값 상한·추가 배율은 **가중치·마스킹 세부 계약**이다. `weight=ticketCount`로 확정하지 않는다. 유효 후보보다 자리가 많으면 상위 등수부터 배정하고 나머지를 UNFILLED로 기록한다. 모든 등수에 걸친 중복 당첨을 방지한다.

## 당첨 취소와 즉시 재추첨

`AwardCancellationRequest`: 필수 `reason:string`; 선택 `abuseCaseId:UUID`. 어뷰징 근거라면 확정된 검토 건과의 연결을 요구하도록 제안한다. 자격 미달 확인도 이유와 근거를 기록한다.

`AwardCancellationResult`: `cancellationId:UUID`, `canceledDrawResultId:UUID`, `replacementDrawRunId:UUID`, `status:PREPARING/READY/RUNNING/CONFIRMED/FAILED`, `canceledAt:instant`.

AD05는 관리자의 자격 미달·부정 확인 후 취소와 재추첨 실행 생성을 연결한다. 새 실행은 즉시 시작하고 진행 결과는 AD02에서 확인한다. 중간 장애나 재시도에도 취소 한 건이 여러 재추첨 실행으로 늘어나지 않는다. **한 요청은 당첨 한 건**을 대상으로 제안한다.

유지되는 당첨자와 경품은 건드리지 않고 빈 자리만 보충한다. 후보는 최초 유효 낙첨자에서 이후 제외 확정자 등을 제거하며, 기존 가중치와 응모 자격을 유지한다. 임의 재추첨·새 후보 모집·낙첨자 부정만으로 전체 재추첨하는 API는 없다.

## 공개 명단 반영

`PublicationUpdateResult`: `eventId:UUID`, `publicationId:UUID`, `drawRunId:UUID`, `revision:int`, `publishedAt:instant`, `updatedAt:instant`, `updatedBy:UUID`.

`PublicListChange`: `id:UUID`, `publicationId:UUID`, `revision:int`, `drawRunId:UUID`, `reason:string`, `confirmedBy:UUID`, `confirmedAt:instant`, `updatedAt:instant`, `changes:{prizeId:UUID,slotNumber:int,beforeUserId:UUID?,afterUserId:UUID?}[]`.

AD06은 **최초 발표 이후의 재추첨 결과 확인 및 공개 명단 갱신**용이다. 최초 추첨/발표 승인 API로 사용하지 않는다. 미확정 결과·취소 이벤트에는 사용할 수 없다. 동일 실행의 반영 재시도는 revision과 변경 알림을 중복 생성하지 않아야 한다. 여러 빈자리의 재추첨이 연달아 처리되어도 현재 공개 명단 전체의 유효성을 유지해야 한다.

최초 발표 전의 당첨 취소·재추첨은 예정 시각 전 공개할 수 없다. 해당 시점의 확인·자동 발표와 취소 결과 표시 연결은 담당자 계약에 따른다. 관리자 미확인 결과는 일반 사용자·본인 결과·알림 어느 곳에도 노출하지 않는다. 공개 갱신 이후 취소자·새 당첨자만 결과 변경 안내를 받고 다른 응모자에게 전체 재발표 알림을 보내지 않는다.

[drawing]: ../02-domain/drawing.md
