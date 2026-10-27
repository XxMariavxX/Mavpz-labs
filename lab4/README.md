# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

Джерело: `spec.md` (процес), `lab3` (sequence), `lab2` (UC-5/UC-6 сценарії), `lab1` (ERD).

```mermaid
flowchart TD
    Start([Початок: submitCheckout cartItems]) --> VU{User.status == active?}
    VU -- "ні" --> Auth[Редирект на автентифікацію або Оформлення недоступне REQ-UW-3, REQ-SD-2]
    Auth --> End1([Кінець: cancelled])

    VU -- "так" --> Pending{Є Order у статусі created для цього кошика?}
    Pending -- "так" --> Block[Блокування повторного списання REQ-SD-1, UC5-S4]
    Block --> End2([Кінець: чекати оплату])

    Pending -- "ні" --> Loop[Цикл: для кожної позиції cartItems]
    Loop --> Hold{Hold активний? REQ-ED-7}
    Hold -- "так" --> Verified[stockVerified]
    Hold -- "ні" --> Stock{stock_quantity >= quantity?}
    Stock -- "так" --> CreateHold[createHold 30 хв REQ-UB-5]
    CreateHold --> Verified
    Stock -- "ні" --> Insufficient[stockFailed InsufficientStock]
    Verified --> Next{Ще позиції}
    Insufficient --> Next
    Next -- "так" --> Loop
    Next -- "ні" --> CheckFail{Є stockFailed}
    CheckFail -- "так" --> CancelHolds[Скасувати holds, показати Товару немає в наявності REQ-UW-1, REQ-UW-6]
    CancelHolds --> End3([Кінець: не створено])

    CheckFail -- "ні" --> InsertOrder[insertOrder status=created + insertOrderItems + totalAmount]
    InsertOrder --> PayInit[PaymentService.initiatePayment -> PaymentGateway createPaymentSession]
    PayInit --> Redirect[UI redirect на сторінку оплати]
    Redirect --> WaitWebhook{Webhook від шлюзу}

    WaitWebhook -- "PAID" --> Paid[updateOrderStatus paid]
    Paid --> Fork{fork}
    Fork --> Deduct[commitStockDeduction, stock_quantity зменшено]
    Fork --> NotifyOk[Сповіщення про успіх]
    Deduct --> Join{join}
    NotifyOk --> Join
    Join --> End4([Кінець: paid])

    WaitWebhook -- "FAILED" --> Cancelled[updateOrderStatus cancelled]
    Cancelled --> Release[releaseStockReservation / deleteHold]
    Release --> NotifyFail[Повідомлення + Повторити оплату UC5-S5]
    NotifyFail --> Retry{Повторити оплату?}
    Retry -- "так" --> InsertOrder
    Retry -- "ні" --> End5([Кінець: cancelled])

    WaitWebhook -- "TTL" --> Timeout[opt: expireHold, status=cancelled]
    Timeout --> End6([Кінець: cancelled])
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
D:/МАВПЗ/lab4/DEFENSE.md