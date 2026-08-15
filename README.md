# Завдання 1: SQL 

## Що робить запит

Запит будує платіжну воронку користувачів на основі `SALE`-транзакцій, визначаючи:
- **`renewal_number`** — порядковий номер циклу підписки (місяць 1, місяць 2...)
- **`retry_number`** — порядковий номер спроби оплати в межах одного `renewal_number`
- **`net_revenue`** — фактичний дохід з транзакції з урахуванням повернень і комісій

## Фінальний запит

```sql
WITH sales AS (
    SELECT
        user_id, transaction_id, project,
        plan_id, transaction_time,
        status, amount_usd,
        transaction_type, fee_usd
    FROM test_task
    WHERE transaction_type = 'SALE'
),
adjustments  AS(
    SELECT reference_transaction_id AS sale_transaction_id,
        SUM( CASE
            WHEN transaction_type IN (
                'REFUND',
                'CHARGEBACK',
                'VOID',
                'ALERT',
                'DISPUTE'
            )
            THEN COALESCE(amount_usd, 0)
            ELSE 0
        END
        ) AS adjustment_amount_usd,
        SUM(COALESCE(fee_usd, 0)) AS related_fees_usd
    FROM test_task
    WHERE reference_transaction_id IS NOT NULL
    GROUP BY reference_transaction_id
),
previous AS (
    SELECT
        *,
        LAG(transaction_time) OVER (
            PARTITION BY user_id, project, plan_id
            ORDER BY transaction_time, transaction_id
        ) AS prev_time,

        LAG(status) OVER (
            PARTITION BY user_id, project, plan_id
            ORDER BY transaction_time, transaction_id
        ) AS prev_status,

        LAG(amount_usd) OVER (
            PARTITION BY user_id, project, plan_id
            ORDER BY transaction_time, transaction_id
        ) AS prev_amount

    FROM sales
),

flagged AS (
    SELECT
        *,
        CASE
            WHEN prev_time IS NULL THEN 1
            WHEN prev_status = 'SUCCESS' THEN 1
            ELSE 0
        END AS is_new_renewal
    FROM previous
),
numbered AS (
    SELECT
        *,
        SUM(is_new_renewal) OVER (
            PARTITION BY user_id, project, plan_id
            ORDER BY transaction_time, transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS renewal_number
    FROM flagged
)
SELECT
    numbered.user_id,
    numbered.transaction_id,
    numbered.project,
    numbered.plan_id,
    numbered.transaction_time,
    numbered.status,
    numbered.renewal_number,
    ROW_NUMBER() OVER (
        PARTITION BY
            numbered.user_id,
            numbered.project,
            numbered.plan_id,
            numbered.renewal_number
        ORDER BY
            numbered.transaction_time,
            numbered.transaction_id
    ) AS retry_number,

    ROUND(
        CASE
            WHEN numbered.status = 'SUCCESS'
            THEN COALESCE(numbered.amount_usd, 0)
            ELSE 0
        END
        + COALESCE(adjustments.adjustment_amount_usd, 0)
        - COALESCE(numbered.fee_usd, 0)
        - COALESCE(adjustments.related_fees_usd, 0),
        2
    ) AS net_revenue

FROM numbered
LEFT JOIN adjustments
    ON numbered.transaction_id = adjustments.sale_transaction_id
ORDER BY
    numbered.user_id,
    numbered.project,
    numbered.plan_id,
    numbered.transaction_time,
    numbered.transaction_id;
```

---

## Хід дослідження та обґрунтування рішень

### #1. Дослідження plan_id

**Питання:** Чи багато користувачів мають різні plan_id?

```sql
SELECT user_id, COUNT(DISTINCT plan_id) AS distinct_plans, STRING_AGG(DISTINCT plan_id, ', ') AS plans
FROM sale_table
GROUP BY user_id
HAVING COUNT(DISTINCT plan_id) > 1
ORDER BY distinct_plans DESC
LIMIT 20;
```

**Інсайт:** переважна більшість юзерів (>99%) мають лише один `plan_id` за весь час, тобто партиціонування по `plan_id` практично еквівалентне партиціонуванню по `user_id`. Для рідкісних multi-plan юзерів перевірка хронології (`u_ad1ebeac25c2`) показала, що зустрічаються випадки, коли `plan_id` хаотично змінюється між **послідовними** спробами оплати з малим часовим гапом (кілька днів). Це нагадує швидкі ретраї з різними планами, а не окремі підписки.

**Рішення:** незважаючи на цей edge case, обрано партиціонування по `user_id + plan_id` (буквальне трактування умови), оскільки на масштабі всього датасету це не спотворює результат, а рідкісні винятки задокументовано як відоме обмеження.

### #2. Визначення межі `renewal_number`: чому саме 14 днів

**Питання:** як відрізнити "новий цикл підписки" від "retry в межах поточного циклу"?

```sql
WITH success_only AS (
    SELECT user_id, plan_id, transaction_time,
           LAG(transaction_time) OVER (PARTITION BY user_id, plan_id ORDER BY transaction_time) AS prev_time
    FROM sale_table
    WHERE status = 'SUCCESS'
)
SELECT
    APPROX_QUANTILE(DATE_DIFF('day', prev_time, transaction_time), 0.25) AS p25_days,
    APPROX_QUANTILE(DATE_DIFF('day', prev_time, transaction_time), 0.5) AS median_days,
    APPROX_QUANTILE(DATE_DIFF('day', prev_time, transaction_time), 0.75) AS p75_days
FROM success_only
WHERE prev_time IS NOT NULL;
```

**Результат:** p25=21, медіана=29, p75=37 днів.

**Інсайт:** навіть нижній квартиль реальних інтервалів між успішними ребілами (21 день) суттєво перевищує 14 днів. Тобто, спочатку працювало, але згодом прибрала цю логіку розбиття воронки, бо все-таки можуть бути платежі, які часовим проміжком не розділиш правильно.

### #3. Визначення межі `renewal_number`: чому обрано лише статус, без часового порогу

**Розглянуті варіанти:**
1. Чистий часовий поріг (гап > N днів)
2. Комбінація часу і зміни ціни (trial→regular price)
3. Статус попередньої транзакції (`SUCCESS` завжди закриває цикл)

**Чому відхилено часовий поріг:**
Перевірка розподілу інтервалів між успішними ребілами показала p25=21, медіана=29, p75=37 днів, це дало б обґрунтування для порогу (напр. 14 днів). Але подальше дослідження конкретних кейсів (`u_010dd86cd5b3`) показало межові випадки: перехід із trial-ціни на повну ($15.28 → $61.18) стається з гапом усього 16 днів — близько до будь-якого розумного порогу, і будь-яке конкретне число (14, 20, 25 днів) залишається довільним рішенням без чіткого обґрунтування межі.

**Фінальне рішення:** прибрано часовий поріг, залишено єдине, логічно обґрунтоване правило — **`SUCCESS` завжди завершує поточний renewal-цикл**, і будь-яка наступна транзакція (незалежно від часу) починає новий цикл. Це узгоджується з природою даних: успішна оплата = підписка активна на новий період, тому наступна спроба списання завжди відноситься до наступного періоду. Серія `DECLINED → DECLINED → ...` без жодного SUCCESS між ними трактується як один безперервний retry-цикл поточного renewal, незалежно від того, скільки часу тривала ця серія.

**Перевірена проблема цього підходу:** на межових прикладах (`u_00072477fff5`, дві SUCCESS-транзакції з різницею лише 2 години) правило коректно, хоч і несподівано, визначає це як два окремі renewal — технічно правильно за визначенням правила (SUCCESS завжди закриває цикл), хоча в реальності це можуть бути дублікати чи швидка повторна оплата. Це задокументовано як відоме обмеження.
### #4. Чи використовувати `reference_transaction_id` для визначення retry-меж

```sql
SELECT
    transaction_type,
    COUNT(*) AS total,
    COUNT(reference_transaction_id) AS with_reference,
    COUNT(reference_transaction_id) * 100.0 / COUNT(*) AS pct_with_reference
FROM test_task
GROUP BY transaction_type;
```

**Інсайт:** `SALE`-транзакції мають **0%** заповнення `reference_transaction_id` — це поле використовується виключно для зв'язку REFUND/CAPTURE/CHARGEBACK/VOID/ALERT/DISPUTE із батьківською SALE, а не для зв'язку retry-спроб між собою. Тому розмежування renewal/retry обов'язково будується на комбінації часу і статусу, а не на прямих reference-зв'язках.

### #5. Розрахунок `net_revenue`: чому саме така формула
