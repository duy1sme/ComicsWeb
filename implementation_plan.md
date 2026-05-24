# Sửa lỗi & cải thiện code dự án ComicsWeb

Sau khi đọc kỹ toàn bộ code, tôi tìm được **11 lỗi/vấn đề** cần sửa, phân thành 3 nhóm: **Lỗi logic gây crash**, **Lỗi giao diện/UX**, và **Vấn đề dữ liệu**.

---

## Nhóm 1: Lỗi logic gây crash / hoạt động sai

### 🔴 BUG 1 — `auth.js` throw Error từ object thay vì string
Khi login/register thất bại, `err` trả từ server là **object** (vd: `{"message":"Cannot find user"}`), nhưng `throw new Error(err)` sẽ hiển thị `[object Object]` thay vì nội dung thông báo lỗi.

> [!CAUTION]
> Lỗi này khiến người dùng nhìn thấy thông báo `[object Object]` khi đăng nhập sai — rất khó hiểu.

**File**: [auth.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/auth.js#L21-L22)
```diff
// login (line 21)
- throw new Error(err || 'Email hoặc mật khẩu không đúng');
+ throw new Error(err.message || err || 'Email hoặc mật khẩu không đúng');

// register (line 53)
- throw new Error(err || 'Đăng ký thất bại');
+ throw new Error(err.message || err || 'Đăng ký thất bại');
```

---

### 🔴 BUG 2 — `addcomics.js` gửi `id` do user nhập → gây trùng ID + ghi đè dữ liệu
`json-server` sẽ tự động tạo `id` khi POST. Nếu user nhập trùng ID có sẵn (vd: `id: 1`), `json-server` sẽ **ghi đè** truyện cũ thay vì tạo mới.

**File**: [addcomics.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/addcomics.js#L11)
**Sửa**: Bỏ trường `id` khỏi payload — để `json-server` tự sinh ID.

---

### 🔴 BUG 3 — `addcomics.js` tạo slug bằng `.replace(/\s+/g, '-')` → không xử lý tiếng Việt
Cách tạo slug hiện tại không loại bỏ dấu tiếng Việt (vd: "Bảy viên ngọc rồng" → "bảy-viên-ngọc-rồng" thay vì "bay-vien-ngoc-rong"). Trong khi file `utils.js` **đã có sẵn hàm `slugify()`** xử lý đúng. Tương tự lỗi xảy ra ở cả `editcomic.js`.

**File**: [addcomics.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/addcomics.js#L15-L19), [editcomic.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/editcomic.js#L91-L96)
**Sửa**: Dùng `slugify(title)` thay vì `.toLowerCase().replace(/\s+/g, '-')`.

---

### 🔴 BUG 4 — `addcomics.html` thiếu `components.js` → `initComponents()` không tồn tại
Trang addcomics.html không load `components.js`, dẫn đến không có Navbar/Footer. Tuy `addcomics.js` không gọi `initComponents()` nên không crash, nhưng trang không có menu điều hướng → không nhất quán UX.

**File**: [addcomics.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/admin/addcomics.html#L95-L102)
**Sửa**: Thêm `<script src="../../js/components.js">` và gọi `initComponents('admin')` trong `addcomics.js`.

---

### 🟡 BUG 5 — `editcomic.html` có `<div id="navbar">` và `<div id="footer">` rỗng → không render
File `editcomic.html` có 2 placeholder div (`#navbar` và `#footer`) nhưng `components.js` render navbar bằng `insertAdjacentHTML('afterbegin')` vào `body`, không phải vào `#navbar`. Kết quả: navbar được render 1 lần vào body (đúng), nhưng div `#navbar` vẫn trống không — thừa thãi.

**Sửa**: Xoá 2 div placeholder `<div id="navbar"></div>` và `<div id="footer"></div>` khỏi `editcomic.html`.

---

### 🟡 BUG 6 — `editcomic.html` load Bootstrap JS **sau cùng** → navbar toggle có thể không hoạt động
Tất cả các trang khác load Bootstrap JS **trước** các script ứng dụng. Riêng `editcomic.html` lại load Bootstrap JS **cuối cùng** (sau `editcomic.js`). Nếu code JS cần dùng Bootstrap API (vd: collapse, dropdown) khi DOMContentLoaded, nó sẽ chưa sẵn sàng.

**File**: [editcomic.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/admin/editcomic.html#L254-L263)
**Sửa**: Di chuyển `<script src="bootstrap.bundle.min.js">` lên trước các script ứng dụng.

---

## Nhóm 2: Lỗi dữ liệu trong `db.json`

### 🟡 BUG 7 — Genre "Action" viết hoa ở truyện Attack on Titan, các truyện khác dùng "action" viết thường
Hàm `filterByGenre()` trong `search.js` so sánh genres bằng `===` (case-sensitive) với slug `"action"`. Truyện Attack on Titan có genre `"Action"` (viết hoa) → **không bao giờ khớp bộ lọc**.

**File**: [db.json](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/backend/db.json#L170-L172)
```diff
- "Action"
+ "action"
```

---

### 🟡 BUG 8 — `login.html` thiếu `api.js` → `showToast` gọi được nhưng `fetchAPI` không dùng được
Trang login.html không load `api.js`. May mắn là hiện tại login không cần gọi API thông qua `fetchAPI` (dùng `fetch` trực tiếp trong `Auth.login()`), nhưng nếu sau này mở rộng, sẽ gặp lỗi. Tương tự với `register.html`.

**File**: [login.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/login.html#L46-L49), [register.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/register.html#L56-L59)
**Sửa**: Thêm `<script src="../js/api.js">` vào cả 2 trang.

---

## Nhóm 3: Vấn đề nhỏ cải thiện chất lượng code

### 🟢 BUG 9 — `api.js` redirect cứng `/pages/login.html` khi 401 → sai khi ở trang admin
Khi gặp lỗi 401, `fetchAPI()` redirect tới `/pages/login.html` bằng đường dẫn tuyệt đối. Nếu mở qua file:// hoặc đường dẫn khác, sẽ bị lỗi. Nên dùng `getBasePath()` để tạo đường dẫn tương đối.

**File**: [api.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/api.js#L28)
```diff
- window.location.href = '/pages/login.html';
+ window.location.href = getBasePath() + 'pages/login.html';
```

---

### 🟢 BUG 10 — Truyện Attack on Titan: `description` là "Abc", `author` là "Eren" → dữ liệu placeholder
Đây là dữ liệu test/placeholder chưa hoàn chỉnh.

**File**: [db.json](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/backend/db.json#L162-L169)

---

### 🟢 BUG 11 — `search.js` → `filterByGenre` không kết hợp với text search
Khi người dùng vừa nhập text tìm kiếm, vừa bấm genre filter, hai bộ lọc không kết hợp với nhau — genre filter lọc toàn bộ `__allComics`, bỏ qua kết quả text search.

> Đây là improvement nhỏ, có thể bỏ qua nếu không cần.

---

## Proposed Changes

### Backend
#### [MODIFY] [db.json](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/backend/db.json)
- Sửa genre `"Action"` → `"action"` cho truyện Attack on Titan
- Sửa description và author placeholder

---

### Frontend JS
#### [MODIFY] [auth.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/auth.js)
- Sửa `throw new Error(err)` → `throw new Error(err.message || err || '...')`

#### [MODIFY] [api.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/api.js)
- Sửa redirect 401 dùng `getBasePath()`

#### [MODIFY] [addcomics.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/addcomics.js)
- Bỏ trường `id` khỏi payload
- Dùng `slugify()` thay vì `.replace()`
- Gọi `initComponents('admin')` để render navbar

#### [MODIFY] [editcomic.js](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/js/editcomic.js)
- Dùng `slugify()` thay vì `.replace()`

---

### Frontend HTML
#### [MODIFY] [addcomics.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/admin/addcomics.html)
- Thêm `components.js` vào scripts

#### [MODIFY] [editcomic.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/admin/editcomic.html)
- Di chuyển Bootstrap JS lên trước scripts ứng dụng
- Xoá 2 div placeholder thừa

#### [MODIFY] [login.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/login.html)
- Thêm `api.js`

#### [MODIFY] [register.html](file:///e:/DUY/ComicsWeb-main%20(2)/ComicsWeb-main/ComicsWeb-main/frontend/pages/register.html)
- Thêm `api.js`

---

## Verification Plan

### Manual Verification
- Chạy `npm start` trong backend, mở trang chủ, kiểm tra tất cả truyện hiển thị
- Test đăng nhập sai → kiểm tra thông báo lỗi có hiển thị đúng (không còn `[object Object]`)
- Test thêm truyện mới → kiểm tra ID được tự sinh, slug xử lý đúng tiếng Việt
- Test bộ lọc genre → kiểm tra truyện Attack on Titan xuất hiện khi lọc "Hành động"
- Test trang edit comic → kiểm tra navbar và Bootstrap dropdown hoạt động đúng
