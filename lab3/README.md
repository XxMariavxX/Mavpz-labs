## Sequence діаграма
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

	OrderService->>Storage: getUserStatus(userId)
	Storage-->>OrderService: user.status

	break Користувач banned або не авторизований (REQ-SD-2, REQ-UW-3)
		OrderService-->>UI: error(UserBlocked)
		UI-->>User: error("Оформлення недоступне")
	end

	OrderService->>Storage: findPendingOrder(userId, cartItems)
	Storage-->>OrderService: none | order(status='created')

	break Order вже у статусі 'created' (REQ-SD-1, UC5-S4)
		OrderService-->>UI: error(PaymentInProgress)
		UI-->>User: error("Оплата вже обробляється, зачекайте")
	end

	loop Each cart item
		OrderService->>InventoryService: checkAndReserveStock(productId, quantity)
		InventoryService->>Storage: getStock(productId)
		Storage-->>InventoryService: stock_quantity

		alt Stock is available
			InventoryService->>Storage: createHold(productId, quantity, timeout=30m)
			Storage-->>InventoryService: holdAck
			InventoryService-->>OrderService: stockHoldSuccess
		else Out of stock (REQ-UW-1, REQ-UW-6)
			break Недостатньо залишку
				InventoryService-->>OrderService: error(InsufficientStock)
				OrderService->>InventoryService: cancelCartHolds(cartItems)
				OrderService-->>UI: displayStockError()
				UI-->>User: error("Товару немає в наявності")
			end
		end
	end

	OrderService->>Storage: insertOrder(userId, orderNumber, totalAmount, status='created')
	Storage-->>OrderService: orderId
	OrderService->>Storage: insertOrderItems(orderId, cartItems)

	OrderService->>PaymentService: initiatePayment(orderId, totalAmount)
	PaymentService->>PG: createPaymentSession(orderId, totalAmount, returnUrl)
	PG-->>PaymentService: sessionCreated(paymentUrl, transactionId)
	PaymentService-->>OrderService: paymentUrl

	OrderService-->>UI: redirectToPayment(paymentUrl)
	UI-->>User: redirect(paymentUrl)

	alt Вебхук від шлюзу
		User->>PG: submitPaymentDetails()
		PG->>PaymentService: postWebhook(transactionId, orderId, status)

		alt status == 'SUCCESS' (REQ-ED-2)
			PaymentService->>OrderService: confirmPayment(orderId)
			OrderService->>Storage: updateOrderStatus(orderId, status='paid')
			OrderService->>InventoryService: commitStockDeduction(orderId)
			InventoryService->>Storage: finalizeStockDeduction(orderId)
			PaymentService-->>PG: 200 OK
			PaymentService->>UI: notifyPaymentSuccess(orderId)
			UI-->>User: displaySuccess("Order paid")

		else status == 'FAILED' (REQ-UW-2, REQ-UW-7)
			PaymentService->>OrderService: handlePaymentFailure(orderId)
			OrderService->>Storage: updateOrderStatus(orderId, status='cancelled')
			OrderService->>InventoryService: releaseStockReservation(orderId)
			InventoryService->>Storage: deleteHold(orderId)
			PaymentService-->>PG: 200 OK
			PaymentService->>UI: notifyPaymentFailed(orderId)
			UI-->>User: displayError("Оплата не пройшла. Повторити оплату")
		end

	else Таймаут 30 хв (REQ-SD-3, REQ-UW-7)
		Note over OrderService,InventoryService: Фоновий worker (TTL = 30m)
		OrderService->>Storage: updateOrderStatus(orderId, status='cancelled')
		OrderService->>InventoryService: releaseStockReservation(orderId)
		InventoryService->>Storage: expireHold(orderId)
		OrderService->>UI: notifySessionTimeout(orderId)
		UI-->>User: displayError("Час оплати минув. Резерв знято")
	end
```