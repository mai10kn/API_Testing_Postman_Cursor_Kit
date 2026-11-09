# API Test Design — Admin Profile `/api/admin/auth/me`

## 1. Phân tích requirement

### Overview

- Nguồn: `basic design/document.json` — OpenAPI 3.1.0, version 1.0.0.
- Base URL được tài liệu khai báo: `https://api-dev.stg.welvielearning.com`.
- API áp dụng Bearer Authentication theo security scheme toàn cục `http`.
- Phạm vi:
  - `GET /api/admin/auth/me`: lấy profile của admin đang đăng nhập.
  - `POST /api/admin/auth/me`: cập nhật tên profile của admin đang đăng nhập.
- Ngoài phạm vi: UI, API profile của staff, đổi mật khẩu, đổi email và upload avatar.

### Actors & Preconditions

- Actor: tài khoản admin có thể đăng nhập Admin Portal.
- Login qua `POST /api/admin/auth/login`.
- Lấy `data.access_token` và lưu vào Postman Collection Variable `access_token`.
- Các request trong bộ test sử dụng `Authorization: Bearer {{access_token}}`.

### Functional Breakdown

| REQ ID | Requirement |
|---|---|
| REQ-APRO-01 | API được bảo vệ bằng Bearer token. |
| REQ-APRO-02 | GET thành công trả HTTP 200 và envelope gồm `message`, `errors`, `data`, `meta`. |
| REQ-APRO-03 | `data` của GET tuân theo `ProfileResource`: `id`, `name`, `email`, `role`, `status`, `created_at`. |
| REQ-APRO-04 | POST nhận JSON body có trường `name` bắt buộc, kiểu string, tối đa 255 ký tự. |
| REQ-APRO-05 | POST hợp lệ trả HTTP 200 và `ProfileResource` trong envelope chuẩn. |
| REQ-APRO-06 | Request không xác thực trả HTTP 401; tài liệu cũng khai báo trường hợp 403. |
| REQ-APRO-07 | POST vi phạm schema/validation trả HTTP 422 với `message`, `errors`, `data`, `meta`. |

### Field Specifications

| Field | Location | Type | Required | Constraint |
|---|---|---:|---:|---|
| `name` | POST body | string | Có | `maxLength: 255`; tài liệu không khai báo `minLength` |
| `id` | Response `data` | integer | Có | Không khai báo range |
| `name` | Response `data` | string | Có | — |
| `email` | Response `data` | string | Có | OpenAPI không khai báo format tại resource này |
| `role` | Response `data` | integer | Có | — |
| `status` | Response `data` | integer | Có | — |
| `created_at` | Response `data` | string hoặc null | Có | Khi là string phải đúng `date-time` |

### Flows

- Happy: login → GET profile; login → POST cập nhật `name` → GET xác nhận dữ liệu.
- Negative: thiếu token, token sai, token hết hạn; thiếu/sai kiểu/vượt giới hạn `name`.
- Exception chưa đủ expected: 403, malformed JSON, sai Content-Type, payload quá lớn, rate limit.

### Ambiguities & QnA

| ID | Điểm chưa rõ | Ảnh hưởng |
|---|---|---|
| Q1 | `name` rỗng `""` hoặc chỉ gồm khoảng trắng có được chấp nhận không? | Chưa thể đặt expected cho min length/blank validation. |
| Q2 | Có trim khoảng trắng và hỗ trợ đầy đủ Unicode/emoji cho `name` không? | Chưa thể kết luận giá trị được lưu. |
| Q3 | Role/trạng thái tài khoản nào tạo được response 403? | Chưa có test data cụ thể để chạy nhánh 403. |
| Q4 | Nội dung chính xác của `message` và lỗi `errors.name` là gì? | Chỉ kiểm tra kiểu/cấu trúc, không kiểm tra exact text. |
| Q5 | Expected cho sai Content-Type, malformed JSON, payload quá lớn và rate limit là gì? | Không tự giả định 400/413/415/429. |
| Q6 | Thuộc tính ngoài schema như `role`, `status`, `email`, `id` phải bị bỏ qua hay trả lỗi? | Chưa thể kết luận mass assignment. |
| Q7 | SLA response time là bao nhiêu? | Chưa test Performance. |

### Assumptions

- A1: `{{expired_access_token}}` là fixture token hết hạn của một admin test, không phải secret hard-code.
- A2: Mỗi case cập nhật tên phải khôi phục `name` ban đầu trong teardown để test độc lập.
- Không dùng assumption để đặt expected cho Q1–Q7.

## 2. Test Viewpoint

| VP ID | Category | Viewpoint | Source | Technique | Risk |
|---|---|---|---|---|---|
| VP_APRO_001 | Functional | Lấy đúng profile của admin đang xác thực | REQ-APRO-02,03 | EP | Medium |
| VP_APRO_002 | Response | Response thành công tuân theo envelope và `ProfileResource` | REQ-APRO-02,03,05 | Schema Validation | High |
| VP_APRO_003 | Authorization | Từ chối request thiếu, sai hoặc hết hạn Bearer token | REQ-APRO-01,06 | Decision Table | High |
| VP_APRO_004 | Authorization | Từ chối principal đã xác thực nhưng không có quyền | REQ-APRO-06 | EP | High |
| VP_APRO_005 | Functional | Cập nhật profile với `name` hợp lệ | REQ-APRO-04,05 | EP | High |
| VP_APRO_006 | Input / Field | Từ chối request thiếu trường `name` | REQ-APRO-04,07 | EP | High |
| VP_APRO_007 | Input / Field | Từ chối `name` không phải string | REQ-APRO-04,07 | EP | Medium |
| VP_APRO_008 | Boundary | Xử lý đúng biên 255 và 256 ký tự của `name` | REQ-APRO-04,07 | BVA | High |
| VP_APRO_009 | Data Integrity | Giá trị cập nhật được lưu và đọc lại nhất quán | REQ-APRO-03,05 | State Verification | Medium |
| VP_APRO_010 | Security | Response không làm lộ credential hoặc token | REQ-APRO-02,03,05 | Error Guessing | High |
| VP_APRO_011 | Security | Chống mass assignment qua thuộc tính ngoài request schema | REQ-APRO-04 | Error Guessing | High |
| VP_APRO_012 | Error Handling | Xử lý protocol/payload exception theo contract | Q5 | Error Guessing | Medium |

Các nhóm không áp dụng: Combination, State Transition, UI/A11y, Compatibility và Integration vì contract hiện tại không mô tả các hành vi này. Performance chưa test do thiếu SLA.

## 3. Test Cases

> Quy ước chung: mọi response JSON đều kiểm tra `Content-Type` là `application/json`. Với response 422/401 chỉ kiểm tra cấu trúc và kiểu dữ liệu đã được OpenAPI định nghĩa; không kiểm tra exact message khi Q4 chưa được trả lời.

| TC ID | Module | VP ID | Risk Level | Test Scenario | Pre-Condition | Test Steps | Expected Result | Test Data | Priority | Automatable | Auto Type | Tags |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| WELVIE_ADMIN_PROFILE_TC_001 | Admin Profile API | VP_APRO_001 | Medium | Lấy profile bằng token hợp lệ | Login thành công; lưu `data.access_token` vào `{{access_token}}` | 1. Gửi GET `/api/admin/auth/me` với Bearer `{{access_token}}`.<br>2. Đối chiếu `data.id` và `data.email` với admin đã login. | 1. HTTP 200; response là JSON.<br>2. Trả đúng `id` và `email` của admin hiện tại. | Account fixture: `admin_profile_api_01@welvie.test`; token lấy động từ Login API | P0 | Yes | API | `@Smoke` `@CriticalPath` |
| WELVIE_ADMIN_PROFILE_TC_002 | Admin Profile API | VP_APRO_002 | High | Kiểm tra contract response GET thành công | Có `{{access_token}}` hợp lệ | 1. Gửi GET `/api/admin/auth/me`.<br>2. Validate envelope.<br>3. Validate `data` theo `ProfileResource`. | 1. HTTP 200.<br>2. Có đủ `message:string`, `errors:null`, `data:object`, `meta:object`.<br>3. Có đủ `id:integer`, `name:string`, `email:string`, `role:integer`, `status:integer`, `created_at:string(date-time) hoặc null`. | Bearer `{{access_token}}` | P0 | Yes | API | `@Smoke` `@Schema` |
| WELVIE_ADMIN_PROFILE_TC_003 | Admin Profile API | VP_APRO_003 | High | GET không có Authorization header | Không cần login | 1. Gửi GET `/api/admin/auth/me` không có Authorization header.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Không gửi header `Authorization` | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_004 | Admin Profile API | VP_APRO_003 | High | GET với token không hợp lệ | Không cần login | 1. Gửi GET với `Authorization: Bearer invalid.jwt.token`.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Token: `invalid.jwt.token` | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_005 | Admin Profile API | VP_APRO_003 | High | GET với token hết hạn | Có fixture `{{expired_access_token}}` | 1. Gửi GET với Bearer `{{expired_access_token}}`.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Token fixture đã hết hạn | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_006 | Admin Profile API | VP_APRO_005 | High | Cập nhật tên hợp lệ | Có token hợp lệ; đã lưu tên ban đầu | 1. Gửi POST `/api/admin/auth/me` với body `{"name":"Admin Profile API 01"}`.<br>2. Kiểm tra giá trị `data.name`. | 1. HTTP 200; response theo envelope thành công.<br>2. `data.name` bằng chính xác `Admin Profile API 01`. | `name`: `Admin Profile API 01` | P0 | Yes | API | `@Smoke` `@CriticalPath` |
| WELVIE_ADMIN_PROFILE_TC_007 | Admin Profile API | VP_APRO_002 | High | Kiểm tra contract response POST thành công | Có token hợp lệ; đã lưu tên ban đầu | 1. Gửi POST với `{"name":"Admin Schema Check 01"}`.<br>2. Validate envelope.<br>3. Validate `data` theo `ProfileResource`. | 1. HTTP 200.<br>2. Có đủ `message:string`, `errors:null`, `data:object`, `meta:object`.<br>3. `data` có đúng các field bắt buộc và kiểu dữ liệu của `ProfileResource`. | `name`: `Admin Schema Check 01` | P0 | Yes | API | `@Schema` `@Regression` |
| WELVIE_ADMIN_PROFILE_TC_008 | Admin Profile API | VP_APRO_008 | High | Cập nhật tên tại max length | Có token hợp lệ; đã lưu tên ban đầu | 1. Gửi POST với `name` gồm đúng 255 ký tự `A`.<br>2. Kiểm tra `data.name` và độ dài. | 1. HTTP 200.<br>2. `data.name` có đúng 255 ký tự `A`, không bị truncate. | `name`: chuỗi `A` lặp đúng 255 lần | P0 | Yes | API | `@Boundary` `@Regression` |
| WELVIE_ADMIN_PROFILE_TC_009 | Admin Profile API | VP_APRO_008 | High | Từ chối tên vượt max length | Có token hợp lệ | 1. Gửi POST với `name` gồm đúng 256 ký tự `A`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `message:string`, `errors:object` chứa lỗi gắn với `name`, `data:null`, `meta:object`. | `name`: chuỗi `A` lặp đúng 256 lần | P0 | Yes | API | `@Boundary` `@Negative` |
| WELVIE_ADMIN_PROFILE_TC_010 | Admin Profile API | VP_APRO_006 | High | Từ chối body thiếu `name` | Có token hợp lệ | 1. Gửi POST với body `{}`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `message:string`, `errors:object` chứa lỗi gắn với `name`, `data:null`, `meta:object`. | Body: `{}` | P0 | Yes | API | `@Negative` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_011 | Admin Profile API | VP_APRO_007 | Medium | Từ chối `name` là null | Có token hợp lệ | 1. Gửi POST với `{"name":null}`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `message:string`, `errors:object` chứa lỗi gắn với `name`, `data:null`, `meta:object`. | `name`: `null` | P1 | Yes | API | `@Negative` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_012 | Admin Profile API | VP_APRO_007 | Medium | Từ chối `name` là số | Có token hợp lệ | 1. Gửi POST với `{"name":12345}`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `message:string`, `errors:object` chứa lỗi gắn với `name`, `data:null`, `meta:object`. | `name`: `12345` (JSON number) | P1 | Yes | API | `@Negative` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_013 | Admin Profile API | VP_APRO_003 | High | POST không có Authorization header | Không cần login | 1. Gửi POST với body hợp lệ nhưng không có Authorization header.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Body: `{"name":"Unauthorized Update"}` | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_014 | Admin Profile API | VP_APRO_003 | High | POST với token không hợp lệ | Không cần login | 1. Gửi POST với Bearer `invalid.jwt.token` và body hợp lệ.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Token: `invalid.jwt.token`; body: `{"name":"Invalid Token Update"}` | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_015 | Admin Profile API | VP_APRO_003 | High | POST với token hết hạn | Có fixture `{{expired_access_token}}` | 1. Gửi POST với Bearer `{{expired_access_token}}` và body hợp lệ.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Token fixture đã hết hạn; body: `{"name":"Expired Token Update"}` | P0 | Yes | API | `@Negative` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_016 | Admin Profile API | VP_APRO_009 | Medium | Dữ liệu cập nhật được lưu và đọc lại | Có token hợp lệ; đã lưu tên ban đầu | 1. GET và lưu `id`, `email`, `role`, `status`, `created_at` hiện tại.<br>2. POST `{"name":"Admin Persistence 01"}`.<br>3. GET profile lần nữa.<br>4. So sánh profile trước và sau. | 1. GET trả HTTP 200 và lưu được dữ liệu gốc.<br>2. POST trả HTTP 200, `data.name` = `Admin Persistence 01`.<br>3. GET trả HTTP 200, `data.name` = `Admin Persistence 01`.<br>4. `id`, `email`, `role`, `status`, `created_at` không bị thay đổi bởi request chỉ cập nhật tên. | `name`: `Admin Persistence 01` | P0 | Yes | API | `@Regression` `@DataIntegrity` |
| WELVIE_ADMIN_PROFILE_TC_017 | Admin Profile API | VP_APRO_010 | High | GET không làm lộ dữ liệu nhạy cảm | Có token hợp lệ | 1. Gửi GET profile.<br>2. Duyệt toàn bộ key trong response để tìm dữ liệu nhạy cảm. | 1. HTTP 200.<br>2. Response không chứa key `password`, `password_hash`, `access_token`, `refresh_token`, `secret`, `secretKey`. | Danh sách key cấm như Expected Result | P0 | Yes | API | `@Security` `@Regression` |
| WELVIE_ADMIN_PROFILE_TC_018 | Admin Profile API | VP_APRO_010 | High | POST không làm lộ dữ liệu nhạy cảm | Có token hợp lệ; đã lưu tên ban đầu | 1. Gửi POST với `{"name":"Admin Sensitive Check 01"}`.<br>2. Duyệt toàn bộ key trong response để tìm dữ liệu nhạy cảm. | 1. HTTP 200.<br>2. Response không chứa key `password`, `password_hash`, `access_token`, `refresh_token`, `secret`, `secretKey`. | `name`: `Admin Sensitive Check 01`; danh sách key cấm như Expected Result | P0 | Yes | API | `@Security` `@Regression` |
| WELVIE_ADMIN_PROFILE_TC_019 | Admin Profile API | VP_APRO_003 | High | GET với Bearer token rỗng | Không cần login | 1. Gửi GET với header `Authorization: Bearer`.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Header: `Authorization: Bearer` | P0 | Yes | API | `@Abnormal` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_020 | Admin Profile API | VP_APRO_003 | High | GET dùng Basic auth thay cho Bearer | Không cần login | 1. Gửi GET với `Authorization: Basic Zm9vOmJhcg==`.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Basic credential mã hóa từ `foo:bar` | P0 | Yes | API | `@Abnormal` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_021 | Admin Profile API | VP_APRO_003 | High | GET chỉ truyền token qua query | Có `{{access_token}}` hợp lệ | 1. Gửi GET `?access_token={{access_token}}` nhưng không có Authorization header.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Query: `access_token={{access_token}}` | P0 | Yes | API | `@Abnormal` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_022 | Admin Profile API | VP_APRO_003 | High | POST với Bearer token rỗng | Không cần login | 1. Gửi POST body `{"name":"Empty Bearer Update"}` với `Authorization: Bearer`.<br>2. Validate error envelope. | 1. HTTP 401.<br>2. Có đủ `message:string`, `errors:null`, `data:null`, `meta:object`. | Header: `Authorization: Bearer`; name: `Empty Bearer Update` | P0 | Yes | API | `@Abnormal` `@Security` |
| WELVIE_ADMIN_PROFILE_TC_023 | Admin Profile API | VP_APRO_007 | Medium | Từ chối `name` là boolean | Có token hợp lệ | 1. Gửi POST với `{"name":true}`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `errors:object` chứa lỗi gắn với `name`; `data:null`. | `name`: `true` (JSON boolean) | P1 | Yes | API | `@Abnormal` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_024 | Admin Profile API | VP_APRO_007 | Medium | Từ chối `name` là array | Có token hợp lệ | 1. Gửi POST với `{"name":["Admin"]}`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `errors:object` chứa lỗi gắn với `name`; `data:null`. | `name`: `["Admin"]` | P1 | Yes | API | `@Abnormal` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_025 | Admin Profile API | VP_APRO_007 | Medium | Từ chối `name` là object | Có token hợp lệ | 1. Gửi POST với `{"name":{"value":"Admin"}}`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `errors:object` chứa lỗi gắn với `name`; `data:null`. | `name`: `{"value":"Admin"}` | P1 | Yes | API | `@Abnormal` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_026 | Admin Profile API | VP_APRO_006 | High | Từ chối body JSON null | Có token hợp lệ | 1. Gửi POST với body JSON `null`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `errors:object` chứa lỗi required của `name`; `data:null`. | Body: `null` | P0 | Yes | API | `@Abnormal` `@Validation` |
| WELVIE_ADMIN_PROFILE_TC_027 | Admin Profile API | VP_APRO_006 | High | Từ chối request body rỗng | Có token hợp lệ | 1. Gửi POST với body rỗng và `Content-Type: application/json`.<br>2. Validate validation envelope. | 1. HTTP 422.<br>2. `errors:object` chứa lỗi required của `name`; `data:null`. | Raw body: chuỗi rỗng | P0 | Yes | API | `@Abnormal` `@Validation` |

### Teardown

1. Gửi POST `/api/admin/auth/me` bằng `{{access_token}}` để khôi phục chính xác tên đã lưu trước test.
2. GET lại profile và xác nhận tên đã được khôi phục.
3. Không ghi token hoặc credential vào source code/report.

## 4. Viewpoint Coverage

| VP ID | Category | Viewpoint | Risk | TC IDs | # TCs | Status |
|---|---|---|---|---|---:|---|
| VP_APRO_001 | Functional | Lấy đúng profile admin hiện tại | Medium | TC_001 | 1 | ✅ Covered |
| VP_APRO_002 | Response | Contract response thành công | High | TC_002, TC_007 | 2 | ✅ Covered |
| VP_APRO_003 | Authorization | Thiếu/sai/hết hạn token | High | TC_003, TC_004, TC_005, TC_013, TC_014, TC_015, TC_019, TC_020, TC_021, TC_022 | 10 | ✅ Covered |
| VP_APRO_004 | Authorization | Principal không có quyền | High | — | 0 | ❌ GAP — Q3 |
| VP_APRO_005 | Functional | Cập nhật tên hợp lệ | High | TC_006, TC_008 | 2 | ✅ Covered |
| VP_APRO_006 | Input / Field | Thiếu `name` | High | TC_010, TC_026, TC_027 | 3 | ✅ Covered |
| VP_APRO_007 | Input / Field | Sai kiểu `name` | Medium | TC_011, TC_012, TC_023, TC_024, TC_025 | 5 | ✅ Covered |
| VP_APRO_008 | Boundary | Biên 255/256 ký tự | High | TC_008, TC_009 | 2 | ✅ Covered |
| VP_APRO_009 | Data Integrity | Lưu và đọc lại nhất quán | Medium | TC_016 | 1 | ✅ Covered |
| VP_APRO_010 | Security | Không lộ dữ liệu nhạy cảm | High | TC_017, TC_018 | 2 | ✅ Covered |
| VP_APRO_011 | Security | Mass assignment | High | — | 0 | ❌ GAP — Q6 |
| VP_APRO_012 | Error Handling | Protocol/payload exception | Medium | — | 0 | ❌ GAP — Q5 |

Coverage hiện tại: **9/12 viewpoint Covered = 75%**. Chưa đạt cổng High/Medium 100% vì API document thiếu expected và test data cho VP_APRO_004, VP_APRO_011, VP_APRO_012. Không có test case orphan.

## N/A theo API document

- Duplicate/idempotency: POST được dùng làm update nhưng OpenAPI không cam kết tính idempotent.
- Performance: chưa có SLA.
- Destructive security/stress/rate-limit: không thực thi khi chưa được cho phép và chưa có expected.
