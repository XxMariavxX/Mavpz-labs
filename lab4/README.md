# Activity-діаграма процесу: UC-5 «Оформити замовлення» та UC-6 «Обробити підтвердження оплати»

```mermaid
flowchart TB
    START((старт)) --> A1([Client: submitCheckout])
    A1 --> D_USER{System: користувач авторизований і активний?}
    D_USER -- "ні: AuthRequired" --> A_AUTH([Client: redirect на автентифікацію])
    A_AUTH --> FIN_AUTH((кінець: не авторизовано))
    D_USER -- "ні: UserBlocked" --> A_BLK([Client: Оформлення недоступне])
    A_BLK --> FIN_BLK((кінець: заблоковано))
    D_USER -- так --> D_PEND{System: є Order у статусі created?}
    D_PEND -- так --> A_PAYING([Client: Оплата вже обробляється])
    A_PAYING --> FIN_PAYING((кінець: очікує оплату))
    D_PEND -- ні --> L_HOLD[/"System: цикл по позиціях кошика"/]
    L_HOLD --> D_HOLD{System: активний Hold для позиції?}
    D_HOLD -- так --> V_OK([System: позиція підтверджена])
    D_HOLD -- "ні або прострочений" --> D_STOCK{System: stock_quantity >= quantity?}
    D_STOCK -- так --> A_CREATEHOLD([System: створити Hold на 30 хв])
    A_CREATEHOLD --> V_OK
    D_STOCK -- ні --> A_STOCKFAIL([System: InsufficientStock])
    A_STOCKFAIL --> A_CANCELHOLDS([System: скасувати всі Hold кошика])
    A_CANCELHOLDS --> A_NOSTOCK([Client: Товару немає в наявності])
    A_NOSTOCK --> FIN_NOSTOCK((кінець: недостатній залишок))
    V_OK --> D_ALL{System: усі позиції перевірено?}
    D_ALL -- ні --> L_HOLD
    D_ALL -- так --> D_ANYFAIL{System: є позиція без залишку?}
    D_ANYFAIL -- так --> A_CANCELHOLDS
    D_ANYFAIL -- ні --> A_CREATEORDER([System: створити Order status created + OrderItem, total])
    A_CREATEORDER --> A_INITPAY([System: ініціалізувати платіж])
    A_INITPAY --> A_REDIRECT([Client: redirect на шлюз])
    A_REDIRECT --> D_RESULT{Gateway: результат платежу?}
    D_RESULT -- PAID --> A_PAID([System: Order status paid])
    A_PAID --> F1((fork))
    F1 --> A_DEDUCT([System: commitStockDeduction])
    F1 --> A_NOTIFY([System: сповіщення клієнту])
    A_DEDUCT --> J1((join))
    A_NOTIFY --> J1
    J1 --> FIN_PAID((кінець: оплачено))
    D_RESULT -- FAILED --> A_CANCELLED([System: Order status cancelled])
    A_CANCELLED --> A_RELEASE([System: releaseStockReservation])
    A_RELEASE --> A_RETRYOFFER([Client: Оплата не пройшла. Повторити?])
    A_RETRYOFFER --> D_RETRY{Client: повторити оплату?}
    D_RETRY -- так --> L_HOLD
    D_RETRY -- ні --> FIN_CANCELLED((кінець: скасовано))
    D_RESULT -- TTL --> A_EXPIREHOLD([System: expireHold])
    A_EXPIREHOLD --> A_CANCELLED2([System: Order status cancelled])
    A_CANCELLED2 --> A_RELEASE2([System: releaseStockReservation])
    A_RELEASE2 --> A_TIMEOUT([Client: Час оплати минув])
    A_TIMEOUT --> FIN_EXPIRED((кінець: сесія закінчилась))
```

## Покриття сценаріїв

| Сценарій | Шлях у діаграмі |
|---|---|
| UC5-S1 (успішне створення) | `START` → `D_USER` так → `D_PEND` ні → `L_HOLD` → цикл перевірки hold/залишку → `D_ALL` так → `D_ANYFAIL` ні → `A_CREATEORDER` → `A_INITPAY` → `A_REDIRECT` |
| UC5-S2 (PAID / UC-6) | `D_RESULT` PAID → `A_PAID` → `F1` fork → `A_DEDUCT` ∥ `A_NOTIFY` → `J1` join → `FIN_PAID` |
| UC5-S3 (FAILED / UC-6) | `D_RESULT` FAILED → `A_CANCELLED` → `A_RELEASE` → `A_RETRYOFFER` → `D_RETRY` |
| UC5-S4 (блокування повтору) | `D_PEND` так → `A_PAYING` → `FIN_PAYING` |
| UC5-S5 (повторна оплата) | `D_RETRY` так → `L_HOLD` → повторна перевірка залишку/hold → новий `Order` |
| UC5-S6 (недостатній залишок) | `D_STOCK` ні → `A_STOCKFAIL` → `A_CANCELHOLDS` → `A_NOSTOCK` → `FIN_NOSTOCK` |
| Неавторизований | `D_USER` AuthRequired → `A_AUTH` → `FIN_AUTH` |
| Заблокований | `D_USER` UserBlocked → `A_BLK` → `FIN_BLK` |
| TTL | `D_RESULT` TTL → `A_EXPIREHOLD` → `A_CANCELLED2` → `A_RELEASE2` → `FIN_EXPIRED` |
