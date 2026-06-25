# 前置工作

## 安装相关包

先激活已经安装 Isaac Lab 的 conda 环境，然后安装本工程：

```bash
conda activate env_isaaclab
./unitree_rl_lab.sh -i
```

安装命令会执行：

- `pip install -e source/unitree_rl_lab/`
- 写入 conda activate 环境变量脚本
- 配置命令补全

安装完成后建议重新打开 shell，或者重新激活 conda 环境：

```bash
conda deactivate
conda activate env_isaaclab
```

## 定义机器人

机器人资产配置集中在：

```text
source/unitree_rl_lab/unitree_rl_lab/assets/robots/unitree.py
```

该文件中有两个核心路径：

```python
UNITREE_MODEL_DIR = "path/to/unitree_model"
UNITREE_ROS_DIR = "/root/unitree_rl_lab/unitree_ros/unitree_ros"
```

其中：

- `UNITREE_MODEL_DIR` 用于 USD 资产。
- `UNITREE_ROS_DIR` 用于 URDF 资产。

当前工程中的 G1 29dof velocity 任务已经配置为从 URDF 加载：

```python
UNITREE_G1_29DOF_CFG = UnitreeArticulationCfg(
    spawn=UnitreeUrdfFileCfg(
        asset_path=f"{UNITREE_ROS_DIR}/robots/g1_description/g1_29dof_rev_1_0.urdf",
    ),
    ...
)
```

因此运行 `Unitree-G1-29dof-Velocity` 时不应再去找 G1 的 USD 文件。如果后续重新拉取仓库或切换分支，请优先检查这里的 `spawn` 是否仍然是 `UnitreeUrdfFileCfg`。

# 任务相关

## 注册/开始任务

常用入口是：

```bash
./unitree_rl_lab.sh -t --task Unitree-G1-29dof-Velocity
```

等价于：

```bash
python scripts/rsl_rl/train.py --headless --task Unitree-G1-29dof-Velocity
```

---

``` shell
./unitree_rl_lab.sh -t \
  --task Unitree-G1-29dof-Velocity \
  --num_envs 64 \
  --max_iterations 50 \
  --run_name debug_g1
```

### 参数含义

| 参数                  | 含义                                     |
| --------------------- | ---------------------------------------- |
| `--task`              | 指定训练任务                             |
| `--num_envs 64`       | 覆盖并行仿真环境数量，调试时降低显存压力 |
| `--max_iterations 50` | 覆盖 PPO 最大训练迭代数，调试时快速结束  |
| `--run_name debug_g1` | 给日志目录增加后缀，方便区分实验         |

### 注册文件示例

G1 29dof velocity 任务任务注册位置：

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/robots/g1/29dof/__init__.py
```

核心注册信息：

```python
gym.register(
    id="Unitree-G1-29dof-Velocity",
    entry_point="isaaclab.envs:ManagerBasedRLEnv",
    kwargs={
        "env_cfg_entry_point": "...velocity_env_cfg:RobotEnvCfg",
        "play_env_cfg_entry_point": "...velocity_env_cfg:RobotPlayEnvCfg",
        "rsl_rl_cfg_entry_point": "unitree_rl_lab.tasks.locomotion.agents.rsl_rl_ppo_cfg:BasePPORunnerCfg",
    },
)
```

这表示：

- 训练时使用 `RobotEnvCfg`。
- 回放时使用 `RobotPlayEnvCfg`。
- PPO 使用 `BasePPORunnerCfg`。

## 查看注册的任务

``` shell
./unitree_rl_lab.sh -l
```

实际调用的`scripts/list_envs.py`。

## 回放指定任务

训练完成后可以回放策略：

``` shell
./unitree_rl_lab.sh -p \
  --task Unitree-G1-29dof-Velocity \
  --num_envs 32 \
  --real-time
```

实际调用`scripts/rsl_rl/play.py`。

---

默认情况下，`play.py` 会从对应实验目录中寻找 checkpoint。如果要指定 checkpoint：

```bash
./unitree_rl_lab.sh -p \
  --task Unitree-G1-29dof-Velocity \
  --checkpoint logs/rsl_rl/unitree_g1_29dof_velocity/<run_dir>/model_100.pt
```

回放脚本会做三件重要事情：

1. 加载环境配置和 PPO 配置。
2. 加载 checkpoint 并运行 policy。
3. 导出部署模型。

导出目录：

```text
logs/rsl_rl/unitree_g1_29dof_velocity/<run_dir>/exported/
```

导出文件：

| 文件 | 说明 |
| --- | --- |
| `policy.pt` | TorchScript/JIT 格式 |
| `policy.onnx` | ONNX 格式，常用于部署控制器 |

# 核心文件

## 机器人硬件配置：`unitree.py`

### 文件路径

```text
source/unitree_rl_lab/unitree_rl_lab/assets/robots/unitree.py
```

### 主要作用

主要负责：

- 机器人资产路径：URDF 或 USD。
- 初始 base 高度和关节角。
- actuator 类型、刚度、阻尼、力矩限制、速度限制。
- `joint_sdk_names`，即部署侧关节顺序。

G1 29dof velocity 任务使用：

```python
UNITREE_G1_29DOF_CFG
```

常见改动：

- 换 URDF 或 USD。
- 修改初始站姿。
- 修改某组关节的 stiffness、damping。
- 修改 effort limit 或 velocity limit。
- 修正部署关节顺序。

注意：`joint_sdk_names` 会参与生成 `deploy.yaml` 中的 `joint_ids_map`。如果部署端关节顺序不一致，policy 输出会错位。

## 环境配置：`velocity_env_cfg.py`

### 文件路径

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/robots/g1/29dof/velocity_env_cfg.py
```

### 主要作用

这是 G1 velocity 任务最核心的文件，包含：

| 配置类 | 作用 |
| --- | --- |
| `RobotSceneCfg` | 地形、机器人、传感器、灯光 |
| `EventCfg` | reset、随机化、外力扰动 |
| `CommandsCfg` | 目标速度命令 |
| `ActionsCfg` | policy 输出如何转成关节动作 |
| `ObservationsCfg` | policy 和 critic 的 observation |
| `RewardsCfg` | reward 项和权重 |
| `TerminationsCfg` | episode 终止条件 |
| `CurriculumCfg` | 课程学习 |
| `RobotEnvCfg` | 训练环境总配置 |
| `RobotPlayEnvCfg` | 回放环境配置 |

G1 velocity 环境配置位置：

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/robots/g1/29dof/velocity_env_cfg.py
```

默认训练环境：

```python
scene: RobotSceneCfg = RobotSceneCfg(num_envs=4096, env_spacing=2.5)
```

默认仿真和控制频率：

```python
self.decimation = 4
self.sim.dt = 0.005
```

因此 policy 控制周期为：

```text
step_dt = sim.dt * decimation = 0.005 * 4 = 0.02 s
```

也就是 50 Hz 控制频率。

## PPO 配置：`rsl_rl_ppo_cfg.py`

### 文件路径

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/agents/rsl_rl_ppo_cfg.py
```

### 主要作用

```python
num_steps_per_env = 24
max_iterations = 50000
save_interval = 100
actor_hidden_dims = [512, 256, 128]
critic_hidden_dims = [512, 256, 128]
learning_rate = 1.0e-3
entropy_coef = 0.01
gamma = 0.99
lam = 0.95
```

新手建议先不要大幅修改 PPO 超参数。多数行为问题优先从 reward、command range、action scale 和机器人 actuator 配置排查。

## MDP 函数目录

### 文件路径

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/mdp/
```

### 主要内容

| 文件 | 作用 |
| --- | --- |
| `rewards.py` | 自定义 reward 函数 |
| `observations.py` | 自定义 observation 函数 |
| `curriculums.py` | 课程学习函数 |
| `commands/velocity_command.py` | velocity command 配置扩展 |

`mdp/__init__.py` 会同时导入 Isaac Lab 自带 MDP 函数和本工程自定义函数，所以在环境配置中可以直接写：

```python
func=mdp.track_lin_vel_xy_yaw_frame_exp
func=mdp.energy
func=mdp.gait_phase
```

## 训练结果/日志

### 文件路径

```text
logs/rsl_rl/<experiment_name>/
```

对于 `Unitree-G1-29dof-Velocity`，默认实验名由 task 名自动生成：

```text
logs/rsl_rl/unitree_g1_29dof_velocity/
```

一次具体运行会生成时间戳目录，例如：

```text
logs/rsl_rl/unitree_g1_29dof_velocity/2026-06-25_12-00-00_debug_g1/
```

### 常见文件

| 文件                         | 说明                           |
| ---------------------------- | ------------------------------ |
| `model_*.pt`                 | RSL-RL 保存的 checkpoint       |
| `params/env.yaml`            | 本次训练实际使用的环境配置     |
| `params/agent.yaml`          | 本次训练实际使用的 PPO 配置    |
| `params/deploy.yaml`         | 部署所需的关节、动作、观测配置 |
| `params/velocity_env_cfg.py` | 本次训练复制保存的环境配置源码 |
| `videos/train/`              | 开启 `--video` 时保存训练视频  |

建议每次改动后先看：

```text
params/env.yaml
params/agent.yaml
```

它们记录的是最终生效配置，比只看源码更可靠。

##  `deploy.yaml` 

训练时会调用：

```text
source/unitree_rl_lab/unitree_rl_lab/utils/export_deploy_cfg.py
```

生成：

```text
params/deploy.yaml
```

该文件包含部署需要的信息，例如：

- `joint_ids_map`
- `step_dt`
- `stiffness`
- `damping`
- `default_joint_pos`
- `commands`
- `actions`
- `observations`

后续做 Sim2Sim 或 Sim2Real 时，控制器需要这些信息保证 policy 的输入输出和训练环境一致。

# 常见改动位置速查

## 换 URDF、USD 或机器人模型

修改：

```text
source/unitree_rl_lab/unitree_rl_lab/assets/robots/unitree.py
```

关注：

```python
UNITREE_ROS_DIR
UNITREE_MODEL_DIR
UNITREE_G1_29DOF_CFG.spawn
```

URDF 示例：

```python
spawn=UnitreeUrdfFileCfg(
    asset_path=f"{UNITREE_ROS_DIR}/robots/g1_description/g1_29dof_rev_1_0.urdf",
)
```

USD 示例：

```python
spawn=UnitreeUsdFileCfg(
    usd_path=f"{UNITREE_MODEL_DIR}/G1/29dof/usd/g1_29dof_rev_1_0/g1_29dof_rev_1_0.usd",
)
```

## 改初始姿态

修改：

```python
UNITREE_G1_29DOF_CFG.init_state
```

典型内容：

```python
init_state=ArticulationCfg.InitialStateCfg(
    pos=(0.0, 0.0, 0.8),
    joint_pos={
        "left_hip_pitch_joint": -0.1,
        "right_hip_pitch_joint": -0.1,
        ".*_knee_joint": 0.3,
    },
)
```

如果机器人一 reset 就倒、穿地或姿态异常，优先检查 base 高度 `pos` 和默认关节角。

## 改目标速度范围

修改：

```text
velocity_env_cfg.py
```

关注：

```python
class CommandsCfg:
    base_velocity = mdp.UniformLevelVelocityCommandCfg(...)
```

其中：

- `ranges` 是训练初始命令范围。
- `limit_ranges` 是课程学习最终允许扩展到的范围。

调试阶段可以先缩小速度范围，让机器人先学会稳定站立和慢速行走。

## 改 observation

修改：

```python
class ObservationsCfg
```

当前 policy observation 包括：

```text
base_ang_vel
projected_gravity
velocity_commands
joint_pos_rel
joint_vel_rel
last_action
```

如果要加入 gait phase，可以参考文件中已有注释：

```python
# gait_phase = ObsTerm(func=mdp.gait_phase, params={"period": 0.8})
```

对应函数在：

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/mdp/observations.py
```

注意：改变 policy observation 会改变神经网络输入维度，旧 checkpoint 通常不能直接继续训练或回放。

## 改 reward 权重

修改：

```python
class RewardsCfg
```

常见 reward：

| reward | 作用 |
| --- | --- |
| `track_lin_vel_xy` | 跟踪 x/y 线速度 |
| `track_ang_vel_z` | 跟踪 yaw 角速度 |
| `alive` | 存活奖励 |
| `base_height` | 维持机身高度 |
| `flat_orientation_l2` | 保持身体姿态水平 |
| `feet_slide` | 惩罚脚滑 |
| `feet_clearance` | 鼓励摆腿离地 |
| `undesired_contacts` | 惩罚非期望碰撞 |
| `energy` | 惩罚能耗 |
| `action_rate` | 惩罚动作变化过快 |

如果只是调参，通常只改 `weight` 或 `params`。如果要新增 reward 函数，步骤是：

1. 在 `tasks/locomotion/mdp/rewards.py` 中实现函数。
2. 在 `RewardsCfg` 中新增 `RewTerm`。
3. 短训练验证 reward 是否数值正常。
4. 查看训练曲线和回放行为。

## 改 action 幅度

修改：

```python
class ActionsCfg
```

当前动作配置：

```python
JointPositionAction = mdp.JointPositionActionCfg(
    asset_name="robot",
    joint_names=[".*"],
    scale=0.25,
    use_default_offset=True,
)
```

含义：

- policy 输出会缩放到关节位置目标。
- `scale=0.25` 控制动作幅度。
- `use_default_offset=True` 表示动作围绕默认关节姿态偏移。

如果机器人动作过大、抖动明显，可以先减小 `scale`。

## 改训练规模

临时改动优先使用 CLI：

```bash
./unitree_rl_lab.sh -t \
  --task Unitree-G1-29dof-Velocity \
  --num_envs 512 \
  --max_iterations 1000
```

长期默认值再改：

```python
class RobotEnvCfg:
    scene: RobotSceneCfg = RobotSceneCfg(num_envs=4096, env_spacing=2.5)
```

## 改 PPO 网络和学习率

修改：

```text
source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/agents/rsl_rl_ppo_cfg.py
```

常见调参项：

```python
actor_hidden_dims
critic_hidden_dims
learning_rate
entropy_coef
num_learning_epochs
num_mini_batches
max_iterations
save_interval
```

建议先保持默认。只有在 reward 和环境配置比较稳定后，再系统调整 PPO 超参数。

# 推荐学习路径

## 跑通最小训练

```bash
./unitree_rl_lab.sh -t \
  --task Unitree-G1-29dof-Velocity \
  --num_envs 64 \
  --max_iterations 50 \
  --run_name smoke_test
```

目标：

- 确认 Isaac Sim 可以启动。
- 确认 G1 URDF 可以加载。
- 确认训练日志可以生成。
- 确认 checkpoint 可以保存。

## 回放 checkpoint

```bash
./unitree_rl_lab.sh -p \
  --task Unitree-G1-29dof-Velocity \
  --num_envs 32 \
  --real-time
```

目标：

- 确认 policy 可以加载。
- 确认环境可以正常 step。
- 确认 `policy.pt` 和 `policy.onnx` 可以导出。

## 读一次日志配置

查看：

```text
logs/rsl_rl/unitree_g1_29dof_velocity/<run_dir>/params/env.yaml
logs/rsl_rl/unitree_g1_29dof_velocity/<run_dir>/params/agent.yaml
logs/rsl_rl/unitree_g1_29dof_velocity/<run_dir>/params/deploy.yaml
```

目标：

- 理解训练最终使用了哪些参数。
- 确认 CLI 覆盖是否生效。
- 理解部署文件和训练配置之间的关系。

## 只改一个变量做对比实验

建议从以下改动中选一个：

- 缩小或扩大 `CommandsCfg.base_velocity.limit_ranges`。
- 降低 `ActionsCfg.JointPositionAction.scale`。
- 调整 `RewardsCfg.base_height.weight`。
- 调整 `RewardsCfg.feet_slide.weight`。

每次只改一个点，重新跑短训练并回放观察。

## 再做系统性改动

当你已经理解训练链路后，再考虑：

- 新增 observation。
- 新增 reward。
- 修改 actuator 参数。
- 新建一个任务注册名。
- 增加 mimic 任务训练流程。
- 对接 Sim2Sim 或 Sim2Real。

# 常见问题

## 运行时提示找不到 USD 文件，但我想用 URDF

检查：

```text
source/unitree_rl_lab/unitree_rl_lab/assets/robots/unitree.py
```

确认 `UNITREE_G1_29DOF_CFG.spawn` 是：

```python
spawn=UnitreeUrdfFileCfg(...)
```

而不是：

```python
spawn=UnitreeUsdFileCfg(...)
```

同时确认：

```python
UNITREE_ROS_DIR
```

指向真实存在的 `unitree_ros/unitree_ros` 目录。

## 修改 observation 后无法加载旧 checkpoint

这是正常现象。policy 输入维度变化后，旧模型参数形状不匹配。

处理方式：

- 从头训练新 policy。
- 或者恢复到旧 observation 配置再加载旧 checkpoint。

## 机器人 reset 后很快倒下

优先检查：

- URDF/USD 是否加载正确。
- `init_state.pos` 高度是否合理。
- 默认关节角是否合理。
- actuator stiffness/damping 是否过弱或过强。
- `ActionsCfg.scale` 是否过大。
- `TerminationsCfg.bad_orientation` 是否过于严格。

## 训练很慢或显存不够

调低并行环境数量：

```bash
./unitree_rl_lab.sh -t \
  --task Unitree-G1-29dof-Velocity \
  --num_envs 128 \
  --max_iterations 100
```

也可以先使用 `--max_iterations` 做短实验，确认配置方向正确后再放大训练。

## 回放时找不到 checkpoint

可以显式指定 checkpoint：

```bash
./unitree_rl_lab.sh -p \
  --task Unitree-G1-29dof-Velocity \
  --checkpoint logs/rsl_rl/unitree_g1_29dof_velocity/<run_dir>/model_100.pt
```

如果目录不存在，先确认训练是否正常结束，以及日志是否写入：

```text
logs/rsl_rl/unitree_g1_29dof_velocity/
```

##  什么时候改源码，什么时候用 CLI

优先用 CLI 的场景：

- 临时改 `num_envs`
- 临时改 `max_iterations`
- 临时指定 checkpoint
- 临时指定 run name

需要改源码的场景：

- 改机器人资产。
- 改 reward、observation、action。
- 改 command range 默认值。
- 改 PPO 默认配置。
- 新增任务或新增机器人。

# 附录：核心文件速查

| 文件 | 用途 |
| --- | --- |
| `README.md` | 项目安装和基本运行说明 |
| `unitree_rl_lab.sh` | 安装、任务列表、训练、回放的命令包装入口 |
| `scripts/list_envs.py` | 列出已注册 Unitree 任务 |
| `scripts/rsl_rl/train.py` | RSL-RL PPO 训练主脚本 |
| `scripts/rsl_rl/play.py` | checkpoint 回放和 policy 导出脚本 |
| `source/unitree_rl_lab/unitree_rl_lab/assets/robots/unitree.py` | Unitree 机器人资产、初始状态、actuator、关节映射 |
| `source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/robots/g1/29dof/__init__.py` | G1 29dof velocity 任务注册 |
| `source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/robots/g1/29dof/velocity_env_cfg.py` | G1 velocity 任务环境配置 |
| `source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/agents/rsl_rl_ppo_cfg.py` | PPO 默认超参数 |
| `source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/mdp/rewards.py` | 自定义 reward 函数 |
| `source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/mdp/observations.py` | 自定义 observation 函数 |
| `source/unitree_rl_lab/unitree_rl_lab/utils/export_deploy_cfg.py` | 生成部署配置 `deploy.yaml` |
