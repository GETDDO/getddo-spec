# 작업 기록: 출석 보상의 브론즈 응모권 등급 확정 반영

- 작성일: 2026-10-07
- 관련 이슈 / PR: 사용자 요청 / PR 미생성
- 상태: 검토 대기
- 영향 범위: 공통 / 프론트엔드 / 백엔드

## 목적

출석 보상에 브론즈 응모권을 지급하자는 사용자 요청을 공용 명세에 반영한다.

## 한 일

- 요구사항과 출석·응모권 규칙, 미결정 목록을 갱신했다.
- 출석 API 초안에 등급 표시·잔액 연결의 담당자 후속 검토를 기록했다.
- ADR-016을 추가하고 ADR-014에 후속 결정을 연결했다.

## 확인된 내용

- 로컬 HEAD와 원격 main은 변경 전 모두 e9e8a71이었다.
- 사용자 요청에 따른 정책은 [출석 규칙](../02-domain/attendance.md)을 원본으로 관리한다.
- 기존 지급 수량과 게임·미션 무작위 지급 정책은 유지한다.
- FE·BE 코드, 공개 API 필드와 ERD는 변경하지 않았다.

## 다음 할 일

- [ ] 담당자가 등급별 지급·잔액 계약과 출석 API 표시를 검토하고 구현에 연결한다.
- [ ] 기존 무등급 응모권의 전환·적용 시점과 나머지 등급 계약을 결정한다.

## 미결정·차단 사항

기존 지급분 전환·응모 차감 선택·반환 등급은 [미결정 사항](../00-requirements/pending-decisions.md#등급-응모권)에 유지한다.

## 반영 결과

- 변경한 명세: [요구사항](../00-requirements/functional-requirements.md), [출석 규칙](../02-domain/attendance.md), [응모권 규칙](../02-domain/ticket.md), [미결정 사항](../00-requirements/pending-decisions.md), [출석 API 초안](../05-api/attendance.md)
- 관련 ADR: [ADR-016](../03-decisions/016-attendance-bronze-tickets.md), [ADR-014 후속](../03-decisions/014-random-ticket-grades.md#후속-결정)
- 관련 PR: 미생성
