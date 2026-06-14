# DECISIONS.md — Architectural Decision Log

This document records the key architectural decisions made during the development of Spreetail, the options considered, and the rationale behind each choice.

---

### Decision 1: In-Memory Dynamic Balances vs. Cached Database Balance Tables
- **Options Considered**:
  - *Option A*: Store pre-calculated net balances in a column on the `User` or `GroupMember` tables. Update these fields using database hooks or transaction triggers whenever an expense or settlement is created.
  - *Option B (Chosen)*: Compute balances dynamically in-memory on request by querying all active group expenses, splits, and settlements.
- **Rationale**:
  - *Option A* introduces a risk of "desyncs" where database transaction failures leave user balance columns in an inconsistent state.
  - *Option B* acts as the single source of truth. As long as the raw logs of expenses and settlements are correct, the compiled net balance is guaranteed to be accurate. The in-memory Greedy matching matches debtors directly to creditors dynamically.

---

### Decision 2: Currency Safe Decimals vs. Standard Binary Floats
- **Options Considered**:
  - *Option A*: Use standard JavaScript/JSON floats (`number` type) for splits and expense records.
  - *Option B (Chosen)*: Map database types to Prisma `Decimal(10, 2)` and parse them carefully.
- **Rationale**:
  - Binary floating-point representation cannot represent fractions like `1/3` accurately (e.g., `$10 / 3` becomes `$3.3333333333333335`). Over thousands of records, these fractional roundings accumulate into significant monetary deficits.
  - Utilizing `Decimal(10, 2)` ensures exact cents arithmetic on database queries. In combination with penny-rounding logic on split creators (assigning remainder cents to the payer), the ledger is mathematically watertight.

---

### Decision 3: HttpOnly Cookie Rotation vs. localStorage Session Caching
- **Options Considered**:
  - *Option A*: Store both Access and Refresh JWT tokens in `localStorage` or `sessionStorage` in the browser, passing them in the authorization headers.
  - *Option B (Chosen)*: Keep Access JWT in-memory (Zustand state) and place the Refresh JWT in an encrypted, `httpOnly` secure cookie. Rotate the Refresh JWT on every refresh request.
- **Rationale**:
  - *Option A* leaves the application vulnerable to Cross-Site Scripting (XSS) attacks. If a malicious script runs, it can read `localStorage` and steal the user session.
  - *Option B* blocks XSS access to the refresh token because JavaScript cannot read `httpOnly` cookies. Token rotation blocks replay attacks if a refresh token is somehow intercepted.

---

### Decision 4: Separated Monorepo vs. Hoisted npm Workspaces
- **Options Considered**:
  - *Option A*: Create a hoisted monorepo workspace package structure utilizing npm/yarn/pnpm workspaces in the root folder.
  - *Option B (Chosen)*: Keep independent, nested folders for `/frontend` and `/backend` with their own independent `package.json` configurations.
- **Rationale**:
  - Hoisted package managers sometimes create conflicts with native binary builds (like `bcrypt`) or deploy targets (such as Vercel and Railway) trying to read parent directories.
  - Option B provides simple, isolated environments. Vercel builds the `/frontend` sub-directory directly, and Railway deploys `/backend` in isolation, preventing hoisting path issues.

---

### Decision 5: Formless UI State vs. Standard HTML `<form>` Tags
- **Options Considered**:
  - *Option A*: Wrap all input fields in native `<form>` tags and trigger submits.
  - *Option B (Chosen)*: Use custom input triggers, React click handlers, and Zustand global store states.
- **Rationale**:
  - Form submissions often trigger page refreshes, which clear in-memory JWTs and state.
  - Formless UI controls provide a sleek SPA experience. Input values are validated on key/click events, leading to a much smoother user experience.
