# A013 — Test case quản lý Admin Users

Nguồn: đặc tả người dùng cung cấp ngày 2026-11-09 và ảnh màn hình `Users`, `Add user`, `Edit user`.

## 1. Phân tích requirement

### 1.1 Phạm vi

- Actor: Admin đang có quyền quản lý người dùng.
- Chức năng: tạo người dùng, sửa tên/quyền, đổi mật khẩu, khóa/mở khóa, đăng nhập và kiểm tra không lộ mật khẩu.
- Ba role quan sát được trên UI: `Administrator`, `General`, `View`.
- Ngoài phạm vi: khóa, đổi mật khẩu hoặc hạ quyền tài khoản đang dùng để vận hành.

### 1.2 Requirement và Acceptance Criteria

| ID | Nội dung |
|---|---|
| REQ-01 | Admin tạo được người dùng mới với Display name, Email, Role, Initial password và Password confirmation. |
| AC-01 | Người dùng được tạo hợp lệ đăng nhập được với đúng role đã chọn. |
| AC-02 | Email trùng không tạo được người dùng và form hiển thị lỗi ngay. |
| AC-03 | Display name trống không tạo được người dùng và form hiển thị lỗi ngay. |
| AC-04 | Password confirmation không khớp không tạo được người dùng và form hiển thị lỗi ngay. |
| REQ-02 | Admin sửa được tên của tài khoản thử. |
| REQ-03 | Admin sửa được role của tài khoản thử. |
| AC-05 | Admin không thể tự thay đổi role của chính mình. |
| REQ-04 | Admin đổi được mật khẩu tài khoản thử; tài khoản đăng nhập được bằng mật khẩu mới. |
| REQ-05 | Admin khóa và mở khóa được tài khoản thử. |
| AC-06 | Tài khoản bị khóa không đăng nhập được. |
| AC-07 | Tài khoản đã mở khóa đăng nhập lại được. |
| AC-08 | Mật khẩu không hiển thị trên danh sách người dùng hoặc audit log. |
| SAFE-01 | Không khóa, đổi mật khẩu hoặc hạ quyền bất kỳ tài khoản vận hành nào. |

### 1.3 Field specification

| Field | Bắt buộc | Rule đã nêu | Điểm chưa nêu |
|---|---:|---|---|
| Display name | Có | Không được để trống | Min/max length, trim khoảng trắng, ký tự cho phép |
| Email | Có | Phải duy nhất | Format, max length, phân biệt hoa/thường khi kiểm tra trùng |
| Role | Có | Một trong `Administrator`, `General`, `View` | Ma trận quyền cụ thể của từng role |
| Initial password | Có | Phải khớp Password confirmation | Chính sách độ dài/độ phức tạp |
| Password confirmation | Có | Phải khớp Initial password | Message lỗi chính xác |

### 1.4 Luồng

- Happy path: tạo user → đăng nhập → sửa tên/role → đổi mật khẩu → đăng nhập bằng mật khẩu mới → khóa → đăng nhập bị từ chối → mở khóa → đăng nhập thành công.
- Exception path: email trùng; tên trống; hai mật khẩu không khớp.
- Authorization path: Admin mở trang sửa chính mình nhưng không thể tự đổi role.
- Security path: giá trị mật khẩu không xuất hiện trên danh sách và audit log.

### 1.5 Cần làm rõ

| ID | Câu hỏi | Ảnh hưởng |
|---|---|---|
| Q-A013-01 | Nội dung chính xác của từng validation message là gì? | Chỉ có thể kiểm tra đúng ý nghĩa lỗi, chưa thể đối chiếu từng ký tự. |
| Q-A013-02 | Ma trận menu/action của `Administrator`, `General`, `View` là gì? | Chưa thể xác nhận đầy đủ “đúng quyền”; chỉ xác nhận role lưu/hiển thị đúng và đăng nhập thành công. |
| Q-A013-03 | Khi Admin tự sửa mình, ô Role phải ẩn, disabled hay backend từ chối khi submit? | Expected hiện chỉ khẳng định role không được thay đổi. |
| Q-A013-04 | Message khi tài khoản bị khóa đăng nhập là gì? | Chỉ kiểm tra đăng nhập bị từ chối, chưa đối chiếu message chính xác. |
| Q-A013-05 | Audit log nằm ở đâu và phải ghi những event/field nào cho create, edit, password change, lock/unlock? | Cần để chuẩn bị quyền và đường dẫn trước khi chạy TC audit. |

### 1.6 Assumption và guardrail thực thi

- Chỉ dùng các tài khoản QA bên dưới. Không thực hiện lock, password change hoặc role downgrade với `mai.hoang@10kn.io` hay tài khoản vận hành khác.
- Với chính tài khoản Admin đang đăng nhập, chỉ mở trang Edit và quan sát/kiểm tra khả năng đổi Role; không lưu thay đổi cho field khác.
- `SEC-A013-V1` và `SEC-A013-V2` là hai secret khác nhau lấy từ kho bí mật tại thời điểm chạy; không ghi giá trị mật khẩu vào repo, evidence hoặc audit.
- Trước mỗi TC, đưa tài khoản QA về đúng trạng thái ghi trong Pre-Condition bằng cơ chế cleanup đã được phê duyệt.

## 2. Test Viewpoint

| VP ID | Category | Viewpoint | Source | Technique | Risk |
|---|---|---|---|---|---|
| VP_A013_001 | Functional | Tạo user hợp lệ và đăng nhập thành công | REQ-01, AC-01 | EP | High |
| VP_A013_002 | Authorization | User sau khi tạo có đúng role đã chọn | AC-01 | Decision Table | High |
| VP_A013_003 | Data Integrity | Email đã tồn tại không được tạo trùng | AC-02 | EP | High |
| VP_A013_004 | Input / Field | Display name trống bị từ chối ngay trên form | AC-03 | EP | Medium |
| VP_A013_005 | Combination | Initial password và confirmation khác nhau bị từ chối | AC-04 | Decision Table | High |
| VP_A013_006 | Data Integrity | Tên mới được lưu đúng vào tài khoản thử | REQ-02 | EP | Medium |
| VP_A013_007 | Authorization | Role mới được lưu đúng vào tài khoản thử | REQ-03 | State Transition | High |
| VP_A013_008 | Functional | Đổi mật khẩu tài khoản thử thành công | REQ-04 | EP | High |
| VP_A013_009 | Integration | Tài khoản đăng nhập được bằng mật khẩu mới | REQ-04 | EP | High |
| VP_A013_010 | State | Tài khoản thử chuyển từ Active sang Disabled | REQ-05 | State Transition | High |
| VP_A013_011 | Authorization | Tài khoản Disabled không đăng nhập được | AC-06 | State Transition | High |
| VP_A013_012 | State | Tài khoản thử chuyển từ Disabled sang Active | REQ-05 | State Transition | High |
| VP_A013_013 | Authorization | Tài khoản được mở khóa đăng nhập lại được | AC-07 | State Transition | High |
| VP_A013_014 | Authorization | Admin không tự thay đổi được role của chính mình | AC-05 | Negative Authorization | High |
| VP_A013_015 | Security | Mật khẩu không xuất hiện trên danh sách Users | AC-08 | Error Guessing | High |
| VP_A013_016 | Security | Mật khẩu không xuất hiện trong audit log | AC-08 | Error Guessing | High |

Các nhóm Boundary, Compatibility, Performance và Resilience chưa sinh viewpoint vì đặc tả không có giới hạn field, browser, SLA hoặc yêu cầu xử lý double-submit. Không tự bổ sung rule ngoài nguồn.

## 3. Test data dùng chung

| ID | Display name | Email | Role/State ban đầu | Mục đích |
|---|---|---|---|---|
| U-GENERAL | QA A013 General | `qa.a013.general.20261109@10kn.io` | Chưa tồn tại | Tạo role General |
| U-VIEW | QA A013 View | `qa.a013.view.20261109@10kn.io` | Chưa tồn tại | Tạo role View |
| U-ADMIN | QA A013 Administrator | `qa.a013.administrator.20261109@10kn.io` | Chưa tồn tại | Tạo role Administrator |
| U-EXISTING | QA A013 Existing | `qa.a013.existing.20261109@10kn.io` | General / Active | Email trùng |
| U-LIFECYCLE | QA A013 Lifecycle | `qa.a013.lifecycle.20261109@10kn.io` | General / Active | Edit, password, lock/unlock |
| ADMIN-CURRENT | Hoang Thi Mai | `mai.hoang@10kn.io` | Administrator / Active | Chỉ kiểm tra self-edit |

Mật khẩu:

- `SEC-A013-V1`: secret ban đầu của các tài khoản QA.
- `SEC-A013-V2`: secret mới của `U-LIFECYCLE`, bắt buộc khác `SEC-A013-V1`.

## 4. Test Cases

### WELVIE_A013_TC_001 — Tạo và đăng nhập user role General

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_001 |
| Risk Level | High |
| Pre-Condition | Admin đã đăng nhập; `U-GENERAL` chưa tồn tại; có quyền tạo user. |
| Test Data | `QA A013 General`; `qa.a013.general.20261109@10kn.io`; Role `General`; secret `SEC-A013-V1`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Smoke @Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở Users và chọn Add user. | Form Add user hiển thị. |
| 2 | Nhập Display name, Email, chọn `General`, nhập `SEC-A013-V1` vào hai ô mật khẩu rồi chọn Create user. | User được tạo một lần và xuất hiện trong danh sách với role `General`, status `Active`. |
| 3 | Đăng xuất Admin, đăng nhập bằng email của `U-GENERAL` và `SEC-A013-V1`. | Đăng nhập thành công bằng tài khoản vừa tạo. |

### WELVIE_A013_TC_002 — Tạo và đăng nhập user role View

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_001 |
| Risk Level | High |
| Pre-Condition | Admin đã đăng nhập; `U-VIEW` chưa tồn tại; có quyền tạo user. |
| Test Data | `QA A013 View`; `qa.a013.view.20261109@10kn.io`; Role `View`; secret `SEC-A013-V1`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Smoke @Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở form Add user. | Form Add user hiển thị. |
| 2 | Nhập test data, chọn `View`, nhập cùng `SEC-A013-V1` vào hai ô mật khẩu và tạo user. | User được tạo một lần, hiển thị role `View` và status `Active`. |
| 3 | Đăng xuất Admin và đăng nhập bằng `U-VIEW`. | Đăng nhập thành công bằng tài khoản vừa tạo. |

### WELVIE_A013_TC_003 — Tạo và đăng nhập user role Administrator

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_001 |
| Risk Level | High |
| Pre-Condition | Admin đã đăng nhập; `U-ADMIN` chưa tồn tại; có quyền tạo user. |
| Test Data | `QA A013 Administrator`; `qa.a013.administrator.20261109@10kn.io`; Role `Administrator`; secret `SEC-A013-V1`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Smoke @Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở form Add user. | Form Add user hiển thị. |
| 2 | Nhập test data, chọn `Administrator`, nhập cùng `SEC-A013-V1` vào hai ô mật khẩu và tạo user. | User được tạo một lần, hiển thị role `Administrator` và status `Active`. |
| 3 | Đăng xuất Admin và đăng nhập bằng `U-ADMIN`. | Đăng nhập thành công bằng tài khoản vừa tạo. |

### WELVIE_A013_TC_004 — Role được lưu đúng theo lựa chọn

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_002 |
| Risk Level | High |
| Pre-Condition | `U-GENERAL`, `U-VIEW`, `U-ADMIN` đã được tạo và Active. |
| Test Data | Ba account QA tương ứng ba role. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Đăng nhập lần lượt từng account QA và mở thông tin tài khoản/menu được cấp. | Mỗi account hiển thị đúng role đã chọn khi tạo. Việc đối chiếu menu/action chi tiết bị chặn bởi Q-A013-02. |

### WELVIE_A013_TC_005 — Không tạo user khi email trùng

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_003 |
| Risk Level | High |
| Pre-Condition | `U-EXISTING` đã tồn tại và Active; Admin đã đăng nhập. |
| Test Data | Display name `QA A013 Duplicate`; email `qa.a013.existing.20261109@10kn.io`; Role `General`; secret `SEC-A013-V1`. |
| Priority | P0 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở Add user, nhập toàn bộ test data với email của `U-EXISTING`, rồi chọn Create user. | Form vẫn hiển thị và báo lỗi email đã tồn tại ngay trên form. |
| 2 | Quay lại/tìm trong Users theo email `qa.a013.existing.20261109@10kn.io`. | Chỉ có bản ghi `U-EXISTING`; không có user mới `QA A013 Duplicate`. |

### WELVIE_A013_TC_006 — Không tạo user khi Display name trống

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_004 |
| Risk Level | Medium |
| Pre-Condition | Admin đã đăng nhập; email test chưa tồn tại. |
| Test Data | Display name để trống; email `qa.a013.blank-name.20261109@10kn.io`; Role `General`; secret `SEC-A013-V1`. |
| Priority | P0 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở Add user, để trống Display name, nhập các field còn lại rồi chọn Create user. | Form vẫn hiển thị và báo lỗi Display name bắt buộc ngay trên form. |
| 2 | Tìm email `qa.a013.blank-name.20261109@10kn.io` trong Users. | Không tồn tại user mang email này. |

### WELVIE_A013_TC_007 — Không tạo user khi mật khẩu xác nhận không khớp

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_005 |
| Risk Level | High |
| Pre-Condition | Admin đã đăng nhập; email test chưa tồn tại. |
| Test Data | Display name `QA A013 Password Mismatch`; email `qa.a013.password-mismatch.20261109@10kn.io`; Role `General`; Initial password `SEC-A013-V1`; confirmation `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở Add user, nhập test data với hai secret khác nhau rồi chọn Create user. | Form vẫn hiển thị và báo lỗi hai mật khẩu không khớp ngay trên form. |
| 2 | Tìm email `qa.a013.password-mismatch.20261109@10kn.io` trong Users. | Không tồn tại user mang email này. |

### WELVIE_A013_TC_008 — Sửa Display name của tài khoản thử

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_006 |
| Risk Level | Medium |
| Pre-Condition | `U-LIFECYCLE` tồn tại với tên `QA A013 Lifecycle`, role `General`, status `Active`; Admin đã đăng nhập. |
| Test Data | Tên mới `QA A013 Lifecycle Updated`. |
| Priority | P1 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Từ Users, mở Edit của `U-LIFECYCLE`. | Form Edit hiển thị đúng email và tên hiện tại. |
| 2 | Đổi Display name thành `QA A013 Lifecycle Updated` và Save changes. | Lưu thành công và quay lại/hiển thị trạng thái thành công. |
| 3 | Quan sát bản ghi `U-LIFECYCLE` trong Users và mở lại Edit. | Cả danh sách và form Edit hiển thị `QA A013 Lifecycle Updated`; email không đổi. |

### WELVIE_A013_TC_009 — Sửa role của tài khoản thử

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_007 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` tồn tại với role `General`, status `Active`; Admin đã đăng nhập. |
| Test Data | Role mới `View`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở Edit của `U-LIFECYCLE`. | Form Edit hiển thị role hiện tại `General`. |
| 2 | Chọn role `View` và Save changes. | Lưu thành công. |
| 3 | Mở lại bản ghi `U-LIFECYCLE`. | Danh sách và form Edit hiển thị role `View`; việc xác nhận menu/action chi tiết chờ Q-A013-02. |

### WELVIE_A013_TC_010 — Đổi mật khẩu tài khoản thử

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_008 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` Active và đang dùng `SEC-A013-V1`; Admin đã đăng nhập. |
| Test Data | Mật khẩu mới `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Regression @CriticalPath @Security` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Tại Users, chọn action đổi mật khẩu của `U-LIFECYCLE`. | Form/dialog đổi mật khẩu của đúng account hiển thị. |
| 2 | Nhập `SEC-A013-V2` theo các field được yêu cầu và xác nhận đổi mật khẩu. | Hệ thống báo đổi mật khẩu thành công và không hiển thị lại giá trị secret. |

### WELVIE_A013_TC_011 — Đăng nhập bằng mật khẩu mới

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_009 |
| Risk Level | High |
| Pre-Condition | TC_010 hoàn tất; `U-LIFECYCLE` Active. |
| Test Data | Email `qa.a013.lifecycle.20261109@10kn.io`; mật khẩu `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Smoke @Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở màn đăng nhập trong phiên mới và nhập email của `U-LIFECYCLE` cùng `SEC-A013-V2`. | Credential được chấp nhận và tài khoản đăng nhập thành công. |

### WELVIE_A013_TC_012 — Khóa tài khoản thử

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_010 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` đang Active; không có session đang dùng cho bước quản trị; Admin đã đăng nhập. |
| Test Data | `qa.a013.lifecycle.20261109@10kn.io`. |
| Priority | P0 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Tại Users, chọn action khóa/disable của `U-LIFECYCLE` và xác nhận nếu UI yêu cầu. | Trạng thái của đúng `U-LIFECYCLE` chuyển từ `Active` sang `Disabled`; không tài khoản nào khác đổi trạng thái. |

### WELVIE_A013_TC_013 — Tài khoản bị khóa không đăng nhập được

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_011 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` đang Disabled; TC_012 hoàn tất. |
| Test Data | Email `qa.a013.lifecycle.20261109@10kn.io`; mật khẩu `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Regression @CriticalPath @Security` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Trong phiên mới, thử đăng nhập bằng credential của `U-LIFECYCLE`. | Đăng nhập bị từ chối; không tạo authenticated session và vẫn ở luồng đăng nhập. Nội dung message chờ Q-A013-04. |

### WELVIE_A013_TC_014 — Mở khóa tài khoản thử

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_012 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` đang Disabled; Admin đã đăng nhập. |
| Test Data | `qa.a013.lifecycle.20261109@10kn.io`. |
| Priority | P0 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Tại Users, chọn action mở khóa/enable của `U-LIFECYCLE` và xác nhận nếu UI yêu cầu. | Trạng thái của đúng `U-LIFECYCLE` chuyển từ `Disabled` sang `Active`; không tài khoản nào khác đổi trạng thái. |

### WELVIE_A013_TC_015 — Tài khoản được mở khóa đăng nhập lại được

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_013 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` đang Active; TC_014 hoàn tất. |
| Test Data | Email `qa.a013.lifecycle.20261109@10kn.io`; mật khẩu `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Smoke @Regression @CriticalPath` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Trong phiên mới, đăng nhập bằng credential của `U-LIFECYCLE`. | Đăng nhập thành công và tạo authenticated session. |

### WELVIE_A013_TC_016 — Admin không tự đổi role

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_014 |
| Risk Level | High |
| Pre-Condition | Đang đăng nhập bằng `ADMIN-CURRENT` (`mai.hoang@10kn.io`), role `Administrator`. |
| Test Data | Role hiện tại `Administrator`; role thấp hơn cần thử chọn `View`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Regression @CriticalPath @Security` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Từ Users, mở trang Edit của chính `mai.hoang@10kn.io`. | Trang Edit của đúng tài khoản hiện tại mở được. |
| 2 | Quan sát và thử thao tác ô Role nhưng không thay đổi field khác. | Admin không thể chọn/lưu role `View` cho chính mình; role vẫn là `Administrator`. Cách UI ngăn thao tác chờ Q-A013-03. |
| 3 | Quay lại Users mà không lưu thay đổi khác. | `mai.hoang@10kn.io` vẫn `Active` và role vẫn là `Administrator`. |

### WELVIE_A013_TC_017 — Mật khẩu không hiển thị trên danh sách Users

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_015 |
| Risk Level | High |
| Pre-Condition | `U-LIFECYCLE` tồn tại và đã đổi sang `SEC-A013-V2`; Admin đã đăng nhập. |
| Test Data | `U-LIFECYCLE`; hai secret `SEC-A013-V1`, `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Yes / UI |
| Tags | `@Regression @Security` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở Users và quan sát toàn bộ dữ liệu/cột/action của dòng `U-LIFECYCLE`. | Không có password/initial password/password confirmation hoặc giá trị của `SEC-A013-V1`/`SEC-A013-V2` trên danh sách. |

### WELVIE_A013_TC_018 — Mật khẩu không hiển thị trong audit log

| Thuộc tính | Giá trị |
|---|---|
| VP ID | VP_A013_016 |
| Risk Level | High |
| Pre-Condition | Đã phát sinh event tạo user và đổi mật khẩu cho `U-LIFECYCLE`; Admin có quyền xem audit log. |
| Test Data | Email `qa.a013.lifecycle.20261109@10kn.io`; hai secret `SEC-A013-V1`, `SEC-A013-V2`. |
| Priority | P0 |
| Automatable / Auto Type | Partial / UI |
| Tags | `@Regression @Security` |

| # | Test Steps | Expected Result |
|---:|---|---|
| 1 | Mở audit log và lọc/tìm các event của `qa.a013.lifecycle.20261109@10kn.io`. | Các event liên quan có thể được nhận diện theo account/action nhưng không hiển thị giá trị `SEC-A013-V1`, `SEC-A013-V2` hoặc password confirmation ở bất kỳ field/detail nào. |

## 5. Viewpoint Coverage

| VP ID | Category | Risk | TC IDs | # TCs | Status |
|---|---|---:|---|---:|---|
| VP_A013_001 | Functional | High | TC_001, TC_002, TC_003 | 3 | ✅ Covered |
| VP_A013_002 | Authorization | High | TC_004 | 1 | ⚠️ Weak — thiếu ma trận quyền Q-A013-02 |
| VP_A013_003 | Data Integrity | High | TC_005 | 1 | ✅ Covered |
| VP_A013_004 | Input / Field | Medium | TC_006 | 1 | ✅ Covered |
| VP_A013_005 | Combination | High | TC_007 | 1 | ✅ Covered |
| VP_A013_006 | Data Integrity | Medium | TC_008 | 1 | ✅ Covered |
| VP_A013_007 | Authorization | High | TC_009 | 1 | ⚠️ Weak — thiếu ma trận quyền Q-A013-02 |
| VP_A013_008 | Functional | High | TC_010 | 1 | ✅ Covered |
| VP_A013_009 | Integration | High | TC_011 | 1 | ✅ Covered |
| VP_A013_010 | State | High | TC_012 | 1 | ✅ Covered |
| VP_A013_011 | Authorization | High | TC_013 | 1 | ✅ Covered |
| VP_A013_012 | State | High | TC_014 | 1 | ✅ Covered |
| VP_A013_013 | Authorization | High | TC_015 | 1 | ✅ Covered |
| VP_A013_014 | Authorization | High | TC_016 | 1 | ✅ Covered |
| VP_A013_015 | Security | High | TC_017 | 1 | ✅ Covered |
| VP_A013_016 | Security | High | TC_018 | 1 | ✅ Covered |

Coverage hiện tại: **14/16 viewpoint Covered (87,5%)**, **2/16 Weak**. Không có TC orphan. Tất cả REQ/AC có viewpoint và test case; hai gap chỉ có thể đóng sau khi cung cấp ma trận quyền chi tiết của ba role.

