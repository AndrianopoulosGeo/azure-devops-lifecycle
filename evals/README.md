# Eval Suite — Portable Dev Lifecycle Commands

## Eval Files

| File | Type | Commands Tested | Test Cases |
|------|------|----------------|------------|
| `trigger-evals.json` | Trigger accuracy | All 10 commands | 20 scenarios |
| `validate-env-evals.json` | Execution | `/validate-env` | 2 scenarios |
| `next-evals.json` | Execution | `/next` | 3 scenarios |
| `quick-fix-evals.json` | Execution | `/quick-fix` | 2 scenarios |
| `staging-release-evals.json` | Execution | `/staging`, `/release` | 3 scenarios |
| `hotfix-evals.json` | Execution | `/hotfix` | 2 scenarios |

## Running Evals

### Trigger Evals (test command selection accuracy)

Use the skill-creator plugin:
```
/skill-creator run trigger evals for validate-env using evals/trigger-evals.json
```

### Execution Evals (test command behavior)

Use the skill-creator plugin:
```
/skill-creator run evals for validate-env using evals/validate-env-evals.json
```

### Full Benchmark

Run all evals and aggregate:
```
/skill-creator benchmark all commands using evals/
```

## Metrics to Track

| Metric | Target | How to Measure |
|--------|--------|---------------|
| Trigger accuracy | >90% | Trigger evals — correct command invoked |
| State management | 100% | Every command reads/writes .state.md |
| Config portability | 100% | All commands use .env.claude, not hardcoded |
| Commit hygiene | 100% | No AI attribution in any commit |
| Track completion | >80% | Full track runs without manual intervention |
| Pipeline integration | >70% | Pipeline triggered and monitored correctly |

## Eval Schedule

- After modifying any command: run its execution eval
- After modifying command descriptions: run trigger evals
- Weekly: full benchmark run to detect drift
