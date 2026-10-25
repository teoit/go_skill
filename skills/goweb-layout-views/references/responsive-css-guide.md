# Hướng Dẫn Chuẩn Viết HTML5 & CSS Responsive Bằng `rem`, `em`, `font-size` Cho `sunway_goweb`

Tài liệu này đúc kết toàn bộ kiến trúc CSS thực tế từ `public/css/default.css`, `public/css/style.css`, `public/css/responsive.css`, và các stylesheet chuyên biệt (`products.css`, `login.css`). Đây là kim chỉ nam bắt buộc tuân thủ khi viết mã HTML5 và CSS trong dự án `sunway_goweb`.

---

## 1. Bản Chất Hệ Thống Đơn Vị Trong Dự Án

Dự án áp dụng mô hình **Root-Anchored Scalable Component Architecture**:

```
[html font-size: 62.5% = 10px] (Thay đổi theo Breakpoint: 10px -> 8px -> 7px)
      │
      ├──> [Component Container font-size: 1rem] (Neo theo root font-size)
      │          │
      │          ├──> padding, gap, border-radius, width, height (Dùng em)
      │          ├──> font-size thẻ con: 1.2em, 1.4em, 1.6em, 2.2em... (Dùng em)
      │          └──> Icon, button dimensions: width: 2.8em; height: 2.8em (Dùng em)
      │
      └──> [Layout Container / Grid] (Dùng max-width %, clamp, minmax, em)
```

### 1.1. Gốc Tỷ Lệ 62.5% (`1rem = 10px`)
Trong `public/css/default.css`:
```css
html {
    scroll-behavior: smooth;
    font-size: 62.5%;
}
```
* Mặc định trình duyệt là `16px`.
* `16px * 62.5% = 10px`.
* Do đó: **`1rem = 10px`** trên màn hình Desktop tiêu chuẩn (>= 1200px).
* **Công thức chuyển đổi nhanh**:
  - `8px` = `0.8rem` (hoặc `0.8em`)
  - `10px` = `1rem` (hoặc `1em`)
  - `12px` = `1.2rem` (hoặc `1.2em`)
  - `14px` = `1.4rem` (hoặc `1.4em`)
  - `16px` = `1.6rem` (hoặc `1.6em`)
  - `24px` = `2.4rem` (hoặc `2.4em`)
  - `32px` = `3.2rem` (hoặc `3.2em`)

---

## 2. Chiến Lược Responsive Toàn Cục (Root Font-Size Scaling)

Trong `public/css/responsive.css`, thay vì viết lại hàng trăm thuộc tính `width`, `height`, `padding`, `margin` cho từng phần tử khi đổi màn hình, dự án sử dụng chiến lược **thu nhỏ font-size của thẻ `html`**:

### Bảng Phân Tầng Breakpoints
| Breakpoint | Khoảng màn hình | Cấu hình `html font-size` | Giá trị 1rem | Tỷ lệ co giãn |
| :--- | :--- | :--- | :--- | :--- |
| **Desktop (xl, xxl)** | `>= 1200px` | `62.5%` (mặc định) | `10px` | 100% |
| **Laptop / Tablet ngang (lg)** | `992px - 1199px` | `8px` | `8px` | Giảm 20% toàn bộ |
| **Tablet dọc (md)** | `768px - 991px` | `7px` | `7px` | Giảm 30% toàn bộ |
| **Mobile (sm, xs)** | `< 768px` / `< 576px` | Reset / Fluid | `clamp`, `vw`, explicit | Chuyển layout cột |

### Mã nguồn mẫu trong `responsive.css`:
```css
/* Tablet ngang (992px - 1199px) */
@media (min-width: 992px) and (max-width: 1199px) {
    html {
        font-size: 8px; /* TẤT CẢ các thành phần tính theo rem tự động thu nhỏ 20% */
    }
}

/* Tablet dọc (768px - 991px) */
@media (min-width: 768px) and (max-width: 991px) {
    html {
        font-size: 7px; /* TẤT CẢ các thành phần tính theo rem tự động thu nhỏ 30% */
    }
}
```

---

## 3. Quy Tắc Phân Định Khi Nào Dùng `rem` và Khi Nào Dùng `em`

Đây là quy tắc thiết kế cốt lõi của codebase:

### 3.1. Khi nào dùng `rem`?
1. **Neo gốc cho Component Wrapper / Card**:
   Mọi component lớn (Card, Form, Button to, Modal) đặt `font-size: 1rem;` ở thẻ bao ngoài cùng để kết nối với tỷ lệ của `html`:
   ```css
   .product {
       font-size: 1rem; /* Neo vào root */
   }
   .login {
       font-size: 1rem;
   }
   .products_filters {
       font-size: 1rem;
   }
   ```
2. **Co giãn cục bộ cho Component ở Mobile**:
   Khi muốn một component thu nhỏ riêng biệt ở màn hình nhỏ, chỉ cần đổi `font-size` của nó sang `rem`:
   ```css
   @media (min-width: 576px) and (max-width: 767px) {
       .qacard {
           font-size: 0.8rem; /* Toàn bộ padding, icon, chữ con bên trong co lại 20% */
       }
       .breadcrumb {
           font-size: 0.7rem;
       }
   }
   ```

### 3.2. Khi nào dùng `em`?
Dùng `em` cho **TẤT CẢ** các thuộc tính bên trong component:
1. **Padding, Margin, Gap**:
   ```css
   .product {
       font-size: 1rem;
       padding: 1.6em;      /* 16px ở desktop, 12.8px ở lg, 11.2px ở md */
       border-radius: 1.6em;/* bo góc cũng co giãn đồng bộ */
       gap: 1.6em;
   }
   ```
2. **Cỡ chữ của các phần tử con**:
   ```css
   .product_name {
       font-size: 1.4em;    /* 1.4 lần font-size của component cha */
   }
   .product_price_current {
       font-size: 1.8em;
       font-weight: 600;
   }
   .product_tag > span {
       font-size: 1.3em;
   }
   ```
3. **Kích thước Icon & Nút phụ thuộc chữ**:
   ```css
   .slide_next {
       font-size: 1rem;
       width: 4.2em;
       height: 4.2em;
   }
   .see_more > svg {
       width: 1.6em;
       height: 1.6em;
   }
   ```
4. **Độ rộng lưới / Sidebar theo em**:
   ```css
   .products {
       font-size: 1rem;
       display: grid;
       grid-template-columns: 30em minmax(0, 1fr); /* 300px ở desktop, 240px ở lg, 210px ở md */
       gap: 2.4em;
   }
   ```

### 3.3. Quy Tắc Bọc `<span>` Cho Button / Thẻ Để Bảo Toàn Chiều Cao (`height`) Khi Đổi `font-size`

Đây là một nguyên lý cực kỳ quan trọng cần ghi nhớ để tránh lỗi **"Bẫy Phóng Đại Kích Thước em" (The em Cascading Trap)**:

#### Vấn đề:
Trong CSS, khi một phần tử có thuộc tính kích thước (`height`, `width`, `padding`, `border-radius`) viết bằng `em`, trình duyệt sẽ tính toán dựa trên `font-size` của **CHÍNH NÓ**, chứ không phải của thẻ cha:

```css
/* ❌ SAI: font-size đặt cùng thẻ với height */
.alert_btn {
    width: 100%;
    height: 4.4em;        /* Kỳ vọng 44px */
    font-size: 1.5em;     /* 15px */
    /* => THỰC TẾ CHIỀU CAO: 4.4 * 15px = 66px (Bị phình to 150%!) */
}
```

#### Giải pháp chuẩn:
Để cố định chiều cao nút chính xác theo tỷ lệ component gốc (1rem = 10px), **luôn giữ thẻ cha theo kích thước chuẩn và bọc văn bản bên trong bằng thẻ `<span>`**:

```templ
<!-- File .templ -->
<a href="/login" class="alert_btn">
    <span>Login</span>
</a>

<button type="submit" class="register_submit">
    <span>Create your new account</span>
</button>
```

```css
/*  ĐÚNG: Thẻ cha giữ kích thước chuẩn, thẻ con nhận font-size */
.alert_btn {
    width: 100%;
    height: 4.4em;        /* Đúng chuẩn 44px (4.4 * 10px) */
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: var(--brand-green);
    color: var(--neutral-white);
    border-radius: 1em;   /* Đúng chuẩn 10px */
    text-decoration: none;
    transition: background-color 0.25s ease;
}

.alert_btn > span {
    font-size: 1.5em;     /* Chữ to 15px mà KHÔNG làm vỡ chiều cao 44px của nút */
    font-weight: 500;
}
```

#### Các ví dụ thực tế trong codebase:
* `.alert_btn > span` trong `public/css/alerts.css`
* `.login_submit > span` trong `public/css/login.css`
* `.register_submit > span` trong `public/css/register.css`
* `.service-card_link > span` trong `public/css/style.css`

### 3.4. Bẫy User Agent Stylesheet: Tại Sao Thẻ Form (`<button>`, `<input>`) Bắt Buộc Phải Có `font-size: 1rem;`?

Một trong những cạm bẫy phổ biến nhất khi áp dụng kiến trúc tỷ lệ `rem`/`em` là quên đặt `font-size: 1rem` trên các phần tử form:

#### 1. Vấn đề từ Trình Duyệt:
Mặc định trình duyệt (Chrome, Safari, Firefox) không cho phép `<button>`, `<input>`, `<select>`, `<textarea>` kế thừa `font-size` từ thẻ cha, mà luôn đặt sẵn:
```css
/* User Agent Stylesheet của trình duyệt */
button, input, select, textarea {
    font-size: 13.333px; /* Không thừa kế từ cha */
}
```

#### 2. Sai Lầm Thực Tế:
```css
/* ❌ SAI: Thiếu font-size: 1rem; */
.btn_faq_load_more {
    height: 4.4em;        /* Kỳ vọng 44px */
    padding: 0 2.4em;     /* Kỳ vọng 24px */
    /* => THỰC TẾ: 4.4 * 13.333px = 58.66px! Nút bị phình to bất thường! */
}
```

#### 3. Quy Chuẩn Khắc Phục:
```css
/*  ĐÚNG: Khai báo rõ font-size: 1rem để neo về gốc dự án */
.btn_faq_load_more {
    font-size: 1rem;      /* Ghi đè 13.333px của browser => Trở về đúng 10px */
    height: 4.4em;        /* Chuẩn xác 4.4 * 10px = 44px */
    padding: 0 2.4em;     /* Chuẩn xác 24px */
    border-radius: 0.8em; /* Chuẩn xác 8px */
}

.btn_faq_load_more > span {
    font-size: 1.4em;     /* Chuẩn xác 14px */
}
```

---

## 4. Typography Động (Fluid Typography Với `clamp` và `vw`)

Trên màn hình di động (< 768px và < 575px), để tiêu đề không bị vỡ hoặc quá to/quá nhỏ, dự án kết hợp hàm `clamp()` và đơn vị Viewport `vw`:

### Tiêu đề Section & Hero
```css
/* Tự động co giãn mượt mà theo chiều rộng màn hình mobile */
.section_titles > h2 {
    font-size: clamp(1.7em, 5.2vw, 2.2em);
}

.qaheadline-title {
    font-size: clamp(1.7em, 5.2vw, 2.2em);
}

.title_center {
    font-size: clamp(1.7em, 5.2vw, 2.2em);
}
```

### Banner & Flash Sale
```css
/* Chữ banner tỷ lệ trực tiếp với viewport width */
.slide-home_title > span {
    font-size: 5vw;
}
.slide-home_des {
    font-size: 2.5vw;
}

/* Flash sale card co giãn chất lỏng */
.flash-sale, .flash-sale_time {
    font-size: clamp(5px, 1vw, 10px);
}
```

---

## 5. Quy Chuẩn Viết HTML5 Kèm CSS Mẫu (Pattern Chuẩn Dự Án)

### 5.1. HTML5 Semantic Chuẩn
* Luôn dùng đúng thẻ: `<section class="container">`, `<article class="tên-card">`, `<header>`, `<nav>`, `<footer>`, `<picture>`, `<time>`.
* Media wrappers bắt buộc có `aspect-ratio` để tránh Layout Shift (CLS):
  - Thumbnail sản phẩm: `aspect-ratio: 1 / 1;`
  - Banner: `aspect-ratio: 1440 / 616;` hoặc trên mobile `aspect-ratio: 375 / 460;`.
* Ảnh responsive dùng thẻ `<picture>` với `<source media="(max-width: 767px)">`.

### 5.2. Component Mẫu Đầy Đủ

#### Cấu trúc HTML (trong file `.templ`):
```templ
package components

import "github.com/networld-solution/gos/templates"

templ ServiceCard(title string, desc string, iconID string) {
    <article class="service-card">
        <div class="service-card_icon">
            <svg><use xlink:href={ "#" + iconID }></use></svg>
        </div>
        <div class="service-card_content">
            <h3 class="service-card_title">{ title }</h3>
            <p class="service-card_desc">{ desc }</p>
            <a class="service-card_link" href="#">
                <span>Learn more</span>
                <svg><use xlink:href="#arrow-right-icon"></use></svg>
            </a>
        </div>
    </article>
}
```

#### Cấu trúc CSS Chuẩn:
```css
/* Base Component Desktop */
.service-card {
    font-size: 1rem;          /* 1. Neo component vào root (1rem = 10px) */
    display: flex;
    flex-direction: column;
    gap: 1.6em;               /* 2. Dùng em cho khoảng cách nội bộ */
    padding: 2.4em;           /* 3. Dùng em cho padding */
    border-radius: 1.6em;     /* 4. Dùng em cho bo góc */
    background-color: var(--neutral-white);
    border: 1px solid var(--neutral-light-gray);
    box-shadow: var(--shadow-1);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.service-card:hover {
    transform: translateY(-0.4em);
}

.service-card_icon {
    width: 4.8em;             /* 48px ở desktop, tự co lại theo root/mobile */
    height: 4.8em;
    border-radius: 1.2em;
    background-color: var(--secondary-lime-10);
    display: flex;
    align-items: center;
    justify-content: center;
}

.service-card_icon > svg {
    width: 2.4em;
    height: 2.4em;
    color: var(--brand-green);
}

.service-card_title {
    font-size: 1.8em;         /* 18px ở desktop */
    font-weight: 600;
    color: var(--secondary-green-deep);
}

.service-card_desc {
    font-size: 1.4em;         /* 14px ở desktop */
    color: var(--neutral-gray);
    line-height: 1.5;
}

.service-card_link {
    display: inline-flex;
    align-items: center;
    gap: 0.8em;
    color: var(--brand-green);
}

.service-card_link > span {
    font-size: 1.4em;
    font-weight: 500;
}

.service-card_link > svg {
    width: 1.6em;
    height: 1.6em;
    transition: transform 0.3s ease;
}

.service-card_link:hover > svg {
    transform: translateX(0.4em);
}

/* Responsive Overrides (trong responsive.css hoặc cuối component file) */
@media (max-width: 575px) {
    .service-card {
        padding: 1.6em;       /* Tinh chỉnh riêng ở mobile siêu nhỏ nếu cần */
        gap: 1.2em;
    }
    
    .service-card_title {
        font-size: clamp(1.6em, 4.5vw, 1.8em); /* Fluid typography */
    }
}
```

---

## 6. Checklist Kiểm Tra CSS Trước Khi Hoàn Thành

- [ ] File CSS có được load đúng qua `templates.AssetURL("/static/css/...")` trong `<head>` không?
- [ ] Đã đặt `font-size: 1rem` ở thẻ bao bọc cấp cao nhất của component chưa?
- [ ] Toàn bộ `padding`, `margin`, `gap`, `border-radius`, `width/height` nội bộ đã dùng `em` thay vì `px` cứng chưa?
- [ ] Các nút/thẻ có cố định `height` (ví dụ `4.4em`, `4.8em`) đã bọc văn bản trong `<span>` và đặt `font-size` trên thẻ con `> span` để tránh làm vỡ chiều cao chưa?
- [ ] Cỡ chữ các thành phần con (`h1-h6`, `p`, `span`, `label`) đã dùng `em` (ví dụ `1.2em`, `1.4em`, `2.2em`) chưa?
- [ ] Màu sắc đã sử dụng biến `--brand-*`, `--secondary-*`, `--neutral-*` trong `:root` chưa?
- [ ] Ở màn hình tablet (768px - 1199px), component có hiển thị tỷ lệ đẹp tự động nhờ `html { font-size: 8px / 7px }` chưa?
- [ ] Ở màn hình mobile (< 768px), layout có chuyển `grid-template-columns: 1fr` hoặc có `clamp()` cho tiêu đề chưa?
