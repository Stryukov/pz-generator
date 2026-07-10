# Правила валидации (шаг 3)

Код в n8n (Function-node). Вход: манифест (`docs/02-manifest-schema.md`). Задача — сверить
ключевые числа, чтобы поймать ошибки извлечения из PDF.

Объём — минимальный: **пример-месяц + итоговые суммы**. Остальные месяцы поштучно не
пересчитываются.

Обозначение: `nds_mult = 1.2, если тариф "without"; иначе 1.0`.

## Проверки по `example_month`
1. `sum_hvs ≈ V_hvs × tariff_hvs × nds_mult`
2. `V_stoki == V_hvs (+ V_gvs, если has_gvs)`
3. `sum_stoki ≈ V_stoki × tariff_vo × nds_mult`
4. если `has_nvcs`: `sum_nvcs ≈ sum_stoki × k_coefficient` **и** `k_coefficient ∈ {0.5, 2.5}`
5. если `has_p203`: `sum_p203 ≈ sum_stoki × 2`
6. `month_total ≈ sum_hvs + sum_stoki (+ sum_nvcs) (+ sum_p203)` — слагаемые по флагам

## Проверки итогов
7. `charged ≈ Σ month_total` по всем месяцам
8. `balance == charged − paid`

## Допуски
- Построчные (1-6): ±0,05 руб.
- Суммовые (7-8): ±1,00 руб.
- Коэффициент K: ±0,01.

## Поведение при расхождении
- Любая непройденная проверка → обратная связь агенту извлечения (какая проверка, какое
  поле, ожидалось/получено) → повторное извлечение. Оркестрация петли — в `00-flow-n8n.md`.
- После повтора не сошлось → затронутые поля в `issues.assumption` (значение как есть +
  текст расхождения). Генерация не блокируется.
- `k_coefficient` вне {0,5; 2,5} → `assumption` с пометкой «нетипичный коэффициент, проверьте».

## Референс-реализация (набросок для Function-node)
```js
const eq = (a, b, tol) => Math.abs(a - b) <= tol;
const T = { line: 0.05, sum: 1.0, k: 0.01 };
const m = manifest;
const em = m.months.find(x => x.month === m.example_month);
const f = m.flags;
const t = m.tariffs.find(x => em.month >= x.date_from.slice(0,7) && em.month <= x.date_to.slice(0,7));
const mult = t.nds === "without" ? 1.2 : 1.0;
const fails = [];

if (!eq(em.sum_hvs, em.V_hvs * t.tariff_hvs * mult, T.line)) fails.push("1:ХВС");
if (em.V_stoki !== em.V_hvs + (f.has_gvs ? em.V_gvs : 0))     fails.push("2:V_stoki");
if (!eq(em.sum_stoki, em.V_stoki * t.tariff_vo * mult, T.line)) fails.push("3:стоки");
if (f.has_nvcs) {
  if (!eq(em.sum_nvcs, em.sum_stoki * f.k_coefficient, T.line)) fails.push("4:НВЦС");
  if (!eq(f.k_coefficient, 0.5, T.k) && !eq(f.k_coefficient, 2.5, T.k)) fails.push("4:K");
}
if (f.has_p203 && !eq(em.sum_p203, em.sum_stoki * 2, T.line)) fails.push("5:п203");
let comp = em.sum_hvs + em.sum_stoki + (f.has_nvcs ? em.sum_nvcs : 0) + (f.has_p203 ? em.sum_p203 : 0);
if (!eq(em.month_total, comp, T.line)) fails.push("6:итог_месяца");

const sumMonths = m.months.reduce((s, x) => s + x.month_total, 0);
if (!eq(m.totals.charged, sumMonths, T.sum)) fails.push("7:начислено");
if (!eq(m.totals.balance, m.totals.charged - m.totals.paid, T.sum)) fails.push("8:остаток");

return { passed: fails.length === 0, fails };
```
