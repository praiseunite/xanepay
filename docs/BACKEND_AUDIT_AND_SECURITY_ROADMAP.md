# XanePay Backend Security Audit & Technical Roadmap
## Comprehensive Analysis of Remaining Work, Vulnerabilities, and Hardening Measures

> **Date:** September 2026  
> **Target Services:** `XaneApp/` (Main App), `XaneApp/engine/` (Treasury Engine), `XaneApp/backend/` (Conversion Backend)  
> **Status:** Actionable Security Audit & Technical Roadmap

---

## Executive Summary

A deep static code analysis across all backend services revealed strong foundational architectural patterns in the **Treasury Engine** (immutable double-entry ledger, hash chains, Redis locks, BullMQ rate caps), but identified several **critical security vulnerabilities, architectural gaps, and unfinished features** in the **Main Application (`XaneApp/`)** that must be resolved prior to handling real user capital.

---

## 1. Critical Security Findings & Immediate Remediation

### 🔴 Finding 1: Service Account Private Keys Committed in Git
- **Location:** [`XaneApp/src/config/serviceAccountKey.json`](file:///c:/Projects/XanePay/XaneApp/src/config/serviceAccountKey.json)
- **Vulnerability:** A full Google Cloud Firebase service account private key (`firebase-adminsdk-fbsvc@xaneapp-backend-prod.iam.gserviceaccount.com`) is committed into git history.
- **Risk:** Anyone with repository access can authenticate as the GCP service account and compromise Firebase resources.
- **Remediation Action:**
  1. Revoke the key in the Google Cloud IAM console immediately.
  2. Add `serviceAccountKey.json` to `.gitignore`.
  3. Load the credential in production dynamically from AWS Secrets Manager or the environment variable `FIREBASE_SERVICE_ACCOUNT_JSON`.

---

### 🔴 Finding 2: Insecure Direct Object Reference (IDOR) & Arbitrary Balance Overwrites
- **Location:** [`XaneApp/src/routes/walletRoutes.ts`](file:///c:/Projects/XanePay/XaneApp/src/routes/walletRoutes.ts#L68-L85)
- **Vulnerability:** 
  ```typescript
  router.put("/:walletId", authMiddleware, async (req: Request, res: Response) => {
    const { walletId } = req.params;
    const { newBalance } = req.body;
    const wallet = await updateWalletBalance(walletId, newBalance);
    res.json({ success: true, wallet });
  });
  ```
  Any authenticated user can send `PUT /wallet/<any-wallet-id>` with `{ "newBalance": 1000000 }` to set their balance (or any other user's balance) to any arbitrary number.
- **Risk:** Complete economic exploit.
- **Remediation Action:**
  1. **Delete the direct `PUT /wallet/:walletId` endpoint immediately.**
  2. Balances must NEVER be set directly by user-facing endpoints. Balances must be modified exclusively through ledger entry transactions triggered by deposits, completed conversions, or payouts.
  3. Enforce that queries verify ownership: `WHERE wallet_id = :id AND user_id = :authenticatedUserId`.

---

### 🔴 Finding 3: Hardcoded Fallback JWT Secret
- **Location:** 
  - [`XaneApp/src/services/userService.ts`](file:///c:/Projects/XanePay/XaneApp/src/services/userService.ts#L5)
  - [`XaneApp/src/services/jwtService.ts`](file:///c:/Projects/XanePay/XaneApp/src/services/jwtService.ts#L4)
  - [`XaneApp/src/middleware/auth.ts`](file:///c:/Projects/XanePay/XaneApp/src/middleware/auth.ts)
- **Vulnerability:**
  `const JWT_SECRET = process.env.JWT_SECRET || "fallbackSecretKey123";`
- **Risk:** If `JWT_SECRET` is missing in production, attackers can forge valid authentication tokens using `fallbackSecretKey123` and impersonate any user.
- **Remediation Action:**
  - Enforce fail-closed validation on startup: if `!process.env.JWT_SECRET || process.env.JWT_SECRET.length < 32`, throw an exception and refuse to boot.

---

### 🟡 Finding 4: In-Memory Passkey Challenge Storage
- **Location:** [`XaneApp/src/services/passkeyService.ts`](file:///c:/Projects/XanePay/XaneApp/src/services/passkeyService.ts#L6)
- **Vulnerability:**
  `const challengeStore: Map<string, string> = new Map();`
- **Risk:** When deploying to AWS ECS Fargate with multiple tasks, a user beginning passkey registration on Task A will fail verification if `/finish` hits Task B (challenge not found).
- **Remediation Action:**
  - Store WebAuthn challenges in Redis with a 5-minute TTL: `SETEX passkey:challenge:<userId> 300 <challenge>`.

---

### 🟡 Finding 5: Missing Rate Limiting & Input Validation on Auth Routes
- **Location:** [`XaneApp/src/routes/userRoutes.ts`](file:///c:/Projects/XanePay/XaneApp/src/routes/userRoutes.ts)
- **Vulnerability:** `/user/register` and `/user/login` have no rate limiting and lack Zod schema validation (e.g. phone number E.164 formatting, password minimum entropy).
- **Risk:** Vulnerable to credential stuffing, brute-force password guessing, and malformed database records.
- **Remediation Action:**
  - Attach `express-rate-limit` (5 requests/minute per IP) on `/user/login` and `/user/register`.
  - Validate payloads using strict Zod schemas before processing.

---

### 🟡 Finding 6: Blockchain Private Key Custody in Environment
- **Location:** [`XaneApp/src/services/transactionService.ts`](file:///c:/Projects/XanePay/XaneApp/src/services/transactionService.ts#L49)
- **Vulnerability:** Smart contract transactions sign payloads using raw private keys read directly from `process.env[CHAIN_PRIVATE_KEY]`.
- **Risk:** Leaked container logs or memory dumps could expose the operational relayer wallets.
- **Remediation Action:**
  - Use AWS KMS or Web3Auth MPC for transaction signing so private keys never exist in plaintext memory.

---

## 2. Functional & Architectural Gaps (What is Yet to be Done)

| Feature Area | Current State | Target Production State |
|---|---|---|
| **XaneApp ↔ Engine Bridge** | Separate microservices | XaneApp client calls Engine via HMAC-signed internal client (`POST /api/v1/conversions/quote` & `execute`) |
| **MPC Wallet Implementation** | Stubbed in [`src/services/mpcService.ts`](file:///c:/Projects/XanePay/XaneApp/src/services/mpcService.ts) | Full Web3Auth CoreKit integration with 2-of-3 threshold key shares |
| **Passkey Challenge Store** | Local in-memory `Map` | Distributed Redis store with TTL |
| **Fincra Bank Rail Webhooks** | Signature validator middleware built | End-to-end automated reconciliation of inbound bank deposit webhooks |
| **Double-Entry Wallet Ledger** | Direct balance field on `wallets` table | All balance updates backed by `ledger_entries` audit records |
| **Cross-Chain Bridge Fallback** | Dynamic fee calculator estimates routes | Automated execution fallback from Hashport to Wormhole if primary rail times out |

---

## 3. Security Hardening Checklist (Pre-Production Gate)

### Authentication & Secrets
- [ ] Rotate and remove `serviceAccountKey.json` from git history using `git-filter-repo` or BFG.
- [ ] Remove all fallback secrets (`"fallbackSecretKey123"`). Fail-closed on startup.
- [ ] Enforce JWT expiration (maximum 1 hour for access tokens; rotating refresh tokens).
- [ ] Transition WebAuthn challenge store from in-memory `Map` to Redis.

### API & Network Security
- [ ] Remove `PUT /wallet/:walletId` arbitrary balance modification endpoint.
- [ ] Attach rate limiters to `/user/login`, `/user/register`, `/otp/verify`, and `/wallet`.
- [ ] Configure strict CORS whitelist in production (reject `*`).
- [ ] Ensure `helmet` security headers are active on all Express apps.
- [ ] Verify that error responses in `NODE_ENV=production` never return stack traces or raw SQL errors.

### Data & Ledger Integrity
- [ ] Ensure all database connections enforce `sslmode=verify-full` in staging/production.
- [ ] Apply database triggers on financial ledger tables (`BEFORE UPDATE OR DELETE RAISE EXCEPTION`).
- [ ] Run nightly ledger balance reconciliation jobs with automated PagerDuty/Slack alarms.

---

## 4. Prioritized Implementation Roadmap

### Phase 1: Security Fixes (Immediate)
1. Delete dangerous `PUT /wallet/:walletId` balance endpoint.
2. Remove fallback secrets in `userService.ts`, `jwtService.ts`, `auth.ts`.
3. Add Zod validation + rate limiters to `userRoutes.ts`.
4. Switch `passkeyService.ts` challenge storage to Redis.

### Phase 2: Engine & App Integration
1. Build internal HMAC client in XaneApp to invoke Treasury Engine conversion endpoints (`/api/v1/conversions/quote` and `/execute`).
2. Map completed engine conversions to user wallet balance updates.

### Phase 3: Production Hardening on AWS
1. Deploy containers to AWS ECS Fargate using the configurations in [`docs/AWS_DEPLOYMENT_AND_SECURITY_GUIDE.md`](file:///c:/Projects/XanePay/docs/AWS_DEPLOYMENT_AND_SECURITY_GUIDE.md).
2. Store all keys (Fincra, OnSwitch, Database, JWT) in AWS Secrets Manager.
3. Configure AWS WAF on Application Load Balancers.
