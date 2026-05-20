# Pangolin — Testnet End-to-End Testing Guide

Full flow: create escrow → freelancer accepts → freelancer submits delivery → client releases budget → USDC lands in freelancer wallet.

---

## Prerequisites

- Node.js installed
- [Freighter browser extension](https://www.freighter.app/) installed
- `.env.local` configured (copy from `pangolin/.env.example` and fill in values)
- Dev server running:

```powershell
cd pangolin
npm run dev
```

Open `http://localhost:3000`

---

## Part 1 — Generate two wallets

Run both commands from the `pangolin/` directory. Save all four keys somewhere safe.

**Client wallet:**
```powershell
node -e "const {Keypair}=require('@stellar/stellar-sdk'); const kp=Keypair.random(); console.log('CLIENT PUBLIC: ',kp.publicKey()); console.log('CLIENT SECRET: ',kp.secret())"
```

**Freelancer wallet:**
```powershell
node -e "const {Keypair}=require('@stellar/stellar-sdk'); const kp=Keypair.random(); console.log('FREELANCER PUBLIC: ',kp.publicKey()); console.log('FREELANCER SECRET: ',kp.secret())"
```

---

## Part 2 — Fund both wallets

**Fund client** — gets XLM + USDC trustline + ~100 USDC via DEX swap:
```powershell
node scripts/fund-testnet.mjs --secret <CLIENT_SECRET_S...>
```

**Fund freelancer** — gets XLM + USDC trustline (no USDC balance needed, just the trustline so it can receive):
```powershell
node scripts/fund-testnet.mjs --secret <FREELANCER_SECRET_S...>
```

Both scripts print final balances. Client should show ~100 USDC. Freelancer will show 0 USDC — that is expected.

> **Important:** The freelancer wallet MUST run this script before the client releases payment. Without the USDC trustline the transfer will fail with a `trustline entry is missing` error.

---

## Part 3 — Import wallets into Freighter

1. Open Freighter extension → click your account name at the top
2. **Add Account → Import** → paste `CLIENT_SECRET_S...` → name it `Client`
3. **Add Account → Import** → paste `FREELANCER_SECRET_S...` → name it `Freelancer`

You will switch between these two accounts during the test.

---

## Part 4 — Create escrow (Client)

1. Switch Freighter to **Client** account
2. Go to `http://localhost:3000/create-escrow`
3. Fill in the form:
   - **Project title**: anything
   - **Amount**: `90` (USDC)
   - **Freelancer wallet**: paste `FREELANCER_PUBLIC_G...`
   - **Min guarantee**: any value between `10` and `50`
   - **Deadline**: any future date
4. Click through to Step 3 → click **"Fund Escrow Now"**
5. Freighter will prompt **twice** — approve both (create + fund)
6. Wait for the success screen

---

## Part 5 — Verify escrow on dashboard

1. Go to `http://localhost:3000/dashboard`
2. Hard refresh (`Ctrl+Shift+R`)
3. Your new escrow should appear in the table

---

## Part 6 — Accept invite (Freelancer)

1. Switch Freighter to **Freelancer** account
2. Go to `http://localhost:3000/freelancer`
3. You will see the invite screen with the escrow details
4. Click **"Accept & Connect Wallet"**
5. Freighter prompts with the Freelancer account — approve
6. Success screen appears → click **"Continue to Dashboard →"**

---

## Part 7 — Submit delivery (Freelancer)

1. Keep Freighter on **Freelancer** account
2. Go to `http://localhost:3000/delivery`
3. Upload any file or paste any URL in the link field
4. Add a delivery note (optional)
5. Click **"Submit Delivery"**
6. Freighter prompts — approve
7. Wait for the progress bar to finish → "Delivery recorded on-chain ✓"

---

## Part 8 — Release budget (Client)

1. Switch Freighter to **Client** account
2. Go to `http://localhost:3000/escrow`
3. The escrow shows with delivery info
4. Click **"Approve & Release $90.00"**
5. Freighter prompts — approve
6. A green success banner appears with the transaction hash

---

## Part 9 — Verify the transfer

Open Stellar Expert with your freelancer public key:

```
https://stellar.expert/explorer/testnet/account/<FREELANCER_PUBLIC_G...>
```

The freelancer should have **~87.75 USDC** (90 USDC minus 2.5% platform fee).

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Error(Contract, #3)` on create | Invalid input | Min guarantee must be 10–50; deadline must be a future date |
| `Error(Contract, #2)` on accept | Escrow already accepted | Click Accept again — it detects the status and skips to dashboard |
| `trustline entry is missing` on release | Freelancer wallet has no USDC trustline | Run `fund-testnet.mjs --secret <FREELANCER_SECRET>` |
| Wrong wallet signing | Stale Freighter connection | Switch to the correct account in Freighter BEFORE clicking the action button |
| Escrow not in dashboard | Page not refreshed | Hard refresh with `Ctrl+Shift+R` |
| DEX swap failed during funding | No testnet liquidity | Trustline is still created — the 0 USDC balance on freelancer is fine |
