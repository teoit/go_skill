---
name: goweb-page-sections-option
description: >-
  Hướng dẫn và quy chuẩn quản lý các section tĩnh hoặc linh hoạt của trang web
  (Banners, Intro, Content Blocks, Call-to-actions, Popups...) bằng Option Group và Option Value.
  Phân biệt rõ: 1 section đơn lẻ (Option Value riêng) và nhiều sections trên 1 trang (Option Group -> map DataSection[key] ra Views).
---

# Page Sections Pattern (Quản Lý Section Bằng Option Group & Option Value)

Tài liệu này chuẩn hóa quy trình triển khai các **Section nội dung CMS** (Banners, Khối giới thiệu, Khối cam kết, Khối thống kê, Khối nội dung HTML tùy chỉnh...) trên mọi trang web trong dự án `sunway_goweb`.

---

## 1. Bản Chất Kỹ Thuật (Architecture Concept)

Khi xây dựng một trang web (như *About Us, Contact, News, Home, Order Instructions, Terms, Privacy...*), các khối nội dung được chia thành 2 loại:
1. **Dữ liệu động thuần túy (Entities)**: Có bảng database riêng với nghiệp vụ phức tạp (ví dụ: `cms.posts` cho bài viết, `products` cho sản phẩm, `orders` cho đơn hàng).
2. **Khối nội dung quản trị CMS (Page Sections)**: Được quản lý linh hoạt bằng 2 bảng có sẵn:
   - **Option Group** (`cms.option_group`): Đại diện cho một trang hoặc một nhóm ngữ cảnh (ví dụ: `home`, `about-us`, `contact`).
   - **Option Value** (`cms.options_values`): Đại diện cho từng section cụ thể, lưu tiêu đề (`name`), khóa định danh (`key`), ảnh (`img`), và toàn bộ mã HTML (`content`).

---

## 2. Quy Tắc Phân Loại Triển Khai

Mỗi khi làm một trang có section CMS, cần xác định số lượng section của trang đó:

### Trường Hợp 1: Trang chỉ có 1 Section đơn lẻ (<= 1)
- **Cách làm**: Dùng trực tiếp **Option Value** theo `key` riêng lẻ, không cần tạo group.
- **Ví dụ thực tế**: Banner trang News (`key = "blog-news-banner"`).
- **Truy vấn**: Dùng `FirstOptionValueRepo` với điều kiện:
  ```go
  conds := map[string]interface{}{
      "key = ?":    consts.KEY_BLOG_NEWS_BANNER,
      "status = ?": enum.STATUS_ACTIVE_NUMBER,
  }
  ```
- **Response**: Trả về trực tiếp trường `BannerHTML string` trong response struct.

---

### Trường Hợp 2: Trang có nhiều Section (> 1 section)
- **Cách làm**: Dùng **Option Group** để gom toàn bộ các section của trang lại.
- **Quy tắc đặt tên Group**: **Lấy tên trang / slug trang làm tên cho Option Group** (ví dụ: trang About Us dùng group `about-us`, trang Contact dùng group `contact`).
- **Truy vấn**: Dùng `OptionShared.GetOptionValuesByGroupKey(ctx, groupKey)` (hỗ trợ sẵn cache Redis tự động).
- **Chuyển thành Map `DataSection[key]`**: 
  - Tại UseCase / Mapping, chuyển danh sách `[]OptionValueResp` thành `map[string]responses.OptionValueResp` (hoặc `map[string]string`).
  - Ngoài View `.templ`, truy xuất thẳng theo từng vị trí: **`data.DataSection["key-section"]`**.

---

## 3. Quy Chuẩn Đặt Tên (Naming Conventions)

> [!IMPORTANT]
> **Tuyệt đối KHÔNG gán cứng chuỗi (Magic Strings)**. Mọi `key` của Option Group và Option Value bắt buộc phải khai báo hằng số trong file [modules/web/common/consts/common_option.go](file:///d:/projects/2026_projects/sunway_goweb/modules/web/common/consts/common_option.go).

### A. Quy ước đặt tên Option Group Key:
Dùng tên hoặc slug của trang:
```go
const (
	GROUP_HOME               = "home"
	GROUP_ABOUT_US           = "about-us"
	GROUP_CONTACT            = "contact"
	GROUP_ORDER_INSTRUCTIONS = "order-instructions"
	GROUP_TERMS_OF_USE       = "terms-of-use"
)
```

### B. Quy ước đặt tên Option Value Key (Từng Section):
Theo công thức: `<page-prefix>-<section-name>`:
```go
const (
	// Trang About Us
	KEY_ABOUT_BANNER       = "about-banner"
	KEY_ABOUT_STORY        = "about-story"
	KEY_ABOUT_CORE_VALUES  = "about-core-values"
	KEY_ABOUT_STATISTICS   = "about-statistics"

	// Trang Contact
	KEY_CONTACT_BANNER     = "contact-banner"
	KEY_CONTACT_INFO_BLOCK = "contact-info-block"
	KEY_CONTACT_HOURS      = "contact-hours"

	// Trang News (Section đơn lẻ)
	KEY_BLOG_NEWS_BANNER   = "blog-news-banner"
)
```

---

## 4. Mẫu Triển Khai Chuẩn Cho Trang Có Nhiều Sections (Trường Hợp 2)

Dưới đây là mẫu chuẩn khi triển khai trang (ví dụ trang **About Us** có 3 section: *Banner, Câu chuyện thương hiệu, Giá trị cốt lõi*):

### Bước 1: Khai báo Hằng số (`common_option.go`)
```go
// modules/web/common/consts/common_option.go
const (
	GROUP_ABOUT_US        = "about-us"
	KEY_ABOUT_BANNER      = "about-banner"
	KEY_ABOUT_STORY       = "about-story"
	KEY_ABOUT_CORE_VALUES = "about-core-values"
)
```

### Bước 2: Định nghĩa Response (`about_us_resp.go`)
Response struct phải có trường `DataSection map[string]responses.OptionValueResp`:
```go
// modules/web/transport/responses/about_us_resp.go
package responses

type AboutUsResp struct {
	IsLogin     bool
	DataSection map[string]OptionValueResp // Map chứa các section theo key
}

func NewAboutUsResp(isLogin bool) *AboutUsResp {
	return &AboutUsResp{
		IsLogin:     isLogin,
		DataSection: make(map[string]OptionValueResp),
	}
}
```

### Bước 3: UseCase xử lý gom Map (`about_us_usc.go`)
```go
package usecase

import (
	"context"
	"goweb/modules/web/common/consts"
	"goweb/modules/web/transport/responses"
	"goweb/modules/web/usecase/shared"
)

type aboutUsUsc struct {
	optionShared *shared.OptionShared
}

func (usc *aboutUsUsc) GetAboutUsPageUsc(ctx context.Context, isLogin bool) (*responses.AboutUsResp, error) {
	resp := responses.NewAboutUsResp(isLogin)

	// Lấy tất cả option values của group "about-us" (đã có Redis cache)
	options, err := usc.optionShared.GetOptionValuesByGroupKey(ctx, consts.GROUP_ABOUT_US)
	if err != nil {
		return resp, nil // Không làm gián đoạn trang nếu lỗi CMS
	}

	// CHUYỂN SLICE THÀNH MAP THEO KEY ĐỂ VIEW TRUY XUẤT NHANH
	for _, opt := range options {
		resp.DataSection[opt.Key] = opt
	}

	return resp, nil
}
```

### Bước 4: Hiển thị ngoài View Templ (`views/pages/about_us.templ`)
Truy xuất từng section thông qua `data.DataSection[consts.KEY_...]`, kèm fallback mặc định an toàn:

```templ
package pages

import (
	"goweb/modules/web/common/consts"
	"goweb/modules/web/transport/responses"
	"goweb/views/layouts"
)

templ PageAboutUs(data *responses.AboutUsResp) {
	@layouts.Master(data.IsLogin, nil) {
		<main class="about_page">
			<!-- 1. Section Banner -->
			if sec, ok := data.DataSection[consts.KEY_ABOUT_BANNER]; ok && sec.Content != "" {
				@templ.Raw(sec.Content)
			} else {
				<!-- Fallback Banner mặc định -->
				<section class="banners">
					<h1>About Us</h1>
				</section>
			}

			<!-- 2. Section Story -->
			if sec, ok := data.DataSection[consts.KEY_ABOUT_STORY]; ok && sec.Content != "" {
				@templ.Raw(sec.Content)
			} else {
				<!-- Fallback Story mặc định -->
				<section class="about_story">
					<p>Our story...</p>
				</section>
			}

			<!-- 3. Section Core Values -->
			if sec, ok := data.DataSection[consts.KEY_ABOUT_CORE_VALUES]; ok && sec.Content != "" {
				@templ.Raw(sec.Content)
			}
		</main>
	}
}
```

---

## 5. Checklist Triển Khai

- [ ] Xác định số lượng section: **1 section** (dùng trực tiếp Option Value) hay **nhiều section** (dùng Option Group).
- [ ] Khai báo Group Key và Option Value Keys vào `common_option.go`.
- [ ] Khai báo `DataSection map[string]OptionValueResp` trong Response struct.
- [ ] Trong UseCase, gọi `optionShared.GetOptionValuesByGroupKey` và lặp gán `resp.DataSection[opt.Key] = opt`.
- [ ] Ngoài View `.templ`, truy xuất dạng `data.DataSection[key]`, luôn có khối `else` làm fallback giao diện mặc định.
