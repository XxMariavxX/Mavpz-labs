# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

Джерело: `spec.md` (процес), `lab1` (ERD), `lab2` (UC-5/UC-6), `lab3` (Sequence).

Позначення: один початковий вузол, дії — дієслівні фрази, умови в `[дужках]` на кожному виході з ромба, `merge` (●) для злиття альтернатив, `fork/join` (чорна смуга) для паралельних дій, доріжки — ролі.

```mermaid
flowchart TD
    %% ---------- Client ----------
    subgraph Client["Користувач / UI"]
        start(( ))
        a1[Підтвердити оформлення]
        a2[Перенаправити на оплату]
        a3[Показати помилку оплати]
        dRetry{Повторити оплату?}
        aOk[Показати успішне оформлення]
        aAuth[Перенаправити на автентифікацію]
        aBlk[Показати: оформлення недоступне]
        aNoStock[Показати: товару немає в наявності]
    end

    %% ---------- System ----------
    subgraph System["Система (OrderService / InventoryService)"]
        dUser{Користувач активний?}
        dPend{Є Order у статусі created?}
        aBlock[Заблокувати повторне списання]
        mLoop((" "))
        aCheck[Перевірити позицію кошика]
        dHold{Hold активний?}
        dStock{Залишок ≥ кількості?}
        aHold[Створити hold 30 хв]
        aShort[Позначити позицію як недостатню]
        mItem((" "))
        dAll{Усі позиції перевірено?}
        dFail{Є недостатні позиції?}
        aRelease0[Скасувати holds]
        aOrder[Створити Order created і OrderItem]
        aPay[Ініціалізувати платіж]
        aPaid[Встановити статус paid]
        aCancel[Встановити статус cancelled]
        aExpire[Закрити сесію, expireHold]
        fork1[" "]
        aDeduct[Списати залишок]
        aNotify[Надіслати сповіщення]
        join1[" "]
        aRelease[Звільнити резерв]
    end

    %% ---------- Gateway ----------
    subgraph Gateway["Платіжний шлюз"]
        dWh{Результат оплати}
    end

    finNoAuth(((" ")))
    finBlock(((" ")))
    finWait(((" ")))
    finNoStock(((" ")))
    finPaid(((" ")))
    finCancel(((" ")))

    %% ---------- Flow ----------
    start --> a1 --> dUser
    dUser -->|"[ні: AuthRequired]"| aAuth --> finNoAuth
    dUser -->|"[ні: UserBlocked]"| aBlk --> finBlock
    dUser -->|"[так]"| dPend

    dPend -->|"[так]"| aBlock --> finWait
    dPend -->|"[ні]"| mLoop

    mLoop --> aCheck --> dHold
    dHold -->|"[так]"| mItem
    dHold -->|"[ні]"| dStock
    dStock -->|"[так]"| aHold --> mItem
    dStock -->|"[ні]"| aShort --> mItem

    mItem --> dAll
    dAll -->|"[ні]"| mLoop
    dAll -->|"[так]"| dFail

    dFail -->|"[так]"| aRelease0 --> aNoStock --> finNoStock
    dFail -->|"[ні]"| aOrder --> aPay --> a2 --> dWh

    dWh -->|"[PAID]"| aPaid --> fork1
    fork1 --> aDeduct --> join1
    fork1 --> aNotify --> join1
    join1 --> aOk --> finPaid

    dWh -->|"[FAILED]"| aCancel --> aRelease --> a3 --> dRetry
    dWh -->|"[TTL]"| aExpire --> aCancel

    dRetry -->|"[так]"| mLoop
    dRetry -->|"[ні]"| finCancel

    %% ---------- Styling ----------
    classDef bar fill:#000,stroke:#000,color:#000
    classDef merge fill:#fff,stroke:#000
    class fork1,join1 bar
    class mLoop,mItem merge
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
| TTL (REQ-SD-3) | `dWh [TTL]` → `aExpire` → `aCancel` → `aRelease` |
