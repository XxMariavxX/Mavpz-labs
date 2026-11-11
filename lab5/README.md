# UML Component Diagram — архітектура платформи «My Little Pony Online»

Межа системи: `PaymentGateway`, `Email/SMS Provider` і `OAuth Provider` — зовнішні інтеграції.

````
flowchart LR
    subgraph system["My Little Pony Online"]
        webui["Web UI<br/>React/Next.js"]
        api["API Gateway<br/>Node.js"]
        auth["AuthService<br/>Node.js"]
        catalog["CatalogService<br/>Node.js"]
        order["OrderService<br/>Node.js"]
        inventory["InventoryService<br/>Node.js"]
        payment["PaymentService<br/>Node.js"]
        data["Data Access<br/>Repository"]
        db[("Database<br/>PostgreSQL")]
    end

    pg["PaymentGateway<br/>(зовнішній)"]
    notify["Email/SMS Provider<br/>(зовнішній)"]
    oauth["OAuth Provider<br/>(зовнішній)"]

    webui -->|HTTPS/JSON| api
    api -->|IAuth| auth
    api -->|ICatalog, ICart, IReview| catalog
    api -->|IOrder| order
    order -->|IInventory| inventory
    order -->|IPayment| payment
    auth -->|IUserRepo| data
    catalog -->|IProductRepo, ICharacterRepo, IReviewRepo| data
    order -->|IOrderRepo| data
    inventory -->|IStockRepo| data
    data -->|SQL| db
    payment -->|IPaymentGateway| pg
    order -->|INotification| notify
    auth -->|INotification| notify
    auth -->|IOAuth| oauth
```mermaid
flowchart LR
    subgraph system["My Little Pony Online"]
        webui["Web UI"]
        api["API Gateway"]
        auth["AuthService"]
        catalog["CatalogService"]
        order["OrderService"]
        inventory["InventoryService"]
        payment["PaymentService"]
        data["Data Access"]
        db[("Database")]
    end

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
```mermaid
C4Component
    title Component Diagram — My Little Pony Online

    Container_Boundary(system, "My Little Pony Online") {
        Component(webui, "Web UI", "React/Next.js", "Інтерфейс користувача")
        Component(api, "API Gateway", "Node.js", "Маршрутизація, автентифікація")
        Component(auth, "AuthService", "Node.js", "Реєстрація, вхід, статус користувача")
        Component(catalog, "CatalogService", "Node.js", "Каталог, категорії, персонажі, відгуки")
        Component(order, "OrderService", "Node.js", "Замовлення, ініціалізація оплати, TTL")
        Component(inventory, "InventoryService", "Node.js", "Резерв/списання залишку (hold)")
        Component(payment, "PaymentService", "Node.js", "Сесії, вебхуки PAID/FAILED")
        Component(data, "Data Access", "Repository", "Єдиний доступ до БД")
        ContainerDb(db, "Database", "PostgreSQL", "Сутності lab1")
    }

    System_Ext(pg, "PaymentGateway", "Зовнішній платіжний шлюз")
    System_Ext(notify, "Email/SMS Provider", "Зовнішній провайдер сповіщень")
    System_Ext(oauth, "OAuth Provider", "Зовнішній провайдер автентифікації")

    Rel(webui, api, "HTTPS/JSON")
    Rel(api, auth, "IAuth")
    Rel(api, catalog, "ICatalog, ICart, IReview")
    Rel(api, order, "IOrder")
    Rel(order, inventory, "IInventory")
    Rel(order, payment, "IPayment")
    Rel(auth, data, "IUserRepo")
    Rel(catalog, data, "IProductRepo, ICharacterRepo, IReviewRepo")
    Rel(order, data, "IOrderRepo")
    Rel(inventory, data, "IStockRepo")
    Rel(data, db, "SQL")
    Rel(payment, pg, "IPaymentGateway")
    Rel(order, notify, "INotification")
    Rel(auth, notify, "INotification")
    Rel(auth, oauth, "IOAuth")
````

## Покриття компонентів

| Компонент | Трасування |
| --- | --- |
| Web UI | UC-1…UC-7 (lab2), взаємодія (lab3) |
| OrderService / InventoryService / PaymentService | UC-5/UC-6 (lab2), Sequence (lab3), процес (lab4) |
| Data Access / Database | ERD (lab1) |
| PaymentGateway / Email/SMS / OAuth | актори (lab2), UC-6 (lab3) |