---
name: goweb-code-comments
description: >-
  Quy chuẩn viết comment (docblock và comment từng bước nghiệp vụ bằng tiếng Việt)
  cho các hàm trong UseCase, Service, Handler, và Repository của dự án sunway_goweb.
  Sử dụng skill này bất cứ khi nào tạo mới, chỉnh sửa, hoặc được yêu cầu viết comment cho code Go.
---

# Quy Chuẩn Viết Comment Nghiệp Vụ (`sunway_goweb`)

Tài liệu này quy định chuẩn viết comment tiếng Việt rõ ràng, mạch lạc theo từng bước nghiệp vụ cho các tầng **UseCase**, **Service**, **Handler**, và **Helper** trong dự án `sunway_goweb`.

---

## 1. Nguyên Tắc Chung

1. **Ngôn ngữ**: Sử dụng **tiếng Việt** ngắn gọn, sát thuật ngữ nghiệp vụ của dự án (Option Group, Option Value, Flash Sale, Best Selling, Redis cache, Subquery...).
2. **Định dạng Docblock**: Dùng khối `/** ... */` ngay phía trên khai báo hàm.
3. **Cấu trúc 2 lớp đối với hàm chính (UseCase / Service)**:
   - **Lớp 1 (Docblock đầu hàm)**: Mô tả mục đích tổng quát của hàm + liệt kê `Các bước nghiệp vụ:` đánh số `1, 2, 3...`.
   - **Lớp 2 (Inline comment trong thân hàm)**: Đánh số `// 1. ...`, `// 2. ...` tương ứng với các bước đã nêu ở Docblock để dễ đối chiếu khi đọc code.
4. **Cấu trúc rút gọn đối với hàm phụ trợ (private helper)**:
   - Dùng Docblock 1 dòng hoặc ngắn gọn:
     ```go
     /**
      * <TênHàm>: <Mô tả ngắn gọn mục đích của hàm phụ trợ>
      */
     ```

---

## 2. Mẫu Chuẩn Cho Tầng UseCase (`*_usc.go`)

Mỗi hàm UseCase chính tổng hợp dữ liệu cho trang hoặc xử lý luồng nghiệp vụ phải có đầy đủ Docblock liệt kê các bước và comment đánh số bên trong thân hàm:

```go
/**
 * HomeUsc: Tổng hợp toàn bộ dữ liệu hiển thị cho trang chủ (Home Page)
 * Các bước nghiệp vụ:
 * 1. Nếu khách hàng đã đăng nhập (IsLogin = true), lấy thông tin tổng quan tài khoản (đơn hàng đang xử lý, hóa đơn đến hạn, dư nợ, giá trị đơn trung bình tháng)
 * 2. Lấy thông tin chương trình khuyến mãi Flash Sale hiển thị trên trang chủ (có cache)
 * 3. Lấy danh sách các chương trình khuyến mãi đang hoạt động để tính giá khuyến mãi cho biến thể sản phẩm (có cache)
 * 4. Lấy danh sách sản phẩm bán chạy (Best Selling Products) kèm giá khuyến mãi (có cache)
 * 5. Lấy danh sách sản phẩm mới / xem gần đây (Recent Products) kèm giá khuyến mãi (có cache)
 * 6. Lấy dữ liệu khám phá sản phẩm (Explore Products: Danh mục cấp 2, Thương hiệu, Quốc gia) (có cache)
 * 7. Lấy thông tin khách hàng đang đăng nhập từ context
 * 8. Lấy danh sách câu hỏi thường gặp (FAQs) hiển thị trên trang chủ (có cache)
 * 9. Lấy danh sách nội dung cấu hình giao diện (Option Values: banner login/no-login, why-partner, ready-our-partner...) theo group key "home" (có cache)
 * 10. Đóng gói toàn bộ dữ liệu vào HomeResp và trả về cho Handler render giao diện
 */
func (usc *homeUsc) HomeUsc(ctx context.Context, checkLoginReq *requests.CheckLoginReq) (*responses.HomeResp, error) {
	// 1. Lấy thông tin tổng quan tài khoản nếu đã đăng nhập
	...
	// 2. Lấy chương trình khuyến mãi Flash Sale trang chủ
	...
	// 9. Lấy danh sách option values theo group key "home" (banner, đối tác...)
	homeOptions, err := usc.optionService.GetOptionByKeyGroupCacheService(ctx, consts.KEY_HOME_OPTION, true)
	if err != nil {
		return nil, err
	}

	// 10. Đóng gói dữ liệu trả về cho trang Home
	return homeResp, nil
}
```

---

## 3. Mẫu Chuẩn Cho Tầng Service (`*_service.go`)

Các hàm Service (đặc biệt là các hàm kết hợp Redis Cache + Database Query) cần nêu rõ luồng kiểm tra tham số -> đọc Cache -> truy vấn DB (Subquery/Columns) -> lưu Cache -> trả kết quả:

```go
/**
 * GetOptionByKeyGroupCacheService: Lấy danh sách các Option Value thuộc một Option Group theo slug/key của group bằng 1 câu truy vấn Subquery (hỗ trợ Redis cache)
 * Các bước nghiệp vụ:
 * 1. Kiểm tra tính hợp lệ của keyGroup đầu vào (nếu rỗng thì trả về map rỗng)
 * 2. Nếu bật cờ isCache = true, ưu tiên đọc chuỗi JSON từ Redis cache theo khóa PREFIX_CACHE_REDIS_OPTION_GROUP + keyGroup và giải mã ra map[string]string
 * 3. Nếu chưa có cache (hoặc isCache = false):
 *    - Bước 3.1: Truy vấn trực tiếp bảng cms.options_values (chỉ lấy 2 cột key, content) đang active với điều kiện Subquery lấy id từ cms.option_group theo keyGroup
 *    - Bước 3.2: Chuyển đổi danh sách Option Value thành map[key]content để truy xuất nhanh theo từng section key
 * 4. Nếu có dữ liệu và isCache = true, mã hóa JSON map kết quả và lưu vào Redis cache với TTL cấu hình sẵn
 * 5. Trả về map[string]string chứa cặp (option_value_key -> content HTML) của group
 */
func (s *optionService) GetOptionByKeyGroupCacheService(ctx context.Context, keyGroup string, isCache bool) (map[string]string, error) {
	...
	// 1. Lấy từ Redis cache nếu isCache = true
	...
	// 2. Truy vấn danh sách Option Value từ Database bằng 1 câu SQL Subquery
	...
	// 3. Gom danh sách Option Value vào map[key]content
	...
	// 4. Lưu vào Redis cache nếu isCache = true và có dữ liệu
	...
}
```

---

## 4. Mẫu Chuẩn Cho View `.templ`

Khi hiển thị các section động hoặc điều kiện hiển thị theo trạng thái đăng nhập (`IsLogin`), ghi chú ngắn gọn ngay trên khối điều kiện:

```templ
// 1 - if login và key home-banner-login ngược lại home-banner-no-login
if resp.IsLogin {
	if banner, ok := resp.HomeOptions[consts.KEY_HOME_BANNER_LOGIN]; ok && banner != "" {
		@templ.Raw(banner)
	}
} else {
	if banner, ok := resp.HomeOptions[consts.KEY_HOME_BANNER_NO_LOGIN]; ok && banner != "" {
		@templ.Raw(banner)
	}
}

if !resp.IsLogin {
	// if không có login và key home-why-partner show
	if whyPartner, ok := resp.HomeOptions[consts.KEY_HOME_WHY_PARTNER]; ok && whyPartner != "" {
		@templ.Raw(whyPartner)
	}

	// if không có login và key home-ready-our-partner show
	if readyOurPartner, ok := resp.HomeOptions[consts.KEY_HOME_READY_OUR_PARTNER]; ok && readyOurPartner != "" {
		@templ.Raw(readyOurPartner)
	}
}
```

