# n8n workflow

`pz-generator.workflow.json` — импортируемый флоу. Импорт: n8n → Workflows → Import from File.

## Принцип
Контракты и структура **не хранятся во флоу**. Узел `Load Contracts` тянет их из репозитория
по `docs_base_url`. Правки вносятся в `.md` в git, флоу остаётся неизменным. Промпты агентов
намеренно простые — вся логика в контрактах.

## Настроить перед запуском
1. **Config** (Set-узел):
   - `docs_base_url` — raw-ссылка на папку `docs` репозитория
     (напр. `https://raw.githubusercontent.com/USER/pz-generator/main/docs`).
   - `model` — доступная вам модель Anthropic.
   - `self_email`, `subject_tag` (`[ПЗ]`).
   - `converter_url` — endpoint HTML→DOCX (если используете; иначе оставить пустым).
2. **Креды:**
   - IMAP — входящая почта (узел `Email Trigger`).
   - SMTP — ответы (`Reply *`).
   - Anthropic — HTTP Header Auth с заголовком `x-api-key` = ваш ключ (все агенты).
3. **HTML→DOCX** — узел отключён по умолчанию: записка вложится как HTML. Чтобы получать DOCX,
   включите узел и задайте `converter_url` (Gotenberg/CloudConvert/свой сервис).

## Схема
```
Email Trigger → Config → Filter Subject ─true→ Prepare Inputs → Has Calculation ─true→ Load Contracts
                                     └false→ Ignore                         └false→ Reply Need Calc
Load Contracts → Extraction Agent → Parse Manifest → Validate → Passed?
   ├─true→  Generation Agent → Parse Output → [HTML→DOCX] → Reply With Zapiska
   └─false→ Re-extract → Parse Manifest 2 → Validate 2 → Mark Assumptions → Generation Agent
```

## Замечания
- Модели передаётся расчёт как PDF-документ (base64) — модель должна поддерживать документы.
- Петля повтора однократная (повтор → допущение), как в `../docs/03-validation-rules.md`.
- Узлы помечены `REPLACE_*` в кредах — привяжите свои учётные данные при импорте.
- Обработка вложений/бинарных данных может требовать подстройки под вашу версию n8n.
