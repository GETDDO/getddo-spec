# 배너 API

- 상태: B01·AB01~AB05 사용자 채택 — 2026-10-15
- 공통 계약: [공통 API 계약](common.md)
- 도메인 기준: [배너 규칙](../02-domain/banner.md)
- 주 담당: 전민규. 이미지 기반은 A, 이벤트 기반은 E, 관리자 화면은 FE 담당자와 연동한다.

아래 경로는 /api/v1 접두사를 생략한다. 사용자 문맥과 성공·실패 봉투를 재사용한다. 비로그인 조회를 새로 추가하지 않는다. 멤버십 헤더는 선택이며 전달하면 기존 DB 대조를 유지한다.

## 사용자 배너 B01

| ID | 권한 | 메서드·경로 | 성공 |
| --- | --- | --- | --- |
| B01 | 활성 USER/ADMIN | GET /banners | 200 Banner[] |

Banner: id:UUID,eventId:UUID,imageUrl:string,displayOrder:int. 삭제·취소 이벤트는 제외하고 displayOrder ASC,id ASC로 반환한다. 빈 목록은 배열이며 별도 페이지를 추가하지 않는다. 숨김 후 사용자 목록의 순서 숫자는 연속이 아닐 수 있지만 상대 순서는 유지한다.

## 관리자 AB01~AB05

모든 요청은 활성 ADMIN 전용이다.

| ID | 메서드·경로 | 요청 | 성공 |
| --- | --- | --- | --- |
| AB01 | GET /admin/banners | 없음 | 200 AdminBanner[] |
| AB02 | POST /admin/banners | BannerWrite | 201 AdminBanner |
| AB03 | PUT /admin/banners/{bannerId} | BannerWrite | 200 AdminBanner |
| AB04 | DELETE /admin/banners/{bannerId} | 없음 | 200 null |
| AB05 | PUT /admin/banners/order | bannerIds:UUID[] 전체 목록 | 200 AdminBanner[] |

BannerWrite: 필수 eventId:UUID,imageKey:string(앞뒤 공백 제거 후1~500자),displayOrder:int. AdminBanner: Banner 전체+imageKey:string,createdAt:instant,updatedAt:instant. createdBy는 반환하지 않는다. 목록은 숨김 배너를 포함하며1~N 순서다.

- 등록 순서는 현재 N개에서1~N+1, N=5이면 거절한다. 사이에 삽입하면 뒤로 민다. 숨김도 전체5개에 포함한다.
- 수정 순서는1~N이며 사이의 배너를 함께 이동한다. 삭제하면 빈자리를 당긴다.
- 연결할 이벤트의 존재·미삭제 여부를 검증한다. 취소된 이벤트 신규 연결은409로 거절한다. 이미 연결된 뒤 삭제·취소되면 사용자에게만 숨긴다.
- 전체 순서는 현재 전체 ID를 중복 없이 받아 원자적으로 반영한다. 누락·추가·중복·동시 목록 변경은409이며 다시 조회한다. 순서 변경의 일부만 성공시키지 않는다.
- 이미지 키를 공개 기준 주소로 변환한다. 저장소 접근 주소를 응답하지 않는다. 실제 키 형식·업로드·형식/용량은 별도 연동 확인 대상이며 현재 확인되지 않은 서비스를 완료로 표시하지 않는다.
- 등록·수정·삭제·전체 순서는 공통 감사 기록과 같은 업무 트랜잭션에서 확정한다. 감사 조회가 아직 채택하지 않은 전후 JSON 필드를 임의 공개하지 않는다.

## 오류

| HTTP·코드 | 공개 메시지 |
| --- | --- |
| 400 BANNER-001 | 배너 설정이 올바르지 않습니다. |
| 403 BANNER-002 | 활성 관리자만 사용할 수 있습니다. |
| 404 BANNER-003 | 배너를 찾을 수 없습니다. |
| 409 BANNER-004 | 배너는 최대 5개까지 등록할 수 있습니다. |
| 409 BANNER-005 | 배너 목록이 변경되었습니다. 다시 조회해 주세요. |
| 404 BANNER-006 | 연결할 이벤트를 찾을 수 없습니다. |
| 409 BANNER-007 | 취소된 이벤트에는 배너를 연결할 수 없습니다. |
| 403 BANNER-008 | 활성 사용자만 배너를 조회할 수 있습니다. |

필수 필드·길이 위반은 기존 COMMON-002, JSON·UUID·타입 해석 오류는 COMMON-005다. 공통 사용자 문맥 오류를 덮지 않는다. 위 코드는 이번 배너 계약에서 채택한 업무 오류다.

## 배포·담당자 후속

기존 URL 행의 유무·연결을 확인하고 기존 데이터는 보존한다. 새 eventId 모델만 보고 기존 행에 임의 UUID를 채우거나 삭제하지 않는다. 실제 DB 이전과 동시성 저장 방식은 BE에서 별도 검증한다. FE의0기반 순서·목 자동 삭제·API 접두사·실제 이미지 연결은 후속 조정 대상이다. 이미지 업로드·실제 FE·배포 확인은 API 코드 구현과 구분한다.
