
erDiagram
    users {
        int id PK "Primary Key"
        string email "Unique email for login"
        string encrypted_password "Hashed password"
        string role "User role (e.g., 'member', 'admin')"
        datetime created_at
        datetime updated_at
    }

       discoveries {
           int id PK "Primary Key"
           string name "Name of the discovery"
           text description "Detailed description"
           string street_address
           string district
           string city
           string country
           float latitude "For map display"
           float longitude "For map display"
           int user_id FK "Foreign Key to users.id"
           datetime created_at
           datetime updated_at
       }

       comments {
           int id PK "Primary Key"
           text body "Content of the comment"
           int user_id FK "Foreign Key to users.id"
           int discovery_id FK "Foreign Key to discoveries.id"
           datetime created_at
           datetime updated_at
       }

       ratings {
           int id PK "Primary Key"
           int value "Rating value (e.g., 1-5)"
           int user_id FK "Foreign Key to users.id"
           int discovery_id FK "Foreign Key to discoveries.id"
           datetime created_at
           datetime updated_at
       }

       tags {
           int id PK "Primary Key"
           string name "Unique tag name"
           datetime created_at
           datetime updated_at
       }

       taggings {
           int id PK "Primary Key"
           int tag_id FK "Foreign Key to tags.id"
           int taggable_id "ID of the tagged item (e.g., a discovery's ID)"
           string taggable_type "Model name of the tagged item (e.g., 'Discovery')"
           datetime created_at
           datetime updated_at
       }

       users ||--o{ discoveries : "creates"
       users ||--o{ comments : "writes"
       users ||--o{ ratings : "gives"

       discoveries ||--o{ comments : "has"
       discoveries ||--o{ ratings : "has"
       discoveries ||--o{ taggings : "is_tagged_with"

       tags ||--o{ taggings : "is_used_in"