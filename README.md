# OpenRoboto Miner and Protocol

> **Season change (2026-09-01): the simulation track now runs LingBot-VLA 2.0**
> (competition 2). The π0.5 simulation season is archived — its final board stays
> on the site. All submissions, both tracks, go through the packaged `openroboto`
> CLI, **1.1.0 or newer** — `pip install -U openroboto`
> ([PyPI](https://pypi.org/project/openroboto/) ·
> [source](https://github.com/openroboto-ai/openroboto-cli) ·
> [LingBot miner guide](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/MINER_LINGBOT.md)).
> The legacy `rt.py` flow in this repository predates the season vocabulary and is
> **deprecated for the new season** — it can burn a fee on a submission the season
> will refuse. This repository remains the season-1 (π0.5) protocol record.
> Rules and season status: [openroboto.ai/#/benchmark](https://www.openroboto.ai/#/benchmark) ·
> [docs](https://www.openroboto.ai/#/docs).
>
> **Real-robot track (xArm 6).** Its 20% share of emissions accrues to a dedicated
> prize-pool hotkey (UID 2, `5HVjAxFQ36vsNPcAWP5LBefutCtw8ishCCQj6VRfsDvERAZo`) and is
> settled once per season (95% to the champion over 120 days,
> 5% to qualified entries over 30 days). Every scored trial is published on complete
> video; a rewarded model must stay public for its whole payout period, and during that
> time **anyone can challenge it** by opening an issue in this repository — an upheld
> challenge burns whatever has not been paid. Full rules:
> [REAL_TRACK.md](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/REAL_TRACK.md).

OpenRoboto is a Bittensor mainnet subnet for improving vision-language-action models. This repository contains the public miner, on-chain protocol helpers, weight-setting validator, training runner, configuration examples, and reproducibility documentation for netuid 80.

[Protocol overview](docs/SUBNET_OVERVIEW.md) · [Seed derivation](docs/SEED_GENERATION.md) · [Evaluation toolkit](https://github.com/openroboto-ai/openroboto-evaluation)

## Public trust boundary

The following components are public:

- miner participation, local training, Hugging Face upload, burn, and chain announcement;
- chain commitment formats and weight-setting logic;
- evaluation rules, baseline methodology, LIBERO tooling, and seed derivation;
- the miner-visible `control.json` schema and read-only API contract.

Held-out task data, the scoring service deployment, and subnet-owner operational tools are outside this repository. Seed derivation remains public because the future block hash and drand value do not exist when a miner submits a model.

## Miner flow

*(Season-1 / π0.5 record. For the live LingBot-VLA 2.0 season, follow the
[CLI LingBot guide](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/MINER_LINGBOT.md)
instead.)*

1. Read the current public `control.json`.
2. Download the public training resources and base checkpoint.
3. Train through the isolated `openpi-runner` container.
4. Merge any LoRA adapter into the π0.5 base and export a complete checkpoint.
5. Upload the complete checkpoint to Hugging Face.
6. Pay the current evaluation burn and announce the exact model commit on chain.
7. Reproduce the published evaluation seed and run the public validator toolkit locally.

> **Submission format.** The evaluation service only accepts complete model checkpoints —
> an openpi JAX `params/` directory or a PyTorch `model.safetensors`, together with
> `assets/physical-intelligence/libero/norm_stats.json`. A bare LoRA adapter is rejected
> by a CPU pre-check before any GPU evaluation. Exact requirements and a local pre-check
> command are documented in [docs/SUBNET_OVERVIEW.md](docs/SUBNET_OVERVIEW.md).

## Installation

Requirements:

- Linux with an NVIDIA GPU and recent driver;
- Python 3.11;
- Docker with NVIDIA Container Toolkit;
- a registered Bittensor mainnet hotkey;
- a Hugging Face account with a write token.

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp miner.example.yaml miner.yaml
```

Fill the placeholders in `miner.yaml`. The file is ignored by Git.

## Run

Train:

```bash
python miner.py --config miner.yaml
```

Upload, burn, and announce:

```bash
python rt.py submit --config miner.yaml --round 1
```

The steps can also run separately:

```bash
python rt.py upload --config miner.yaml --round 1
python rt.py burn --config miner.yaml --round 1
python rt.py announce --config miner.yaml --round 1
```

Docker Compose starts only the miner service:

```bash
docker compose up --build miner
```

## Weight-setting validator

`validator.py` reads `control.json`, retrieves weights from a read-only API endpoint, normalizes them, and calls Bittensor `set_weights`. It does not contain the evaluation service or any owner controls.

```bash
cp validator.example.yaml validator.yaml
python validator.py --config validator.yaml
```

## Reproduce a seed

```python
from protocol.seed import derive_seed

seed = derive_seed(block_hash, round_num, drand_randomness)
```

The exact formula, drand chain identifier, verification steps, and security assumptions are documented in [docs/SEED_GENERATION.md](docs/SEED_GENERATION.md).

## Repository map

| Path | Purpose |
|---|---|
| `miner.py` | Miner training entry point (reference sample; its default LoRA output must be merged into the base model before submission) |
| `rt.py` | Upload, burn, and chain announcement CLI |
| `validator.py` | Read-only weight fetch and on-chain `set_weights` |
| `payment.py` | Evaluation-burn transaction helper |
| `miner/` | Training pipeline and Hugging Face publishing |
| `openpi-runner/` | Isolated OpenPI training runtime |
| `protocol/` | Public protocol types and seed derivation |
| `utils/` | Shared configuration, chain, download, and logging helpers |
| `docs/` | Miner, validator, protocol, API, and reproducibility documentation |

Local configuration, runtime state, logs, databases, environments, and model weights are excluded by `.gitignore`.

