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

- MaskedMimic checkpoint, normally
  `data/pretrained_models/masked_mimic/smpl/last.ckpt`
- motion file referenced by `CrowdSim/config/env.yaml`
- warehouse or other scene USD referenced by `CrowdSim/config/scenes/*.yaml`
- trained robot PPO and reward-model checkpoints

Place each artifact at its configured path or update the relevant YAML setting.
The required sources and setup steps are:

```bash
# Fetch the official ProtoMotions source and its Git-LFS checkpoint files.
git submodule update --init --recursive protomotions
git lfs install
git -C protomotions lfs pull
```

| Artifact | Exact source | Where to put it / configuration change |
| --- | --- |
| MaskedMimic checkpoint | The official `NVLabs/ProtoMotions` submodule, at [`protomotions/data/pretrained_models/masked_mimic/smpl/`](https://github.com/NVlabs/ProtoMotions/tree/main/data/pretrained_models/masked_mimic/smpl). `git -C protomotions lfs pull` downloads `last.ckpt` and its companion files. | The default in `CrowdSim/config/env.yaml` is `protomotions/data/pretrained_models/masked_mimic/smpl/last.ckpt`. Keep all resolved-config files next to `last.ckpt`. |
| SMPL motion library | [AMASS](https://amass.is.tue.mpg.de/) (registration and its license are required), prepared with the [ProtoMotions AMASS guide](https://nvlabs.github.io/ProtoMotions/getting_started/amass_preparation.html). | Save the generated MotionLib anywhere locally, then set `humanoid.motion_file` in `CrowdSim/config/env.yaml`; the example expects `../Assets/motion/amass_smpl_validation.pt`. |
| Default warehouse USD | Isaac Sim's installed/Nucleus environment asset: [`/Isaac/Environments/Simple_Warehouse/warehouse_multiple_shelves.usd`](https://docs.isaacsim.omniverse.nvidia.com/latest/assets/usd_assets_environments.html). | Point `scene_usd` in `CrowdSim/config/scenes/warehouse.yaml` to the local Isaac Sim asset-root copy or its Nucleus URL. This file is not shipped by this repository. |
| DynBench USD scenes and maps | [Hugging Face: `zgong313/DynBench`](https://huggingface.co/datasets/zgong313/DynBench), as documented by the [official FLUX repository](https://github.com/Zeying-Gong/FLUX). | Download with `huggingface-cli download zgong313/DynBench --repo-type dataset --local-dir /path/to/DynBench`; set the `scene_usd` and `scene_map` entries in `CrowdSim/config/scenes/dynabench_*.yaml` to `/path/to/DynBench/isaacsim_scene/...`. |
| Robot PPO and reward-model checkpoints | No pretrained artifact is published with this repository. | Train with this pipeline, or obtain the exact checkpoint from its experiment owner, then pass its path to `--resume`, `--base-ppo`, or `--reward-ckpt`. |

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
