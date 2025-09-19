## System Architecture Diagrams

### Data Flow and Relationships Diagram

```mermaid
graph TB
    %% User Authentication Layer
    subgraph "Authentication Layer"
        AUTH[Supabase Auth]
        PIN[PIN Verification]
        JWT[JWT Tokens]
    end

    %% Mobile App Layer
    subgraph "Mobile App (WatermelonDB)"
        UI[React UI Components]
        CACHE[Memory Cache<br/>React Query]
        LOCAL[Local Database<br/>SQLite]
        SYNC[Sync Manager]
        OFFLINE[Offline Queue]
    end

    %% Supabase Layer
    subgraph "Supabase Backend"
        API[REST API]
        RLS[Row Level Security]
        REALTIME[Real-time Subscriptions]
        FUNCTIONS[SQL Functions]
    end

    %% Database Layer
    subgraph "PostgreSQL Database"
        subgraph "Core Tables"
            USERS[app_users]
            BUSINESS[businesses]
            STORES[stores]
            ROLES[roles]
            STAFF[staff]
        end
        
        subgraph "Product Tables"
            CATEGORIES[categories]
            PRODUCTS[products]
            INVENTORY[inventory]
        end
        
        subgraph "Transaction Tables"
            CUSTOMERS[customers]
            TRANSACTIONS[transactions]
            TX_ITEMS[transaction_items]
            CREDIT_TX[credit_transactions]
        end
    end

    %% Data Relationships
    USERS -->|owner_id| BUSINESS
    USERS -->|user_id| STAFF
    USERS -->|user_id| TRANSACTIONS
    USERS -->|manager_id| STORES
    
    BUSINESS -->|business_id| STORES
    BUSINESS -->|business_id| ROLES
    BUSINESS -->|business_id| CATEGORIES
    BUSINESS -->|business_id| PRODUCTS
    BUSINESS -->|business_id| CUSTOMERS
    
    STORES -->|store_id| STAFF
    STORES -->|store_id| INVENTORY
    STORES -->|store_id| TRANSACTIONS
    STORES -->|store_id| CREDIT_TX
    
    ROLES -->|role_id| STAFF
    
    CATEGORIES -->|category_id| PRODUCTS
    PRODUCTS -->|product_id| INVENTORY
    PRODUCTS -->|product_id| TX_ITEMS
    
    CUSTOMERS -->|customer_id| TRANSACTIONS
    CUSTOMERS -->|customer_id| CREDIT_TX
    
    TRANSACTIONS -->|transaction_id| TX_ITEMS
    TRANSACTIONS -->|transaction_ids| CREDIT_TX

    %% Authentication Flow
    AUTH -->|login| UI
    PIN -->|quick access| UI
    JWT -->|authorization| API

    %% Data Sync Flow
    UI -->|queries| CACHE
    CACHE -->|cache miss| LOCAL
    LOCAL -->|sync| SYNC
    SYNC -->|upload/download| API
    API -->|RLS filtered| RLS
    RLS -->|permissions| FUNCTIONS
    FUNCTIONS -->|query| USERS
    FUNCTIONS -->|query| BUSINESS
    FUNCTIONS -->|query| STORES
    FUNCTIONS -->|query| ROLES
    FUNCTIONS -->|query| STAFF
    FUNCTIONS -->|query| CATEGORIES
    FUNCTIONS -->|query| PRODUCTS
    FUNCTIONS -->|query| INVENTORY
    FUNCTIONS -->|query| CUSTOMERS
    FUNCTIONS -->|query| TRANSACTIONS
    FUNCTIONS -->|query| TX_ITEMS
    FUNCTIONS -->|query| CREDIT_TX

    %% Real-time Updates
    REALTIME -->|live updates| SYNC
    SYNC -->|apply changes| LOCAL
    LOCAL -->|notify| CACHE
    CACHE -->|re-render| UI

    %% Offline Operations
    UI -->|offline actions| OFFLINE
    OFFLINE -->|queue operations| SYNC
    SYNC -->|retry on connect| API

    %% Verification Flow
    subgraph "Verification Process"
        VERIFY[Data Validation]
        CONFLICT[Conflict Resolution]
        AUDIT[Audit Trail]
    end

    API -->|validate| VERIFY
    VERIFY -->|check permissions| RLS
    VERIFY -->|validate data| FUNCTIONS
    FUNCTIONS -->|business rules| VERIFY
    VERIFY -->|resolve conflicts| CONFLICT
    CONFLICT -->|log changes| AUDIT
    AUDIT -->|track modifications| TRANSACTIONS

    %% Styling
    classDef authClass fill:#e1f5fe
    classDef appClass fill:#f3e5f5
    classDef supabaseClass fill:#e8f5e8
    classDef dbClass fill:#fff3e0
    classDef verifyClass fill:#ffebee

    class AUTH,PIN,JWT authClass
    class UI,CACHE,LOCAL,SYNC,OFFLINE appClass
    class API,RLS,REALTIME,FUNCTIONS supabaseClass
    class USERS,BUSINESS,STORES,ROLES,STAFF,CATEGORIES,PRODUCTS,INVENTORY,CUSTOMERS,TRANSACTIONS,TX_ITEMS,CREDIT_TX dbClass
    class VERIFY,CONFLICT,AUDIT verifyClass
```

### Data Synchronization Flow Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Mobile UI
    participant C as Cache
    participant L as Local DB
    participant S as Sync Manager
    participant API as Supabase API
    participant RLS as RLS Engine
    participant DB as PostgreSQL

    %% Initial Login and Sync
    U->>UI: Login (email/password)
    UI->>API: Authenticate
    API->>DB: Verify credentials
    DB-->>API: User data + permissions
    API-->>UI: JWT token + user profile
    
    UI->>S: Initialize sync (user_id, role)
    S->>API: Request role-based collections
    API->>RLS: Apply permission filters
    RLS->>DB: Query filtered data
    DB-->>RLS: Business data
    RLS-->>API: Filtered results
    API-->>S: Collection data
    S->>L: Store locally
    L-->>C: Cache frequently used data
    C-->>UI: Render interface

    %% Real-time Operations
    loop Real-time Updates
        DB->>API: Data change event
        API->>S: Push notification
        S->>L: Update local record
        L->>C: Invalidate cache
        C->>UI: Re-render component
    end

    %% Offline Operations
    U->>UI: Create transaction (offline)
    UI->>L: Store locally
    L->>S: Queue for sync
    S->>S: Mark as pending
    
    %% Online Sync
    Note over S: Connection restored
    S->>API: Upload pending changes
    API->>RLS: Validate permissions
    RLS->>DB: Check business rules
    DB-->>RLS: Validation result
    
    alt Validation Success
        RLS->>DB: Insert/Update data
        DB-->>API: Success response
        API-->>S: Confirmation
        S->>L: Mark as synced
    else Validation Failed
        RLS-->>API: Error response
        API-->>S: Conflict detected
        S->>S: Resolve conflict
        S->>L: Update with resolution
    end

    %% Background Sync
    loop Every 5 minutes
        S->>API: Check for updates
        API->>RLS: Apply filters
        RLS->>DB: Query changes
        DB-->>RLS: Updated records
        RLS-->>API: Filtered updates
        API-->>S: Incremental data
        S->>L: Apply updates
        L->>C: Refresh cache
    end
```

### Permission Verification Flow

```mermaid
flowchart TD
    START[User Action Request] --> AUTH{User Authenticated?}
    AUTH -->|No| LOGIN[Redirect to Login]
    AUTH -->|Yes| ROLE[Get User Role & Permissions]
    
    ROLE --> PERM{Has Required Permission?}
    PERM -->|No| DENY[Access Denied]
    PERM -->|Yes| SCOPE[Check Data Scope]
    
    SCOPE --> OWNER{Is Business Owner?}
    OWNER -->|Yes| FULL[Full Access Granted]
    OWNER -->|No| STAFF[Check Staff Assignment]
    
    STAFF --> STORE{Assigned to Store?}
    STORE -->|No| DENY
    STORE -->|Yes| FILTER[Apply RLS Filters]
    
    FILTER --> QUERY[Execute Database Query]
    QUERY --> RESULT[Return Filtered Data]
    
    FULL --> QUERY
    DENY --> ERROR[Return Error Message]
    LOGIN --> AUTH

    %% Styling
    classDef startEnd fill:#e8f5e8
    classDef decision fill:#fff3e0
    classDef process fill:#e1f5fe
    classDef error fill:#ffebee

    class START,RESULT startEnd
    class AUTH,PERM,OWNER,STORE decision
    class ROLE,SCOPE,STAFF,FILTER,QUERY process
    class LOGIN,DENY,ERROR error
```

### Data Conflict Resolution Process

```mermaid
flowchart TD
    CONFLICT[Conflict Detected] --> TYPE{Conflict Type?}
    
    TYPE -->|Timestamp| TIME[Compare Timestamps]
    TYPE -->|Inventory| STOCK[Check Stock Levels]
    TYPE -->|Transaction| TX[Validate Transaction]
    
    TIME --> NEWER{Which is Newer?}
    NEWER -->|Local| LOCAL_WIN[Use Local Data]
    NEWER -->|Remote| REMOTE_WIN[Use Remote Data]
    NEWER -->|Same| MERGE[Apply Merge Strategy]
    
    STOCK --> ATOMIC[Atomic Stock Check]
    ATOMIC --> CALC[Calculate Final Stock]
    CALC --> STOCK_WIN[Use Calculated Value]
    
    TX --> VALID{Transaction Valid?}
    VALID -->|Yes| TX_WIN[Keep Transaction]
    VALID -->|No| REJECT[Reject Transaction]
    
    LOCAL_WIN --> APPLY[Apply Resolution]
    REMOTE_WIN --> APPLY
    MERGE --> APPLY
    STOCK_WIN --> APPLY
    TX_WIN --> APPLY
    REJECT --> LOG[Log Rejection]
    
    APPLY --> SYNC[Update Both Systems]
    SYNC --> AUDIT[Log Resolution]
    AUDIT --> COMPLETE[Conflict Resolved]
    
    LOG --> NOTIFY[Notify User]
    NOTIFY --> COMPLETE

    %% Styling
    classDef conflict fill:#ffebee
    classDef decision fill:#fff3e0
    classDef process fill:#e1f5fe
    classDef result fill:#e8f5e8

    class CONFLICT,LOG,NOTIFY conflict
    class TYPE,NEWER,VALID decision
    class TIME,STOCK,TX,ATOMIC,CALC,APPLY,SYNC,AUDIT process
    class LOCAL_WIN,REMOTE_WIN,MERGE,STOCK_WIN,TX_WIN,REJECT,COMPLETE result
```

### Cache Hierarchy and Data Flow

```mermaid
graph TB
    subgraph "Client Side"
        subgraph "UI Layer"
            COMP[React Components]
            HOOKS[Custom Hooks]
        end
        
        subgraph "Cache Layer"
            RQ[React Query Cache]
            MEM[Memory Cache]
        end
        
        subgraph "Storage Layer"
            WM[WatermelonDB]
            SQL[SQLite Storage]
        end
    end
    
    subgraph "Server Side"
        subgraph "API Layer"
            REST[REST API]
            RT[Real-time API]
        end
        
        subgraph "Security Layer"
            RLS[Row Level Security]
            AUTH[Authentication]
        end
        
        subgraph "Database Layer"
            PG[PostgreSQL]
            IDX[Indexes]
        end
    end

    %% Data Flow Arrows
    COMP -->|queries| HOOKS
    HOOKS -->|cache check| RQ
    RQ -->|cache hit| COMP
    RQ -->|cache miss| WM
    WM -->|query| SQL
    SQL -->|data| WM
    WM -->|result| RQ
    RQ -->|cached data| COMP

    %% Sync Flow
    WM -->|sync| REST
    REST -->|authenticate| AUTH
    AUTH -->|authorize| RLS
    RLS -->|filter| PG
    PG -->|query| IDX
    IDX -->|results| PG
    PG -->|filtered data| RLS
    RLS -->|authorized data| REST
    REST -->|sync data| WM

    %% Real-time Flow
    RT -->|live updates| WM
    WM -->|notify| RQ
    RQ -->|invalidate| COMP

    %% Cache Invalidation
    MEM -->|timeout| RQ
    RQ -->|stale data| WM
    WM -->|fresh data| RQ

    %% Styling
    classDef uiClass fill:#e8f5e8
    classDef cacheClass fill:#e1f5fe
    classDef storageClass fill:#f3e5f5
    classDef apiClass fill:#fff3e0
    classDef securityClass fill:#ffebee
    classDef dbClass fill:#e0f2f1

    class COMP,HOOKS uiClass
    class RQ,MEM cacheClass
    class WM,SQL storageClass
    class REST,RT apiClass
    class RLS,AUTH securityClass
    class PG,IDX dbClass
