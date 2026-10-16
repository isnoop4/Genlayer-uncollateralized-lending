# Undercollateralized Lending on GenLayer

A reputation-based lending prototype built as a GenLayer Intelligent Contract. A borrower requests a loan and links a public profile page. GenLayer validators fetch that page, check that it proves the borrower owns it, produce an AI evidence review and a 0–100 score, and reach consensus on the result. A lender can then approve the loan if the score meets a configurable minimum. Repayments and defaults build an on-chain history for the borrower, but only after the lender confirms them.

> **Prototype only.** The contract records loan state and history. It does **not** escrow or transfer native tokens or stablecoins. Integrate a supported escrow/token mechanism before using real funds.

- **Contract (Studionet, v2):** `0xB0CE8d3185AC9e4906e9cb26335E792556F1d30D`
- **Live demo:** https://isnoop4.github.io/Genlayer-uncollateralized-lending/
- **Contract source:** `contracts/undercollateralized_lending.py`

## What changed in v2

This version addresses the steward feedback on the earlier submissions.

### 1. Profile evidence is bound to the borrower

Each loan gets an ownership code: `genlayer-verify:<loan_id>:<borrower_address>`.

- The borrower must publish this exact code on the public profile page (they must be able to edit it).
- During `review_borrower`, validators check whether the code appears on the page. If it does not, the score is forced to 0 and `approve_loan` is rejected.
- Anyone can read the code with `get_ownership_challenge`.

### 2. Repayment needs lender confirmation

- The borrower calls `report_repayment`. This only adds to a **pending** amount and does not count toward history.
- The lender calls `confirm_repayment` (up to the reported amount). Only confirmed amounts count.
- When confirmed repayments reach the loan amount, the loan becomes `REPAID` and the borrower's history is updated.
- Only the loan's lender can confirm. The lender cannot be the borrower.

### 3. Reputation resists simple gaming

- Each (borrower, lender) pair counts only once toward completed loans or defaults.
- Reputation also tracks the number of **distinct lenders**.
- `mark_default` can be called by the loan's lender or the contract owner.

## How it works

```
request_loan ─► (publish code on profile) ─► review_borrower ─► approve_loan
     │                                              │                │
     │                                              │                ├─► report_repayment ─► confirm_repayment ─► REPAID
     │                                              │                └─► mark_default ─► DEFAULTED
     └──────────────────────── cancel_loan (borrower, before approval) ─► CANCELLED
```

| Status | Meaning |
|---|---|
| `REQUESTED` | Borrower submitted amount, term, purpose and profile link |
| `REVIEWED` | Validators produced the binding result and score |
| `ACTIVE` | Lender approved the loan |
| `REPAID` | Lender confirmed repayments covering the full amount |
| `DEFAULTED` | Lender or owner marked a default |
| `CANCELLED` | Borrower cancelled before approval |

## Methods

| Method | Caller | Purpose |
|---|---|---|
| `request_loan` | Borrower | Create a request and get the ownership code |
| `review_borrower` | Anyone | AI review, binding check and score via validators |
| `approve_loan` | Lender (not the borrower) | Approve if bound and score ≥ minimum |
| `cancel_loan` | Borrower | Cancel before approval |
| `report_repayment` | Borrower | Report a repayment (pending only) |
| `confirm_repayment` | Lender | Confirm a reported repayment and update history |
| `mark_default` | Lender or owner | Record a default |
| `set_min_score` | Owner | Change the minimum score (0–100) |
| `get_loan`, `get_ownership_challenge`, `get_borrower_reputation`, `get_protocol_stats` | Anyone | Read state |

## Try it

1. Open the live demo (Studionet test accounts are created in your browser).
2. As the borrower, request a loan, then paste the shown code into your public profile page.
3. Run the review. Without the code the score is 0.
4. Switch to a different account as the lender and approve.
5. As the borrower, report repayment. As the lender, confirm it.

## Known limits

- No real fund movement (see the prototype note above).
- Scores come from AI review of public evidence and are advisory only.
- Lender confirmation is trust-based. One person controlling two accounts can still confirm their own repayment. The one-count-per-pair rule limits the impact, but escrow or asset backing is the real fix.
