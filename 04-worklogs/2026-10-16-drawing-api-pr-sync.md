# 작업 기록: 추첨 공개 계약과 GD-199 PR 동기화

- 작성일: 2026-10-16 KST
- 관련 이슈 / PR: GD-137·GD-169·GD-199 / [BE PR #108](https://github.com/Uplus-GETDDO/getddo-be/pull/108)
- 상태: 검토 대기 — Codex 작성, 담당자 확인 전
- 영향 범위: 공통 / 프론트엔드 / 백엔드

## 목적

사용자가 요청한 추첨 스펙 정리를 수행한다. 현재 dev와 미병합 GD-199 PR의 적용 상태를 구분하고 기존 로컬의 단일 발표 결정을 보존한다. 초기 요청은 커밋·push를 제외했으며, 이후 사용자가 내용을 확인하고 커밋·push를 승인했다. BE PR 병합은 요청 범위에 포함하지 않는다.

## 한 일

- 관리자 조회의 drawType·execution·취소 요약을 실제 HTTP DTO에 맞춰 정리했다.
- 실패·업무별 실행 감사 경로와 페이지·오류·응답을 기록하고 공통 감사 조회 제외를 유지했다.
- 공개 변경의 executions 목록과 기존 실행 확인자/실제 발표자를 구분했다.
- GD-199의 최신 runId 검사·본문 없는 단일 발표·기존 DTO/오류·최초 버전 재시도를 PR 기준으로 기록했다.
- API 안내와 추첨 후속 목록을 정렬했다. 공유 시 단일 발표 정책과 충돌하지 않도록 기존 로컬 결정의 요구사항·범위·추첨/이벤트 규칙·ADR-022를 같은 내용으로 게시한다. 원격의 최신 게임·감사·이벤트 취소 계약과 기존 작업 기록은 보존한다. 새 API 업무 번호·필드·테이블·오류 코드를 만들지 않았다.

## 확인된 내용

- BE origin/dev는 88833cef9971068d8042bb4fd7078db3da65f381, spec origin/main은 32e105545a389fffdc843bf1d11f8d7129163a14를 확인했다.
- GD-199 PR #108은 OPEN, head는 65889fc6e76d0ed090e05324d81c90be431c0176이다. 문서 반영은 병합·배포·FE 적용 완료가 아니다.
- 두 원본 작업 트리의 기존 미커밋 변경을 보존했다. 원본 저장소에 pull/stash/reset하지 않고 원격 스펙은 임시 복제본에서 읽었다.
- 확정 단일 발표 정책은 [추첨 규칙](../02-domain/drawing.md#2026-10-15-단일-발표-요청으로-최종-확인공개), 공개 API의 실제 필드와 PR 적용 상태는 [추첨 API](../05-api/drawing.md)를 따른다.

## 다음 할 일

- [ ] 담당자가 현행 구현/PR 대조 상세를 검토하고 FE에 공유한다.
- [ ] GD-199 리뷰·병합·배포와 FE 공개 전후 연동을 확인한다.
- [ ] PUBLICATION-002의 최초 결과 준비 거절에도 맞는 안내 문구를 담당자가 확인한다.

## 미결정·차단 사항

- 추가 경로의 업무 ID는 임의로 배정하지 않았다.
- 개인정보 파기·보관 세부와 발표 후 중간 사용자 표시 등 기존 후속 정책은 이번 작업에서 확정하지 않았다.
- 알림 담당 문서의 자동 최초 발표 문구는 기존 동기화 후속으로 유지한다.
- 원본 로컬 감사 API 초안은 덮어쓰지 않았다. 공유용 복제본은 최신 main을 기준으로 하므로 원격 감사 명세와 ADR-024를 보존한다. 원본 로컬 감사 문서 최신화는 별도 후속이다.

## 반영 결과

- 변경한 명세: [추첨 API](../05-api/drawing.md), [API 안내](../05-api/README.md), [후속 목록](../00-requirements/pending-decisions.md), [요구사항](../00-requirements/functional-requirements.md), [범위](../00-requirements/scope.md), [추첨 규칙](../02-domain/drawing.md), [이벤트 규칙](../02-domain/event.md), [ADR-022](../03-decisions/022-manual-initial-publication.md)
- 관련 ADR: [ADR-022](../03-decisions/022-manual-initial-publication.md), [감사 기록/조회 제외 ADR](https://github.com/Uplus-GETDDO/getddo-spec/blob/32e105545a389fffdc843bf1d11f8d7129163a14/03-decisions/024-audit-recording-without-query.md)
- 검증: 초기 수정의 로컬 링크·앵커 9개와 공유용 변경의 추가 링크·앵커 16개, HTTP DTO 필드 집합 11개, 경로 표 15개와 공백·충돌 표시를 검사했다. 문서 변경만 수행해 test/build는 실행하지 않았다. 기존 코드/PR 테스트 소스를 확인한 것과 테스트 실행 성공은 구분한다.
- 관련 PR: [BE #108](https://github.com/Uplus-GETDDO/getddo-be/pull/108). 사용자 확인 후 spec main에 직접 커밋·push하며 문서 PR·BE 병합은 하지 않는다.
