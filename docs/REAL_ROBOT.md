# Real-robot documentation

> Updated 2026-09-07. Confirmed interface requirements are published below.
> This is not a competition-opening notice.

OpenRoboto is preparing parallel physical tracks for π0.5 and LingBot-VLA 2.0.
Model-facing participation documentation belongs in the packaged CLI repository;
this older subnet toolkit is not a second real-robot submission implementation.

- [Parallel real-robot tracks](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/REAL_TRACKS.md)
- [Real-robot miner guide](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/MINER_REAL.md)
- [Confirmed workstation interface](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/WORKSTATION_DATA.md)
- [Existing reward and challenge rules](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/REAL_TRACK.md)

The official evaluator handles hardware adaptation. Miners match the published
model-facing observation/action interface and the selected model family's
checkpoint requirements. The confirmed contract is 7-D joint-space state and
actions, with one continuous absolute gripper target. Joint deltas reference the
same inference-start state throughout a 50-step prediction; the default executed
prefix is 25 steps. The CLI document specifies radians, normalization, camera
preprocessing and safety behavior.

## Camera-view reference

![Workstation camera view showing the xArm6, gripper and tabletop work area](https://raw.githubusercontent.com/openroboto-ai/openroboto-cli/main/docs/images/real-track/workstation-camera-view.png)

Original viewer capture supplied by the workstation team, before model
preprocessing. The visible objects illustrate the scene and do not define an
evaluation task. The viewer controls are not part of the model image input.
See the [camera and preprocessing reference](https://github.com/openroboto-ai/openroboto-cli/blob/main/docs/WORKSTATION_DATA.md#camera-view-reference).

Use the selected environment's published competition ID and terms. Do not use
the deprecated `rt.py` flow in this repository to enter a new real competition.
A successful local format check is not proof of hardware compatibility or
backend admission. Retain the exact submitted revision and verify chain and
roster confirmation before treating an entry as submitted.
