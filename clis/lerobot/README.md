# lerobot

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/lerobot>

**Hugging Face's end-to-end robot-learning stack — datasets, policies, simulation, and real-hardware drivers in one Python package.**
`lerobot` is the robotics-shaped sibling of `transformers` /
`diffusers`: a pretrained-model-and-dataset hub model applied to
manipulation, locomotion, and visuomotor control. Policies (ACT,
Diffusion Policy, π0 / π0-fast, SmolVLA, VQ-BeT, TD-MPC, SAC)
download from the HF Hub like any other model, datasets follow a
unified `LeRobotDataset` schema (timestamped frames + sensor
streams + episode boundaries) that converts to / from RLDS / MCAP
/ ROS2 bag, and the same training script targets a simulator
(Mujoco / IsaacSim / PyBullet via `gymnasium` envs from
`gym-aloha` / `gym-pusht` / `gym-xarm`) or a physical robot
(SO-100 / SO-101 arms, Koch arms, ALOHA bimanual, Stretch, Trossen
WidowX) over the same `Robot` interface. v0.5.x landed in
**2026-04** with a stabilised teleop API, expanded VLA policy
zoo, and tighter `transformers` / `accelerate` integration.

## 1. Footprint

- Python library + CLI (`lerobot-record`, `lerobot-replay`,
  `lerobot-teleoperate`, `lerobot-train`, `lerobot-eval`,
  `lerobot-calibrate`, `lerobot-find-port`).
- Hardware drivers ship in-tree for Feetech / Dynamixel / SO-ARM
  family / Koch / Stretch / Trossen WidowX / Unitree quadrupeds
  via USB-serial; cameras over OpenCV / RealSense / IntelRealSense
  / Phone (`lerobot-phone-camera`).
- Sim envs come as separate `gym-aloha` / `gym-pusht` /
  `gym-xarm` / `gym-hil` packages installed via extras
  (`pip install "lerobot[aloha]"`).
- Python ≥ 3.10, PyTorch ≥ 2.4. Real-robot loops run on CPU + a
  single GPU; VLA training (π0, SmolVLA) wants H100-class.

## 2. Repo + version + license

- Repo: <https://github.com/huggingface/lerobot>
- Latest release: **`v0.5.1`** (2026-04-07)
- HEAD on `main`: `05a5223` (2026-04-24)
- License: **Apache-2.0** —
  <https://github.com/huggingface/lerobot/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License blob SHA: `a603343cdda0228e53d8d9cdeb355f342f1e8f2c`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: Python

## 3. What's in the box

- **Policies**: ACT (Action Chunking Transformer, the original
  ALOHA paper's model), Diffusion Policy, VQ-BeT, TD-MPC2, SAC,
  π0 (Physical Intelligence's flow-matching VLA), π0-fast
  (autoregressive π0), SmolVLA (HF's compact VLA), Pi0-fast-base
  / Pi0-base / Pi0-fast-droid pretrained checkpoints from the
  Hub.
- **Datasets**: ~250+ datasets on HF Hub under `lerobot/*` and
  community namespaces (LIBERO, RoboCasa, DROID-100, OXE subsets,
  ALOHA-static, PushT, XArm-stacking, SO-100 community uploads).
  `LeRobotDataset` v2 format is the standard write target for
  new SO-100 / SO-101 user uploads.
- **Hardware**: SO-100 / SO-101 (HF + The Robot Studio's
  open-source ~$200 6-DoF arm), Koch v1.1, ALOHA bimanual,
  Stretch 3, Trossen WidowX-250 / VX-300, Unitree Go2 / G1.
  Teleop modalities: leader-follower (puppet a follower arm with
  an identical leader), VR (Meta Quest via `lerobot-teleop-vr`),
  smartphone, gamepad, keyboard.
- **Sim**: Mujoco-based `gym-aloha` / `gym-pusht` / `gym-xarm`
  / `gym-hil` envs that share the `LeRobotDataset` data shape
  with real-hardware recordings, so the same policy code trains
  in sim and runs on hardware.

## 4. Simple usage

```bash
# 1. Find which serial ports the SO-100 leader and follower are on
lerobot-find-port

# 2. Calibrate (writes joint offsets to ~/.cache/huggingface/lerobot)
lerobot-calibrate \
    --robot.type=so100_follower --robot.port=/dev/tty.usbmodem58760431541

# 3. Teleoperate: leader arm controls follower in real time
lerobot-teleoperate \
    --robot.type=so100_follower --robot.port=/dev/tty.usbmodem... \
    --teleop.type=so100_leader  --teleop.port=/dev/tty.usbmodem...

# 4. Record 30 demonstrations of "pick the cube and place in the bowl"
lerobot-record \
    --robot.type=so100_follower --robot.port=/dev/tty.usbmodem... \
    --teleop.type=so100_leader  --teleop.port=/dev/tty.usbmodem... \
    --dataset.repo_id=alice/so100-pick-cube --dataset.num_episodes=30 \
    --dataset.single_task="Pick the red cube and place it in the white bowl"

# 5. Train an ACT policy on those demonstrations
lerobot-train \
    --policy.type=act --dataset.repo_id=alice/so100-pick-cube \
    --output_dir=outputs/act-so100-pick-cube --batch_size=8 --steps=80000

# 6. Replay the trained policy on the real robot
lerobot-record \
    --robot.type=so100_follower --robot.port=/dev/tty.usbmodem... \
    --policy.path=outputs/act-so100-pick-cube/checkpoints/last/pretrained_model \
    --dataset.repo_id=alice/so100-pick-cube-eval --dataset.num_episodes=10
```

## 5. Why it's interesting

- **The HF Hub model applied to robotics** — datasets, policies,
  and pretrained VLAs (π0, SmolVLA, Pi0-fast) are all `repo_id`s
  you `huggingface-cli download`. New manipulation skill in the
  community? It's a `lerobot/<dataset_id>` upload, and `lerobot-train
  --dataset.repo_id=...` reproduces it; the same Hub muscle that
  made `transformers` ubiquitous applied to a field that previously
  shipped state in tar.gz on Google Drive.
- **One Python interface across leader-follower teleop, VR,
  gamepad, smartphone, and policy inference** — the
  `Robot` / `Teleop` / `Policy` ABCs are small enough to read in
  one sitting; adding a new arm is implementing `connect`,
  `disconnect`, `get_observation`, `send_action`. Real
  laboratories adopt it because the data-collection script and the
  policy-rollout script share the same `Robot` object.
- **VLA-class policies (π0 / SmolVLA) are first-class in-tree** —
  Physical Intelligence's π0 and HF's own SmolVLA are
  `policy.type=pi0` / `policy.type=smolvla` arguments to
  `lerobot-train`; the broader VLA conversation (RT-2,
  OpenVLA, π0, SmolVLA, RDT, Octo) lands as policy plugins
  rather than separate forks.
- **SO-100 hardware kit makes the on-ramp ~$200, not ~$30k** —
  the SO-100 / SO-101 are open-source 3D-printed-plus-Feetech-
  servos arms designed jointly with The Robot Studio specifically
  to be the cheapest plausible "real robot" for the LeRobot
  data-collection loop; "buy SO-100, follow tutorial, train ACT
  on 30 demos, watch it autonomously stack cubes" is a weekend
  project with bills of materials in the repo.
- **Sim and real share the dataset schema** — `LeRobotDataset`
  format is identical for `gym-aloha` rollouts and physical
  ALOHA recordings, so a sim-trained policy → real-hardware
  rollout is one `--policy.path=` swap, no data-format glue.

## 6. Caveats

- API still evolves — v0.4 → v0.5 renamed several CLI flags
  (`--robot.type` was previously `--robot-config.type`), and the
  `LeRobotDataset` v1 → v2 schema migration left some older
  community datasets needing `lerobot-dataset-convert` before
  they train. Pin the release tag.
- **Real-robot safety is your problem** — `lerobot-replay` and
  `lerobot-record --policy.path=...` happily drive a 6-DoF arm
  into the table; emergency stop, joint-limit checks, and
  workspace-volume guards are application code, not framework
  code. The docs say this; the code doesn't enforce it.
- **VLA training is GPU-heavy** — π0 / SmolVLA training jobs
  want H100-class with multi-GPU; ACT / Diffusion Policy / VQ-BeT
  on a single 24GB consumer card is fine for SO-100-scale tasks.
- **Hardware coverage is real but bounded** — beyond the listed
  arms / quadrupeds, USD humanoid platforms, industrial Universal
  Robots / Franka / Kuka are not supported in-tree (community
  forks exist); this is "research-grade SO-100 / ALOHA / Koch /
  Stretch", not a ROS2 replacement.

## 7. Where it sits in the catalog

- **Pair with**: [`huggingface-cli`](../huggingface-cli/) (push /
  pull datasets and policy checkpoints), [`trl`](../trl/) (RL
  post-training of VLA policies — GRPO over rollouts in the same
  `gymnasium` envs), [`peft`](../peft/) (LoRA-style fine-tuning of
  π0 / SmolVLA on a new task without retraining the whole VLA),
  [`text-generation-inference`](../text-generation-inference/) /
  [`vllm`](../vllm/) (serve the language-conditioning side of a
  VLA), [`gradio`](https://github.com/gradio-app/gradio) (the
  built-in `lerobot.scripts.visualize_dataset_html` UI is
  Gradio-based).
- **Don't use for**: classical-robotics motion planning (use ROS2
  / MoveIt2); industrial-arm certified-safety stacks (use vendor
  SDKs); humanoid whole-body control beyond Unitree (out of
  scope).
