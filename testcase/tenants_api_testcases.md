# API Test Design — Tenants

## 1. Requirement Analysis

Nguồn: `basic design/document.json` — OpenAPI 3.1.0.

| REQ ID | Phạm vi |
|---|---|
| REQ-TN-01 | CRUD tenant: list/create/show/update/delete. |
| REQ-TN-02 | Lock/Unlock tenant. |
| REQ-TN-03 | Export CSV, status summary, tenant options và parent options. |
| REQ-TN-04 | Đếm phân công CS và bàn giao toàn bộ tenant giữa hai CS. |
| REQ-TN-05 | Create/update/change-password/delete Tenant Administrator. |
| REQ-TN-06 | Gửi billing notice. |
| REQ-TN-07 | Thêm service type/branch cho tenant. |
| REQ-TN-08 | API dùng Bearer token và các response lỗi 401/403/404/422 theo từng endpoint. |

### Dữ liệu chính theo contract

- Create Tenant bắt buộc: `name`, `type`, `status`, `prefecture_id`, `address`, `staff_limit`, `customer_support_admin_id`, `service_type_ids`, `tenant_administrator`, `branches`.
- `name` tối đa 255; `address` tối đa 500; `staff_limit >= 1`; `service_type_ids` tối thiểu 1 phần tử.
- Tenant Administrator: `name`, email đúng format, password 8–255 ký tự.
- Lock reason enum `1..3`; billing reason enum `1..3`; `unpaid_months` từ 1–12 phần tử.
- Collection lấy CS ID và Service Type ID từ API setup, không hard-code master ID.

### QnA / Assumption

| ID | Nội dung |
|---|---|
| TN-Q01 | Bàn giao CS tác động nhiều tenant; positive case chỉ chạy khi người dùng chủ động điền `handover_from_cs_id` và `handover_to_cs_id`. |
| TN-Q02 | Exact error message không được tài liệu định nghĩa; chỉ assert status và schema lỗi. |
| TN-Q03 | Không có SLA, rule unique tenant name, rule sort, encoding CSV chi tiết ngoài `text/csv; charset=UTF-8`. |
| TN-Q04 | Billing notice có thể tạo side effect gửi mail; collection dùng tenant/admin test với email động `@example.com`. |
| TN-A01 | `admin_email` là super admin để có quyền delete tenant; nếu không, case delete có thể trả 403 đúng phân quyền. |

## 2. Test Viewpoint

| VP ID | Category | Viewpoint | Source | Technique | Risk |
|---|---|---|---|---|---|
| VP_TN_001 | Functional | List/filter tenant | REQ-TN-01 | EP | Medium |
| VP_TN_002 | Functional | Create/show/update/delete tenant test | REQ-TN-01 | State Transition | High |
| VP_TN_003 | Input/Boundary | Validation create/update tenant | REQ-TN-01 | EP/BVA | High |
| VP_TN_004 | Authorization | Request thiếu token bị từ chối | REQ-TN-08 | EP | High |
| VP_TN_005 | Error Handling | Resource không tồn tại trả 404 | REQ-TN-08 | EP | Medium |
| VP_TN_006 | Functional | Options/status summary/parent options | REQ-TN-03 | EP | Medium |
| VP_TN_007 | Export | Export CSV đúng contract | REQ-TN-03 | Schema | Medium |
| VP_TN_008 | Functional | Count và handover CS | REQ-TN-04 | EP | High |
| VP_TN_009 | Functional | CRUD Tenant Administrator | REQ-TN-05 | State Transition | High |
| VP_TN_010 | Input/Boundary | Validation Tenant Administrator | REQ-TN-05 | EP/BVA | High |
| VP_TN_011 | State | Lock/Unlock tenant | REQ-TN-02 | State Transition | High |
| VP_TN_012 | Input | Validation lock | REQ-TN-02 | EP | High |
| VP_TN_013 | Functional | Gửi billing notice | REQ-TN-06 | EP | High |
| VP_TN_014 | Input/Boundary | Validation billing notice | REQ-TN-06 | EP/BVA | High |
| VP_TN_015 | Functional | Gán service type/branch | REQ-TN-07 | EP | High |
| VP_TN_016 | Input | Validation service type/branch | REQ-TN-07 | EP | High |
| VP_TN_017 | Data Integrity | Thay đổi được phản ánh qua GET | REQ-TN-01,05 | State Verification | High |

## 3. Test Cases

| TC ID | VP ID | Scenario | Test Data | Expected |
|---|---|---|---|---|
| WELVIE_TENANTS_TC_001 | VP_TN_001 | GET list tenant | Không filter | 200; list envelope đúng schema |
| WELVIE_TENANTS_TC_002 | VP_TN_001 | GET list với `status=1&page=1` | `status=1`, `page=1` | 200 |
| WELVIE_TENANTS_TC_003 | VP_TN_003 | Page dưới minimum | `page=0` | 422; `errors.page` |
| WELVIE_TENANTS_TC_004 | VP_TN_004 | GET list không token | No auth | 401 |
| WELVIE_TENANTS_TC_005 | VP_TN_002 | Create tenant động | type=1, status=1, prefecture=13, staff_limit=10 | 200; lưu `tenant_id` |
| WELVIE_TENANTS_TC_006 | VP_TN_003 | Create thiếu name | Bỏ `name` | 422; `errors.name` |
| WELVIE_TENANTS_TC_007 | VP_TN_003 | Create staff_limit=0 | `staff_limit=0` | 422; `errors.staff_limit` |
| WELVIE_TENANTS_TC_008 | VP_TN_003 | Create service_type_ids rỗng | `[]` | 422; `errors.service_type_ids` |
| WELVIE_TENANTS_TC_009 | VP_TN_002 | Show tenant vừa tạo | `tenant_id` | 200; đúng ID/name |
| WELVIE_TENANTS_TC_010 | VP_TN_005 | Show tenant không tồn tại | ID `2147483647` | 404 |
| WELVIE_TENANTS_TC_011 | VP_TN_002 | Update tenant | name thêm `_updated`, staff_limit=20 | 200 |
| WELVIE_TENANTS_TC_012 | VP_TN_003 | Update name 256 ký tự | 256 ký tự `A` | 422; `errors.name` |
| WELVIE_TENANTS_TC_013 | VP_TN_005 | Update tenant không tồn tại | ID `2147483647` | 404 |
| WELVIE_TENANTS_TC_014 | VP_TN_017 | GET xác nhận update | `tenant_id` | 200; name/staff_limit đã cập nhật |
| WELVIE_TENANTS_TC_015 | VP_TN_006 | GET tenant options | — | 200; item có id/name |
| WELVIE_TENANTS_TC_016 | VP_TN_006 | GET status summary | — | 200; các count là integer |
| WELVIE_TENANTS_TC_017 | VP_TN_006 | Parent options hợp lệ | `child_type=3` | 200 |
| WELVIE_TENANTS_TC_018 | VP_TN_006 | Parent options enum sai | `child_type=1` | 422 |
| WELVIE_TENANTS_TC_019 | VP_TN_007 | Export tenant CSV | `status=1` | 200; Content-Type CSV; Content-Disposition tồn tại |
| WELVIE_TENANTS_TC_020 | VP_TN_007 | Export status sai | `status=99` | 422 |
| WELVIE_TENANTS_TC_021 | VP_TN_008 | Count assignment theo CS | CS ID lấy từ setup | 200; tenant_count integer |
| WELVIE_TENANTS_TC_022 | VP_TN_008 | Count thiếu CS ID | Không query | 422 |
| WELVIE_TENANTS_TC_023 | VP_TN_008 | Handover CS có chủ đích | Hai ID từ environment | 200; mặc định skip |
| WELVIE_TENANTS_TC_024 | VP_TN_008 | Handover thiếu to ID | Chỉ có from ID | 422 |
| WELVIE_TENANTS_TC_025 | VP_TN_009 | Create admin thứ hai | Email động `@example.com`, password `Tenant123!` | 200 |
| WELVIE_TENANTS_TC_026 | VP_TN_017 | GET capture admin thứ hai | Match email động | 200; lưu `tenant_admin_id` |
| WELVIE_TENANTS_TC_027 | VP_TN_010 | Create admin email sai | `invalid-email` | 422; `errors.email` |
| WELVIE_TENANTS_TC_028 | VP_TN_009 | Update admin thứ hai | Tên thêm `_updated` | 200 |
| WELVIE_TENANTS_TC_029 | VP_TN_010 | Password admin 7 ký tự | `1234567` | 422; `errors.password` |
| WELVIE_TENANTS_TC_030 | VP_TN_009 | Change password hợp lệ | `Tenant456!` | 200 |
| WELVIE_TENANTS_TC_031 | VP_TN_009 | Delete admin thứ hai | `tenant_admin_id` | 200 |
| WELVIE_TENANTS_TC_032 | VP_TN_011 | Lock tenant | reason=1, không gửi email | 200 |
| WELVIE_TENANTS_TC_033 | VP_TN_012 | Lock thiếu reason | `{}` | 422 |
| WELVIE_TENANTS_TC_034 | VP_TN_011 | Unlock tenant | không gửi email | 200 |
| WELVIE_TENANTS_TC_035 | VP_TN_005 | Unlock tenant không tồn tại | ID `2147483647` | 404 |
| WELVIE_TENANTS_TC_036 | VP_TN_013 | Gửi billing notice | reason=1, content cụ thể, tháng `2026-10` | 200 |
| WELVIE_TENANTS_TC_037 | VP_TN_014 | Billing tháng rỗng | `unpaid_months=[]` | 422 |
| WELVIE_TENANTS_TC_038 | VP_TN_014 | Billing content 1001 ký tự | 1001 ký tự `B` | 422 |
| WELVIE_TENANTS_TC_039 | VP_TN_005 | Billing tenant không tồn tại | ID `2147483647` | 404 |
| WELVIE_TENANTS_TC_040 | VP_TN_015 | Thêm service type/branch | branch động, service ID setup | 200 |
| WELVIE_TENANTS_TC_041 | VP_TN_016 | Thiếu branch_name | Chỉ service_type_id | 422 |
| WELVIE_TENANTS_TC_042 | VP_TN_005 | Add service type tenant không tồn tại | ID `2147483647` | 404 |
| WELVIE_TENANTS_TC_043 | VP_TN_002 | Delete tenant test | `tenant_id` | 200 |
| WELVIE_TENANTS_TC_044 | VP_TN_017 | Show tenant sau delete | ID đã xóa | 404 |

Mỗi request trong Postman có assertion status, content type và envelope tương ứng. Các case ghi dữ liệu chỉ dùng tenant động; fallback cleanup chấp nhận 200/404.

## 4. Viewpoint Coverage

| VP ID | TC IDs | Status |
|---|---|---|
| VP_TN_001 | 001,002 | ✅ |
| VP_TN_002 | 005,009,011,043 | ✅ |
| VP_TN_003 | 003,006,007,008,012 | ✅ |
| VP_TN_004 | 004 | ✅ |
| VP_TN_005 | 010,013,035,039,042 | ✅ |
| VP_TN_006 | 015–018 | ✅ |
| VP_TN_007 | 019,020 | ✅ |
| VP_TN_008 | 021–024 | ⚠️ Positive handover cần opt-in |
| VP_TN_009 | 025,028,030,031 | ✅ |
| VP_TN_010 | 027,029 | ✅ |
| VP_TN_011 | 032,034 | ✅ |
| VP_TN_012 | 033 | ✅ |
| VP_TN_013 | 036 | ✅ |
| VP_TN_014 | 037,038 | ✅ |
| VP_TN_015 | 040 | ✅ |
| VP_TN_016 | 041 | ✅ |
| VP_TN_017 | 014,026,044 | ✅ |

Coverage thiết kế: **16 Covered, 1 Weak/opt-in**. Không có orphan test case.
