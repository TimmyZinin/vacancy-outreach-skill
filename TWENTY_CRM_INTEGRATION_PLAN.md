# Twenty CRM → Vacancy Outreach — План интеграции

**Version:** 6.0 (v5.4 + Sprint Protocol раздел с Codex gate в каждом спринте)
**Date:** 2026-04-11
**Цель:** Развернуть Twenty CRM (self-hosted) на Contabo VPS 30 и сделать его частью workflow для любого взаимодействия с HH.ru.

**Приоритеты:** простота > фичи, честность формулировок > громких claims, measurable system properties > circular процессы.

---

## 0. Что мы реально гарантируем (честно)

### Guarantee Matrix (v3)

| Класс действия | Гарантия |
|---|---|
| Любой запуск `hh-cdp.mjs` с URL `hh.ru/*` | Durable WAL-запись с `fsync` **до** CDP-вызова. WAL — first-class persistent log. |
| Любая попытка вызвать `cdp.mjs` / WebFetch / Playwright / Safari+osascript / `open -a Chrome` / Jina / curl к `hh.ru` через tool call Claude | Блокируется PreToolUse хуком `hh-touch-guard.py` с **narrow matcher** `Bash\|WebFetch\|mcp__playwright__.*\|mcp__computer-use__.*` (Codex NEW-3: не catch-all чтобы не добавлять latency на Read/Edit/Glob/Grep). |
| `mcp__computer-use__screenshot` когда браузер уже на hh.ru | **Не блокируется технически**, но hook блокирует `mcp__computer-use__open_application` и `type`/`click` с hh.ru в args. Привычка: не использовать computer-use для HH — записано в SKILL.md и в attestation warning. |
| Параллельный процесс или другой macOS-пользователь, который напрямую гоняет Chrome DevTools | **Не контролируем.** Out of scope: мы гарантируем поведение Claude-сессий, не всей ОС. |
| WAL запись → Twenty | **Eventually consistent, no silent loss.** Drain раз в 60с досылает пачками. Validation failure → fallback payload → если и он fails → `.failed.permanent` + mandatory runbook + attestation блокирует новые runs пока не разрешено. Не "no terminal state" (Codex NEW-8), а "guaranteed human-in-the-loop reconciliation". |
| Файловая целостность wrapper + hook | SHA256 hash залочен в `~/.secrets/twenty-integrity.json` — отдельно для каждого файла `hh-cdp.mjs`, `hh-touch-guard.py`, `twenty-client.mjs`, `twenty-preflight.mjs`, плюс **конкретный hook stanza** из settings.json (не целый блок — Codex NEW-2). Attestation при каждом запуске сверяет. Несовпадение → fail-closed. Trust root: `~/.secrets/` chmod 600 + KeePassXC backup hash. |

**Что мы НЕ утверждаем:**
- ❌ "Bypass физически невозможен" — неправда, пользователь-админ машины всегда может отключить хук.
- ❌ "Twenty — синхронный source of truth" — неправда, это view на WAL с задержкой.
- ❌ "Все HH-взаимодействия включая пассивное наблюдение экрана" — неправда, мы покрываем tool-initiated flow, не passive vision.

**Что мы утверждаем:**
- ✅ Любой tool-initiated touch на `hh.ru` из Claude-сессии → durable WAL + eventual Twenty sync.
- ✅ File-integrity attestation ловит любую подмену wrapper'а или удаление хука, запускается в начале каждого HH-скрипта.
- ✅ `.failed.permanent` не существует как терминальное состояние — каждая неудачная запись реконсилируется либо через retry, либо через fallback payload, либо через явный runbook. Нет молчаливой потери.

---

## 1. Контекст Twenty

| Критерий | Значение |
|---|---|
| Лицензия | AGPL, self-hosted, $0 |
| Зрелость | v1.21.0 (09.04.2026), 43.8K★, 62 релиза |
| Стек | NestJS + Postgres 16 + Redis |
| API | REST `/rest/*`, Bearer, **100 req/min**, **max 60 записей/batch** |
| Custom objects/fields | `/rest/metadata/*` |
| RAM | 2 GB min (на сервере 18 из 23 GB свободно) |
| Порт | `127.0.0.1:3003:3000` (Umami уже на 3000) |

---

## 2. Архитектура

```
 ┌────────────── Mac (пользовательская сессия) ───────────┐
 │                                                         │
 │  hh-outreach скрипты (hh-send-batch, hh-read-desc, ...) │
 │            │                                            │
 │            │ (hh-check-env.mjs self-check)              │
 │            ▼                                            │
 │  hh-cdp.mjs  (единственная точка входа)                │
 │     1. assert SOURCE_SCRIPT                             │
 │     2. write WAL {tmp} → fsync → rename               (POSIX atomic)
 │     3. spawn cdp.mjs <cmd> <args>                       │
 │     4. append result line → fsync                       │
 │     5. return stdout                                    │
 │            │                                            │
 │            ▼                                            │
 │  ~/hh-outreach-data/twenty-wal/*.json                  │
 │            │                                            │
 │            │ read                                       │
 │            ▼                                            │
 │  twenty-drain.mjs (launchd user agent, StartInterval 60)│
 │     ├─ atomic O_EXCL lockfile single-instance           │
 │     ├─ batch up to N (request-budget, не count)         │
 │     ├─ upsert → batch create touches                    │
 │     ├─ rename → .sent                                   │
 │     └─ reconcile .failed через fallback payload          │
 │                                                         │
 │  ~/.claude/hooks/hh-touch-guard.py  (PreToolUse, .*)   │
 │     блок для: Bash / WebFetch / Playwright MCP /        │
 │     computer-use (open/type/click с hh.ru)              │
 │                                                         │
 │  ~/.secrets/twenty-integrity.json — SHA256 pin          │
 └────────────┬────────────────────────────────────────────┘
              │ HTTPS
              ▼
 ┌──────────── Contabo VPS 30 ─────────────┐
 │  nginx 443 SSL + certbot renewal hook    │
 │  docker compose:                         │
 │    twenty-server  :3000 (internal)      │
 │    twenty-worker                         │
 │    twenty-db   Postgres 16              │
 │    twenty-redis                          │
 │  cron: pg_dump daily → /opt/backups/    │
 └──────────────────────────────────────────┘
```

---

## 3. Модель данных

### 3.1. Company

| Field | Type | Constraint | Source |
|---|---|---|---|
| `name` | text | — | из HH |
| `hhEmployerId` ⭐ | text | **isUnique:true, isNullable:false** | `/employer/{id}` или `orphan-<uuidv5(name)>` если id нет |
| `hhEmployerType` ⭐ | select: `ooo/ip/fio/other` | — | |
| `niche` ⭐ | select (11 значений) | — | |
| `city` ⭐ | text | — | |

**Race fix (Codex NH5, NEW-6, R3, I5-1 — order-stable):** `hhEmployerId` всегда НЕ null. Если HH не дал employer_id — используется **стабильный id независимый от порядка обработки вакансий**.

```js
// ~/hh-outreach/scripts/lib/deterministic-id.mjs
import { createHash } from 'node:crypto';
const NS_EMP = 'hh-employer-orphan-v1';
export function orphanEmployerId(rawName) {
  const norm = rawName.trim().toLowerCase()
    .replace(/^(ооо|ао|зао|пао|ип|тоо|llc|оао|ано|фгбу)\s+/i, '');
  const h = createHash('sha256').update(`${NS_EMP}|${norm}`).digest('hex');
  return `orphan-${h.substring(0, 32)}`;
}
```

**Key property (I5-1 fix):** id зависит **только** от normalized name. Вакансии разного порядка дают одинаковый id → no duplicate company, no order dependency.

**Collision tradeoff:** два разных employer с полностью одинаковым normalized name (редкий случай — "ООО Ромашка" где две несвязанные компании) получат **один и тот же** orphan id. Это **не тихая потеря**, а детектируемая аномалия:

**Orphan registry (Codex I6-1 fix):** Для каждого orphan-id ведётся persistent registry `~/hh-outreach-data/orphan-registry.json`:

```json
{
  "orphan-<hash>": {
    "normalizedName": "ромашка",
    "displayNames": ["ООО Ромашка", "Ромашка"],
    "vacancies": ["131695122", "132024133", ...],
    "firstSeenAt": "...",
    "reviewRequired": false
  }
}
```

- При каждом upsert orphan company — atomic append в registry (O_EXCL lock).
- Обновляются `displayNames` (Set) и `vacancies` (Array).
- **Flag `reviewRequired: true` устанавливается** если:
  - `vacancies.length > 1` И вакансии имеют признаки разных компаний (разные `city`, разные `website domain` в description, разные зарплатные категории вне 2x разброса) — **heuristic**, не factual
  - ИЛИ если вручную triggered через `twenty-mark-orphan.mjs <id>`
- Любой `reviewRequired: true` orphan без matching resolved line в `docs/orphan-audit.md` → attestation Sprint 9 **step 14** fails → fail-closed.
- **Sprint 9 step 14 reads ONLY `orphan-registry.json`** (canonical source). Нет отдельных `orphan-reconciliation.log`, `orphan-collision.log` — unified в registry.

**Runbook `docs/orphan-reconciliation.md`:** прочитать registry entry, сравнить vacancies вручную (просмотреть URL), принять решение:
- **Merge** (та же компания в разных написаниях) — `twenty-mark-orphan.mjs <id> --resolve merge` → clears reviewRequired
- **Split** (разные бизнесы) — `twenty-mark-orphan.mjs <id> --resolve split --vacancy <vid> --new-suffix <manual>` → создаёт новую Company с `hhEmployerId = orphan-<sha256(name|suffix)>`, перевешивает указанную vacancy, clears reviewRequired

**Сводка гарантий:**
- Та же компания, любой порядок vacancy → один stable id (I5-1 fixed).
- Все orphan companies tracked в registry, все проверяются attestation.
- Detectable heuristic для suspicious merges → `reviewRequired: true` → attestation block.
- Silent merging разных бизнесов **возможен** только если heuristic не сработал (одинаковый город, бюджет, домен) — тогда human review через monthly audit `docs/orphan-audit.md`.

Node native `crypto`, без внешних зависимостей.

### 3.2. Opportunity

| Field | Type | Constraint | Source |
|---|---|---|---|
| `name` | text | — | title |
| `stage` | 6-stage pipeline | — | mapping |
| `companyId` | FK | — | |
| `hhVacancyId` ⭐ | text | **isUnique:true, isNullable:false** | `131695122` или `orphan-<sha1(url+title)[:32]>` (см. `lib/deterministic-id.mjs`) |
| `hhVacancyUrl` ⭐ | url | — | |
| `salaryRaw` ⭐ | text | — | |
| `salaryAmount` ⭐ | number nullable | — | RUB |
| `templateUsed` ⭐ | text | — | `BOLD_FOUNDER` |
| `messagePreview` ⭐ | text(500) | — | |
| `firstTouchAt` ⭐ | datetime | — | |
| `lastTouchAt` ⭐ | datetime | — | |
| `replyReceivedAt` ⭐ | datetime nullable | — | |
| `replyType` ⭐ | select: `invite/reject/question/spam/none` | — | |

### 3.3. Pipeline stages
`New → Sent → Delivered → Replied → Meeting → Won / Lost`

### 3.4. HhTouch (custom object)

Пишется на КАЖДЫЙ `hh-cdp.mjs`-вызов с hh.ru URL. Без heuristics, без фильтров.

| Field | Type | Notes |
|---|---|---|
| `occurredAt` | datetime | |
| `touchType` | select: `vacancy_view / employer_view / resume_view / search_view / negotiations_view / message_sent / reply_read / other / migrated / synthetic` | `synthetic` — attestation tests. `migrated` — миграция. |
| `cdpCommand` | text | nav / eval / type / clickxy / shot / html / snap |
| `hhUrl` | url nullable | |
| `hhEntityId` | text nullable | |
| `opportunityId` | FK nullable | |
| `companyId` | FK nullable | |
| `sourceScript` | text required | |
| `resultSummary` | text(200) | |
| `rawPayloadRedacted` | text(2000) | **whitelist only** — см. 3.5 |
| `walFile` | text | basename для traceability |

### 3.5. WAL privacy (Codex NH3)

`rawPayloadRedacted` = результат whitelist-трансформации, не raw dump:

**Whitelisted ключи** (одобренные для payload):
- `vacancyTitle`, `employerName`, `salaryText`, `statusText` (`Резюме доставлено` и т.п.)
- `elementText` для проверенных селекторов (`h1[data-qa="vacancy-title"]`, etc.)
- `cdpCommand`, `cdpArgs[0]` (tab id — не секрет)

**Blacklist (strip regex, применяется ДО whitelist):**
- `Bearer\s+\S+`, `api[_-]?key[=:]\s*\S+`, `Authorization:\s*\S+`
- `Cookie:\s*[^\r\n]+`, `Set-Cookie:\s*[^\r\n]+`, `csrftoken=\S+`, `sessionid=\S+`
- Email: `[\w.-]+@[\w.-]+\.\w+`
- RU phone: `\+?7[\s(-]?\d{3}[\s)-]?\d{3}[\s-]?\d{2}[\s-]?\d{2}`
- Любые `<input type=password>` значения
- Полный `document.body.innerText` — максимум 200 символов, в остальных случаях drop

**Retention:** `.sent` WAL файлы удаляются после 14 дней **daily** cleanup job (Sprint 6.5 создаёт). `twenty-wal/` chmod 700. (Codex NEW-7: inconsistency "daily vs weekly" резолвится — везде **daily**.)

---

## 3.9. Sprint Protocol — 11 шагов на каждый спринт

Каждый спринт (S0..S12) проходит один и тот же workflow. Внутри каждого шага есть измеримый gate.

```
1. ENTER  →  2. PLAN  →  3. ADVERSARIAL AUDIT  →  4. IMPLEMENT  →  5. TEST
     →  6. CODE REVIEW  →  7. SEC  →  8. DEPLOY  →  9. VERIFY  →  10. LOG  →  11. EXIT
```

### 1. ENTER (~2 мин)
- Прочитать `memory/session_handoff.md` — убедиться что предыдущий спринт закрыт
- Прочитать `memory/logs/hh-outreach.md` последнюю запись
- `git status` чистый
- Прочитать секцию текущего спринта в этом плане (source of truth)
- **Gate:** предыдущий спринт должен иметь `post-S{N-1}-verified` git tag, иначе стоп

### 2. PLAN (~3 мин)
- Микро-план этого спринта в TodoWrite
- Dependencies между задачами
- Expected evidence для каждой
- **Gate:** 3-8 задач; если >10 — дробить спринт

### 3. ADVERSARIAL AUDIT — Codex gate №1 (~5 мин)
- Запустить `codex-adversarial` на план спринта и все предварительные решения
- Codex атакует trade-offs, failure modes, simpler alternatives
- **Gate:** если findings CRITICAL/HIGH — пересмотр плана + повторный adversarial (max 3 попытки)
- Если всё OK или только MEDIUM/LOW — идём в IMPLEMENT
- Threshold: субъективный, главное — CRITICAL/HIGH = 0

### 4. IMPLEMENT
- Пишу код строго по plan spec из этого документа
- Минимальные изменения, без over-engineering
- Atomic commits: одна задача = один commit
- TodoWrite обновляется в процессе

### 5. TEST
- Unit-тесты для нового кода (пишу одновременно)
- Запуск: `npm test` / `node file.test.mjs`
- **Gate:** 0 failed

### 6. CODE REVIEW — Codex gate №2 (~8 мин)
- `codex-review` на реальный diff спринта
- Проверка: correctness, security, edge cases, tests
- **Gate промежуточный (S0..S10):** score ≥ 9/10
- **Gate финальный (S12):** score = 10/10 на полный diff проекта
- Если < threshold → фикс → повторный review (до 3 попыток)

### 7. SEC (IL-1, ~2 мин)
5-пунктный чеклист из конституции:
- [ ] Новые API endpoints → auth?
- [ ] SQL → параметризованный?
- [ ] Новые deps → `npm audit` clean?
- [ ] .env в `.gitignore`?
- [ ] Path/recipient ограничения?

Пропускается если спринт не добавляет кода (S0, S7, S10).

### 8. DEPLOY
- Локальные спринты: `git push` в repo
- Серверные спринты (S1, S2, S8): SSH + docker / API вызов / миграция на прод
- Cron спринты (S6, S6.5, S8.5): `launchctl load`
- **Gate:** deploy command exit 0

### 9. VERIFY (IL-2)
Реальные evidence, не декларация:
- `curl -sI` на эндпоинты → HTTP 200/201
- `docker compose ps` → healthy
- `node test.mjs` → `all green`
- `launchctl list` → loaded
- Вывод Codex review (если был в спринте)
- **Gate:** IL-2 — никакого "готово" без fresh evidence

### 10. LOG
- `memory/logs/hh-outreach.md` → `[HH:MM] S{N} done — {evidence summary}`
- `memory/session_handoff.md` → append new block (не перезаписывать)
- TodoWrite: этот спринт → completed
- Git tag `post-S{N}-verified`
- git commit с осмысленным message `feat(twenty): S{N} {title}`

### 11. EXIT
- HITL-пауза? (см. таблицу ниже)
- TG notify если крупный milestone
- Переход к S{N+1}

---

## 3.10. Матрица Codex gate по спринтам

| Спринт | Adversarial фокус | Code review фокус | Min score | HITL |
|---|---|---|---|---|
| **S0** | Безопасность секретов, DNS риски | Config files only | adv: clean | — |
| **S1** | nginx/certbot bootstrap order, Docker isolation, fallback paths | nginx config, docker-compose, .env | 9 | ⏸ после bootstrap |
| **S2** | Schema trade-offs, uniqueness, API key lifecycle | `twenty-schema.mjs`, metadata API calls | 9 | — |
| **S3** | Rate limiter race conditions, retry strategy | SDK code, unit tests, O_EXCL lock | 9 | — |
| **S4** | WAL consistency, integrity bypass, fail-closed logic | Wrapper, fsync, subprocess spawn | 9 | — |
| **S5** | ALL bypass paths (browser/MCP/shell/computer-use) | Hook Python, regex, JSON walker | 9 | — |
| **S6** | Reconciliation, poison messages, budget exhaustion | Drain logic, batch accounting | 9 | — |
| **S6.5** | Retention edge cases, `.failed.*` handling | Shell script, find args | 9 | — |
| **S7** | Reversibility, orphan scripts, archive pattern | File moves, preflight imports | 9 | ⏸ перед архивацией |
| **S8** | Mapping correctness, edge cases, idempotency | Migration script, dry-run vs commit | 9 | ⏸ после dry-run |
| **S8.5** | Expiry alert reliability, upgrade governance | launchd plists, shell scripts | 9 | — |
| **S9** | 16 checks полнота, false positives, cost per run | `hh-check-env.mjs`, integration point | 9 | — |
| **S10** | DoD completeness, measurable criteria | (verification script) | 9 | ⏸ перед разморозкой |
| **S12** | **Architecture — что сломается первым в prod** | **Полный diff pre→post-cutover** | **10** | ⏸ финальный |

**HITL-паузы** (по умолчанию работаю автономно, пауза только в 5 точках):
1. После S1 — ты смотришь Twenty UI в браузере, подтверждаешь живой
2. После S7 — перед точкой невозврата
3. После S8 dry-run — ты смотришь `migration-preview.json`, подтверждаешь commit
4. После S10 — перед разморозкой HH-pipeline и отправкой первых откликов через CRM
5. S12 финал — ты видишь Codex 10/10 verdict

Остальные 9 спринтов прохожу без пауз. Оповещение в TG только о крупных milestones (S1, S4, S5, S6, S8 commit, S10, S12).

---

## 4. Спринты

### Sprint 0: Подготовка (15 мин)

- [ ] DNS A `crm.timzinin.com` → `185.202.239.165`
- [ ] Секреты в KeePassXC: `APP_SECRET`, `PG_DATABASE_PASSWORD`
- [ ] Backup `sent.json` → `sent.json.pre-twenty-20260411`
- [ ] Создать `TimmyZinin/twenty-hh-integration` (private)
- [ ] Git tag в `hh-outreach` repo: `pre-twenty-cutover`

**Verify:** `dig +short crm.timzinin.com` = `185.202.239.165`.

### Sprint 1: Deploy Twenty — two-phase bootstrap (45 мин)

На сервере:

1. `/opt/twenty/`:
```bash
mkdir -p /opt/twenty && cd /opt/twenty
curl -sLO https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-docker/docker-compose.yml
curl -sLO https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-docker/.env.example
mv .env.example .env
```

2. `.env`:
```
TAG=v1.21.0
APP_SECRET=<32 bytes base64>
PG_DATABASE_PASSWORD=<24 bytes base64>
SERVER_URL=https://crm.timzinin.com
STORAGE_TYPE=local
SIGN_IN_PREFILLED=false
IS_SIGN_UP_DISABLED=false          # temporary — for bootstrap
```

3. `docker-compose.yml` правки:
   - `server.ports`: `"127.0.0.1:3003:3000"`
   - `restart: unless-stopped` везде
   - `db` и `redis`: никаких `ports` (только internal network)

4. **Phase A — nginx HTTP-only:**
```nginx
server {
    listen 80;
    server_name crm.timzinin.com;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 404; }
}
```
`mkdir -p /var/www/certbot && nginx -t && systemctl reload nginx`

5. **Issue cert (Codex H2):**
```bash
certbot certonly --webroot -w /var/www/certbot \
  -d crm.timzinin.com \
  --non-interactive --agree-tos -m tim.zinin@gmail.com \
  --deploy-hook "systemctl reload nginx"
```
Параметр `--deploy-hook` (Codex NH6) автоматически reload-ит nginx после каждого renewal. Проверка: `systemctl cat certbot.timer` — ожидается daily enabled.

6. **Phase B — nginx HTTPS:** (full server block из плана v2 + добавить `error_log /var/log/nginx/crm.err.log warn;`)

7. **Start Twenty:**
```bash
cd /opt/twenty && docker compose up -d
docker compose logs -f server   # ждать "Server ready"
```

8. **Phase C — bootstrap user, then disable signup:**
   - Открыть `https://crm.timzinin.com` → Sign up → `tim@timzinin.com`
   - `docker compose down` (сохраняет volumes)
   - `.env`: `IS_SIGN_UP_DISABLED=true`
   - `docker compose up -d`
   - Проверить signup недоступен

**Verify:**
```bash
curl -sI https://crm.timzinin.com/healthz              # 200
curl -sI https://crm.timzinin.com/                     # 200
# Renewal hook wired
grep deploy-hook /etc/letsencrypt/renewal/crm.timzinin.com.conf  # should match
```

**Rollback (Codex M1, non-destructive):**
```bash
cd /opt/twenty && docker compose stop    # volumes preserved
# Data wipe (только вручную после подтверждения):
# pg_dump ... > /opt/backups/twenty-emergency.sql.gz
# docker compose down    # (без -v)
# docker volume ls / rm — решение отдельно
```

### Sprint 2: Schema + API key (30 мин)

1. **UI:** Settings → APIs & Webhooks → Create key `hh-integration-key`, expires 2027-04-11. Сохранить в `~/.secrets/twenty.env`:
```
TWENTY_API_URL=https://crm.timzinin.com
TWENTY_API_KEY=<...>
TWENTY_KEY_EXPIRES=2027-04-11
```
`chmod 600 ~/.secrets/twenty.env`. Создаётся вручную **один раз** — MVP, reproducibility через export/import позже (Codex NM4 accepted).

2. Скрипт `twenty-schema.mjs` идемпотентно создаёт все custom fields через `/rest/metadata/*`:
   - Company: `hhEmployerId {isUnique:true, isNullable:false}`, `hhEmployerType`, `niche`, `city`
   - Opportunity: `hhVacancyId {isUnique:true, isNullable:false}`, все другие из 3.2
   - HhTouch object с enum `touchType` (10 значений включая `migrated` и `synthetic`)
   - Все Opportunity stages: `New / Sent / Delivered / Replied / Meeting / Won / Lost`

3. Проверка:
```bash
source ~/.secrets/twenty.env
curl -sH "Authorization: Bearer $TWENTY_API_KEY" \
  "$TWENTY_API_URL/rest/metadata/fields?filter=name[eq]:hhVacancyId" \
  | jq '.data[0] | {name,isUnique,isNullable}'
# → {name:"hhVacancyId", isUnique:true, isNullable:false}
```

### Sprint 3: `twenty-client.mjs` с request-budget (45 мин) (Codex NH1)

Публичные функции:
- `upsertCompany(payload)` — 1 request (POST → 409 → PATCH max 2 calls, in practice 1 call by `external_id` endpoint semantics)
- `upsertOpportunity(payload)` — аналогично
- `createTouchesBatch(records[])` — 1 request на до 60 записей
- `flushQueue(records[])` — оркестратор для drain, возвращает budget используемый
- `healthCheck()`

**Request budget math (Codex H5, NH1, NEW-1):**
- Twenty limit: 100 req/min
- Safety margin: 80% → **80 req/min effective budget**
- **Cross-process rate limiter** (Codex NEW-1, R2 fix — macOS-native): shared state file + atomic lock без `flock` (которого нет на macOS).
  - Lockfile: `~/hh-outreach-data/twenty-rate-budget.lock`
  - Lock acquire: `fs.openSync(lockPath, 'wx')` — O_CREAT|O_EXCL, atomic. Если EEXIST → retry 250ms.
  - Lock release: `fs.unlinkSync(lockPath)` + `fs.closeSync(fd)`
  - Stale lock detection: file mtime > 30s → force remove + retry
  - State file: `~/hh-outreach-data/twenty-rate-budget.json` с `{ windowStartMs, used }`
- Flow для любого процесса, использующего `twenty-client`:
  1. acquire lock (atomic open)
  2. Read state JSON. Если `now - windowStartMs >= 60000` → reset (`used=0`, `windowStartMs=now`)
  3. Если `used + needed > 80` → release lock, `sleep 250ms`, retry
  4. `used += needed`, atomic write (`tmp + rename`)
  5. Release lock
  6. Make HTTP request
- **Все HTTP-вызовы** через `twenty-client` (включая `healthCheck()` в preflight — Codex R8) проходят через лимитер. Нет байпасов.
- Корректно работает для N параллельных процессов (drain + migrate + preflight + ручные hh-скрипты).

**Drain batching rule (Codex NH1 resolve):**
- Drain батчит ЗАПИСИ WAL и считает **reserved request cost** не по количеству touch, а по real request count:
  - Уникальные Companies × 1 req
  - Уникальные Opportunities × 1 req
  - Chunks of HhTouch (60 each) × 1 req
- Если `reserved > bucket.available` → не берём пачку, ждём refill.
- Worst case per drain run: **max 30 req** (10 companies + 10 opps + up to 10 HhTouch batches = 600 touches).
- Никогда не превышаем 30 req / 60 sec = 30 req/min.

**Retry:** 3 попытки с 500ms / 2s / 8s. На 429 — extra wait `Retry-After`. На 5xx — retry. На 4xx (validation) — NOT retry, mark as requiring reconciliation (см. Sprint 6 reconciliation).

**Unit tests:** 12 минимум (happy path, 409, 429 with Retry-After, 5xx retry exhausted, 400 validation, timeout, batch chunking, token bucket exhaustion, concurrent calls lock, etc.).

### Sprint 4: `hh-cdp.mjs` (60 мин)

**CLI surface (Codex R4):**

Normal CDP proxy commands (проксируются в реальный `cdp.mjs`):
```
hh-cdp.mjs nav     <tab> <url>
hh-cdp.mjs eval    <tab> <expr>
hh-cdp.mjs type    <tab> <text>
hh-cdp.mjs shot    <tab> [file]
hh-cdp.mjs clickxy <tab> <x> <y>
hh-cdp.mjs html    <tab> [selector]
hh-cdp.mjs click   <tab> <selector>
hh-cdp.mjs snap    <tab>
```

Service modes:
```
hh-cdp.mjs --integrity-check     # sha256 check, exit 0/5
hh-cdp.mjs --wal-selftest        # local-only test in twenty-wal-selftest/
hh-cdp.mjs --version             # print wrapper version + pinned cdp.mjs sha
```

**`--wal-selftest` spec (Codex R4):**
- Writes a file to `~/hh-outreach-data/twenty-wal-selftest/{ts}-{random}.json`
- Does NOT spawn cdp.mjs, does NOT call Twenty, does NOT write to main WAL directory
- Validates: fsync works, rename works, file readable, content matches
- Deletes file before exit
- Output: `selftest ok: wrote+read+deleted in Nms`
- Exit 0 on success, exit 4 on any fs error

**`twenty-drain.mjs` MUST scan only `~/hh-outreach-data/twenty-wal/*.json` — explicit exclude of `twenty-wal-selftest/` via `path.basename(dirname) === 'twenty-wal'` check.** Sprint 9 attestation verifies.

**Pre-flight (run ONCE при import):**
```js
import { execSync } from 'child_process';
try {
  execSync('node ~/hh-outreach/scripts/hh-check-env.mjs --silent --no-touch', { stdio:'inherit' });
} catch (e) {
  console.error('attestation failed — refuse to proceed');
  process.exit(9);
}
```

**Runtime logic:**
```
1. ASSERT process.env.SOURCE_SCRIPT (whitelisted list, see hh-check-env)
2. Parse argv: cmd, target, args
3. Resolve CURRENT URL of target tab via cdp.mjs eval 'location.href'
   (for nav cmd — use the target URL arg)
4. Check if URL matches hh.ru/(vacancy|resume|search|applicant|employer|negotiations)
5. If yes:
   a. Construct WAL record {schemaVersion, occurredAt (ISO), sourceScript,
      cdpCommand, hhUrl, hhEntityId (extracted), cdpArgsRedacted (sanitize),
      walFile}
   b. WAL_DIR = ~/hh-outreach-data/twenty-wal
   c. tmp = `${WAL_DIR}/${ts}-${nanoid}.json.tmp`
   d. fs.writeFileSync(tmp, JSON.stringify(record))
   e. fs.fsyncSync(fd)
   f. fs.renameSync(tmp, final)    # POSIX atomic same-filesystem rename
   g. Any error → process.exit(4, "WAL_WRITE_FAILED")
6. Spawn cdp.mjs <same args> via child_process, capture stdout/stderr
7. If WAL was written, append second line to WAL file:
   {"result": redacted result, "exitCode", "durationMs"}
   fsync + return
8. stdout original result
```

**File integrity lock (Codex NC1, NEW-2):**
`hh-cdp.mjs` при запуске вычисляет SHA256 своего содержимого и сверяет с `~/.secrets/twenty-integrity.json`. Несовпадение → exit 5 "integrity check failed". Записи в integrity файле:
```json
{
  "files": {
    "hh-cdp.mjs": {"sha256": "...", "lockedAt": "..."},
    "hh-touch-guard.py": {"sha256": "...", "lockedAt": "..."},
    "twenty-client.mjs": {"sha256": "...", "lockedAt": "..."},
    "twenty-preflight.mjs": {"sha256": "...", "lockedAt": "..."}
  },
  "settingsHookStanza": {
    "stanza": {
      "matcher": "^(Bash|WebFetch|mcp__playwright__.*|mcp__computer-use__.*)$",
      "hooks": [{"type":"command","command":"~/.claude/hooks/hh-touch-guard.py"}]
    },
    "sha256OfCanonicalJson": "...",
    "lockedAt": "..."
  }
}
```

**Granular settings pinning (Codex NEW-2 fix):** вместо SHA целого `hooks.PreToolUse` блока — pin-им **единственную конкретную stanza** (canonical JSON sort-keys). Добавление других, нерелевантных хуков НЕ ломает integrity. Attestation ищет stanza с matching matcher и проверяет что её canonical hash совпадает.

Обновление hash — только через явную команду `twenty-integrity.mjs --relock` с подтверждением.

### Sprint 5: Hook `hh-touch-guard.py` — narrow anchored matcher (Codex C2, NM2, NEW-3, R10, I5-2) (45 мин)

**Anchored narrow matcher** — покрывает только browser-capable tools, не добавляет latency на Read/Edit/Glob/Grep/TodoWrite:

`~/.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "^(Bash|WebFetch|mcp__playwright__.*|mcp__computer-use__.*)$",
        "hooks": [{"type":"command","command":"~/.claude/hooks/hh-touch-guard.py"}]
      }
    ]
  }
}
```

Покрытие всех путей к браузеру/hh.ru, но **не** к Read/Edit/Glob/Grep/TodoWrite/etc. Хук не запускается на нерелевантных tool calls → нет global latency.

`hh-touch-guard.py` (псевдокод):
```python
import json, re, sys
HH_URL_RX = re.compile(r"https?://[^\s'\"]*hh\.ru/(vacancy|resume|search|applicant|employer|negotiations)")
WRAPPER_RX = re.compile(r"hh-outreach/scripts/hh-cdp\.mjs\b")

def block(msg):
    print(f"BLOCKED (hh-touch-guard): {msg}\n"
          f"Use: SOURCE_SCRIPT=<name> node ~/hh-outreach/scripts/hh-cdp.mjs <args>",
          file=sys.stderr)
    sys.exit(2)

def any_value_contains_hh(obj):
    if isinstance(obj, str): return bool(HH_URL_RX.search(obj)) or "hh.ru" in obj
    if isinstance(obj, dict): return any(any_value_contains_hh(v) for v in obj.values())
    if isinstance(obj, list): return any(any_value_contains_hh(v) for v in obj)
    return False

data = json.loads(sys.stdin.read())
tool = data.get("tool_name","")
inp = data.get("tool_input",{})
flat = json.dumps(inp)

# Bash
if tool == "Bash":
    cmd = inp.get("command","")
    # Allow git commits even if they mention hh.ru in message
    if re.match(r"^\s*git\s+(commit|log|show|diff|status)\b", cmd):
        sys.exit(0)
    if HH_URL_RX.search(cmd) or ("hh.ru" in cmd and not "hh-outreach" in cmd):
        if not WRAPPER_RX.search(cmd):
            block("Bash command touches hh.ru without hh-cdp.mjs wrapper")

# WebFetch
elif tool == "WebFetch":
    url = inp.get("url","")
    if "hh.ru" in url:
        block("WebFetch to hh.ru forbidden — use hh-cdp.mjs")

# Playwright MCP — any tool name starting with mcp__playwright__
elif tool.startswith("mcp__playwright__"):
    if any_value_contains_hh(inp):
        block(f"Playwright MCP ({tool}) to hh.ru forbidden")

# Computer-use — block explicit hh.ru inputs
elif tool.startswith("mcp__computer-use__"):
    # Screenshot without args can't be judged → allow but attestation warns
    if tool == "mcp__computer-use__screenshot":
        sys.exit(0)
    if any_value_contains_hh(inp):
        block(f"computer-use ({tool}) with hh.ru input forbidden")

# Any other tool — allow (extension point)
sys.exit(0)
```

**Narrow anchored matcher** `^(Bash|WebFetch|mcp__playwright__.*|mcp__computer-use__.*)$` (см. выше) плюс explicit skip для `git commit/log/show/diff/status` и прямого `mcp__computer-use__screenshot` без args (attestation warning если Chrome frontmost hh.ru — см. Sprint 9 step 15).

**Tests (`hh-touch-guard.test.sh`):** минимум 20 кейсов:
- Прямой `cdp.mjs + hh.ru/vacancy/1` → BLOCK
- `WebFetch hh.ru` → BLOCK
- `playwright MCP navigate hh.ru` → BLOCK
- `open -a Chrome hh.ru` → BLOCK
- `osascript Safari hh.ru` → BLOCK
- `curl r.jina.ai hh.ru` → BLOCK
- `curl hh.ru/vacancy/1` → BLOCK
- `nohup node cdp.mjs ... hh.ru &` → BLOCK
- `hh-cdp.mjs nav hh.ru` → PASS
- `cdp.mjs list` → PASS
- `cdp.mjs stop` → PASS
- `git commit -m "fix hh.ru issue"` → PASS
- Playwright MCP hh.ru в nested param → BLOCK
- computer-use type "go to hh.ru" → BLOCK
- computer-use screenshot без args → PASS (с warning в attestation)
- ...

### Sprint 6: `twenty-drain.mjs` + reconciliation (Codex NC3, C4, NH2) (60 мин)

**Core drain logic:**
```
1. Acquire single-instance lock via fs.openSync('/tmp/twenty-drain.lock', 'wx'); on EEXIST exit 0 (another instance running). Stale >30s → force remove + retry.
2. List ~/hh-outreach-data/twenty-wal/*.json (not .sent/.failed/.tmp)
3. Sort by filename (chronological)
4. Request budget = 30 req (safe under 80/min bucket)
5. Iterate:
   a. Parse WAL entry
   b. Compute required requests:
      - seen_companies: upsert if not seen this run → +1 req
      - seen_opportunities: +1 req
      - touch → accumulate into pending batch
   c. If pending batch reaches 60 → flush touches → +1 req
   d. If budget_used >= 30 → stop, continue next run
6. Successful → rename file to .sent
7. Validation failure (HTTP 400) → reconcile:
   a. Attempt simplified payload (only required fields, no custom that may have been rejected)
   b. If still 400 → create HhTouch with just {occurredAt, touchType:'other',
      sourceScript, resultSummary:'reconciled', walFile} — minimum viable
   c. Only if simplified payload ALSO fails → rename .failed.N (N = retry count)
8. .failed.{N} retry schedule: 1min, 5min, 30min, 2h, 24h
9. After 5 failed retries → rename .failed.permanent → TG alert "manual review"
```

**Reconciliation guarantee (Codex NC3, NEW-8 — честная формулировка):**
- **No silent loss.** `.failed.permanent` — это терминальное automated состояние, НО оно **никогда не финально без человека**: TG alert + запись в `~/hh-outreach-data/reconciliation-queue.log` + attestation блокирует новые HH-runs пока `reconciliation-queue.log` не пустой. Получается "guaranteed human-in-the-loop reconciliation" (Codex NEW-8 reword).
- Runbook `docs/reconciliation-runbook.md`: step-by-step — прочитать WAL, проверить payload, создать вручную в Twenty UI или через API-скрипт, переименовать `.failed.permanent` → `.sent.manual`, закрыть строку в queue log.
- Attestation Sprint 9 step 7 → fail-closed если `reconciliation-queue.log` имеет открытые строки.

**Честно:** система гарантирует не "нет потерь вообще", а "нет потерь без человеческого решения". Это разные вещи (Codex NEW-8).

**Metrics file:**
```json
{
  "lastRunAt": "ISO",
  "processed": N,
  "succeeded": N,
  "reconciled": N,
  "failedRetryable": N,
  "failedPermanent": N,
  "pendingFiles": N,
  "bucketUsedLastRun": N
}
```

**launchd plist** — `StartInterval: 60`, `RunAtLoad: true`, `KeepAlive` on error.

**Accepted constraint (Codex NH2):** launchd runs only after user login. If Mac rebooted and user didn't log in, WAL accumulates. **Explicitly documented constraint**, not a hidden bug. Mitigation:
- Daily cron (runs via launchctl `com.apple.bsd` context при наличии) — если user logged in, запускает drain если backlog > 100
- TG alert от external healthcheck (uptime kuma ping-alert): если `/metrics` endpoint через ssh tunnel не обновлялся >6 часов — alert
- Runbook на `docs/reboot-recovery.md`: что сделать после reboot

**WAL backlog monitoring:**
```
ls ~/hh-outreach-data/twenty-wal/*.json 2>/dev/null | wc -l
```
- `< 10` — healthy
- `10-100` — warning (drain catching up)
- `> 100` — TG alert "backlog"
- `> 500` — TG critical + attestation blocks new HH runs

### Sprint 6.5: WAL cleanup daily job (Codex NEW-7, R7) (15 мин)

Один скрипт `~/bin/twenty-wal-cleanup.sh` (обрабатывает все типы файлов):

```bash
#!/bin/bash
set -e
WAL=~/hh-outreach-data/twenty-wal
# .sent older than 14 days
find "$WAL" -maxdepth 1 -name "*.sent" -mtime +14 -delete
# .sent.manual (resolved by human) older than 30 days
find "$WAL" -maxdepth 1 -name "*.sent.manual" -mtime +30 -delete
# .failed.N retryable — keep for drain to handle, no cleanup
# .failed.permanent — only delete after 60 days AND reconciliation-queue.log confirms resolution
for f in "$WAL"/*.failed.permanent; do
  [ -f "$f" ] || continue
  age_days=$(( ($(date +%s) - $(stat -f %m "$f")) / 86400 ))
  [ "$age_days" -lt 60 ] && continue
  basename=$(basename "$f" .failed.permanent)
  if grep -q "^$basename.*resolved" ~/hh-outreach-data/reconciliation-queue.log 2>/dev/null; then
    rm "$f"
  fi
done
```

launchd agent `~/Library/LaunchAgents/com.timzinin.wal-cleanup.plist`:
```xml
<plist><dict>
  <key>Label</key><string>com.timzinin.wal-cleanup</string>
  <key>ProgramArguments</key>
    <array>
      <string>/bin/bash</string>
      <string>/Users/timofeyzinin/bin/twenty-wal-cleanup.sh</string>
    </array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>4</integer><key>Minute</key><integer>15</integer></dict>
  <key>RunAtLoad</key><false/>
  <key>StandardErrorPath</key><string>/tmp/wal-cleanup.err</string>
  <key>StandardOutPath</key><string>/tmp/wal-cleanup.log</string>
</dict></plist>
```

**Install:** `launchctl load -w ~/Library/LaunchAgents/com.timzinin.wal-cleanup.plist`

**Verify:** `launchctl list com.timzinin.wal-cleanup` → loaded. Full retention matrix в section 5.

### Sprint 7: Cutover freeze (Codex H6, NH7, NEW-4) (15 мин)

**Переформулировано (Codex NEW-4):** убираем концепт "am I invoked through wrapper". Вместо этого — **каждый `hh-*.mjs` скрипт self-requires preflight environment check и всегда spawn-ит `hh-cdp.mjs` как subprocess**. Нет понятия "вызван через wrapper"; есть понятие "вызывает wrapper".

1. `mkdir -p ~/hh-outreach-archive/pre-twenty-20260411/`
2. `mv ~/hh-outreach-data/batch-send.mjs ~/hh-outreach-archive/pre-twenty-20260411/`
3. Все `hh-*.mjs` в `~/hh-outreach/scripts/` получают **preflight** + pattern:
```js
// preflight: env safe, integrity ok, drain alive, backlog ok, API healthy
import './twenty-preflight.mjs';

// usage: spawn hh-cdp.mjs as subprocess, NEVER call cdp.mjs directly
import { spawnSync } from 'child_process';
function hhCdp(cmd, ...args) {
  const r = spawnSync('node', [
    '/Users/timofeyzinin/hh-outreach/scripts/hh-cdp.mjs',
    cmd, ...args
  ], { encoding:'utf8', env: {...process.env, SOURCE_SCRIPT: 'hh-send-batch'} });
  return r;
}
```
4. Old scripts (`batch-send.mjs` etc.) физически перенесены в archive → нет possibility запустить их случайно.
5. Git tag `pre-twenty-cutover-v3` зафиксирован, `git diff pre-twenty-cutover-v3 HEAD -- 'scripts/hh-*'` показывает full cutover.
6. SKILL.md обновлён: раздел "LEGACY" перенесён в archive, все примеры — через `hhCdp()` helper.

**Comprehensive coverage (Codex NH7 resolve):** Sprint 9 attestation перечисляет каждый файл в `~/hh-outreach/scripts/hh-*.mjs` и проверяет что:
- Все импортируют `./twenty-preflight.mjs` в первой строке
- Все используют `spawnSync(... hh-cdp.mjs ...)` для CDP, ни одно упоминание `skills/chrome-cdp/scripts/cdp.mjs` напрямую
Любой новый файл без этого — attestation fail.

**Что это даёт вместо "provenance check":**
- Невозможно случайно использовать cdp.mjs напрямую — код физически не содержит такого вызова
- Preflight защищает от запуска в poisoned environment
- Hook защищает от ручных command-line bypass
- Integrity check защищает от подмены preflight/hh-cdp/wrapper файлов

### Sprint 8: Migration (Codex H3 resolved in v2, kept) (30 мин)

Без изменений от v2. Полная status-map table. Idempotent upsert. `--dry-run` default, `--commit` explicit.

Единственное добавление: **migration runs under budget-aware drain logic** — batched через ту же `twenty-client` token bucket, не обход.

### Sprint 9: `hh-check-env.mjs` — fix + non-destructive (Codex M3, NH4) (45 мин)

**Attestation spec (v5, aligned with v4/v5 fixes — Codex R1):**

```
hh-check-env.mjs [--silent] [--no-touch]

Checks in order (каждая падает с конкретным exit code):

1. [exit 1] Files integrity: SHA256 of hh-cdp.mjs, hh-touch-guard.py,
   twenty-client.mjs, twenty-preflight.mjs matches twenty-integrity.json.files.*
2. [exit 2] settings.json: contains stanza
   matcher="^(Bash|WebFetch|mcp__playwright__.*|mcp__computer-use__.*)$"
   with hook command="~/.claude/hooks/hh-touch-guard.py" —
   compute canonical JSON hash and compare with
   twenty-integrity.json.settingsHookStanza.sha256OfCanonicalJson
3. [exit 3] Every ~/hh-outreach/scripts/hh-*.mjs file:
   - Line 1-5 imports "./twenty-preflight.mjs"
   - Contains exactly one spawnSync or similar pointing to hh-cdp.mjs
   - Contains ZERO references to "skills/chrome-cdp/scripts/cdp.mjs"
4. [exit 4] launchd drain: `launchctl list com.timzinin.twenty-drain` loaded
5. [exit 5] twenty-drain metrics.json exists, lastRunAt ≤ 120s
6. [exit 6] WAL backlog: ls ~/hh-outreach-data/twenty-wal/*.json | wc -l < 500
7. [exit 7] reconciliation-queue.log is empty (no open lines)
8. [exit 8] TWENTY_API_KEY loaded, healthCheck 200 (THROUGH shared rate limiter)
9. [exit 9] TWENTY_KEY_EXPIRES > today + 7 days
10. [exit 10] orphan integrity: for every Company with hhEmployerId starting with
    'orphan-', there must be at least one Opportunity linked (via companyId).
    Orphans without opportunities → fail
11. [exit 11] WAL directory config check (CHEAP — runtime):
    - Verify `twenty-drain.mjs` at runtime exposes its scan-path via `--print-scan-path` flag
    - Expected output: `/Users/timofeyzinin/hh-outreach-data/twenty-wal` (exact)
    - Any other value → exit 11
    - **Behavioral canary test перенесён в install-time** — см. ниже.

12. [exit 12] Guard positive test (non-destructive, Codex NH4):
    - Send synthetic input to hh-touch-guard.py via pipe
    - Assert exit 2 for: WebFetch hh.ru, playwright MCP hh.ru, bash cdp.mjs hh.ru
    - No file creation, no HH request, no WAL writes

13. [exit 13] WAL self-test (local-only, Codex NEW-5, R4):
    - Spawn `hh-cdp.mjs --wal-selftest`
    - Verify: exit 0, output contains "selftest ok", file deleted,
      main twenty-wal/ unchanged
    - No Twenty write, no drain pollution

14. [exit 14] Orphan review — **single source of truth** (Codex I6-1 unified):
    - Read `~/hh-outreach-data/orphan-registry.json`
    - Fail if any entry has `reviewRequired: true` AND no matching resolved line in docs/orphan-audit.md
    - `orphan-registry.json` is the ONLY canonical source (не `orphan-reconciliation.log`, не отдельные файлы)

15. [exit 15] Install-verification freshness (fail-closed, Codex I6-2):
    - Check `~/hh-outreach-data/.install-verified` exists
    - Check mtime age < 14 days
    - Missing or stale → exit 15

16. [exit 0, non-blocking warning] Browser awareness (Codex NC2):
    - osascript query frontmost app + URL (Chrome/Safari)
    - If hh.ru detected → print warning "do not use computer-use screenshot for HH"
    - Warning only, does NOT fail attestation

Exit 0 = fully healthy.

`--no-touch` flag: skip step 13 (WAL self-test) only.
```

**Install-time verification (Codex I6-2 — вынесено из per-run attestation):**
Отдельный скрипт `hh-verify-install.mjs` — запускается:
- Сразу после Sprint 5 install
- Еженедельный launchd cron (воскресенье 03:00)
- Вручную через `node hh-verify-install.mjs`

Heavy checks (до 120s total, НЕ в runtime path):
- **Drain exclusion canary:** create canary in `twenty-wal-selftest/`, kickstart drain, wait 60s, assert canary unchanged and not in Twenty
- **Bypass path pen-test:** отправить в hh-touch-guard через stdin 20 тестовых bypass сценариев (все must exit 2)
- **Rate limiter cross-process test:** spawn 3 параллельных dummy процесса, проверить что `used` не превышает 80 в любом моменте
- **Integrity sweep:** сверка всех pinned файлов
- **Orphan registry audit:** **точно те же criteria что Sprint 9 step 14** — fail если есть `reviewRequired: true` без matching `resolved` line в `docs/orphan-audit.md`. Install-time и runtime используют идентичную логику — fail-closed одинаково.

Failure → TG alert + install-verification flag `~/hh-outreach-data/.install-verified` удаляется. Runtime `hh-check-env.mjs` step 15 проверяет existence этого флага — если нет, fail-closed.

Каждый `hh-*.mjs` startup не ждёт 90s, но runtime attestation требует чтобы install-verification был свежий (< 14 дней).

**Non-destructive promise:** `--no-touch` режим вообще не пишет WAL, не делает network calls за исключением `/healthz`.

**Integration:** каждый `hh-*.mjs` при старте runs `hh-check-env.mjs` (без `--no-touch`). Этот вызов сам по себе НЕ запускает рекурсию потому что он использует `--no-touch` для attestation-внутри-attestation: при detection `SOURCE_SCRIPT=hh-check-env` → skip step 12.

### Sprint 10: Production verification (20 мин)

Чистый functional test — все пункты независимо измеримы (Codex M2):

- [ ] `curl -sI https://crm.timzinin.com/healthz` → 200
- [ ] `curl -sH "Authorization: Bearer $KEY" $URL/rest/opportunities?limit=1` → JSON
- [ ] `curl -sH ... $URL/rest/metadata/objects?filter=name[eq]:hhTouch` → 1 result
- [ ] Мигрировано: `totalCount >= 470`
- [ ] `unresolved.log` reviewed вручную
- [ ] Unit тесты `hh-cdp.mjs` (≥ 10) — все зелёные
- [ ] Unit тесты `hh-touch-guard.py` (≥ 20) — все зелёные
- [ ] Unit тесты `twenty-client.mjs` (≥ 12) — все зелёные
- [ ] Integrity attestation: `node hh-check-env.mjs --no-touch` exit 0
- [ ] Live trace: `SOURCE_SCRIPT=test-manual node hh-cdp.mjs nav <tab> https://hh.ru/vacancy/131695122` → WAL file exists, через ≤ 70 сек в `.sent`, в Twenty новый HhTouch
- [ ] BLOCKED: прямой `cdp.mjs` + hh.ru → exit 2
- [ ] BLOCKED: `WebFetch hh.ru` → exit 2
- [ ] BLOCKED: `mcp__playwright__browser_navigate hh.ru` → exit 2
- [ ] BLOCKED: `open -a Chrome https://hh.ru` → exit 2
- [ ] BLOCKED: любая `osascript + Safari` hh.ru → exit 2
- [ ] PASSED: `git commit -m "fix hh.ru parsing"` → exit 0
- [ ] Certbot renewal test: `certbot renew --dry-run --deploy-hook "echo ok"` → exit 0 + "ok" в логе
- [ ] pg_dump cron на Contabo: `/opt/backups/twenty-$(date +%F).sql.gz` exists
- [ ] Uptime Kuma monitor зелёный ≥ 10 мин
- [ ] `reconciliation-queue.log` пуст
- [ ] Twenty-drain metrics: `pendingFiles < 10`, `lastRunAt` свежее
- [ ] Git tag `post-twenty-cutover` создан

### Sprint 12: Codex review iteration N → 10/10

(Codex M2 resolution: **не часть DoD**. Отдельный gate процесса, не система.) После Sprint 10 prod verified — запуск `/codex-review` на diff `pre-twenty-cutover → post-twenty-cutover`. Если < 10 — фикс и повторный review.

---

## Minimum Viable Cutover — single-pass execution (Codex I6-5)

Что должно существовать **до** того как запустить `/hh-outreach 50` первый раз. Точная последовательность:

```
# ————— Day 1 (~3 часа) —————

# 1. DNS (Sprint 0)
# В DNS провайдере: A crm.timzinin.com → 185.202.239.165
# Wait 5 мин, verify:
dig +short crm.timzinin.com           # → 185.202.239.165

# 2. Deploy Twenty (Sprint 1)
ssh root@185.202.239.165 '
  mkdir -p /opt/twenty && cd /opt/twenty &&
  curl -sLO https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-docker/docker-compose.yml &&
  curl -sLO https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-docker/.env.example &&
  mv .env.example .env
'
# Ручная правка .env: APP_SECRET, PG_DATABASE_PASSWORD, SERVER_URL, IS_SIGN_UP_DISABLED=false
# Ручная правка compose: 127.0.0.1:3003:3000
# nginx HTTP-only block + certbot + HTTPS block (см. Sprint 1)
ssh root@185.202.239.165 'cd /opt/twenty && docker compose up -d'
# Bootstrap user в UI: https://crm.timzinin.com → Sign up
# Disable signup:
ssh root@185.202.239.165 'sed -i s/IS_SIGN_UP_DISABLED=false/IS_SIGN_UP_DISABLED=true/ /opt/twenty/.env && cd /opt/twenty && docker compose up -d'
curl -sI https://crm.timzinin.com/healthz | head -1  # → HTTP/2 200

# 3. Schema + API key (Sprint 2)
# UI: Settings → Create API key → save
# Local:
mkdir -p ~/.secrets && chmod 700 ~/.secrets
cat > ~/.secrets/twenty.env <<EOF
TWENTY_API_URL=https://crm.timzinin.com
TWENTY_API_KEY=<paste>
TWENTY_KEY_EXPIRES=2027-04-11
EOF
chmod 600 ~/.secrets/twenty.env
source ~/.secrets/twenty.env
node ~/hh-outreach/scripts/twenty-schema.mjs --apply

# 4. Implement + integrity lock (Sprints 3-5)
# Write: twenty-client.mjs, hh-cdp.mjs, hh-touch-guard.py, twenty-preflight.mjs
# Compute initial hashes:
node ~/hh-outreach/scripts/twenty-integrity.mjs --relock
# Install hook in settings.json (narrow anchored matcher)
python3 ~/.claude/hooks/hh-touch-guard.py --self-test   # exit 0

# 5. Drain + crons (Sprints 6, 6.5, 8.5)
launchctl load -w ~/Library/LaunchAgents/com.timzinin.twenty-drain.plist
launchctl load -w ~/Library/LaunchAgents/com.timzinin.wal-cleanup.plist
launchctl load -w ~/Library/LaunchAgents/com.timzinin.twenty-key-check.plist
launchctl load -w ~/Library/LaunchAgents/com.timzinin.twenty-upgrade-check.plist
launchctl load -w ~/Library/LaunchAgents/com.timzinin.wal-sanity.plist
launchctl list | grep timzinin   # 5 agents loaded

# 6. Cutover freeze (Sprint 7)
mkdir -p ~/hh-outreach-archive/pre-twenty-20260411/
mv ~/hh-outreach-data/batch-send.mjs ~/hh-outreach-archive/pre-twenty-20260411/
cd ~/hh-outreach && git tag pre-twenty-cutover && git push --tags

# 7. Migration (Sprint 8)
source ~/.secrets/twenty.env
node ~/hh-outreach/scripts/migrate-sent-to-twenty.mjs --dry-run
# review migration-preview.json
node ~/hh-outreach/scripts/migrate-sent-to-twenty.mjs --commit
# verify:
curl -sH "Authorization: Bearer $TWENTY_API_KEY" "$TWENTY_API_URL/rest/opportunities?limit=1" | jq '.totalCount'

# 8. Install-time verification (Sprint 9 heavy checks)
node ~/hh-outreach/scripts/hh-verify-install.mjs
# exit 0 → touch ~/hh-outreach-data/.install-verified

# 9. Runtime attestation smoke test
SOURCE_SCRIPT=test-manual node ~/hh-outreach/scripts/hh-check-env.mjs --no-touch   # exit 0

# 10. Prod verification (Sprint 10 DoD table)
# Run through all [ ] items, check each

# 11. Ready — first batch:
# Load & run hh-send-batch (which now uses wrapper)
SOURCE_SCRIPT=hh-send-batch node ~/hh-outreach/scripts/hh-send-batch.mjs 50
```

**Daily routine after cutover:**
```
# Check backlog (visible in Twenty UI or):
ls ~/hh-outreach-data/twenty-wal/*.json 2>/dev/null | wc -l
# Check reply replies cron:
node ~/hh-outreach/scripts/hh-check-replies.mjs
# Review reconciliation queue:
cat ~/hh-outreach-data/reconciliation-queue.log
cat ~/hh-outreach-data/orphan-registry.json | jq '[to_entries[] | select(.value.reviewRequired)] | length'
```

---

## 4.5. Sprint 8.5: Cron jobs (Codex R6, R9, I6-4) (20 мин)

Создаются последним спринтом перед prod-verification:

1. **API key expiry alert cron** (`~/bin/twenty-key-expiry-check.sh`):
```bash
#!/bin/bash
source ~/.secrets/twenty.env
EXPIRY=$(date -j -f "%Y-%m-%d" "$TWENTY_KEY_EXPIRES" "+%s")
NOW=$(date +%s)
DAYS_LEFT=$(( (EXPIRY - NOW) / 86400 ))
if [ "$DAYS_LEFT" -le 7 ]; then
  node ~/hh-outreach/scripts/notify-tim.mjs "Twenty API key expires in $DAYS_LEFT days"
fi
```
launchd agent `com.timzinin.twenty-key-check`, StartCalendarInterval daily 09:00.

2. **Twenty version upgrade reminder** (`com.timzinin.twenty-upgrade-check`):
   - Monthly, 1 числа, 10:00
   - Сверяет текущий `TAG=` из `/opt/twenty/.env` (через SSH) с latest release GitHub API
   - Если разница > 2 минорные версии → TG alert "Review upgrade from vX.Y.Z to vA.B.C — run Codex review gate per upgrade-governance.md"

3. **WAL backlog sanity cron** (`com.timzinin.wal-sanity`):
   - Каждые 6 часов
   - Counts WAL files + `.failed.*` + `.sent.manual`
   - Alerts TG если backlog > 100 or failures > 0

4. **Upgrade governance** — `docs/upgrade-governance.md`:
   - Owner: Тим Зинин (sole operator)
   - Cadence: review каждые 30 дней после monthly cron alert
   - Gate: Codex review минорных изменений → minor upgrade OK; major (v2.x) → full plan re-review
   - Never auto-upgrade

## 5. Security + Secret lifecycle (Codex M4)

Полный runbook — `docs/secret-lifecycle-runbook.md`:

| Secret | Storage | Rotation | Expiry alert |
|---|---|---|---|
| `TWENTY_API_KEY` | `~/.secrets/twenty.env` (600) + KeePassXC | Yearly or on compromise | Daily cron checks `TWENTY_KEY_EXPIRES`, TG alert 7/3/1 days before |
| `APP_SECRET` | `/opt/twenty/.env` (600) + KeePassXC | Only on compromise (invalidates sessions) | — |
| `PG_DATABASE_PASSWORD` | `/opt/twenty/.env` (600) + KeePassXC | Yearly | — |
| Backup archive access | SSH key `root@contabo` + local `/opt/backups/` 700 | SSH key yearly | — |
| WAL retention (Codex R7) | Daily cleanup (Sprint 6.5): `.sent` older 14d, `.sent.manual` older 30d, `.failed.permanent` older 60d (after reconciliation check). Sprint 8.5 cron alerts on any `.failed.*` > 0. `twenty-wal/` chmod 700 | — | Monthly audit of permissions |
| Integrity tamper recovery (Codex R5) | Runbook `docs/integrity-recovery.md`: (1) compare `~/.secrets/twenty-integrity.json` sha256 vs KeePass backup → mismatch = tamper. (2) Restore integrity.json from KeePass. (3) Recompute all file hashes, verify against git-committed `scripts/*.mjs` in `TimmyZinin/vacancy-outreach-skill` via `git hash-object` + git show against last known good commit. (4) If wrapper files changed — re-clone from git + re-lock. (5) Attestation must pass before resuming. | On any integrity mismatch | — |
| Let's Encrypt certs | `/etc/letsencrypt/` — auto-renew via `certbot.timer` + `--deploy-hook` reload | Every 60 days automatic | TG if `certbot renew --dry-run` fails |

**Compromise runbook for API key:**
1. Revoke in Twenty UI → Settings → APIs → Delete key
2. Create new key `hh-integration-key-<date>`
3. Update `~/.secrets/twenty.env`
4. `launchctl kickstart -k gui/<uid>/com.timzinin.twenty-drain` — force restart
5. Verify drain uses new key: check next metrics update
6. Audit `Twenty → Settings → Audit log` for last-30-days actions by old key

---

## 6. Failure modes + reconciliation

| Риск | Behavior | Recovery |
|---|---|---|
| Twenty down | Drain blocks, WAL grows | Auto-drain on recovery |
| Contabo down | WAL grows + nginx cert renewal pauses | Auto-recover |
| Disk full (WAL) | `hh-cdp.mjs` fail-closed exit 4 | Weekly cron `.sent` cleanup + TG alert |
| drain crashed | launchd restarts | launchd stderr log |
| Migration run twice | Idempotent upsert by unique id | no-op |
| Twenty v1.22 breaking | Pinned `TAG=v1.21.0` | Manual upgrade gate after Codex review |
| Hook not loaded in session | `hh-check-env` step 2/11 fails at script start | User re-launches Claude Code, checks settings.json |
| API key expired | Drain 401 → `.failed.N` → eventually `.failed.permanent` → TG alert via runbook | Rotate key per Sprint 5 runbook |
| Race 2x `hh-cdp.mjs` | nanoid filenames unique + fs.renameSync atomic on APFS same-volume (Codex NM3 documented) | no race |
| 429 Twenty | Token bucket 80/min, Retry-After respected | no escalation |
| `.failed.permanent` | Runbook + TG alert + attestation blocks new HH runs | Manual reconcile per runbook |
| Mac reboot no login | Drain doesn't start (Codex NH2 accepted constraint) | User logs in → launchd activates → runbook mentioned |
| Wrapper/hook tampered | Integrity check (Sprint 4, 9) → exit 5 | `--relock` only after manual review |
| Freeze removed (`rm .freeze`) | Obsolete — архивация вместо файла (Codex NH7) | old scripts in `~/hh-outreach-archive/`, untouched path |
| `computer-use screenshot` of already-open HH tab | Warning in attestation, not blocked (Codex NC2 accepted); relies on SKILL.md discipline | — |
| Secrets leaked in WAL | Whitelist + blacklist sanitizer (Codex NH3) | 14-day `.sent` retention, weekly audit |

---

## 7. Definition of Done (Codex M2: no circular references)

System properties only. Each item has concrete evidence command:

- [ ] `curl -sI https://crm.timzinin.com/healthz` → HTTP 200
- [ ] `docker compose ps` все 4 сервиса healthy
- [ ] `~/.secrets/twenty.env` chmod 600 (`stat -f %Op`)
- [ ] `hhEmployerId.isUnique == true, isNullable == false` (metadata API)
- [ ] `hhVacancyId.isUnique == true, isNullable == false`
- [ ] `hhTouch` object exists, `touchType` enum has 10 values including `migrated` + `synthetic`
- [ ] Migration complete: `succeeded + skipped == 583`
- [ ] `unresolved.log` — manual review signed off в `docs/migration-review.md`
- [ ] Unit tests: `hh-cdp.mjs ≥ 10`, `hh-touch-guard.py ≥ 20`, `twenty-client.mjs ≥ 12`, all green
- [ ] File-integrity file `~/.secrets/twenty-integrity.json` existed, SHA256 verified on rerun
- [ ] `launchctl list | grep twenty-drain` loaded
- [ ] `twenty-drain-metrics.json` lastRunAt ≤ 2 min
- [ ] WAL test: manual WAL file → through ≤ 70 sec `.sent`
- [ ] Live trace test: `hh-cdp.mjs nav hh.ru/vacancy/131695122` → HhTouch appears in Twenty
- [ ] BLOCKED tests: all bypass paths (Sprint 5) exit 2
- [ ] PASSED tests: git/allowed commands exit 0
- [ ] `certbot renew --dry-run` exits 0 + `--deploy-hook` ran
- [ ] `/opt/backups/twenty-YYYY-MM-DD.sql.gz` exists from nightly cron
- [ ] Uptime Kuma monitor green ≥ 10 min consecutive
- [ ] `reconciliation-queue.log` empty
- [ ] Secret lifecycle runbook committed: `docs/secret-lifecycle-runbook.md`
- [ ] Reconciliation runbook committed: `docs/reconciliation-runbook.md`
- [ ] Reboot recovery runbook committed: `docs/reboot-recovery.md`
- [ ] Git tag `post-twenty-cutover` created in `hh-outreach` repo
- [ ] Old HH scripts archived in `~/hh-outreach-archive/pre-twenty-20260411/`

**НЕ входит в DoD:**
- ❌ "Codex ≥ 10/10" — это отдельный process gate, не система (Codex M2 resolution)

---

## 8. НЕ делаем

- Webhooks Twenty → external
- FlowDeal/Lead-Mashina integration
- LinkedIn/Kariyer в Twenty (separate pipeline)
- Custom HTML dashboard (native Twenty UI сухватает)
- Team features
- LaunchDaemon (system-boot drain) — accepted user-session constraint
- File-integrity HMAC signing (SHA256 достаточно для MVP)
- Блокировка passive computer-use screenshot — out of scope

---

## 9. Источники

- https://docs.twenty.com/developers/self-hosting/docker-compose
- https://docs.twenty.com/developers/extend/api.md
- https://docs.twenty.com/developers/extend/webhooks.md
- https://github.com/twentyhq/twenty (v1.21.0, 43.8K★, AGPL)
- Codex review iteration 1 (3/10): все 17 findings → addressed
- Codex review iteration 2 (6/10): `docs/codex-review-2.md` → addressed

---

## Changelog

### v6.0 (2026-04-11) — Sprint Protocol section added
- Added section 3.9 with 11-step per-sprint workflow
- Added section 3.10 with Codex gate matrix per sprint (adversarial + code review in every sprint)
- Thresholds: ≥9/10 intermediate, =10/10 final S12
- 5 HITL pauses defined (S1, S7, S8, S10, S12)

### v5.4 (2026-04-11) — Codex review iteration 8 (9.7/10 → target 10/10)

- **I8-1** Install-time `hh-verify-install.mjs` и runtime `hh-check-env.mjs` step 14 теперь используют **идентичные** criteria: fail если `reviewRequired: true` без matching `resolved` line в `docs/orphan-audit.md`. Раньше install-time пропускал `open`, runtime требовал `resolved` — это могло привести к ситуации где `.install-verified` свежий, но runtime падал.

### v5.3 (2026-04-11) — Codex review iteration 7 (9.0/10 → target 10/10)

- **I7-1 (was I6-1 partial)** Unified orphan source: Sprint 9 step 14 читает **только** `orphan-registry.json`. Убраны разрозненные ссылки на `orphan-reconciliation.log`, `orphan-collision.log`, `orphan-audit.md` (последний остался как human decision log, но не как auth source).
- **I7-2 (was I6-2 partial)** Sprint 9 step 15 разделён: step 15 = install-verification freshness (fail-closed), warning step 16 = browser awareness (non-blocking).
- **I7-3 (was I6-4 partial)** Stale "Sprint 11" → "Sprint 10" в finale references (Sprint 12 и Minimum Viable Cutover).

### v5.2 (2026-04-11) — Codex review iteration 6 (8.8/10 → target 10/10)

- **I6-1** Orphan registry `orphan-registry.json` tracks все orphan companies + display names + vacancies + reviewRequired heuristic. Attestation step 14 проверяет.
- **I6-2** Heavy canary test вынесен в `hh-verify-install.mjs` (install-time / weekly cron), runtime attestation step 11 остался cheap config check `--print-scan-path`.
- **I6-3** Все оставшиеся `flock` ссылки заменены на atomic O_EXCL lockfile (Architecture diagram, Sprint 6, changelog).
- **I6-4** Duplicate Sprint 11 → переименовано: cron jobs стали Sprint 8.5, Codex review gate стал Sprint 12.
- **I6-5** Новая секция "Minimum Viable Cutover" — single-pass execution commands от DNS до первого batch.

### v5.1 (2026-04-11) — Codex review iteration 5 (9.0/10 → target 10/10)

- **I5-1** Orphan employer id = `sha256(normalized_name)` без URL — stable across processing order. Collision detection через `orphan-collision.log` + attestation block + manual runbook.
- **I5-2** Sprint 5 section-level leftover "catch-all" переписан → "narrow anchored matcher" везде.
- **Sprint 9 step 11** переделан с brittle grep на behavioral canary test.

### v5.0 (2026-04-11) — Codex review iteration 4 (8.4/10 → target 10/10)

- **R1** Sprint 9 attestation переписан под v4/v5 реалии (narrow matcher, preflight+spawn, per-stanza hash, selftest separate dir)
- **R2** `flock` заменён на native macOS-совместимый atomic lock через `fs.openSync(..., 'wx')` с stale detection
- **R3** Orphan employer id теперь включает URL первой vacancy как discriminator → нет collision двух разных employer с одинаковым именем. Runbook reconciliation для неоднозначных случаев.
- **R4** `--wal-selftest` + `--integrity-check` вынесены в explicit CLI surface Sprint 4. Drain explicitly excludes `twenty-wal-selftest/`.
- **R5** `docs/integrity-recovery.md` runbook — что делать если `twenty-integrity.json` сам подменён. KeePass backup + git-based re-verification.
- **R6** Sprint 11 создан с expiry alert cron, WAL backlog sanity, upgrade reminder.
- **R7** WAL retention покрывает все типы: `.sent` 14d, `.sent.manual` 30d, `.failed.permanent` 60d с reconciliation check. `.failed.N` не удаляется (drain handles).
- **R8** `healthCheck()` явно использует shared rate limiter как все остальные HTTP-вызовы.
- **R9** `docs/upgrade-governance.md` — owner, cadence, gate для Twenty version upgrade.
- **R10** Matcher regex anchored: `^(Bash|WebFetch|mcp__playwright__.*|mcp__computer-use__.*)$`

### v4.0 (2026-04-11) — Codex review iteration 3 (8/10 → target 10/10)

- **NEW-1** Cross-process rate limiter через shared `twenty-rate-budget.json` + atomic O_EXCL lock. Не per-process — глобально.
- **NEW-2** Integrity pinning per-stanza (canonical JSON), не весь hooks block. Другие хуки могут добавляться без false-fail.
- **NEW-3** Hook matcher изменён с `.*` на `^(Bash|WebFetch|mcp__playwright__.*|mcp__computer-use__.*)$`. Нет global latency на Read/Edit/Glob.
- **NEW-4** `assertWrapper` концепт удалён. Вместо этого каждый `hh-*.mjs` импортирует `twenty-preflight.mjs` и spawn-ит `hh-cdp.mjs` как subprocess — нет вопроса "кто меня вызвал".
- **NEW-5** Sprint 9 step 12 rewrite: WAL self-test идёт в separate directory `twenty-wal-selftest/`, drain не видит. Evidence drain liveness — через metrics.lastRunAt, не через synthetic record.
- **NEW-6** UUID v5 заменён на standalone `createHash('sha1')` из Node native `crypto`. Нет `package.json` изменений.
- **NEW-7** Sprint 6.5 добавлен: daily WAL cleanup launchd job. `.sent` files >14 days deleted. Daily везде (не weekly).
- **NEW-8** NC3 reword: "no silent loss" вместо "no terminal state". Честная формулировка: `.failed.permanent` существует, но attestation блокирует работу пока не разрешено человеком.

### v3.0 (2026-04-11) — Codex review iteration 2 (6/10 → target 10/10)

**Critical resolutions:**
- **NC1** Integrity pinning via SHA256 `~/.secrets/twenty-integrity.json`, checked at script start. Claim downgraded from "physically impossible" to "best-effort local enforcement with integrity check" (section 0).
- **NC2** Hook matcher `.*` catch-all + explicit computer-use handling. Screenshot без args → warning в attestation, не block (честно документировано).
- **NC3** Reconciliation guarantee: нет `.failed.permanent` как терминала. Runbook + TG alert + attestation блокирует новые runs пока не разрешено.

**High resolutions:**
- **NH1** Request-budget math: drain берёт по reserved request cost (не touch count), worst case ≤ 30 req per minute, safely under 80/min bucket (section Sprint 3, 6).
- **NH2** LaunchAgent user-session constraint явно accepted + documented + `docs/reboot-recovery.md`.
- **NH3** WAL whitelist + blacklist sanitization, 14-day retention, chmod 700.
- **NH4** `hh-check-env.mjs` полностью non-destructive: synthetic WAL не трогает HH, `--no-touch` для attestation-внутри-attestation, integrity check + guard positive test via pipe.
- **NH5** Company race fix: `hhEmployerId` never null, deterministic `orphan-{uuidv5(name)}` fallback.
- **NH6** Certbot `--deploy-hook "systemctl reload nginx"` wired in Sprint 1.
- **NH7** Freeze через архивацию файлов + `assertWrapper` import check во всех `hh-*.mjs`, comprehensive.

**Medium resolutions:**
- **NM1** DoD без "Codex 10/10" circular item.
- **NM2** Hook — mix string + parsed JSON value walk for nested params.
- **NM3** `fs.renameSync` atomicity on APFS/HFS+ same-volume явно задокументировано.
- **NM4** Manual UI key creation accepted как MVP, export/import automation отложен.

**Partial resolutions (previous):**
- C2 → C2-resolved в v3 catch-all matcher
- C4 → partial: user-session launchd, explicitly accepted
- H2 → deploy-hook fully wires renewal
- H4 → NH5 deterministic fallback
- H5 → NH1 request-budget
- H6 → NH7 archival
- M3 → NH4 non-destructive + integrity
- M4 → full secret lifecycle runbook
