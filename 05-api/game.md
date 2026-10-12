# 게임 API 초안

- 상태: 색깔 게임 G03/G04 세부 계약 사용자 채택 (2026-10-12) / 그 외 상세 API 계약은 담당자 검토 대기
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [게임 규칙](../02-domain/game.md)

색깔 게임 세부 계약에서 채택한 범위를 제외한 경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

- [글자색깔 맞추기 연동 계약](game-color-word-draft.md): 2026-10-12 사용자 채택. 색깔 게임의 G03에는 `rules`·`gameData`가 추가되며, G04의 `validationData`·등급 보상 응답·시간·재시도는 해당 계약을 따른다. 다른 게임과 관리자 API의 상세 계약을 일괄 확정하지 않는다.

근거: [게임 규칙][game]. 조회 R, 플레이·개인 통계 U. 게임 사용자·관리자 API의 주 담당은 전민규다. 게임 규칙·서버 결과 판정·점수와 통계·게임별 일일 보상 자격을 처리한다. 실제 등급 응모권 지급·지갑·원장은 윤태형의 응모권 기능과 연동하고, 지급 전 검증의 차단 기록·조회는 석종수의 어뷰징 기능과 연동한다. 지급 후 부정 검토·확정 계약은 후속 확인 전까지 기존 정책을 유지한다.

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| G01 | R | `GET /games` | 없음 | 200 `GameSummary[]` | 공통 |
| G02 | R | `GET /games/{gameId}` | 없음 | 200 `GameDetail` | 공통 |
| G03 | U | `POST /games/{gameId}/plays` | 본문 없음 | 201 `GamePlayStart` | 409 이용 불가 |
| G04 | U | `PUT /games/{gameId}/plays/{playId}/result` | `GameResultRequest` | 200 `GamePlayResult`; 처리 중 202 동일 타입 | 409 플레이/규칙 충돌, 429 |
| G05 | U | `GET /games/{gameId}/statistics/me` | 없음 | 200 `MyGameStatistics` | 공통 |

| DTO | 필드 |
| --- | --- |
| `GameSummary` | `id:UUID`, `code:string(≤50)`, `name:string(≤100)`, `description:string?` |
| `GameDetail` | GameSummary 전체 + `ruleVersion:string(≤30)`, `rules:object`, `serverTime:instant` |
| `GamePlayStart` | `playId:UUID`, `playToken:string`, `ruleVersion:string`, `startedAt:instant` |
| `GameResultRequest` | 필수 `playToken:string`, `validationData:object` |
| `GamePlayResult` | `playId:UUID`, `status:PROCESSING/VALID/INVALID`, `score:long?`, `completedAt:instant?`, `reward:RewardReceipt?` |
| `MyGameStatistics` | `gameId:UUID`, `bestScore:long`, `totalScore:long`, `validPlayCount:long`, `rewardDate:date`, `rewardClaimedToday:boolean`, `nextResetAt:instant`, `serverTime:instant` |

`rules`의 공개 가능한 구조와 `validationData`의 게임별 스키마는 **게임별 검증·운영 계약**이다. 색깔 게임의 G03/G04는 연결된 세부 계약을 사용한다. 그 외 미확정 구조가 있는 G02~G04와 AG03은 구현 가능한 최종 계약이 아니다. 서버 점수 판정을 전제로 하며 `score`를 확정값으로 입력받지 않는다. 판정 내부 규칙을 공개 `rules`에 그대로 복사하지 않는다.

최소 보상 점수는 [게임 보상 규칙](../02-domain/game.md#보상-지급)을 따른다. 기준 미만의 정상 완료는 `status=VALID`, 서버 판정 점수, `reward=null`을 반환한다. 점수·통계는 반영하되 보상 청구와 실제 지급은 호출하지 않고 `rewardClaimedToday`도 새로 변경하지 않는다. 기준 이상이어도 이미 당일 보상을 받았다면 `reward=null`이다.

`PROCESSING`은 응답용 상태다. 처리 중 재시도는 같은 playId로 수행한다. 동일 플레이는 최초 확정 결과를 반환한다. 다른 플레이를 여러 번 해도 사용자·게임·KST 보상일당 1장만 지급한다. 의심만으로 유효 점수·보상을 보류하지 않으며, 확인된 부정의 점수 무효화는 검토 결정과 연결한다. 자정을 걸친 플레이의 보상일 판정 시점은 게임별 검증·운영 계약이다. 색깔 게임은 서버의 최초 유효 결과 확정 시각을 기준으로 한다.

## 관리자 게임 관리

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AG01 | `GET /admin/games` | `page,size,isActive,keyword` | 200 `Page<AdminGame>` | 공통 |
| AG02 | `GET /admin/games/{gameId}` | 없음 | 200 `AdminGame` | 공통 |
| AG03 | `POST /admin/games` | `GameWrite` | 201 `AdminGame` | 409 code 중복; 게임별 검증·운영 계약 선결 |

`GameWrite`: 필수 `code:string(1~50)`, `name:string(1~100)`, `ruleVersion:string(1~30)`, `rules:object`; 선택 `description:string/null`. `AdminGame`: `id:UUID`, GameWrite 전체 + `isActive:boolean`, `createdAt:instant`, `updatedAt:instant`.

신규 게임 등록용 외형만 제안하며 게임 종류·검증·rules 스키마 확정이 먼저다. 기존 규칙 변경, 게임 활성/비활성 전환 시 진행 중 플레이 처리, 게임 보상 조건 예약 변경은 **게임별 검증·운영 계약**에 남긴다. 일반 게임 CRUD로 기존 플레이의 규칙 버전과 점수를 소급 변경하지 않는다.

[game]: ../02-domain/game.md

## 등급 응모권 정책 반영 후속 — 2026-10-06

게임·미션 보상은 [등급 응모권 규칙](../02-domain/ticket.md#게임미션-보상의-등급-추첨)을 따른다. 기존 DTO 표는 등급별 보유량·지급 결과·가중치를 표현하지 못하는 부분이 있어 구현 전 담당자와 갱신해야 한다. 게임·미션 보상의 실제 지급 장수는 1이며 가중치 값을 `ticketCount`나 `quantity`에 넣지 않는다. 미션의 보상 수량을 임의 설정하는 종전 요청 초안도 새 정책에 맞춰 재검토한다. 이번 정책 문서 수정만으로 새로운 공개 필드·차감 등급 선택 계약을 확정하지 않는다. 미결정 항목은 [등급 응모권 후속 사항](../00-requirements/pending-decisions.md#등급-응모권)을 따른다.

## 지급 전 차단 전환

- 차단된 지급 요청에는 응모권을 지급하지 않고 관련 요청·사유·시각·검증 근거를 관리자 조회로 연결한다. 게임 담당의 결과 검증·점수 계약이나 공개 DTO를 이번 문서 변경으로 확정하지 않는다.
- 차단 적용 대상·판정 기준과 사후 점수 무효화 유지 여부는 [후속 확인](../00-requirements/pending-decisions.md#지급-전-차단-전환의-후속-확인)을 따른다.
