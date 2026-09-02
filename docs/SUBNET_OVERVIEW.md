# OpenRoboto Subnet — Protocol & Incentive Mechanism

**Bittensor subnet for open robot-learning models. Mainnet netuid 80.**

> **Season note (2026-09-01).** The simulation track now runs
> **LingBot-VLA 2.0** (competition 2, base model
> `openroboto-ai/lingbot-vla-v2-6b-libero`); the π0.5 simulation season is
> archived. The π0.5 walkthroughs below stand as the season-1 record — the
> protocol mechanics (fees, seeds, ranking, weights) are unchanged. The current
> miner path is the [`openroboto` CLI ≥ 1.1.0](https://github.com/openroboto-ai/openroboto-cli)
> and its [LingBot miner guide](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/MINER_LINGBOT.md).

This document explains how the subnet works, end to end: what miners submit, what it costs, how evaluation is randomized and executed, how ranking and on-chain weights are derived, and what keeps the loop honest. Nothing here needs to be taken on trust — each mechanism leaves a public trace you can check yourself: a chain transaction, an API response, a Hugging Face commit, a drand beacon round.

---

## 1. What this subnet does

Miners fine-tune an open vision-language-action (VLA) base model — **π0.5 (~3B params, [openpi](https://github.com/Physical-Intelligence/openpi))** — and publish their fine-tunes as **complete model checkpoints on Hugging Face**. Any training recipe is fine, LoRA included — but what you upload must be the full merged model, not a bare adapter (see §3 for the exact artifact requirements). The subnet evaluates every submission in simulation (LIBERO task suites in MuJoCo), ranks the results against the base-model baseline and each other, and pays miners through Bittensor emissions proportional to rank.

The output of the subnet is public: every submitted model is an open artifact anyone can download, and every score is reproducible from a deterministic, publicly verifiable seed.

## 2. Roles

| Role | Runs | Responsibility |
|---|---|---|
| **Miner** | `miner.py` (train) + `rt.py` (submit) | Fine-tune π0.5 (any recipe), export a full merged checkpoint, upload to own HF repo, pay the evaluation fee (burn), announce on chain |
| **Backend** | `backend/` service | Scan the chain for announcements, verify payment, derive seeds, manage the evaluation queue, compute rankings, serve the public API |
| **Benchmark worker** | separate GPU machine(s) | Poll the queue, load the pinned HF revision, run LIBERO suites in MuJoCo, push scores back (authenticated) |
| **Validator** | `validator.py` | Read the settled ranking from the API, normalize, call `set_weights` on chain |
| **Owner** | `owner/tools/` | Publish round config (`control.json`): round number, fee rate, training params, dataset URLs |

The backend is deliberately decoupled: miners never talk to it directly (they only write to the chain and Hugging Face), and validators only read from it.

## 3. Submission lifecycle

```
miner                      chain (netuid 80)                   backend + worker + validator
─────                      ──────────────────                  ────────────────────────────
fine-tune pi0.5 (any recipe)
merge → full checkpoint
upload to Hugging Face ──► burn (eval fee)
                           commitment {repo, commit, burn tx}
                                        │
                                        ▼
                           ChainScanner (60 s poll) ──────────► payment verified? ──no──► burn_rejected (terminal)
                                                                    │ yes
                                                                    ▼
                                                                model_hash check (LFS fingerprint)
                                                                    │
                                                                    ├── plagiarism detected → rejected (terminal)
                                                                    └── OK → seed computation
                                                                        │
                                                                        ├── drand available → pending (enqueued)
                                                                        └── drand unavailable → seed_failed (retryable)
                                                                            │
                                                                            ▼
                                                                        retry next scan cycle
                                                                            │
                                                                            ├── success → pending
                                                                            └── failure → seed_retry_count++

                                                              worker evaluates (6 suites)
                                                              scores posted → evaluated / eval_failed
                                                              ranking resolved
                           set_weights ◄────────────────────────  validator reads ranking
```

A submission moves through these states (unified status model, defined in `backend/protocol/status.py`):

| Status | Meaning | Terminal |
|--------|---------|----------|
| `received` | Announcement seen on chain, not yet verified | No |
| `burn_checking` | Burn verification in progress | No |
| `burn_passed` | Burn verified, awaiting seed computation | No |
| `burn_rejected` | Burn verification failed | **Yes** |
| `pending` | Payment verified, task enqueued for evaluation | No |
| `seed_failed` | Seed computation failed (drand unavailable), auto-retried each scan cycle | No |
| `evaluating` | Worker is evaluating (stage: `downloading`/`prechecking`/`running`) | No |
| `evaluated` | Evaluation completed, scores recorded | **Yes** |
| `eval_failed` | Evaluation ran but failed (e.g. corrupted weights); no score, no ranking entry | **Yes** |
| `rejected` | Payment or format verification failed, or plagiarism detected; never enqueued | **Yes** |

**Old status mapping** (backward compatible): `done` → `evaluated`, `failed` → `eval_failed`, `enqueued` → `pending`, `waiting` → `evaluating`.

One commitment per hotkey per round. Re-submitting overwrites the previous entry and resets it to the start of the pipeline.

### What the uploaded repo must contain

The evaluation worker loads **complete model checkpoints only**. Before any GPU time is spent, every submission goes through a structural pre-check (`libero_eval/check_model.py` in the evaluation repo); a submission that fails it is marked `eval_failed`, and the rejection reason is recorded so the miner can see exactly why.

| Requirement | Detail |
|---|---|
| Checkpoint format (one of) | openpi JAX: a `params/` directory (orbax OCDBT) · openpi PyTorch: a `model.safetensors` file |
| Normalization stats | `assets/physical-intelligence/libero/norm_stats.json` (state dim 8, action dim 7) |
| Architecture | Must match π0.5 (`pi05_libero` inference config); total parameter count within 2.5B–4.5B |
| **Not accepted** | **A bare LoRA adapter** (`adapter_config.json` + `adapter_model.safetensors`). The worker performs no merging — if you train with LoRA, merge the adapter back into the π0.5 base and export the full checkpoint before uploading. |

The pre-check is pure CPU and public — run it yourself before paying the submission fee:

```bash
# from the evaluation repo (github.com/openroboto-ai/openroboto-evaluation)
uv run libero_eval/check_model.py --model /path/to/checkpoint --config pi05_libero
# exit 0 = will be accepted, exit 1 = would be rejected (reasons printed)
```

## 4. The evaluation fee (burn)

Every submission is preceded by a small payment made by **burning TAO** — destroyed, not transferred.

- **Why charge?** Each submission consumes real GPU hours (6 simulation suites per model). A per-submission fee makes flooding the queue economically irrational.
- **Why burn instead of pay?** There is no recipient. The operator earns nothing from fees, which removes any incentive to farm submissions or sell evaluation slots.
- **How much?** Published in `control.json` → `payment.burn_rate_tao`. Currently **0.1 TAO**. A rate of 0 opens a free period (burn verification is skipped entirely).
- **Payment is bound to the submission.** The burn transaction hash (`b`) and block number (`bb`) are embedded in the commitment payload itself (see §11), so a submission cannot claim someone else's payment.

The backend verifies each payment **against the chain, fail-closed**:

1. The burn tx exists in the claimed block (single-block lookup, no trust in the miner's claim).
2. The signer is the submitting hotkey (or its owning coldkey).
3. The amount matches the published rate.
4. The tx has not been used by any prior submission (anti-replay, DB + in-memory check).
5. The burn block is within a bounded window of the commitment block — currently **10 blocks (~2 minutes)**. This is the second half of the anti-replay design: a burn cannot be stockpiled and attached to a later submission, and a stale burn cannot be reused after a failed attempt.

If any check fails, the submission is marked `burn_rejected` and never enters the queue — and the burned TAO is **not refunded**. Payment records are auditable via `/api/v1/payments/*` (admin) and summarized publicly.

**Burn hash verification uses strict exact match** — no `startswith` prefix matching. This prevents false positives from truncated `extrinsic_hash`.

> **Practical implication for miners:** the burn tx and the commitment must land on chain within ~2 minutes of each other. Do not run `burn` and `announce` as separate manual steps — any delay between them (wallet prompt, network retry, debugging) will exceed the window, the submission is rejected, and the fee is lost. Always use **`rt.py submit`**, which runs upload → burn → announce back-to-back in a single command.

## 5. Deterministic, unpredictable seeds

Each task's evaluation seed is derived from **two independent public beacons**, neither of which the miner or the operator controls:

1. **Bittensor block hash** of the block containing the miner's commitment (miners cannot choose which block includes their transaction), and
2. **drand** — the public distributed randomness beacon ([drand.love](https://drand.love)), fetched at task creation.

```
seed = uint32( last 4 bytes of SHA256( "{block_hash}:{round_num}:{drand_randomness}" ) )
```

The seed is unpredictable before submission, frozen after it, and reproducible by anyone: the API exposes `block_hash`, `drand_round`, `drand_random`, and the derived `seed` for every task, and the drand value can be checked byte-for-byte at `https://api.drand.sh/public/{round}`. Even if one beacon were compromised, the other still guarantees unpredictability.

**Seed failure handling**: If drand is temporarily unavailable (network issue, rate limit), the task enters `seed_failed` status — a non-terminal, retryable state. The scanner automatically retries seed computation on each scan cycle. On success, the task moves to `pending` (enqueued). On continued failure, `seed_retry_count` is incremented. This is an infrastructure issue, not a submission fault — submissions are never rejected due to drand unavailability.

Full derivation and verification code lives in [SEED_GENERATION.md](./SEED_GENERATION.md).

## 6. Evaluation

Submissions are evaluated in MuJoCo on the LIBERO task suites. The current round scores six: `libero_spatial`, `libero_object`, `libero_goal`, `libero_10`, plus two perturbation variants, `libero_object_swap` and `libero_spatial_swap`.

The swap suites earn their place: an internal red-team study showed the four base suites alone could be saturated for roughly $50 of GPU time by memorizing public demonstrations. The swap perturbations showed zero transfer from that attack — an attacker has to pay separately for each perturbation dimension — which is what makes the composition defensible. Any change to the scoring composition lands as a new round, never silently mid-round.

The worker loads the exact HF revision pinned in the commitment, so the artifact under evaluation is frozen at submission time. Scores are pushed back over an authenticated, fail-closed endpoint (`POST /api/v1/benchmark/task/{id}/score`, admin key): unauthenticated score submission is rejected, and a failed evaluation (`eval_failed`) writes no scores and creates no ranking entry.

The worker can also report progress via `PATCH /api/v1/benchmark/task/{id}/status` (stages: `downloading`, `prechecking`, `running`), which updates the `stage` column in the database for real-time visibility.

## 7. Ranking — challenge rules (king-of-the-hill)

The subnet does not rank by raw score alone; it uses a challenge system designed to reward beating the incumbent, not tying it:

1. Scored miners are ordered by earliest scoring time; the first becomes the initial **champion** (Rank 1).
2. Each subsequent miner **challenges the current champion**: the challenge succeeds only if `challenger_score > champion_score + champion_margin` (default **0.01**) — **strictly greater**. A lead of exactly the margin is a failed challenge; ties lose on purpose, so resubmitting the champion's own weights can never take the crown.
3. If the challenge succeeds, the challenger becomes the new champion, the old champion drops to Rank 2, and the rest shift down.
4. If it fails, the challenger **does not appear on the board at all**.
5. The board is capped at **Top 3**, with emission weights **70 / 20 / 10**.

Consequences worth noting: copying the current champion's weights cannot dethrone it (a copy ties, and a tie loses by margin); and a settled round's champion is **held** until someone clears the bar in a later round.

## 8. Weights on chain

The backend resolves the ranking and exposes it via the API. A lightweight validator reads it, normalizes to u16, and calls `set_weights` on netuid 80 — with strict success checking (the call's actual `is_success` flag is verified rather than assumed). Anyone can confirm the emitted weights:

```bash
btcli subnet metagraph --netuid 80 --network finney
```

Validator weight is how the **simulation** track pays its champion. The **real-robot**
track (xArm 6) is paid differently: its 20% share of emissions accrues to a dedicated
prize-pool hotkey (UID 2, `5HVjAxFQ36vsNPcAWP5LBefutCtw8ishCCQj6VRfsDvERAZo`, visible in
the metagraph command above) and is settled once per season — 95% to the single champion vested
over 120 days, 5% to every entry that clears the qualification bar vested over 30 days,
anything the rules withhold burned and recorded publicly. Every scored trial is
published on complete video with its record. A rewarded model must stay public for its
whole payout period, and anyone may challenge it for cheating during that time by
opening an issue on this repository; an upheld challenge burns whatever has not yet
been paid. The full ruleset lives in
[REAL_TRACK.md](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/REAL_TRACK.md).

## 9. Anti-gaming summary

| Attack | Countermeasure |
|---|---|
| Flood the queue with junk submissions | Per-submission burn fee (§4); one commitment per hotkey per round (§3) |
| Overfit to a known evaluation seed | Seed derived from block hash + drand, unknowable pre-submission (§5); seed_failed retry instead of silent failure |
| Memorize public demos to saturate suites | Swap-perturbation suites with zero attack transfer (§6) |
| Swap model weights after scoring | Commitment pins the exact HF commit hash; changing weights changes the hash and resets the submission (§3, §11) |
| Resubmit someone else's weights under a new name | Weight fingerprinting via HF LFS metadata (sha256 of every shard) — stored as `repo_hash` in DB |
| Impersonate another miner's repo | HF repo name must end with the submitting hotkey's last 12 SS58 characters |
| Point at someone else's fee payment | Burn tx embedded in the commitment payload; signer must match the submitting hotkey (§4) |
| Reuse one burn tx across submissions | Anti-replay check, DB + in-memory (§4); strict exact match for tx hash |
| Copy the champion's model to steal Rank 1 | Challenge margin: a tie is a failed challenge (§7) |
| Forge scores into the backend | Score endpoint is authenticated and fail-closed (§6) |

## 10. Public API

Live at **`https://api.openroboto.ai`** — the endpoints below are public, **no API key required**:

| Endpoint | Returns |
|---|---|
| `GET /health` | Liveness |
| `GET /api/v1/rounds/current` | Active round, baseline model, champion |
| `GET /api/v1/leaderboard` | Ranked rows with scores, delta vs baseline, audit links |
| `GET /api/v1/queue/status` | Evaluation queue: per-task status (pending / evaluating / evaluated / eval_failed) |
| `GET /api/v1/submissions/{task_id}` | Single-submission audit record (`task_id` = `task_{hotkey}_{round}`) |

The website renders the same data live: [openroboto.ai/#/benchmark](https://www.openroboto.ai/#/benchmark) (leaderboard) and [openroboto.ai/#/queue](https://www.openroboto.ai/#/queue) (queue). The full endpoint reference, including authenticated worker/admin routes, is in [api_reference_en.md](./api_reference_en.md).

## 11. Chain commitment format

The miner's announcement is a JSON payload stored on chain via Commitments (fits `Data::BigRaw`, ≤ 512 bytes):

```json
{
  "s":  "<hotkey SS58, full>",
  "h":  "<block hash at announcement, hex, no 0x>",
  "c":  "<HF commit hash, 40 hex chars>",
  "r":  <round number>,
  "i":  "<HF repo id, e.g. user/pi05-xxxxxxxxxxxx>",
  "b":  "<burn tx hash, hex, no 0x>",
  "bb": <burn block number>
}
```

This single payload binds together the miner's identity (`s`), the exact model artifact (`i` + `c`), the fee payment (`b` + `bb`), and the round (`r`). The backend's chain scanner decodes it, runs the §4 payment checks, and creates the evaluation task.

**Repo naming rule:** the HF repo name must end with the last 12 characters of the submitting hotkey's SS58 address (e.g. hotkey `…AAAAAAAAAAAA` → repo `user/pi05-AAAAAAAAAAAA`). This makes repo squatting and impersonation detectable at scan time.

## 12. Round configuration (`control.json`)

The owner publishes one JSON file over plain HTTPS that every miner polls (ETag-cached). It carries the current `round` number and `status`, `payment.burn_rate_tao`, dataset URLs, and reference training hyper-parameters. Round transitions, fee changes, and pauses are all announced through this file — there is no privileged channel between the operator and any miner. See [control_json.md](./control_json.md) for details.

## 13. Links

| What | Where |
|---|---|
| Website (live data) | <https://www.openroboto.ai> |
| Leaderboard | <https://www.openroboto.ai/#/benchmark> |
| Public queue | <https://www.openroboto.ai/#/queue> |
| API health | <https://api.openroboto.ai/health> |
| Open Data Pool | <https://huggingface.co/buckets/openroboto-ai/datapool> |
| Base model (openpi π0.5) | <https://github.com/Physical-Intelligence/openpi> |
| LIBERO benchmark | <https://github.com/Lifelong-Robot-Learning/LIBERO> |
| drand beacon | <https://drand.love> |
| Miner guide | [MINER.md](./MINER.md) · [MINER_DEPLOY.md](./MINER_DEPLOY.md) |
| Validator guide | [VALIDATOR.md](./VALIDATOR.md) |
| Seed derivation spec | [SEED_GENERATION.md](./SEED_GENERATION.md) |
| API reference | [api_reference_en.md](./api_reference_en.md) |