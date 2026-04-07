# AutoVLA: A Vision-Language-Action Model for End-to-End Autonomous Driving with Adaptive Reasoning and Reinforcement Fine-Tuning

**Authors:** Zewei Zhou, Tianhui Cai, Seth Z. Zhao, Yun Zhang, Zhiyu Huang, Bolei Zhou, Jiaqi Ma

**Venue:** NeurIPS 2025

**arXiv:** [2506.13757](https://arxiv.org/abs/2506.13757)

**Code:** [github.com/ucla-mobility/AutoVLA](https://github.com/ucla-mobility/AutoVLA)

---

## Goal / Problem

End-to-end autonomous driving has increasingly leveraged Vision-Language-Action (VLA) models to benefit from pretrained world knowledge and chain-of-thought (CoT) reasoning. However, existing VLA approaches suffer from several key shortcomings: they often produce physically infeasible actions due to continuous action spaces misaligned with language model outputs, impose unnecessarily complex multi-module architectures, or apply slow CoT reasoning indiscriminately even in trivial scenarios where it adds latency without benefit. AutoVLA addresses these gaps by unifying reasoning and action generation in a single autoregressive model with discrete feasible action tokens, and by training the model to adaptively switch between fast and slow thinking based on scenario complexity.

---

## Main Methods

AutoVLA builds on a pretrained Vision-Language Model (Qwen2.5-VL-3B) and extends it with trajectory tokenization and a two-stage fine-tuning strategy — Supervised Fine-Tuning (SFT) followed by Reinforcement Fine-Tuning (RFT) — enabling end-to-end planning directly from raw visual observations and natural language navigation commands.

### 1. Physical Action Tokenization

Continuous vehicle trajectories are discretized into sequences of discrete, feasible action tokens. Each token represents a short-term movement $(Δx, Δy, Δθ)$ over a 0.5-second interval, capturing both spatial displacement and heading change. An action codebook of $K = 2048$ tokens is constructed using K-Disk clustering applied jointly to real-world (Waymo Open Motion Dataset / WOMD) and simulation (CARLA-Garage) motion data:

$$\mathcal{C} = \{c_1, c_2, \dots, c_K\}, \quad c_k \in \mathbb{R}^3, \quad K = 2048$$

A full planning horizon trajectory is represented as a sequence of such tokens, which are then appended to the language model's vocabulary for direct autoregressive generation. This ensures that all generated actions are physically feasible by construction (drawn from observed real and simulated driving motions), avoiding infeasible outputs that plague regression-based VLA methods.

**What is an Action Codebook?**
An action codebook is a fixed, finite vocabulary of motion primitives — analogous to a word dictionary, but for vehicle movements instead of words. Each entry is a short-horizon motion vector $(Δx, Δy, Δθ)$ representing 0.5 seconds of driving. With $K = 2048$ entries, the codebook covers the full range of observed driving behaviors (gentle curves, sharp turns, acceleration, braking, etc.). The key motivation: language models work natively over discrete tokens, not continuous real-valued outputs. By mapping every trajectory segment to the nearest codebook token, the planning problem becomes **next-token prediction** — the same operation LLMs perform for text. This also guarantees physical feasibility by construction, since every token was derived from real or simulated driving data.

**What is K-Disk Clustering?**
K-Disk clustering is a variant of K-Means adapted for motion data. Standard K-Means represents each cluster by a point centroid. K-Disk instead represents each cluster as a **disk** (a center + radius), so any motion primitive falling within the disk radius maps to the same token. This has two advantages over vanilla K-Means: (1) it tolerates small deviations in motion — reducing over-sensitivity to sensor noise — and (2) it distributes coverage more uniformly across the motion space, preventing over-clustering in dense regions. The clustering is run jointly on WOMD (real-world) and CARLA-Garage (simulation) motion segments, so the 2048 resulting tokens cover both real-world urban/highway driving styles and the broader behavioral diversity that simulation provides.

**Why auto-regressive generation is necessary:** With 2048 tokens per step and a 0.5s interval, an 8-second trajectory spans 16 steps, giving $2048^{16} = 2^{176} \approx 10^{53}$ possible sequences — exhaustive search is completely intractable. Instead, the model generates one token at a time conditioned on all prior tokens:

$$p(\tau) = \prod_{t=1}^{16} p(a_t \mid a_{\lt t}, \text{scene})$$

This requires only 16 forward passes, each scoring 2048 candidates. The tradeoff is that auto-regressive decoding finds a good sequence rather than the globally optimal one; beam search can partially mitigate this by maintaining the top-$B$ partial sequences at each step.

### 2. Dual Thinking Modes via Supervised Fine-Tuning (SFT)

SFT trains the model jointly on two types of outputs:

- **Fast thinking (action-only):** The model directly outputs the action token sequence without any intermediate reasoning. Used for straightforward scenarios.
- **Slow thinking (CoT + action):** The model first generates a chain-of-thought reasoning trace — describing scene context, intent, and decision rationale — before outputting the action tokens. High-quality CoT annotations are distilled from a large-scale VLM.

Both modes are represented within the same autoregressive generation framework, allowing the model to seamlessly switch between them at inference time.

### 3. Reinforcement Fine-Tuning (RFT) with Adaptive Reasoning

After SFT, RFT is applied using **Group Relative Policy Optimization (GRPO)** to further optimize the policy with task-specific reward functions. GRPO updates the policy relative to a group of sampled trajectories:

$$\mathcal{L}_{\text{GRPO}} = -\mathbb{E}\left[\sum_{t} \min\left(r_t \hat{A}_t, \text{clip}(r_t, 1-\epsilon, 1+\epsilon)\hat{A}_t\right)\right]$$

where $r_t$ is the probability ratio and $\hat{A}_t$ is the advantage estimated from group-relative rewards.

RFT's key contribution is **adaptive reasoning**: it penalizes unnecessary CoT generation in simple, unambiguous scenarios (where fast thinking suffices) while retaining slow thinking for genuinely complex situations. This reduces inference-time reasoning overhead by 66.8% (measured over 500 samples) without sacrificing planning quality.

### Additional Innovations

- **Unified autoregressive framework:** Both semantic reasoning and trajectory prediction are cast as next-token prediction within the same LM, eliminating separate planning heads or post-processing modules.
- **Action codebook construction from diverse domains:** Clustering across both real-world (WOMD) and simulated (CARLA-Garage) motion ensures the codebook covers a broad range of driving behaviors, improving generalization.
- **CoT data distillation:** Reasoning annotations are bootstrapped from a stronger large-scale VLM rather than human labeling, enabling scalable collection of high-quality training data.

---

## Datasets & Simulation Environments

### nuPlan
Large-scale real-world driving dataset from Motional, containing over 1,300 hours of expert driving across multiple cities. Used for open-loop and closed-loop planning evaluation with the nuPlan benchmark (PDMS metric). AutoVLA experiments show that CoT reasoning scales better than action-only training: with more than 50k training samples, CoT-augmented models consistently outperform action-only models in PDMS and Collision Score.

### nuScenes
Real-world dataset with 1,000 driving scenes (700 train / 150 val / 150 test), multi-camera video, LiDAR, and HD maps. Used for open-loop trajectory prediction evaluation. Metrics include L2 distance (positional error) and collision rate. CoT reasoning training shows consistent improvement over action-only training as training set size increases.

### Waymo Open Motion Dataset (WOMD)
Large-scale real-world driving dataset from Waymo, used both as a source of real-world motion primitives for action codebook construction and as a benchmark. AutoVLA participates in the **Waymo Vision-based End-to-End Driving Challenge**, achieving top scores on the RFS Spotlight metric (most challenging scenarios) and competitive rankings on RFS Overall and ADE.

### CARLA / CARLA-Garage (Closed-Loop Simulation)
Open-source urban driving simulator used for closed-loop evaluation. CARLA-Garage provides a curated set of scenarios and motion data used for codebook construction. AutoVLA outperforms existing end-to-end driving models on overall driving score and success rate in the closed-loop CARLA test, demonstrating generalization from real-world pretraining to simulation.

### NAVSIM
A non-reactive, vision-based driving benchmark derived from nuPlan data that evaluates open-loop planning using the **PDMS** (nuPlan Driving Motion Score) metric. Used to measure the impact of RFT: AutoVLA achieves a **10.6% improvement in PDMS** on the NAVSIM testing set after RFT.

---

## Metrics & Benchmarks

**Primary Benchmarks:** nuPlan (closed-loop), nuScenes (open-loop), Waymo E2E Challenge, CARLA (closed-loop), NAVSIM (open-loop).

| Metric | Benchmark | Description | AutoVLA Result |
|--------|-----------|-------------|----------------|
| PDMS | nuPlan / NAVSIM | nuPlan Driving Motion Score; composite closed-loop quality metric | +10.6% improvement via RFT |
| L2 Distance | nuScenes | Average positional error of predicted trajectory vs. ground truth | Improved with CoT + more data |
| Collision Rate | nuScenes / nuPlan | Rate of collisions in predicted/executed trajectories | Reduced with CoT + more data |
| RFS Spotlight | Waymo E2E Challenge | Reaction-based score on hardest scenarios | **Top score** (1st place, as of May 2025) |
| RFS Overall | Waymo E2E Challenge | Overall reaction-based score | Top-tier ranking |
| ADE | Waymo E2E Challenge | Average Displacement Error | Top-tier ranking |
| Driving Score | CARLA | Composite score of route completion, safety, comfort | Outperforms prior E2E models |
| Runtime | NAVSIM | Inference time per sample (500 samples) | **66.8% reduction** via RFT |

> **Key result:** AutoVLA achieves the top RFS Spotlight score on the Waymo Vision-based End-to-End Driving Challenge and delivers a 10.6% PDMS improvement on NAVSIM with RFT, while simultaneously reducing reasoning runtime by 66.8% through adaptive thinking.

---

## Limitations

- **Small backbone (3B parameters):** AutoVLA uses Qwen2.5-VL-3B for deployment efficiency, but larger VLM backbones may offer significantly better reasoning and generalization that are not explored.
- **Non-reactive simulation gap:** NAVSIM and nuPlan open-loop evaluations do not involve reactive agents; real closed-loop interaction with dynamic actors may expose additional failure modes.
- **Discrete codebook coverage:** The $K = 2048$ action codebook may not capture rare or highly complex maneuvers (e.g., emergency evasion), as motion diversity is bounded by the clustering resolution.
- **CoT quality dependence:** Slow-thinking performance relies on distilled CoT annotations from a larger VLM; errors or biases in the teacher model propagate to the student.
- **Sim-to-real gap not fully addressed:** While the codebook is built from both real (WOMD) and simulated (CARLA-Garage) data, no explicit domain adaptation is applied for deployment on real vehicles.
- **Evaluation primarily on established benchmarks:** Generalization to diverse geographic regions, weather conditions, or edge cases outside nuPlan/Waymo/CARLA distributions is not demonstrated.
- **Code/weights release pending:** As of the paper's NeurIPS 2025 submission, the code, model weights, and datasets were announced to be released but may not yet be fully available.

---

## Sources

- [AutoVLA arXiv](https://arxiv.org/abs/2506.13757)
- [AutoVLA NeurIPS 2025 Poster](https://neurips.cc/virtual/2025/poster/120167)
- [AutoVLA GitHub Repository](https://github.com/ucla-mobility/AutoVLA)
- [AutoVLA Project Page](https://autovla.github.io/)
- [nuPlan Dataset](https://nuplan.org/)
- [nuScenes Dataset](https://nuscenes.org/)
- [Waymo Open Motion Dataset](https://waymo.com/open/data/motion/)
- [NAVSIM Benchmark](https://github.com/autonomousvision/navsim)
