# 공용 결정 기록

- [ADR-016: 기본·연속 출석 보상의 브론즈 응모권 지급](016-attendance-bronze-tickets.md) — 제안

- [ADR-015: 이벤트 일시 중단·재개 제외](015-remove-event-suspension.md)

- [ADR-014: 게임·미션 보상의 무작위 응모권 등급과 등급별 가중치](014-random-ticket-grades.md) — 제안

프론트엔드와 백엔드 모두에 영향을 주는 중요한 결정과 그 이유를 ADR로 기록합니다.

- [ADR-013: 추첨 가중치의 선형 환산과 별도 상한·배율 없음](013-linear-drawing-weight.md)

- [ADR-012: 이벤트·경품 동시 등록과 이벤트 수정 제한](012-event-registration-and-editing-rules.md)

- [ADR-011: 일정 앞당김의 추가 알림 제외 및 이벤트 삭제 시 알림 제거](011-event-notification-changes-and-deletion.md) — ADR-012로 대체됨

- [ADR-010: 일반 가중치 적용 이벤트의 응모권 사용 상한 5장](010-five-ticket-entry-limit.md)

- [ADR-009: 응모 마감 후 5분 검토 및 자동 최초 발표](009-five-minute-auto-publication.md)

## 작성 대상

- 여러 기능이나 코드 저장소에 영향을 주는 결정
- 변경 비용이 크거나 되돌리기 어려운 결정
- 검토한 대안과 선택 이유를 이후 작업에서도 참고해야 하는 결정
- 기존 요구사항이나 도메인 정책의 해석 기준이 되는 결정

특정 코드 저장소의 구현에만 영향을 주는 기술 결정은 해당 코드 저장소에서 관리합니다.

## 파일 이름

    001-decision-title.md
    002-next-decision.md

번호는 세 자리로 작성하고, 기존 문서를 확인한 뒤 다음 번호를 사용합니다. 기존 결정을 대체할 때는 이전 문서를 삭제하지 않고 새 ADR을 연결합니다.
