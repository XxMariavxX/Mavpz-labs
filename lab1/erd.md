```mermaid
erDiagram
	User {
		UUID user_id PK
		VARCHAR username UK
		VARCHAR email UK
		VARCHAR password_hash
		UUID favorite_character_id FK
		DATE registration_date
		VARCHAR status
	}

	Character {
		UUID character_id PK
		VARCHAR name
		VARCHAR pony_type
		TEXT location
		TEXT cutie_mark_description
		TEXT biography
		VARCHAR slug UK
	}

	Generation {
		UUID generation_id PK
		VARCHAR code_name UK
		INT release_year
		TEXT description
	}

	Category {
		UUID category_id PK
		VARCHAR name
		UUID parent_category_id FK
		TEXT description
		VARCHAR slug UK
	}

	Product {
		UUID product_id PK
		VARCHAR name
		UUID category_id FK
		MONEY price
		INT stock_quantity
		TEXT description
		VARCHAR sku UK
		VARCHAR slug UK
	}

	Order {
		UUID order_id PK
		UUID user_id FK
		DATE created_date
		VARCHAR status
		MONEY total_amount
		VARCHAR order_number UK
	}

	OrderItem {
		UUID order_id PK, FK
		UUID product_id PK, FK
		INT quantity
		MONEY price_at_purchase
	}

	Review {
		UUID review_id PK
		UUID user_id FK
		UUID product_id FK
		TEXT comment
		INT rating
		TIMESTAMP published_at
	}
	
  User }o--o| Character : "selects favorite"
	User ||--o{ Order : places
	User ||--o{ Review : writes

	Generation }|--|{ Character : contains
	Category o|--o{ Category : contains
	Category ||--o{ Product : contains
	Character }|--|{ Product : features

	Product ||--o{ Review : receives
	Product ||--o{ OrderItem : includes
	Order ||--|{ OrderItem : contains
```