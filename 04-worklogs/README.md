# 공용 명세 작업 기록

공용 요구사항과 도메인 정책을 검토하고 정리한 과정, 확인된 내용과 다음 할 일을 작업별로 기록합니다.

## 관리 범위

- 공용 요구사항 검토 및 변경
- 도메인 용어와 정책 정리
- 프론트엔드와 백엔드가 함께 따르는 공용 문서와 협업 규칙
- 공용 정책의 미결정 사항과 후속 작업
- 명세 변경 과정과 프론트엔드·백엔드 영향 범위

## 관리하지 않는 내용

- 백엔드 내부 구조와 구현 과정
- 데이터베이스, 성능, 테스트, 빌드, 배포 등 백엔드 기술 작업
- 특정 코드 저장소에만 적용되는 기술적 결정

백엔드 기술 작업은 [GETDDO-BE의 `docs/05-worklogs`](https://github.com/GETDDO/getddo-be/tree/dev/docs/05-worklogs)에 기록합니다.

## 작성 규칙

- 작업 하나당 파일 하나를 작성합니다.
- 파일명은 `YYYY-MM-DD-kebab-case.md` 형식을 사용합니다.
- 공용 명세에 관한 작업만 기록하고 특정 코드 저장소의 구현 작업은 해당 저장소에서 관리합니다.
- 확인된 사실, 확정된 정책, 미결정 사항과 다음 할 일을 구분합니다.
- 확정된 정책은 해당 요구사항 또는 도메인 문서에 반영하고 작업 기록에는 링크만 남깁니다.
- 중요한 결정과 근거는 ADR로 분리합니다.
- 미확정 정책은 `00-requirements/pending-decisions.md`에서 관리합니다.
- 완료된 작업에는 실제 반영 문서, 관련 ADR과 Pull Request를 연결합니다.

작업 기록은 진행 과정과 맥락을 보존하기 위한 문서입니다. 확정된 요구사항이나 도메인 정책의 기준 문서를 대신하지 않습니다.

## 상태

- `진행 중`: 현재 작업하고 있습니다.
- `검토 대기`: 작업은 끝났으며 담당자 확인을 기다립니다.
- `완료`: 검토와 반영이 끝났습니다.
- `중단`: 작업을 계속하지 않으며 사유가 기록되어 있습니다.

## 기록 목록

- [2026-10-16 이벤트 검색·입력 상한·취소 재요청 계약 최신화](2026-10-16-event-query-validation-contract.md) — 검토 대기

- [2026-10-16 사용자 문맥 오류와 내 정보 비활성 거절 명세 정렬](2026-10-16-user-context-errors.md) — 검토 대기

- [2026-10-14 이벤트 연결 배너 수동 삭제 정책 최신화](2026-10-14-manual-banner-deletion.md)

- [2026-10-12 게임별 최소 보상 점수 반영](2026-10-12-game-reward-minimum-score.md)
- [2026-10-12 사용자 문맥 헤더 `X-User-Role` 충돌 정리와 결정](2026-10-12-user-context-header-decision.md)

- [2026-10-12 글자색깔 맞추기 연동 계약 채택 반영](2026-10-12-color-word-api-adoption.md)

- [2026-10-12 글자색깔 맞추기 API 연동 초안](2026-10-12-color-word-api-draft.md)

- [2026-10-12 글자색깔 맞추기 플레이 규칙 확정 반영](2026-10-12-color-word-rules.md)

- [2026-10-12 알림 자동 재시도·관리자 실패 복구 정책 명확화](2026-10-12-notification-failure-recovery.md)

- [2026-10-08 관리자 이벤트 응답의 createdBy 제거](2026-10-08-remove-event-created-by.md)

- [2026-10-07 응모권 회수 기능 제외](2026-10-07-exclude-ticket-recovery.md)

- [2026-10-07 출석 브론즈 응모권 보상 반영](2026-10-07-attendance-bronze-tickets.md)

- [2026-10-06 이벤트 중단·재개 기능 제외](2026-10-06-remove-event-suspension.md)

- [2026-10-06 전민규 게임 담당 범위 정리](2026-10-06-game-ownership-handoff.md)
- [2026-10-06 게임·미션 무작위 등급 응모권 정책 반영](2026-10-06-random-ticket-grades.md)

- [2026-09-30 PR 제목의 Jira 이슈 키 형식 정리](2026-09-30-pr-title-jira-key.md)

- [2026-09-30 사용자 알림 N01~N03 API 계약 확정](2026-09-30-notification-api-contract.md)

- [2026-09-29 응모권 보상 수량 하한 확정](2026-09-29-reward-quantity-minimum.md)
- [2026-09-29 추첨 가중치 정책 확정 설계](2026-09-29-drawing-weight-policy.md)

- [2026-09-29 Jira 에픽·태스크 템플릿 추가](2026-09-29-jira-templates.md)
- [2026-09-28 API 초안 도메인별 분리](2026-09-28-api-spec-split.md)

- [2026-09-23 이벤트 등록·경품 구성·수정 제한 반영](2026-09-23-event-registration-and-editing.md)

- [2026-09-23 이벤트 일정 변경·삭제 알림 정책 반영](2026-09-23-event-notification-policy.md)

- [2026-09-22 서버 UTC·사용자 및 관리자 화면 KST 기준 명시](2026-09-22-timezone-display.md)

- [2026-09-22 응모권 최대 5장 사용 상한 반영](2026-09-22-five-ticket-entry-limit.md)

- [2026-09-22 응모 마감 후 5분 자동 발표 요구사항 변경](2026-09-22-five-minute-auto-publication.md)

- [2026-09-21 이벤트 삭제 시 연결 배너 처리 확정](2026-09-21-event-banner-deletion.md)
- [2026-09-17 핵심 기능 용어 통일 및 확정 표현 반영](2026-09-17-requirements-refinement.md)
- [2026-09-21 출석·미션·게임 도메인 규칙 정립](2026-09-21-reward-domain-rules.md)

- [2026-09-21 서비스 공통 미션 및 반복 달성 최신화](2026-09-21-common-repeatable-missions.md)

- [2026-09-21 미결정 사항 순차 확정](2026-09-21-pending-decisions.md)
- [최초 추첨 실패 재개 API의 확인된 계약 반영](2026-10-12-initial-draw-resume-api.md) — 검토 대기

- [2026-10-12 개별 당첨 취소·재추첨 승인 API 반영](2026-10-12-award-cancellation-api.md)
- [2026-10-16 타꼬런·타꼬 타이밍·타꼬 주문 검증 계약 초안](2026-10-16-game-validation-drafts.md) — 검토 대기

- [2026-10-16 추첨 공개 계약과 GD-199 PR 동기화](2026-10-16-drawing-api-pr-sync.md) — 검토 대기
