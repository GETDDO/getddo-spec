# 배너 API 초안

- 상태: 검토 대기 — 담당자 확인 전
- 공통 계약: [공통 API 계약](common.md)
- 주 담당: 전민규 — 사용자 배너 조회와 관리자 배너·노출 순서 관리

경로·메서드·DTO·업무 오류 코드는 구현 전 제안이다. 확정된 업무 정책은 연결된 요구사항·도메인 문서를 따른다.

## 사용자 배너

| ID | 권한 | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- | --- |
| B01 | R | `GET /banners` | 없음 | 200 `Banner[]` | 공통 |

`Banner`: `id:UUID`, `eventId:UUID`, `imageUrl:string`, `displayOrder:int`. 노출 순서는 `displayOrder ASC, id ASC`로 제안한다. 연결 URL은 eventId로 이벤트 화면에 연결한다. 최대 5개이며 노출 기간·자동 슬라이드 간격을 별도 서버 정책으로 추가하지 않는다.

## 관리자 배너 관리

| ID | 메서드·경로 | 요청 | 성공 | 주요 오류 |
| --- | --- | --- | --- | --- |
| AB01 | `GET /admin/banners` | 없음 | 200 `AdminBanner[]` | 공통 |
| AB02 | `POST /admin/banners` | `BannerWrite` | 201 `AdminBanner` | 409 최대 5개 초과 |
| AB03 | `PUT /admin/banners/{bannerId}` | `BannerWrite` | 200 `AdminBanner` | 공통 |
| AB04 | `DELETE /admin/banners/{bannerId}` | 없음 | 200 `null` | 공통 |
| AB05 | `PUT /admin/banners/order` | 필수 `bannerIds:UUID[]` | 200 `AdminBanner[]` | 409 현재 목록과 불일치 |

`BannerWrite`: 필수 `eventId:UUID`, `imageKey:string(1~500)`, `displayOrder:int(≥0)` 제안. `AdminBanner`: Banner 전체 + `imageKey:string`, `createdBy:UUID`, `createdAt:instant`, `updatedAt:instant`.

AB05는 현재 배너 전체 ID를 중복 없이 노출 순서대로 받아 원자적으로 반영한다. 연결 이벤트 존재와 동시 등록 시 최대 5개 제한을 서버에서 검증한다. 이미지 형식·용량 수치는 이미지 처리 계약이며 임의로 5MB 등을 채택하지 않는다.
