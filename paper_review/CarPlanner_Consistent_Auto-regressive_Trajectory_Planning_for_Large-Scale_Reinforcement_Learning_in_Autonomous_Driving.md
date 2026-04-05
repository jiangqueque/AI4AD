# CarPlanner: Consistent Auto-regressive Trajectory Planning for Large-Scale Reinforcement Learning in Autonomous Driving

**Venue:** CVPR 2025
**arXiv:** [2502.19908](https://arxiv.org/abs/2502.19908)

---

## Goal / Problem

The paper targets **closed-loop trajectory planning** for autonomous driving. While reinforcement learning (RL) is theoretically well-suited for planning — avoiding issues like distribution shift and causal confusion that plague imitation learning (IL) — RL planners had never surpassed IL- or rule-based state-of-the-art on large-scale real-world datasets. The core bottleneck is **training instability and inefficiency** at scale, particularly because naive auto-regressive RL policies suffer from temporal inconsistency in mode selection (the vehicle's behavioral intent keeps shifting across time steps), making the policy hard to learn.

---

## Main Methods

CarPlanner is a **generation-selection framework** built around four components:

### 1. Non-Reactive Transition Model
Predicts future trajectories of surrounding agents given the initial scene state, without modeling ego-agent interactions. This simplifies the environment model for RL training.

### 2. Mode Selector
Outputs a soft score over a discrete set of lateral driving modes (e.g., lane-keep, lane-change-left, lane-change-right) conditioned on the current state. This is the RL policy's primary output.

### 3. Auto-Regressive Trajectory Generator
Conditioned on a *consistent* selected mode, generates multi-modal trajectory proposals in an auto-regressive manner. The key insight is enforcing **mode consistency** across time steps — once a mode is selected, subsequent trajectory generation stays aligned with it, stabilizing the RL training signal.

**Auto-regressive generation** means the trajectory is produced **one waypoint at a time**, where each new point is conditioned on all previously generated points. Formally, the full trajectory $\tau = (w_1, w_2, \ldots, w_T)$ is factored as:

$$p(\tau \mid s, m) = \prod_{t=1}^{T} p(w_t \mid w_{\lt t}, s, m)$$

where $s$ is the scene state and $m$ is the selected driving mode. At each step, the model attends to all prior waypoints and predicts the distribution over the next one — similar in spirit to how a language model generates tokens sequentially. This structure lets the generator produce **diverse, long-horizon trajectories** while ensuring local geometric coherence (smooth transitions between consecutive waypoints). The conditioning on a fixed mode $m$ throughout the entire rollout is what enforces **temporal consistency**: the trajectory cannot abruptly "change intent" mid-sequence, a failure mode that plagued earlier auto-regressive planners trained with RL.

### 4. Rule-Augmented Selector
Post-processes mode scores by incorporating rule-based metrics (safety, comfort, progress) to select the final trajectory. This hybrid approach grounds the RL policy with interpretable safety constraints.

### Additional Innovations

- **Invariant-View Module (IVM):** Transforms scene inputs (agents, map, route) into the ego vehicle's current coordinate frame and clips distant information, providing a time-agnostic, consistent observation space for the policy and improving generalization.
- **Expert-Guided Reward Function:** Shapes the RL reward using demonstrations from an expert planner, easing reward sparsity and accelerating policy learning.

---

## Datasets & Simulation Environments

### nuPlan Dataset
The paper exclusively uses the **[nuPlan](https://www.nuplan.org/)** dataset and its associated closed-loop simulator.

- **Scale:** ~1,300 hours of real-world driving logs collected across 4 cities (Boston, Pittsburgh, Singapore, Las Vegas), covering a wide range of traffic densities and road topologies.
- **Data format:** Each scenario provides HD map, ego vehicle state, and tracked surrounding agent states at 10 Hz.
- **Scenarios:** nuPlan defines a large set of named reactive and non-reactive scenario types (e.g., lane changes, unprotected turns, stop-sign interactions) used for both training and evaluation.

### nuPlan Closed-Loop Simulator
nuPlan's simulator replays logged scenarios and evaluates the ego planner in closed loop — the ego's actions affect its own future state, but (in non-reactive mode) surrounding agents follow their logged trajectories.

- **Non-Reactive (NR) mode:** Other agents replay pre-recorded paths regardless of the ego. CarPlanner is primarily evaluated here (CLS-NR metric).
- **Reactive (R) mode:** Other agents respond to the ego using a built-in reactive agent model. CarPlanner's evaluation in this mode is less emphasized.
- **Split used:** The paper follows the standard nuPlan benchmark train/val/test splits, evaluating on the official test set of 1,000+ scenarios.

> No additional simulation environments (e.g., CARLA, MetaDrive) or datasets (e.g., Waymo Open Dataset, WOMD) are used.

---

## Metrics & Benchmarks

**Benchmark:** [nuPlan](https://www.nuplan.org/) — large-scale real-world autonomous driving planning benchmark with closed-loop simulation.

| Metric | Description | CarPlanner | vs. PDM-Closed | vs. PLUTO |
|--------|-------------|------------|----------------|-----------|
| CLS-NR | Closed-Loop Score (Non-Reactive) | **94.07** | +4.02 | +2.15 |
| S-PR | Progress Score | Substantially higher | — | — |
| S-CR | Collision Rate | Comparable | — | — |

**Baselines compared:** PDM-Open, PDM-Closed (nuPlan Challenge 2023 winner), PLUTO, and other RL- and IL-based planners.

> CarPlanner is the **first RL-based planner to surpass both IL- and rule-based SOTAs** on the nuPlan benchmark, without relying on common IL techniques such as data augmentation or ego-history masking.

---

## Limitations

- Evaluated primarily in **non-reactive simulation** on nuPlan; performance in fully reactive multi-agent environments (where other agents respond to the ego) is less thoroughly demonstrated.
- The **non-reactive transition model** simplifies agent behavior prediction, which may be insufficient in highly interactive scenarios (e.g., merges, adversarial agents).
- The framework relies on a **discrete mode space** for lateral behavior, which may not capture all nuances of complex maneuvers.
- Like most simulation-based RL approaches, there remains a **sim-to-real gap** — whether the policy transfers robustly to real vehicle deployment is not addressed.

---

## Sources

- [CarPlanner arXiv (2502.19908)](https://arxiv.org/abs/2502.19908)
- [CVPR 2025 Open Access](https://openaccess.thecvf.com/content/CVPR2025/html/Zhang_CarPlanner_Consistent_Auto-regressive_Trajectory_Planning_for_Large-Scale_Reinforcement_Learning_in_CVPR_2025_paper.html)
- [nuPlan Benchmark](https://www.nuplan.org/)
