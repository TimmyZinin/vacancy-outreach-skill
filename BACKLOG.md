# Vacancy Outreach - Бэклог и спринты (v4, Codex 9/10)

## Бэклог

| ID | Задача | Приоритет | Спринт | Статус |
|----|--------|-----------|--------|--------|
| HH-01 | SKILL.md: создать с полной спецификацией | crit | 1 | backlog |
| HH-02 | DATA: создать ~/hh-outreach-data/ и sent.json с 64 записями | crit | 1 | backlog |
| HH-03 | DATA: валидация sent.json при загрузке (malformed JSON, missing fields, dupes) | high | 1 | backlog |
| HH-04 | DATA: backup sent.json перед каждым запуском | high | 1 | backlog |
| HH-05 | DATA: atomic write sent.json (tmp + rename) | crit | 1 | backlog |
| HH-06 | ARGS: парсинг количества (позиционный аргумент) | crit | 1 | backlog |
| HH-07 | ARGS: парсинг --query, позиционный запрос | crit | 1 | backlog |
| HH-08 | ARGS: парсинг --period, --min-salary | high | 1 | backlog |
| HH-09 | ARGS: парсинг --dry-run, --preview, --any-format, --no-filter | crit | 1 | backlog |
| HH-10 | CDP: healthcheck (list, проверка наличия HH таба) | crit | 1 | backlog |
| HH-11 | CDP: tab reacquisition при потере соединения | high | 1 | backlog |
| HH-12 | CDP: navigation timeout handling (15s max) | high | 1 | backlog |
| HH-13 | CDP: eval timeout handling (10s max) | high | 1 | backlog |
| HH-14 | Тест Спринт 1: `/vacancy-outreach 1 --dry-run` - CDP подключается, аргументы парсятся, sent.json загружается | crit | 1 | backlog |
| HH-15 | SEARCH: построение URL поиска HH из аргументов | crit | 2 | backlog |
| HH-16 | SEARCH: дефолтный фильтр remote-only + --any-format override | high | 2 | backlog |
| HH-17 | SEARCH: извлечение списка вакансий через CDP eval (ID, title, company) | crit | 2 | backlog |
| HH-18 | SEARCH: фильтрация по exclusion list (WB, Ozon, продажи, HR, стажёры, дизайнеры, разработчики) + --no-filter | crit | 2 | backlog |
| HH-19 | SEARCH: дедупликация по sent.json | high | 2 | backlog |
| HH-20 | SEARCH: пагинация (страницы 0-N, до набора нужного количества) | high | 2 | backlog |
| HH-21 | SEARCH: обработка 0 результатов (показать сообщение, завершить) | med | 2 | backlog |
| HH-22 | READ: навигация на вакансию через CDP nav | crit | 2 | backlog |
| HH-23 | READ: извлечение описания (первые 1000 символов) через CDP eval | crit | 2 | backlog |
| HH-24 | READ: парсинг зарплаты - "от X", "до Y", "от X до Y" | high | 2 | backlog |
| HH-25 | READ: парсинг зарплаты - gross/net, $/руб/евро (курс: $1=100₽, €1=110₽) | high | 2 | backlog |
| HH-26 | READ: непарсируемая зарплата = null (не упоминать в тексте) | med | 2 | backlog |
| HH-27 | READ: проверка "Вы откликнулись" через eval | high | 2 | backlog |
| HH-28 | Тест Спринт 2: `/vacancy-outreach 5 --dry-run` находит 5 вакансий, читает описания. Проверить: ID, компания, должность, зарплата (нормализованная), описание 200 символов. 0 ошибок CDP. | crit | 2 | backlog |
| HH-29 | GENERATE: модульный шаблон (6 блоков: OPENING, VACANCY_REACTION, OFFER, WHAT_I_DO, SLOTS, CTA) | crit | 3 | backlog |
| HH-30 | GENERATE: 5 вариантов OPENING (random выбор) | high | 3 | backlog |
| HH-31 | GENERATE: 7 модулей WHAT_I_DO (match по ключевым словам вакансии: SMM→контент-завод, B2B→лид-машина, воронки→автоворонки, SEO→AI-тексты, CRM→сегментация, видео→HeyGen, блогеры→AI-поиск) | crit | 3 | backlog |
| HH-32 | GENERATE: блок OFFER с нормализованной зарплатой (*0.8) или без цены | high | 3 | backlog |
| HH-33 | GENERATE: anti-AI проверка (без тире —, без hedge words, без passive voice, без "инновационный/комплексный/синергия") | high | 3 | backlog |
| HH-34 | PREVIEW: показать сгенерированный текст, ждать ввод пользователя (OK/Skip), записать статус SKIP_USER или продолжить | high | 3 | backlog |
| HH-35 | Тест Спринт 3: `/vacancy-outreach 10 --dry-run` генерит 10 текстов. Проверить: все 10 уникальны, каждый содержит название компании, каждый содержит calendly.com/timzinin, ни в одном нет тире (—). | crit | 3 | backlog |
| HH-36 | SEND: DOM-based клик "Откликнуться" (eval по textContent, rect.y > 100, rect.width > 100) | crit | 4 | backlog |
| HH-37 | SEND: обработка popup "вакансия в другой стране" (eval по textContent "Все равно") | high | 4 | backlog |
| HH-38 | SEND: DOM-based клик "Добавить сопроводительное" (eval по textContent, rect.y > 500) | crit | 4 | backlog |
| HH-39 | SEND: polling textarea (каждые 500мс, timeout 5с, 10 попыток) | crit | 4 | backlog |
| HH-40 | SEND: обработка обязательного сопроводительного (textarea сразу в модалке) | high | 4 | backlog |
| HH-41 | SEND: обработка выбора резюме (если несколько - выбрать первое или "AI-автоматизатор") | med | 4 | backlog |
| HH-42 | SEND: ввод текста через CDP type | crit | 4 | backlog |
| HH-43 | SEND: DOM-based клик "Откликнуться" в модалке (eval, rect.y > 500) | crit | 4 | backlog |
| HH-44 | SEND: polling "Резюме доставлено" (каждые 500мс, timeout 5с) | crit | 4 | backlog |
| HH-45 | SEND: screenshot на failure (CDP shot → /tmp/hh-fail-{id}.png) для отладки | high | 4 | backlog |
| HH-46 | SEND: detect captcha/anti-bot (проверить наличие "капча", "подтвердите", "подозрительная активность" → STOP + reason FAILED_ANTI_BOT) | crit | 4 | backlog |
| HH-47 | THROTTLE: пауза 5-10 сек (Math.random * 5 + 5) между откликами | crit | 4 | backlog |
| HH-48 | THROTTLE: лимит 30 откликов за сессию (hardcoded) | high | 4 | backlog |
| HH-49 | THROTTLE: предупреждение при sent.json > 150, подтверждение при > 180 | med | 4 | backlog |
| HH-50 | Тест Спринт 4: `/vacancy-outreach 3 --preview` отправить 3 отклика. Проверить: все 3 SENT, sent.json обновлён (atomic), дневной лог записан, throttle ~7сек между каждым. Ни одного clickxy в процессе. | crit | 4 | backlog |
| HH-51 | LOG: reason codes полный набор (SENT, SKIP_NO_TEXTAREA, SKIP_ALREADY_APPLIED, SKIP_DRY_RUN, SKIP_USER, FAILED_NO_CONFIRMATION, FAILED_CDP_ERROR, FAILED_CDP_TIMEOUT, FAILED_ANTI_BOT) | crit | 5 | backlog |
| HH-52 | LOG: дневной лог log-YYYY-MM-DD.md (все статусы: SENT, SKIP, FAILED, DRY_RUN) | high | 5 | backlog |
| HH-53 | LOG: message_preview в sent.json (первые 100 символов) | med | 5 | backlog |
| HH-54 | REPORT: итог (sent/skip/failed с breakdown по reason codes) | crit | 5 | backlog |
| HH-55 | REPORT: топ-3 вакансии по зарплате (если зарплата указана) | med | 5 | backlog |
| HH-56 | REPORT: итого за всё время (из sent.json count) | med | 5 | backlog |
| HH-57 | ERROR: graceful stop при CDP unavailable (показать частичный отчёт) | crit | 5 | backlog |
| HH-58 | ERROR: graceful stop при anti-bot detection | crit | 5 | backlog |
| HH-59 | ERROR: recovery при corrupted sent.json (restore from .bak) | high | 5 | backlog |
| HH-60 | ERROR: DOM fallback при изменении UI (скриншот + warning если кнопка не найдена) | high | 5 | backlog |
| HH-61 | Тест Спринт 5: `/vacancy-outreach 10` полный автономный цикл. Проверить: >=8 SENT, sent.json не corrupted, дневной лог полный, отчёт содержит breakdown, throttle соблюдён (>50 сек на 10 откликов). | crit | 5 | backlog |
| HH-62 | Wiki: обновить с результатами всех тестов | med | 5 | backlog |
| HH-63 | Codex review финальный (target: 10/10) | crit | 5 | backlog |

## Спринт 1: Инфраструктура (данные + аргументы + CDP)

**Цель:** надёжный фундамент - данные, аргументы, CDP соединение

**Задачи:** HH-01 .. HH-14 (14 задач)

**Deliverables:**
- SKILL.md зарегистрирован
- sent.json с валидацией, backup, atomic write
- Все CLI аргументы парсятся
- CDP: healthcheck, tab reacquisition, timeouts

**Критерий готовности (измеримый):**
- `/vacancy-outreach 1 --dry-run` запускается без ошибок
- sent.json загружается, бэкапится, содержит 64 записи
- `--query "тест"` корректно подставляется в URL
- CDP list возвращает таб, eval возвращает данные
- При недоступном CDP - сообщение "Нужен Allow в Chrome", graceful exit

## Спринт 2: Поиск и чтение

**Цель:** скилл находит вакансии, читает описания, парсит зарплату

**Задачи:** HH-15 .. HH-28 (14 задач)

**Deliverables:**
- Построение URL из аргументов
- Remote-only фильтр + override
- Exclusion list + --no-filter
- Извлечение вакансий с пагинацией
- Дедупликация
- Парсинг зарплаты (все форматы)
- Обработка 0 результатов

**Критерий готовности (измеримый):**
- `/vacancy-outreach 5 --dry-run` возвращает ровно 5 вакансий
- Каждая содержит: ID (число >0), company (непустое), title (непустое), salary (число или null), description (>100 символов)
- Ни одна не содержит "Wildberries" или "Ozon" в title
- Ни одна не дублирует ID из sent.json
- Зарплата "от 100 до 150" парсится как 125000
- Зарплата "$2000" парсится как 200000
- Зарплата "gross" парсится как net * 0.87
- При 0 результатах - сообщение "0 вакансий найдено", exit

## Спринт 3: Генерация текста

**Цель:** персонализированные тексты по модульному шаблону

**Задачи:** HH-29 .. HH-35 (7 задач)

**Deliverables:**
- 6 блоков шаблона работают
- 5 вариантов OPENING
- 7 модулей WHAT_I_DO
- OFFER с нормализованной ценой
- Anti-AI фильтр
- Preview flow (OK/Skip)

**Критерий готовности (измеримый):**
- `/vacancy-outreach 10 --dry-run` генерит 10 текстов
- Все 10 содержат "calendly.com/timzinin"
- Все 10 содержат название компании
- Ни один не содержит символ "—" (em dash)
- Ни один не содержит слова "инновационный", "комплексный", "синергия"
- Минимум 3 разных OPENING среди 10 текстов
- Для вакансии с зарплатой 100К - текст содержит "80К" или "80 000"
- Для вакансии без зарплаты - текст НЕ содержит конкретную цену
- `/vacancy-outreach 1 --preview` показывает текст, ждёт ввода

## Спринт 4: Отправка

**Цель:** надёжная отправка через CDP с DOM-based кликами

**Задачи:** HH-36 .. HH-50 (15 задач)

**Deliverables:**
- Все клики через DOM eval (0 clickxy)
- Все ожидания через polling (0 фиксированных sleep для UI)
- Обработка всех модалок HH
- Anti-bot detection
- Screenshot на failure
- Throttling
- Лимиты

**Критерий готовности (измеримый):**
- `/vacancy-outreach 3 --preview` отправляет 3 отклика
- Все 3 получают SENT в sent.json
- sent.json.bak существует
- Дневной лог содержит 3 строки с SENT
- grep "clickxy" в логе/коде = 0 результатов
- Время выполнения > 15 сек (throttle 5+ сек * 3)
- При FAILED - screenshot сохранён в /tmp/hh-fail-{id}.png

## Спринт 5: Логирование, отчёты, hardening

**Цель:** полный production-ready скилл

**Задачи:** HH-51 .. HH-63 (13 задач)

**Deliverables:**
- Полный набор reason codes (9 штук)
- Дневной лог со всеми статусами
- Отчёт с breakdown по reason codes
- Graceful stop при CDP и anti-bot
- Recovery при corrupted sent.json
- DOM fallback при UI изменениях
- Wiki обновлена
- Codex 10/10

**Критерий готовности (измеримый):**
- `/vacancy-outreach 10` выполняется автономно
- >=8 из 10 получают SENT
- sent.json не corrupted после прерывания (kill -9 во время записи → восстановление из .bak)
- Отчёт содержит: "Отправлено: X, Пропущено: Y (breakdown), Ошибок: Z (breakdown)"
- Codex review >= 9/10

## Timeline

| Спринт | Задач | Оценка | Примечание |
|--------|-------|--------|-----------|
| 1: Инфраструктура | 14 | ~4 часа | CDP hardening требует тестирования edge cases |
| 2: Поиск и чтение | 14 | ~4 часа | Парсинг зарплаты + пагинация + фильтры |
| 3: Генерация | 7 | ~3 часа | LLM-генерация + anti-AI + preview flow |
| 4: Отправка | 15 | ~6 часов | DOM interactions + модалки + anti-bot + polling |
| 5: Hardening | 13 | ~5 часов | Recovery + fault injection + Codex review iterations |
| **Итого** | **63** | **~22 часа** | ~4-5 сессий Claude Code |

Оценка в токенах: ~15-20M токенов (при текущем темпе ~4M/сессия).
Оценка по сессиям: 4-5 сессий по 4-5 часов.
