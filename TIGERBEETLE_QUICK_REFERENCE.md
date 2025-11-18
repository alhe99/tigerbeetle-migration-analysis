# TigerBeetle Migration - Quick Reference Guide for Developers

**Last Updated:** 2025-11-18
**For:** Backend Developers working on wallet migration

---

## TL;DR - What You Need to Know

1. **What:** Migrating wallet system from MongoDB to TigerBeetle (high-performance financial DB)
2. **Why:** 5-10x faster, 200x throughput, built-in double-entry accounting, immutable audit trail
3. **When:** 4 months (4 phases: dual-write → shadow read → read cutover → full migration)
4. **How:** Clean Architecture compatible; MediatR pipeline behaviors for dual-write; feature flags for gradual rollout

---

## File Locations - Where to Find What

### Documentation (Project Root)
```
/TIGERBEETLE_WALLET_MIGRATION.md         # Full technical analysis (65 KB, read this first)
/TIGERBEETLE_MIGRATION_SUMMARY.md        # Executive summary (15 min read)
/TIGERBEETLE_MIGRATION_DIAGRAMS.md       # All sequence diagrams (visual reference)
/TIGERBEETLE_README.md                   # Documentation index
/TIGERBEETLE_DOTNET_IMPLEMENTATION.md    # .NET implementation guide (35 KB, detailed code)
/TIGERBEETLE_IMPLEMENTATION_SUMMARY.md   # Backend developer review & recommendations
/TIGERBEETLE_QUICK_REFERENCE.md          # This file (quick start guide)
```

### Current Wallet Code
```
src/Domain/Entities/Mongo/Wallet/
  PaymentsWallet.cs                      # Wallet entity (ClientId, Credit, Debit, HistoricalCredit)
  PaymentsWalletTransaction.cs           # Transaction log entity

src/Infrastructure/Persistence/Repositories/Mongo/Wallet/
  PaymentsWalletRepository.cs            # MongoDB repository (FindBy, GetBalance, GetWalletsBy)

src/Application/Wallets/
  Commands/AddCredits/AddCreditCommand.cs          # Add credit handler
  Commands/RemoveCredits/RemoveCreditsCommand.cs   # Subtract credit handler
  Commands/Void/VoidTrasactionCommand.cs           # Void transaction handler
  Queries/GetBalance/GetWalletBalanceQuery.cs      # Get balance handler
  Queries/GetMultiBalance/GetMultiBalanceQuery.cs  # Get multi-balance handler

src/WebUI/Controllers/
  WalletsController.cs                   # API endpoints (5 operations)
```

### Future TigerBeetle Code (To Be Added)
```
src/Infrastructure/Persistence/TigerBeetle/
  TigerBeetleClient.cs                   # TigerBeetle client wrapper
  TigerBeetleWalletRepository.cs         # Main repository implementation
  AccountIdGenerator.cs                  # SHA-256 composite key to UInt128
  LedgerManager.cs                       # Currency + country → Ledger ID
  CurrencyConverter.cs                   # Decimal to cents (smallest unit)
  SystemAccountManager.cs                # System account initialization
  AccountRegistry.cs                     # MongoDB registry for ClientId lookups
  TransferRegistry.cs                    # MongoDB registry for void state

src/Application/Common/Behaviors/
  DualWritePipelineBehavior.cs           # MediatR behavior for transparent dual-write

src/Infrastructure/BackgroundServices/
  WalletReconciliationWorker.cs          # Balance validation worker (Phase 1-2)

tests/Application.IntegrationTests/TigerBeetle/
  TigerBeetleTestFixture.cs              # Test infrastructure
  TigerBeetleWalletRepositoryTests.cs    # Integration tests
```

---

## Key Concepts - Understanding TigerBeetle

### 1. Accounts (Balances)

MongoDB Wallet:
```csharp
public class PaymentsWallet
{
    public string ClientId { get; set; }
    public decimal Credit { get; set; }        // Positive balance
    public decimal Debit { get; set; }         // Debt owed
    public decimal HistoricalCredit { get; set; }
}

// Net Balance = Credit - Debit
```

TigerBeetle Account:
```csharp
var account = new Account
{
    Id = UInt128 accountId,               // Generated from ClientId+Country+Currency+Issuer
    Ledger = uint ledgerId,               // Currency+Country partition
    CreditsPosted = ulong,                // Total credits added (in cents)
    DebitsPosted = ulong,                 // Total debits spent (in cents)
};

// Net Balance = (CreditsPosted - DebitsPosted) / 100   (convert cents to dollars)
// Credit = netBalance >= 0 ? netBalance : 0
// Debit = netBalance < 0 ? Math.Abs(netBalance) : 0
// HistoricalCredit = CreditsPosted / 100
```

### 2. Transfers (Money Movements)

MongoDB Transaction:
```csharp
var transaction = new PaymentsWalletTransaction
{
    Amount = 50.00m,
    Type = TransactionType.Credit,  // or Debit, CreditVoid, DebitVoid
    OldBalance = "100",
    NewBalance = "150",
    ReferenceId = "payment_abc123"
};
```

TigerBeetle Transfer:
```csharp
var transfer = new Transfer
{
    Id = UInt128 transferId,              // Unique transfer ID
    DebitAccountId = UInt128,             // Account being debited (money out)
    CreditAccountId = UInt128,            // Account being credited (money in)
    Amount = 5000,                        // $50.00 in cents
    Ledger = uint ledgerId,               // Must match both accounts
    Code = (ushort)TransferCode.Credit,  // Credit, Debit, CreditVoid, DebitVoid
    UserData128 = UInt128 referenceId    // Store MongoDB transaction ID (hashed)
};
```

### 3. Double-Entry Accounting

Every operation creates two entries (debit + credit):

**Add Credit ($50):**
```
Debit:  System Reserve Account    -$50
Credit: Customer Wallet            +$50
────────────────────────────────────────
Net:                               $0    (money conserved)
```

**Subtract Credit ($30):**
```
Debit:  Customer Wallet            -$30
Credit: System Expense Account     +$30
────────────────────────────────────────
Net:                               $0    (money conserved)
```

**Invariant:** Sum(All Debits) = Sum(All Credits) across entire system

---

## Code Examples - Common Operations

### 1. Generate Account ID

```csharp
// MongoDB composite key
var wallet = await _repo.FindBy(clientId, country, currency, issuerIdentifier);

// TigerBeetle account ID
var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuerIdentifier);
// Returns UInt128 (SHA-256 hash of "clientId:country:currency:issuer")
```

### 2. Get Balance

**MongoDB:**
```csharp
var wallet = await _mongoRepo.GetBalance(clientId, country, currency, issuer);
var netBalance = wallet.Credit - wallet.Debit;  // Example: 100 - 0 = 100
```

**TigerBeetle:**
```csharp
var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer);
var accounts = await _client.LookupAccountsAsync(new[] { accountId });
var account = accounts[0];

var netBalanceInCents = (long)account.CreditsPosted - (long)account.DebitsPosted;
var netBalance = _currencyConverter.FromSmallestUnit((ulong)Math.Abs(netBalanceInCents), currency);

var credit = netBalanceInCents >= 0 ? netBalance : 0;
var debit = netBalanceInCents < 0 ? netBalance : 0;
```

### 3. Add Credit

**MongoDB:**
```csharp
var wallet = await _repo.FindBy(clientId, country, currency, issuer);
if (wallet == null)
{
    wallet = new PaymentsWallet { ClientId = clientId, Credit = 50, HistoricalCredit = 50 };
}
else
{
    wallet.Credit += 50;
    wallet.HistoricalCredit += 50;
}
await _repo.UpsertAsync(wallet);
await _txRepo.UpsertAsync(new PaymentsWalletTransaction { Amount = 50, Type = Credit });
```

**TigerBeetle:**
```csharp
var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer);
var ledgerId = _ledgerManager.GetLedgerId(currency, country);

// Check if account exists
var accounts = await _client.LookupAccountsAsync(new[] { accountId });
if (accounts.Length == 0)
{
    // Create account
    await _client.CreateAccountsAsync(new[]
    {
        new Account { Id = accountId, Ledger = ledgerId, Code = 1000 }
    });
    await _accountRegistry.RegisterAccountAsync(clientId, accountId, country, currency, issuer, ledgerId);
}

// Create transfer (System Reserve → Customer)
var transfer = new Transfer
{
    Id = GenerateTransferId(),
    DebitAccountId = _systemAccountManager.GetSystemReserveAccountId(ledgerId),
    CreditAccountId = accountId,
    Amount = _currencyConverter.ToSmallestUnit(50.00m, currency),  // 5000 cents
    Ledger = ledgerId,
    Code = (ushort)TransferCode.Credit,
    UserData128 = HashReferenceId(transactionId)
};

await _client.CreateTransfersAsync(new[] { transfer });
await _transferRegistry.RegisterTransferAsync(new TransferRegistryEntry { TransactionId = transactionId, ... });
```

### 4. Currency Conversion

```csharp
// Decimal to smallest unit (cents)
var amountInCents = _currencyConverter.ToSmallestUnit(100.00m, "USD");  // 10000

// Smallest unit to decimal
var amountInDollars = _currencyConverter.FromSmallestUnit(10000, "USD"); // 100.00

// Supports different decimal places
_currencyConverter.ToSmallestUnit(100.00m, "JPY");  // 100 (no decimals for Yen)
_currencyConverter.ToSmallestUnit(1.00m, "BTC");    // 100000000 (8 decimals for Bitcoin)
```

### 5. Dual-Write Pattern (Phase 1)

**Option A: Manual (Not Recommended):**
```csharp
// In command handler
var mongoResult = await _mongoHandler.Handle(request, cancellationToken);

if (_options.Value.EnableDualWrite)
{
    try
    {
        await _tigerBeetleRepo.AddCreditAsync(...);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "TigerBeetle dual-write failed");
        // Don't fail request - MongoDB is authoritative
    }
}

return mongoResult;
```

**Option B: MediatR Pipeline Behavior (Recommended):**
```csharp
// In DualWritePipelineBehavior.cs
public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
{
    // Execute MongoDB write
    var response = await next();

    // Dual-write to TigerBeetle (if enabled)
    if (_options.Value.EnableDualWrite && IsWalletCommand(request))
    {
        _ = Task.Run(() => WriteToTigerBeetleAsync(request, response), ct);
    }

    return response;
}

// No changes to existing handlers needed!
```

---

## Configuration - appsettings.json

```json
{
  "TigerBeetle": {
    "ClusterId": 0,
    "ReplicaAddresses": [
      "tigerbeetle-0.internal:3000",
      "tigerbeetle-1.internal:3000",
      "tigerbeetle-2.internal:3000"
    ],
    "EnableDualWrite": false,     // Set to true in Phase 1
    "EnableShadowRead": false,    // Set to true in Phase 2
    "ReadCacheTtlSeconds": 60
  },
  "FeatureFlags": {
    "WalletTigerBeetleRead": {
      "Enabled": false,           // Gradual rollout in Phase 3: 0% → 10% → 50% → 90% → 100%
      "RolloutPercentage": 0
    }
  }
}
```

---

## Testing - How to Run Tests

### Unit Tests (Fast)

```bash
# Run all domain unit tests
dotnet test tests/Domain.UnitTests/Domain.UnitTests.csproj

# Run specific test class
dotnet test --filter "FullyQualifiedName~AccountIdGeneratorTests"
```

### Integration Tests (Requires MongoDB + TigerBeetle)

```bash
# Start TigerBeetle test server (in separate terminal)
tigerbeetle format --cluster=0 --replica=0 /tmp/tigerbeetle_test.data
tigerbeetle start --addresses=127.0.0.1:3000 /tmp/tigerbeetle_test.data

# Run all integration tests
dotnet test tests/Application.IntegrationTests/Application.IntegrationTests.csproj

# Run TigerBeetle-specific tests
dotnet test --filter "FullyQualifiedName~TigerBeetleWalletRepositoryTests"
```

### Test Data Helpers

```csharp
// Use existing DataCreationHelper for test data
var paymentRequest = DataCreationHelper.PaymentRequest;
paymentRequest.Order.Country = "USA";
paymentRequest.Order.Category = "hugoBusiness";

// Use Testing.cs helper methods
await SendAsync(new AddCreditCommand { ClientId = "test-client", Amount = 100 });
var wallet = await MongoFindAsync<PaymentsWallet>(walletId);
await ResetState();  // Clean MongoDB + TigerBeetle between tests
```

---

## Common Tasks - Copy-Paste Snippets

### Task 1: Add New Ledger for Currency/Country

```csharp
// In LedgerManager.cs
private readonly Dictionary<string, uint> _ledgerIds = new()
{
    { "USD:USA", 1 },
    { "USD:SLV", 10 },
    { "MXN:MEX", 3 },
    { "NEW_CURRENCY:NEW_COUNTRY", 11 }  // Add here
};
```

### Task 2: Initialize System Accounts for Ledger

```csharp
// In SystemAccountManager.cs
await InitializeLedgerAsync(ledgerId: 11, currency: "NEW_CURRENCY", country: "NEW_COUNTRY");
```

### Task 3: Check Balance Match Rate (Reconciliation)

```csharp
// In WalletReconciliationWorker.cs
var mongoBalance = wallet.Credit - wallet.Debit;
var tbBalance = await _tigerBeetleRepo.GetBalanceAsync(wallet.ClientId, wallet.Country, wallet.Currency, wallet.IssuerTypeIdentifier);
var tbNetBalance = tbBalance.Credit - tbBalance.Debit;

if (Math.Abs(mongoBalance - tbNetBalance) > 0.01m)
{
    _logger.LogError("Balance mismatch: {WalletId}, Mongo={Mongo}, TB={TB}",
        wallet.Id, mongoBalance, tbNetBalance);
}
```

### Task 4: Query TigerBeetle Account Details

```csharp
var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer);
var accounts = await _client.LookupAccountsAsync(new[] { accountId });

if (accounts.Length > 0)
{
    var account = accounts[0];
    Console.WriteLine($"Credits Posted: {account.CreditsPosted}");
    Console.WriteLine($"Debits Posted: {account.DebitsPosted}");
    Console.WriteLine($"Balance: {(long)account.CreditsPosted - (long)account.DebitsPosted}");
}
```

### Task 5: Handle TigerBeetle Transfer Errors

```csharp
var errors = await _client.CreateTransfersAsync(transfers);

if (errors.Length > 0)
{
    var error = errors[0];
    switch (error.Result)
    {
        case CreateTransferResult.ExceedsCredits:
            throw new InsufficientFundsException("Insufficient balance");

        case CreateTransferResult.AccountsMustHaveTheSameLedger:
            throw new TigerBeetleException("Cross-ledger transfer not allowed");

        case CreateTransferResult.LinkedEventFailed:
            // Retry logic
            break;

        default:
            _logger.LogError("Transfer failed: {ErrorCode}", error.Result);
            throw new TransferFailedException(error.Result);
    }
}
```

---

## Troubleshooting - Common Issues

### Issue 1: "Account not found" error

**Cause:** Account hasn't been created yet

**Solution:**
```csharp
var accounts = await _client.LookupAccountsAsync(new[] { accountId });
if (accounts.Length == 0)
{
    // Create account before creating transfer
    await CreateAccountAsync(accountId, clientId, country, currency, issuer, ledgerId);
}
```

### Issue 2: Balance mismatch between MongoDB and TigerBeetle

**Cause:** Currency conversion error or missing dual-write

**Debug:**
```csharp
// Check MongoDB balance
var mongoWallet = await _mongoRepo.FindBy(clientId, country, currency, issuer);
Console.WriteLine($"MongoDB: Credit={mongoWallet.Credit}, Debit={mongoWallet.Debit}");

// Check TigerBeetle balance
var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer);
var accounts = await _client.LookupAccountsAsync(new[] { accountId });
var account = accounts[0];
Console.WriteLine($"TigerBeetle: Credits={account.CreditsPosted}, Debits={account.DebitsPosted}");

// Check conversion
var tbBalanceInDollars = (account.CreditsPosted - account.DebitsPosted) / 100m;
Console.WriteLine($"TB Balance (converted): {tbBalanceInDollars}");
```

### Issue 3: "Transfer exceeds credits" error

**Cause:** Account has `DebitsMustNotExceedCredits` flag set (prevents overdraft)

**Solution:**
```csharp
// Option A: Remove flag to allow negative balances (current behavior)
Flags = AccountFlags.None

// Option B: Check balance before creating transfer
var account = await _client.LookupAccountsAsync(new[] { accountId });
var balance = (long)account[0].CreditsPosted - (long)account[0].DebitsPosted;
if (balance < amountToDebit)
{
    throw new InsufficientFundsException(amountToDebit, balance);
}
```

### Issue 4: TigerBeetle client connection timeout

**Cause:** TigerBeetle cluster not reachable or replica addresses incorrect

**Debug:**
```bash
# Check if TigerBeetle is running
telnet tigerbeetle-0.internal 3000

# Check configuration
cat appsettings.json | grep TigerBeetle -A 10

# Check logs
docker logs tigerbeetle-0
```

### Issue 5: Account ID collision (very rare)

**Cause:** SHA-256 collision (probability: ~1 in 2^128)

**Solution:**
```csharp
// Add nonce-based rehashing
public UInt128 Generate(string clientId, string country, string currency, string issuer, int nonce = 0)
{
    var compositeKey = $"{clientId}:{country}:{currency}:{issuer ?? "null"}:{nonce}";
    // Hash and return UInt128
}

// Detect collision and retry
var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer);
var existingAccount = await _accountRegistry.GetAccountAsync(accountId);
if (existingAccount != null && existingAccount.ClientId != clientId)
{
    // Collision detected! Retry with nonce
    accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer, nonce: 1);
}
```

---

## Performance Tips

### 1. Use Caching for Account Registry

```csharp
// Cache ClientId → AccountIDs mapping
var cached = await _cache.GetAsync($"account-registry:{clientId}");
if (cached != null)
{
    return JsonSerializer.Deserialize<List<AccountMapping>>(cached);
}

// Cache miss - fetch from MongoDB
var accounts = await _accountRegistry.GetAccountsForClientAsync(clientId);
await _cache.SetAsync($"account-registry:{clientId}", JsonSerializer.Serialize(accounts), TimeSpan.FromMinutes(30));
```

### 2. Batch Account Lookups

```csharp
// Don't do this (N queries)
foreach (var clientId in clientIds)
{
    var accountId = _accountIdGenerator.Generate(clientId, country, currency, issuer);
    var accounts = await _client.LookupAccountsAsync(new[] { accountId });  // N round trips
}

// Do this (1 batch query)
var accountIds = clientIds.Select(c => _accountIdGenerator.Generate(c, country, currency, issuer)).ToArray();
var accounts = await _client.LookupAccountsAsync(accountIds);  // 1 round trip (up to 8,191 accounts)
```

### 3. Use ValueTask for Hot Paths

```csharp
// Reduces allocations for frequently called methods
public async ValueTask<WalletBalanceVm> GetBalanceAsync(...)
{
    // Implementation
}
```

### 4. Parallelize Independent Operations

```csharp
// Don't do this (sequential)
var wallet = await _mongoRepo.GetBalance(clientId, country, currency, issuer);
var tbWallet = await _tigerBeetleRepo.GetBalanceAsync(clientId, country, currency, issuer);

// Do this (parallel)
var mongoTask = _mongoRepo.GetBalance(clientId, country, currency, issuer);
var tbTask = _tigerBeetleRepo.GetBalanceAsync(clientId, country, currency, issuer);
await Task.WhenAll(mongoTask, tbTask);
var wallet = await mongoTask;
var tbWallet = await tbTask;
```

---

## Migration Phase Checklist - What Developers Need to Do

### Phase 1: Dual-Write (Weeks 1-4)

**Your Tasks:**
- [ ] Implement `TigerBeetleWalletRepository` with all 5 operations
- [ ] Implement `AccountRegistry` and `TransferRegistry` (MongoDB collections)
- [ ] Add `DualWritePipelineBehavior` to MediatR pipeline
- [ ] Write integration tests for TigerBeetle operations
- [ ] Deploy with `EnableDualWrite: false` initially
- [ ] Enable for 1% traffic and monitor reconciliation

**No changes to:**
- Controllers
- Existing command/query handlers
- DTOs or view models

### Phase 2: Shadow Read (Weeks 5-8)

**Your Tasks:**
- [ ] Add shadow read logic to `GetWalletBalanceQueryHandler`
- [ ] Add shadow read logic to `GetMultiBalanceQueryHandler`
- [ ] Emit metrics to Datadog for comparison
- [ ] Create dashboards for match rate, latency comparison

**No changes to:**
- API responses (still return MongoDB data)
- Controllers

### Phase 3: Read Cutover (Weeks 9-12)

**Your Tasks:**
- [ ] Add feature flag check to query handlers
- [ ] Add MongoDB fallback logic on TigerBeetle errors
- [ ] Monitor fallback rate (should be <0.1%)
- [ ] Gradually increase rollout: 10% → 50% → 90% → 100%

**Changes to:**
- `GetWalletBalanceQueryHandler` (feature flag + fallback)
- `GetMultiBalanceQueryHandler` (feature flag + fallback)

### Phase 4: Full Migration (Weeks 13-16)

**Your Tasks:**
- [ ] Set `EnableDualWrite: false` (stop MongoDB writes)
- [ ] Archive MongoDB wallet collections (`mongodump`)
- [ ] Remove MongoDB repository code
- [ ] Remove dual-write pipeline behavior
- [ ] Update documentation

---

## Resources & Links

### Documentation
- **TigerBeetle Official Docs:** https://docs.tigerbeetle.com/
- **TigerBeetle C# Client:** https://docs.tigerbeetle.com/clients/dotnet/
- **TigerBeetle GitHub:** https://github.com/tigerbeetle/tigerbeetle

### Internal Documentation
- **Full Migration Analysis:** `/TIGERBEETLE_WALLET_MIGRATION.md`
- **.NET Implementation Guide:** `/TIGERBEETLE_DOTNET_IMPLEMENTATION.md`
- **Executive Summary:** `/TIGERBEETLE_MIGRATION_SUMMARY.md`

### Team Contacts
- **Tech Lead:** [TBD]
- **API Architect:** [Auto-generated documentation]
- **Backend Developer (you):** [Leading implementation]
- **Platform Engineering:** [TBD - for cluster setup]

### Support Channels
- **Slack:** #payments-backend-migration
- **TigerBeetle Community Slack:** https://join.slack.com/t/tigerbeetle/...
- **Weekly Standup:** Thursdays 10:00 AM (during migration)

---

## FAQ - Frequently Asked Questions

**Q: Do I need to change the API endpoints?**
A: No, controllers remain unchanged. All migration happens in repositories and handlers.

**Q: Will tests break during migration?**
A: Existing tests continue to work. Add new tests for TigerBeetle operations.

**Q: How do I test TigerBeetle locally?**
A: Download `tigerbeetle` binary, run `tigerbeetle start`, point client to `127.0.0.1:3000`.

**Q: What if TigerBeetle is down?**
A: During Phase 3, MongoDB fallback logic ensures no service disruption. After Phase 4, TigerBeetle is critical (99.99% SLA with 3-node cluster).

**Q: Can I query TigerBeetle by ClientId?**
A: No, TigerBeetle only supports lookups by account ID. Use Account Registry (MongoDB) to map ClientId → AccountIDs.

**Q: How do I void a transaction?**
A: Create a compensating transfer (reverse direction) and mark original as voided in Transfer Registry.

**Q: What about currency conversion?**
A: Use `CurrencyConverter` service. TigerBeetle stores amounts in smallest unit (cents, satoshis, etc.).

**Q: How do I monitor migration progress?**
A: Check reconciliation worker logs, Datadog dashboards for match rate, error rate, latency comparison.

---

## Quick Commands - Copy-Paste Ready

```bash
# Start TigerBeetle locally
tigerbeetle format --cluster=0 --replica=0 /tmp/tb_test.data
tigerbeetle start --addresses=127.0.0.1:3000 /tmp/tb_test.data

# Run tests
dotnet test tests/Application.IntegrationTests/Application.IntegrationTests.csproj --filter "FullyQualifiedName~TigerBeetle"

# Check configuration
cat appsettings.json | jq '.TigerBeetle'

# View reconciliation logs
docker logs wallet-reconciliation-worker --tail 100 --follow

# Query MongoDB wallet
mongo --eval 'db.wallet.findOne({ClientId: "client123", "Country.CountryCode": "USA", "Currency.Code": "USD"})'

# Query TigerBeetle account
# (Use .NET app or CLI tool)
```

---

**Last Updated:** 2025-11-18
**Version:** 1.0
**For Questions:** Contact Tech Lead or check #payments-backend-migration Slack channel

**Happy Coding!**