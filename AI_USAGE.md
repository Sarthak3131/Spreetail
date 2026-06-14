# AI_USAGE.md — AI Tools & Collaboration Log

This document logs the AI tools utilized during development, key instructions executed, and three concrete cases where the AI made incorrect assertions, how they were caught, and what changes were made to resolve them.

---

## 1. AI Tools & Key Prompts

- **AI Tools Used**: Claude (Anthropic) via [claude.ai](https://claude.ai) & Antigravity (Google DeepMind).
- **Key Prompts Used**:
  - *"audit each and every thing from UI to backend and connection and logics"*
  - *"make UI more premium and unique and production grade"*
  - *"there are still many UI errors and also when adding expense shouldn't there be a option to pay later and pay now and payment method should also be integrated"*
  - *"still shows paid by even though I clicked pay later and logically shouldnt it also show in dashboard in owed column?"*

---

## 2. Concrete Cases of AI Errors & Resolutions

### Case 1: Decimal Types JSON Serialization Crash
- **The Issue**: The AI passed Decimal columns (representing split amounts) directly from the database response via Express to the client. Since JSON does not natively support arbitrary-precision decimals, the Prisma client serialized these as strings or object notations, causing client-side arithmetic operations (like `toFixed()` or additions) to crash with type errors.
- **How It Was Caught**: Manual testing of the expense log showed page load failures and `TypeError: amount.toFixed is not a function` in the browser console.
- **What Was Changed**: Modified the API controllers and frontend components to explicitly cast decimal outputs via `Number(amount)` or format them string-safely on the backend prior to dispatching responses, preventing type crashes.

### Case 2: Aggressive Vite HMR Stylesheet Caching
- **The Issue**: The AI changed the layout structure and CSS class definitions in `index.css`, but the local browser display failed to render them. The AI initially assumed the CSS file compiled incorrectly and kept writing redundant styling overrides.
- **How It Was Caught**: The user reported that "it opens the old shows no change."
- **What Was Changed**: Discovered that Vite aggressively caches local stylesheets. Running `npx vite --force` cleared the Vite dev dependencies pre-optimization cache. Adding instructions to perform a browser hard reload (`Ctrl + F5` or clearing site data) resolved the local cache issues.

### Case 3: Broken WebSocket Paths under Vercel Reverse Proxy
- **The Issue**: When configuring WebSockets (`Socket.io`) on the deployed app, the AI initially instantiated the WebSocket client pointed directly to the root domain (`window.location.origin`).
- **How It Was Caught**: During post-deployment verification on Vercel, real-time message posting failed, and the browser console reported recurrent `404 Not Found` errors on `/socket.io`.
- **What Was Changed**: Under the multi-service deployment setup, the backend runs under the prefix `/_/backend`. The WebSocket handshake had to be directed to `/_/backend/socket.io`. A custom path config resolver was added to the Socket.io client initialization:
  ```ts
  const socket = io(SOCKET_URL, {
    path: '/_/backend/socket.io'
  });
  ```
  This allowed the Vercel reverse proxy to route real-time connections to the backend server.
