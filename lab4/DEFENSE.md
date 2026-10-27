# DEFENSE — Завдання 4: Процес / поведінка (Activity)

## 1. Намір і критерії

*Намір*: описати бізнес-процес оформлення замовлення (UC-5 і UC-6) однією UML Activity-діаграмою з доріжками відповідальності (Swimlanes), узгодженою з ERD (lab1), EARS/UC (lab2) і sequence (lab3).

*Критерії*:
- Окрема Activity-діаграма на процес UC-5/UC-6 із 3 Swimlanes (`Client`, `System`, `Gateway`).
- Гілки PAID/FAILED/TTL/повторне списання/недостатній залишок.
- У діаграмі є розгалуження (`if/else`) і паралельна гілка (`fork/join`).
- Діаграма рендериться в Mermaid без помилок.
- Повнота переходів і досяжність станів; узгодженість із UC (lab2) і Sequence (lab3).

## 2. Топ-3 розбіжності (spec ↔ артефакт) і виправлення

1. **Повторне списання (REQ-SD-1).** Спершу блокування робили в UI. Виправлено: перевірка `Order` у статусі `created` на сервері (гілка `Pending` у Swimlane `System`). 
   Коміт: []
2. **Резерв при оформленні (REQ-ED-7, REQ-UB-5).** Діаграма створювала hold на оформленні. Виправлено: hold створюється при додаванні в кошик; на оформленні `checkAndReserveStock` перевіряє активний hold і створює новий лише при простроченні. 
   Коміт: [].
3. **Структура процесу та доріжки відповідальності (Swimlanes).** Перша версія діаграми була монолітною без розмежування відповідальності. Виправлено: процес розбито на 3 доріжки (`Client`, `System`, `Gateway`), виділено паралельні дії (`Fork/Join` при `PAID`) та впорядковано зворотну стрілку повторної оплати (`Retry`).
   Коміт: []

## 3. Ключове рішення (ADR)

- `adr/adr-0001-process-flow.md` — обрано одну Activity-діаграму з 3 Swimlanes та основними гілками (PAID/FAILED/TTL/повтор/недостача) замість окремих діаграм на сценарій: узгоджено з sequence (lab3).
- Паралельна гілка (fork/join) при PAID: одночасно `commitStockDeduction` і сповіщення покупця.

## 4. Перевірка узгодженості з попередньою моделлю

Перевіряла так:
1. Звірила кожну гілку Activity та відповідні Swimlanes із sequence-діаграмою Lab 3 (UC5-S1…S6).
2. Звірила статуси `Order`/`User`, сутності ERD (lab1) і REQ/UC-ідентифікатори (lab2).
3. Рендер Mermaid без помилок; недосяжних станів немає.

Покриття сценаріїв за доріжками:

| Сценарій | Доріжки (Swimlanes) | Вузол діаграми |
|---|---|---|
| UC5-S1 | Client → System → Gateway | VU → Pending → Loop → InsertOrder → Redirect |
| UC5-S2 | Gateway → System | WaitWebhook (PAID) → Paid → Fork (Deduct / NotifyOk) → Join |
| UC5-S3 | Gateway → System → Client | WaitWebhook (FAILED) → Cancelled → Release → NotifyFail |
| UC5-S4 | Client → System | Pending (так) → Block |
| UC5-S5 | Client → System | Retry (так) → InsertOrder |
| UC5-S6 | System | Stock (ні) → Insufficient → CancelHolds |