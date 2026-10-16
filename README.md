# Undercollateralized Lending on GenLayer

A reputation-based lending prototype built as a GenLayer Intelligent Contract. A borrower requests a loan and submits profile evidence. GenLayer validators fetch the evidence, produce an AI review and a 0–100 score, and reach consensus on the result. A lender can approve the loan if the score meets a configurable minimum. Repayments and defaults build an on-chain history for the borrower, but only after the lender confirms them.

> **Prototype only.** The contract records loan state and history. It does **not** escrow or transfer native tokens or stablecoins. Integrate a supported escrow/token mechanism before using real funds.

- **Contract (Studionet, v2):** `0xB0CE8d3185AC9e4906e9cb26335E792556F1d30D`
- **Live demo:** https://isnoop4.github.io/Genlayer-uncollateralized-lending/
- **Contract source:** `contracts/undercollateralized_lending.py`

## What changed in v2

This version addresses the steward feedback on the previous submission.

### 1. Evidence is bound to the borrower

Previously, profile evidence was accepted without proving the borrower owns it. Now:

- Evidence is tied to the borrower's address when the loan is requested.
- Anyone can dispute evidence with `challenge_evidence`. Validators re-check the claim.
- Evidence that fails verification or loses a challenge does not count toward the score, and the loan cannot be approved on it.

### 2. Repayment needs lender confirmation

Previously, a borrower-only `record_repayment` call could create completed-loan history. Now:

1. The borrower calls `report_repayment`. The loan moves to `REPAYMENT_REPORTED`.
2. The lender calls `confirm_repayment`. Only then is the loan marked `REPAID` and written into the borrower's history.
3. If the lender does not confirm, the report stays pending and no history is created.
4. `mark_default` is lender-only and records a default in the borrower's history.

A borrower can no longer create positive history on their own.

## How it works

```
request_loan ──► review_borrower ──► approve_loan ──► report_repayment ──► confirm_repayment
     │                  │                 │                                      │
     │                  │                 └──► mark_default                      ▼
     │                  │                                                    REPAID
     │                  └──► challenge_evidence (re-review)
     └──────────────────────────────► cancel_loan (borrower, before approval)
```

| Status | Meaning |
|---|---|
| `REQUESTED` | Borrower submitted amount, term and evidence |
| `REVIEWED` | Validators produced a score and review |
| `APPROVED` | Lender approved (score met the minimum) |
| `REPAYMENT_REPORTED` | Borrower reported repayment, waiting for lender |
| `REPAID` | Lender confirmed, history updated |
| `DEFAULTED` | Lender marked default, history updated |
| `CANCELLED` | Borrower cancelled before approval |

## Main methods

| Method | Caller | Purpose |
|---|---|---|
| `request_loan` | Borrower | Create a loan request with evidence |
| `review_borrower` | Anyone | Run the AI review and scoring through validators |
| `challenge_evidence` | Anyone | Dispute the evidence and trigger a re-check |
| `approve_loan` | Lender | Approve if the score meets the minimum |
| `report_repayment` | Borrower | Report that the loan was repaid |
| `confirm_repayment` | Lender | Confirm repayment and write history |
| `mark_default` | Lender | Record a default |
| `cancel_loan` | Borrower | Cancel before approval |

## Run it

1. Open [GenLayer Studio](https://studio.genlayer.com).
2. Paste `contracts/undercollateralized_lending.py` and deploy.
3. Call the methods above in order, or use the live demo.

## Known limits

- No real fund movement (see the prototype note above).
- Scores come from AI review of public evidence, so they are advisory.
- Lender confirmation is trust-based until escrow or asset backing is added.
