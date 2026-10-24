# ConditionOracle

A reusable **GenLayer Intelligent Contract** primitive for resolving real-world claims on-chain through validator consensus.

Prediction markets, bounties, escrow, grants and DAO votes all need a trustworthy answer to one question: *"Did X happen?"* ConditionOracle is that building block. Other contracts can read its verdict and branch on it.

## How it works

1. **Create a claim.** Anyone calls `create_claim(statement, source_url, criteria)`.
   - `statement`: what is being claimed
   - `source_url`: a public web page that can prove or disprove it
   - `criteria`: plain-language rules for what counts as TRUE
2. **Resolve it.** Anyone calls `resolve(claim_id)`.
   - Each validator independently fetches the source page.
   - Each validator asks an LLM to judge the page against the criteria.
   - The verdict is one of `TRUE`, `FALSE` or `INCONCLUSIVE`.
3. **Read the result.** Any contract or user calls `get_verdict(claim_id)`.

## Consensus design

| Piece | Choice | Why |
|---|---|---|
| Non-deterministic block | Web fetch + LLM judgment | The answer depends on live web content |
| Equivalence principle | `gl.eq_principle.prompt_comparative` | Validators compare verdicts, not exact text |
| Output format | JSON with a fixed verdict vocabulary | Small, stable value makes agreement reliable |
| Agreement rule | `verdict` must be identical; `reasoning` may differ in wording | Wording varies between LLMs, the decision should not |
| Fallback | Anything unparseable becomes `INCONCLUSIVE` | Fails safe instead of guessing |
| Resolution | One-shot (`OPEN` to final) | Prevents replay and flip-flopping |
| Prompt injection | Page content is labeled untrusted in the prompt | Reduces the risk of hostile pages steering the verdict |
| Page size | Truncated to 8000 characters | Keeps the prompt bounded and validator-friendly |

## Contract interface

**Write methods**

| Method | Description |
|---|---|
| `create_claim(statement, source_url, criteria) -> int` | Registers a claim and returns its id |
| `resolve(claim_id) -> str` | Runs consensus and stores the final verdict |

**View methods**

| Method | Description |
|---|---|
| `get_verdict(claim_id) -> str` | `OPEN`, `TRUE`, `FALSE` or `INCONCLUSIVE` |
| `get_claim(claim_id) -> str` | Full claim record as a JSON string, including reasoning |
| `total_claims() -> int` | Number of claims created |

## Using it from other contracts

Other contracts can reference a deployed ConditionOracle by address and call `get_verdict(claim_id)`, then act on `"TRUE"`, `"FALSE"` or `"INCONCLUSIVE"` (release funds, settle a market, close a bounty, and so on).

## Testing in GenLayer Studio

1. Open GenLayer Studio and upload `condition_oracle.py`.
2. Keep the two header lines at the top of the file exactly as they are (version and `Depends` line).
3. Deploy with no constructor arguments.
4. Call `create_claim` with:
   - `statement`: `The page describes GenLayer`
   - `source_url`: `https://www.genlayer.com`
   - `criteria`: `The page mentions GenLayer`
5. Call `resolve` with `0`.
6. Call `get_verdict` with `0`. Expected: `"TRUE"`.
7. Call `get_claim` with `0` to see the stored reasoning.

## Test results

Deployed and tested in GenLayer Studio (Normal, Full Consensus mode).

| Step | Result |
|---|---|
| Deploy | ACCEPTED |
| `create_claim` (genlayer.com, "page mentions GenLayer") | FINALIZED |
| `resolve(0)` | ACCEPTED |
| `get_verdict(0)` | `"TRUE"` |
| `get_claim(0)` | Status `TRUE`, with validator reasoning stored on-chain |

Stored reasoning for the test claim: the page repeatedly references GenLayer, including headings and multiple mentions throughout the content.

Screenshots of the deployment and results are in the `screenshots/` folder (add them if you have uploaded them).

**Edge cases the contract handles**

- Resolving the same claim twice: error `claim already resolved`.
- Unknown claim id: error `claim not found`.
- URL without `http(s)://`: error `source_url must be an http(s) URL`.
- Unreadable or missing page content: verdict falls back to `INCONCLUSIVE`.

## Limitations and ideas

- Verdict quality depends on the source page being stable and readable as text.
- Pages that change after resolution do not change the stored verdict (by design).
- Possible extensions: multiple source URLs per claim, a dispute or re-resolve window, and a resolution deadline.

## License

MIT
