# Research Pipeline Example

Полный пример multi-phase research pipeline на фреймворке.

## Структура

```
research-pipeline/
├── .claude/
│   ├── agents/
│   │   ├── aspect-researcher.md    # Worker: исследует один аспект
│   │   └── report-generator.md     # Worker: генерирует финальный отчёт
│   ├── skills/
│   │   ├── manager-research.md     # Manager: оркестрация pipeline
│   │   ├── research-planner.md     # Atomic: декомпозиция темы
│   │   ├── synthesis.md            # Composite: агрегация findings
│   │   └── quality-gate.md         # Atomic: проверка качества
│   └── interfaces/
│       └── dashboard.yaml          # UI definition
├── workflows/
│   └── research.yaml               # Workflow definition
└── artifacts/                      # Runtime outputs (gitignored)
    └── {session_id}/
        ├── state.yaml
        ├── plan.yaml
        ├── aspects/*.yaml
        ├── synthesis.yaml
        ├── quality.yaml
        └── FINAL_REPORT.md
```

## Запуск

```
/research "AI agents orchestration patterns"
```

## Pipeline

```
┌─────────────┐     ┌─────────────────┐     ┌───────────┐
│  Planning   │────▶│ Parallel Research│────▶│ Synthesis │
│  (planner)  │     │ (N researchers)  │     │           │
└─────────────┘     └─────────────────┘     └─────┬─────┘
                                                  │
                    ┌─────────────────┐     ┌─────▼─────┐
                    │  Report Gen     │◀────│  Quality  │
                    │  (generator)    │     │   Gate    │
                    └─────────────────┘     └───────────┘
```

## Фазы

| Phase | Agent/Skill | Input | Output |
|-------|-------------|-------|--------|
| 1. Planning | research-planner | topic | plan.yaml |
| 2. Research | aspect-researcher ×N | plan.aspects | aspects/*.yaml |
| 3. Synthesis | synthesis | aspects/*.yaml | synthesis.yaml |
| 4. Quality | quality-gate | synthesis.yaml | quality.yaml |
| 5. Report | report-generator | synthesis.yaml | FINAL_REPORT.md |

## Data Flow

```yaml
L1 (Directives):
  - plan.yaml           # Decomposed topic, constraints

L2 (Operational):
  - aspects/tech.yaml   # Research findings per aspect
  - aspects/market.yaml
  - synthesis.yaml      # Aggregated insights
  - quality.yaml        # Quality verdict

L3 (Artifacts):
  - FINAL_REPORT.md     # Deliverable
```
