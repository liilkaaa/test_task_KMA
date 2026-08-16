# Завдання 1: SQL 

## Що робить запит

Запит будує платіжну воронку користувачів на основі `SALE`-транзакцій, визначаючи:
- **`renewal_number`** — порядковий номер циклу підписки
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
ORDER BY distinct_plans DESC;
```

**Інсайт:** переважна більшість юзерів мають лише один `plan_id` за весь час, тобто розділення по `plan_id` дорівнює розділенню по `user_id`. Для рідкісних multi-plan юзерів перевірка хронології (`u_ad1ebeac25c2`) показала, що зустрічаються випадки, коли `plan_id` змінюється між **послідовними** спробами оплати з малим часовим гапом. Це нагадує швидкі ретраї з різними планами, а не окремі підписки. Але, можливо, це і спроби купити різні плани.
<img width="1482" height="359" alt="Screenshot 2026-08-16 094102" src="https://github.com/user-attachments/assets/bfdf9d11-c3e7-4b19-96da-5ad0c22764ed" />


**Рішення:** незважаючи на цей edge case, обрано партиціонування по `user_id + plan_id`, оскільки на масштабі всього датасету це не спотворює результат.

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
2. Комбінація часу і зміни ціни (trial -> regular price)
3. Статус попередньої транзакції (`SUCCESS` завжди закриває цикл)

**Чому відхилено часовий поріг:**
Перевірка розподілу інтервалів між успішними ребілами показала p25=21, медіана=29, p75=37 днів, це дало б обґрунтування для порогу (напр. 14 днів). Але подальше дослідження конкретних кейсів (`u_010dd86cd5b3`) показало межові випадки: перехід із trial-ціни на повну ($15.28 → $61.18) стається з гапом усього 16 днів. У результаті, можуть виникнути платежі і менші за 16 днів, тому краще тут не опиратись на час.
<img width="1477" height="496" alt="Screenshot 2026-08-16 094255" src="https://github.com/user-attachments/assets/474114db-f01e-4b22-ab72-7a10f0de2d5f" />


**Фінальне рішення:** прибрано часовий поріг, залишено єдине правило **`SUCCESS` завжди завершує поточний renewal-цикл**, і будь-яка наступна транзакція (незалежно від часу) починає новий цикл. . Серія `DECLINED -> DECLINED -> ...` без жодного SUCCESS між ними трактується як один безперервний retry-цикл поточного renewal, незалежно від того, скільки часу тривала ця серія.

**Перевірена проблема цього підходу:** на межових прикладах (`u_00072477fff5`, дві SUCCESS-транзакції з різницею лише 2 години) правило коректно, хоч і визначає це як два окремі renewal. Тобто це технічно правильно (SUCCESS завжди закриває цикл), хоча в реальності це можуть бути дублікати чи швидка повторна оплата. Це може бути обмеженням

<img width="1490" height="155" alt="Screenshot 2026-08-16 094640" src="https://github.com/user-attachments/assets/05955ee4-c75a-4732-9dd9-d49f024420f0" />

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
<img width="1454" height="318" alt="Screenshot 2026-08-16 094725" src="https://github.com/user-attachments/assets/4ef5679d-be50-483e-a150-1f8241b24708" />

**Інсайт:** `SALE`-транзакції мають **0%** заповнення `reference_transaction_id`. Це поле використовується виключно для зв'язку REFUND/CAPTURE/CHARGEBACK/VOID/ALERT/DISPUTE із батьківською SALE, а не для зв'язку retry-спроб між собою.

### #5. Розрахунок `net_revenue`: чому саме така формула
**Перевірка знаку `amount_usd`:**
```sql
SELECT transaction_type, MIN(amount_usd), MAX(amount_usd), AVG(amount_usd)
FROM test_task
GROUP BY transaction_type;
```
<img width="1439" height="324" alt="Screenshot 2026-08-16 094946" src="https://github.com/user-attachments/assets/4a3395aa-6c86-44a8-9423-8fc168eafa33" />


**Інсайт:** REFUND, CHARGEBACK, VOID, DISPUTE, ALERT завжди мають `amount_usd <= 0`, тобто повернення коштів вже записані з від'ємним знаком у даних. Це підтверджує, що правильна операція це **додавання** (не віднімання) `adjustment_amount_usd` до суми.

**Роль CAPTURE:**
```sql
SELECT s.status AS sale_status, COUNT(*), SUM(c.amount_usd)
FROM test_task c
JOIN test_task s ON c.reference_transaction_id = s.transaction_id
WHERE c.transaction_type='CAPTURE'
GROUP BY s.status;
```
<img width="1111" height="146" alt="Screenshot 2026-08-16 095046" src="https://github.com/user-attachments/assets/0670406b-4882-413a-9fc9-d6aeb3e84297" />

**Інсайт:** `CAPTURE.amount_usd` завжди дорівнює 0. CAPTURE є суто технічним маркером двостадійного флоу, без власної фінансової ваги. **Але** `CAPTURE.fee_usd` ненульовий, тобто платіжна система бере окрему комісію за факт захоплення коштів. Тому в підсумковій формулі суми CAPTURE ігноруються, а комісії враховуються.


## Верифікація результату
- Кількість рядків у результаті = кількість SALE-транзакцій у вихідних даних (347 031 = 347 031)
- Унікальність `transaction_id`: 0 дублікатів
- `retry_number` послідовний (1, 2, 3...) у кожній групі `(user_id, project, plan_id, renewal_number)`
- `renewal_number` послідовний у кожній групі `(user_id, project, plan_id)`
- Жодних DECLINED-транзакцій з `net_revenue > 0`
- Жодних SUCCESS-транзакцій з `net_revenue > amount_usd`



# Завдання 2, частина 1: EDA

Повний аналіз — у `EDA.ipynb`.

## Ключові висновки

- Датасет містить 463 521 транзакцію за період 25.12.2024–09.12.2025 (майже рік), 7 типів транзакцій (SALE домінує - 75%)
- Усі пропуски в даних логічно пояснюються природою транзакцій (тип транзакції, тип платіжного методу)
- Причини відмов (`decline_code`) доміновані двома кодами (7322, 4670)
- **Success Rate суттєво варіюється за платіжним провайдером і типом платіжного методу**
- Знайдено outlier: одна SALE-транзакція мала 7 повʼязаних ALERT-записів, що суттєво занизило її net_revenue - але це не закономірність у всіх даних

# Завдання 2, частина 2: High-level Payment Dashboard

**Інструмент:** Tableau

## Посилання на Дашборд
https://public.tableau.com/shared/Q5N7SNBMP?:display_count=n&:origin=viz_share_link
<img width="2034" height="1745" alt="Dashboard (4)" src="https://github.com/user-attachments/assets/d632ac7e-9586-4861-8100-a5b0ffb9c1ea" />


*Примітка: дані охоплюють період 25.12.2024 по 09.12.2025. Грудень 2025 містить дані лише до 9 числа, тому це неповний місяць. Через це в кінці графіків "Net Revenue over time" і "Fee Overhead % over time" видно різкий спад. Це не реальне падіння показників, а просто наслідок того, що дані за грудень обірвані на середині місяця.*

---

## Структура дашборда

KPI Cards: Success Rate | Net Revenue | Refund Rate | Chargeback Rate | Fee Overhead
↓
Success Rate by PSP | Net Revenue over time
↓
Top Decline Reasons | Refund Rate by PSP
↓
Chargeback Rate by PSP | Fee Overhead % over time


Спочатку йдуть загальні KPI картки, які одразу показують загальний стан справ. Далі метрики розбиті по платіжних провайдерах (PSP), щоб було видно, де саме проблема. Потім йде тренд у часі і причини відмов.

---

## Метрики

### 1. Success Rate (Authorization Rate)

Формула застосовується на аркуші, відфільтрованому лише на транзакції типу SALE.


SUM(IF [Status] = 'SUCCESS' THEN 1 ELSE 0 END)
/
COUNT([Transaction Id])

**Чому важливо.** Ця метрика показує, скільки зі спроб оплатити пройшли успішно. Якщо число падає, значить десь щось не так, і компанія втрачає гроші ще до того, як це стане помітно у загальній виручці.

**Поточне значення:** 63.02%

### 2. Success Rate by PSP

Той самий розрахунок, але окремо для кожного платіжного провайдера.

**Головна знахідка.** Один провайдер (Vertex Ltd) пропускає 99% платежів, а інший (Global Payments) лише 47%. Це вже конкретна підказка команді, куди звернути увагу.

### 3. Net Revenue

SUM(IF [Transaction Type]='SALE' AND [Status]='SUCCESS' THEN [Amount Usd] ELSE 0 END)
+ SUM(IF [Transaction Type] IN ('REFUND','CHARGEBACK','VOID','ALERT','DISPUTE') THEN [Amount Usd] ELSE 0 END)
- SUM([Fee Usd])

**Чому важливо.** Це гроші, які компанія реально заробила, а не просто сума всіх успішних платежів.

**Поточне значення:** $8,986,554.79

### 4. Refund Rate
COUNTD(IF [Transaction Type] = 'REFUND' THEN [Reference Transaction Id] END)
/
SUM(IF [Transaction Type] = 'SALE' AND [Status] = 'SUCCESS' THEN 1 ELSE 0 END)

**Чому важливо.** Показує, яка частка людей, що заплатили, потім передумали і забрали гроші назад. Це сигнал того, наскільки клієнти задоволені продуктом.

**Поточне значення:** 4.12%

### 5. Chargeback Rate

COUNTD(IF [Transaction Type] = 'CHARGEBACK' THEN [Reference Transaction Id] END)
/
SUM(IF [Transaction Type] = 'SALE' AND [Status] = 'SUCCESS' THEN 1 ELSE 0 END)

**Чому важливо.** Chargeback це коли клієнт іде не до компанії за поверненням, а прямо до свого банку і каже, що не визнає цю оплату. Це гірше за звичайний рефанд, бо якщо таких випадків забагато, Visa чи Mastercard можуть почати штрафувати компанію або взагалі обмежити можливість приймати картки. Тобто це реальний ризик для бізнесу.

**Поточне значення:** 0.55%

### 6. Fee Overhead

SUM([Fee Usd])
/
SUM(IF [Transaction Type] = 'SALE' AND [Status] = 'SUCCESS' THEN [Amount Usd] ELSE 0 END)

**Чому важливо.** Показує, яку частку від кожного зароблженого долара забирають комісії платіжних систем. Якщо цифра починає рости, варто подумати, чи не платить компанія забагато конкретному провайдеру.

**Поточне значення:** 4.66%.

### 7. Top Decline Reasons

Рахує кількість унікальних транзакцій, згрупованих по коду відмови, тільки серед тих, що не пройшли.

**Чому важливо.** Коли платіж не проходить, система записує код причини. Дивлячись на найпопулярніші коди, можна зрозуміти, чи проблема масова і куди дивитись у першу чергу.

---

## Фільтри і параметри

| Фільтр | Поле | Навіщо потрібен |
|---|---|---|
| PSP Name | `psp_name` | Подивитись на конкретного платіжного провайдера |
| Payment Method Type | `payment_method_type` | card, apple_pay, google_pay або paypal |
| Project | `project` | Nexora, Flowvia або Corely |
| Month of Transaction Time | `transaction_time` | Звузити період аналізу |

---

## Рекомендації

### Яких даних не вистачає

* Довідник кодів decline_code. Без розшифровки неможливо зрозуміти, які відмови можна виправити повторною спробою, а які ні.
* Дані про те, коли людина відкрила форму оплати. Наявні дані показують лише вже здійснені спроби оплатити. Тому було б добре порахувати, скільки людей взагалі дійшли до кнопки оплатити, але передумали.
* Явний статус підписки (активна, скасована). Без нього складно точно порахувати, скільки людей перестали платити саме через проблеми з оплатою, а не через власне рішення.

### Які ще метрики варто додати

* Retry Success Rate. Показує, чи допомагають повторні спроби оплати перетворити відмову на успіх. Для цього потрібні дані з SQL-запиту (Завдання 1), бо в початкових даних нема інформації про це.
* Payment Provider Uptime. Показує, скільки часу платіжна система технічно доступна і працює без збоїв. У наших даних немає логів доступності системи, тому цю метрику неможливо порахувати з наявного датасету.
* Fraud Rate. Показує частку транзакцій, пов'язаних з ALERT, як індикатор потенційно шахрайської активності. Оскільки ALERT не означає підтверджений fraud, для кращого розрахунку Fraud Rate потрібні окремі fraud-рішення / fraud labels.

---
