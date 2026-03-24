# SkillOrchestra 深度研究报告

**论文**: SkillOrchestra: Learning to Route Agents via Skill Transfer
**作者**: Jiayu Wang et al. (UW-Madison / Salesforce AI Research)
**ArXiv**: 2602.19672 (2026)
**代码**: https://github.com/jiayuww/SkillOrchestra

---

## 一、代码运行验证

### 1.1 核心模块运行结果

成功运行 SkillOrchestra 核心模块，验证了以下功能：

| 模块 | 测试内容 | 结果 |
|------|---------|------|
| `SkillHandbook` | 创建、保存、加载、完整性校验 | ✅ 正常 |
| `BetaCompetence` | Beta 分布能力建模 | ✅ 正常 |
| `select_agent()` | 多层级 tie-breaking 路由 | ✅ Math→Claude, Creative→GPT4 |
| `_identify_skills_by_indicators()` | 关键词匹配技能识别 | ✅ 正确匹配 |
| `generate_depth_candidates()` | 粒度感知候选生成 | ✅ 生成 2 个候选 |
| `Serialization` | JSON 序列化/反序列化 | ✅ 无完整性问题 |

### 1.2 路由逻辑验证

```
Math-heavy query (math:0.8, factual:0.1, creative:0.1) → Selected: claude (α=9, β=1)
Factual-heavy query (math:0.1, factual:0.8, creative:0.1) → Selected: claude (α=8, β=2)
Creative-heavy query (math:0.1, factual:0.1, creative:0.8) → Selected: gpt4 (α=9, β=1)
```

路由逻辑正确：根据技能权重加权 Beta competence mean 选择最优 agent。

---

## 二、代码架构深度解读

### 2.1 整体架构

```
SkillOrchestra/
├── skillorchestra/           # 核心库
│   ├── core/                 # 数据结构层
│   │   ├── types.py          # Skill, AgentProfile, BetaCompetence, CostStats
│   │   ├── handbook.py       # SkillHandbook: 注册表+路由+序列化
│   │   └── traces.py         # ExecutionTrace, ExplorationBundle
│   ├── learning/             # 学习层（三阶段）
│   │   ├── pipeline.py       # 全流程编排器
│   │   ├── discoverer.py     # Phase 1a: 技能发现
│   │   ├── profiler.py       # Phase 1b: Agent Profile 构建
│   │   ├── refiner.py        # Phase 2: 分裂/合并精炼
│   │   └── versioner.py      # 版本管理
│   ├── selection/            # 选择层
│   │   ├── candidates.py     # 粒度感知候选生成
│   │   └── pareto.py         # Pareto 最优选择
│   ├── routing/              # 部署层
│   │   └── orchestrator.py   # 运行时路由
│   ├── converters/           # 格式转换
│   └── prompts/              # LLM Prompt 模板
├── model_routing/            # 模型路由评测
├── orchestration/            # Agent 编排评测
└── scripts/pipeline.py       # 主入口
```

### 2.2 核心数据流

```
ExplorationBundle (多轨迹)
    │
    ▼
┌─────────────────┐
│ Phase 1a:       │  LLM + contrastive pairs → 分层技能分类法
│ SkillDiscoverer │  输出: Skill(skill_id, name, description, indicators, mode)
└────────┬────────┘
         ▼
┌─────────────────┐
│ Phase 1b:       │  LLM 技能识别 + Beta 更新
│ AgentProfiler   │  α += I[success], β += I[failure]
│                 │  输出: AgentProfile(skill_competence, cost_stats, strengths)
└────────┬────────┘
         ▼
┌─────────────────┐
│ Phase 2:        │  方差驱动分裂 + 性能驱动合并
│ HandbookRefiner │  LLM 审核决策 + competence 转移
└────────┬────────┘
         ▼
┌─────────────────┐
│ Granularity     │  Depth-based 候选生成
│ Selection       │  Oracle/Live Pareto 评估
└────────┬────────┘
         ▼
┌─────────────────┐
│ Deployment:     │  skill_weights → weighted_competence → 4层 tie-breaking
│ SkillOrchestrator│
└─────────────────┘
```

### 2.3 关键算法详解

#### A. Beta 分布能力建模

```python
class BetaCompetence:
    alpha: float = 1.0  # 先验 + 成功次数
    beta: float = 1.0   # 先验 + 失败次数

    @property
    def mean(self):     return self.alpha / (self.alpha + self.beta)
    @property
    def variance(self): return (self.alpha * self.beta) / ((self.alpha + self.beta)**2 * (self.alpha + self.beta + 1))
```

以 Beta(1,1) 为 uninformative prior，每次观测更新：
- 成功: α += 1
- 失败: β += 1

**设计优势**: 贝叶斯估计天然处理小样本不确定性，prior 避免了零观测问题。

#### B. 四层 Agent 选择机制

```python
def select_agent(mode, skill_weights, lambda_c):
    # Layer 1: Weighted competence score
    score[agent] = Σ(weight[skill] × competence[agent][skill].mean)

    # Layer 2: Category-level tie-breaking
    # 上升到父类别粒度比较

    # Layer 3: Mode-level overall success rate
    # 使用全局成功率打破僵局

    # Layer 4: Cost-penalized probabilistic sampling
    # adjusted = max(0, 1 - λ_c × cost)
    # 归一化为概率，随机采样
```

这是一个渐进降级的决策机制：从最精细的技能粒度开始，逐步上升到更粗的粒度直到打破僵局。

#### C. 分裂/合并精炼逻辑

**分裂条件**: variance(agent_scores) ≥ threshold (默认 0.15)
- 高方差意味着不同 agent 在该技能上表现差异大
- LLM 审核并生成子技能定义

**合并条件**: max(|perf_diff|) ≤ threshold (默认 0.0)
- 所有 agent 在两个技能上表现相同
- LLM 审核语义是否确实冗余

**Competence 转移**:
```python
merged_bc = BetaCompetence(
    alpha=(bc1.alpha + bc2.alpha) / 2,
    beta=(bc1.beta + bc2.beta) / 2
)
```

#### D. 粒度感知候选生成

基于技能层次深度生成候选 Handbook：
- depth=0: 无技能（纯 mode-level）
- depth=1: 仅类别级技能
- depth=2+: 完整细粒度技能

对每个 mode 的深度进行笛卡尔积组合，生成多个候选 Handbook，在验证集上评估选择 Pareto 最优。

---

## 三、三个核心观点

### 核心观点 1: Skill 作为中间抽象层，将"学什么"与"怎么路由"解耦

**论文论述**:
SkillOrchestra 的核心创新不是路由算法本身，而是引入了 Skill 作为一个**可复用的中间抽象层**，将两个原本耦合的问题解耦：

1. **学什么** (Skill Handbook Learning): 从执行轨迹中离线学习可复用的技能知识、agent 能力画像和路由元数据
2. **怎么路由** (Deployment Routing): 在推理时基于已学知识做在线决策

传统 RL 方法（如 Router-R1）将这两个问题耦合在端到端策略中，导致：
- 策略与特定 orchestrator 绑定（不可迁移）
- 需要大量在线交互（昂贵）
- 路由坍缩（degeneration to single model）

**代码验证**:
在代码中，`SkillHandbook` 是一个纯数据结构（JSON 可序列化），完全独立于 orchestrator 实现。`to_ar.py` 和 `to_stage_router.py` 转换器证明同一个 Handbook 可以服务于不同的路由前端。

**实验证据**:
论文 Figure 6 (right) 展示了从 Qwen2.5-3B 学到的 Handbook 迁移到 7B/8B/70B 模型后仍然有效（+14.8pp ~ +24.3pp），且更强的 backbone 获得更大增益。这证明 Skill 知识是**模型无关的可迁移资产**。

**深层意义**: 这一解耦思路类似于软件工程中"接口与实现分离"的原则。Skill Handbook 定义了"做什么"的接口（技能分类 + 能力画像），而具体的 orchestrator 是"怎么做"的实现。这使得知识可以跨系统、跨模型复用。

---

### 核心观点 2: 粒度不是越细越好——Granularity-Aware Selection 揭示了一个反直觉的设计原则

**论文论述**:
直觉上，更细粒度的技能描述应该带来更精确的路由决策。但 SkillOrchestra 发现事实恰恰相反：

> "Activating numerical_approximation instead of symbolic_logic may route to an agent specialized in numeric computation but suboptimal for symbolic reasoning. Operating at a coarser granularity reduces sensitivity to such misidentification and yields more stable routing decisions." (Section 4.1)

**代码验证**:
`candidates.py` 中的 `generate_depth_candidates()` 为每个 mode 生成从 depth=0 到 max_depth 的所有可能粒度组合。`pareto.py` 中的 `evaluate_candidate_oracle()` 在验证集上评估每个候选。关键的技能识别函数 `_identify_skills_by_indicators()` 使用简单的关键词匹配：

```python
for indicator in skill.indicators:
    if indicator.lower() in query_lower:
        hits += 1
```

当技能数量增多时，关键词匹配的歧义性急剧上升，错误的技能识别导致错误的路由决策。

**深层意义**: 这一发现与 Paper 1 (MAS→SAS) 的**相变理论**高度一致。当技能库规模增长且语义混淆度上升时，选择准确率存在非线性退化。SkillOrchestra 的 Granularity-Aware Selection 本质上是在**信息量**（更细的技能描述）与**噪声鲁棒性**（更粗的粒度减少误识别）之间寻找最优平衡点。

这揭示了一个更普适的设计原则：**最优粒度取决于下游消费者（orchestrator）的能力上限**。弱 orchestrator 需要粗粒度（减少认知负载），强 orchestrator 可以利用细粒度（更精确的匹配）。

---

### 核心观点 3: Routing Collapse 是 RL 方法的系统性缺陷，而非实现 bug

**论文论述**:
Router-R1 在 9 个 benchmark 上将 98.02% 的调用路由到 LLaMA-3.1-70B，而其他 4 个模型几乎未被使用。这不是因为 LLaMA-3.1-70B 在所有任务上都最优，而是 RL 训练过程中的**策略退化**：

1. 奖励信号稀疏（只有最终成功/失败），梯度估计高方差
2. 选择大模型的短期奖励更高，形成正反馈循环
3. 探索不足导致对小模型的能力估计不准确

**代码验证**:
SkillOrchestra 的路由完全不涉及 RL。`select_agent()` 是一个纯推理过程，基于预计算的 Beta competence 做确定性（或轻度随机）决策。Figure 6 (left) 显示其路由分布为 Mixtral 44.53%、Qwen 25.99%、LLaMA 15.38%、Qwen-3B 11.50%——远比 RL 方法均衡。

**深层意义**: Routing Collapse 类似于 RL 中的 exploration-exploitation 困境。SkillOrchestra 通过将"探索"（Exploration Phase 收集多模型轨迹）和"利用"（Deployment 路由）分离到不同阶段，避免了这个问题。探索阶段强制收集所有模型的数据，确保能力估计的完整性；利用阶段基于完整数据做最优决策。

这启示了一个更广泛的系统设计原则：**在复合 AI 系统中，不应该让路由策略自己去探索 agent 能力，而应该通过结构化的 profiling 阶段预先建立完整的能力画像**。

---

## 四、缺点与改进点

### 缺点 1: 冷启动问题——需要预收集全 agent 池的执行轨迹

**问题描述**:
SkillOrchestra 的 Phase 1 需要对每个查询用每个候选 agent 运行一遍，收集 `ExplorationBundle`。对于 N 个查询和 K 个 agent，需要 N×K 次推理。这在以下场景下是不可行的：

- 新增 agent 时需要重新收集轨迹
- agent 池很大时探索成本与池大小线性增长
- 实时生产环境中无法暂停服务做全面探索

**代码证据**:
`scripts/pipeline.py` 的 `explore` 阶段逐模型逐查询收集数据：
```python
# model_routing/explore.py: 对每个 pool model 运行所有查询
```

**改进方向**:
- **增量探索**: 新增 agent 时只收集少量轨迹，利用已有 Handbook 的技能分类做 warm-start
- **主动学习**: 用不确定性采样（Beta 分布方差最大的技能-agent 组合）选择最有信息量的探索查询
- **合成轨迹**: 利用 LLM 生成模拟轨迹进行初步 profiling

---

### 缺点 2: 技能识别依赖 LLM 或关键词匹配，两者都有根本性局限

**问题描述**:
在部署时识别当前查询的 active skills 有两种模式：

1. **LLM-based** (`_identify_skills_with_llm`): 准确但引入额外的 LLM 调用开销
2. **Indicator-based** (`_identify_skills_by_indicators`): 快速但仅靠关键词匹配，语义理解能力弱

**代码证据**:
```python
# pareto.py 中的 oracle 评估使用 indicator-based（快速但不精确）
def _identify_skills_by_indicators(query, skills):
    for indicator in skill.indicators:
        if indicator.lower() in query_lower:
            hits += 1
```

这种基于字面匹配的方法无法处理：
- 同义词（"compute" vs "calculate"）
- 隐式技能需求（"What is the GDP of China minus Japan?" 隐含 math + factual）
- 多语言场景

**改进方向**:
- **轻量级分类器**: 训练一个小型 embedding 模型（如 MiniLM）做技能分类，平衡精度与成本
- **向量检索**: 用技能 description 的 embedding 做语义匹配而非关键词匹配
- **层级分类**: 先粗粒度分类（低成本），再按需细化（高成本），类似 FrugalGPT 的 cascade

---

### 缺点 3: 静态 Handbook 无法适应分布漂移

**问题描述**:
Handbook 在离线学习后固定，不随运行时反馈更新。当任务分布发生变化时：
- 技能分类可能不再适用
- Agent 能力可能随模型更新而变化
- 新类型的查询可能不在已知技能范围内

**代码证据**:
`SkillOrchestrator` 的 `route()` 方法是纯读取操作，不修改 Handbook。没有任何在线更新机制。

**改进方向**:
- **在线 Beta 更新**: 每次路由决策后，基于结果反馈增量更新 α/β（已有数据结构支持，缺少在线更新管线）
- **漂移检测**: 监控路由准确率的滑动窗口，当检测到显著下降时触发 re-profiling
- **版本化 A/B 测试**: 维护多版本 Handbook，用 bandit 策略选择当前最优版本

---

### 缺点 4: Competence 合并策略过于简单

**问题描述**:
合并两个技能时，competence 的转移使用简单平均：
```python
merged_bc = BetaCompetence(
    alpha=(bc1.alpha + bc2.alpha) / 2,
    beta=(bc1.beta + bc2.beta) / 2
)
```

这导致信息丢失。例如 Beta(10,2) + Beta(2,10) → Beta(6,6)，原来一个强一个弱，合并后变成中等。

**改进方向**:
- **加权合并**: 按技能出现频率加权
- **保留子技能信息**: 合并后保留原始技能作为"虚拟子节点"的 competence
- **Beta mixture**: 使用 Beta 混合分布而非简单平均

---

### 缺点 5: 缺乏安全治理机制

**问题描述**:
- 技能发现完全依赖 LLM，可能引入幻觉（虚构不存在的技能）
- 没有技能质量验证机制（如 SkillsBench 的 deterministic verifier）
- Agent Profile 的自然语言摘要（strengths/weaknesses）未经验证
- 没有 trust-tiered execution（如 SoK 论文建议的安全分级）

**改进方向**:
- 增加确定性验证层：技能定义后用测试用例验证其区分力
- 引入置信度阈值：低置信度的 Beta competence 标记为 "uncertain"，避免基于不可靠数据做决策
- 实现 ClawHavoc 式攻击测试（SoK 论文方案）

---

### 缺点 6: Pareto 选择的评估效率瓶颈

**问题描述**:
候选 Handbook 数量为各 mode 最大深度的笛卡尔积。例如 3 个 mode 各有深度 0-4，则有 5³ = 125 个候选。每个候选需要在验证集上完整评估。

Live 评估（运行真实 orchestrator）的成本与候选数量线性增长。

**改进方向**:
- **贪心搜索**: 逐 mode 搜索最优深度，而非全笛卡尔积
- **早停**: 低于当前最优 score 的候选提前终止评估
- **代理模型**: 训练一个轻量级回归模型预测候选的性能

---

### 缺点 7: 单 mode 内的 Agent Profile 假设

**问题描述**:
当前架构要求每个 AgentProfile 绑定到单一 mode。这意味着同一个底层模型（如 GPT-5）在不同 mode（search/code/answer）下会有独立的 profile，且它们之间没有信息共享。

**代码证据**:
```python
class AgentProfile:
    mode: str  # 绑定到单一 mode
```

**改进方向**:
- **跨 mode 能力迁移**: 如果 GPT-5 在 "code" mode 展现了强逻辑能力，这个信号应该部分迁移到 "answer" mode
- **层次化 Profile**: 全局能力 → mode 能力 → skill 能力的三层模型

---

## 五、演进路线建议

### 短期（1-2个月）

1. **实现在线 Beta 更新管线** — 最小改动，最大收益
2. **替换 indicator-based 识别为 embedding 语义匹配** — 显著提升路由精度
3. **增加贪心搜索替代笛卡尔积候选** — 降低评估成本

### 中期（3-6个月）

4. **增量冷启动方案** — 新 agent 加入时无需全面重新探索
5. **在线漂移检测 + 自适应 re-profiling**
6. **跨 mode 能力迁移模型**

### 长期（6-12个月）

7. **与 AgentSkillOS 集成**: 利用 Capability Tree 做大规模技能组织，SkillOrchestra 做路由
8. **与 SkillsBench 集成**: 用确定性验证器评估技能质量
9. **DAG + Handbook 混合编排**: 确定性子任务用 DAG（Paper 2），不确定性子任务用 Handbook 路由

---

*本报告基于 SkillOrchestra 最新代码（commit 9a487ce）和论文 arXiv:2602.19672 的深度分析。*
