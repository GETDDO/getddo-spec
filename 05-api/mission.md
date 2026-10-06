# 미션 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [미션 규칙](../02-domain/mission.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

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

## 관리자 미션·문항

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AM01 | `GET /admin/missions` | `page,size,missionType,status,keyword,from,to` | 200 `Page<AdminMission>` | 공통 |
| AM02 | `GET /admin/missions/{missionId}` | 없음 | 200 `AdminMission` | 공통 |
| AM03 | `POST /admin/missions` | `MissionWrite` | 201 `AdminMission` | 400 문항·정답·기간·수량 오류 |

`MissionWrite`: 필수 `title:string(1~200)`, `description:string`, `missionType:SURVEY/QUIZ`, `startsAt:instant`, `endsAt:instant`, `rewardTicketCount:int(≥1)`, `questions:AdminQuestionWrite[]`; 선택 `imageKey:string/null(≤500)`.

`AdminQuestionWrite`: 필수 `questionType`, `questionText:string`, `displayOrder:int`; 설문에는 `required:boolean`; 선택형에는 `options:AdminOptionWrite[]`; 단답 퀴즈에는 `correctAnswer:string`. `AdminOptionWrite`: `optionText:string`, `displayOrder:int`, 퀴즈에만 `isCorrect:boolean`.

서버가 미션·문항·선택지·보상 정책의 ID를 생성하고 함께 저장한다. 퀴즈와 설문의 허용 타입은 [미션 문항](#미션문항)을 따른다. 선택형 퀴즈의 정답 개수, OX 표현, 단답 문자열 정규화와 다문항 완료 조건은 **미션 완료·채점 세부 계약**이다. 필수 여부와 정답 여부를 같은 개념으로 사용하지 않는다. 보상 수량 범위는 보상 수량·예약 정책 계약이다.

`AdminMission`: `id:UUID`, MissionWrite 전체, `status:DRAFT/ACTIVE/ENDED`, `rewardPolicyId:UUID`, `createdBy:UUID`, `createdAt:instant`, `updatedAt:instant`; `questions`와 `options`에는 각각 생성된 `id:UUID`를 추가한다.

운영 중인 미션의 문항·조건·보상은 수정하지 않는다. 변경은 새 미션 AM03으로 등록한다. 시작 전 초안 수정·공개·종료·삭제의 허용 범위와 요청 계약은 **미션 상태 운영 계약**에 따른다.

[mission]: ../02-domain/mission.md

## 등급 응모권 정책 반영 후속 — 2026-10-06

게임·미션 보상은 [등급 응모권 규칙](../02-domain/ticket.md#게임미션-보상의-등급-추첨)을 따른다. 기존 DTO 표는 등급별 보유량·지급 결과·가중치를 표현하지 못하는 부분이 있어 구현 전 담당자와 갱신해야 한다. 게임·미션 보상의 실제 지급 장수는 1이며 가중치 값을 `ticketCount`나 `quantity`에 넣지 않는다. 미션의 보상 수량을 임의 설정하는 종전 요청 초안도 새 정책에 맞춰 재검토한다. 이번 정책 문서 수정만으로 새로운 공개 필드·차감 등급 선택 계약을 확정하지 않는다. 미결정 항목은 [등급 응모권 후속 사항](../00-requirements/pending-decisions.md#등급-응모권)을 따른다.
