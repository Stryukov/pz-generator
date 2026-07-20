# Правила валидации (шаг 3)

Код в n8n (Function-node). Вход: манифест (`docs/02-manifest-schema.md`). Задача — сверить
ключевые числа, чтобы поймать ошибки извлечения из PDF.

Объём — минимальный: **пример-месяц + итоговые суммы**. Остальные месяцы поштучно не
пересчитываются.

Обозначение: `nds_mult = 1.2, если тариф "without"; иначе 1.0`.

> Суммы в манифесте — **с НДС**; тариф в расчёте — без НДС, поэтому строчные проверки
> умножают на `nds_mult`. Объём стоков (`V_stoki`) задаётся в расчёте **отдельно** и не
> выводится из `V_hvs` (бывшая проверка 2 убрана). Нет `months`/`tariffs`/`example_month`
> или тарифа под месяц → `schema:*` (понятная обратная связь петле, а не краш).

## Проверки по `example_month`
1. `sum_hvs ≈ V_hvs × tariff_hvs × nds_mult`
3. `sum_stoki ≈ V_stoki × tariff_vo × nds_mult`
4. если `has_nvcs`: `sum_nvcs ≈ sum_stoki × k_coefficient` **и** `k_coefficient ∈ {0.5, 2.5}`
5. если `has_p203`: `sum_p203` заполнен *(формула п.203 уточняется у эксперта — пока только наличие)*
6. `month_total ≈ sum_hvs + sum_stoki (+ sum_nvcs) (+ sum_p203)` — слагаемые по флагам

## Проверки итогов
7. `charged ≈ Σ month_total` по всем месяцам
8. `balance == charged − paid`

## Допуски
- Построчные (1-6): ±0,05 руб.
- Суммовые (7-8): ±1,00 руб.
- Коэффициент K: ±0,01.

## Поведение при расхождении
- Любая непройденная проверка → обратная связь агенту извлечения → повторное извлечение.
  Оркестрация петли — в `00-flow-n8n.md`.
- **Fails — информативные строки, не коды**: «что проверяли, ожидалось X (формула), получено Y»
  + подсказка, если распознан типовой промах (напр. сумма ≈ V×тариф без ×1,2 → «взята колонка
  БЕЗ НДС, возьми итоговую “с НДС”»; итог месяца не сходится со слагаемыми → «строки взяты не
  из тех колонок»). Эти строки уходят в промпт повторного извлечения как есть — от их
  детальности зависит, починит ли петля.
- После повтора не сошлось → затронутые поля в `issues.assumption` (значение как есть +
  текст расхождения). Генерация не блокируется.
- `k_coefficient` вне {0,5; 2,5} → `assumption` с пометкой «нетипичный коэффициент, проверьте».

## Референс-реализация (набросок для Function-node)
```js
const eq = (a, b, tol) => Math.abs(a - b) <= tol;
const r2 = (x) => Math.round(x * 100) / 100;
const T = { line: 0.05, sum: 1.0, k: 0.01 };
const m = manifest;
const fails = [];
try {
  if (!m || !Array.isArray(m.months) || !m.months.length) throw new Error("нет months[]");
  if (!m.example_month) throw new Error("нет example_month");
  const em = m.months.find(x => x.month === m.example_month);
  if (!em) throw new Error("example_month отсутствует в months");
  const f = m.flags || {};
  const t = (m.tariffs || []).find(x => em.month >= x.date_from.slice(0,7) && em.month <= x.date_to.slice(0,7));
  if (!t) { fails.push("schema: не найден тарифный интервал для месяца " + em.month + " — проверь tariffs[].date_from/date_to"); }
  else {
    const mult = t.nds === "without" ? 1.2 : 1.0;   // суммы в манифесте — с НДС
    const expHvs = r2(em.V_hvs * t.tariff_hvs * mult);
    if (!eq(em.sum_hvs, expHvs, T.line)) {
      let hint = "";
      if (eq(em.sum_hvs, r2(em.V_hvs * t.tariff_hvs), T.line)) hint = " — похоже, сумма взята из колонки БЕЗ НДС; возьми итоговую колонку «с НДС» (без НДС × 1,2)";
      fails.push("1:ХВС — ожидалось " + expHvs + " (V_hvs " + em.V_hvs + " × тариф " + t.tariff_hvs + " × 1,2), получено " + em.sum_hvs + hint);
    }
    const expSt = r2(em.V_stoki * t.tariff_vo * mult);
    if (!eq(em.sum_stoki, expSt, T.line)) {
      let hint = "";
      if (eq(em.sum_stoki, r2(em.V_stoki * t.tariff_vo), T.line)) hint = " — похоже, сумма взята из колонки БЕЗ НДС; возьми итоговую колонку «с НДС» (без НДС × 1,2)";
      fails.push("3:стоки — ожидалось " + expSt + " (V_stoki " + em.V_stoki + " × тариф " + t.tariff_vo + " × 1,2), получено " + em.sum_stoki + hint);
    }
  }
  if (f.has_nvcs) {
    const expN = r2(em.sum_stoki * f.k_coefficient);
    if (!eq(em.sum_nvcs, expN, T.line)) fails.push("4:НВЦС — ожидалось " + expN + " (sum_stoki " + em.sum_stoki + " × K " + f.k_coefficient + "), получено " + em.sum_nvcs);
    if (!eq(f.k_coefficient, 0.5, T.k) && !eq(f.k_coefficient, 2.5, T.k)) fails.push("4:K — нетипичный коэффициент " + f.k_coefficient + " (ожидается 0,5 или 2,5); K = sum_nvcs / sum_stoki");
  }
  if (f.has_p203 && em.sum_p203 == null) fails.push("5:п203 — has_p203=true, но sum_p203 пуст: возьми итоговую колонку п.203 «с НДС»");
  const comp = r2(em.sum_hvs + em.sum_stoki + (f.has_nvcs ? em.sum_nvcs : 0) + (f.has_p203 ? (em.sum_p203 || 0) : 0));
  if (!eq(em.month_total, comp, T.line)) fails.push("6:итог_месяца — ожидалось " + comp + " (сумма слагаемых), получено " + em.month_total + " — если не сходится, строки взяты не из тех колонок (без НДС vs с НДС)");
  const sumMonths = r2(m.months.reduce((s, x) => s + x.month_total, 0));
  if (!eq(m.totals.charged, sumMonths, T.sum)) fails.push("7:начислено — ожидалось " + sumMonths + " (сумма month_total всех месяцев), получено " + m.totals.charged);
  if (!eq(m.totals.balance, m.totals.charged - m.totals.paid, T.sum)) fails.push("8:остаток — ожидалось " + r2(m.totals.charged - m.totals.paid) + " (charged − paid), получено " + m.totals.balance);
} catch (e) { fails.push("schema:" + e.message); }

return { passed: fails.length === 0, fails };
```
