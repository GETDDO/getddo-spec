# 출석 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 도메인 규칙: [출석 규칙](../02-domain/attendance.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

근거: [출석 규칙][attendance]. 권한 U, 주 담당 윤태형.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AT01 | `GET /attendances/today` | 없음 | 200 `AttendanceToday` | 공통 |
| AT02 | `POST /attendances` | 본문 없음 | 201 `AttendanceReceipt`; 같은 날 재요청 200 | 429 |
| AT03 | `GET /attendances` | 필수 `month` | 200 `AttendanceMonth` | 공통 |

| DTO | 필드 |
| --- | --- |
| `AttendanceToday` | `attendanceDate:date`, `attended:boolean`, `consecutiveDays:int`, `dailyRewardTicketCount:int`, `milestones:AttendanceMilestone[]`, `nextResetAt:instant`, `serverTime:instant` |
| `AttendanceMilestone` | `milestoneDays:int`, `rewardTicketCount:int`, `claimed:boolean`, `claimedAt:instant?` |
| `AttendanceReceipt` | `attendanceId:UUID`, `attendanceDate:date`, `consecutiveDays:int`, `rewards:RewardReceipt[]`, `createdAt:instant` |
| `RewardReceipt` | `claimId:UUID`, `ticketCount:int`, `grantedAt:instant?`, `expiresAt:instant?` |
| `AttendanceMonth` | `month:month`, `attendanceDates:date[]`, `milestones:AttendanceMilestone[]`, `serverTime:instant` |

AT02는 날짜나 보상량을 입력받지 않는다. 서버 KST 업무일로 출석과 일일·단계 보상을 함께 반영한다. 반복 응답의 `rewards`는 해당 출석에서 이미 확정한 보상이며 이번 호출에서 다시 지급했다는 뜻이 아니다. 보상이 0장이면 지급 원장 없이 보상 기록만 존재할 수 있어 `grantedAt=null`, `expiresAt=null`을 제안한다. 일일 보상 0장 허용 여부는 보상 수량·예약 정책 계약을 따른다.

매월 단계 보상은 각 단계당 1회다. 월중 연속이 끊겨도 이미 받은 단계 보상은 다시 지급하지 않는다. 초기 단계는 서버 정책에서 조회한다. 사용자 답변에 따라 `consecutiveDays`는 **같은 달의 실제 연속 일수(29~31일 포함)**를 반환한다. 단계 설정의 최대 28일과 실제 일수는 구분한다.

## 관리자 출석 정책

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AP01 | `GET /admin/policies/attendance/daily` | `page,size` | 200 `Page<DailyAttendancePolicy>` | 공통 |
| AP02 | `POST /admin/policies/attendance/daily` | 필수 `rewardTicketCount:int(≥1)` | 201 `DailyAttendancePolicy` | 400 수량 오류, 409 적용일 중복; 최대값·예약 정책 재수정은 보상 수량·예약 정책 계약 |
| AP03 | `GET /admin/policies/attendance/streak` | `page,size` | 200 `Page<StreakPolicySet>` | 공통 |
| AP04 | `POST /admin/policies/attendance/streak` | 필수 `milestones:MilestoneWrite[]` | 201 `StreakPolicySet` | 400 단계 오류, 409 적용월 중복 |

`DailyAttendancePolicy`: `id:UUID`, `rewardTicketCount:int`, `effectiveFrom:instant`, `effectiveUntil:instant?`, `createdBy:UUID`, `createdAt:instant`.

`MilestoneWrite`: `milestoneDays:int(1~28)`, `rewardTicketCount:int(≥1)`. 배열은 단계 일수 오름차순이고 중복이 없어야 한다. `StreakPolicySet`: `id:UUID`, `effectiveMonth:month`, `milestones:{id:UUID,milestoneDays:int,rewardTicketCount:int}[]`, `createdBy:UUID`, `createdAt:instant`.

일일 정책의 적용 시각은 다음 KST 자정, 연속 정책은 다음 KST 월초를 서버가 계산한다. 사용자가 과거 적용일을 지정하지 못한다. 같은 적용일/월의 예약 정책 재수정 방식은 **보상 수량·예약 정책 계약**이며 이 초안은 중복 생성 충돌을 제안한다. 현재 월의 판정·기존 보상 기록은 유지한다. 보상 1장 고정인 게임에 임의 수량 변경 API를 만들지 않는다.

[attendance]: ../02-domain/attendance.md
