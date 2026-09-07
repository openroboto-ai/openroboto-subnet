# Real-robot documentation

> Documentation draft · 2026-09-07. Publish together with the linked CLI drafts.
> The linked new pages may not yet exist on GitHub main while this change is under review.

OpenRoboto is preparing parallel physical tracks for π0.5 and LingBot-VLA 2.0.
Model-facing participation documentation belongs in the packaged CLI repository;
this older subnet toolkit is not a second real-robot submission implementation.

- [Parallel real-robot tracks](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/REAL_TRACKS.md)
- [Real-robot miner guide](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/MINER_REAL.md)
- [Workstation and reference data](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/WORKSTATION_DATA.md)
- [Existing reward and challenge rules](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/REAL_TRACK.md)

The official evaluator handles hardware adaptation. Miners match the published
model-facing observation/action interface and the selected model family's
checkpoint requirements. Exact interface parameters and reference data are
pending publication; this draft intentionally does not invent them.

Use the selected environment's published competition ID and terms. Do not use
the deprecated `rt.py` flow in this repository to enter a new real competition.
A successful local format check is not proof of hardware compatibility or
backend admission. Retain the exact submitted revision and verify chain and
roster confirmation before treating an entry as submitted.
