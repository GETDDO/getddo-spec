# 사용자 API 초안

- 상태: AO01/AO02 관리자 조회 계약 사용자 확인 완료 (2026-10-15) / 본인 정보 초안 (U01 비활성 거절은 2026-10-16 사용자 요청으로 현행 구현 반영)
- 공통 계약: [공통 API 계약](common.md)

AO01/AO02는 아래 상세 계약을 따른다. 본인 정보는 아래에 현행 구현으로 명시한 비활성 거절 조건을 제외하고 초안 상태를 유지한다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 본인 정보

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| U01 | U/A | `GET /users/me` | 없음 | 200 `UserProfile` | 공통, `403 / USER-007` 비활성 사용자 |

`UserProfile`: `id:UUID`, `name:string(≤20)`, `role:USER/ADMIN`, `status:ACTIVE/INACTIVE`, `membership:excellent/vip/vvip/null`, `phoneNum:string?`, `email:string?`.

U01의 현행 구현은 공통 사용자 문맥 검증을 통과한 사용자가 `INACTIVE`이면 USER/ADMIN 역할에 관계없이 `403 / USER-007`(`비활성 사용자는 내 정보를 조회할 수 없습니다.`)을 반환한다. 이 제한은 U01에 적용하며 다른 API의 비활성 사용자 허용 여부를 일괄 변경하지 않는다.

멤버십 코드는 소문자를 유지한다. 화면 표시는 `excellent=우수`, `vip=VIP`, `vvip=VVIP`다. 별도 회원 생성·로그인·로그아웃·권한 변경 API는 이번 헤더 방식에 추가하지 않는다. 정지/해제 API는 이번 계약에 포함하지 않는다.

## 관리자 사용자 조회

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AO01 | `GET /admin/users` | `page,size,userId,keyword,role,status,membership` | 200 `Page<AdminUser>` | 공통 |
| AO02 | `GET /admin/users/{userId}` | 없음 | 200 `AdminUser` | 공통 |

| DTO | 필드 |
| --- | --- |
| `AdminUser` | `id:UUID`, `name:string(≤20)`, `role:USER/ADMIN`, `status:ACTIVE/INACTIVE`, `membership:excellent/vip/vvip/null`, `createdAt:instant`, `updatedAt:instant` |

관리자 조회는 읽기 전용이다. 사용자 역할·멤버십 모델은 백엔드 사용자 기능과 협의한다.

### 관리자 조회 필드 — 2026-10-15 사용자 선택

- 변경 전: AdminUser는 UserProfile 전체와 정지 시각·사유까지 포함하는 초안이었다.
- 변경 후: 사용자가 기본 운영 정보 7개(id·이름·역할·상태·멤버십·생성/수정 시각)를 선택했다. AO01/AO02의 AdminUser는 위 표의 필드만 제공하며 연락처·정지 시각·정지 사유는 포함하지 않는다.
- 이 선택은 관리자 조회 응답 범위에 대한 결정이다. U01 본인 정보나 사용자 저장 모델을 변경하는 결정이 아니다. 필드 선택 후 별도 확인을 거쳐 아래 상세 계약도 채택했다.
- 기존 FE 가상 사용자 선택·관리 화면의 `/virtual-users`와 AO01/AO02는 다른 계약이다. 관리자 조회 API를 일반 사용자의 시연 계정 선택 API로 대체하지 않는다.

### AO01/AO02 상세 계약 — 2026-10-15 사용자 확인

사용자는 상세안 채택, spec main 커밋·푸시, GD-62 하위 작업 생성과 구현을 명시적으로 확인했다. 본 절은 AO01/AO02에 한정하며 다른 API의 초안을 함께 확정하지 않는다.

- 실제 경로는 `/api/v1/admin/users`와 `/api/v1/admin/users/{userId}`다. 공통 사용자 문맥으로 확인한 ACTIVE ADMIN만 호출한다. 선택 멤버십 헤더는 기존 공통 대조를 따른다.
- page 기본 1·1 이상, size 기본 20·1~100. 정렬은 createdAt DESC·id DESC이며 Page의 totalElements는 선택 필터에 맞는 전체 건수다. 빈 결과·마지막 페이지 이후는 빈 items다.
- 선택 필터 userId·keyword·role·status·membership는 AND로 적용한다. userId는 정확한 UUID, role/status는 기존 대문자 코드, membership는 기존 소문자 코드다. 조회 대상은 비활성 사용자·관리자도 포함하며 전달한 필터로 좁힌다.
- keyword는 양끝 공백을 제거한 이름 부분 검색이다. 빈 검색어는 이름 필터가 없으며 최대 20자다. `%`·`_` 등의 문자는 와일드카드로 확대하지 않고 입력한 문자로 검색한다.
- 성공은 HTTP 200과 기존 응답 봉투다. 목록은 Page<AdminUser>, 상세는 AdminUser이며 membership이 없으면 null이다. 필수 생성/수정 시각은 UTC instant다.
- UUID·enum·숫자 타입 해석 실패는 공통 400 COMMON-005다. 페이지/크기·검색어 조건 위반은 400 USER-008(`사용자 조회 조건이 올바르지 않습니다.`), 호출자 자격 위반은 403 USER-009(`활성 관리자만 사용자 정보를 조회할 수 있습니다.`), 없는 상세는 404 USER-010(`사용자를 찾을 수 없습니다.`)을 사용한다. 현재 사용자 식별·헤더 대조 오류는 공통 계약을 그대로 따른다.
- 조회는 사용자 정보·권한·멤버십·정지 상태를 변경하지 않는다. FE 사용자 선택 경로와 A 담당의 사용자·인증 기능에 영향을 주는 수정은 먼저 담당 경계를 확인한다.
