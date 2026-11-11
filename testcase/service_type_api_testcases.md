# API Test Design — Service Type Management

## 1. Requirement Analysis

Nguồn: `basic design/document.json` (OpenAPI 3.1.0).

| REQ ID | API | Requirement chính |
|---|---|---|
| REQ-ST-01 | `GET /api/admin/service-types` | Bearer auth; 200 trả mảng item gồm `id`, `name`, `sort_order`, `office_count`; có 401/403. |
| REQ-ST-02 | `POST /api/admin/service-types` | Body bắt buộc: `name:string,maxLength=255`, `sort_order:integer,minimum=1`; có 200/401/403/422. |
| REQ-ST-03 | `PUT /api/admin/service-types/{serviceType}` | Path ID là integer; body giống create; có 200/401/403/404/422. |
| REQ-ST-04 | `DELETE /api/admin/service-types/{serviceType}` | Xóa thành công trả 200 với `data:null`; có 400/401/403/404. |
| REQ-ST-05 | Authorization | Matrix hiện tại cho phép `super`, `content_admin`, `cs_lead`, `cs`; từ chối `viewer`. |

### Field Specifications

| Field | Type | Required | Constraint |
|---|---|---|---|
| `name` | string | Có | Tối đa 255 ký tự; không có `minLength` hoặc unique rule |
| `sort_order` | integer | Có | Tối thiểu 1; không có maximum |
| `serviceType` | integer path | Có với PUT/DELETE | Không có range |

### QnA / Gap

| ID | Điểm chưa rõ | Ảnh hưởng |
|---|---|---|
| ST-Q01 | `name=""` hoặc chỉ khoảng trắng có hợp lệ không? | Không sinh assertion. |
| ST-Q02 | `name` có unique và có trim không? | Không sinh duplicate test. |
| ST-Q03 | Service Type nào chắc chắn đang được office sử dụng? | Chưa chạy an toàn nhánh DELETE 400. |
| ST-Q04 | Exact error message là gì? | Chỉ assert status và schema lỗi. |
| ST-Q05 | Expected cho malformed JSON, sai Content-Type, oversized payload, rate limit? | Không tự giả định 400/413/415/429. |
| ST-Q06 | Role `viewer` chưa được chốt cuối cùng trong BD. | Case 403 dùng expected từ matrix hiện tại; nếu lệch cần xác nhận, không tự kết luận nghiệp vụ. |

### Test Data Strategy

- Tạo động: `auto_api_service_type_<timestamp>_<random>`.
- Chỉ xóa record do collection tạo.
- Collection lưu `service_type_id` và `service_type_boundary_id`, sau đó cleanup chấp nhận 200 hoặc 404.
- Credential lấy từ environment: `admin_email`, `admin_password`, `viewer_email`, `viewer_password`.

## 2. Test Viewpoint

| VP ID | Category | Viewpoint | Source | Technique | Risk |
|---|---|---|---|---|---|
| VP_ST_001 | Functional | Danh sách trả đúng Service Type | REQ-ST-01 | EP | Medium |
| VP_ST_002 | Response | Success response đúng contract | REQ-ST-01..04 | Schema Validation | High |
| VP_ST_003 | Authorization | Thiếu/sai token bị từ chối | REQ-ST-01..04 | Decision Table | High |
| VP_ST_004 | Authorization | Viewer bị từ chối | REQ-ST-05 | Decision Table | High |
| VP_ST_005 | Functional | Tạo Service Type hợp lệ | REQ-ST-02 | EP | High |
| VP_ST_006 | Input | Required/type của `name` | REQ-ST-02,03 | EP | High |
| VP_ST_007 | Boundary | `name` tại 255/256 ký tự | REQ-ST-02,03 | BVA | High |
| VP_ST_008 | Input | Required/type/min của `sort_order` | REQ-ST-02,03 | EP/BVA | High |
| VP_ST_009 | Functional | Cập nhật Service Type hợp lệ | REQ-ST-03 | EP | High |
| VP_ST_010 | Error Handling | ID không tồn tại trả 404 | REQ-ST-03,04 | EP | Medium |
| VP_ST_011 | Functional | Xóa record do test tạo | REQ-ST-04 | EP | High |
| VP_ST_012 | Data Integrity | Create/update/delete phản ánh đúng trong danh sách | REQ-ST-01..04 | State Verification | High |
| VP_ST_013 | Error Handling | Xóa Service Type đang được sử dụng trả 400 | REQ-ST-04 | EP | High |

## 3. Test Cases

Precondition chung: import environment Welvie STG; chạy collection đúng thứ tự; setup login và tạo dữ liệu động.

| TC ID | Module | VP ID | Risk | Scenario | Steps | Expected Result | Test Data | Priority | Automatable | Auto Type | Tags |
|---|---|---|---|---|---|---|---|---|---|---|---|
| WELVIE_SERVICE_TYPE_TC_001 | Service Type | VP_ST_001 | Medium | Lấy danh sách hợp lệ | 1. GET danh sách bằng admin token.<br>2. Validate từng item. | 1. HTTP 200, success envelope.<br>2. Mỗi item có `id/name/sort_order/office_count` đúng kiểu. | Token động | P0 | Yes | API | `@Smoke` |
| WELVIE_SERVICE_TYPE_TC_002 | Service Type | VP_ST_003 | High | GET thiếu token | 1. GET không Authorization.<br>2. Validate lỗi. | 1. HTTP 401.<br>2. AuthenticationException đúng schema. | No auth | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_SERVICE_TYPE_TC_003 | Service Type | VP_ST_003 | High | GET token sai | 1. GET bằng `invalid.jwt.token`.<br>2. Validate lỗi. | 1. HTTP 401.<br>2. AuthenticationException đúng schema. | `invalid.jwt.token` | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_SERVICE_TYPE_TC_004 | Service Type | VP_ST_004 | High | Viewer GET danh sách | 1. GET bằng viewer token.<br>2. Validate lỗi. | 1. HTTP 403.<br>2. AuthorizationException đúng schema. | Viewer token động | P0 | Yes | API | `@Permission` |
| WELVIE_SERVICE_TYPE_TC_005 | Service Type | VP_ST_005 | High | Create hợp lệ | 1. POST dữ liệu động.<br>2. Lưu ID response. | 1. HTTP 200, resource đúng schema/value.<br>2. `service_type_id` được lưu. | Dynamic name; `sort_order=101` | P0 | Yes | API | `@Smoke` |
| WELVIE_SERVICE_TYPE_TC_006 | Service Type | VP_ST_006 | High | Create thiếu name | 1. POST chỉ có sort_order.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.name` là array. | `{"sort_order":101}` | P0 | Yes | API | `@Validation` |
| WELVIE_SERVICE_TYPE_TC_007 | Service Type | VP_ST_006 | High | Create name null | 1. POST name null.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.name` là array. | `name=null` | P1 | Yes | API | `@Validation` |
| WELVIE_SERVICE_TYPE_TC_008 | Service Type | VP_ST_006 | High | Create name number | 1. POST name number.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.name` là array. | `name=12345` | P1 | Yes | API | `@Validation` |
| WELVIE_SERVICE_TYPE_TC_009 | Service Type | VP_ST_007 | High | Create name 255 ký tự | 1. POST name gồm 255 `A`.<br>2. Lưu boundary ID. | 1. HTTP 200; name giữ đủ 255 ký tự.<br>2. Boundary ID được lưu. | 255 ký tự `A`; `sort_order=102` | P0 | Yes | API | `@Boundary` |
| WELVIE_SERVICE_TYPE_TC_010 | Service Type | VP_ST_007 | High | Create name 256 ký tự | 1. POST name gồm 256 `A`.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.name` là array. | 256 ký tự `A` | P0 | Yes | API | `@Boundary` |
| WELVIE_SERVICE_TYPE_TC_011 | Service Type | VP_ST_008 | High | Create thiếu sort_order | 1. POST chỉ có name.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.sort_order` là array. | Dynamic name | P0 | Yes | API | `@Validation` |
| WELVIE_SERVICE_TYPE_TC_012 | Service Type | VP_ST_008 | High | Create sort_order bằng 0 | 1. POST sort_order=0.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.sort_order` là array. | `sort_order=0` | P0 | Yes | API | `@Boundary` |
| WELVIE_SERVICE_TYPE_TC_013 | Service Type | VP_ST_008 | High | Create sort_order string | 1. POST sort_order `"1"`.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.sort_order` là array. | `sort_order="1"` | P1 | Yes | API | `@Validation` |
| WELVIE_SERVICE_TYPE_TC_014 | Service Type | VP_ST_003 | High | POST thiếu token | 1. POST body hợp lệ không token.<br>2. Validate lỗi. | 1. HTTP 401.<br>2. AuthenticationException đúng schema. | Dynamic name | P0 | Yes | API | `@Security` |
| WELVIE_SERVICE_TYPE_TC_015 | Service Type | VP_ST_004 | High | Viewer POST | 1. POST bằng viewer token.<br>2. Validate lỗi. | 1. HTTP 403.<br>2. Không tạo record. | Dynamic name | P0 | Yes | API | `@Permission` |
| WELVIE_SERVICE_TYPE_TC_016 | Service Type | VP_ST_009 | High | Update hợp lệ | 1. PUT record TC005.<br>2. Validate response. | 1. HTTP 200.<br>2. Name/sort_order bằng request. | Suffix `_updated`; `sort_order=201` | P0 | Yes | API | `@Smoke` |
| WELVIE_SERVICE_TYPE_TC_017 | Service Type | VP_ST_007 | High | Update name 255 ký tự | 1. PUT name 255 `B`.<br>2. Validate response. | 1. HTTP 200.<br>2. Name giữ đủ 255 ký tự. | 255 ký tự `B` | P0 | Yes | API | `@Boundary` |
| WELVIE_SERVICE_TYPE_TC_018 | Service Type | VP_ST_007 | High | Update name 256 ký tự | 1. PUT name 256 `B`.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.name` là array. | 256 ký tự `B` | P0 | Yes | API | `@Boundary` |
| WELVIE_SERVICE_TYPE_TC_019 | Service Type | VP_ST_006 | High | Update thiếu name | 1. PUT chỉ có sort_order.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.name` là array. | `sort_order=202` | P0 | Yes | API | `@Validation` |
| WELVIE_SERVICE_TYPE_TC_020 | Service Type | VP_ST_008 | High | Update sort_order=0 | 1. PUT sort_order=0.<br>2. Validate lỗi. | 1. HTTP 422.<br>2. `errors.sort_order` là array. | `sort_order=0` | P0 | Yes | API | `@Boundary` |
| WELVIE_SERVICE_TYPE_TC_021 | Service Type | VP_ST_010 | Medium | Update ID không tồn tại | 1. PUT ID `2147483647`.<br>2. Validate lỗi. | 1. HTTP 404.<br>2. ModelNotFoundException đúng schema. | ID `2147483647` | P1 | Yes | API | `@Negative` |
| WELVIE_SERVICE_TYPE_TC_022 | Service Type | VP_ST_003 | High | PUT thiếu token | 1. PUT record test không token.<br>2. Validate lỗi. | 1. HTTP 401.<br>2. AuthenticationException đúng schema. | Test ID | P0 | Yes | API | `@Security` |
| WELVIE_SERVICE_TYPE_TC_023 | Service Type | VP_ST_004 | High | Viewer PUT | 1. PUT record test bằng viewer.<br>2. Validate lỗi. | 1. HTTP 403.<br>2. Record không bị cập nhật. | Test ID | P0 | Yes | API | `@Permission` |
| WELVIE_SERVICE_TYPE_TC_024 | Service Type | VP_ST_004 | High | Viewer DELETE | 1. DELETE record test bằng viewer.<br>2. Validate lỗi. | 1. HTTP 403.<br>2. Record không bị xóa. | Test ID | P0 | Yes | API | `@Permission` |
| WELVIE_SERVICE_TYPE_TC_025 | Service Type | VP_ST_003 | High | DELETE thiếu token | 1. DELETE record test không token.<br>2. Validate lỗi. | 1. HTTP 401.<br>2. AuthenticationException đúng schema. | Test ID | P0 | Yes | API | `@Security` |
| WELVIE_SERVICE_TYPE_TC_026 | Service Type | VP_ST_010 | Medium | DELETE ID không tồn tại | 1. DELETE ID `2147483647`.<br>2. Validate lỗi. | 1. HTTP 404.<br>2. ModelNotFoundException đúng schema. | ID `2147483647` | P1 | Yes | API | `@Negative` |
| WELVIE_SERVICE_TYPE_TC_027 | Service Type | VP_ST_011 | High | DELETE record test | 1. DELETE ID từ TC005.<br>2. Validate response. | 1. HTTP 200.<br>2. `data=null`. | `service_type_id` | P0 | Yes | API | `@Smoke` |
| WELVIE_SERVICE_TYPE_TC_028 | Service Type | VP_ST_012 | High | Danh sách không còn ID đã xóa | 1. GET danh sách.<br>2. Tìm ID TC005. | 1. HTTP 200.<br>2. Không có ID đã xóa. | Deleted ID | P0 | Yes | API | `@DataIntegrity` |
| WELVIE_SERVICE_TYPE_TC_029 | Service Type | VP_ST_010 | Medium | DELETE lại cùng ID | 1. DELETE ID đã xóa.<br>2. Validate lỗi. | 1. HTTP 404.<br>2. ModelNotFoundException đúng schema. | Deleted ID | P1 | Yes | API | `@Negative` |
| WELVIE_SERVICE_TYPE_TC_030 | Service Type | VP_ST_011 | High | DELETE boundary record | 1. DELETE ID từ TC009.<br>2. Validate response. | 1. HTTP 200.<br>2. `data=null`. | `service_type_boundary_id` | P0 | Yes | API | `@Cleanup` |

## 4. Viewpoint Coverage

| VP ID | TC IDs | Status |
|---|---|---|
| VP_ST_001 | 001 | ✅ |
| VP_ST_002 | 001,005,009,016,017,027,030 | ✅ |
| VP_ST_003 | 002,003,014,022,025 | ✅ |
| VP_ST_004 | 004,015,023,024 | ✅ |
| VP_ST_005 | 005,009 | ✅ |
| VP_ST_006 | 006,007,008,019 | ✅ |
| VP_ST_007 | 009,010,017,018 | ✅ |
| VP_ST_008 | 011,012,013,020 | ✅ |
| VP_ST_009 | 016,017 | ✅ |
| VP_ST_010 | 021,026,029 | ✅ |
| VP_ST_011 | 027,030 | ✅ |
| VP_ST_012 | 005,016,027,028 | ✅ |
| VP_ST_013 | — | ❌ GAP — ST-Q03 |

Coverage: **12/13 VP = 92.3%**. High/Medium chưa đạt 100% do thiếu fixture an toàn cho DELETE 400.
