# SCOPE.md — Anomaly & Database Schema Log

This document serves as the data anomaly log for Spreetail data ingestion and details the full relational database schema.

---

## 1. Data Ingestion Anomaly Log

Spreetail utilizes an ingestion layer to seed database records. In a production scenario where member lists, expenses, and transaction logs are imported via CSV files, the following anomalies are validated, filtered, and handled:

| Ingested Field | Detected Anomaly | Severity | Resolution Strategy |
| :--- | :--- | :--- | :--- |
| `User.email` | Uppercase characters & trailing whitespace | Low | Normalized by lowercasing and trimming the string (e.g., ` Alice@Example.com ` → `alice@example.com`). |
| `User.email` | Duplicate email address registration | High | Skipped record insertion and logged as conflict to prevent authentication credential poisoning. |
| `Expense.amount` | Negative values (e.g. `-$50.00`) | High | Rejected record. All expense ledger items must be positive numbers; negative entries must be represented as settlements. |
| `Expense.amount` | Floating-point precision error (e.g. `$100.9999`) | Medium | Truncated to 2 decimal places (`$100.99`) to match standard currency constraints. |
| `ExpenseParticipant` | Total split sum does not match expense amount | High | Automatically allocated the cent remainder (e.g., `$10.00` divided 3 ways leaves `$0.01` remainder) to the payer of the bill to ensure ledger completeness. |
| `ExpenseParticipant` | Participant split percentage does not equal exactly 100% | High | Rejected the import and logged split validation error. |
| `GroupMember.role` | Missing or invalid roles | Low | Defaulted membership roles to `MEMBER` unless explicitly flagged as `ADMIN`. |

---

## 2. Database Schema (Full Prisma Schema)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id                  String               @id @default(uuid())
  name                String
  email               String               @unique
  passwordHash        String
  createdAt           DateTime             @default(now())
  updatedAt           DateTime             @updatedAt
  groupMembers        GroupMember[]
  paidExpenses        Expense[]            @relation("PaidBy")
  createdExpenses     Expense[]            @relation("CreatedBy")
  expenseParticipants ExpenseParticipant[]
  sentSettlements     Settlement[]         @relation("Payer")
  receivedSettlements Settlement[]         @relation("Payee")
  chatMessages        ChatMessage[]
  refreshTokens       RefreshToken[]
}

model Group {
  id          String       @id @default(uuid())
  name        String
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
  members     GroupMember[]
  expenses    Expense[]
  settlements Settlement[]
}

model GroupMember {
  id        String    @id @default(uuid())
  groupId   String
  userId    String
  role      GroupRole @default(MEMBER)
  isActive  Boolean   @default(true)
  joinedAt  DateTime  @default(now())
  group     Group     @relation(fields: [groupId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([groupId, userId])
}

model Expense {
  id           String               @id @default(uuid())
  groupId      String
  description  String
  amount       Decimal              @db.Decimal(10, 2)
  date         DateTime
  paidById     String
  createdById  String
  splitMethod  SplitMethod
  createdAt    DateTime             @default(now())
  updatedAt    DateTime             @updatedAt
  deletedAt    DateTime?
  group        Group                @relation(fields: [groupId], references: [id], onDelete: Cascade)
  paidBy       User                 @relation("PaidBy", fields: [paidById], references: [id])
  createdBy    User                 @relation("CreatedBy", fields: [createdById], references: [id])
  participants ExpenseParticipant[]
  messages     ChatMessage[]
}

model ExpenseParticipant {
  id          String   @id @default(uuid())
  expenseId   String
  userId      String
  amountOwed  Decimal  @db.Decimal(10, 2)
  shareValue  Float?
  createdAt   DateTime @default(now())
  expense     Expense  @relation(fields: [expenseId], references: [id], onDelete: Cascade)
  user        User     @relation(fields: [userId], references: [id])
  @@unique([expenseId, userId])
}

model Settlement {
  id            String   @id @default(uuid())
  groupId       String
  payerId       String
  payeeId       String
  amount        Decimal  @db.Decimal(10, 2)
  note          String?
  paymentMethod String   @default("CASH")
  createdAt     DateTime @default(now())
  group         Group    @relation(fields: [groupId], references: [id], onDelete: Cascade)
  payer         User     @relation("Payer", fields: [payerId], references: [id])
  payee         User     @relation("Payee", fields: [payeeId], references: [id])
}

model ChatMessage {
  id        String    @id @default(uuid())
  expenseId String
  senderId  String
  content   String?
  createdAt DateTime  @default(now())
  deletedAt DateTime?
  expense   Expense   @relation(fields: [expenseId], references: [id], onDelete: Cascade)
  sender    User      @relation(fields: [senderId], references: [id])
}

model RefreshToken {
  id        String   @id @default(uuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

enum GroupRole {
  ADMIN
  MEMBER
}

enum SplitMethod {
  EQUAL
  UNEQUAL
  PERCENTAGE
  SHARE
}
```
