# AgentSkillOS 深度研究报告

**论文**: Organizing, Orchestrating, and Benchmarking Agent Skills at Ecosystem Scale
**作者**: Hao Li, Chunjiang Mu, Jianhao Chen et al. (上海人工智能实验室)
**ArXiv**: 2603.02176 (2026)
**代码**: https://github.com/ynulihao/AgentSkillOS

---

## 一、代码运行验证

### 1.1 核心模块运行结果

成功运行 AgentSkillOS 核心模块，验证了以下功能：

| 模块 | 测试内容 | 结果 |
|------|---------|------|
| `DynamicTreeConfig` | 参数派生（branching_factor=8 → max_skills_per_node=12, expand_threshold=5 等） | ✅ 正常 |
| `TreeNode` | 多级树构建、递归 skill 计数、叶节点判断 | ✅ 正常 |
| `Skill` | 数据模型创建、属性设置 | ✅ 正常 |
| `DependencyGraph` | DAG 构建、环检测、拓扑排序、并行分层 | ✅ 正常 |
| `build_graph_from_nodes` | 从 JSON 节点数据构建 DAG | ✅ 正常 |
| `Serialization` | TreeNode → dict → TreeNode 往返序列化 | ✅ 无数据丢失 |
| `fit_bradley_terry` | MM 算法拟合、Laplace 平滑、分数缩放 | ✅ 正常 |
| `consolidate_fwd_rev_verdict` | 正反向评判合并（一致/矛盾/单侧错误） | ✅ 正常 |

### 1.2 DAG 执行逻辑验证

```
样例 DAG: 生成演示文稿（4 节点）

Phase 1 (parallel): [generate-images, write-outline]
Phase 2 (sequential): [create-slides]  (depends on Phase 1)
Phase 3 (sequential): [export-pdf]     (depends on Phase 2)

拓扑排序: generate-images → write-outline → create-slides → export-pdf
环检测: None（无环）
```

### 1.3 失败级联验证

```
generate-images FAILED →
  create-slides SKIPPED (依赖 generate-images)
  export-pdf SKIPPED (依赖 create-slides)
  write-outline PENDING (无依赖关系，不受影响)
```

级联失败逻辑正确：仅跳过直接和间接依赖节点，不影响独立节点。

### 1.4 Bradley-Terry 排名验证

```
#1 Quality-First        Score: 100.00
#2 Efficiency-First     Score:  62.48
#3 Simplicity-First     Score:  45.28
#4 w/Full Pool          Score:  20.15
#5 Vanilla              Score:   0.00
```

MM 算法收敛正常，排名结果符合预期：结构化编排 >> 非结构化调用 >> 无 skill。

---

## 二、代码架构深度解读

### 2.1 整体架构

```
AgentSkillOS/
├── src/
│   ├── manager/                    # Manage Skills 阶段
│   │   ├── tree/                   # 核心: Capability Tree
│   │   │   ├── models.py           # TreeNode, Skill, DynamicTreeConfig
│   │   │   ├── builder.py          # 两阶段树构建 (Structure Discovery + Anchored Classification)
│   │   │   ├── searcher.py         # 多级树搜索 (LLM 逐层选择)
│   │   │   ├── layered_searcher.py # 分层搜索 (active + dormant)
│   │   │   ├── layer_processor.py  # Active/Dormant 分割
│   │   │   ├── dormant_indexer.py  # 休眠 skill 向量索引
│   │   │   ├── dormant_searcher.py # 休眠 skill 向量搜索
│   │   │   ├── skill_scanner.py    # SKILL.md 文件扫描解析
│   │   │   ├── scheduled_updater.py# 增量更新调度
│   │   │   ├── prompts.py          # 所有 LLM prompt 模板
│   │   │   └── visualizer.py       # 树可视化 (HTML)
│   │   ├── vector/                 # 向量索引管理
│   │   └── direct/                 # 直接管理 (无树)
│   ├── orchestrator/               # Solve Tasks 阶段
│   │   ├── dag/                    # 核心: DAG 编排引擎
│   │   │   ├── engine.py           # SkillOrchestrator 主引擎
│   │   │   ├── graph.py            # DependencyGraph DAG 结构
│   │   │   ├── prompts.py          # Planner/Executor prompt
│   │   │   ├── skill_registry.py   # Skill 注册表
│   │   │   └── throttler.py        # 并发节流器
│   │   ├── runtime/                # 运行时基础设施
│   │   │   ├── client.py           # Claude SDK 客户端封装
│   │   │   ├── models.py           # SkillNode, NodeStatus, ExecutionPhase
│   │   │   ├── run_context.py      # 隔离执行环境
│   │   │   └── prompts.py          # 直接执行 prompt
│   │   ├── direct/                 # 直接执行引擎 (无 DAG)
│   │   └── freestyle/              # 自由编排引擎
│   ├── web/                        # Web UI 服务
│   ├── workflow/                   # 批量执行工作流
│   └── skill_orchestrator/         # 旧版编排器 (兼容层)
├── benchmark/                      # 评测基准
│   └── AgentSkillOS_bench/
│       ├── tasks/                  # 30 个任务定义 (5 类 × 6 个)
│       ├── ranking/                # Bradley-Terry 排名系统
│       │   ├── rank.py             # 多方法排名 pipeline
│       │   ├── compare.py          # LLM 成对比较
│       │   └── checkpoint.py       # 检查点缓存
│       ├── evaluators/             # 客观评估器
│       └── utils/                  # 工具函数
├── data/
│   ├── skill_seeds/                # 种子 skill 集合 (docx, pptx, pdf, ...)
│   └── capability_trees/           # 预构建的 capability tree
└── run.py                          # 主入口
```

### 2.2 核心数据流

```
Skill Ecosystem (S)
    │
    ▼
┌─────────────────────────┐
│ Usage-Frequency Queue   │  TopK(install_count) → S_T
│ (200K → 10K active)     │  User-pinned → S_user
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Capability Tree Builder │  Node-Level Recursive Categorization
│                         │  固定5大根类 → LLM Group Discovery → LLM Skill Assignment
│                         │  递归分裂 until |skills_per_node| ≤ C
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Task-Driven Retrieval   │  LLM 逐层树搜索 → 候选 skills
│                         │  + 向量搜索补充 dormant skills
│                         │  → Pruning (去重 + 排名 → Top M)
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ DAG-based Orchestration │  3 策略: Quality-First / Efficiency-First / Simplicity-First
│                         │  LLM 生成 3 个 DAG plan
│                         │  用户选择 → 构建 DependencyGraph
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ Multi-skill Execution   │  分层并行执行
│                         │  每个节点: 独立 Claude SDK session
│                         │  artifact 传递 + usage_hints
│                         │  失败级联 + 超时保护
└─────────────────────────┘
```

### 2.3 关键算法详解

#### A. Node-Level Recursive Categorization (树构建)

```python
# builder.py 的核心逻辑
def _build_tree(root, skills):
    # 根节点使用固定 5 大类别
    if is_root:
        categories = FIXED_ROOT_CATEGORIES  # 手动指定，确保稳定性
    else:
        categories = LLM_group_discovery(skills, B-3 ~ B+2 groups)

    assignments = LLM_skill_assignment(skills, categories)

    for category in categories:
        if len(assigned_skills) == 1:
            merge_into_nearest_category()  # 单 skill 合并
        elif len(assigned_skills) <= C:
            make_leaf_node()  # 直接作为叶节点
        else:
            recurse(_build_tree(category_node, assigned_skills))

    # 未分配的 skill: 二次分配 → 最大类别兜底
```

**设计亮点**: 通过固定根类别确保一级分类的稳定性，同时用 LLM 自适应地发现子类别。单 skill 合并和叶节点提前终止减少了树深度。

#### B. Multi-Level Tree Search (树搜索)

```python
# searcher.py 的核心逻辑
def _search_node(query, node, depth):
    if node.is_leaf:
        return LLM_skill_selection(query, node.skills)  # 从叶节点选 skill

    if len(node.children) <= expand_threshold:
        # 子节点数少，全部展开
        return parallel_search_all_children(query, node.children)
    else:
        # 子节点数多，LLM 选择相关分支
        selected_branches = LLM_node_selection(query, node.children)
        return parallel_search(query, selected_branches)

# 剪枝阶段
def _prune_skills(query, candidates):
    return LLM_dedup_and_rank(query, candidates, top_M=8)
```

**设计亮点**: 自适应展开策略——小扇出全展开避免遗漏，大扇出 LLM 选择避免信息过载。并行搜索兄弟分支提高效率。

#### C. DAG Execution Engine (DAG 执行)

```python
# engine.py 的核心逻辑
class SkillOrchestrator:
    async def run_with_visualizer(task, skill_names):
        # Step 1: Load skills
        skills = registry.find_by_names(skill_names)

        # Step 2: Generate 3 plans (Quality/Efficiency/Simplicity)
        plans = await _generate_plans(task, skills)  # LLM 生成 DAG

        # Step 3: User selects plan → Build DependencyGraph
        graph = build_graph_from_nodes(selected_plan["nodes"])
        phases = graph.get_execution_phases()

        # Step 4: Execute layer by layer
        for phase in phases:
            results = await _execute_phase_parallel(phase)  # 并行执行同层节点

    async def _execute_node_isolated(node_id):
        # 每个节点独立 Claude SDK session (防止上下文污染)
        async with SkillClient(session_id=f"node-{node_id}") as client:
            prompt = build_isolated_executor_prompt(
                overall_task, skill_name, node_purpose,
                artifacts_context,  # 上游产物路径 + usage_hints
                outputs_summary, downstream_hint
            )
            response = await client.execute(prompt)
```

**设计亮点**: 每个 DAG 节点使用独立的 Claude SDK session，确保真正的并行执行和上下文隔离。artifacts_context + usage_hints 实现了跨节点的信息传递。

#### D. Bradley-Terry Evaluation (评测体系)

```python
# rank.py 的核心逻辑
# 1. 所有 C(N,2) 对的成对比较，正反两个方向
for i, j in combinations(N, 2):
    fwd_result = LLM_judge(task, system_A=i, system_B=j)
    rev_result = LLM_judge(task, system_A=j, system_B=i)
    verdict = consolidate_fwd_rev_verdict(fwd, rev)
    # 一致→采纳, 矛盾→tie, 单侧error→用有效侧

# 2. MM 算法拟合 Bradley-Terry
def fit_bradley_terry(wins, alpha=0.1):
    # Laplace 平滑: w += alpha
    # 迭代: π_i = w_i / Σ_j (n_ij / (π_i + π_j))
    # 归一化: geometric mean = 1
    # 收敛: max|Δlog(π)| < tol

# 3. 线性缩放到 [0, 100]
S_i = (β_i - β_min) / (β_max - β_min) × 100
```

---

## 三、三个核心观点

### 核心观点 1: 结构化组合是释放 Skill 生态潜力的关键——不是"有 skill"而是"怎么用 skill"

**论文论述**:
AgentSkillOS 最核心的发现是：**即使给予完全相同的 oracle skill 集合，有 DAG 编排的系统也显著优于无编排的 flat invocation**。这意味着 skill 生态系统的价值瓶颈不在于 skill 的数量或质量，而在于**组合方式**。

论文 Figure 5 的消融实验清楚展示了这一点：
- w/ Oracle Skills（有 oracle skill，无编排）vs Quality-First（有编排）：Quality-First 在 200 skill 规模下 22W-5T-3L 大幅领先
- 这证明 DAG 编排的增益是**独立于 skill 检索质量的**

**代码验证**:
在 `engine.py` 中，DAG 编排通过三个机制实现结构化组合：
1. **分层依赖**: `get_execution_phases()` 将 DAG 分解为可并行执行的层次
2. **Artifact 传递**: `_build_artifacts_context()` 构建上游产物路径和 usage_hints
3. **独立 session**: `_execute_node_isolated()` 为每个节点创建隔离的执行环境

三种策略（Quality/Efficiency/Simplicity）生成结构性截然不同的 DAG（Figure 6）：
- Quality-First: 最多节点、最深、最多边（精细化多阶段流水线）
- Efficiency-First: 相似节点数但更宽更浅（最大化并行）
- Simplicity-First: 最少节点和最稀疏连接（最小不可约核心）

**深层意义**: 这与软件工程中的"组合优于继承"原则高度一致。Skill 就像函数，单个函数的能力是有限的，真正的力量来自于函数之间的组合和编排。AgentSkillOS 证明了：**在 AI agent 领域，结构化的任务分解和多 skill 协作是超越单 skill 能力上限的核心机制**。这对未来 agent 平台的设计有深远影响——平台应该投入更多精力在编排层而非仅仅扩大 skill 数量。

---

### 核心观点 2: Capability Tree 是解决"生态规模悖论"的有效中间层——树结构使 LLM 能够"推理发现"而非仅"语义检索"

**论文论述**:
存在一个根本性的矛盾：**skill 生态越大，单个 agent 发现和调用正确 skill 的能力越弱**。论文称之为"fundamental tension"。w/ Full Pool 配置（将整个生态直接提供给 Claude Code）在所有规模下都表现不佳，且随规模增长进一步退化。这是因为 Claude Code 的原生 skill 调用机制存在上下文窗口限制——当 skill 数量超过上限时，大部分 skill 对 agent 不可见。

Capability Tree 解决这个问题的方式是**将 O(N) 的全局搜索转化为 O(B × log_B(N)) 的树搜索**。

**代码验证**:
在 `searcher.py` 中，树搜索的关键设计是：
- **自适应展开**: `expand_threshold` 控制小扇出全展开 vs 大扇出 LLM 选择
- **并行搜索**: 兄弟分支并行查询，降低延迟
- **早停**: `early_stop_skill_count` 当剩余 skill 数量足够小时直接返回

更重要的是 `layered_searcher.py` 的 Active/Dormant 分层策略：
- 200K skills → TopK(install_count, 10000) 构建 active tree
- 剩余 190K skills → dormant index（向量语义索引）
- 搜索时：先搜 active tree，再从 dormant 补充建议

这使得系统可以在保持 O(log N) 搜索效率的同时，不完全丢失长尾 skill 的可达性。

**深层意义**: Capability Tree 的价值不仅在于效率，更在于**推理发现（reasoning-based discovery）vs 语义检索（embedding-based retrieval）的范式差异**。纯语义检索只能找到与查询语义相近的 skill，而树搜索通过 LLM 的推理能力，可以发现语义上不直接相关但功能上互补的 skill。例如，用户请求"制作量子计算教学包"时，树搜索可以通过推理发现 image-generation（用于 Bloch 球可视化）和 manim（用于动画），而纯语义检索可能只找到"quantum"关键词匹配的 skill。

---

### 核心观点 3: 评测范式创新——从"是否完成"到"产物质量"的跨越，LLM-as-Judge + Bradley-Terry 实现了开放式任务的可量化评估

**论文论述**:
现有 agent 评测基准（SWE-bench, AgentBench, GAIA 等）有两个结构性限制：
1. 工具集预定义且有限，不涉及开放式 skill 空间
2. 评估指标为端到端成功率（binary pass/fail），无法区分产物质量

AgentSkillOS-bench 的创新在于：
- 30 个任务要求生成**完整的终端用户产物**（PDF, PPTX, HTML, 视频, 图片），而非代码或答案
- 使用 **LLM 成对比较** 替代绝对打分，消除了评分标准的主观性
- **双向评判 + 矛盾检测** 消除位置偏差
- **Bradley-Terry 模型** 将成对偏好聚合为连续分数，实现细粒度排名

**代码验证**:
`rank.py` 实现了完整的评测 pipeline：
1. `run_all_comparisons()`: 对 N 个方法的所有 C(N,2) 对运行双向比较
2. `consolidate_fwd_rev_verdict()`: 合并正反向结果（一致→采纳, 矛盾→tie）
3. `fit_bradley_terry()`: MM 算法拟合（Laplace 平滑 α=0.1）
4. `scores_to_scale()`: 线性缩放到 [0,100]
5. `aggregate_per_task_scores()`: 按任务和类别聚合

Checkpoint 机制（`checkpoint.py`）保证了大规模评测的可恢复性。

**深层意义**: 这种评测范式有三个重要启示：
1. **产物质量 > 任务完成率**：未来的 agent 评测应该更关注"做得多好"而非"是否做完"
2. **Bradley-Terry 的优越性**：相比 Elo、TrueSkill 等竞争排名算法，BT 模型在有限比较次数下更稳定，且数学性质使其适合无记忆的批量评测
3. **可扩展性**：per-task BT fitting + category aggregation 的两层结构使得评测结果既有细粒度可解释性，又有全局可比性

---

## 四、缺点与改进点

### 缺点 1: Capability Tree 构建严重依赖 LLM，成本高且不可控

**问题描述**:
树构建过程需要对每个节点调用 LLM 两次（Group Discovery + Skill Assignment），且递归直到所有节点 skill 数 ≤ C。对于 200K skill 的生态：
- Active 树（10K skills）构建需要 O(10K / B × 2) = ~2500 次 LLM 调用
- 每次树更新（新增 skill）需要路径上每个节点的重新分类

**代码证据**:
```python
# builder.py
# Group Discovery: 1 次 LLM 调用
categories = LLM(GROUP_DISCOVERY_PROMPT, skills)
# Skill Assignment: 1 次 LLM 调用 (or batched)
assignments = LLM(SKILL_ASSIGNMENT_PROMPT, skills, categories)
# 递归: 每个子节点重复上述过程
```

LLM 的非确定性可能导致同一批 skill 的不同运行生成不同的树结构，影响评测的可复现性。

**改进方向**:
- **Embedding 预分类 + LLM 精炼**: 用 embedding 聚类做初始分组（O(N) 且确定性），仅用 LLM 精炼类别名称和描述
- **增量构建**: 新 skill 只做路径上的 O(depth) 次分类，不触发全局重建
- **缓存 + 指纹**: 代码中已有 `_prompt_fingerprint_version` 和 cache 机制，但可以进一步利用确定性哈希避免重复调用

---

### 缺点 2: DAG 编排的"one-shot"生成——无反馈、无修正

**问题描述**:
DAG plan 由 LLM 一次性生成，没有验证或修正机制：
- LLM 可能生成不合理的依赖关系
- 节点失败后只有 cascade skip，没有重试或替代路径
- 没有 plan 质量的自动评估

**代码证据**:
```python
# engine.py
async def _generate_plans(self, task, skills, context):
    prompt = build_planner_prompt(task, skill_info, context_str)
    response = await self.client.execute(prompt)
    result = extract_json(response)  # 直接解析，无验证
```

节点失败后的处理非常简单：
```python
def fail_node(self, node_id):
    # 标记失败 + 级联 skip 所有后继
    skip_dependents(node_id)  # 无重试、无替代
```

**改进方向**:
- **Plan Verification**: 生成后用 LLM 验证 DAG 的完整性（数据流是否闭合、是否覆盖所有需求）
- **Adaptive Re-planning**: 节点失败后触发局部重新规划（类似 Re-planning in LLM agents）
- **备选路径**: DAG 中预定义 fallback 边，某节点失败时自动切换到备选 skill
- **Plan Selection 自动化**: 当前需要用户手动选择 3 个 plan 之一，可以用轻量级评估自动选择

---

### 缺点 3: 评测的 LLM-as-Judge 偏差未充分控制

**问题描述**:
虽然双向评判消除了位置偏差，但 LLM judge 还存在其他系统性偏差：
- **Verbosity bias**: 倾向于偏好更长、更详细的输出
- **Self-preference bias**: Claude 评判可能偏好 Claude 生成的产物
- **Format bias**: 倾向于偏好格式更精美的输出，即使内容不如对手

**代码证据**:
`compare.py` 的评判仅依赖 LLM 判断，没有客观评估器的校正：
```python
# 唯一的偏差缓解：双向评判 + 矛盾检测
verdict = consolidate_fwd_rev_verdict(fwd_pref, rev_pref)
```

虽然 `evaluators/objective/` 目录下有客观评估器（numeric_evaluators, format_evaluators 等），但它们在 ranking pipeline 中**未被使用**。

**改进方向**:
- **混合评估**: 将客观评估器得分作为 Bradley-Terry 的先验或权重
- **多 Judge 交叉验证**: 用多个 LLM（Claude + GPT-4 + Gemini）作为 judge，取共识
- **人类标注校准**: 收集少量人类偏好标注，评估 LLM judge 与人类的一致率
- **Control for confounders**: 在评判 prompt 中明确要求忽略格式/长度差异

---

### 缺点 4: 搜索策略缺乏学习——每次搜索从零开始

**问题描述**:
`Searcher` 每次搜索都是无记忆的：不记录历史搜索路径，不学习哪些路径对特定任务类型最有效。这意味着：
- 相似任务会重复探索相同的无关分支
- 无法利用历史成功案例优化搜索策略
- 搜索效率不会随使用而提升

**代码证据**:
```python
# searcher.py
def search(self, query, verbose=False):
    self._llm_calls = 0  # 每次重置
    self._explored_nodes = []  # 无历史
    selected_skills = self._search_node(query, self._tree, depth=0)
```

虽然 `web/recipe.py` 实现了 DAG plan 的复用机制（基于任务描述的向量相似度），但 **skill 检索路径**没有类似的复用机制。

**改进方向**:
- **Search Path Caching**: 缓存 (task_embedding → selected_branches) 映射，相似任务直接复用路径
- **UCB-based Exploration**: 用 Upper Confidence Bound 平衡探索 vs 利用已知路径
- **Feedback Loop**: 任务执行结果反馈到 skill 权重，下次搜索时优先选择历史表现好的 skill

---

### 缺点 5: 跨 skill 数据流的"弱约束"——usage_hints 仅是自然语言

**问题描述**:
DAG 节点之间的数据传递依赖 `artifacts_context`（上游产物路径）和 `usage_hints`（自然语言描述如何使用），没有类型化的接口定义。

**代码证据**:
```python
# engine.py
artifact_lines.append(
    f"### {nid} ({n.name})\n"
    f"- Path: {n.output_path}\n"
    f"- How to use: {usage_hint}"  # 纯自然语言
)
```

这导致：
- 下游节点可能误解上游的输出格式
- 无法自动验证数据流的类型兼容性
- 当上游输出路径不存在时，下游才发现错误（运行时失败）

**改进方向**:
- **Schema-based Artifacts**: 为 skill 定义 input/output schema（类型、格式、路径模式）
- **Pre-flight Validation**: 执行前检查所有节点的 output schema 是否满足下游的 input schema
- **Artifact Registry**: 中央化的产物注册表，替代路径字符串传递

---

### 缺点 6: Active/Dormant 分层策略过于简单——仅基于 install count

**问题描述**:
当 |S| > K 时，仅按 install count 选择 Top-K 进入 active tree。这导致：
- 新 skill（install=0）永远无法进入 active tree，除非用户手动 pin
- 高安装量但低质量的 skill 占据 active 位置
- 没有考虑 skill 的互补性——active tree 可能在某些类别过于密集、另一些类别稀缺

**代码证据**:
```python
# models.py
class SkillStatus(str, Enum):
    ACTIVE = "active"      # Top N by installs
    DORMANT = "dormant"    # rest
    PINNED = "pinned"      # user-pinned
```

**改进方向**:
- **多维度排名**: 综合 install count、star count、质量评分、覆盖度缺口
- **Diversity-aware Selection**: 确保每个根类别至少有 K/5 个 active skill
- **Dynamic Promotion/Demotion**: 基于使用频率和成功率动态调整 active/dormant 状态
- **Cold-start Exposure**: 为新 skill 提供"试用期"——在 active tree 中临时曝光一定次数

---

### 缺点 7: 并发执行的资源管理不够精细

**问题描述**:
DAG 同一 phase 的所有节点并行执行，每个节点启动一个独立的 Claude SDK session。当 phase 宽度大（如 Efficiency-First 策略）时：
- 同时启动多个 Claude API 调用，可能触发 rate limit
- 没有考虑不同 skill 的资源需求差异（CPU/GPU/内存）
- 超时设置是全局统一的 `node_timeout`，不区分任务复杂度

**代码证据**:
```python
# engine.py
self.throttler = ExecutionThrottler(max_concurrent=_max_concurrent)
# 全局统一超时
self.node_timeout = node_timeout if node_timeout is not None else ocfg.node_timeout
```

**改进方向**:
- **Per-skill 资源标签**: 在 SKILL.md 中声明资源需求（api_calls, gpu, memory）
- **动态并发调整**: 根据当前 API 使用率和错误率动态调整 max_concurrent
- **自适应超时**: 根据 skill 历史执行时间设置 per-node timeout
- **优先级调度**: 关键路径上的节点优先执行

---

## 五、与 SkillOrchestra 的对比分析

| 维度 | AgentSkillOS | SkillOrchestra |
|------|-------------|----------------|
| **核心问题** | 大规模 skill 生态的组织与编排 | 多 agent 路由与能力画像 |
| **Skill 组织** | Capability Tree（层次化分类） | Skill Handbook（平面技能+Beta 能力值） |
| **任务分解** | DAG 多 skill 编排 | 单 skill 路由选择 |
| **学习方式** | 无学习（LLM 构建树+LLM 搜索） | 离线 profiling + Beta 更新 |
| **评测方式** | LLM 成对比较 + Bradley-Terry | 标准 benchmark 正确率 |
| **规模验证** | 200 ~ 200K skills | 数十个 benchmark |
| **互补性** | 适合"多个 skill 协作"场景 | 适合"选一个最优 agent"场景 |

**互补融合方向**:
1. **SkillOrchestra 的 Beta 能力建模 + AgentSkillOS 的 Capability Tree**: 在树的每个叶节点 skill 上附加 Beta competence，搜索时用能力值加权选择
2. **AgentSkillOS 的 DAG 编排 + SkillOrchestra 的多 agent 路由**: DAG 中每个节点不是固定 skill，而是通过 SkillOrchestra 的路由机制动态选择最优 agent 执行
3. **统一评测**: 用 AgentSkillOS-bench 的产物质量评测替代 SkillOrchestra 的 pass/fail 评测

---

## 六、演进路线建议

### 短期（1-2个月）

1. **实现 Search Path Caching** — 缓存历史搜索路径，相似任务直接复用
2. **增加 Plan Verification** — LLM 验证生成的 DAG 完整性
3. **集成客观评估器到 ranking pipeline** — 混合 LLM 判断和客观指标

### 中期（3-6个月）

4. **Embedding 预分类 + LLM 精炼** — 降低树构建的 LLM 调用成本
5. **Adaptive Re-planning** — 节点失败后局部重新规划
6. **Schema-based Artifact Flow** — 为 skill 定义类型化的输入输出接口
7. **多维度 Active/Dormant 策略** — 综合质量、覆盖度、使用频率

### 长期（6-12个月）

8. **与 SkillOrchestra 融合**: Capability Tree + Beta Competence + DAG 编排 + 动态路由
9. **Skill Self-Evolution**: 基于执行反馈自动精炼 skill 指令
10. **Automated Skill Discovery**: 从开源代码库自动发现、生成、集成新 skill
11. **Federated Skill Ecosystem**: 跨组织、跨平台的分布式 skill 管理

---

*本报告基于 AgentSkillOS 最新代码和论文 arXiv:2603.02176 的深度分析。代码运行验证于 2026-03-24。*
