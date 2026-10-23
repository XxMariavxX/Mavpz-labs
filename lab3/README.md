## Sequence діаграми за сценаріями UC-5 «Оформити замовлення»

UC-6 «Обробити підтвердження оплати» деталізовано в межах UC5-S2 (PAID) та UC5-S3 (FAILED / TTL), див. обґрунтування в `spec.md`, розділ «Покриття UC-6».

| Сценарій | Діаграма |
|---|---|
| UC5-S1 Успішне створення замовлення | Діаграма 1 |
| UC5-S2 Успішне проведення оплати (UC-6, PAID) | Діаграма 2 |
| UC5-S3 Помилка оплати / TTL (UC-6, FAILED) | Діаграма 3 |
| UC5-S4 Блокування повторного списання | Діаграма 4 |
| UC5-S5 Повторна оплата після помилки | Діаграма 5 |
| UC5-S6 Недостатній залишок під час оформлення | Діаграма 6 |

### Діаграма 1. UC5-S1: Успішне створення замовлення

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
	autonumber
	actor User
	participant UI
	participant OrderService
	participant InventoryService
	participant PaymentService
	actor PG as PaymentGateway
	participant Storage

	User->>UI: submitCheckout(cartItems)
	UI->>OrderService: submitOrder(userId, cartItems)
	activate OrderService

	OrderService->>Storage: validateUser(userId)
	activate Storage
	Storage-->>OrderService: user.status | none
	deactivate Storage

	break Не авторизований або заблокований (REQ-UW-3, REQ-SD-2, Варіанти 5-6)
		alt Користувач не авторизований (user = none)
			OrderService-->>UI: error(AuthRequired)
			UI-->>User: redirect(authScreen)
		else User.status = banned або inactive (REQ-SD-2)
			OrderService-->>UI: error(UserBlocked)
			UI-->>User: error("Оформлення недоступне")
		end
	end

	OrderService->>Storage: findPendingOrder(userId, cartItems)
	activate Storage
	Storage-->>OrderService: none
	deactivate Storage

	activate InventoryService
	loop Для кожної позиції cartItems (REQ-UB-9)
		OrderService->>InventoryService: checkAndReserveStock(productId, quantity)
		InventoryService->>Storage: getHold(productId, userId)
		activate Storage
		Storage-->>InventoryService: hold | none
		deactivate Storage
		alt Hold активний (створений при додаванні в кошик, REQ-ED-7)
			InventoryService-->>OrderService: stockVerified
		else Hold відсутній або прострочений (REQ-UB-5)
			InventoryService->>Storage: getStock(productId)
			activate Storage
			Storage-->>InventoryService: stock_quantity
			deactivate Storage
			alt stock_quantity >= quantity
				InventoryService->>Storage: createHold(productId, quantity, timeout=30m)
				activate Storage
				Storage-->>InventoryService: holdAck
				deactivate Storage
				InventoryService-->>OrderService: stockReserved
			else stock_quantity < quantity
				InventoryService-->>OrderService: stockFailed(InsufficientStock)
			end
		end
	end

	break Є позиція без залишку (stockFailed, REQ-UW-1, REQ-UW-6, Варіант 1)
		OrderService->>InventoryService: cancelCartHolds(cartItems)
		OrderService-->>UI: displayStockError()
		UI-->>User: error("Товару немає в наявності")
	end
	deactivate InventoryService

	OrderService->>Storage: insertOrder(userId, orderNumber, createdDate, totalAmount, status='created')
	activate Storage
	Storage-->>OrderService: orderId
	deactivate Storage

	OrderService->>Storage: insertOrderItems(orderId, cartItems with price_at_purchase)
	activate Storage
	Storage-->>OrderService: itemsInserted
	deactivate Storage

	activate PaymentService
	OrderService->>PaymentService: initiatePayment(orderId, totalAmount)
	PaymentService->>PG: createPaymentSession(orderId, totalAmount, returnUrl)
	PG-->>PaymentService: sessionCreated(paymentUrl, transactionId)
	PaymentService-->>OrderService: paymentUrl
	deactivate PaymentService

	OrderService-->>UI: redirectToPayment(paymentUrl)
	deactivate OrderService
	UI-->>User: redirect(paymentUrl)
```

### Діаграма 2. UC5-S2: Успішне проведення оплати (UC-6, PAID)

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
	autonumber
	actor User
	participant UI
	participant OrderService
	participant InventoryService
	participant PaymentService
	actor PG as PaymentGateway
	participant Storage

	Note over OrderService,PG: Передумова: Order у статусі 'created' (діаграма 1)

	PG-)PaymentService: postWebhook(transactionId, orderId, status='PAID')
	activate PaymentService
	activate OrderService

	alt status == 'PAID' (REQ-ED-2, UC-6)
		PaymentService->>OrderService: confirmPayment(orderId)
		OrderService->>Storage: updateOrderStatus(orderId, status='paid')
		activate Storage
		Storage-->>OrderService: statusUpdated
		deactivate Storage

		OrderService->>InventoryService: commitStockDeduction(orderId)
		activate InventoryService
		InventoryService->>Storage: finalizeStockDeduction(orderId)
		activate Storage
		Storage-->>InventoryService: stock_quantity decreased
		deactivate Storage
		InventoryService-->>OrderService: deductionCommitted
		deactivate InventoryService

		OrderService-->>PaymentService: paymentConfirmed
		PaymentService-->>PG: 200 OK
		PaymentService->>UI: notifyPaymentSuccess(orderId)
		UI-->>User: displaySuccess("Order paid")
	end

	deactivate OrderService
	deactivate PaymentService
```

### Діаграма 3. UC5-S3: Помилка оплати або TTL (UC-6, FAILED)

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
	autonumber
	actor User
	participant UI
	participant OrderService
	participant InventoryService
	participant PaymentService
	actor PG as PaymentGateway
	participant Storage

	Note over OrderService,PG: Передумова: Order у статусі 'created' (діаграма 1)

	PG-)PaymentService: postWebhook(transactionId, orderId, status='FAILED')
	activate PaymentService
	activate OrderService

	alt status == 'FAILED' (REQ-UW-2, REQ-UW-7, UC-6)
		PaymentService->>OrderService: handlePaymentFailure(orderId)
		OrderService->>Storage: updateOrderStatus(orderId, status='cancelled')
		activate Storage
		Storage-->>OrderService: statusUpdated
		deactivate Storage

		OrderService->>InventoryService: releaseStockReservation(orderId)
		activate InventoryService
		InventoryService->>Storage: deleteHold(orderId)
		activate Storage
		Storage-->>InventoryService: holdDeleted
		deactivate Storage
		InventoryService-->>OrderService: stockReleased
		deactivate InventoryService

		OrderService-->>PaymentService: failureHandled
		PaymentService-->>PG: 200 OK
		PaymentService->>UI: notifyPaymentFailed(orderId)
		UI-->>User: displayError("Оплата не пройшла. Повторити оплату")
	end

	deactivate OrderService
	deactivate PaymentService

	Note over InventoryService,Storage: Фоновий TTL 30 хв для незавершеної сесії (REQ-SD-3)
	Storage-)InventoryService: holdTTLExpired(orderId)
	activate InventoryService
	activate OrderService

	opt Order.status == 'created' (сесія прострочена, REQ-SD-3, Варіант 3)
		InventoryService->>Storage: expireHold(orderId)
		activate Storage
		Storage-->>InventoryService: holdExpired
		deactivate Storage
		OrderService->>Storage: updateOrderStatus(orderId, status='cancelled')
		activate Storage
		Storage-->>OrderService: statusUpdated
		deactivate Storage
		OrderService->>InventoryService: releaseStockReservation(orderId)
		OrderService->>UI: notifySessionTimeout(orderId)
		UI-->>User: displayError("Час оплати минув. Резерв знято")
	end

	deactivate OrderService
	deactivate InventoryService
```

### Діаграма 4. UC5-S4: Блокування повторного списання

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
	autonumber
	actor User
	participant UI
	participant OrderService
	participant Storage

	Note over OrderService,Storage: Перша транзакція вже обробляється шлюзом (Order у статусі 'created')

	User->>UI: clickPay(orderId)
	UI->>OrderService: submitOrder(userId, cartItems)
	activate OrderService

	OrderService->>Storage: findPendingOrder(userId, cartItems)
	activate Storage
	Storage-->>OrderService: order(status='created')
	deactivate Storage

	break Order вже у статусі 'created' (REQ-SD-1, Варіант 4, UC5-S4)
		OrderService-->>UI: error(PaymentInProgress)
		UI-->>User: error("Оплата вже обробляється, зачекайте")
	end

	deactivate OrderService
	Note over OrderService,Storage: PaymentService і PaymentGateway не викликаються, другий запит на списання не створюється
```

### Діаграма 5. UC5-S5: Повторна оплата після помилки

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
	autonumber
	actor User
	participant UI
	participant OrderService
	participant InventoryService
	participant PaymentService
	actor PG as PaymentGateway
	participant Storage

	Note over OrderService,Storage: Передумова: Order cancelled після помилки оплати (UC5-S3), резерв знято

	User->>UI: clickRetryPayment(cancelledOrderId)
	UI->>OrderService: retryOrder(cancelledOrderId, userId)
	activate OrderService

	OrderService->>Storage: getOrder(cancelledOrderId)
	activate Storage
	Storage-->>OrderService: order(status='cancelled'), orderItems
	deactivate Storage

	activate InventoryService
	loop Для кожної позиції OrderItem (REQ-UB-9)
		OrderService->>InventoryService: checkAndReserveStock(productId, quantity)
		InventoryService->>Storage: getStock(productId)
		activate Storage
		Storage-->>InventoryService: stock_quantity
		deactivate Storage
		alt stock_quantity >= quantity
			InventoryService->>Storage: createHold(productId, quantity, timeout=30m)
			activate Storage
			Storage-->>InventoryService: holdAck
			deactivate Storage
			InventoryService-->>OrderService: stockReserved
		else stock_quantity < quantity
			InventoryService-->>OrderService: stockFailed(InsufficientStock)
		end
	end

	break Є позиція без залишку (stockFailed, REQ-UW-6)
		OrderService-->>UI: error(InsufficientStock)
		UI-->>User: error("Товару немає в наявності")
	end
	deactivate InventoryService

	OrderService->>Storage: insertOrder(userId, newOrderNumber, createdDate, totalAmount, status='created')
	activate Storage
	Storage-->>OrderService: newOrderId
	deactivate Storage

	OrderService->>Storage: insertOrderItems(newOrderId, same items with price_at_purchase)
	activate Storage
	Storage-->>OrderService: itemsInserted
	deactivate Storage

	activate PaymentService
	OrderService->>PaymentService: initiatePayment(newOrderId, totalAmount)
	PaymentService->>PG: createPaymentSession(newOrderId, totalAmount, returnUrl)
	PG-->>PaymentService: sessionCreated(paymentUrl, transactionId)
	PaymentService-->>OrderService: paymentUrl
	deactivate PaymentService

	OrderService-->>UI: redirectToPayment(paymentUrl)
	deactivate OrderService
	UI-->>User: redirect(paymentUrl)
```

### Діаграма 6. UC5-S6: Недостатній залишок під час оформлення

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'background': '#ffffff' }}}%%
sequenceDiagram
	autonumber
	actor User
	participant UI
	participant OrderService
	participant InventoryService
	participant Storage

	Note over OrderService,Storage: Передумова: hold з кошика прострочено (REQ-UB-5), доступний залишок < кількість у кошику

	User->>UI: submitCheckout(cartItems)
	UI->>OrderService: submitOrder(userId, cartItems)
	activate OrderService

	OrderService->>Storage: validateUser(userId)
	activate Storage
	Storage-->>OrderService: user.status = 'active'
	deactivate Storage

	OrderService->>Storage: findPendingOrder(userId, cartItems)
	activate Storage
	Storage-->>OrderService: none
	deactivate Storage

	activate InventoryService
	loop Для кожної позиції cartItems (REQ-UB-9)
		OrderService->>InventoryService: checkAndReserveStock(productId, quantity)
		InventoryService->>Storage: getHold(productId, userId)
		activate Storage
		Storage-->>InventoryService: none (hold прострочено)
		deactivate Storage
		InventoryService->>Storage: getStock(productId)
		activate Storage
		Storage-->>InventoryService: stock_quantity < quantity
		deactivate Storage
		InventoryService-->>OrderService: stockFailed(productId, available)
	end

	break Розбіжність залишку в позиції (REQ-UW-6, UC5-S6)
		OrderService->>InventoryService: cancelCartHolds(cartItems)
		OrderService-->>UI: error(InsufficientStock, productId, available)
		UI-->>User: showDiscrepancy(productId, available)
	end
	deactivate InventoryService
	deactivate OrderService
	Note over OrderService,Storage: Order не створюється, позиція залишається в кошику без резерву
```