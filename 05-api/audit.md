# 감사 로그 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AU01 | `GET /admin/audit-logs` | `page,size,actorId,action,targetType,targetId,from,to` | 200 `Page<AuditLogSummary>` | 공통 |
| AU02 | `GET /admin/audit-logs/{auditLogId}` | 없음 | 200 `AuditLogDetail` | 공통 |

`AuditLogSummary`: `id:UUID`, `actorId:UUID?`, `action:string`, `targetType:string`, `targetId:UUID`, `reason:string?`, `requestId:string?`, `createdAt:instant`.

`AuditLogDetail`: Summary 전체 + `beforeData:object?`, `afterData:object?`. 상세 데이터는 관리자가 검토할 변경 근거에 한정하고 비밀값·무관한 개인정보를 기록하거나 응답하지 않는다. 감사 로그 수정·삭제 API는 제공하지 않는다.
