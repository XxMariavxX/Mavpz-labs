## Сутності `User`, `Order`, `OrderItem`, `Product`

## Опис та вибір сценарію

*   Сценарії: _Деталізація прецедентів UC-5: Оформити замовлення та UC-6: Обробити підтвердження оплати_
*   Мета: _показати взаємодію клієнта, внутрішніх сервісів платформи та зовнішнього платіжного шлюзу від моменту натискання \<Купити> до отримання повідомлення про оплату_

## Учасники сценарію:

*   Покупець (User) - авторизований покупець;
*   UI (фронтенд) - клієнтський веб-інтерфейс;
*   OrderService - сервіс управління життєвим циклом замовлень (Order);
*   InventoryService - сервіс контролю залишків та керування тимчасовими резервами;
*   PaymentService - сервіс обробки платежів та взаємодії зі шлюзом;
*   PaymentGateway - зовнішній банківський платіжний шлюз;
*   Storage - сховище доменної моделі (User, Order, OrderItem, Product);

## Кроки сценарію

# Успішний варіант:

```
1.  Покупець(авторизований користувач) у кошику натискає `<<Оформити замовлення>>`;
2.  UI передає запит у OrderService;
3.  Цикл перевірки: для кожної позиції товару OrderService запитує у InventoryService перевірку та тимчасове резервування залишку;
4.  OrderService зберігає сутність Order у статусі 'created', записує позиції OrderItem та розраховує загальну суму;
5.  OrderService звертається до PaymentService для ініціалізації оплати;
6.  PaymentService створює платіжну сесію у зовнішньому PaymentGateway та отримує посилання на оплату;
7.  UI перенаправляє покупця на захищену сторінку шлюзу, де покупець вводить реквізити та підтверджує платіж;
8.  PaymentGateway асинхронно надсилає вебхук із підтвердженням;
9.  PaymentService сповіщає OrderService про успішну оплату;
10.  OrderService оновлює статус замовлення в сховищі на 'paid' та дає команду InventoryService на остаточне списання зарезервованого залишку;
11.  Покупець отримує сповіщення про успішне оформлення замовлення;
```

# Альтернативні варіанти:

```
*   Варіант1: Невистачає продукції на складі (_REQ-UW-1, REQ-UW-6_):
    1.  Якщо під час перевірки чи наявний це товар на складі, що його немає або недостатня кількість товарів, скільки хоче користувач, то InventoryService повертає помилку InsufficientStock;
    2.  OrderService скасовує тимчасові броні для інших позицій цього кошика. Замовлення не створюється;
    3.  UI виводить сповіщення: `<<Товару немає в наявності>>`;
*   Варіант2: Помилка оплати(_REQ-UW-2_):
    1.  Якщо платіжний шлюз повертає помилку (status = 'FAILED'), PaymentService повідомляє OrderService;
    2.  OrderService оновлює статус замовлення в сховищі на 'cancelled';
    3.  OrderService надсилає команду в InventoryService зняти бронь (releaseStockReservation(orderId)), повертаючи товар у вільний продаж;
    4.  UI показує повідомлення про збій списання та дію `<<Повторити оплату>>`(яка ініціює новий `Order`);
*   Варіант3:Таймаут оплати та закінчення резерву у 30 хв(_REQ-SD-3, REQ-UW-7_):
    1.  Якщо покупець закрив сторінку оплати або не ввів реквізити протягом 30 хвилин, платіжна сесія вважається простроченою;
    2.  Фоновий таймер сервісу переводить замовлення зі статусу `'created'` у `'cancelled'`;
    3.  InventoryService автоматично анулює тимчасовий резерв товару на складі;
*   Варіант4: Блокування повторного списання(_REQ-SD-1, UC5-S4_):
    1.  Якщо користувач повторно натискає кнопку оплати або надсилає повторний запит, поки перша операція ще обробляється шлюзом (`status = 'created'`), система блокує створення другого запиту;
    2.  UI виводить індикатор: `«Оплата вже обробляється, зачекайте»`;
```

## Sequence-діаграма
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
    autonumber
    actor User
    participant UI
    participant OrderService
    participant InventoryService
    participant PaymentService
    actor PG as PaymentGateway
    participant Storage

    User->>UI: submitCheckout(cartItems)

    opt Already processing (REQ-SD-1)
        UI-->>User: error("Payment already processing")
    end

    UI->>OrderService: createOrder(userId, cartItems)

    loop Each cart item
        OrderService->>InventoryService: checkAndReserveStock(productId, quantity)
        InventoryService->>Storage: getStock(productId)
        Storage-->>InventoryService: stock_quantity
        
        alt Out of stock (REQ-UW-1, REQ-UW-6)
            InventoryService-->>OrderService: error(InsufficientStock)
            OrderService->>InventoryService: cancelCartHolds(cartItems)
            OrderService-->>UI: displayStockError()
            UI-->>User: error("Out of stock")
        else Stock is available
            InventoryService->>Storage: createHold(productId, quantity, timeout=30m)
            Storage-->>InventoryService: holdAck
            InventoryService-->>OrderService: stockHoldSuccess
        end
    end

    OrderService->>Storage: insertOrder(userId, items, totalAmount, status='created')
    Storage-->>OrderService: orderId

    OrderService->>PaymentService: initiatePayment(orderId, totalAmount)
    PaymentService->>PG: createPaymentSession(orderId, totalAmount, returnUrl)
    PG-->>PaymentService: sessionCreated(paymentUrl, transactionId)
    PaymentService-->>OrderService: paymentUrl

    OrderService-->>UI: redirectToPayment(paymentUrl)
    UI-->>User: redirect(paymentUrl)

    alt Payment via Gateway Webhook
        User->>PG: submitPaymentDetails()
        PG->>PaymentService: postWebhook(transactionId, orderId, status)

        alt status == 'SUCCESS' (REQ-ED-2)
            PaymentService->>OrderService: confirmPayment(orderId)
            OrderService->>Storage: updateOrderStatus(orderId, status='paid')
            OrderService->>InventoryService: commitStockDeduction(orderId)
            InventoryService->>Storage: finalizeStockDeduction(orderId)
            PaymentService-->>PG: 200 OK
            PaymentService->>UI: notifyPaymentSuccess(orderId)
            UI-->>User: displaySuccess("Order paid")

        else status == 'FAILED' (REQ-UW-2)
            PaymentService->>OrderService: handlePaymentFailure(orderId)
            OrderService->>Storage: updateOrderStatus(orderId, status='cancelled')
            OrderService->>InventoryService: releaseStockReservation(orderId)
            InventoryService->>Storage: deleteHold(orderId)
            PaymentService-->>PG: 200 OK
            PaymentService->>UI: notifyPaymentFailed(orderId)
            UI-->>User: displayError("Payment failed. Try again")
        end

    else Timeout 30 min expired (REQ-SD-3, REQ-UW-7)
        Note over OrderService,InventoryService: Background worker (TTL = 30m)
        OrderService->>Storage: updateOrderStatus(orderId, status='cancelled')
        OrderService->>InventoryService: releaseStockReservation(orderId)
        InventoryService->>Storage: expireHold(orderId)
        OrderService->>UI: notifySessionTimeout(orderId)
        UI-->>User: displayError("Checkout expired. Reservation released")
    end