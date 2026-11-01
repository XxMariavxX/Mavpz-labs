# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

Позначення: один початковий вузол, дії — дієслівні фрази, умови в `[дужках]` на кожному виході з ромба, `fork` / `join` (вузли з позначкою) для паралельних дій.

```mermaid
flowchart TB
    START((старт)) --> C1([Customer: Оформити замовлення])
    C1 --> W1{WebApp: сесія дійсна?}
    W1 -- ні --> W2[[WebApp: 401 Unauthorized]]
    W2 --> C3([Customer: Переглянути статус])
    W1 -- так --> O1[OrderService: прочитати ціни та залишки]
    O1 --> O2{OrderService: товар є на складі?}
    O2 -- ні --> O3[[insufficientStock]]
    O3 --> C3
    O2 -- так --> O4[OrderService: створити ORDER status = Pending]
    O4 --> C2{Customer: онлайн-оплата?}
    C2 -- ні --> PEND((Pending: автоскасування, UC_ExpireOrder))
    C2 -- так --> O10[OrderService: передати на оплату]
    O10 --> P1[PaymentGateway: списати кошти]
    P1 --> P2{PaymentGateway: схвалено?}
    P2 -- approved --> F1((fork))
    F1 --> O6[OrderService: ORDER.status = Paid]
    F1 --> O7[OrderService: insert outbox OrderPaidEvent]
    O6 --> J1((join: коміт))
    O7 --> J1
    J1 --> R1[OrderService: Relay читає outbox]
    R1 --> R2[OrderService: надіслати email]
    R2 --> R3{Доставлено?}
    R3 -- так --> C3
    R3 -- ні --> R4[Спроба +1]
    R4 -- спроба < 4 --> R2
    R4 -- спроба ≥ 4 --> R5[[FAILED, DLQ]]
    R5 --> C3
    P2 -- declined --> O9[OrderService: status = Unpaid, резерв знято]
    O9 --> O11[[Статус замовлення]]
    O11 --> C3
    O9 --> UNP((Unpaid: повтор оплати, UC_RetryPayment))
    C3 --> STOP((кінець))
```

## Покриття сценаріїв

| Сценарій | Шлях у діаграмі |
|---|---|
| UC5-S1 (успішне створення) | `start` → `a1` → `dUser [так]` → `dPend [ні]` → `mLoop` → цикл перевірки → `dFail [ні]` → `aOrder` → `aPay` → `a2` → `dWh` |
| UC5-S2 (PAID / UC-6) | `dWh [PAID]` → `aPaid` → `fork1` → `aDeduct` ∥ `aNotify` → `join1` → `aOk` |
| UC5-S3 (FAILED / UC-6) | `dWh [FAILED]` → `aCancel` → `aRelease` → `a3` → `dRetry` |
| UC5-S4 (блокування повтору) | `dPend [так]` → `aBlock` |
| UC5-S5 (повторна оплата) | `dRetry [так]` → `mLoop` → повторна перевірка залишку → новий `Order` |
| UC5-S6 (недостатній залишок) | `dStock [ні]` → `aShort` → `dAll` → `dFail [так]` → `aRelease0` → `aNoStock` |
| Неавторизований (REQ-UW-3) | `dUser [ні: AuthRequired]` → `aAuth` |
| Заблокований (REQ-SD-2) | `dUser [ні: UserBlocked]` → `aBlk` |
| TTL (REQ-SD-3) | `dWh [TTL]` → `aExpire` → `aCancel2` → `aRelease2` → `finExpired` (без показу помилки оплати) |
| Неавторизований / заблокований (REQ-UW-3, REQ-SD-2) | `dUser [ні]` → `aAuth`/`aBlk` → кінцевий стан (включено в spec.md, крок 1) |