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

The ProtoMotions runtime source and bundled robot assets used by this pipeline
are available directly in [the repository's `protomotions/` directory](https://github.com/yh83305/NavIsaaclab2.0/tree/main/protomotions).

| Artifact | Download or preparation method |
| --- | --- |
| MaskedMimic checkpoint | Obtain the checkpoint matching this repository's [ProtoMotions runtime](https://github.com/yh83305/NavIsaaclab2.0/tree/main/protomotions) from the [upstream pre-trained-model documentation](https://nvlabs.github.io/ProtoMotions/getting_started/pretrained_models.html). Keep its resolved-config companion files beside `last.ckpt`. |
| SMPL motion library | Prepare the AMASS motion library using the [upstream AMASS preparation guide](https://nvlabs.github.io/ProtoMotions/getting_started/amass_preparation.html), then set `motion_file` to the generated local file. AMASS access is subject to its own license. |
| USD scenes | Download each scene from its original distribution and set `scene_usd` in `CrowdSim/config/scenes/*.yaml` to its local path. DynBench USD assets are not bundled in this repository. |
| Robot PPO/reward checkpoints | Train them with this pipeline or obtain the exact experiment artifact from its owner. |

Do not use an arbitrary older MaskedMimic checkpoint: checkpoint formats and
resolved configurations can be incompatible across ProtoMotions revisions.

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
