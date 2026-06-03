# 1、Cross Sea-Air Optical Beam Alignment

This project implements the optical beam alignment part of the paper, including the JONSWAP-based dynamic sea surface, cross sea-air refraction, received optical power calculation, link interruption judgment, and three continuous-control algorithms: E2-PPO, standard PPO, and TD3.

## Environment

```bash
pip install -r requirements.txt
```

The sea state folders are fixed as follows:

| Folder | Meaning |
| --- | --- |
| `zero` | calm |
| `two` | sea state 2 |
| `four` | sea state 4 |

## Training

The default E2-PPO settings are: `max_ep_len=1000`, `T=3e6`, `update_timestep=4000`, `K_epochs=80`, `epsilon=0.2`, `gamma=0.99`, `lr_actor=0.0003`, `lr_critic=0.001`, `sigma0=0.6`, and `sigma_min=0.1`. E2-PPO uses an exponential action-standard-deviation schedule with `delta=1.0` and `decay=0.35`.

```bash
python train.py --algo e2ppo --sea four --seed 0
python train.py --algo ppo --sea two --seed 0
python train.py --algo td3 --sea zero --seed 0
```

Trained models are saved to:

```text
PPO_preTrained/{sea}/{algo}/
```

## Testing

```bash
python test.py --algo e2ppo --sea four --checkpoint latest --episodes 1
```

`test.py` reports the episode reward, average pointing error, average received optical power, interrupted steps, and total angular adjustment. When `--checkpoint latest` is used, the latest `.pth` file in the corresponding sea state and algorithm folder is loaded automatically.

# 2、Cross Sea-Air Target Tracking

A multi-agent target tracking system based on MADDPG / MASAC / ED-MADDPG. An AAV (Aerial Autonomous Vehicle) and an AUV (Autonomous Underwater Vehicle) collaborate to track a target via optical communication links under various sea states.

## Environment Setup

```bash
# 1. Create and activate Conda environment
conda create -n XXX python=3.6
activate XXX

# 2. Install dependencies
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple tensorflow==1.14.0 gym==0.10.5 numpy-stl

# 3. Install project packages
cd Tracking
pip install -e .
cd multiagent-particle-envs
pip install -e .
cd ..
```

## Project Structure

```
Target Tracking/
├── Tracking/
│   ├── maddpg/              # MADDPG / MASAC algorithm implementation
│   │   ├── common/          # TensorFlow utilities, distributions
│   │   ├── trainer/         # Experience replay buffer
│   │   ├── maddpg_gai.py    # MADDPG trainer
│   │   └── masac_tf.py      # MASAC trainer
│   ├── multiagent-particle-envs/
│   │   └── multiagent/
│   │       └── scenarios/   # Simulation environment
│   │           ├── threeD.py          # Cross sea-air target tracking scenario definition
│   │           ├── tracking_utils.py  # Channel model, EKF, sea states
│   │           └── environment.py     # Gym environment wrapper
│   ├── experiments/
│   │   └── train_tmp.py     # Training entry script
│   └── setup.py
└── paper.tex
```

## Agent System

| Agent | Role | Acceleration | Max Speed | Action Space |
|-------|------|--------------|-----------|--------------|
| Agent 0 (AAV) | Aerial vehicle | 1.0 | 1.0 | `[-1.0, 1.0]³` |
| Agent 1 (AUV) | Underwater vehicle | 0.6 | 0.6 | `[-0.6, 0.6]³` |

- **Observation space**: 15-dimensional (self velocity + position + target relative position + other agent relative position + other agent velocity)
- **Reward function**: Shared reward = optical communication quality (log power ratio) + tracking distance reward
- **ED-MADDPG**: AAV estimates AUV position via EKF + belief fusion instead of using ground-truth position

## Training

```bash
cd Tracking/experiments

python train_tmp.py --scenario threeD --algorithm alo-name --sea-state rank
```

### Common Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--algorithm` | `ed-maddpg` | Algorithm: `ed-maddpg` / `maddpg` / `masac` |
| `--sea-state` | `calm` | Sea state: `calm` / `sea2` / `sea4` |
| `--seed` | `1` | Random seed |
| `--num-episodes` | `20000` | Number of training episodes |
| `--max-episode-len` | `500` | Maximum steps per episode |
| `--lr` | `1e-3` | Adam learning rate |
| `--gamma` | `0.95` | Discount factor |
| `--batch-size` | `1024` | Batch size |
| `--save-rate` | `100` | Model saving interval (episodes) |
| `--restore` | — | Resume training from a saved model |

## Testing / Benchmark

```bash
# Load a trained model for testing
python train_tmp.py --scenario threeD --algorithm ed-maddpg --sea-state calm --benchmark --load-dir <model_path>
```

Results are saved in `Tracking/experiments/results/{algorithm}/{sea-state}/seed_{seed}/`.
