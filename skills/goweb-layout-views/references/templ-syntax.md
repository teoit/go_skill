# a-h/templ Cú Pháp & Best Practices Cho `sunway_goweb`

Tài liệu tham khảo nhanh cú pháp `templ` áp dụng trực tiếp trong dự án `sunway_goweb`.

---

## 1. Cơ Bản Về Component

Mỗi component trong file `.templ` được khai báo bằng từ khóa `templ`:

```templ
package components

templ SectionTitle(title string, subtitle string) {
    <div class="section-title">
        <h2>{ title }</h2>
        if subtitle != "" {
            <p>{ subtitle }</p>
        }
    </div>
}
```

## 2. Nhúng Component Con & Slot (`children...`)

### Gọi Component Con
Sử dụng `@` trước tên component:
```templ
@components.SectionTitle("Featured Products", "Best deals this week")
```

### Component Dạng Container / Layout Nhận Con
Trong component cha, sử dụng `{ children... }`:
```templ
templ Card(title string) {
    <div class="card">
        <div class="card-header">{ title }</div>
        <div class="card-body">
            { children... }
        </div>
    </div>
}
```

Khi sử dụng cha:
```templ
@Card("Thông Tin Đơn Hàng") {
    <p>Nội dung nằm bên trong card-body</p>
}
```

## 3. Điều Kiện (`if / else`)

Cú pháp giống Go, không dùng dấu ngoặc đơn quanh điều kiện:
```templ
if item.IsActive {
    <span class="status active">Active</span>
} else if item.IsPending {
    <span class="status pending">Pending</span>
} else {
    <span class="status inactive">Inactive</span>
}
```

## 4. Vòng Lặp (`for`)

Duyệt slice, mảng hoặc map:
```templ
<ul class="product-list">
    for idx, item := range items {
        <li class={ "item", templ.KV("item--highlight", idx == 0) }>
            @partials.ProductCard(item)
        </li>
    }
}
```

Vòng lặp đếm cơ bản:
```templ
for i := 0; i < 10; i++ {
    @partials.Product()
}
```

## 5. Dynamic Attributes & Classes

### Thẻ điều kiện class (`templ.KV` hoặc `templ.Classes`)
```templ
<div class={ "menu_item", templ.KV("active", isActive), templ.KV("disabled", isDisabled) }>
    ...
</div>
```

### Xử Lý Attribute Có Điều Kiện
```templ
<button
    type="submit"
    class="btn-submit"
    disabled?={ isSubmitting }
>
    <span>Submit</span>
</button>
```

## 6. Static URLs & Safe URLs

Dự án dùng `github.com/networld-solution/gos/templates`:

```templ
import "github.com/networld-solution/gos/templates"

<!-- Đường dẫn tài nguyên tĩnh (css, js, images) -->
<link rel="stylesheet" href={ templates.AssetURL("/static/css/custom.css") }/>
<img src={ templates.AssetURL("/static/img/banner.jpg") } alt="Banner"/>

<!-- Đường dẫn liên kết chuyển trang -->
<a href={ templates.SafeURL("/products?category=" + catID) }>Xem sản phẩm</a>
```

## 7. Sử Dụng SVG Sprite từ `views/layouts/svg.templ`

Dự án định nghĩa các icon trong `views/layouts/svg.templ`. Để sử dụng icon trong bất kỳ view nào:
```templ
<svg><use xlink:href="#cart-icon"></use></svg>
<svg><use xlink:href="#arrow-line-down"></use></svg>
<svg><use xlink:href="#search-icon"></use></svg>
```
Nếu cần thêm icon mới, hãy thêm thẻ `<g id="new-icon-id">...</g>` hoặc symbol vào `views/layouts/svg.templ`.
