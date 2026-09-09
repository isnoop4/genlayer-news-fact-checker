# News Fact Checker — GenLayer Intelligent Contract

A fact-checking contract where GenLayer's leader/validator consensus
fetches real sources itself and cross-verifies a claim against them —
grounded in verifiable evidence, not a claim the caller could
fabricate.

## Project Summary

Fact-checking today usually relies on centralized editorial judgment
or simple keyword matching. This project lets GenLayer's AI validator
set fetch up to three independent sources itself and judge whether
they support, refute, or conflict on a plain-language claim —
settling `SUPPORTED` / `REFUTED` / `MIXED` / `UNVERIFIABLE` on-chain,
with every validator independently re-fetching the same sources
before consensus is reached.

## Why GenLayer

- **AI judgement is the core value**: deciding whether real-world,
  unstructured text confirms, contradicts, or is silent on a claim is
  exactly the kind of nuanced, natural-language task a deterministic
  smart contract cannot perform.
- **Web-aware, multi-source decisions**: the contract fetches every
  source itself inside the non-deterministic execution flow, rather
  than trusting claims typed in by whoever calls the contract.
- **Consensus, not a single model call**: each validator
  independently re-fetches the same sources and re-runs the judgement,
  then only needs to agree with the leader on the settlement decision
  — not the exact wording of the reasoning.

## Live Demo

[DEMO_URL — e.g. https://isnoop4.github.io/genlayer-news-fact-checker/]

## Contract Details

| Field | Value |
|---|---|
| Network | GenLayer Studio (Studionet) |
| RPC | https://studio.genlayer.com/api |
| Chain ID | 61999 |
| Contract address | 0x8A511e3CDC8CAA7907E4c45707A15bB77abBD67C |
| Explorer link | https://explorer-studio.genlayer.com/address/0x8A511e3CDC8CAA7907E4c45707A15bB77abBD67C |

## Tech Stack

- **GenLayer Intelligent Contract** — Python contract (`NewsFactChecker.py`)
- **Frontend** — single-file static HTML/JS (`index.html`), calling the
  contract directly through the `genlayer-js` SDK, hosted on GitHub
  Pages (no build step)
- **Backend/database** — none; the contract is the sole source of
  truth for the claim, verification status, and verdict

## How It Works

1. **Deploy** — Contract is deployed with a plain-language claim
   (e.g. *"The Eiffel Tower is located in Paris, France"*).
2. **Verify** — Anyone can call `verify_claim(source_url_1,
   source_url_2, source_url_3)` with one to three links (empty string
   for unused slots). This triggers the leader/validator consensus
   flow:
   - The **leader** fetches every provided URL itself
     (`gl.nondet.web.render`) and asks an LLM to judge the claim
     strictly against that fetched content, labeled per source.
   - Each **validator** independently fetches the same URLs and
     independently queries the model, then compares only the
     `verdict` field against the leader's result — not the wording of
     the reasoning, since independent LLM calls may phrase things
     differently even when they agree on the outcome.
3. **Settle** — Once consensus is reached, the verdict is written
   on-chain along with a confidence level and a short explanation
   citing which source(s) support or contradict the claim. The claim
   is marked verified and cannot be verified again.

### Verdict categories

- **SUPPORTED** — the sources consistently confirm the claim
- **REFUTED** — the sources consistently contradict the claim
- **MIXED** — the sources conflict with each other on the claim
- **UNVERIFIABLE** — the sources don't contain enough relevant
  information to judge the claim

### Verdict JSON schema

```json
{
  "verdict": "SUPPORTED" | "REFUTED" | "MIXED" | "UNVERIFIABLE",
  "confidence": "HIGH" | "MEDIUM" | "LOW",
  "reason": "brief explanation citing which source(s) support or contradict the claim"
}
```

### Contract methods

**Write**
- `verify_claim(source_url_1: str, source_url_2: str, source_url_3: str)` —
  callable once, by anyone; pass `""` for unused source slots

**Read**
- `get_claim()` — the claim being checked
- `get_is_verified()` — whether the claim has been settled
- `get_verdict()` — the verdict JSON (`PENDING` until verified)

## How to Run Locally

```bash
# 1. Clone the repo
git clone [YOUR_REPO_URL]
cd [YOUR_REPO_NAME]

# 2. Open GenLayer Studio and load NewsFactChecker.py
# 3. Deploy the contract with a constructor arg:
#    claim: <plain-language factual claim>

# 4. Call verify_claim from any account, passing 1-3 source URLs
#    (use "" for unused slots)

# 5. Check get_verdict / get_is_verified to see the result
```

### Running the frontend

`index.html` requires no build step. Two values must be set near the
bottom of the file before use:

```js
const CONTRACT_ADDRESS = "<your deployed contract address>";
const NETWORK = "studionet"; // or "testnetAsimov" once on public testnet
```

Then either open the file directly in a browser, or serve it locally:

```bash
python -m http.server
```

To deploy on GitHub Pages: place `index.html` at the repo root (or in
a `docs/` folder), then enable Pages in the repo settings pointing at
that branch/folder.

## Live Instance

The contract linked above is deployed with the claim:

> "The Great Wall of China is not visible from space with the naked eye"

This claim is left **unverified (PENDING)** intentionally, so it can
be tested interactively:

- Suggested source: `https://en.wikipedia.org/wiki/Great_Wall_of_China`
- Call `verify_claim` with that URL as `source_url_1` (leave
  `source_url_2` / `source_url_3` as `""`) and expect `SUPPORTED`,
  since this is a well-documented myth the source explicitly
  addresses.

## Demo Evidence (historical, from development testing)

Separate, already-verified deployments of this same contract, kept
only for reference — **not** the live contract linked above:

- **Claim**: `The Eiffel Tower is located in Paris, France`
  - Source: `https://en.wikipedia.org/wiki/Eiffel_Tower` →
    `SUPPORTED` (confidence: HIGH)
  - Two sources at once (`.../Eiffel_Tower` +
    `.../Gustave_Eiffel`) → `SUPPORTED` (confidence: HIGH), with the
    reasoning explicitly citing "Source 1" content
- **Claim**: `The Eiffel Tower is located in London, England`
  - Source: `https://en.wikipedia.org/wiki/Eiffel_Tower` →
    `REFUTED` (confidence: HIGH), citing the source's coordinates and
    explicitly noting the contradiction

## Known Limitations

- Sources protected by anti-bot measures (e.g. Cloudflare challenge
  pages) will fail to fetch with a `WEBPAGE_LOAD_FAILED` error —
  publicly accessible pages without such protection (e.g. Wikipedia,
  most standard news sites) work reliably.
- If a source's content changes between the leader's and a
  validator's fetch (e.g. live-updating pages), validators may
  disagree even on a claim with a clear answer — static reference
  pages are more reliable than live/dynamic content.
- `verify_claim` currently has no caller restriction — any address
  can trigger verification once, choosing any URLs. This is an
  intentional "anyone can verify" design; it does not restrict
  verification to the deployer.
- Only up to three sources are supported per claim (fixed parameters,
  not a dynamic list) to avoid relying on untested array-type
  contract parameters.
- Each contract instance handles exactly one claim. A new
  fact-check requires deploying a new instance.
- No dispute/appeal mechanism if a verdict is considered incorrect
  after the fact — verification is final once consensus is reached.

## Future Roadmap

- Support for a dynamic list of sources instead of a fixed three
- Optional caller allowlist for who may trigger `verify_claim`
- Factory contract to support multiple concurrent claims without
  redeploying
- Source credibility weighting (e.g. treating some domains as more
  authoritative than others)

## Security Notes

- All state changes (`verdict_json`, `is_verified`) occur strictly in
  the deterministic write path, **after** the leader/validator
  non-deterministic consensus completes — no writes are reachable
  from inside the AI evaluation itself.
- The model's response is parsed strictly against an expected JSON
  schema; malformed or out-of-range values (e.g. a verdict outside
  the four allowed categories) raise an error rather than being
  silently accepted.
- Evidence is fetched by the contract itself from caller-supplied
  URLs, not accepted as free-form text — this prevents a caller from
  fabricating "evidence" to manipulate the outcome directly, though
  resolution quality still depends on which URLs are supplied.
- The wallet-facing frontend never handles a private key; signing is
  delegated to the user's browser wallet extension.

---

*Built with GenLayer Intelligent Contracts.*
