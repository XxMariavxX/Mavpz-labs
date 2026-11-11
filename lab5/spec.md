# Специфікація архітектури: компоненти та межі UML Component Diagram

## 1. Намір і межі

- **Намір**: описати архітектуру платформи «My Little Pony Online» як набір компонентів з інтерфейсами, залежностями та чіткою межею системи, узгоджену з ERD (lab1), UC/EARS (lab2), Sequence (lab3) і процесом (lab4).
- **Вхідні умови**: веб-клієнт звертається до системи через HTTPS; зовнішні інтеграції — платіжний шлюз та email/SMS.
- **Вихідні умови**: чітка межа системи, визначені provided/required-інтерфейси, трасованість кожного компонента до попередніх моделей.
- **За межами**: інфраструктура (k8s, CI/CD), моніторинг, кешування, CDN.

## 2. Компоненти та інтерфейси

1. **Web UI (Client)** — SPA/SSR-інтерфейс; required: `IAuth`, `ICatalog`, `ICart`, `IOrder`, `IReview`.
2. **API Gateway / Backend** — єдина точка входу; маршрутизація, автентифікація, валідація; provided: `IAuth`, `ICatalog`, `ICart`, `IOrder`, `IReview`.
3. **AuthService** — реєстрація/вхід, валідація токенів, статус користувача; required: `IUserRepo`; provided: `IAuth`.
4. **CatalogService** — каталог, категорії, персонажі, покоління, відгуки; required: `IProductRepo`, `ICharacterRepo`, `IReviewRepo`; provided: `ICatalog`, `ICart`, `IReview`.
5. **OrderService** — створення/оновлення `Order`, ініціалізація оплати, TTL, повторна оплата; required: `IOrderRepo`, `IInventory`, `IPayment`; provided: `IOrder`.
6. **InventoryService** — перевірка/резерв/списання залишку (hold, commit, release); required: `IStockRepo`; provided: `IInventory`.
7. **PaymentService** — сесії, вебхуки PAID/FAILED, TTL; required: `IPaymentGateway`; provided: `IPayment`.
8. **Data Access (Repository Layer)** — єдиний доступ до БД; provided: `IUserRepo`, `IProductRepo`, `IOrderRepo`, `IStockRepo`, `IReviewRepo`, `ICharacterRepo`.
9. **Database** — реляційна БД (PostgreSQL), сутності lab1.
10. **Зовнішні**: PaymentGateway (зовнішній), Email/SMS Provider (зовнішній).

## 3. Інтерфейси (provided/required)

- IAuth: provided by AuthService; required by Web UI, API Gateway.
- ICatalog: provided by CatalogService; required by Web UI, API Gateway.
- ICart: provided by CatalogService; required by Web UI, API Gateway.
- IOrder: provided by OrderService; required by Web UI, API Gateway.
- IReview: provided by CatalogService; required by Web UI, API Gateway.
- IInventory: provided by InventoryService; required by OrderService.
- IPayment: provided by PaymentService; required by OrderService.
- IUserRepo: provided by Data Access; required by AuthService.
- IProductRepo: provided by Data Access; required by CatalogService.
- ICharacterRepo: provided by Data Access; required by CatalogService.
- IReviewRepo: provided by Data Access; required by CatalogService.
- IOrderRepo: provided by Data Access; required by OrderService.
- IStockRepo: provided by Data Access; required by InventoryService.
- IPaymentGateway: provided by PaymentGateway; required by PaymentService.
- INotification: provided by Email/SMS Provider; required by OrderService, AuthService.
- IOAuth: provided by OAuth Provider; required by AuthService.

## 4. Залежності та інтеграції

- Web UI → API Gateway (HTTPS/JSON).
- API Gateway → AuthService, CatalogService, OrderService.
- OrderService → InventoryService, PaymentService.
- Усі сервіси → Data Access → Database.
- PaymentService → PaymentGateway (HTTPS/webhook, PAID/FAILED).
- OrderService / AuthService → Email/SMS Provider (асинхронно).
- Зовнішні інтеграції ізольовані за межею системи (`System_Ext`).

### 4.1 Зовнішні інтеграції

- **PaymentGateway** — provided: `IPaymentGateway`; required by PaymentService; протокол HTTPS/webhook (PAID/FAILED).
- **Email/SMS Provider** — provided: `INotification`; required by OrderService, AuthService; асинхронна відправка.
- **OAuth Provider** — provided: `IOAuth`; required by AuthService; OAuth 2.0 для входу через зовнішнього провайдера.

## 5. Відповідальність компонентів

- **Web UI**: відображення каталогу, кошика, оформлення, статусів; не містить бізнес-логіки.
- **API Gateway**: єдина точка входу, маршрутизація, перевірка токенів, rate limiting.
- **AuthService**: реєстрація, вхід, валідація, статус (`active`/`banned`/`inactive`).
- **CatalogService**: каталог, категорії, персонажі, покоління, відгуки, фільтрація.
- **OrderService**: створення `Order`, ініціалізація оплати, TTL, повторна оплата, оновлення статусів.
- **InventoryService**: перевірка/створення hold, commit списання, release резерву, expireHold.
- **PaymentService**: створення сесій, обробка вебхуків PAID/FAILED, TTL.
- **Data Access**: єдиний доступ до БД, транзакції, мапінг сутностей.
- **Database**: зберігання сутностей lab1.

## 6. Трасування до попередніх моделей

- **lab1 (ERD)**: сутності `User`, `Product`, `Order`, `OrderItem`, `Review`, `Character`, `Category`, `Generation` реалізовані через `Data Access`/`Database`.
- **lab2 (UC/EARS)**: компоненти трасуються до UC-1…UC-9 і REQ-* (UC-5/UC-6 — Order/Payment/Inventory).
- **lab3 (Sequence)**: взаємодія `OrderService` → `InventoryService` → `PaymentService` → `Data Access` відповідає діаграмам 1–6.
- **lab4 (Процес)**: гілки PAID/FAILED/TTL, hold/залишок, повторна оплата відповідають компонентам `OrderService`/`InventoryService`/`PaymentService`.

## 7. Критерії прийняття

- Є одна Component-діаграма з чіткою межею системи та зовнішніми інтеграціями.
- Кожен компонент трасується до lab1 (сутності), lab2 (UC/REQ), lab3 (взаємодія), lab4 (процес).
- Provided/required-інтерфейси визначені в spec.md (розділ 3).
- Залежності та інтеграції описані (розділ 4).
- Діаграма рендериться без помилок.