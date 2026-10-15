# API 계약

- 상태: N01~N03 확정 / 2026-10-11 추첨 제외 폐지 정책 변경 검토 완료 / 나머지 상세 계약 검토 대기 — 담당자 확인 전
- 기준: [확정 요구사항](../00-requirements/functional-requirements.md), [구현 범위](../00-requirements/scope.md), [도메인 규칙](../02-domain/README.md)

프론트엔드와 백엔드가 함께 사용하는 HTTP 요청·응답 계약이다. [사용자 알림 N01~N03](notification.md#사용자-알림--n01n03-확정)은 2026-09-30 사용자 지시로 현재 구현 기준 계약을 확정했다. [관리자 알림 AN01~AN05](notification.md#관리자-알림-운영--an01an05-확정)는 2026-10-12 사용자 추천안 승인으로 별도 확정했다. 그 밖에 개별 확정이 표시되지 않은 경로·메서드·DTO·업무 오류 코드는 담당자 확인 전 제안이며, 확정된 요구사항·도메인 규칙과 충돌하면 해당 기준 문서를 우선한다. 별도 기재가 없으면 경로는 [공통 계약](common.md)의 `/api/v1` 접두사를 생략한다.

| 파일 | API ID | 내용 |
| --- | --- | --- |
| [공통](common.md) | — | 권한·응답·페이징·오류·사용자 문맥·중복 요청 |
| [사용자](user.md) | U01, AO01~AO02 | 내 정보·관리자 사용자 조회 |
| [이벤트](event.md) | E01~E02, AE01~AE05, AE08 | 이벤트 조회·등록·취소 |
| [응모](entry.md) | E03~E07, AO06~AO07 | 응모·자격·응모 이력 |
| [추첨·결과](drawing.md) | E08~E09, AD01~AD09 | 공개 결과·재추첨·명단 갱신 |
| [출석](attendance.md) | AT01~AT03, AP01~AP04 | 출석·단계 보상·운영 정책 |
| [미션](mission.md) | M01~M04, AM01~AM03 | 미션 제출·운영 |
| [게임](game.md) | G01~G05, AG01~AG03 | 플레이·점수·게임 운영 |
| [응모권](ticket.md) | T01~T02, AO03~AO05, AO08 | 보유 조회·이력·운영 정책 |
| [어뷰징](abuse.md) | AR01~AR02, AR07 (AR03 제외, AR06 후속 검토, AR04·AR05 제외) | 차단 내역 조회·사후 무효화 후속 |
| [알림](notification.md) | N01~N03, AN01~AN05 | 사용자 알림·관리자 발송 이력 |
| [배너](banner.md) | B01, AB01~AB05 | 배너 조회·관리 |
| [감사 기록](audit.md) | 외부 API 없음 | 내부 기록 유지·공통 조회 제외 |

DB 대응, 저장 방식, 내부 처리와 구현 검증 항목은 [백엔드 API 문서](https://github.com/GETDDO/getddo-be/tree/dev/docs/02-api)와 [DB 문서](https://github.com/GETDDO/getddo-be/tree/dev/docs/03-database)에서 관리한다. 담당자 판단이 필요한 도메인 정책은 [미결정 사항](../00-requirements/pending-decisions.md)을 따른다.
