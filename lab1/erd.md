%%{init: {'theme': 'neutral'}}%%
```erDiagram
    User {
        UUID user_id PK
        VARCHAR(100) username UK
        TEXT email UK
        TEXT password_hash
        UUID like_character_id FK
        DATE date_registration
        TEXT status  
    }
    Character {
        UUID character_id PK
        VARCHAR(100) name
        VARCHAR(50) pony_type
        TEXT location
        TEXT cutie_mark_description
        TEXT biography
        VARCHAR(100) slug UK
    }
    Generation {
        UUID generation_id PK
        VARCHAR(50) code UK
        INT release_year
        TEXT description
    }
    Product {
        UUID product_id PK
        VARCHAR(100) name
        INT category_id FK
        UUID character_id FK
        INT price
        INT stock_number
        TEXT description
        VARCHAR(50) sku UK
        VARCHAR(100) slug UK
    }
    Category {
        INT category_id PK
        VARCHAR(100) name
        INT parent_category_id FK
        TEXT description
        VARCHAR(100) slug UK
    }
    Order {
        UUID order_id PK
        UUID user_id FK
        DATE created_date
        VARCHAR(50) status
        INT total_amount
        VARCHAR(20) order_number UK
    }
    OrderItem {
        UUID order_id PK, FK
        UUID product_id PK, FK
        INT number
        INT price_at_purchase
    }
    Review {
        INT review_id PK
        UUID user_id FK
        UUID product_id FK
        TEXT comment
        INT rating
        TIMESTAMP published_at
    }

    User }|--o| Character : "selects favorite"
    User ||--o{ Order : "belongs to"
    User ||--o{ Review : "writes"

    Generation }|--|{ Character : "belongs to"
    Category ||--o{ Category : "has subcategory"
    Category ||--o{ Product : "contains"
    Character ||--|{ Product : "featured in"

    Product ||--o{ Review : "receives"
    Product ||--o{ OrderItem : "ordered in"
    Order ||--|{ OrderItem : "includes"
```