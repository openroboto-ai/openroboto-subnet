# OpenRoboto Miner and Protocol

> **New: the real-robot track (xArm 6).** Submissions to the real-robot track are made
> through the packaged `openroboto` CLI — `pip install openroboto`
> ([PyPI](https://pypi.org/project/openroboto/) ·
> [source](https://github.com/openroboto-ai/openroboto-cli)) — which resolves the season,
> runs free local pre-checks, pays the entry fee, and announces on chain in one command.
> **The simulation track is unchanged**: everything documented in this repository,
> including `rt.py` submission, keeps working exactly as before. Rules and season status:
> [openroboto.ai/#/real](https://www.openroboto.ai/#/real) ·
> [docs](https://www.openroboto.ai/#/docs).

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

