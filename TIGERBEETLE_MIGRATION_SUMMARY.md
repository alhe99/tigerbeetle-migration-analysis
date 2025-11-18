# TigerBeetle Wallet Migration - Executive Summary

**Project:** MongoDB to TigerBeetle Migration for Wallet System
**Status:** Planning Phase
**Timeline:** 4 months (3 months migration + 1 month stabilization)
**Risk Level:** Medium
**Impact:** High Performance Improvement

---

## Quick Facts

| Metric | Current (MongoDB) | Target (TigerBeetle) | Improvement |
|--------|-------------------|----------------------|-------------|
| **Balance Query Latency (p95)** | 5-10ms | <1ms | 5-10x faster |
| **Write Latency (p99)** | 50-100ms | <5ms | 10-20x faster |
| **Max Throughput** | ~5,000 tx/sec | ~1,000,000 tx/sec | 200x |
| **Infrastructure Cost** | Baseline | -50% projected | Lower ops complexity |
| **ACID Guarantees** | Single document | Distributed + double-entry | Stronger |

---

## Why TigerBeetle?

### Current Pain Points (MongoDB)
1. **Balance Calculation Complexity:** Credit/Debit fields require manual balance computation
2. **No Double-Entry Guarantees:** Possible to lose money due to application bugs
3. **Performance Bottlenecks:** Document-based storage not optimized for financial transactions
4. **Audit Trail Gaps:** Transactions stored separately from balance changes
5. **Scaling Challenges:** Sharding required for high throughput scenarios

### TigerBeetle Advantages
1. **Built-in Double-Entry Accounting:** Impossible to create/destroy money
2. **Immutable Audit Trail:** All transfers are permanent and auditable
3. **10-20x Lower Latency:** Sub-millisecond account lookups
4. **200x Higher Throughput:** 1M transactions/second capacity
5. **Simpler Infrastructure:** No sharding needed, built-in consensus protocol
6. **Financial-Grade Consistency:** Strict serializability with Jepsen-verified correctness

---

## Current Wallet System (Stable - develop branch)

### Entity Structure

**PaymentsWallet:**
- **Composite Key:** ClientId + Country + Currency + IssuerTypeIdentifier
- **Balance Fields:** Credit (positive), Debit (negative), HistoricalCredit
- **Net Balance:** `Credit - Debit`

**PaymentsWalletTransaction:**
- **Types:** Credit, Debit, CreditVoid, DebitVoid
- **Fields:** Amount, OldBalance, NewBalance, ReferenceId, Voided flag

### Operations (5 total)
1. **GetBalance:** Query single wallet balance by composite key
2. **GetMultiBalance:** Query all wallets for a client
3. **AddCredits:** Add funds (with automatic debt paydown)
4. **SubtractCredits (Debit):** Spend funds (allow negative balance)
5. **VoidTransaction:** Reverse a previous credit or debit

### Current Performance
- **Average Load:** ~50 tx/sec
- **Peak Load:** ~500 tx/sec (events like Black Friday)
- **Latency:** 5-30ms depending on operation complexity

---

## TigerBeetle Architecture

### Core Concepts

**1. Accounts**
- Represent wallet balances
- Store cumulative `credits_posted` and `debits_posted`
- Immutable except for balance fields
- 128-bit unique ID (generated from composite key hash)

**2. Transfers**
- Represent money movements between accounts
- Always involve exactly 2 accounts (debit + credit)
- Immutable once created
- Support pending (two-phase) transfers for authorization flows

**3. Ledgers**
- Partition accounts by currency/country
- Enforce same-ledger transfer rules (prevents cross-currency errors)
- One ledger per `Currency + Country` combination

### Double-Entry Accounting Model

Every operation creates a balanced transfer:

| Operation | Debit Account | Credit Account | Effect |
|-----------|---------------|----------------|--------|
| **Add Credit** | System Reserve | Customer Wallet | Customer gains balance |
| **Subtract Credit** | Customer Wallet | System Expense | Customer loses balance |
| **Void Credit** | Customer Wallet | System Reserve | Reverse add operation |
| **Void Debit** | System Expense | Customer Wallet | Reverse spend operation |

**Invariant:** `Sum(All Credits) = Sum(All Debits)` across entire system

---

## Data Model Mapping

### Composite Key → Account ID
```
MongoDB Composite Key:
  ClientId + CountryCode + CurrencyCode + IssuerTypeIdentifier

TigerBeetle Account ID:
  UInt128 = SHA256(ClientId:CountryCode:CurrencyCode:IssuerTypeIdentifier)

Example:
  "client123:USA:USD:VISA" → 0x7a3f8e2b9c4d1a6f5e8b7c2d9a3f8e2b
```

### Balance Representation
```
MongoDB:
  Credit: 100.00 (positive balance)
  Debit: 0.00 (debt owed)
  Net Balance: 100.00

TigerBeetle:
  credits_posted: 25000 (all credits added, in cents)
  debits_posted: 15000 (all debits spent, in cents)
  Net Balance: (25000 - 15000) / 100 = 100.00
  HistoricalCredit: 25000 / 100 = 250.00
```

### Wallet → Accounts Mapping

Each MongoDB wallet maps to **2 TigerBeetle accounts:**
1. **Customer Account** (tracks client balance)
2. **System Counterparty Account** (reserve or expense, depending on operation)

**System Accounts per Ledger:**
- System Reserve (credits source)
- System Expense (debits destination)
- System Fees (future)
- System Refunds (future)

---

## Migration Strategy (4 Months)

### Phase 1: Dual-Write (Weeks 1-4)
**Goal:** Write to both MongoDB and TigerBeetle, read from MongoDB only

**Activities:**
- Deploy TigerBeetle cluster (3 nodes, multi-AZ)
- Implement account/transfer repository
- Enable dual-write for 1% → 10% → 100% of traffic
- Run continuous reconciliation worker
- Monitor balance match rate (target: >99.99%)

**Success Criteria:**
- Zero TigerBeetle write failures for 7 consecutive days
- Balance mismatch rate <0.01%
- Reconciliation worker finds no data integrity issues

**Rollback:** Disable TigerBeetle writes via feature flag (instant)

---

### Phase 2: Shadow Read (Weeks 5-8)
**Goal:** Read from both databases, compare results, return MongoDB data

**Activities:**
- Implement shadow read logic in repository layer
- Log comparison metrics to Datadog/CloudWatch
- Analyze latency differences (TigerBeetle should be 5-10x faster)
- Fix any data transformation bugs

**Success Criteria:**
- Match rate >99.99% for 7 consecutive days
- TigerBeetle read latency <50% of MongoDB latency
- Error rate <0.1%

**Rollback:** Stop shadow reads (no impact, MongoDB still authoritative)

---

### Phase 3: Gradual Read Cutover (Weeks 9-12)
**Goal:** Shift read traffic from MongoDB to TigerBeetle using feature flags

**Activities:**
- Week 9: 10% traffic to TigerBeetle
- Week 10: 50% traffic to TigerBeetle
- Week 11: 90% traffic to TigerBeetle
- Week 12: 100% traffic to TigerBeetle (MongoDB fallback on error)
- Monitor fallback rate (should be <0.1%)

**Success Criteria:**
- Fallback rate <0.1% for 7 consecutive days
- p95 latency <2ms for TigerBeetle reads
- Zero customer-reported balance discrepancies

**Rollback:** Feature flag instantly reverts to MongoDB (with fallback logic)

---

### Phase 4: Full Migration & Cleanup (Weeks 13-16)
**Goal:** TigerBeetle becomes single source of truth, deprecate MongoDB wallets

**Activities:**
- Week 13: 100% reads on TigerBeetle, stop dual-write
- Week 14: MongoDB in read-only mode (for rollback safety)
- Week 15: Archive MongoDB wallet collections (mongodump)
- Week 16: Remove legacy code, update docs, final audit

**Success Criteria:**
- 14 days of stable TigerBeetle-only operation
- Final reconciliation report shows 100% match
- Audit log completeness verified

**Rollback:** Re-enable MongoDB writes + restore from archive (1-4 hours)

---

## Technical Implementation Details

### Account Registry (MongoDB)
**Purpose:** Enable "find by ClientId" queries (TigerBeetle has no secondary indexes)

**Schema:**
```javascript
{
  "_id": "client123",
  "accounts": [
    {
      "tigerbeetleAccountId": "0x7a3f8e2b...",
      "country": "USA",
      "currency": "USD",
      "issuerTypeIdentifier": "VISA",
      "ledgerId": 1,
      "createdAt": ISODate("2025-11-18T00:00:00Z")
    },
    { /* additional wallets */ }
  ]
}
```

**Index:** `{ "_id": 1 }` (unique on ClientId)

---

### Transfer Registry (MongoDB)
**Purpose:** Track void state and application-level transaction IDs

**Schema:**
```javascript
{
  "_id": "tx_abc123",  // Application transaction ID
  "tigerbeetleTransferId": "0x9f3e2d1c...",
  "accountId": "0x7a3f8e2b...",
  "type": "Credit",
  "amount": 5000,  // cents
  "referenceId": "payment_xyz789",
  "voided": false,
  "voidTransferId": null,
  "createdAt": ISODate("2025-11-18T12:00:00Z")
}
```

**Indexes:**
- `{ "_id": 1 }` (unique on transaction ID)
- `{ "tigerbeetleTransferId": 1 }` (unique)
- `{ "referenceId": 1 }`
- `{ "voided": 1, "createdAt": -1 }`

---

### Ledger Assignment
```csharp
Dictionary<string, uint> _ledgerIds = new()
{
    { "USD:USA", 1 },
    { "USD:SLV", 10 },
    { "MXN:MEX", 3 },
    { "EUR:ESP", 4 },
    { "GTQ:GTM", 5 },
    { "HNL:HND", 6 },
    { "NIO:NIC", 7 },
    { "CRC:CRI", 8 },
    { "PEN:PER", 9 }
};
```

**Why separate ledgers?**
- Prevents accidental cross-currency transfers
- Enables country-specific compliance rules
- Simplifies reporting and reconciliation

---

### Currency Precision
**Challenge:** TigerBeetle uses unsigned 128-bit integers (no decimals)

**Solution:** Store amounts in smallest unit (cents, satoshis, etc.)

```csharp
// Conversion
decimal amount = 50.00m;  // $50.00 USD
ulong amountInCents = (ulong)(amount * 100);  // 5000 cents

// Storage in TigerBeetle
Transfer transfer = new()
{
    Amount = 5000,  // cents
    // ...
};

// Retrieval
decimal balance = account.CreditsPosted / 100m;  // Convert back to dollars
```

**Precision Guarantees:**
- No floating-point rounding errors
- Exact balance calculations
- Supports currencies with 0-8 decimal places

---

## Risk Assessment & Mitigation

### High Risks

**1. Data Loss During Migration**
- **Impact:** Critical (customer funds affected)
- **Probability:** Low (with dual-write + reconciliation)
- **Mitigation:**
  - MongoDB remains active throughout migration
  - Continuous reconciliation with alerts on mismatch
  - Automated rollback if error rate >1%

**2. Balance Mismatches**
- **Impact:** High (customer trust, compliance)
- **Probability:** Medium (data transformation bugs)
- **Mitigation:**
  - Reconciliation worker runs every 5 minutes
  - Alert on >$0.01 difference
  - Shadow read phase validates transformation logic
  - Extensive unit tests for all currency conversions

**3. TigerBeetle Downtime**
- **Impact:** Critical (payment processing halted)
- **Probability:** Low (99.99% SLA)
- **Mitigation:**
  - 3-node cluster with multi-AZ deployment
  - MongoDB fallback logic in read operations
  - Circuit breaker pattern for TigerBeetle calls

**4. Transfer Registry Inconsistency**
- **Impact:** Medium (void operations fail)
- **Probability:** Medium (registry write failures)
- **Mitigation:**
  - Registry writes in same transaction as MongoDB (dual-write phase)
  - Background sync jobs to reconcile registry
  - Retry logic for registry operations

### Medium Risks

**5. Performance Degradation**
- **Impact:** Medium (user experience)
- **Probability:** Low (TigerBeetle faster than MongoDB)
- **Mitigation:**
  - Load testing with 10x expected traffic
  - Gradual rollout with performance monitoring
  - Caching layer for frequently accessed balances

**6. Account ID Collisions**
- **Impact:** High (data corruption)
- **Probability:** Very Low (~2^-128 with SHA-256)
- **Mitigation:**
  - Collision detection on account creation
  - Nonce-based rehashing if collision detected
  - Monitoring for duplicate account IDs

---

## Success Metrics

### Performance KPIs
- **p50 Latency:** <1ms for balance queries (vs 5ms MongoDB)
- **p95 Latency:** <2ms for balance queries (vs 10ms MongoDB)
- **p99 Latency:** <5ms for writes (vs 50-100ms MongoDB)
- **Throughput:** Support 10,000 tx/sec peak load (vs current 500 tx/sec)

### Data Integrity KPIs
- **Balance Match Rate:** >99.99% (dual-write phase)
- **Zero Data Loss:** 100% of transactions migrated and verified
- **Audit Trail Completeness:** All historical transactions accessible

### Operational KPIs
- **Uptime:** >99.99% (TigerBeetle cluster)
- **Fallback Rate:** <0.1% (read cutover phase)
- **Reconciliation Errors:** <0.01% (continuous validation)

### Business KPIs
- **Infrastructure Cost Reduction:** -50% (simpler ops, higher density)
- **Developer Productivity:** +30% (simpler data model, fewer bugs)
- **Customer Complaints:** -90% (faster balance updates, no race conditions)

---

## Key Decisions Required

### Before Phase 1
1. **Approve TigerBeetle infrastructure budget** (~$10k/month for 3-node cluster)
2. **Assign dedicated team** (2 backend engineers, 1 platform engineer, 1 QA)
3. **Define rollback criteria** (error thresholds, escalation paths)

### Before Phase 3
4. **Approve gradual rollout schedule** (10% → 50% → 90% → 100%)
5. **Define customer communication plan** (in case of issues)

### Before Phase 4
6. **Approve MongoDB deprecation** (legal/compliance review of data retention)
7. **Archive strategy** (how long to keep MongoDB backups)

---

## Dependencies

### Infrastructure
- TigerBeetle cluster (3 nodes, Azure/AWS)
- Redis/Memory cache for account registry
- MongoDB for account/transfer registries

### External Services
- MassTransit (event publishing)
- Datadog/CloudWatch (metrics, alerts)
- GrowthBook (feature flags)

### Team Skills
- TigerBeetle SDK (.NET client library)
- Double-entry accounting principles
- MongoDB query optimization (for registries)

---

## Stakeholder Communication

### Weekly Updates (During Migration)
- Balance match rate, error rate, performance metrics
- Issues encountered and resolutions
- Rollout progress (% of traffic on TigerBeetle)

### Final Migration Report (Week 16)
- Total transactions migrated
- Performance improvement achieved
- Lessons learned
- Recommendations for future migrations (merchant balances, escrow, etc.)

---

## Next Steps

**Immediate (This Week):**
1. Schedule kickoff meeting with stakeholders
2. Request TigerBeetle infrastructure approval
3. Assign team members

**Week 1-2 (Proof of Concept):**
1. Deploy TigerBeetle sandbox cluster
2. Implement account ID generation
3. Test all 5 wallet operations with synthetic data
4. Benchmark performance vs MongoDB

**Week 3 (Implementation Start):**
1. Implement dual-write repository layer
2. Deploy reconciliation worker
3. Create monitoring dashboards
4. Begin Phase 1 rollout (1% traffic)

---

## Questions & Answers

**Q: Can we migrate incrementally (e.g., only USD wallets first)?**
A: Yes, feature flags can enable TigerBeetle per currency/country. However, complexity increases (maintaining both systems longer). Recommended: Full migration in 4 months.

**Q: What happens to historical transactions in MongoDB?**
A: They remain in MongoDB for audit purposes (Transfer Registry stores metadata). TigerBeetle only stores balances + new transactions. Historical data archived after 1 year.

**Q: Can we query TigerBeetle for transaction history?**
A: No, TigerBeetle doesn't support transaction history queries. Use Transfer Registry (MongoDB) for "list transactions by client" queries. TigerBeetle excels at balance lookups and transfer creation.

**Q: How do we handle currency exchange (multi-currency wallets)?**
A: TigerBeetle enforces same-ledger transfers. For currency exchange, create an "Exchange Account" on both ledgers and perform two transfers (e.g., USD → Exchange USD, Exchange EUR → EUR customer).

**Q: What if TigerBeetle fails during a transfer?**
A: Transfers are atomic and durable once acknowledged. If network fails mid-operation, retry with idempotency key. TigerBeetle guarantees exactly-once semantics.

---

## Resources

**Documentation:**
- [TigerBeetle Official Docs](https://docs.tigerbeetle.com/)
- [TigerBeetle GitHub](https://github.com/tigerbeetle/tigerbeetle)
- [Rafiki Integration Case Study](https://interledger.org/developers/blog/rafiki-tigerbeetle-integration/)
- [Jepsen TigerBeetle Analysis](https://jepsen.io/analyses/tigerbeetle-0.16.11)

**Internal Documents:**
- `/TIGERBEETLE_WALLET_MIGRATION.md` (full technical analysis)
- `/TIGERBEETLE_MIGRATION_DIAGRAMS.md` (all mermaid diagrams)

**Team Contacts:**
- Tech Lead: [TBD]
- Platform Engineering: [TBD]
- Database Admin: [TBD]

---

**Document Version:** 1.0
**Last Updated:** 2025-11-18
**Next Review:** 2025-12-01 (after PoC completion)

**Approval Status:**
- [ ] Tech Lead
- [ ] VP Engineering
- [ ] CTO
- [ ] Platform Engineering Lead
- [ ] Security Team