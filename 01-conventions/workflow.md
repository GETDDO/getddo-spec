# 기본 개발 흐름

모든 개발 작업은 다음 흐름을 따른다.

```text
Jira 이슈 생성
      ↓
Jira에서 GitHub 브랜치 생성 (dev 기준)
      ↓
로컬에서 작업
      ↓
Commit
      ↓
Push
      ↓
Pull Request → dev
      ↓
CI Build / Test
      ↓
Code Review
      ↓
Squash and Merge
      ↓
dev
```

예시:

```text
Jira: GD-21 쿠폰 발급 API 구현
      ↓
GitHub Branch: feat/GD-21
      ↓
Pull Request: feat/GD-21 → dev
      ↓
CI 성공
      ↓
1명 이상 Approve
      ↓
Squash and Merge
```

## 공용 명세 작업 기록

공용 명세를 검토하거나 정리하는 작업은 `04-worklogs`에 작업별로 기록한다.

1. 작업을 시작할 때 목적, 영향 범위와 다음 할 일을 작성한다.
2. 작업 중 확인된 사실과 미결정 사항을 구분해 갱신한다.
3. 확정된 정책은 요구사항 또는 도메인 문서에 반영한다.
4. 중요한 결정은 ADR로, 미확정 정책은 `00-requirements/pending-decisions.md`로 분리한다.
5. 작업 완료 시 반영한 문서, 관련 ADR과 Pull Request를 연결하고 상태를 완료로 변경한다.

작업 기록 자체를 확정된 요구사항이나 도메인 정책의 근거로 사용하지 않는다.
