# highload-tickets

Учебный проект по курсу Highload (ИТМО): сервис продажи билетов, стенд на Beelink (KVM + k3s), нагрузка k6.

## Структура

| Папка | Что там |
|---|---|
| `services/` | сервисы (Gradle multi-project) |
| `deploy/` | Helm-чарты: приложение, мониторинг, Postgres |
| `infra/` | хост Beelink, VM, k3s |
| `load/` | сценарии k6, генератор данных, проверки инвариантов |
| `docs/` | требования, архитектура, ADR, эксперименты, runbook |
| `.github/workflows/` | CI |

## Правила коммитов

Формат — [Conventional Commits](https://www.conventionalcommits.org/ru/v1.0.0/):

```
<type>(<scope>): <что сделано>

[зачем / контекст — по желанию]

[Closes #N]
```

**type**

| type | когда |
|---|---|
| `feat` | новая функциональность сервиса |
| `fix` | исправление бага |
| `perf` | оптимизация производительности |
| `refactor` | переделка без изменения поведения |
| `test` | тесты |
| `docs` | документация, ADR, отчёты |
| `infra` | хост, сеть, VM, k3s |
| `deploy` | Helm-чарты, манифесты |
| `load` | сценарии k6, генератор данных, проверки |
| `ci` | GitHub Actions |
| `build` | Gradle, Dockerfile/Jib, Makefile |
| `chore` | прочее (gitignore, зависимости, форматирование) |

**scope** — по желанию, уточняет место: имя сервиса (`booking`), компонент (`network`, `postgres`, `grafana`), этап (`stage0`).

**Описание**

- на английском, в повелительном наклонении: `add`, `fix`, `remove` (не `added`, `fixes`);
- с маленькой буквы, без точки в конце, до 72 символов;
- один коммит — одно логическое изменение.

**Связь с задачами** — в футере `Closes #N` (закрывает issue) или `Refs #N` (только ссылка).

**Ломающие изменения** — `!` после type: `feat(api)!: change booking response format`.

**Примеры**

```
chore: add readme and gitignore
docs: add stand and delivery overview
infra(network): add br-lab bridge and nat scripts
infra(vm): add cloud-init vm creation script
deploy(monitoring): install kube-prometheus-stack
load: add buyer scenario with constant arrival rate
perf(booking): add index on seats(event_id, status)
docs(experiments): add v0 baseline report

Closes #7
```

**Ветки** — работа в `master`; для крупных изменений — ветка `<type>/<N>-<кратко>` (например `infra/3-br-lab`) и PR с `Closes #N`.
