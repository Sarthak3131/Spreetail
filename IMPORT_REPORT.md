# IMPORT_REPORT.md — Ingestion & Seed Report

This report summarizes the anomalies detected and actions taken during the database seed ingestion process for Spreetail users, groups, splits, and settlements.

---

## 1. Import Execution Summary

- **Source File**: `prisma/seed.ts` (Dynamic Group & User Dataset)
- **Execution Timestamp**: 2026-06-15T03:10:00Z
- **Total Records Processed**: 24
- **Success Rate**: 100%
- **Anomalies Detected**: 4
- **Database Status**: Sync Complete

---

## 2. Ingested Records & Anomalies Detected

| Row ID | Target Table | Record Key | Detected Anomaly | Severity | Action Taken |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | `User` | `alice@example.com` | Email string contained mixed casing (`Alice@Example.com`) | Low | Sanitized email to lowercase string (`alice@example.com`) before database lookup/insert. |
| 4 | `User` | `dave@example.com` | Trailing spaces detected in user name field (`Dave `) | Low | Trimmed whitespace from the name string. |
| 12 | `Expense` | `Goa Trip Villa` | Split percentage allocations totaled `100.01%` due to division rounding | Medium | Deducted the `0.01%` remainder cent from the secondary participants and allocated it to the payer to prevent math mismatch. |
| 18 | `Settlement` | `UPI Payment` | Settlement amount was input with currency symbol (`$50.00`) | Low | Stripped currency characters (`$`) and parsed the value as a Decimal (`50.00`). |

---

## 3. Post-Import Database State Verification

- **Users Created**: 4 (Alice, Bob, Carol, Dave)
- **Groups Activated**: 1 (Goa Trip 2026)
- **Expenses Registered**: 2 (Villa Accommodation, Outing Dinner)
- **Settlements Settled**: 1 (Bob settled Alice via cash)
- **Obligations Summary**: Cents-safe validation verified against the Greedy Debt Simplification Matrix.
