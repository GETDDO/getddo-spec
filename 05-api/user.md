# 사용자 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 본인 정보

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| U01 | U/A | `GET /users/me` | 없음 | 200 `UserProfile` | 공통 |

`UserProfile`: `id:UUID`, `name:string(≤20)`, `role:USER/ADMIN`, `status:ACTIVE/INACTIVE`, `membership:excellent/vip/vvip/null`, `phoneNum:string?`, `email:string?`.

멤버십 코드는 소문자를 유지한다. 화면 표시는 `excellent=우수`, `vip=VIP`, `vvip=VVIP`다. 별도 회원 생성·로그인·로그아웃·권한 변경 API는 이번 헤더 방식에 추가하지 않는다. 정지/해제 API는 이번 계약에 포함하지 않는다.

## 관리자 사용자 조회

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AO01 | `GET /admin/users` | `page,size,userId,keyword,role,status,membership` | 200 `Page<AdminUser>` | 공통 |
| AO02 | `GET /admin/users/{userId}` | 없음 | 200 `AdminUser` | 공통 |

| DTO | 필드 |
| --- | --- |
| `AdminUser` | UserProfile 전체 + `createdAt:instant`, `updatedAt:instant`, `suspendedAt:instant?`, `suspensionReason:string?` |

관리자 조회는 읽기 전용이다. 사용자 역할·멤버십 모델은 백엔드 사용자 기능과 협의한다.
