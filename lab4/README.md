# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

Позначення: один початковий вузол, дії — дієслівні фрази, умови в `[дужках]` на кожному виході з ромба, `fork/join` (чорна смуга) для паралельних дій. Відповідальність (Client / System / Gateway) вказана в тексті дій: Gateway — лише рішення `Результат оплати`. Доріжки (swimlanes) у Mermaid flowchart не підтримуються нативно, тому роль не винесено в окремі `subgraph`: це прибрало перетини ліній, які робили діаграму нечитабельною на GitHub.

```
flowchart TD
    start((Старт)) --> a1[Підтвердити оформлення]
    a1 --> dUser{Користувач активний?}

    dUser -->|"[ні: AuthRequired]"| aAuth[Перенаправити на автентифікацію] --> finNoAuth((Редирект на автентифікацію))
    dUser -->|"[ні: UserBlocked]"| aBlk[Показати: оформлення недоступне] --> finBlock((Оформлення недоступне))
    dUser -->|"[так]"| dPend{Є Order у статусі created?}

    dPend -->|"[так]"| aBlock[Заблокувати повторне списання] --> finWait((Очікування оплати))
    dPend -->|"[ні]"| loopStart[Перевірити позиції кошика]

    loopStart --> dHold{Hold активний?}
    dHold -->|"[так]"| dAll{Усі позиції перевірено?}
    dHold -->|"[ні]"| dStock{Залишок ≥ кількості?}
    dStock -->|"[так]"| aHold[Створити hold 30 хв] --> dAll
    dStock -->|"[ні]"| aShort[Позначити позицію як недостатню] --> dAll

    dAll -->|"[ні]"| loopStart
    dAll -->|"[так]"| dFail{Є недостатні позиції?}

    dFail -->|"[так]"| aRelease0[Скасувати holds] --> aNoStock[Показати: товару немає в наявності] --> finNoStock((Товару немає в наявності))
    dFail -->|"[ні]"| aOrder[Створити Order created і OrderItem] --> aPay[Ініціалізувати платіж] --> a2[Перенаправити на оплату] --> dWh{Результат оплати}

    dWh -->|"[PAID]"| aPaid[Встановити статус paid] --> fork1[ ]
    fork1 --> aDeduct[Списати залишок] --> join1[ ]
    fork1 --> aNotify[Надіслати сповіщення] --> join1
    join1 --> aOk[Показати успішне оформлення] --> finPaid((Замовлення оплачено))

    dWh -->|"[FAILED]"| aCancel[Встановити статус cancelled] --> aRelease[Звільнити резерв] --> a3[Показати помилку оплати] --> dRetry{Повторити оплату?}
    dWh -->|"[TTL]"| aExpire[Закрити сесію, expireHold] --> aCancel2[Встановити статус cancelled] --> aRelease2[Звільнити резерв] --> finExpired((Сесію закрито TTL))

    dRetry -->|"[так]"| loopStart
    dRetry -->|"[ні]"| finCancel((Оформлення скасовано))

    classDef bar fill:#000,stroke:#000,color:#000,stroke-width:0px
    class fork1,join1 bar
```

## Покриття сценаріїв

| Сценарій | Шлях у діаграмі |
| --- | --- |
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