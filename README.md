# Reinforcement Learning for Quadrupedal Locomotion on Challenging Terrains

This repository contains the training configuration and results for training the **ANYmal C** quadrupedal robot to achieve robust, high-speed locomotion on rough terrains using **Proximal Policy Optimization (PPO)** in **NVIDIA Isaac Lab**.

---

## 🛠️ Key Configuration Files (Evidence of Work)

The core reinforcement learning configurations and environment setups are defined in the following files:

1. **[rsl_rl_ppo_cfg.py](file:///c:/Users/yokko/OneDrive/Desktop/CodeProgr/Python/robot_rl/IsaacLab/source/isaaclab_tasks/isaaclab_tasks/manager_based/locomotion/velocity/config/anymal_c/agents/rsl_rl_ppo_cfg.py)**
   * **RL Agent/PPO Hyperparameters:** Configures the policy architecture, value network, and optimizer.
   * **Modification:** Changed the PPO learning rate schedule from `"adaptive"` to `"fixed"` (locked at `1.0e-3`) to resolve a critical learning rate trap caused by the adaptive KL scheduler under high exploration noise.

2. **[velocity_env_cfg.py](file:///c:/Users/yokko/OneDrive/Desktop/CodeProgr/Python/robot_rl/IsaacLab/source/isaaclab_tasks/isaaclab_tasks/manager_based/locomotion/velocity/velocity_env_cfg.py)**
   * **MDP Environment Setup:** Defines observation spaces, target command ranges, and reward functions.
   * **Modification:** Expanded target speed ranges from $\pm 1.0\text{ m/s}$ to $\pm 1.5\text{ m/s}$ and scaled up the linear velocity tracking reward weight to $1.5$ to incentivize high-speed locomotion.

---

## 📝 Project Essay & Concluding Findings

### 1. Introduction and Objectives
The goal of this project was to train the ANYmal C quadruped to walk robustly on rough terrains (slopes, stairs, and rocky grids) while maximizing locomotion velocity. Quadrupedal locomotion is a highly non-linear, high-dimensional control problem. Hand-crafting gaits is difficult; reinforcement learning provides an elegant solution by allowing policies to emerge naturally from reward functions. Our objective was to transition the robot from flat ground balance to high-speed locomotion ($\pm 1.5 \text{ m/s}$) over complex, unstructured heightfields.

### 2. Diagnosing the Learning Rate Trap
Early training runs failed to converge. The robots flailed their legs and fell immediately ("eating dirt"), with rewards stagnant near zero and the curriculum locked at Level 0.0 (flat ground). 

A diagnostic review of the TensorBoard event logs revealed that the PPO agent's learning rate had collapsed from its initial value of $1.0\times 10^{-3}$ to a floor of $1.0\times 10^{-5}$ within the first few iterations. The root cause was identified as the **adaptive KL divergence scheduler**. 

In legged robotics, high-exploration noise and impacts with the ground cause sudden, large variations in action distributions. The adaptive scheduler interpreted these spikes as excessive policy drift and updated the learning rate downward *inside the mini-batch loop* up to 20 times per iteration. This permanently trapped the learning rate at $1.0\times 10^{-5}$, halting policy updates before the agent could learn to support its own mass.

To resolve this optimization bottleneck, the schedule was changed from `"adaptive"` to `"fixed"`, locking the learning rate to $1.0\times 10^{-3}$ (the industry standard for PPO locomotion). This simple architectural adjustment restored optimization dynamics, allowing the neural network weights to update 100 times faster per iteration.

### 3. Curriculum Progression and Speed Tuning
With the optimization path clear, we trained two distinct policies:

* **Policy A: Stable Locomotion ($\pm 1.0 \text{ m/s}$)**
  * *Parameters:* Command limits set to $\pm 1.0 \text{ m/s}$ in both $x$ and $y$ axes; tracking reward weight set to $1.0$.
  * *Results:* The robot successfully discovered a stable, rhythmic trotting gait. It climbed the terrain curriculum steadily over 465 iterations, reaching an average **Terrain Level of 4.80** (representing steep slopes and moderate steps) with a **$80.8\%$ survival rate** (timeout percentage) and a **$99.5\%$ step success rate**.
* **Policy B: High-Speed Locomotion ($\pm 1.5 \text{ m/s}$)**
  * *Parameters:* Command limits increased to $\pm 1.5 \text{ m/s}$ ($50\%$ increase); velocity tracking reward weight increased to $1.5$ to incentivize high-velocity tracking over joint torque and acceleration penalties.
  * *Results:* Due to the larger gradient signals (higher tracking weight) and standard fixed learning rate, Policy B converged rapidly in early iterations, traversing flat ground and early slopes much faster than Policy A. In just **36 minutes of headless training**, it surpassed the previous run, achieving an average **Terrain Level of 5.09** at iteration 442.

### 4. The Curriculum Difficulty Wall and Plateaus
As Policy B progressed past iteration 500, the curriculum average flattened and hovered between **Level 5.38 and 5.42**, with the mean reward stabilizing around **`9.5 - 11.0`**. 

This plateau represents a physical constraint in the robot's current MDP configuration:
1. **Sideways Crab-Walking Instability:** Because the policy prefers crab-walking diagonally to follow direction command arrows rather than turning its body (due to high yaw effort penalties), moving at $1.5 \text{ m/s}$ sideways on uneven steps causes the robot's legs to collide with the vertical edges of steps. At high speeds, these lateral collisions create high-torque trips that cause immediate base contact terminations.
2. **Motor Saturation:** Climbing steps at $1.5 \text{ m/s}$ requires joint velocities and torque values that saturate the actuator limits of the ANYmal C model. Lacking the necessary joint-level torque overhead, the robot cannot recover when it slips.

The curriculum engine handled this wall gracefully: when robots tripped and fell on Level 5 steps, they were automatically demoted back to Level 4 to practice and stabilize. This curriculum loop created a stable, self-regulating oscillation, protecting the neural network from divergence.

### 5. Summary of Achievements and Next Steps
Through hyperparameter debugging and reward engineering, we successfully trained a quadruped policy that operates at high velocities ($1.5 \text{ m/s}$) on highly challenging rough terrains (Level 5.4, including staircases and hills) using only 45 minutes of compute time.

To push past the current physical limits, future work will focus on:
* **Heading Alignment:** Introducing a heading reward term to force the robot to face the direction of its commanded velocity vector, eliminating the instability of high-speed lateral crab-walking.
* **Joint Penalty Tuning:** Relaxing joint acceleration and torque penalties at higher speeds to allow the policy to utilize the full range of the motor capacities.

---

## 🚀 Execution & Visualization Guide

### Prerequisites
Activate your Conda environment and navigate to the project directory:
```powershell
cd IsaacLab
conda activate env_isaaclab
```

### 1. Training a Policy
To train a policy from scratch using the current configurations:
```powershell
python scripts/reinforcement_learning/rsl_rl/train.py --task Isaac-Velocity-Rough-Anymal-C-v0
```

### 2. Playing Back checkpoints (Visualization)
To visualize a trained run in the 3D Omniverse Kit viewer:
```powershell
python scripts/reinforcement_learning/rsl_rl/play.py --task Isaac-Velocity-Rough-Anymal-C-v0 --load_run <RUN_DIR_NAME> --num_envs 32 --viz kit
```
*(Replace `<RUN_DIR_NAME>` with the specific run folder, e.g. `2026-06-30_09-02-36`)*.
