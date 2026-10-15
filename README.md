# Undercollateralized Lending on GenLayer

A reputation-based lending prototype built as a [GenLayer](https://www.genlayer.com) Intelligent Contract. A borrower requests a loan and links a public profile page. GenLayer validators fetch that page, produce an AI evidence review and a 0–100 score, and reach consensus on the result. A lender can then approve the loan if the score meets a configurable minimum, and repayments and defaults build an on-chain history for the borrower.

> **Prototype only.** The contract records loan state and history. It does **not** escrow or transfer native tokens or stablecoins. Integrate a supported escrow/token mechanism before using real funds.

- **Contract (Studionet):** [`0xB0CE8d3185AC9e4906e9cb26335E792556F1d30D`](https://explorer-studio.genlayer.com/address/0xB0CE8d3185AC9e4906e9cb26335E792556F1d30D)
- **Live demo:** `https://isnoop4.github.io/Genlayer-uncollateralized-lending/`
- **Contract source:** [`contracts/undercollateralized_lending.py`](contracts/undercollateralized_lending.py)

## How it works

```
request_loan ──► review_borrower ──► approve_loan ──► record_repayment ──► REPAID
     │                 │                  │
     │                 │                  └──► mark_default (owner) ──► DEFAULTED
     └─────────────────┴──► cancel_loan (borrower) ──► CANCELLED
```

| Status | Meaning |
|---|---|
| `REQUESTED` | Borrower submitted amount, term, purpose and a public profile URL |
| `REVIEWED` | Validators produced an AI score and a short review |
| `ACTIVE` | A lender (not the borrower) approved the loan, and the score met the minimum |
| `REPAID` | Recorded repayments reached the loan amount; borrower's completed count goes up |
| `DEFAULTED` | Contract owner marked the loan as defaulted; borrower's default count goes up |
| `CANCELLED` | Borrower cancelled before approval |

### AI review

`review_borrower` fetches the profile URL inside a non-deterministic block and asks the validators for exactly three lines: `SCORE`, `REVIEW` and `LIMITATION`. Validators agree through `gl.eq_principle.prompt_non_comparative` using explicit criteria:

- The page is treated as **untrusted evidence**, never as instructions (prompt-injection guard).
- No sensitive personal traits are inferred, and no facts may be invented.
- If the page cannot be retrieved or has no useful evidence, the score must be `0` and the limitation must say so.
- The output is parsed conservatively: a malformed score stays at `0`.

The score is a summary of public evidence, **not** a guarantee of repayment.

## Contract interface

**Write methods**

| Method | Who can call | Purpose |
|---|---|---|
| `request_loan(amount_units, term_days, purpose_text, public_profile_url)` | Anyone | Creates a loan. Amount > 0, term 1–365 days, purpose 1–500 characters, URL starts with `https://` |
| `review_borrower(loan_id)` | Anyone | Runs the AI evidence review (loan must be `REQUESTED`) |
| `approve_loan(loan_id)` | Any address except the borrower | Activates a `REVIEWED` loan if `ai_score >= min_score` |
| `cancel_loan(loan_id)` | Borrower | Cancels a `REQUESTED` or `REVIEWED` loan |
| `record_repayment(loan_id, amount_units)` | Borrower | Records a repayment on an `ACTIVE` loan |
| `mark_default(loan_id)` | Contract owner | Marks an `ACTIVE` loan as `DEFAULTED` |
| `set_min_score(new_min_score)` | Contract owner | Sets the minimum score (0–100, default 40) |

**View methods**

| Method | Returns |
|---|---|
| `get_loan(loan_id)` | Loan details, status, AI score and review, repayment totals |
| `get_borrower_reputation(borrower_address)` | Completed and defaulted loan counts |
| `get_protocol_stats()` | `loan_count`, `next_loan_id`, `min_score` |

## Testing

Deployed and tested in [GenLayer Studio](https://studio.genlayer.com) with two accounts (A: deployer/borrower, B: lender).

| Area | Checked |
|---|---|
| Happy path | request → review → approve (B) → partial repay → full repay → `REPAID`, reputation `completed_loans: 1` |
| Default | approve → `mark_default` by non-owner rejected → by owner succeeds, reputation `defaulted_loans: 1` |
| Cancel | non-borrower rejected, borrower succeeds, review after cancel rejected |
| Score gate | profile page with no useful text scores 0; approval rejected below `min_score`, allowed after lowering it |
| Roles | borrower cannot approve own loan; only borrower records repayment; only owner marks default or sets minimum |
| Validation | amount `0`, term `0` / `366`, non-`https` URL, empty purpose, `min_score` `101`, unknown loan ID all rejected; failed requests leave no data behind |
| Re-repayment | recording a repayment on a `REPAID` loan is rejected |

## Frontend

`index.html` is a single static page with no build step. It uses [`genlayer-js`](https://github.com/genlayerlabs/genlayer-js) from a CDN to read the contract and send transactions on Studionet.

- On first load it creates a **test account** in your browser (`localStorage`). Studionet needs no gas, so it works immediately.
- Use **New account** or **Import key** to switch between borrower and lender, or to use the deployer key for owner-only actions.
- Never import a private key that holds real funds.
- Reviews can take a minute or two while validators fetch the page and reach consensus.
- Use a plain-text evidence URL (for example a raw GitHub README). A rendered GitHub profile page is mostly HTML and usually scores 0.

### Run locally

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
python3 -m http.server 8000
# open http://localhost:8000
```

### Deploy to GitHub Pages

1. Push this folder to a GitHub repository (`index.html` at the repository root).
2. Open **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**, select `main` and `/ (root)`, then save.
4. After a minute the site is live at `https://<your-username>.github.io/<repo-name>/`.

### Point the frontend at your own deployment

Edit this line near the bottom of `index.html`:

```js
const CONTRACT = "git push
0xB0CE8d3185AC9e4906e9cb26335E792556F1d30D";
```

## Deploy the contract yourself

1. Open [GenLayer Studio](https://studio.genlayer.com).
2. Create a new file and paste `contracts/undercollateralized_lending.py`.
3. Click **Deploy new instance** (the constructor takes no arguments). The deploying account becomes the owner.

## Limitations and next steps

- No escrow or token transfers: amounts are protocol units only.
- Anyone can create fresh addresses, so reputation history is weak against sybil behavior.
- Defaults are marked manually by the owner; there is no on-chain due date.
- Scores rest on one public page. Combining several evidence sources would make them more robust.
- Next steps: escrow integration, due dates with automatic default, multiple evidence URLs, lender-specific minimum scores.

## Repository structure

```
.
├── index.html                              # static frontend (GitHub Pages)
├── README.md
└── contracts/
    └── undercollateralized_lending.py      # Intelligent Contract
```
