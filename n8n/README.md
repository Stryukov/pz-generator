# n8n workflow

`pz-generator.workflow.json` — импортируемый флоу. Импорт: n8n → Workflows → Import from File.

## Принцип
Контракты и структура **не хранятся во флоу**. Узел `Load Contracts` тянет их из репозитория
по `docs_base_url`. Правки вносятся в `.md` в git, флоу остаётся неизменным. Промпты агентов
намеренно простые — вся логика в контрактах.

Три агента (извлечение, повтор, генерация) — узлы `Basic LLM Chain`, питаются от одного
под-узла модели. Во флоу **два** модельных узла:
- **`Router AI (OpenAI)`** — подключён по умолчанию: OpenAI-совместимый роутер
  (https://routerai.ru/v1), модель `anthropic/claude-sonnet-5`, temperature 0, maxTokens 10000,
  **Use Responses API = OFF**.
- **`Gemma (Ollama)`** — локальный запасной вариант, отвязан. Переключение модели =
  перекинуть связь `ai_languageModel` с одного узла на другой.

## Настроить перед запуском
1. **Config** (Set-узел):
   - `docs_base_url` — raw-ссылка на папку `docs` репозитория
     (напр. `https://raw.githubusercontent.com/USER/pz-generator/main/docs`).
   - `self_email`, `subject_tag` (`[ПЗ]`).
   - `converter_url` — endpoint HTML→DOCX (если используете; иначе оставить пустым).
2. **Креды:**
   - IMAP — входящая почта (узел `Email Trigger`).
   - SMTP — ответы (`Reply *`).
   - routerai — OpenAI-креденшл с Base URL `https://routerai.ru/v1` (узел `Router AI (OpenAI)`).
   - Ollama — локальный сервер (узел `Gemma (Ollama)`, запасной; модель `gemma3:27b`).
3. **HTML→DOCX** — конвертация через **pandoc** (Execute Command; pandoc должен быть установлен
   в контейнере n8n). Форматирование DOCX задаёт **`templates/reference-type-A.docx`**
   (Times New Roman 14, по ширине, поля 2/1,5/2/3 см; плейсхолдеры — красный Consolas):
   узел `Fetch RefDoc` скачивает его из репозитория, pandoc получает `--reference-doc`.
   Правите стили в reference-файле в git → вид DOCX меняется без правки флоу.

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
