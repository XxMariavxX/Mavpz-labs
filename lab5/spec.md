# Специфікація архітектури: компоненти та межі (Component Diagram)

## 1. Межі

- **Вхідні умови**: веб-клієнт звертається до системи через HTTPS; зовнішні інтеграції — платіжний шлюз та email або SMS.
- **Вихідні умови**: чітка межа системи, визначені provided/required-інтерфейси, трасованість кожного компонента до попередніх моделей.

## 1. Компоненти та інтерфейси

1. **Web UI (Client)** — SPA/SSR-інтерфейс; required: `IAuth`, `ICatalog`, `ICart`, `IOrder`, `IReview`.
2. **API Gateway / Backend** — єдина точка входу; маршрутизація, автентифікація; provided: `IAuth`, `ICatalog`, `ICart`, `IOrder`, `IReview`.
3. **AuthService** — реєстрація/вхід, валідація токенів, статус користувача; required: `IUserRepo`.
4. **CatalogService** — каталог, категорії, персонажі, покоління, відгуки; required: `IProductRepo`, `ICharacterRepo`, `IReviewRepo`.
5. **OrderService** — створення/оновлення `Order`, ініціалізація оплати, TTL; required: `IOrderRepo`, `IInventory`, `IPayment`.
6. **InventoryService** — перевірка/резерв/списання залишку (hold, commit, release); required: `IStockRepo`.
7. **PaymentService** — сесії, вебхуки PAID/FAILED, TTL; required: `IPaymentGateway`.