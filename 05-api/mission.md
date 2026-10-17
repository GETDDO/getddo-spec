# 미션 API

- 상태: 사용자 API(M01~M04) 구현 반영, 관리자 미션(AM01~AM03) 구현(백엔드 GD-177), 설문 응답 집계(AM04) 제안(백엔드 GD-223)
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [미션 규칙](../02-domain/mission.md)

사용자 API는 구현 결과를 반영했다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

근거: [미션 규칙][mission]. 조회 R, 제출·개인 내역 U. 주 담당 윤태형.

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| M01 | R | `GET /missions` | `page,size,missionType` | 200 `Page<MissionSummary>` | 공통 |
| M02 | R | `GET /missions/{missionId}` | 없음 | 200 `MissionDetail` | 공통 |
| M03 | U | `POST /missions/{missionId}/submissions` | 멱등 헤더, `MissionSubmissionRequest` | 201 `MissionSubmissionResult`; 동일 요청 200 | 409 기간·키 충돌, 429 |
| M04 | U | `GET /missions/{missionId}/submissions/me` | `page,size` | 200 `Page<MissionSubmissionResult>` | 공통 |

## 미션·문항

| DTO | 필드 |
| --- | --- |
| `MissionSummary` | `id:UUID`, `title:string`, `imageUrl:string?`, `missionType:SURVEY/QUIZ`, `startsAt:instant`, `endsAt:instant`, `rewardTicketCount:int`, `completed:boolean`, `serverTime:instant` |
| `MissionDetail` | MissionSummary 전체 + `description:string`, `questions:MissionQuestion[]` |
| `MissionQuestion` | `id:UUID`, `questionType:OX/SINGLE_CHOICE/SHORT_ANSWER/FREE_TEXT`, `questionText:string`, `required:boolean`, `displayOrder:int`, `options:QuestionOption[]` |
| `QuestionOption` | `id:UUID`, `optionText:string`, `displayOrder:int` |

제안하는 문항 유형은 퀴즈의 경우 OX·단일 선택·단답, 설문의 경우 단일 선택·자유 서술을 표현한다. 실제 다문항 완료·채점 방식은 담당자 계약에 따른다. 퀴즈의 `required`는 완료 조건 확정 후 산출한다. 관리자용 정답 필드 `correctAnswer`, `isCorrect`는 사용자 문항 DTO에서 제거한다. 공개 가능한 미션 상태와 DRAFT 노출 범위는 담당자 계약에 따른다.

## 제출

`MissionSubmissionRequest`: 필수 `answers:MissionAnswer[]`.

`MissionAnswer`: 필수 `questionId:UUID`, 선택 `selectedOptionId:UUID`, 선택 `answerText:string`. 선택형은 option만, 서술형은 text만 보내도록 제안한다. 미션·문항·선택지 소속과 중복 문항을 검증한다. 필수 문항 누락은 400이다.

```json
{
  "answers": [
    {
      "questionId": "0199abcd-1234-7000-8000-000000000021",
      "selectedOptionId": "0199abcd-1234-7000-8000-000000000022"
    }
  ]
}
```

`MissionSubmissionResult`: `submissionId:UUID`, `missionId:UUID`, `receivedAt:instant`, `isCompleted:boolean`, `reward:RewardReceipt?`, `createdAt:instant`.

서버 도착 시각이 `[startsAt,endsAt)`이면 종료 후 검증이 끝나도 인정한다. 응모의 접수 확정 시각 기준과 구분한다. 정답을 맞히지 못한 유효 제출은 정상 제출 결과 `isCompleted=false`, `reward=null`이며 다시 시도할 수 있다. 문항별 정답·해설·오답 피드백 범위는 미션 완료·채점 세부 계약에서 정하며 정답 자체는 응답하지 않는다.

완료 후 재제출은 기존 완료 결과를 반환하는 방식으로 제안한다. 이미 완료된 설문 답변을 덮어쓰거나 추가 보상하지 않는다. 같은 미션을 반복 보상하는 주기·재달성 API는 없다.

**현행 구현 범위(백엔드 GD-175, M03)**:

- 제출 ID는 `Idempotency-Key` 헤더의 UUID이며 `MissionSubmissionResult.submissionId`와 같다. 같은 키로 같은 요청이 다시 오면 저장된 결과를 200으로 돌려주고(마감 뒤에도 같다), 처음 요청과 다른 본문이면 409 `MISSION-005`다. 본문이 같은지는 처음 요청(사용자·미션·답변 구성, 답변 순서 무관)의 해시로 판정하므로 처음 요청에 없던 답변을 더하거나 값을 바꾸면 이미 정답으로 완료한 문항이어도 409이고, 처음 요청 그대로 재전송하면 저장하지 않은 답변이 있어도 200이다. 같은 문항이 두 번 있는 요청은 400 `MISSION-002`다. 해시는 답변을 문항 ID 순으로 정렬해 선택지와 입력 문자열을 원문 그대로 반영하고 단답형 채점의 정규화를 적용하지 않는다. 입력 문자열이 비었거나 공백뿐이면 보내지 않은 것으로 보지만 그 밖의 공백·대소문자 차이는 다른 요청이므로(`"겟또"`와 `"겟또 "`는 채점에서는 같은 답이어도 같은 키로 보내면 409) 재시도는 같은 본문으로 보낸다. 새로 접수하면 201이다.
- 퀴즈는 **문항별 정답을 누적**한다. 한 번에 모든 문항을 보낼 필요가 없고, 이전 제출에서 이미 정답으로 완료한 문항에 다시 보낸 답변은 채점·저장하지 않고 무시한다. 남은 문항을 모두 맞히면 `isCompleted=true`가 된다. 보낸 답변이 모두 이미 정답으로 완료한 문항이어서 처리할 답변이 없으면 400 `MISSION-002`다. 오답이 있는 유효한 제출은 201, `isCompleted=false`, `reward=null`이다.
- 설문은 모든 필수 문항에 답해야 하며 필수 문항이 비면 400 `MISSION-002`이고 아무것도 저장하지 않는다. 선택 문항은 비워 둘 수 있다.
- 선택형은 `selectedOptionId`만, 서술형은 `answerText`만 보낸다. 공백뿐인 `answerText`는 보내지 않은 것으로 본다. 단답형 채점은 유니코드 정규화(NFC)·앞뒤 공백 제거·연속 공백 축약·대소문자 무시 후 비교하며 그 밖의 띄어쓰기·철자 차이는 오답이다(구현 제안). 선택형 문항의 정답 개수와 OX 표현은 강제하지 않고 정답으로 표시한 선택지를 고르면 정답이다.
- 서버가 제출을 받은 시각이 `[startsAt, endsAt)`이고 미션이 운영 중이 아니면 409 `MISSION-004`, 없는 미션은 404 `MISSION-003`이다. 이미 완료한 미션을 새 키로 다시 제출하면 기간과 관계없이 먼저 완료한 제출의 결과(그 제출의 `submissionId`·보상 포함)를 200으로 돌려주고 추가 보상하지 않는다.
- `MissionSubmissionResult.reward`는 완료 때만 있고 응모권 서비스가 확정한 값이다: `claimId`(보상 청구 ID, 공용 `RewardReceipt`의 `claimId`와 같다), `ticketCount`(1), `grade`(`BRONZE`·`SILVER`·`GOLD`), `grantedAt`, `expiresAt`. 완료 보상은 사용자·미션당 한 번이며, 제출·정답 진행·보상 청구·응모권 지급은 한 트랜잭션으로 확정되거나 모두 취소된다. `createdAt`은 서버가 제출을 받은 시각으로 `receivedAt`과 같다.
- 요청 빈도 제한(429)은 GD-146으로 미션 제출(M03)에 적용되어 있다([공통 API 계약](common.md)의 429 참고). 사용자 문항 응답에서 정답 필드를 제외하는 것은 조회 API(M02) 구현 때 다룬다.

**현행 구현 범위(백엔드 GD-174, M01·M02)**:

- M01은 `page`(1부터, 기본 1)·`size`(1~100, 기본 20)·`missionType`을 받는다. 운영 상태나 기간으로 거르지 않고 `ACTIVE`·`ENDED` 미션을 모두 시작 시각 내림차순(같으면 ID 내림차순)으로 돌려주므로 진행 여부는 `startsAt`·`endsAt`로 판단한다. `DRAFT`는 스키마에 없다. `completed`는 요청한 사용자의 완료 제출이 있는지로 정한다.
- `rewardTicketCount`는 미션 보상이 항상 응모권 1장이라 1로 고정이며 지급 등급은 완료할 때 정해진다. `imageUrl`은 이미지 저장소가 정해지기 전이라 현재 항상 null이다.
- M02는 `MissionSummary` 필드에 `description`과 `questions`를 더한다. 문항은 퀴즈와 설문을 같은 모양으로 `displayOrder` 오름차순이며, 퀴즈 문항의 `required`는 모든 문항을 맞혀야 완료하므로 항상 true이다. 사용자 응답에는 `correctAnswer`와 선택지의 정답 여부를 싣지 않는다. 없는 미션은 404 `MISSION-003`이다.
- 페이지 번호나 크기가 범위를 벗어나거나 `missionType`이 알 수 없는 값이면 400이다.

**현행 구현 범위(백엔드 GD-176, M04)**:

- M04는 `page`(1부터, 기본 1)·`size`(1~100, 기본 20)를 받아 요청 사용자가 해당 미션에 낸 제출을 최신순(`createdAt` 내림차순, 같으면 ID 내림차순)으로 돌려준다. 없는 미션은 404 `MISSION-003`이고 제출이 없으면 빈 페이지다.
- 항목은 M03의 `MissionSubmissionResult`와 같은 필드이며 `receivedAt`과 `createdAt`은 같다. 오답 제출도 `isCompleted=false`, `reward=null`인 정상 결과로 포함하고, 완료한 제출에만 응모권 서비스가 확정한 `reward`가 붙는다. 제출한 답변과 정답은 응답에 싣지 않는다.
- 페이지 번호나 크기가 범위를 벗어나면 400이다.

## 관리자 계정 호출 (현행 구현)

위 권한 U는 일반 사용자용이다. 현재 구현은 관리자 계정이 미션 조회·제출 API를 호출해도 차단하지 않는다(응모 E04만 관리자를 거절한다). 관리자 호출을 차단할지는 후속 결정이다.

## 관리자 미션·문항

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AM01 | `GET /admin/missions` | `page,size,missionType,status,keyword,from,to` | 200 `Page<AdminMission>` | 공통 |
| AM02 | `GET /admin/missions/{missionId}` | 없음 | 200 `AdminMission` | 공통 |
| AM03 | `POST /admin/missions` | `MissionWrite` | 201 `AdminMission` | 400 문항·정답·기간·수량 오류 |
| AM04 | `GET /admin/missions/{missionId}/survey-summary` | 필수 `from`,`to`(KST 날짜) | 200 `SurveySummary` | 400 설문이 아닌 미션·기간 오류, 404 `MISSION-003` |

`MissionWrite`: 필수 `title:string(1~200)`, `description:string`, `missionType:SURVEY/QUIZ`, `startsAt:instant`, `endsAt:instant`, `rewardTicketCount:int(≥1)`, `questions:AdminQuestionWrite[]`; 선택 `imageKey:string/null(≤500)`.

`AdminQuestionWrite`: 필수 `questionType`, `questionText:string`, `displayOrder:int`; 설문에는 `required:boolean`; 선택형에는 `options:AdminOptionWrite[]`; 단답 퀴즈에는 `correctAnswer:string`. `AdminOptionWrite`: `optionText:string`, `displayOrder:int`, 퀴즈에만 `isCorrect:boolean`.

서버가 미션·문항·선택지·보상 정책의 ID를 생성하고 함께 저장한다. 퀴즈와 설문의 허용 타입은 [미션 문항](#미션문항)을 따른다. 선택형 퀴즈의 정답 개수, OX 표현, 단답 문자열 정규화와 다문항 완료 조건은 **미션 완료·채점 세부 계약**이다. 필수 여부와 정답 여부를 같은 개념으로 사용하지 않는다. 보상 수량 범위는 보상 수량·예약 정책 계약이다.

`AdminMission`: `id:UUID`, MissionWrite 전체, `status:DRAFT/ACTIVE/ENDED`, `rewardPolicyId:UUID`, `createdBy:UUID`, `createdAt:instant`, `updatedAt:instant`; `questions`와 `options`에는 각각 생성된 `id:UUID`를 추가한다.

**현행 구현 범위(백엔드 GD-177)**: 세 경로는 활성 관리자만 호출할 수 있고 아니면 403 `MISSION-006`이다. 미션 상태는 `ACTIVE`/`ENDED`뿐이며 `DRAFT`는 없다. AM03으로 등록한 미션은 바로 `ACTIVE`이고 제출 가능 기간은 `startsAt`·`endsAt`이 정한다. 미션 완료 보상은 등급 응모권 1장으로 고정이라 `rewardTicketCount`는 요청에서 생략하거나 1만 보낼 수 있고(다른 값은 400 `MISSION-001`) 응답 값은 항상 1이며, 서버가 미션마다 수량 1의 보상 정책을 만들어 `rewardPolicyId`로 연결한다. AM01은 운영 시작 시각 내림차순(같으면 `id` 내림차순)이고 `from`·`to`는 `startsAt`이 `[from, to)`인 미션을 거르며, 목록의 `AdminMission`에는 `questions`가 없다. 퀴즈 문항에는 `required`가, 설문 문항에는 `correctAnswer`·`isCorrect`가 없다. 선택형 퀴즈(OX·단일 선택)는 선택지 2개 이상에 정답이 정확히 하나여야 하고 어기면 400 `MISSION-001`이다. 필수값 누락·길이 초과는 400 `COMMON-002`다. 없는 미션은 404 `MISSION-003`이다. 등록은 감사 기록(`MISSION_CREATE`)을 남긴다.

`SurveySummary`: `dailyResponses:{date:date,count:long}[]`, `questions:{questionId:UUID,questionText:string,options:{optionId:UUID,optionText:string,count:long}[]}[]`.

**AM04 설문 응답 집계(제안, 백엔드 GD-223)**: 프론트 요청 R5(관리자 대시보드의 설문 추이)에 따른 새 API이며 경로와 필드는 제안이라 확정 전이다. 활성 관리자만 호출할 수 있고(403 `MISSION-006`) 읽기 전용이다. `from`·`to`는 KST 날짜(`YYYY-MM-DD`, 양끝 포함)로 둘 다 필수이고 `to`가 `from` 이후이며 최대 31일이다(어기면 400 `COMMON-002`, 형식 오류는 `COMMON-005`). 완료된 설문 제출만 센다. `dailyResponses`는 완료된 제출 수를 제출 시각의 KST 날짜별로 담고 응답 없는 날도 `count: 0`으로 포함한다. `questions`는 단일 선택 문항만 담고(자유 서술은 제외) 문항과 선택지는 표시 순서 오름차순이며, 고른 사람이 없는 선택지도 `count: 0`으로 포함한다. 퀴즈 미션이면 400 `MISSION-001`, 없는 미션은 404 `MISSION-003`이다. 긍정/부정 묶음은 프론트가 선택지를 묶어 표시한다.

운영 중인 미션의 문항·조건·보상은 수정하지 않는다. 변경은 새 미션 AM03으로 등록한다. 시작 전 초안 수정·공개·종료·삭제의 허용 범위와 요청 계약은 **미션 상태 운영 계약**에 따른다.

[mission]: ../02-domain/mission.md

## 등급 응모권 정책 반영 후속 — 2026-10-06

게임·미션 보상은 [등급 응모권 규칙](../02-domain/ticket.md#게임미션-보상의-등급-추첨)을 따른다. 기존 DTO 표는 등급별 보유량·지급 결과·가중치를 표현하지 못하는 부분이 있어 구현 전 담당자와 갱신해야 한다. 게임·미션 보상의 실제 지급 장수는 1이며 가중치 값을 `ticketCount`나 `quantity`에 넣지 않는다. 사용자 제출 API(M03)는 이 규칙대로 구현되어 보상에 `claimId`, `ticketCount`(1), `grade`를 돌려준다. 미션의 보상 수량을 임의 설정하는 종전 요청 초안(관리자 AM03의 `rewardTicketCount`)은 구현 때 새 정책에 맞춰 조정한다. 이번 정책 문서 수정만으로 새로운 공개 필드·차감 등급 선택 계약을 확정하지 않는다. 미결정 항목은 [등급 응모권 후속 사항](../00-requirements/pending-decisions.md#등급-응모권)을 따른다.
