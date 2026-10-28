# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

Позначення: один початковий вузол, дії — дієслівні фрази, умови в `[дужках]` на кожному виході з ромба, `fork` / `join` (вузли з позначкою) для паралельних дій.

```mermaid
flowchart TD
    start(["Старт"]) --> a1["Підтвердити оформлення"]
    a1 --> dUser{"Користувач активний?"}

    dUser -->|"ні: AuthRequired"| aAuth["Перенаправити на автентифікацію"]
    aAuth --> finNoAuth(["Редирект на автентифікацію"])
    dUser -->|"ні: UserBlocked"| aBlk["Показати оформлення недоступне"]
    aBlk --> finBlock(["Оформлення недоступне"])
    dUser -->|"так"| dPend{"Є Order у статусі created?"}

    dPend -->|"так"| aBlock["Заблокувати повторне списання"]
    aBlock --> finWait(["Очікування оплати"])
    dPend -->|"ні"| loopStart["Перевірити позиції кошика"]

    loopStart --> dHold{"Hold активний?"}
    dHold -->|"так"| dAll{"Усі позиції перевірено?"}
    dHold -->|"ні"| dStock{"Залишок не менше кількості?"}
    dStock -->|"так"| aHold["Створити hold 30 хв"]
    aHold --> dAll
    dStock -->|"ні"| aShort["Позначити позицію як недостатню"]
    aShort --> dAll

    dAll -->|"ні"| loopStart
    dAll -->|"так"| dFail{"Є недостатні позиції?"}

    dFail -->|"так"| aRelease0["Скасувати holds"]
    aRelease0 --> aNoStock["Показати товару немає в наявності"]
    aNoStock --> finNoStock(["Товару немає в наявності"])
    dFail -->|"ні"| aOrder["Створити Order created і OrderItem"]
    aOrder --> aPay["Ініціалізувати платіж"]
    aPay --> a2["Перенаправити на оплату"]
    a2 --> dWh{"Результат оплати"}

    dWh -->|"PAID"| aPaid["Встановити статус paid"]
    aPaid --> fork1(["fork"])
    fork1 --> aDeduct["Списати залишок"]
    fork1 --> aNotify["Надіслати сповіщення"]
    aDeduct --> join1(["join"])
    aNotify --> join1
    join1 --> aOk["Показати успішне оформлення"]
    aOk --> finPaid(["Замовлення оплачено"])

    dWh -->|"FAILED"| aCancel["Встановити статус cancelled"]
    aCancel --> aRelease["Звільнити резерв"]
    aRelease --> a3["Показати помилку оплати"]
    a3 --> dRetry{"Повторити оплату?"}
    dWh -->|"TTL"| aExpire["Закрити сесію expireHold"]
    aExpire --> aCancel2["Встановити статус cancelled"]
    aCancel2 --> aRelease2["Звільнити резерв"]
    aRelease2 --> finExpired(["Сесію закрито TTL"])

    dRetry -->|"так"| loopStart
    dRetry -->|"ні"| finCancel(["Оформлення скасовано"])
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