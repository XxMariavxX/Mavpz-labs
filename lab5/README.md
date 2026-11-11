# UML Component Diagram — архітектура платформи «My Little Pony Online»

Межа системи: усе всередині `My Little Pony Online` — наша платформа; `PaymentGateway`, `Email/SMS Provider`, `OAuth Provider` і `Database` — зовнішні інтеграції.

```
flowchart LR
    subgraph system["My Little Pony Online"]
        webui["Web UI\nReact/Next.js"]
        api["API Gateway\nNode.js"]
        auth["AuthService\nNode.js"]
        catalog["CatalogService\nNode.js"]
        order["OrderService\nNode.js"]
        inventory["InventoryService\nNode.js"]
        payment["PaymentService\nNode.js"]
        data["Data Access\nRepository"]
    end

    db[("Database\nPostgreSQL")]
    pg["PaymentGateway"]
    notify["Email/SMS Provider"]
    oauth["OAuth Provider"]

    webui -->|HTTPS/JSON| api
    api -->|IAuth| auth
    api -->|ICatalog ICart IReview| catalog
    api -->|IOrder| order
    order -->|IInventory| inventory
    order -->|IPayment| payment
    auth -->|IUserRepo| data
    catalog -->|IProductRepo ICharacterRepo IReviewRepo| data
    order -->|IOrderRepo| data
    inventory -->|IStockRepo| data
    data -->|SQL| db
    payment -->|IPaymentGateway| pg
    order -->|INotification| notify
    auth -->|INotification| notify
    auth -->|IOAuth| oauth
```

## Покриття компонентів

| Компонент | Трасування |
| --- | --- |
| Web UI | UC-1…UC-7 (lab2), взаємодія (lab3) |
| OrderService / InventoryService / PaymentService | UC-5/UC-6 (lab2), Sequence (lab3), процес (lab4) |
| Data Access / Database | ERD (lab1) |
| PaymentGateway / Email/SMS / OAuth | актори (lab2), UC-6 (lab3) |