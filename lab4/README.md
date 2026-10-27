# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

Джерело: `spec.md` (процес), `lab1` (ERD), `lab2` (UC-5/UC-6), `lab3` (Sequence).

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TD
    subgraph Client ["Користувач / UI"]
        Start([Початок: submitCheckout cartItems])
        AuthRedirect[Редирект на автентифікацію]
        BlockMsg["Оформлення недоступне"]
        Redirect[Редирект на сторінку оплати]
        NotifyFail[Повідомлення про помилку оплати]
        Retry{Повторити оплату?}
        EndAuth([Кінець: недоступно])
        EndUserBlock([Кінець: недоступно])
        EndCancel([Кінець: cancelled])
        EndPaid([Кінець: paid])
        EndNoStock([Кінець: не створено])
        EndTimeout([Кінець: cancelled])
        EndProgress([Кінець: оплата обробляється])
    end

    subgraph System ["Система (OrderService / InventoryService / PaymentService)"]
        CheckUser{User існує та User.status == active?}
        CheckPending{Є Order у статусі 'created'?}
        BlockRep[Блокування повторного списання]
        Loop[Цикл: для кожної позиції cartItems]
        HoldCheck{Hold активний?}
        StockCheck{stock_quantity >= quantity?}
        CreateHold[Створити hold 30 хв]
        CreateHoldNew[Створити новий hold 30 хв]
        Verified[stockVerified]
        Insufficient[stockFailed: InsufficientStock]
        NextItem{Ще позиції?}
        HasFail{Є stockFailed?}
        CancelHolds[Скасувати hold-резерви]
        InsertOrder[insertOrder: status='created' + insertOrderItems]
        PayInit[initiatePayment]
        UpdatePaid[updateOrderStatus: paid]
        Fork((Fork))
        Deduct[commitStockDeduction: зменшити stock_quantity]
        NotifyOk[Надіслати сповіщення про успіх]
        Join((Join))
        UpdateCancel[updateOrderStatus: cancelled]
        Release[releaseStockReservation]
        ExpireHold[expireHold]
    end

    subgraph Gateway ["Платіжний шлюз (PaymentGateway)"]
        WaitWebhook{Webhook від шлюзу}
    end

    Start --> CheckUser
    CheckUser -- "ні: AuthRequired (REQ-UW-3)" --> AuthRedirect --> EndAuth
    CheckUser -- "ні: UserBlocked (REQ-SD-2)" --> BlockMsg --> EndUserBlock
    CheckUser -- "так" --> CheckPending

    CheckPending -- "так (REQ-SD-1)" --> BlockRep --> EndProgress
    CheckPending -- "ні" --> Loop

    Loop --> HoldCheck
    HoldCheck -- "так (REQ-ED-7)" --> Verified
    HoldCheck -- "ні або прострочений (REQ-UB-5)" --> StockCheck
    StockCheck -- "так" --> CreateHoldNew --> Verified
    StockCheck -- "ні" --> Insufficient

    Verified --> NextItem
    Insufficient --> NextItem
    NextItem -- "так" --> Loop
    NextItem -- "ні" --> HasFail

    HasFail -- "так (REQ-UW-1, REQ-UW-6)" --> CancelHolds --> EndNoStock
    HasFail -- "ні" --> InsertOrder --> PayInit --> Redirect --> WaitWebhook

    WaitWebhook -- "PAID (REQ-ED-2, UC-6)" --> UpdatePaid --> Fork
    Fork --> Deduct
    Fork --> NotifyOk
    Deduct --> Join
    NotifyOk --> Join
    Join --> EndPaid

    WaitWebhook -- "FAILED (REQ-UW-2, UC-6)" --> UpdateCancel --> Release --> NotifyFail --> Retry
    Retry -- "так (UC5-S5)" --> Loop
    Retry -- "ні" --> EndCancel

    WaitWebhook -- "TTL (REQ-SD-3, UC-6)" --> ExpireHold --> UpdateCancel --> Release --> EndTimeout
```

## Покриття сценаріїв

| Сценарій | Покриття в діаграмі |
|---|---|
| UC5-S1 (Успішне створення) | `Start` → `CheckUser` (active) → `CheckPending` (ні) → цикл перевірки hold/залишку → `InsertOrder` → `Redirect` |
| UC5-S2 (Успішна оплата / UC-6 PAID) | `WaitWebhook` (PAID) → `UpdatePaid` → `Fork` (`Deduct`, `NotifyOk`) → `Join` |
| UC5-S3 (Помилка оплати / UC-6 FAILED) | `WaitWebhook` (FAILED) → `UpdateCancel` → `Release` → `NotifyFail` → `Retry` |
| UC5-S4 (Блокування повторного списання) | `CheckPending` (так) → `BlockRep` |
| UC5-S5 (Повторна оплата після помилки) | `Retry` (так) → `Loop` (повторна перевірка залишку) → створення нового `Order` |
| UC5-S6 (Недостатній залишок) | `StockCheck` (ні) → `Insufficient` → `CancelHolds` |