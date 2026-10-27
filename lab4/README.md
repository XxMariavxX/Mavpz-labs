# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TD
    subgraph Client [" Користувач / UI "]
        CartAdd[Додати товар у кошик] --> CartHold[Зарезервувати залишок: createHold 30 хв]
        CartHold --> Start([Початок: submitCheckout])
        Start --> VU{User.status == active?}
        VU -- "ні" --> Auth[Редирект на автентифікацію / Помилка]
        Auth --> End1([Кінець: cancelled])
        Redirect[UI: Редирект на сторінку оплати]
        NotifyFail[Повідомлення: Помилка оплати] --> Retry{Повторити оплату?}
        Retry -- "ні" --> End5([Кінець: cancelled])
    end

    subgraph System [" Система (Order & Stock Service) "]
        VU -- "так" --> Pending{Є Order у статусі 'created'?}
        Pending -- "так" --> Block[Блокування повторного списання] --> End2([Кінець: чекати оплату])
        
        Pending -- "ні" --> Loop[Цикл: для кожної позиції cartItems]
        Loop --> Hold{Hold активний?}
        Hold -- "так" --> Verified[stockVerified]
        Hold -- "ні" --> Stock{stock_quantity >= quantity?}
        
        Stock -- "так" --> CreateHold[Повторно створити hold 30 хв] --> Verified
        Stock -- "ні" --> Insufficient[stockFailed: InsufficientStock]
        
        Verified --> Next{Ще позиції?}
        Insufficient --> Next
        Next -- "так" --> Loop
        
        Next -- "ні" --> CheckFail{Є stockFailed?}
        CheckFail -- "так" --> CancelHolds[Скасувати holds: Товару немає] --> End3([Кінець: не створено])
        
        CheckFail -- "ні" --> InsertOrder[insertOrder: status=created]
        InsertOrder --> PayInit[PaymentService.initiatePayment]
        PayInit --> Redirect
        
        Paid[updateOrderStatus: paid]
        Paid --> Fork(("Fork: паралельні дії"))
        Fork --> Deduct[commitStockDeduction]
        Fork --> NotifyOk[Надіслати сповіщення про успіх]
        Deduct --> Join(("Join"))
        NotifyOk --> Join
        Join --> End4([Кінець: paid])
        
        Cancelled[updateOrderStatus: cancelled] --> Release[releaseStockReservation]
        Release --> NotifyFail
        
        Timeout[expireHold: status=cancelled] --> End6([Кінець: timeout/cancelled])
    end

    subgraph Gateway [" Платіжний шлюз (Payment Gateway) "]
        Redirect --> WaitWebhook{Webhook від шлюзу}
        WaitWebhook -- "PAID" --> Paid
        WaitWebhook -- "FAILED" --> Cancelled
        WaitWebhook -- "TTL (Timeout)" --> Timeout
    end

    %% Зворотна стрілка для повторної оплати (повернення до створення/ініціалізації)
    Retry -- "так" --> Loop
```

## Покриття сценаріїв

| Сценарій | Вузол діаграми |
|---|---|
| UC5-S1 | VU → Pending → Loop → InsertOrder → Redirect |
| UC5-S2 | WaitWebhook → PAID → Deduct |
| UC5-S3 | WaitWebhook → FAILED → Release |
| UC5-S4 | Pending → Block |
| UC5-S5 | Retry → InsertOrder |
| UC5-S6 | Stock → Insufficient |
```<|end|><|start|> fileList=write xml<|message|><rfl:parameter name="path">