# GETDDO Spec

GETDDO 프로젝트의 공통 요구사항, 도메인 규칙 및 주요 결정 사항을 관리하는 저장소입니다.

프론트엔드와 백엔드가 동일한 기준으로 기능을 구현할 수 있도록 프로젝트 범위, 용어, 정책, 상태와 사용자 동작을 한곳에서 관리합니다.

## 문서 구성

```text
getddo-spec/
├── README.md
├── AGENTS.md
├── 00-requirements/
│   ├── README.md
│   ├── functional-requirements.md
│   ├── scope.md
│   └── pending-decisions.md
├── 01-conventions/
│   ├── README.md
│   ├── branch.md
│   ├── workflow.md
│   ├── commit.md
│   └── pull-request.md
├── 02-domain/
│   ├── README.md
│   ├── glossary.md
│   ├── event.md
│   ├── ticket.md
│   ├── entry.md
│   ├── drawing.md
│   └── notification.md
├── 03-decisions/
│   └── README.md
├── 04-worklogs/
│   ├── README.md
│   └── 2026-09-17-requirements-refinement.md
├── 05-api/
│   ├── README.md
│   ├── common.md
│   └── [도메인별 API 파일]
└── templates/
    ├── adr.md
    └── worklog.md
```

### `00-requirements`

프로젝트의 목표, 기능 요구사항, 구현 범위와 아직 확정되지 않은 정책을 관리합니다.

### `01-conventions`

프론트엔드와 백엔드가 공통으로 사용하는 브랜치, 커밋, Pull Request 및 협업 절차를 관리합니다. 언어나 프레임워크에 종속된 코드 스타일은 각 코드 저장소에서 관리합니다.

### `02-domain`

이벤트, 응모권, 응모, 추첨, 알림에 공통으로 적용되는 용어와 도메인 정책을 관리합니다.

### `03-decisions`

프론트엔드와 백엔드 모두에 영향을 주는 중요한 결정과 그 이유를 ADR로 기록합니다. 특정 구현에만 적용되는 기술 결정은 해당 코드 저장소에서 관리합니다.

### `04-worklogs`

공용 명세를 검토하고 정리한 과정, 확인된 내용과 다음 할 일을 작업별로 기록합니다. 작업 기록은 진행 맥락을 보존하기 위한 문서이며 확정된 요구사항이나 도메인 정책을 대신하지 않습니다.

### `05-api`

프론트엔드와 백엔드가 함께 검토할 [API 계약 초안](05-api/README.md)을 도메인별로 관리합니다. 확정된 요구사항과 도메인 규칙을 우선하며, DB와 내부 구현 설계는 코드 저장소에서 관리합니다.

### `templates`

공용 결정 기록과 작업 기록에 사용하는 문서 양식을 관리합니다.

## 관리 원칙

- 이 저장소의 확정된 요구사항과 도메인 정책을 프론트엔드와 백엔드 구현의 공통 기준으로 사용합니다.
- 확정되지 않은 내용은 임의로 결정하지 않고 `pending-decisions.md`에 기록합니다.
- 공용 정책과 특정 저장소의 구현 세부사항을 구분합니다.
- 정책 변경은 프론트엔드와 백엔드에 미치는 영향을 함께 확인한 뒤 Pull Request로 반영합니다.
- 작업 기록의 확정 사항은 해당 요구사항 또는 도메인 문서에 반영하고 기준 문서를 연결합니다.
- 같은 문서를 여러 저장소에 복사하지 않고 이 저장소의 문서를 원본으로 관리합니다.

## 관련 저장소

- [`getddo-fe`](https://github.com/GETDDO/getddo-fe): 프론트엔드 애플리케이션
- [`getddo-be`](https://github.com/GETDDO/getddo-be): 백엔드 애플리케이션
