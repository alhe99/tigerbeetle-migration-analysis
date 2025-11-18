# TigerBeetle Wallet Migration - Documentation Index

This directory contains comprehensive documentation for migrating the Payments Backend wallet system from MongoDB to TigerBeetle.

---

## Document Overview

### 1. Executive Summary (Start Here)
**File:** `TIGERBEETLE_MIGRATION_SUMMARY.md`

**Purpose:** High-level overview for stakeholders and decision-makers

**Contents:**
- Quick facts and performance comparison
- Why TigerBeetle? (pain points + advantages)
- 4-month migration timeline
- Risk assessment and mitigation strategies
- Success metrics and KPIs
- Key decisions required
- Next steps

**Audience:** VP Engineering, CTO, Tech Leads, Product Managers

**Reading Time:** 15 minutes

---

### 2. Full Migration Analysis (Technical Deep Dive)
**File:** `TIGERBEETLE_WALLET_MIGRATION.md`

**Purpose:** Comprehensive technical analysis and implementation guide

**Contents:**
- Current MongoDB wallet architecture (detailed entity analysis)
- TigerBeetle core concepts (accounts, transfers, ledgers)
- Complete data model mapping strategy
- All 5 wallet operations analyzed:
  1. GetBalance
  2. GetMultiBalance
  3. AddCredits
  4. SubtractCredits (Debit)
  5. VoidTransaction
- Migration strategy (4 phases with detailed steps)
- Technical implementation (account ID generation, ledger assignment, registries)
- Performance benchmarks
- Risk assessment
- Recommendations

**Audience:** Backend Engineers, Platform Engineers, Database Architects

**Reading Time:** 60-90 minutes

**Size:** ~65 KB, 1,800+ lines

---

### 3. Visual Diagrams Reference (Quick Reference)
**File:** `TIGERBEETLE_MIGRATION_DIAGRAMS.md`

**Purpose:** All mermaid diagrams extracted for easy viewing and presentations

**Contents:**
- Operation 1: GetBalance (current vs future flows)
- Operation 2: GetMultiBalance (current vs future flows)
- Operation 3: AddCredits (current vs future flows)
- Operation 4: SubtractCredits (current vs future flows)
- Operation 5: VoidTransaction (current vs future flows)
- Migration phase diagrams (dual-write, shadow read, cutover)
- Double-entry accounting flows
- Account structure diagrams
- Data mapping diagrams

**Audience:** All technical stakeholders, useful for presentations

**Reading Time:** 20-30 minutes (diagram review)

**Size:** ~20 KB, 800+ lines

---

## Quick Start Guide

### For Decision Makers
1. Read `TIGERBEETLE_MIGRATION_SUMMARY.md` (15 min)
2. Review "Why TigerBeetle?" and "Risk Assessment" sections
3. Check "Key Decisions Required" for approval items
4. Review success metrics and timeline

### For Technical Team
1. Skim `TIGERBEETLE_MIGRATION_SUMMARY.md` for context (10 min)
2. Deep dive into `TIGERBEETLE_WALLET_MIGRATION.md` (60-90 min)
3. Study diagrams in `TIGERBEETLE_MIGRATION_DIAGRAMS.md` (20 min)
4. Review specific operation flows relevant to your work
5. Reference "Technical Implementation" section for coding

### For Presentations
1. Use diagrams from `TIGERBEETLE_MIGRATION_DIAGRAMS.md`
2. Extract key metrics from `TIGERBEETLE_MIGRATION_SUMMARY.md`
3. Show "Current vs Future" flow comparisons for operations
4. Highlight performance improvements and risk mitigation

---

## Key Concepts Quick Reference

### MongoDB Current State
- **Entity:** PaymentsWallet
- **Composite Key:** ClientId + Country + Currency + IssuerTypeIdentifier
- **Balance:** Credit (positive) - Debit (negative)
- **Transactions:** PaymentsWalletTransaction (separate collection)

### TigerBeetle Target State
- **Primitives:** Accounts (balances) + Transfers (movements)
- **Account ID:** UInt128 hash of composite key
- **Balance:** credits_posted - debits_posted
- **Double-Entry:** Every transfer has debit + credit account
- **Registries:** Account Registry (for ClientId lookups) + Transfer Registry (for void state)

### Performance Improvements
- **Latency:** 5-10x faster (5-10ms → <1ms)
- **Throughput:** 200x higher (5K tx/sec → 1M tx/sec)
- **Cost:** -50% infrastructure (simpler ops)

### Migration Timeline
- **Phase 1 (Weeks 1-4):** Dual-write (MongoDB + TigerBeetle)
- **Phase 2 (Weeks 5-8):** Shadow read testing
- **Phase 3 (Weeks 9-12):** Gradual read cutover (10% → 100%)
- **Phase 4 (Weeks 13-16):** Full migration + MongoDB deprecation

---

## Document Structure Comparison

| Section | Summary (15 min) | Full Analysis (60-90 min) | Diagrams (20 min) |
|---------|------------------|---------------------------|-------------------|
| **Overview** | ✅ Executive summary | ✅ Detailed architecture | ✅ Visual flows |
| **Current System** | ⚡ Brief description | ✅ Complete entity analysis | ✅ MongoDB diagrams |
| **TigerBeetle Concepts** | ⚡ Key features | ✅ Deep dive into primitives | ✅ Account structure |
| **Data Mapping** | ⚡ High-level | ✅ Detailed transformation | ✅ Mapping diagrams |
| **Operation Flows** | ❌ Not covered | ✅ All 5 operations detailed | ✅ Sequence diagrams |
| **Migration Strategy** | ✅ 4-phase timeline | ✅ Step-by-step implementation | ✅ Phase architecture |
| **Implementation** | ⚡ Key decisions | ✅ Code examples + patterns | ❌ Not covered |
| **Risks** | ✅ Summary table | ✅ Detailed mitigation | ❌ Not covered |
| **Performance** | ✅ Key metrics | ✅ Benchmark tables | ❌ Not covered |

**Legend:**
- ✅ Comprehensive coverage
- ⚡ Brief/high-level
- ❌ Not included

---

## Migration Phase Checklist

### Phase 1: Dual-Write (Weeks 1-4)
- [ ] TigerBeetle cluster deployed (3 nodes, multi-AZ)
- [ ] Account ID generator implemented
- [ ] Account Registry implemented (MongoDB)
- [ ] Transfer Registry implemented (MongoDB)
- [ ] Dual-write repository layer completed
- [ ] Reconciliation worker deployed
- [ ] Monitoring dashboards created (Datadog/CloudWatch)
- [ ] Feature flag configured (GrowthBook)
- [ ] Rollout: 1% → 10% → 50% → 100% traffic
- [ ] 7 days stable with >99.99% match rate

### Phase 2: Shadow Read (Weeks 5-8)
- [ ] Shadow read logic implemented
- [ ] Metrics collection deployed
- [ ] Data transformation validated
- [ ] Performance benchmarks completed
- [ ] 7 days with >99.99% match rate
- [ ] TigerBeetle latency <50% of MongoDB

### Phase 3: Gradual Cutover (Weeks 9-12)
- [ ] Feature flag rollout: 10% traffic (Week 9)
- [ ] Feature flag rollout: 50% traffic (Week 10)
- [ ] Feature flag rollout: 90% traffic (Week 11)
- [ ] Feature flag rollout: 100% traffic (Week 12)
- [ ] Fallback rate <0.1% for 7 days
- [ ] p95 latency <2ms for balance queries

### Phase 4: Full Migration (Weeks 13-16)
- [ ] Stop dual-write to MongoDB (Week 13)
- [ ] MongoDB in read-only mode (Week 14)
- [ ] Archive MongoDB wallet collections (Week 15)
- [ ] Remove legacy code (Week 16)
- [ ] Update documentation and runbooks
- [ ] Final reconciliation report (100% match)
- [ ] Post-migration retrospective

---

## Common Questions

### What is TigerBeetle?
A financial transactions database built specifically for double-entry accounting, offering:
- Strict serializability (ACID guarantees)
- 1M transactions/second throughput
- Sub-millisecond latency
- Immutable audit trail
- Viewstamped Replication consensus

### Why migrate from MongoDB?
- **10x faster:** <1ms balance queries vs 5-10ms MongoDB
- **200x throughput:** 1M tx/sec vs 5K tx/sec
- **Built-in double-entry:** Impossible to create/destroy money
- **Simpler infrastructure:** No sharding, no replication lag
- **Financial-grade consistency:** Jepsen-verified correctness

### What are the main challenges?
1. **No secondary indexes:** Need Account Registry for ClientId lookups
2. **Immutable transfers:** Need Transfer Registry to track void state
3. **Integer-only amounts:** Must store in smallest currency unit (cents)
4. **Data transformation:** Credit/Debit → credits_posted/debits_posted

### How long will migration take?
- **Total:** 4 months (3 months migration + 1 month stabilization)
- **Dual-write:** 4 weeks
- **Shadow read:** 4 weeks
- **Gradual cutover:** 4 weeks
- **Cleanup:** 4 weeks

### What if something goes wrong?
Each phase has instant rollback:
- **Phase 1-2:** Disable feature flag (MongoDB still authoritative)
- **Phase 3:** Revert to MongoDB reads (fallback logic built-in)
- **Phase 4:** Re-enable MongoDB writes + restore from archive (1-4 hours)

### Will customers notice the migration?
No, if executed correctly:
- Dual-write ensures data consistency
- Gradual rollout minimizes risk
- Fallback logic prevents service disruption
- Performance should improve (faster balance queries)

### What about historical transactions?
- Remain in MongoDB (Transfer Registry stores metadata)
- TigerBeetle only stores balances + new transactions
- Archive after 1 year for compliance

### Can we query TigerBeetle for transaction history?
No, TigerBeetle excels at:
- Balance lookups (sub-millisecond)
- Transfer creation (atomic, durable)

Use Transfer Registry (MongoDB) for:
- Transaction history queries
- Filtering by date/reference ID
- Void state tracking

---

## Related Resources

### External Documentation
- [TigerBeetle Official Docs](https://docs.tigerbeetle.com/)
- [TigerBeetle GitHub Repository](https://github.com/tigerbeetle/tigerbeetle)
- [Double-Entry Accounting Concepts](https://docs.tigerbeetle.com/concepts/debit-credit/)
- [TigerBeetle C# Client](https://docs.tigerbeetle.com/clients/dotnet/)
- [Rafiki TigerBeetle Integration Case Study](https://interledger.org/developers/blog/rafiki-tigerbeetle-integration/)
- [Jepsen TigerBeetle Analysis (Correctness Proof)](https://jepsen.io/analyses/tigerbeetle-0.16.11)

### Internal Codebase
- **Current Wallet Entity:** `/src/Domain/Entities/Mongo/Wallet/PaymentsWallet.cs`
- **Wallet Controller:** `/src/WebUI/Controllers/WalletsController.cs`
- **Add Credits Handler:** `/src/Application/Wallets/Commands/AddCredits/AddCreditCommand.cs`
- **Remove Credits Handler:** `/src/Application/Wallets/Commands/RemoveCredits/RemoveCreditsCommand.cs`
- **Void Handler:** `/src/Application/Wallets/Commands/Void/VoidTrasactionCommand.cs`
- **Get Balance Handler:** `/src/Application/Wallets/Queries/GetBalance/GetWalletBalanceQuery.cs`
- **Get Multi-Balance Handler:** `/src/Application/Wallets/Queries/GetMultiBalance/GetMultiBalanceQuery.cs`

### Testing Documentation
- **Wallet Integration Tests:** `/tests/Application.IntegrationTests/Wallets/`
- **Test Data Helpers:** `/tests/Application.IntegrationTests/Helpers/DataCreationHelper.cs`
- **Testing Infrastructure:** `/tests/Application.IntegrationTests/Testing.cs`

---

## Contributing to Documentation

If you find errors or have suggestions:

1. **Technical corrections:** Update the relevant section in the appropriate document
2. **Clarifications:** Add to "Common Questions" in this README
3. **New diagrams:** Add to `TIGERBEETLE_MIGRATION_DIAGRAMS.md`
4. **Implementation notes:** Add to `TIGERBEETLE_WALLET_MIGRATION.md` under "Technical Implementation"

**Document Owners:**
- Tech Lead: [TBD]
- API Architect: [Auto-generated]
- Platform Engineering: [TBD]

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-18 | API Architect Agent | Initial migration analysis and documentation |

---

## Contact & Support

**For questions about this migration:**
- Slack: #payments-backend-migration
- Email: payments-team@company.com
- Weekly standup: Thursdays 10:00 AM (during migration)

**For TigerBeetle technical support:**
- TigerBeetle Slack: [Join here](https://join.slack.com/t/tigerbeetle/shared_invite/...)
- GitHub Issues: [tigerbeetle/tigerbeetle](https://github.com/tigerbeetle/tigerbeetle/issues)

---

**Last Updated:** 2025-11-18
**Next Review:** 2025-12-01 (after Proof of Concept)