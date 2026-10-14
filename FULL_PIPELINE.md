# Full Isaac Lab PPO pipeline

The repository provides two complementary training paths:

1. `src/crowdsim_ppo/`: a simulator-free reference pipeline for quickly reading
   and testing the algorithm.
2. `CrowdSim/` + `protomotions/`: the original Isaac Lab crowd-navigation
   pipeline with humanoids, robot sensors, maps and differential-drive control.

## Included dependency closure

```text
CrowdSim/ppo/train_ppo.py
├── CrowdSim/ppo/ppo_policy.py
├── CrowdSim/world/builder.py
│   ├── CrowdSim/protomotions_runtime.py
│   │   └── protomotions/
│   ├── CrowdSim/scene_setup.py
│   ├── CrowdSim/nav_manager.py
│   │   ├── CrowdSim/world/{map,planning,sfm}.py
│   │   └── CrowdSim/control/drive.py
│   └── CrowdSim/utils/
├── CrowdSim/tools/render_navigation_fast.py
└── CrowdSim/config/env.yaml
```

Preference PPO additionally uses `CrowdSim/pref/`. Expert trajectory collection
uses the selected files under `CrowdSim/flow/`. PPO-specific tests and evaluation
utilities are included under `CrowdSim/tests/` and `CrowdSim/tools/`.

## External files

The following large or separately distributed files are intentionally not stored
in this Git repository:

- MaskedMimic checkpoint at
  `protomotions/data/pretrained_models/masked_mimic/smpl/last.ckpt`
- warehouse USD referenced by `CrowdSim/config/scenes/warehouse.yaml`
- trained robot PPO and reward-model checkpoints

### MaskedMimic checkpoint

The checkpoint is supplied by the official
[`NVLabs/ProtoMotions`](https://github.com/NVlabs/ProtoMotions) submodule. Fetch
the submodule and its Git LFS files:

```bash
git submodule update --init --recursive protomotions
git lfs install
git -C protomotions lfs pull
```

The default in `CrowdSim/config/env.yaml` is
`protomotions/data/pretrained_models/masked_mimic/smpl/last.ckpt`. Keep the
resolved-config companion files beside `last.ckpt`.

### Warehouse USD

The default warehouse comes from the Isaac Sim environment assets:
[`/Isaac/Environments/Simple_Warehouse/warehouse_multiple_shelves.usd`](https://docs.isaacsim.omniverse.nvidia.com/latest/assets/usd_assets_environments.html).
Use the copy installed with Isaac Sim (or its Nucleus URL), then set `scene_usd`
in `CrowdSim/config/scenes/warehouse.yaml` to that location. The USD itself is
not stored in this repository.

### Robot PPO and reward-model checkpoints

No pretrained robot PPO or reward-model checkpoint is published with this
repository. Train one with this pipeline, or obtain the exact artifact from its
experiment owner, then supply its path through `--resume`, `--base-ppo`, or
`--reward-ckpt`.

Do not use an arbitrary older MaskedMimic checkpoint: its checkpoint format and
resolved configurations can be incompatible with the pinned ProtoMotions revision.

## Environment

Use an Isaac Sim / Isaac Lab environment compatible with the versions listed in
`requirements_isaaclab.txt`. The provided entry points import the simulator before
PyTorch in the required order.

## Train

```bash
python CrowdSim/ppo/train_ppo.py \
  --env-config CrowdSim/config/env.yaml \
  --total-steps 5000000 \
  --headless
```

Resume:

```bash
python CrowdSim/ppo/train_ppo.py \
  --env-config CrowdSim/config/env.yaml \
  --resume output/crowdsim_robot_ppo/RUN/robot_ppo_latest.pt \
  --headless
```

Collect expert trajectories:

```bash
python CrowdSim/flow/collect_ppo_buffer.py \
  --env-config CrowdSim/config/collect/ppo.yaml \
  --ppo-ckpt output/crowdsim_robot_ppo/RUN/robot_ppo_latest.pt
```

Preference-reward fine-tuning:

```bash
python CrowdSim/ppo/train_ppo_finetune.py \
  --env-config CrowdSim/config/env.yaml \
  --train-config CrowdSim/config/reward_model_ppo.yaml \
  --base-ppo PATH/robot_ppo_latest.pt \
  --reward-ckpt PATH/reward_model.pt \
  --headless
```
