## 1. 개요

이 문서는 ROBOTIS OpenSourceTeam의 **AI MANIPULATOR #6: Beginner's Guide to Physical AI with OMX & LeRobot** 영상을 바탕으로, OMX Manipulator와 LeRobot을 이용하여 Physical AI를 구현하는 전체 과정을 정리한 문서이다.

LeRobot을 이용하면 실제 로봇을 조작하여 데이터를 수집하고, 수집한 Dataset으로 AI Policy를 학습한 뒤, 학습된 Policy를 실제 로봇에 적용할 수 있다.

전체 과정은 다음과 같다.

```text
환경 설정
   ↓
OMX 연결 및 Calibration
   ↓
Teleoperation
   ↓
Dataset Collection
   ↓
Policy Training
   ↓
Inference / Rollout
   ↓
OMX 자율 동작
```

ROBOTIS 공식 OMX 문서에서도 OMX를 **Imitation Learning과 Teleoperation을 위한 Physical AI Manipulator**로 설명하고 있으며, OMX-F(Follower)와 OMX-L(Leader) 구조를 사용한다.

---

# 2. 시스템 구성

## 2.1 OMX

OMX는 ROBOTIS에서 제공하는 소형 Physical AI Manipulator이다.

OMX 시스템은 기본적으로 다음과 같이 구성된다.

```text
OMX-L (Leader)
      │
      │ Teleoperation
      ↓
OMX-F (Follower)
      │
      ↓
   실제 작업
```

* **OMX-L** : 사람이 직접 조작하는 Leader
* **OMX-F** : Leader의 움직임을 따라가는 Follower
* **Camera** : 로봇 주변 환경을 촬영
* **PC** : LeRobot 실행 및 데이터 수집/학습

OMX-L은 5-DOF + Gripper 구조로 구성된다.

---

# 3. LeRobot 설치 및 환경 구성

LeRobot은 실제 로봇을 제어하고 Dataset을 기록하며 AI Policy를 학습하기 위한 Python 기반 로봇 학습 프레임워크이다.

현재 LeRobot은 다음과 같은 흐름을 중심으로 사용한다.

```text
Teleoperate
     ↓
Record
     ↓
Train
     ↓
Deploy / Rollout
```

### LeRobot 저장소 Clone

```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
```

### Python 환경에서 설치

```bash
pip install -e .
```

설치가 완료되었는지 확인한다.

```bash
lerobot-find-port
```

---

# 4. USB 포트 확인

OMX를 PC에 연결하면 USB Serial Port가 생성된다.

Leader와 Follower의 포트를 확인하기 위해 다음 명령어를 사용한다.

```bash
lerobot-find-port
```

실행하면 연결된 MotorsBus의 포트를 확인할 수 있다.

예:

```text
Finding all available ports for the MotorsBus.

Ports before disconnecting:
['/dev/ttyACM0', '/dev/ttyACM1']

Remove the USB cable from your MotorsBus and press Enter when done.
```

USB를 하나씩 분리하면서 어느 포트가 Leader/Follower에 해당하는지 확인한다.

예를 들어 다음과 같이 설정할 수 있다.

```text
Follower → /dev/ttyACM0
Leader   → /dev/ttyACM1
```

**주의:** 위 포트는 동영상에서의 예시이므로 실제 PC에서는 `lerobot-find-port` 결과를 사용해야 한다.

---

# 5. OMX Teleoperation

환경 설정이 완료되면 먼저 Leader와 Follower가 정상적으로 연결되는지 확인한다.

```bash
python -m lerobot.teleoperate \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --teleop.type=omx_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=omx_leader_arm
```

최신 CLI 환경에서는 다음과 같은 형태로 사용할 수도 있다.

```bash
lerobot-teleoperate \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --teleop.type=omx_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=omx_leader_arm
```

이 명령을 실행하면 사람이 **OMX-L(Leader)**를 움직였을 때 **OMX-F(Follower)**가 해당 움직임을 따라간다.

---

# 6. Camera 연결

Physical AI에서는 로봇의 상태뿐만 아니라 카메라 영상을 Observation으로 사용하는 경우가 많다.

카메라가 `/dev/video0`으로 인식되었다고 가정하면 다음과 같이 설정할 수 있다.

```bash
python -m lerobot.teleoperate \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=omx_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=omx_leader_arm \
    --display_data=true
```

`--display_data=true`를 사용하면 카메라 영상과 로봇 데이터를 확인하면서 Teleoperation을 수행할 수 있다.

---

# 7. Dataset Collection

Teleoperation이 정상적으로 작동한다면 이제 AI 학습에 사용할 Dataset을 수집한다.

Dataset에는 사람이 로봇을 조작하면서 수행한 작업의 Observation과 Action이 기록된다.

```text
Camera Image
     +
Robot State
     +
Action
     ↓
LeRobot Dataset
```

현재 LeRobot에서는 `lerobot-record`를 사용하여 데이터를 기록할 수 있다.

예시:

```bash
lerobot-record \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=omx_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=omx_leader_arm \
    --display_data=true \
    --dataset.repo_id=$HF_USER/omx_pick_and_place \
    --dataset.num_episodes=10 \
    --dataset.single_task="Pick up the cube and place it in the box"
```

### 주요 옵션

| 옵션                       | 설명              |
| ------------------------ | --------------- |
| `--robot.type`           | 사용할 Follower 로봇 |
| `--robot.port`           | Follower USB 포트 |
| `--robot.id`             | Follower ID     |
| `--robot.cameras`        | 카메라 설정          |
| `--teleop.type`          | Leader 타입       |
| `--teleop.port`          | Leader USB 포트   |
| `--teleop.id`            | Leader ID       |
| `--dataset.repo_id`      | Dataset 이름      |
| `--dataset.num_episodes` | 수집할 Episode 수   |
| `--dataset.single_task`  | 학습할 작업 설명       |

---

# 8. Hugging Face 로그인

Dataset이나 학습된 Policy를 Hugging Face Hub에 업로드하려면 로그인이 필요하다.

최신 CLI에서는 다음과 같이 로그인할 수 있다.

```bash
hf auth login
```

또는 환경에 따라 Token을 직접 지정할 수 있다.

```bash
hf auth login --token $HUGGINGFACE_TOKEN
```

로그인한 사용자 이름을 확인한다.

```bash
hf auth whoami
```

예:

```bash
export HF_USER=YOUR_HUGGINGFACE_USERNAME
```

---

# 9. Dataset 확인

수집된 Dataset은 LeRobot Dataset 형식으로 저장된다.

일반적인 데이터 구조는 다음과 같이 생각할 수 있다.

```text
Dataset
├── observation
│   ├── camera
│   └── robot_state
├── action
├── timestamp
└── episode
```

Dataset에는 여러 개의 Episode가 포함될 수 있다.

```text
Episode 0
Episode 1
Episode 2
Episode 3
...
```

여러 번 시연하여 다양한 상황의 데이터를 확보하는 것이 중요하다.

---

# 10. Policy Training

Dataset을 확보했다면 이제 AI Policy를 학습시킨다.

LeRobot에서는 여러 Policy를 사용할 수 있으며, 입문 과정에서는 **ACT(Action Chunking with Transformers)** Policy를 사용할 수 있다.

예시:

```bash
lerobot-train \
    --dataset.repo_id=$HF_USER/omx_pick_and_place \
    --policy.type=act \
    --policy.device=cuda \
    --output_dir=outputs/train/act_omx_pick_and_place \
    --job_name=act_omx_pick_and_place \
    --wandb.enable=true
```

또는 Python module 방식:

```bash
python -m lerobot.scripts.train \
    --dataset.repo_id=$HF_USER/omx_pick_and_place \
    --policy.type=act \
    --policy.device=cuda \
    --output_dir=outputs/train/act_omx_pick_and_place \
    --job_name=act_omx_pick_and_place \
    --wandb.enable=true
```

여기서 `cuda`는 NVIDIA GPU를 이용해 학습한다는 의미이다.

Apple Silicon에서는 환경에 따라 `mps`를 사용할 수 있다.

---

# 11. Training 결과

학습이 진행되면 지정한 Output Directory에 Checkpoint가 생성된다.

예:

```text
outputs/
└── train/
    └── act_omx_pick_and_place/
        └── checkpoints/
            ├── 010000/
            ├── 020000/
            └── last/
                └── pretrained_model/
```

가장 최근의 모델은 일반적으로 다음 위치에서 확인할 수 있다.

```text
outputs/train/act_omx_pick_and_place/checkpoints/last/pretrained_model
```

---

# 12. Training 재개

학습이 중단되었다면 마지막 Checkpoint부터 다시 학습할 수 있다.

```bash
lerobot-train \
    --config_path=outputs/train/act_omx_pick_and_place/checkpoints/last/pretrained_model/train_config.json \
    --resume=true
```

Python module 방식:

```bash
python -m lerobot.scripts.train \
    --config_path=outputs/train/act_omx_pick_and_place/checkpoints/last/pretrained_model/train_config.json \
    --resume=true
```

---

# 13. Policy 업로드

학습된 Policy를 Hugging Face Hub에 업로드할 수도 있다.

```bash
huggingface-cli upload \
    $HF_USER/omx_pick_and_place_act \
    outputs/train/act_omx_pick_and_place/checkpoints/last/pretrained_model
```

업로드하면 다른 환경에서도 해당 Policy를 사용할 수 있다.

ROBOTIS 공식 문서에서도 학습된 OMX Checkpoint를 Hugging Face Hub에 업로드하는 방법을 제공한다.

---

# 14. Inference / Rollout

학습이 완료되면 실제 OMX에서 Policy를 실행한다.

최신 LeRobot에서는 `lerobot-rollout` 명령을 사용한다.

```bash
lerobot-rollout \
    --strategy.type=base \
    --policy.path=$HF_USER/omx_pick_and_place_act \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --task="Pick up the cube and place it in the box" \
    --duration=60
```

여기서

```text
--policy.path
```

에는 학습된 Policy의 경로 또는 Hugging Face Hub의 Model ID를 지정한다.

`--duration=60`은 60초 동안 Rollout을 수행한다는 의미이다.

---

# 15. 전체 실행 과정

실제 프로젝트에서는 다음 순서로 진행할 수 있다.

## STEP 1 — LeRobot 설치

```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e .
```

## STEP 2 — 포트 확인

```bash
lerobot-find-port
```

예:

```text
Follower → /dev/ttyACM0
Leader   → /dev/ttyACM1
```

## STEP 3 — Teleoperation 테스트

```bash
lerobot-teleoperate \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --teleop.type=omx_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=omx_leader_arm
```

## STEP 4 — Dataset 수집

```bash
lerobot-record \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=omx_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=omx_leader_arm \
    --display_data=true \
    --dataset.repo_id=$HF_USER/omx_pick_and_place \
    --dataset.num_episodes=10 \
    --dataset.single_task="Pick up the cube and place it in the box"
```

## STEP 5 — Policy 학습

```bash
lerobot-train \
    --dataset.repo_id=$HF_USER/omx_pick_and_place \
    --policy.type=act \
    --policy.device=cuda \
    --output_dir=outputs/train/act_omx_pick_and_place \
    --job_name=act_omx_pick_and_place \
    --wandb.enable=true
```

## STEP 6 — 학습된 Policy 실행

```bash
lerobot-rollout \
    --strategy.type=base \
    --policy.path=$HF_USER/omx_pick_and_place_act \
    --robot.type=omx_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=omx_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --task="Pick up the cube and place it in the box" \
    --duration=60
```

---

# 16. Physical AI Pipeline

전체 시스템을 한눈에 보면 다음과 같다.

```text
             Human
               │
               ▼
        ┌─────────────┐
        │   OMX-L     │
        │   Leader    │
        └──────┬──────┘
               │
          Teleoperation
               │
               ▼
        ┌─────────────┐
        │   OMX-F     │
        │  Follower   │
        └──────┬──────┘
               │
               │ Observation
               ▼
        ┌─────────────┐
        │   Dataset   │
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │   Training  │
        │     ACT     │
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │ Trained     │
        │   Policy    │
        └──────┬──────┘
               │
            Rollout
               │
               ▼
        ┌─────────────┐
        │   OMX-F     │
        │ Autonomous  │
        └─────────────┘
```

---

# 17. 핵심 명령어 정리

| 목적                  | 명령어                   |
| ------------------- | --------------------- |
| 포트 찾기               | `lerobot-find-port`   |
| Teleoperation       | `lerobot-teleoperate` |
| Dataset 수집          | `lerobot-record`      |
| Policy 학습           | `lerobot-train`       |
| Policy 실행           | `lerobot-rollout`     |
| Hugging Face 로그인    | `hf auth login`       |
| Hugging Face 사용자 확인 | `hf auth whoami`      |

> **주의:** LeRobot은 버전에 따라 CLI 명령어와 옵션이 변경될 수 있다. 따라서 실제 실행 시에는 설치된 LeRobot 버전에 맞는 공식 문서를 확인해야 한다.

---

# 18. 핵심 개념

### Teleoperation

사람이 Leader 로봇을 조작하면 Follower 로봇이 그 움직임을 따라가는 과정.

### Dataset

사람이 수행한 작업을 AI가 학습할 수 있도록 기록한 데이터.

### Policy

현재 Observation을 입력으로 받아 로봇이 수행해야 할 Action을 결정하는 AI 모델.

### Training

수집된 Dataset을 이용하여 Policy가 작업 방법을 학습하는 과정.

### Rollout / Inference

학습이 끝난 Policy를 실제 로봇에 적용하여 자율적으로 작업을 수행하는 과정.

---

# 19. 최종 정리

OMX와 LeRobot을 이용한 Physical AI의 핵심 과정은 다음과 같다.

```text
사람이 작업을 시연
        ↓
OMX Teleoperation
        ↓
Dataset Collection
        ↓
AI Policy Training
        ↓
Trained Policy
        ↓
Inference / Rollout
        ↓
OMX가 작업을 자율적으로 수행
```

즉,

> **Human Demonstration → Dataset → Learning → Policy → Robot Action**

이라는 구조를 통해 사람이 직접 프로그래밍하지 않아도 로봇이 실제 작업을 수행하도록 학습시킬 수 있다.

이러한 과정이 Physical AI와 Imitation Learning의 기본적인 Workflow이다.

---

# 20. 참고 자료

* [ROBOTIS OMX Setup Guide](https://ai.robotis.com/omx/setup_guide_omx.html)
* [ROBOTIS OMX LeRobot Imitation Learning](https://ai.robotis.com/omx/lerobot_imitation_learning_omx.html)
* [Hugging Face LeRobot](https://huggingface.co/docs/lerobot/)
* [LeRobot GitHub](https://github.com/huggingface/lerobot)
* [LeRobot OMX Example](https://github.com/huggingface/lerobot/tree/main/examples/omx)
* [AI MANIPULATOR #6: Beginner's Guide to Physical AI with OMX & LeRobot](https://www.youtube.com/watch?v=uxiOghvNLTs)
