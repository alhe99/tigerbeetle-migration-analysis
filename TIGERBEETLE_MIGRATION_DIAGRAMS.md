# TigerBeetle Wallet Migration - Mermaid Diagrams Reference

**Quick Reference for All Operation Flows**

This document contains all mermaid diagrams from the migration analysis for easy viewing and presentation.

---

## 1. GetBalance Operation

### Current MongoDB Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as GetWalletBalanceQueryHandler
    participant Repo as PaymentsWalletRepository
    participant MongoDB

    Client->>API: GET /balance/client123/USA/USD?issuerTypeIdentifier=VISA
    API->>Handler: GetWalletBalanceQuery
    Handler->>Repo: FindBy(client123, USA, USD, VISA)
    Repo->>MongoDB: Find({ ClientId: "client123", Country.Code: "USA", Currency.Code: "USD", IssuerTypeIdentifier: "VISA" })
    MongoDB-->>Repo: PaymentsWallet document
    Repo-->>Handler: PaymentsWallet entity
    Handler-->>API: WalletBalanceVm { Credit: 100, Debit: 0, HistoricalCredit: 150 }
    API-->>Client: JSON Response
```

### Future TigerBeetle Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as GetWalletBalanceQueryHandler
    participant Repo as TigerBeetleWalletRepository
    participant TB as TigerBeetle
    participant Cache as Redis/Memory Cache

    Client->>API: GET /balance/client123/USA/USD?issuerTypeIdentifier=VISA
    API->>Handler: GetWalletBalanceQuery
    Handler->>Repo: GetBalance(client123, USA, USD, VISA)

    alt Cache Hit
        Repo->>Cache: Get cached balance
        Cache-->>Repo: Cached account data
    else Cache Miss
        Repo->>Repo: accountId = Hash(client123+USA+USD+VISA)
        Repo->>TB: lookup_accounts([accountId])
        TB-->>Repo: Account { credits_posted: 250, debits_posted: 150 }
        Repo->>Cache: Cache account data (TTL: 60s)
    end

    Repo->>Repo: Calculate: balance = credits_posted - debits_posted
    Repo-->>Handler: Balance { Credit: 100, Debit: 0 }
    Handler-->>API: WalletBalanceVm
    API-->>Client: JSON Response
```

---

## 2. GetMultiBalance Operation

### Current MongoDB Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as GetMultiBalanceQueryHandler
    participant Repo as PaymentsWalletRepository
    participant MongoDB

    Client->>API: GET /balances/client123
    API->>Handler: GetMultiBalanceQuery { ClientId: "client123" }
    Handler->>Repo: GetWalletsBy("client123")
    Repo->>MongoDB: Find({ ClientId: "client123" })
    MongoDB-->>Repo: [PaymentsWallet, PaymentsWallet, ...]
    Repo-->>Handler: List<PaymentsWallet> (3 wallets)
    Handler->>Handler: Map to MultiBalanceVm
    Handler-->>API: MultiBalanceVm { Balances: [3 items] }
    API-->>Client: JSON Response
```

### Future TigerBeetle Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as GetMultiBalanceQueryHandler
    participant Repo as TigerBeetleWalletRepository
    participant Mapper as AccountIdMapper
    participant TB as TigerBeetle
    participant Cache as Redis Cache

    Client->>API: GET /balances/client123
    API->>Handler: GetMultiBalanceQuery { ClientId: "client123" }
    Handler->>Repo: GetAllWalletsForClient("client123")

    Repo->>Mapper: GetAccountIdsForClient("client123")
    Mapper->>Mapper: Query client account registry
    Note over Mapper: Registry stores: client123 → [accountId1, accountId2, accountId3]
    Mapper-->>Repo: [accountId1, accountId2, accountId3]

    alt Cache Hit (partial or full)
        Repo->>Cache: Get cached accounts
        Cache-->>Repo: Partial account data
    end

    Repo->>TB: lookup_accounts([accountId1, accountId2, accountId3])
    TB-->>Repo: [Account, Account, Account]

    Repo->>Cache: Cache all account data
    Repo->>Repo: Calculate balances for each account
    Repo-->>Handler: List<WalletBalance>
    Handler-->>API: MultiBalanceVm
    API-->>Client: JSON Response
```

---

## 3. AddCredits Operation

### Current MongoDB Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as AddCreditCommandHandler
    participant WalletRepo as PaymentsWalletRepository
    participant CountryRepo as PaymentsWalletCountryRepository
    participant CurrencyRepo as PaymentsWalletCurrencyRepository
    participant TxRepo as PaymentsWalletTransactionRepository
    participant EventService as DomainEventService
    participant MongoDB

    Client->>API: POST /credit { amount: 50, clientId: "client123", country: "USA", currency: "USD" }
    API->>Handler: AddCreditCommand

    Handler->>WalletRepo: FindBy(client123, USA, USD, null)
    WalletRepo->>MongoDB: Find wallet

    alt Wallet Exists with Debt
        MongoDB-->>WalletRepo: PaymentsWallet { Credit: 0, Debit: 30 }
        WalletRepo-->>Handler: Existing wallet

        Handler->>Handler: Calculate: 50 >= 30, so Credit = 20, Debit = 0
        Handler->>Handler: OldBalance = "-30", NewBalance = "20"
        Handler->>Handler: HistoricalCredit += 50

    else Wallet Exists without Debt
        MongoDB-->>WalletRepo: PaymentsWallet { Credit: 100, Debit: 0 }
        WalletRepo-->>Handler: Existing wallet

        Handler->>Handler: Credit = 100 + 50 = 150
        Handler->>Handler: OldBalance = "100", NewBalance = "150"
        Handler->>Handler: HistoricalCredit += 50

    else Wallet Doesn't Exist
        MongoDB-->>WalletRepo: null
        WalletRepo-->>Handler: null

        Handler->>CountryRepo: FindBy("USA")
        CountryRepo->>MongoDB: Find country metadata
        MongoDB-->>CountryRepo: PaymentsWalletCountry
        CountryRepo-->>Handler: Country

        Handler->>CurrencyRepo: FindBy("USD")
        CurrencyRepo->>MongoDB: Find currency metadata
        MongoDB-->>CurrencyRepo: PaymentsWalletCurrency
        CurrencyRepo-->>Handler: Currency

        Handler->>Handler: Create new wallet { Credit: 50, Debit: 0, HistoricalCredit: 50 }
        Handler->>Handler: OldBalance = "0", NewBalance = "50"
    end

    Handler->>WalletRepo: UpsertAsync(wallet)
    WalletRepo->>MongoDB: Update/Insert wallet
    MongoDB-->>WalletRepo: Success

    Handler->>Handler: Create transaction { Type: Credit, Amount: 50, OldBalance, NewBalance }
    Handler->>TxRepo: UpsertAsync(transaction)
    TxRepo->>MongoDB: Insert transaction
    MongoDB-->>TxRepo: Success

    Handler->>EventService: Publish(WalletTransactionCreated)
    EventService-->>Handler: Event published

    Handler-->>API: WalletBalanceVm { Credit: 150, Debit: 0, TransactionId: "tx123" }
    API-->>Client: 200 OK
```

### Future TigerBeetle Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as AddCreditCommandHandler
    participant Repo as TigerBeetleWalletRepository
    participant Registry as AccountRegistry
    participant TB as TigerBeetle
    participant EventBus as MassTransit

    Client->>API: POST /credit { amount: 50, clientId: "client123", country: "USA", currency: "USD" }
    API->>Handler: AddCreditCommand

    Handler->>Repo: AddCredit(client123, USA, USD, 50, null)
    Repo->>Repo: accountId = Hash(client123+USA+USD+null)

    Repo->>TB: lookup_accounts([accountId])

    alt Account Exists
        TB-->>Repo: Account { credits_posted: 15000, debits_posted: 18000 }
        Note over Repo: Current balance = 150 - 180 = -30 (has $30 debt)

        Repo->>Repo: Create transfer to add $50 credit
        Note over Repo: Transfer: System Reserve Account → Customer Account

        Repo->>TB: create_transfers([{<br/>  id: GenerateUUID(),<br/>  debit_account_id: SYSTEM_RESERVE_USD,<br/>  credit_account_id: accountId,<br/>  amount: 5000 (cents),<br/>  ledger: USD_LEDGER,<br/>  code: CREDIT_OPERATION,<br/>  user_data_128: Hash(referenceId)<br/>}])

        TB->>TB: Atomic double-entry transfer
        Note over TB: System Reserve: debits_posted += 5000<br/>Customer: credits_posted += 5000
        TB-->>Repo: TransferResult { success }

        Repo->>TB: lookup_accounts([accountId])
        TB-->>Repo: Account { credits_posted: 20000, debits_posted: 18000 }
        Note over Repo: New balance = 200 - 180 = +20

    else Account Doesn't Exist
        TB-->>Repo: Account not found

        Repo->>TB: create_accounts([{<br/>  id: accountId,<br/>  ledger: USD_LEDGER,<br/>  code: CUSTOMER_WALLET,<br/>  user_data_128: Hash(clientId),<br/>  user_data_64: EncodeCountryIssuer(USA, null),<br/>  flags: DEBITS_MUST_NOT_EXCEED_CREDITS<br/>}])

        TB-->>Repo: AccountResult { success }

        Repo->>Registry: Register(clientId, accountId, USA, USD, null)
        Registry-->>Repo: Success

        Repo->>TB: create_transfers([{<br/>  debit_account_id: SYSTEM_RESERVE_USD,<br/>  credit_account_id: accountId,<br/>  amount: 5000<br/>}])

        TB-->>Repo: TransferResult { success }
    end

    Repo->>Repo: Calculate Credit/Debit for response
    Repo-->>Handler: WalletBalance { Credit: 20, Debit: 0, TransactionId }

    Handler->>EventBus: Publish(WalletTransactionCreated)
    EventBus-->>Handler: Event published

    Handler-->>API: WalletBalanceVm
    API-->>Client: 200 OK
```

---

## 4. SubtractCredits (Debit) Operation

### Current MongoDB Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as RemoveCreditsCommandHandler
    participant Repo as PaymentsWalletRepository
    participant TxRepo as PaymentsWalletTransactionRepository
    participant EventService as DomainEventService
    participant MongoDB

    Client->>API: PUT /debit { amount: 75, clientId: "client123", country: "USA", currency: "USD", referenceId: "pay_123" }
    API->>Handler: RemoveCreditsCommand

    Handler->>Repo: FindBy(client123, USA, USD, null)
    Repo->>MongoDB: Find wallet

    alt Sufficient Credit
        MongoDB-->>Repo: PaymentsWallet { Credit: 100, Debit: 0 }
        Repo-->>Handler: Existing wallet

        Handler->>Handler: Credit = 100 - 75 = 25
        Handler->>Handler: OldBalance = "100", NewBalance = "25"

    else Insufficient Credit (goes negative)
        MongoDB-->>Repo: PaymentsWallet { Credit: 50, Debit: 0 }
        Repo-->>Handler: Existing wallet

        Handler->>Handler: Deficit = 75 - 50 = 25
        Handler->>Handler: Credit = 0, Debit = 25
        Handler->>Handler: OldBalance = "50", NewBalance = "-25"
    end

    Handler->>Handler: Create transaction { Type: Debit, Amount: 75, ReferenceId: "pay_123" }

    Handler->>TxRepo: UpsertAsync(transaction)
    TxRepo->>MongoDB: Insert transaction
    Handler->>Repo: UpsertAsync(wallet)
    Repo->>MongoDB: Update wallet

    MongoDB-->>Repo: Success
    MongoDB-->>TxRepo: Success

    Handler->>EventService: Publish(WalletTransactionCreated)

    Handler-->>API: WalletBalanceVm { Credit: 0, Debit: 25, TransactionId: "tx456" }
    API-->>Client: 200 OK
```

### Future TigerBeetle Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as RemoveCreditsCommandHandler
    participant Repo as TigerBeetleWalletRepository
    participant TB as TigerBeetle
    participant EventBus as MassTransit

    Client->>API: PUT /debit { amount: 75, referenceId: "pay_123" }
    API->>Handler: RemoveCreditsCommand

    Handler->>Repo: SubtractCredit(client123, USA, USD, 75, "pay_123", null)
    Repo->>Repo: accountId = Hash(client123+USA+USD+null)

    Repo->>TB: lookup_accounts([accountId])
    TB-->>Repo: Account { credits_posted: 10000, debits_posted: 5000 }
    Note over Repo: Current balance = 100 - 50 = +50

    alt Allow Negative Balances (Overdraft)
        Note over Repo: Transfer: Customer Account → System Expense Account
        Repo->>TB: create_transfers([{<br/>  id: GenerateUUID(),<br/>  debit_account_id: accountId,<br/>  credit_account_id: SYSTEM_EXPENSE_USD,<br/>  amount: 7500 (cents),<br/>  ledger: USD_LEDGER,<br/>  code: DEBIT_OPERATION,<br/>  user_data_128: Hash("pay_123"),<br/>  flags: 0 (no restrictions)<br/>}])

        TB->>TB: Atomic transfer
        Note over TB: Customer: debits_posted += 7500<br/>System Expense: credits_posted += 7500
        TB-->>Repo: TransferResult { success }

        Repo->>TB: lookup_accounts([accountId])
        TB-->>Repo: Account { credits_posted: 10000, debits_posted: 12500 }
        Note over Repo: New balance = 100 - 125 = -25 (has $25 debt)

    else Enforce Positive Balance Only
        Note over Repo: Account has DEBITS_MUST_NOT_EXCEED_CREDITS flag
        Repo->>TB: create_transfers([{ amount: 7500, ... }])
        TB-->>Repo: TransferResult { error: exceeds_credits }
        Repo-->>Handler: InsufficientFundsException
        Handler-->>API: 400 Bad Request
        API-->>Client: Error: Insufficient funds
    end

    alt Success Case
        Repo->>Repo: Calculate Credit/Debit representation
        Note over Repo: Credit = 0, Debit = 25
        Repo-->>Handler: WalletBalance { Credit: 0, Debit: 25 }

        Handler->>EventBus: Publish(WalletTransactionCreated)
        Handler-->>API: WalletBalanceVm
        API-->>Client: 200 OK
    end
```

---

## 5. VoidTransaction Operation

### Current MongoDB Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as VoidTransactionCommandHandler
    participant TxRepo as PaymentsWalletTransactionRepository
    participant WalletRepo as PaymentsWalletRepository
    participant EventService as DomainEventService
    participant MongoDB

    Client->>API: POST /void { transactionId: "tx_abc123" }
    API->>Handler: VoidTransactionCommand

    Handler->>TxRepo: GetAsync("tx_abc123")
    TxRepo->>MongoDB: Find transaction
    MongoDB-->>TxRepo: PaymentsWalletTransaction { Type: Credit, Amount: 50, Wallet: {...} }
    TxRepo-->>Handler: Original transaction

    Handler->>WalletRepo: FindBy(clientId, country, currency, issuerIdentifier)
    WalletRepo->>MongoDB: Find wallet
    MongoDB-->>WalletRepo: PaymentsWallet { Credit: 150, Debit: 0, HistoricalCredit: 200 }
    WalletRepo-->>Handler: Current wallet

    Handler->>Handler: Calculate current balance: 150 - 0 = 150

    alt Void Credit (Type = CreditVoid)
        Handler->>Handler: balance = 150 - 50 = 100
        Handler->>Handler: HistoricalCredit = 200 - 50 = 150

        alt Balance >= 0
            Handler->>Handler: Credit = 100, Debit = 0
        else Balance < 0
            Handler->>Handler: Credit = 0, Debit = 50
        end

    else Void Debit (Type = DebitVoid)
        Handler->>Handler: balance = 150 + 50 = 200
        Handler->>Handler: Credit = 200, Debit = 0
    end

    Handler->>Handler: Create void transaction {<br/>  Type: CreditVoid,<br/>  Amount: 50,<br/>  ReferenceTransactionId: "tx_abc123"<br/>}

    Handler->>TxRepo: UpsertAsync(voidTransaction)
    TxRepo->>MongoDB: Insert void transaction

    Handler->>WalletRepo: UpsertAsync(wallet)
    WalletRepo->>MongoDB: Update wallet

    Handler->>Handler: Mark original as voided
    Handler->>TxRepo: UpsertAsync(originalTx { Voided: true, UpdatedAt: now })
    TxRepo->>MongoDB: Update original transaction

    MongoDB-->>TxRepo: Success (3 writes)

    Handler->>EventService: Publish(WalletTransactionCreated)

    Handler-->>API: WalletBalanceVm { Credit: 100, Debit: 0, TransactionType: CreditVoid }
    API-->>Client: 200 OK
```

### Future TigerBeetle Flow
```mermaid
sequenceDiagram
    participant Client
    participant API as WalletsController
    participant Handler as VoidTransactionCommandHandler
    participant Repo as TigerBeetleWalletRepository
    participant TxRegistry as TransferRegistry (MongoDB)
    participant TB as TigerBeetle
    participant EventBus as MassTransit

    Client->>API: POST /void { transactionId: "tx_abc123" }
    API->>Handler: VoidTransactionCommand

    Handler->>Repo: VoidTransaction("tx_abc123")

    Repo->>TxRegistry: GetTransferMetadata("tx_abc123")
    TxRegistry->>TxRegistry: Query MongoDB for transfer metadata
    TxRegistry-->>Repo: TransferMetadata {<br/>  TigerBeetleTransferId: 0x123...,<br/>  Type: Credit,<br/>  Amount: 5000,<br/>  AccountId: 0xabc...,<br/>  Voided: false<br/>}

    alt Transfer Already Voided
        TxRegistry-->>Repo: TransferMetadata { Voided: true }
        Repo-->>Handler: AlreadyVoidedException
        Handler-->>API: 400 Bad Request
        API-->>Client: Error: Transaction already voided
    end

    Repo->>TB: lookup_accounts([accountId])
    TB-->>Repo: Account { credits_posted: 20000, debits_posted: 5000 }
    Note over Repo: Current balance = 200 - 50 = +150

    alt Void Credit (Reverse: Customer → System Reserve)
        Repo->>TB: create_transfers([{<br/>  id: GenerateUUID(),<br/>  debit_account_id: accountId,<br/>  credit_account_id: SYSTEM_RESERVE_USD,<br/>  amount: 5000,<br/>  ledger: USD_LEDGER,<br/>  code: CREDIT_VOID,<br/>  user_data_128: Hash("tx_abc123"),<br/>  user_data_64: originalTransferId<br/>}])

        TB->>TB: Atomic void transfer
        Note over TB: Customer: debits_posted += 5000<br/>System Reserve: credits_posted += 5000
        TB-->>Repo: TransferResult { success }
        Note over Repo: New balance = 200 - 100 = +100<br/>HistoricalCredit decreased by $50

    else Void Debit (Reverse: System Expense → Customer)
        Repo->>TB: create_transfers([{<br/>  debit_account_id: SYSTEM_EXPENSE_USD,<br/>  credit_account_id: accountId,<br/>  amount: 5000,<br/>  code: DEBIT_VOID,<br/>  user_data_128: Hash("tx_abc123")<br/>}])

        TB-->>Repo: TransferResult { success }
        Note over Repo: New balance = 150 + 50 = +200
    end

    Repo->>TB: lookup_accounts([accountId])
    TB-->>Repo: Account { credits_posted: 20000, debits_posted: 10000 }

    Repo->>TxRegistry: MarkTransferAsVoided("tx_abc123", voidTransferId)
    TxRegistry->>TxRegistry: Update { Voided: true, VoidedAt: now, VoidTransferId: ... }
    TxRegistry-->>Repo: Success

    Repo->>Repo: Calculate new Credit/Debit
    Repo-->>Handler: WalletBalance { Credit: 100, Debit: 0 }

    Handler->>EventBus: Publish(WalletTransactionCreated)
    Handler-->>API: WalletBalanceVm
    API-->>Client: 200 OK
```

---

## Migration Strategy Diagrams

### Phase 1: Dual-Write Architecture
```mermaid
graph LR
    A[API Request] --> B[Application Layer]
    B --> C[Wallet Repository]
    C --> D[MongoDB Write]
    C --> E[TigerBeetle Write]
    C --> F[Read from MongoDB]
    F --> G[Response]

    E -.->|Async validation| H[Reconciliation Worker]
    D -.->|Async validation| H
    H --> I[Alert if mismatch]
```

### Phase 2: Shadow Read Architecture
```mermaid
graph LR
    A[API Request] --> B[Application Layer]
    B --> C[Wallet Repository]
    C --> D[MongoDB Read - PRIMARY]
    C -.->|Shadow read| E[TigerBeetle Read]
    D --> F[Response]
    E -.->|Async comparison| G[Metrics Logger]
    D -.->|Async comparison| G
    G --> H[Datadog/CloudWatch]
```

### Phase 3: Gradual Read Cutover
```mermaid
graph LR
    A[API Request] --> B[Application Layer]
    B --> C[Feature Flag Check]
    C -->|10% traffic| D[TigerBeetle Read]
    C -->|90% traffic| E[MongoDB Read]
    D --> F[Response]
    E --> F

    D -.->|Fallback on error| E
```

---

## Double-Entry Accounting Flows

### Add Credit Flow
```
System Reserve Account (Debit)     -$50
Customer Wallet (Credit)           +$50
────────────────────────────────────────
Net Balance Change:                  $0
```

### Subtract Credit (Debit) Flow
```
Customer Wallet (Debit)            -$30
System Expense Account (Credit)    +$30
────────────────────────────────────────
Net Balance Change:                  $0
```

### Void Credit Flow
```
Customer Wallet (Debit)            -$50
System Reserve Account (Credit)    +$50
────────────────────────────────────────
Net Balance Change:                  $0
```

### Void Debit Flow
```
System Expense Account (Debit)     -$30
Customer Wallet (Credit)           +$30
────────────────────────────────────────
Net Balance Change:                  $0
```

---

## Account Structure Diagram

```mermaid
graph TB
    subgraph "USD Ledger (Ledger ID: 1)"
        SR[System Reserve Account<br/>Code: 2000<br/>Credits: 0<br/>Debits: Posted credits given to customers]
        SE[System Expense Account<br/>Code: 3000<br/>Credits: Posted debits from customers<br/>Debits: 0]

        C1[Customer Wallet 1<br/>ID: Hash(client123+USA+USD)<br/>Credits Posted: Total added<br/>Debits Posted: Total spent]
        C2[Customer Wallet 2<br/>ID: Hash(client456+USA+USD)<br/>Credits Posted: Total added<br/>Debits Posted: Total spent]
    end

    subgraph "MXN Ledger (Ledger ID: 3)"
        SR2[System Reserve Account<br/>Code: 2000]
        SE2[System Expense Account<br/>Code: 3000]
        C3[Customer Wallet 3<br/>ID: Hash(client123+MEX+MXN)<br/>Credits Posted: Total added<br/>Debits Posted: Total spent]
    end

    SR -->|Add Credit Transfer| C1
    C1 -->|Debit Transfer| SE
    SR -->|Void Credit Transfer| C1
    SE -->|Void Debit Transfer| C1

    style SR fill:#90EE90
    style SR2 fill:#90EE90
    style SE fill:#FFB6C1
    style SE2 fill:#FFB6C1
    style C1 fill:#87CEEB
    style C2 fill:#87CEEB
    style C3 fill:#87CEEB
```

---

## Data Mapping Diagram

```mermaid
graph LR
    subgraph "MongoDB PaymentsWallet"
        M1[ClientId: client123]
        M2[Country: USA]
        M3[Currency: USD]
        M4[IssuerTypeIdentifier: VISA]
        M5[Credit: 100.00]
        M6[Debit: 0.00]
        M7[HistoricalCredit: 250.00]
    end

    subgraph "TigerBeetle Account"
        T1[id: UInt128 Hash]
        T2[ledger: 1 - USD Ledger]
        T3[code: 1000 - Customer Wallet]
        T4[user_data_128: Hash-clientId-]
        T5[user_data_64: Encode-Country+Issuer-]
        T6[credits_posted: 25000]
        T7[debits_posted: 15000]
    end

    subgraph "Account Registry MongoDB"
        R1[ClientId: client123]
        R2[TigerBeetleAccountId: UInt128]
        R3[CountryCode: USA]
        R4[CurrencyCode: USD]
        R5[IssuerTypeIdentifier: VISA]
    end

    M1 --> T4
    M2 --> T5
    M3 --> T2
    M4 --> T5
    M5 --> T6
    M5 --> T7
    M6 --> T6
    M6 --> T7
    M7 --> T6

    M1 --> R1
    T1 --> R2
    M2 --> R3
    M3 --> R4
    M4 --> R5

    style M5 fill:#FFE4B5
    style M6 fill:#FFE4B5
    style T6 fill:#98FB98
    style T7 fill:#98FB98
```

**Calculation:**
- MongoDB Balance: `Credit - Debit = 100 - 0 = $100`
- TigerBeetle Balance: `(credits_posted - debits_posted) / 100 = (25000 - 15000) / 100 = $100`
- MongoDB HistoricalCredit: `250.00`
- TigerBeetle HistoricalCredit: `credits_posted / 100 = 25000 / 100 = $250`

---

**Document Version:** 1.0
**Last Updated:** 2025-11-18
**Related:** TIGERBEETLE_WALLET_MIGRATION.md