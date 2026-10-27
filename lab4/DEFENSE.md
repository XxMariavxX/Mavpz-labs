# DEFENSE — Завдання 4: Процес / поведінка (Activity)

## 1. Намір і критерії

*Намір*: описати бізнес-процес оформлення замовлення (UC-5 і UC-6) однією UML Activity-діаграмою з доріжками відповідальності (Swimlanes), узгодженою з ERD (lab1), EARS/UC (lab2) і Sequence (lab3).

*Критерії*:
- Одна Activity-діаграма з Swimlanes (`Client`, `System`, `Gateway`).
- Розгалуження (`if/else`) та паралельна гілка (`fork/join`).
- Покрито UC5-S1…S6, гілки PAID/FAILED/TTL/повторне списання/недостатній залишок/авторизація.
- Повнота переходів, досяжність вузлів, узгодженість із UC (lab2) і Sequence (lab3).
- Діаграма рендериться в Mermaid без помилок.

## 2. Топ-3 розбіжності (spec ↔ артефакт) + коміт-виправлення

1. **Повторне списання (REQ-SD-1).** У першій версії блокування було в UI. Виправлено: перевірка `Order` у статусі `created` на стороні `System` (гілка `CheckPending`). Коміт: `3aff223` — `fix: grouped by swimlanes` (перенесення перевірки в System; деталі — у поточних змінах, коміт за автором).
2. **Життєвий цикл `Hold` (REQ-ED-7, REQ-UB-5).** Резерв створюється при додаванні в кошик; на оформленні перевіряється активний hold і створюється новий лише коли попередній відсутній або прострочений (`HoldCheck` → `CreateHoldNew`). Коміт: `d3227be` — `fix(lab4): align hold lifecycle with checkout spec`.
3. **Повторна оплата (UC5-S5, REQ-UW-2).** Раніше `Retry` вів напряму до ініціалізації, оминаючи перевірку залишку. Виправлено: `Retry (так) → Loop` (повторна перевірка hold/залишку), далі створюється новий `Order`. Коміт: буде зафіксовано в поточних змінах.

## 3. Ключове рішення (ADR)

Обрано одну Activity-діаграму з Swimlanes (`Client/System/Gateway`) та гілками PAID/FAILED/TTL/повтор/недостача замість 6 окремих діаграм (узгоджено з Lab 3, зручніше для рев'ю). При PAID застосовано `Fork/Join` для паралельного `commitStockDeduction` та сповіщення. Розбіжності вирішувались через правки `spec.md` та `README.md`, а не через ручний «патч» лише в DEFENSE. ADR: `lab4/adr/adr-0001-process-flow.md`.

## 4. Перевірка узгодженості з попередньою моделлю

- **З Lab 1 (ERD):** статуси `Order` (`created`/`paid`/`cancelled`), `User.status`, `OrderItem`, `Product.stock_quantity` узгоджені.
- **З Lab 2 (UC/EARS):** покрито UC5-S1…S6, `UC-Auth-S1`, `REQ-SD-1/2/3`, `REQ-ED-1/2/7`, `REQ-UW-1/2/6`.
- **З Lab 3 (Sequence):** вузли та переходи узгоджені з діаграмами 1–6; `AuthRequired`/`UserBlocked` розрізнені; TTL відображений.
- **Аудит артефакту:** перевірена наявність розгалужень, `fork/join`, наявність початкового вузла `Start`, досяжність критичних гілок, узгодженість `Retry` з перевіркою залишку. Діаграма валідована через `mermaid-cli` (ререндер без помилок). 

## 5. Покриття сценаріїв

| Сценарій | Доріжки | Ключові вузли |
|---|---|---|
| UC5-S1 | Client → System → Gateway | `Start` → `CheckUser` → `CheckPending` → цикл hold/stock → `InsertOrder` → `Redirect` |
| UC5-S2 | Gateway → System → Client | `WaitWebhook(PAID)` → `UpdatePaid` → `Fork` → `Deduct`/`NotifyOk` → `Join` |
| UC5-S3 | Gateway → System → Client | `WaitWebhook(FAILED)` → `UpdateCancel` → `Release` → `NotifyFail` → `Retry` |
| UC5-S4 | Client → System | `CheckPending(так)` → `BlockRep` |
| UC5-S5 | Client → System → Gateway | `Retry(так)` → `Loop` (повторна перевірка) → `InsertOrder` |
| UC5-S6 | System | `StockCheck(ні)` → `Insufficient` → `CancelHolds` |