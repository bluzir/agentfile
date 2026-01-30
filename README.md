# Agent Orchestration Framework v1.0

Конвенции и шаблоны для организации multi-agent систем поверх примитивов Claude Code.

---

## Philosophy

```
Markdown is the new JavaScript.
YAML is the new PostgreSQL.
```

**Markdown** — это код для LLM. Инструкции, процедуры, constraints. LLM исполняет markdown как runtime исполняет JS.

**YAML** — это данные. Структурированные, версионируемые, читаемые человеком. Заменяет базу данных для большинства agent workflows.

### Подход: Framework + Domain → Pipeline

Ты не пишешь агентов руками. Ты:

1. **Даёшь фреймворк** — конвенции, schemas, templates
2. **Даёшь domain knowledge** — специфика области (здоровье, research, marketing)
3. **Просишь Claude упаковать в pipeline**

```
┌─────────────────┐   ┌─────────────────┐
│   Framework     │ + │  Domain Data    │
│   (conventions) │   │  (your context) │
└────────┬────────┘   └────────┬────────┘
         │                     │
         └──────────┬──────────┘
                    ▼
         ┌─────────────────────┐
         │   Claude Code       │
         │   generates         │
         │   pipeline          │
         └─────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  Working agents     │
         │  + skills           │
         │  + workflows        │
         └─────────────────────┘
```

**Почему это работает:**

- LLM понимает conventions лучше чем API docs
- Markdown templates = примеры для in-context learning
- YAML schemas = constraints для structured output
- Фреймворк = shared language между тобой и Claude

**Пример prompt:**

```
У меня есть фреймворк (framework/README.md) и domain knowledge
о health tracking (data/config/user_profile.yaml, training-science.md).

Создай pipeline который:
1. Читает daily log
2. Сверяет с directives
3. Генерирует recommendations

Используй конвенции фреймворка: agents, skills, data layers.
```

Claude создаёт:
- `manager-daily-review.md` (orchestration skill)
- `log-analyzer.md` (worker agent)
- `recommendation-generator.md` (worker agent)
- `dashboard.yaml` (interface)

Всё по конвенциям. Всё совместимо с другими pipelines.

---

## Для кого этот фреймворк

Ты строишь агентов на Claude Code и сталкиваешься с:
- Pipeline из нескольких шагов (research → analyze → generate)
- Необходимостью переиспользовать knowledge между агентами
- Вопросами "где хранить state?" и "как resume после падения?"
- Желанием иметь структуру, но без тяжёлых фреймворков

Этот фреймворк — набор **конвенций**, не библиотека. Ты используешь его как reference и адаптируешь под себя.

---

## Проблематика

### Почему multi-agent системы сложные

**1. Coordination Explosion**

Один агент = prompt + tools. Просто.

Два агента = кто кого вызывает? как передать данные? что если один упал?

Пять агентов = exponential complexity. Каждый может влиять на каждого.

```
Agents:     1    2    3    4    5
Complexity: O(1) O(n) O(n²) ...
```

**2. No Agreed Patterns**

В веб-разработке есть MVC, REST, microservices. Все понимают что это.

В agent development:
- Кто-то делает monolithic agent на 3000 строк
- Кто-то дробит на 50 micro-agents
- Кто-то пишет на LangChain, кто-то на AutoGen, кто-то на голом API

Нет общего языка. Нет переиспользуемых паттернов.

**3. State Management Chaos**

Где живёт состояние между вызовами агентов?

| Вариант | Проблема |
|---------|----------|
| Context window | Теряется при длинных сессиях, дорого |
| In-memory | Теряется при restart |
| Database | Оверкилл, нужна инфраструктура |
| Files | Какой формат? кто владелец? как версионировать? |

**4. Debugging Hell**

Pipeline из 5 агентов падает на 4-м шаге.

- Где именно сломалось?
- Как воспроизвести?
- Как перезапустить с mid-point?
- Какие промежуточные данные были?

Observability в agent systems — нерешённая проблема индустрии.

**5. Knowledge Duplication**

Каждый агент содержит copy-paste инструкции:
- "Не галлюцинируй"
- "Проверяй источники"
- "Форматируй output как YAML"
- "Не используй AI-типичные фразы"

Code reuse давно решён. Knowledge reuse — нет.

---

### Подходы которые не работают

**Monolithic Agent**
```
Один гигантский system prompt со всей логикой.
```
- Context overflow на сложных задачах
- Невозможно тестировать части
- Одно изменение = риск сломать всё
- Нет параллелизма

**Micro-Agents**
```
Каждое действие = отдельный агент.
```
- Overhead на координацию
- Потеря контекста между вызовами
- Debugging 50 агентов = nightmare
- Latency от sequential calls

**Heavy Frameworks (LangChain, AutoGen)**
```
Использовать framework для всего.
```
- Абстракции leaky
- Сложно кастомизировать под свои нужды
- Зависимость от чужого roadmap
- Документация отстаёт от кода

---

### Как этот фреймворк решает проблемы

| Проблема | Решение |
|----------|---------|
| Coordination explosion | **Flat hierarchy**: ROOT orchestrates, Workers execute |
| No patterns | **Taxonomy**: 4 архитектурных роли, 8 функциональных ролей |
| State chaos | **Data Layers**: L0→L1→L2→L3 с чёткими контрактами |
| Debugging | **File-based state**: inspect, resume, replay |
| Knowledge duplication | **Skills**: shared libraries для instructions |

**Key Insight: Convention over Configuration**

Claude Code уже имеет примитивы:

| Примитив | Что делает |
|----------|------------|
| Task tool | Spawn subagent |
| Skill tool | Load knowledge pack |
| MCP | External integrations |
| Files | Persistent state |

Не нужно изобретать runtime. Нужны **конвенции** как эти примитивы использовать вместе.

**Architectural Constraint as Feature**

```
Claude Code: Subagents CANNOT spawn other subagents
```

Это выглядит как ограничение, но это — design decision:
- Форсирует flat hierarchy (проще debug)
- ROOT = single point of control
- Workers = pure executors (проще test)

**Skills = Knowledge Reuse**

```
Agent A + grounding-protocol = Agent A который не галлюцинирует
Agent B + grounding-protocol = Agent B который не галлюцинирует
```

Skill — не агент. Skill — набор инструкций который загружается в context.
Это и есть knowledge reuse.

**Files = Reliable State**

```
Why files > memory:
✓ Inspectable (человек может читать)
✓ Versionable (git-friendly)
✓ Resumable (restart from any point)
✓ Testable (mock data for testing)
```

---

## Архитектура фреймворка

### Ментальная модель (как ОС)

```
┌─────────────────────────────────────────────────────┐
│                     USER                            │
│                  (slash commands)                   │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│                    ROOT                             │
│              (Claude Code CLI)                      │
│                                                     │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │  Skill  │  │  Skill  │  │  Skill  │  Skills   │
│   │(manager)│  │(atomic) │  │(domain) │  (libs)   │
│   └─────────┘  └─────────┘  └─────────┘           │
│                                                     │
│   ┌─────────────────────────────────────────────┐  │
│   │              Task tool                       │  │
│   │         (spawn subagents)                   │  │
│   └─────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Worker   │  │  Worker   │  │  Worker   │
│ (subagent)│  │ (subagent)│  │ (subagent)│
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │              │              │
      ▼              ▼              ▼
┌───────────────────────────────────────────┐
│                  FILES                     │
│           (L0 → L1 → L2 → L3)             │
│         (persistent state layer)          │
└───────────────────────────────────────────┘
```

### Ключевые принципы

**1. ROOT orchestrates, Workers execute**

ROOT (главный процесс Claude Code) — единственный кто может:
- Spawn subagents (Task tool)
- Load skills (Skill tool)
- Coordinate phases

Workers — изолированные executors:
- Получают task
- Выполняют
- Пишут результат в файл
- Завершаются

**2. Skills are shared libraries**

Skill НЕ агент. Skill — набор инструкций которые загружаются в контекст.

```
Agent A + Skill X = Agent A с knowledge X
Agent B + Skill X = Agent B с knowledge X
```

Типы skills:
| Type | Scope | Example |
|------|-------|---------|
| atomic | Одна операция | tier-weights, slop-check |
| composite | Комбинация atomic | source-evaluation |
| domain | Knowledge pack | training-science, grounding-protocol |
| manager | Workflow instructions | manager-research |

**3. Files are state**

Всё персистится в YAML файлах:
- Reproducible: можно перезапустить с любой точки
- Inspectable: человек может читать и править
- Testable: можно подставить mock data
- Versionable: git-friendly

**4. Layers have contracts**

```
L0 (Config)      →  L1 (Directives)  →  L2 (Operational)  →  L3 (Artifacts)
user_profile.yaml   plan.yaml           aspects/*.yaml       FINAL_REPORT.md
                    directives.yaml     synthesis.yaml
```

Контракты:
- L1 constraints ALWAYS override L2 decisions
- L3 artifacts MUST reference L2 sources
- L2 CAN BE regenerated from L1 + external data

**5. Manager = Skill, not Agent**

Manager не запускается как subagent.
Manager — это skill который ROOT читает и сам выполняет инструкции.

Почему:
- Subagents не могут spawn subagents
- ROOT должен видеть весь pipeline
- Debugging проще когда orchestration в одном месте

---

## Таксономия агентов

### По архитектурной роли

| Role | Execution | Can Spawn | Purpose |
|------|-----------|-----------|---------|
| **commander** | ROOT | Yes | Entry point, slash commands |
| **manager** | ROOT skill | No* | Workflow instructions |
| **worker** | Subagent | No | Isolated task executor |
| **utility** | Inline skill | No | Stateless procedure |

*Manager = Skill loaded into ROOT, ROOT spawns workers.

### По функциональной роли

| Role | Pattern | Example |
|------|---------|---------|
| explorer | Wide search → Structured output | topic-explorer |
| researcher | Queries → Evaluate → Extract | aspect-researcher |
| scorer | Input → Criteria → Scores | source-evaluator |
| aggregator | N inputs → Synthesis | findings-synthesizer |
| validator | Input → Rules → Verdict | quality-gate |
| generator | Context → Template → Artifact | report-generator |
| transformer | Format A → Format B | yaml-to-markdown |
| operator | Command → State change | log-manager |

---

## Data Layers (L0-L3)

| Layer | Name | Mutability | Lifecycle | Example |
|-------|------|------------|-----------|---------|
| **L0** | Config | User editable | Persistent | user_profile.yaml |
| **L1** | Directives | Agent → User approved | Session persistent | plan.yaml |
| **L2** | Operational | Agent generated | Session scoped | aspects/*.yaml |
| **L3** | Artifacts | Append-only | Permanent | FINAL_REPORT.md |

### Layer Contracts

```yaml
L0 → L1:
  trigger: "Strategic review"
  process: "Analyze L0 → Generate directives → User approves"
  validation: "L1 must not contradict L0"

L1 → L2:
  trigger: "Operational command"
  process: "Load L1 constraints → Execute → Write L2"
  validation: "L1 ALWAYS wins over L2 preferences"

L2 → L3:
  trigger: "Quality gate PASS"
  process: "Aggregate L2 → Quality check → Generate artifact"
  validation: "L3 must trace to L2 sources"
```

---

## Orchestration (Deep Dive)

Orchestration — это как ROOT координирует выполнение multi-step pipelines. Это ключевая часть фреймворка.

### Constraint: Flat Hierarchy

```
                    ┌──────────────────────────────────┐
                    │  Claude Code Constraint:         │
                    │  Subagents CANNOT spawn          │
                    │  other subagents                 │
                    └──────────────────────────────────┘
```

Это не баг, это feature. Constraint форсирует архитектуру:

```
✓ ROOT → Worker (allowed)
✗ Worker → Worker (NOT allowed)
```

**Последствия:**
- Вся orchestration logic живёт в ROOT
- Workers = чистые executors, без coordination logic
- Debugging проще: один point of control

### Как обойти constraint: Manager Skills

Проблема: ROOT не знает workflow, но должен orchestrate.

Решение: Manager Skill = инструкции для ROOT.

```
┌────────────────────────────────────────────────────┐
│ ROOT загружает manager-research.md как Skill       │
│                                                    │
│ Manager Skill содержит:                            │
│ - Описание phases                                  │
│ - Условия gates                                    │
│ - Инструкции как spawn workers                     │
│ - State management rules                           │
│                                                    │
│ ROOT читает инструкции и ВЫПОЛНЯЕТ САМ             │
│ (не делегирует другому агенту)                     │
└────────────────────────────────────────────────────┘
```

**Manager ≠ Agent.** Manager — это knowledge pack который ROOT интерпретирует.

### Orchestration Primitives

ROOT имеет три примитива для orchestration:

**1. Task tool (spawn)**
```
Task(
  subagent_type: "general-purpose",
  prompt: "Load X agent, do Y",
  run_in_background: true|false
)
→ Returns: task_id
```

**2. TaskOutput (wait)**
```
TaskOutput(
  task_id: "xxx",
  block: true
)
→ Returns: agent output
```

**3. Skill tool (load knowledge)**
```
Skill(skill: "my-skill")
→ Loads skill instructions into ROOT context
```

### Orchestration Patterns

#### Pattern 1: Fan-out (Parallel Execution)

Когда: N независимых задач которые можно выполнить параллельно.

```
ROOT reads plan.yaml (5 aspects)
  │
  │  ┌─────── Single message with multiple Task calls ───────┐
  │  │                                                        │
  │  │  Task(worker-1, background: true) → task_id_1         │
  │  │  Task(worker-2, background: true) → task_id_2         │
  │  │  Task(worker-3, background: true) → task_id_3         │
  │  │  Task(worker-4, background: true) → task_id_4         │
  │  │  Task(worker-5, background: true) → task_id_5         │
  │  │                                                        │
  │  └────────────────────────────────────────────────────────┘
  │
  │  Wait phase:
  │  TaskOutput(task_id_1, block: true)
  │  TaskOutput(task_id_2, block: true)
  │  ... etc
  │
  ▼
Continue to next phase
```

**Важно:** Все Task calls должны быть в ОДНОМ сообщении для параллельного запуска.

**Код в manager skill:**
```markdown
## Phase 2: Parallel Research

**Orchestration:**
For each aspect in plan.aspects:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load aspect-researcher agent.
      Research: {aspect.name}
      Output: artifacts/{session}/aspects/{aspect.id}.yaml
    run_in_background: true
  )

Wait for all tasks to complete.
```

#### Pattern 2: Pipeline (Sequential Phases)

Когда: Каждая фаза зависит от результата предыдущей.

```
Phase 1        Gate         Phase 2        Gate         Phase 3
┌──────┐    ┌──────┐      ┌──────┐     ┌──────┐      ┌──────┐
│ Plan │───▶│Check │───▶  │Research│───▶│Check │───▶  │Synth │
└──────┘    │exists│      └──────┘     │quality│      └──────┘
    │       └──────┘          │        └──────┘           │
    ▼                         ▼                           ▼
plan.yaml              aspects/*.yaml              synthesis.yaml
```

**Gate = условие перехода между фазами:**
```yaml
gate:
  type: file_exists
  condition: "plan.yaml"

gate:
  type: quality_threshold
  condition: "count(aspects/*.yaml) >= 3"

gate:
  type: custom
  condition: "quality.verdict in ['PASS', 'WARN']"
```

#### Pattern 3: Quality Loop

Когда: Нужно достичь определённого качества, возможно через несколько итераций.

```
┌─────────────────────────────────────────────────┐
│                                                 │
│    ┌──────────┐     ┌───────────┐     ┌─────┐  │
└───▶│ Research │────▶│ Synthesis │────▶│ QG  │──┤
     └──────────┘     └───────────┘     └─────┘  │
                                           │      │
                                    PASS   │      │ FAIL
                                    ───────┘      │
                                                  │
                                    ┌─────────────┘
                                    ▼
                              Gap Analysis
                              (what's missing?)
                                    │
                                    └──── back to Research
                                          with specific gaps
```

**Логика в manager skill:**
```markdown
## Quality Gate Routing

After quality-gate skill returns:

| Verdict | Action |
|---------|--------|
| PASS | Continue to report generation |
| WARN | Continue with caveats noted in report |
| FAIL | Analyze gaps → Re-run research for specific gaps |

On FAIL:
1. Read quality.yaml for missing areas
2. Generate targeted queries for gaps
3. Spawn additional researchers for gap areas only
4. Re-run synthesis with new data
5. Re-check quality
```

#### Pattern 4: Fork-Join

Когда: Несколько parallel branches которые потом merge.

```
              ┌──────────────┐
              │   Planning   │
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Branch  │ │ Branch  │ │ Branch  │
    │   A     │ │   B     │ │   C     │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         └───────────┼───────────┘
                     ▼
              ┌──────────────┐
              │    Merge     │
              │  (synthesis) │
              └──────────────┘
```

**Код:**
```markdown
## Fork Phase
Spawn in parallel:
- Task(branch-a-worker, ...)
- Task(branch-b-worker, ...)
- Task(branch-c-worker, ...)

Wait for all.

## Join Phase
Gate: All branch outputs exist
Action: Synthesis skill merges results
```

#### Pattern 5: Conditional Routing

Когда: Разные paths в зависимости от результатов.

```
              ┌──────────────┐
              │   Analyze    │
              └──────┬───────┘
                     │
              ┌──────┴──────┐
              │  Condition  │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Path A  │  │ Path B  │  │ Path C  │
   │(simple) │  │(medium) │  │(complex)│
   └─────────┘  └─────────┘  └─────────┘
```

**Код:**
```markdown
## Routing Logic

After analysis phase:
- IF complexity_score < 30 → Path A (simple pipeline)
- IF complexity_score < 70 → Path B (standard pipeline)
- ELSE → Path C (full deep research)

Each path has different:
- Number of aspects
- Source requirements
- Quality thresholds
```

### State Management

Orchestration требует state для:
- Resume после interrupt
- Track progress
- Debug failures

**State file structure:**
```yaml
# artifacts/{session}/state.yaml

session_id: "research_20260130_abc"
workflow: "manager-research"
started_at: "2026-01-30T10:00:00Z"
last_updated: "2026-01-30T10:35:00Z"

current_phase: "synthesis"

phase_states:
  planning: completed
  research: completed      # All workers done
  synthesis: in_progress   # Currently running
  quality_gate: pending
  report: pending

# Optional: track individual workers
workers:
  aspect_1: completed
  aspect_2: completed
  aspect_3: failed        # This one failed
  aspect_4: completed
  aspect_5: completed

error: null               # Or error message if failed
```

**State transitions:**
```
pending → in_progress → completed
                     → failed
                     → partial (some workers succeeded)
```

**Resume logic:**
```markdown
## On Resume

1. Read state.yaml
2. Find current_phase
3. Check phase_states:
   - If current_phase is "in_progress" → resume from there
   - If current_phase is "failed" → offer retry options
   - If current_phase is "completed" → move to next
4. Skip all "completed" phases
5. For "partial" phases → only re-run failed workers
```

### Error Handling

#### Worker Failure

```
Worker fails
    │
    ├── Log error to state.yaml
    │
    ├── Continue with other workers
    │
    └── At phase end:
        │
        ├── If completed >= minimum → continue to next phase
        │
        └── If completed < minimum → phase fails
```

**Код:**
```markdown
## Error Handling: Research Phase

On worker failure:
1. Update state: workers.{aspect_id} = "failed"
2. Log error: workers_errors.{aspect_id} = "{error}"
3. Continue with remaining workers

At phase end:
- Count completed workers
- IF completed >= plan.settings.min_aspects → Phase completed
- ELSE → Phase failed, halt pipeline

Recovery option:
- User can run: `/research retry-failed {session}`
- This re-runs only failed workers
```

#### Phase Failure

```
Phase fails
    │
    ├── Update state.yaml with error
    │
    ├── Halt pipeline
    │
    └── Report to user:
        - What phase failed
        - Why it failed
        - What was completed
        - Suggested actions
```

### Observability

Что нужно логировать для debugging:

```yaml
# Per worker
worker_logs:
  aspect_1:
    started_at: timestamp
    completed_at: timestamp
    status: completed
    output_file: "aspects/aspect_1.yaml"
    metrics:
      queries_run: 3
      sources_found: 24
      sources_used: 8

# Per phase
phase_logs:
  research:
    started_at: timestamp
    completed_at: timestamp
    workers_total: 5
    workers_succeeded: 4
    workers_failed: 1
    duration_seconds: 180

# Aggregate
pipeline_metrics:
  total_sources: 32
  total_findings: 48
  quality_score: 0.84
  duration_seconds: 600
```

### Best Practices

**1. Always use state.yaml**

Даже для простых pipelines. Это страховка от interrupt.

**2. Set timeouts**

Workers могут зависнуть. Устанавливай timeout:
```
Task(..., timeout: 300000)  # 5 minutes
```

**3. Minimum thresholds, not exact counts**

```yaml
# Good: flexible
min_aspects_for_synthesis: 3

# Bad: rigid
required_aspects: 5  # Fails if one worker fails
```

**4. Idempotent workers**

Worker который перезапускается должен давать тот же результат.
Проверяй output exists → skip или overwrite.

**5. Clear phase boundaries**

Каждая фаза должна:
- Читать из конкретных файлов
- Писать в конкретные файлы
- Не иметь side effects на другие фазы

---

## Interface Schema (Autogen UI)

Определяет КАК рендерить данные, не сами данные.

```yaml
sources:                              # Data binding
  state: "artifacts/{session}/state.yaml"
  synthesis: "artifacts/{session}/synthesis.yaml"

pages:
  - id: overview
    sections:
      - layout: grid                  # Layout type
        fields:
          - source: synthesis         # Which file
            path: quality_metrics.saturation  # JSONPath
            type: progress_bar        # Widget type
            label: "Saturation"
            threshold: 50
```

Render adapters transform to target:
| Target | Output |
|--------|--------|
| CLI | ASCII tables, progress bars |
| Chat | Markdown |
| API | JSON |
| Web | React components |

---

## Структура фреймворка

```
framework/
├── README.md                         # This file
├── schemas/                          # YAML conventions
│   ├── agent.schema.yaml             # Agent definition
│   ├── skill.schema.yaml             # Skill definition
│   ├── workflow.schema.yaml          # Pipeline definition
│   ├── interface.schema.yaml         # UI definition
│   └── data-layers.schema.yaml       # L0-L3 conventions
├── templates/                        # Starting points
│   ├── agents/
│   │   ├── worker.template.md
│   │   ├── researcher.template.md
│   │   └── generator.template.md
│   └── skills/
│       ├── atomic.template.md
│       ├── composite.template.md
│       ├── domain.template.md
│       └── manager.template.md
└── examples/
    └── research-pipeline/            # Complete working example
```

---

## Использование

### 1. Создать агента

```bash
cp framework/templates/agents/researcher.template.md \
   .claude/agents/my-researcher.md
# Отредактировать под задачу
```

### 2. Создать skill

```bash
cp framework/templates/skills/atomic.template.md \
   .claude/skills/my-skill.md
# Определить input → procedure → output
```

### 3. Создать workflow (manager skill)

```bash
cp framework/templates/skills/manager.template.md \
   .claude/skills/manager-mypipeline.md
# Описать phases, gates, orchestration
```

### 4. Создать interface

```yaml
# .claude/interfaces/dashboard.yaml
sources:
  data: "artifacts/{session}/synthesis.yaml"
pages:
  - id: main
    sections:
      - layout: card
        fields:
          - source: data
            path: key_metric
            type: stat
```

---

## Когда использовать фреймворк

**Подходит для:**
- Multi-step research pipelines
- Data processing workflows (gather → analyze → generate)
- Any task with phases and quality gates
- Projects where intermediate state needs persistence
- Teams that want shared conventions across agents

**НЕ подходит для:**
- Simple single-agent tasks
- Real-time chat applications
- Tasks without intermediate artifacts
- When you need sub-millisecond latency

---

## Adoption Path

### Level 1: Conventions Only
Используй naming conventions и file structure без формальных schemas.
```
.claude/
├── agents/          # Your agents
├── skills/          # Your skills
└── interfaces/      # Your dashboards
artifacts/
└── {session}/       # Runtime outputs
```

### Level 2: Schemas + Templates
Используй templates для consistency. Валидируй структуру агентов и skills.

### Level 3: Full Pipeline
Manager skills для orchestration. Quality gates. State management.

---

## Принципы (TL;DR)

1. **ROOT orchestrates, Workers execute** — no nested spawns
2. **Skills are shared libraries** — knowledge reuse
3. **Files are state** — everything persists to YAML
4. **Layers have contracts** — L1 constrains L2, L2 feeds L3
5. **Interface ≠ Data** — schema describes presentation, not content
6. **Manager = Skill** — workflow instructions for ROOT, not separate agent
