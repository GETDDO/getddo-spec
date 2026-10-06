# 작업 기록: 추첨 ERD 검토에 따른 API·최초 공개 정책 동기화

- 작성일: 2026-10-05
- 작성: Codex — 담당자 확인 전
- 관련 이슈 / PR: [GD-93](https://ureca4.atlassian.net/browse/GD-93) / [백엔드 PR #26](https://github.com/GETDDO/getddo-be/pull/26)
- 상태: 검토 대기
- 영향 범위: 공통 / 프론트엔드 / 백엔드

## 목적

백엔드 추첨 ERD 검토에서 사용자가 결정한 조회 표현과 최초 공개 예외 정책을 공용 원본에 반영하고 중복 API 문서를 정리한다.

## 한 일

- 추첨 API 초안을 사용자 2개·관리자 9개 구성으로 갱신했다. 기존 업무 ID를 유지하고 검증 이력 조회·실행 시작 및 재개에 AD08~AD09를 부여했다.
- 단일 이전 실행·실행자·사유 대신 취소별 원래 결과·실행 ID와 처리 근거 목록을 제공하고 공개 이력 여부와 취소 여부를 분리했다. 실패 문구는 코드로 구성하며 어뷰징 근거 연결은 보류했다.
- 최초 공개 전 재추첨은 확인된 최종 명단으로 공개하고 준비·확인이 늦으면 최초 발표를 지연하는 사용자 결정을 도메인·요구사항·미결정 사항에 반영했다. 정상 마감 + 5분 자동 공개는 유지한다.
- DB 관계·트랜잭션·잠금은 백엔드 문서에서 관리하고 공용 API에 내부 스키마를 중복 작성하지 않았다.

## 확인된 내용

- 기준 정책: [추첨 규칙](../02-domain/drawing.md), [요구사항](../00-requirements/functional-requirements.md). 이번 변경은 실제 구현 완료가 아니다.
- 작업 전 깨끗한 main에서 원격 main을 git pull --ff-only로 최신화했다. 백엔드의 기존 미커밋 변경은 보존했다.
- 문서 갱신 후 두 저장소의 `git diff --check`, 갱신 문서의 로컬 파일 링크와 AD01~AD09 ID 중복·누락 검사를 통과했다. 백엔드 SQL은 주석·COMMENT를 제외한 정의가 HEAD와 동일하며 12개 테이블·20개 FK가 유지된다. 문서 변경으로 Gradle 테스트·빌드는 실행하지 않았다.
- 백엔드 공개 SQL의 SQLite 논리 샘플 검토에서 공개 버전 하나가 최초·재추첨 결과를 함께 참조하는 설계 보완이 필요함을 확인했다. 실제 MySQL 검증은 아니다.

## 다음 할 일

- [ ] 최초 공개의 복수 실행·결과 연결과 재추첨별 관리자 확인 이력 보존을 백엔드에서 설계한다.
- [ ] 최초 공개 전 확인 API·지연 안내·감사 및 알림 연동을 관련 담당과 확인한다.
- [ ] 실제 MySQL·동시성·복구·성능 검증을 수행한다.

## 미결정·차단 사항

공개 후 취소자 임시 표시, 어뷰징 근거 연결, 지연 안내와 장애 복구 세부사항은 [후속 사항](../00-requirements/pending-decisions.md)을 따른다. 정한 정책을 다시 미결정으로 취급하지 않는다.

## 반영 결과

- 변경한 명세: [API](../05-api/drawing.md), [API 목록](../05-api/README.md), [추첨 규칙](../02-domain/drawing.md), [요구사항](../00-requirements/functional-requirements.md), [후속 사항](../00-requirements/pending-decisions.md)
- 관련 ADR: 백엔드 docs/04-decisions/0003-drawing-snapshots-and-publications.md (제안)
- 관련 PR: [백엔드 PR #26](https://github.com/GETDDO/getddo-be/pull/26). 이번 문서 갱신은 커밋·push·PR 갱신 전이다.
