# Ví Dụ: Trang Động Truyền Dữ Liệu Từ Backend (Fiber -> View)

Tài liệu này minh họa cách truyền dữ liệu từ Controller/Handler xuống View và render component trong `sunway_goweb`.

---

## 1. Định Nghĩa Data Model và View

File: `views/promotions/index.templ`

```templ
package promotions

import (
    "fmt"
    "goweb/views/layouts"
    "github.com/networld-solution/gos/templates"
)

type PromotionItem struct {
    ID          string
    Title       string
    Discount    int
    Code        string
    ExpiryDate  string
    BannerURL   string
    IsActive    bool
}

type PromotionPageData struct {
    Title       string
    Items       []PromotionItem
}

templ PagePromotions(data PromotionPageData) {
    @layouts.Master(&[]templ.Component{head()}) {
        <section class="container promotions-page">
            <div class="promotions-header">
                <h1>{ data.Title }</h1>
                <p>Khám phá các ưu đãi đặc quyền hôm nay</p>
            </div>

            <div class="promotions-grid">
                for _, item := range data.Items {
                    @PromotionCard(item)
                }
            </div>
        </section>
    }
}

templ head() {
    <link rel="stylesheet" href={ templates.AssetURL("/static/css/promotions.css") }/>
}

templ PromotionCard(item PromotionItem) {
    <article class={ "promo-card", templ.KV("promo-card--expired", !item.IsActive) }>
        <div class="promo-badge">{ fmt.Sprintf("%d%% OFF", item.Discount) }</div>
        <img class="promo-thumb" src={ templates.AssetURL(item.BannerURL) } alt={ item.Title }/>
        <div class="promo-body">
            <h3>{ item.Title }</h3>
            <p class="promo-code">Mã: <strong>{ item.Code }</strong></p>
            <p class="promo-expiry">HSD: { item.ExpiryDate }</p>
            if item.IsActive {
                <button type="button" class="btn-copy" data-code={ item.Code }>Lưu mã</button>
            } else {
                <span class="text-muted">Đã hết hạn</span>
            }
        </div>
    </article>
}
```

---

## 2. Handler Nhận Request Và Gọi Render

File: `modules/web/transport/handlers/promotions_hdl.go`

```go
package handlers

import (
	"goweb/views/promotions"

	"github.com/gofiber/fiber/v2"
	"github.com/networld-solution/gos/templates"
)

type promotionsUsc interface {
	GetPromotions(ctx *fiber.Ctx) ([]promotions.PromotionItem, error)
}

type promotionsHdl struct {
	usc promotionsUsc
}

func NewPromotionsHdl(usc promotionsUsc) *promotionsHdl {
	return &promotionsHdl{usc: usc}
}

func (h *promotionsHdl) PagePromotionsHdl() fiber.Handler {
	return func(c *fiber.Ctx) error {
		// Mock hoặc lấy từ UseCase
		items := []promotions.PromotionItem{
			{
				ID:         "promo-1",
				Title:      "Giảm giá mùa hè 2026",
				Discount:   25,
				Code:       "SUMMER26",
				ExpiryDate: "30/09/2026",
				BannerURL:  "/static/img/banner-promo1.webp",
				IsActive:   true,
			},
		}

		data := promotions.PromotionPageData{
			Title: "Chương Trình Khuyến Mãi",
			Items: items,
		}

		return templates.Render(c, promotions.PagePromotions(data))
	}
}
```

---

## 3. Composer & Route

File: `modules/web/composers/comp_promotions.go`
```go
package composers

import (
	"goweb/modules/web/transport/handlers"

	"github.com/gofiber/fiber/v2"
	"github.com/networld-solution/sctx"
)

type composerPromotions interface {
	PagePromotionsHdl() fiber.Handler
}

func ComposerPromotionsService(serviceCtx sctx.ServiceContext) composerPromotions {
	return handlers.NewPromotionsHdl(nil)
}
```

Trong `modules/web/routes_web.go`:
```go
promoComp := composers.ComposerPromotionsService(serviceCtx)
app.Get("/promotions", promoComp.PagePromotionsHdl())
```

Sau khi viết xong code, chạy:
```bash
make gen
```
để biên dịch template.
