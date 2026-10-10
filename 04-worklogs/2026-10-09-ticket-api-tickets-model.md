# 작업 기록: 응모권 조회 API를 tickets·ticket_histories 모델로 갱신하고 용어 정리

- 작성일: 2026-10-09
- 관련 이슈 / PR: GD-73 / 백엔드 PR 36 / 스펙 PR 7
- 상태: 검토 대기
- 영향 범위: 공통 / 프론트엔드 / 백엔드

## 목적

최신 ERD에서 월별 지갑과 원장이 없어지고 응모권 한 장 단위의 `tickets`와 추가 전용 `ticket_histories`로 바뀌었다. 구현된 조회 API(T01·T02)에 맞춰 응모권 API 초안을 갱신하고, 응모권을 금액처럼 읽히게 하는 "잔액" 표현을 정리한다.

## 한 일

- [응모권 API](../05-api/ticket.md)의 T01·T02 경로와 DTO를 새 모델로 바꿨다.
- 관리자 조회(AO03·AO04·AO05·AO08)의 DTO 이름을 같은 모델에 맞췄다. 관리자 API는 구현 전 제안이다.
- [기능 요구사항](../00-requirements/functional-requirements.md)의 "이력 합계와 잔액의 일치" 표시를 "응모권의 현재 상태와 마지막 이력의 일치 여부" 표시로 바꿨다. 변경 전: 이력 합계와 잔액의 일치 상태를 표시한다. 변경 후: 현재 상태와 마지막 이력의 일치 여부를 표시하고 불일치 응모권을 목록으로 확인한다.
- [용어집](../02-domain/glossary.md)에 "보유 응모권"을 추가하고 차감 관련 정의에서 "잔액"을 뺐다.
- [응모권 규칙](../02-domain/ticket.md), [요구사항 안내](../00-requirements/README.md), [기능 요구사항](../00-requirements/functional-requirements.md), [API 목록](../05-api/README.md)의 "잔액" 표현을 "보유 응모권"으로 바꿨다.

## 변경 전후

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| T01 경로·응답 | `GET /tickets/wallets/me` → `MyWallets` | `GET /tickets/me` → `MyTickets` |
| T02 경로·응답 | `GET /tickets/ledger/me` → `Cursor<TicketTransaction>` | `GET /tickets/histories/me` → `Cursor<TicketHistory>` |
| 처리 유형 이름 | `transactionType`, 차감은 `SPEND` | `operationType`, 차감은 `USE` |
| 수량 표현 | `quantity`(부호 있음), `balanceAfter` | 이력 한 건이 응모권 한 장이라 수량·누계 필드 없음 |
| 보유 표현 | 월별 지갑 `balance` | 등급·만료 시각별 묶음 `count`, 등급별 `countByGrade` |

## 확인된 내용

- 백엔드 조회 API는 위 경로와 필드로 PR 36에서 구현·테스트했다.
- 경로의 `wallets`·`ledger`는 지갑 테이블이 없어지고 화폐처럼 읽혀 `tickets/me`, `tickets/histories/me`로 바꿨다.
- 요청 필터와 응답 필드는 `operationType` 하나로 통일했다.
- 프론트엔드는 목 데이터로 개발 중일 가능성이 있다. 실제 화면의 의존 여부는 확인하지 못했다.

## 다음 할 일

- [ ] 프론트엔드와 응답 모양, `SPEND`→`USE`, 수량 표현, 월별 지갑 화면 처리를 합의한다.
- [ ] 관리자 경로와 DTO를 구현 시점에 확정한다.
- [ ] "잔액"이 남은 다른 문서(출석·게임·응모 API, ADR-014·016, 요구사항, 미결정 사항)를 담당자와 정리한다.

## 미결정·차단 사항

- 출석 보상 응답에 `grade`를 추가할지는 [출석 API](../05-api/attendance.md) 담당 검토 대상으로 유지한다.
- 이력 한 건을 응모권 한 장으로 두는 대신 `quantity: 1`을 노출할지는 프론트엔드 합의 후 정한다.

## 반영 결과

- 변경한 명세: [응모권 API](../05-api/ticket.md), [응모권 규칙](../02-domain/ticket.md), [용어집](../02-domain/glossary.md), [요구사항 안내](../00-requirements/README.md), [기능 요구사항](../00-requirements/functional-requirements.md), [API 목록](../05-api/README.md)
- 관련 PR: 백엔드 PR 36, 스펙 PR 7
