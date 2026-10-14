# Vacancy Outreach - Архитектура

## Обзор

```mermaid
graph TD
    A[Пользователь: /vacancy-outreach 20] --> B[Claude Code Skill]
    B --> C[Chrome CDP]
    C --> D[HH.ru в Chrome]
    B --> E[sent.json]
    B --> F[Дневной лог]
    D --> G[Поиск вакансий]
    D --> H[Чтение описания]
    D --> I[Отправка отклика]
    B --> J[Генерация текста LLM]
```

## Компоненты

### 1. Скилл (SKILL.md)
- Расположение: `~/.claude/skills/vacancy-outreach/SKILL.md`
- Тип: Claude Code skill (markdown-спецификация)
- Claude читает SKILL.md и выполняет инструкции
- Нет исполняемого кода - только спецификация поведения

### 2. Chrome CDP
- Расположение: `~/.claude/skills/chrome-cdp/scripts/cdp.mjs`
- Node.js 22+ (путь: `~/.nvm/versions/node/v22.22.1/bin/node`)
- WebSocket подключение к Chrome DevTools Protocol
- Требует: Chrome с включённым remote debugging

### 3. Хранилище данных
- Расположение: `~/hh-outreach-data/`
- `sent.json` - история всех откликов (source of truth)
- `log-YYYY-MM-DD.md` - дневные логи (человекочитаемые)

### 4. LLM генерация
- Claude (Opus 4.6) сам генерит текст на основе описания вакансии
- Шаблон в SPEC.md: модульная структура (opening + reaction + offer + what_i_do + slots + cta)
- Без внешних LLM вызовов

## Взаимодействие с HH.ru

```mermaid
sequenceDiagram
    participant C as Claude Code
    participant CDP as Chrome CDP
    participant HH as HH.ru (Chrome)
    
    C->>CDP: list (найти HH таб)
    CDP->>HH: WebSocket
    CDP-->>C: tab ID
    
    C->>CDP: nav(tab, search_url)
    CDP->>HH: navigate
    HH-->>CDP: page loaded
    
    C->>CDP: eval(tab, extract_vacancies_js)
    CDP->>HH: execute JS
    HH-->>CDP: vacancy list
    CDP-->>C: [{id, title, company}]
    
    loop Для каждой вакансии
        C->>CDP: nav(tab, vacancy_url)
        CDP->>HH: navigate
        C->>CDP: eval(tab, extract_description_js)
        HH-->>CDP: description text
        CDP-->>C: description
        
        Note over C: Claude генерит кастомный текст
        
        C->>CDP: eval(tab, click_respond_js)
        CDP->>HH: click Откликнуться
        C->>CDP: eval(tab, click_add_cover_js)
        CDP->>HH: click Добавить сопроводительное
        C->>CDP: type(tab, cover_letter)
        CDP->>HH: input text
        C->>CDP: eval(tab, click_send_js)
        CDP->>HH: click Откликнуться (в модалке)
        
        C->>C: log to sent.json
        C->>C: throttle 5-10 sec
    end
    
    C->>C: generate report
```

## State Management

```mermaid
stateDiagram-v2
    [*] --> INIT: /vacancy-outreach N
    INIT --> SEARCH: CDP доступен
    INIT --> ERROR: CDP недоступен
    SEARCH --> READ: вакансии найдены
    SEARCH --> REPORT: 0 вакансий
    READ --> SKIP: уже откликались
    READ --> GENERATE: новая вакансия
    GENERATE --> DRY_RUN: --dry-run
    GENERATE --> PREVIEW: --preview
    GENERATE --> SEND: полный режим
    PREVIEW --> SEND: пользователь OK
    PREVIEW --> SKIP: пользователь skip
    SEND --> SENT: доставлено
    SEND --> FAILED: ошибка
    SENT --> THROTTLE
    FAILED --> THROTTLE
    SKIP --> THROTTLE
    DRY_RUN --> THROTTLE
    THROTTLE --> READ: ещё вакансии
    THROTTLE --> REPORT: все обработаны
    REPORT --> [*]
    ERROR --> [*]
```

## Зависимости

| Зависимость | Версия | Назначение |
|-------------|--------|-----------|
| Node.js | 22+ | CDP скрипт |
| Chrome | любая | Браузер с remote debugging |
| cdp.mjs | текущая | CDP клиент |
| Claude Code | Opus 4.6 | LLM + оркестрация |
| gh CLI | авторизован | GitHub (опционально) |
