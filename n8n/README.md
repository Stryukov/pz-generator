# n8n workflow

`pz-generator.workflow.json` — импортируемый флоу. Импорт: n8n → Workflows → Import from File.

## Принцип
Контракты и структура **не хранятся во флоу**. Узел `Load Contracts` тянет их из репозитория
по `docs_base_url`. Правки вносятся в `.md` в git, флоу остаётся неизменным. Промпты агентов
намеренно простые — вся логика в контрактах.

Модель — **локальная Gemma через Ollama**. Три агента (извлечение, повтор, генерация) — узлы
`Basic LLM Chain`, подключённые к одному под-узлу `Gemma (Ollama)`. Модель задана жёстко в этом
узле (не в Config).

## Настроить перед запуском
1. **Config** (Set-узел):
   - `docs_base_url` — raw-ссылка на папку `docs` репозитория
     (напр. `https://raw.githubusercontent.com/USER/pz-generator/main/docs`).
   - `self_email`, `subject_tag` (`[ПЗ]`).
   - `converter_url` — endpoint HTML→DOCX (если используете; иначе оставить пустым).
2. **Креды:**
   - IMAP — входящая почта (узел `Email Trigger`).
   - SMTP — ответы (`Reply *`).
   - Ollama — локальный сервер (узел `Gemma (Ollama)`); модель по умолчанию `gemma3`.
3. **HTML→DOCX** — узел отключён по умолчанию: записка вложится как HTML. Чтобы получать DOCX,
   включите узел и задайте `converter_url` (Gotenberg/CloudConvert/свой сервис).
   Для self-hosted рекомендуется **Gotenberg** отдельным контейнером рядом (данные не уходят
   наружу — важно для судебных документов); тогда
   `converter_url = http://gotenberg:3000/forms/libreoffice/convert`.

## Схема
```
Email Trigger → Config → Filter Subject ─true→ Prepare Inputs → Has Calculation ─true→ Load Contracts
                                     └false→ Ignore                         └false→ Reply Need Calc
Load Contracts → Extract Calc Text → Extraction Agent → Parse Manifest → Validate → Passed?
   ├─true→  Generation Agent → Parse Output → [HTML→DOCX] → Reply With Zapiska
   └─false→ Re-extract → Parse Manifest 2 → Validate 2 → Mark Assumptions → Generation Agent
Gemma (Ollama) ─(ai_languageModel)→ Extraction Agent / Re-extract / Generation Agent
```

## Замечания
- Расчёт (PDF; в будущем xlsx) не передаётся модели документом — узел `Extract Calc Text`
  извлекает текст, он и уходит в промпт (Gemma — текстовая модель). Для сканов без текстового
  слоя понадобится OCR либо vision-вариант Gemma 3 (рендер страниц в изображения).
- `Filter Subject` фильтрует только по тегу темы; проверка отправителя убрана (в топологии
  «отдельный ящик ПЗ на приём → ответ на ящик исполнителя» петли нет).
- Петля повтора однократная (повтор → допущение), как в `../docs/03-validation-rules.md`.
- Узлы помечены `REPLACE_*` в кредах — привяжите свои учётные данные при импорте.
- Обработка вложений/бинарных данных может требовать подстройки под вашу версию n8n.
