**Лабораторна робота №2**

# My Little Pony Online

## Опис проєкту
Фанатська платформа та інтернет-магазин мерчу за мотивами всесвіту *My Little Pony*. Платформа поєднує тематичний каталог персонажів і поколінь франшизи з функціоналом електронної комерції: каталогом товарів, категоріями, кошиком, оформленням замовлень та відгуками.

---

## Актори системи
У системі виділено чотири ролі акторів:
1. **Гість** — неавторизований користувач; має доступ до ознайомлення з каталогом мерчу, біографіями персонажів та поколіннями франшизи.
2. **Користувач** — зареєстрований покупець; може формувати кошик, оформлювати замовлення, налаштовувати улюбленого персонажа для персоналізації та залишати відгуки.
3. **Адміністратор** — системний адміністратор; керує асортиментом товарів, структурою категорій та здійснює модерацію акаунтів користувачів.
4. **Платіжний шлюз** — зовнішній сервіс для безпечного проведення безготівкових транзакцій.

---

## Діаграма прецедентів (Use Case Diagram)

Діаграма відображає основні сценарії використання, відношення наслідування між ролями акторів (`Admin --> User --> Guest`), обов'язкові включення (`include`) та опціональні розширення (`extend`):

```mermaid
flowchart LR
	classDef actor fill:#e1f5fe,stroke:#0288d1,stroke-width:1.5px;
	classDef usecase fill:#fff8e1,stroke:#f57c00,stroke-width:1.5px;

	subgraph UsersGroup [Користувачі]
		Guest[Гість Guest]:::actor
		User[Користувач User]:::actor
		Admin[Адміністратор Admin]:::actor
	end

	Admin -->|наслідує| User
	User -->|наслідує| Guest

	subgraph Platform [Платформа My Little Pony Online]
		UC1([UC-1: Переглянути каталог]):::usecase
		UC2([UC-2: Переглянути персонажів]):::usecase
		UC_Filter([UC-Filter: Фільтрувати мерч]):::usecase
		UC4([UC-4: Налаштувати улюбленого поні]):::usecase
		UC7([UC-7: Залишити відгук]):::usecase
		UC3([UC-3: Керувати кошиком]):::usecase
		UC5([UC-5: Оформити замовлення]):::usecase
		UC8([UC-8: Керувати каталогом]):::usecase
		UC9([UC-9: Керувати акаунтами]):::usecase
		UC_Auth([UC-Auth: Автентифікуватися]):::usecase
		UC_Stock([UC-Stock: Перевірити залишки]):::usecase
		UC6([UC-6: Здійснити оплату]):::usecase
	end

	subgraph External [Зовнішній сервіс]
		PaymentGateway[Платіжний шлюз]:::actor
	end

	Guest --> UC1
	Guest --> UC2
	User --> UC4
	User --> UC7
	User --> UC3
	User --> UC5
	Admin --> UC8
	Admin --> UC9
	PaymentGateway --> UC6

	UC_Filter -.->|extend| UC1

	UC4 -.->|include| UC_Auth
	UC7 -.->|include| UC_Auth
	UC3 -.->|include| UC_Auth
	UC3 -.->|include| UC_Stock
	UC5 -.->|include| UC_Auth
	UC5 -.->|include| UC_Stock
	UC5 -.->|include| UC6
	UC8 -.->|include| UC_Auth
	UC9 -.->|include| UC_Auth
```