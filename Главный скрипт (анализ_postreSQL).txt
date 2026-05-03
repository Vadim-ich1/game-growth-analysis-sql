-- Поработаем с таблицей *users*.

-- - Сколько всего пользователей в нашей базе?
-- - Есть ли пользователи, которые регистрировались больше одного раза?
-- - Если да, то сколько их и по сколько раз они регистрировались?
-- - Какое минимальное и максимальное время регистрации в нашей витрине?
-- - Есть ли клиенты с неопределенным временем регистрации?
-- - Растет ли наша пользовательская база с течением времени?
-- - Какой месяц показал наибольшее количество зарегистрированных пользователей?

-- => Общее число записей и уникальных пользователей
SELECT COUNT(*),
       COUNT(DISTINCT id_user)
FROM skygame.users;

-- => Пользователи с несколькими регистрациями (группируем, фильтруем >=2)
SELECT COUNT(reg_date) AS cnt_regs,
       id_user
FROM skygame.users
GROUP BY id_user
HAVING COUNT(reg_date) >= 2
ORDER BY cnt_regs DESC;

-- => Минимальная и максимальная дата регистрации
SELECT MIN(reg_date) AS min_reg_date,
       MAX(reg_date) AS max_reg_date
FROM skygame.users;

-- => Количество записей с пустой датой регистрации (NULL)
SELECT SUM(CASE WHEN reg_date IS NULL THEN 1 ELSE 0 END)
FROM skygame.users;

-- => Динамика регистраций по месяцам
SELECT DATE_TRUNC('month', reg_date),
       COUNT(id_user)
FROM skygame.users
GROUP BY DATE_TRUNC('month', reg_date)
ORDER BY DATE_TRUNC('month', reg_date);

-- => Месяц с максимальным числом новых пользователей
SELECT DATE_TRUNC('month', reg_date),
       COUNT(id_user)
FROM skygame.users
GROUP BY DATE_TRUNC('month', reg_date)
ORDER BY COUNT(id_user) DESC;


-- Перейдем к таблице *game_sessions* и исследуем количество и качество игровых сессий наших пользователей.

-- => Общее число сессий, число "длинных" (>5 мин) и их доля
SELECT COUNT(start_session) AS cnt_session,
       SUM(CASE WHEN end_session - start_session > INTERVAL '5 minute' THEN 1 ELSE 0 END) AS long_session,
       SUM(CASE WHEN end_session - start_session > INTERVAL '5 minute' THEN 1.0 ELSE 0.0 END)/COUNT(start_session) AS long_session_part
FROM skygame.game_sessions;

-- => Помесячная динамика: всего сессий, длинных (>5 мин), их доля
SELECT DATE_TRUNC('month', start_session),
       COUNT(start_session) AS cnt_session,
       SUM(CASE WHEN end_session - start_session > INTERVAL '5 minute' THEN 1 ELSE 0 END) AS long_session,
       SUM(CASE WHEN end_session - start_session > INTERVAL '5 minute' THEN 1.0 ELSE 0.0 END)/COUNT(start_session) AS long_session_part
FROM skygame.game_sessions
GROUP BY DATE_TRUNC('month', start_session)
ORDER BY DATE_TRUNC('month', start_session) ASC;

-- => Помесячно: средняя длина сессии и доля сессий >1 часа (учитываем только сессии >5 мин)
SELECT DATE_TRUNC('month', start_session) AS period,
       AVG(end_session - start_session) AS avg_session_time,
       SUM(CASE WHEN end_session - start_session > INTERVAL '1 hour' THEN 1.0 ELSE 0.0 END)/COUNT(*) AS long_session_share
FROM skygame.game_sessions
WHERE end_session - start_session > INTERVAL '5 minute'
GROUP BY DATE_TRUNC('month', start_session)
ORDER BY DATE_TRUNC('month', start_session);


-- Изучим таблицу *referral*.

-- В ней хранится информация о приглашениях, которые наши пользователи рассылают своим друзьям, чтобы привлечь их в игру.

-- => Обзорный запрос к таблице приглашений
SELECT * FROM skygame.referral;

-- => Общее число приглашений, уникальных приглашавших и доля успешных регистраций
SELECT COUNT(ref_reg) AS cnt_refs,
       COUNT(DISTINCT id_user) AS cnt_user,
       SUM(ref_reg)/COUNT(ref_reg)*100 AS ref_success
FROM skygame.referral;

-- => Топ-50 пользователей по сумме успешных регистраций (пришедших друзей)
SELECT id_user
FROM skygame.referral
GROUP BY id_user
ORDER BY SUM(ref_reg) DESC
LIMIT 50;

-- => Пользователи с >5 приглашениями, где доля успешных >= 50%
SELECT id_user
FROM skygame.referral
GROUP BY id_user
HAVING COUNT(ref_reg) > 5
   AND SUM(ref_reg)/COUNT(ref_reg) >= 0.5;

-- => Пользователи с >6 приглашениями, но без единой регистрации
SELECT id_user
FROM skygame.referral
GROUP BY id_user
HAVING COUNT(ref_reg) > 6 AND SUM(ref_reg) = 0;


-- Отдел маркетинга проводил акцию, по которой более частый заход в игру позволял получить больше бесплатных кристаллов.
-- Акция длилась первые три недели марта 2023 года.
-- Видим ли мы позитивные результаты этой акции на графиках маркетинговых клиентских метрик?

-- => Ежедневная аудитория (DAU)
SELECT DATE_TRUNC('day', start_session) AS day_bin,
       COUNT(DISTINCT id_user) AS "DAU"
FROM skygame.game_sessions
GROUP BY day_bin;

-- => Еженедельная аудитория (WAU)
SELECT DATE_TRUNC('week', start_session) AS week_bin,
       COUNT(DISTINCT id_user) AS "WAU"
FROM skygame.game_sessions
GROUP BY week_bin;

-- => Ежемесячная аудитория (MAU)
SELECT DATE_TRUNC('month', start_session) AS month_bin,
       COUNT(DISTINCT id_user) AS "MAU"
FROM skygame.game_sessions
GROUP BY month_bin;


-- Мы хотим запустить еще одну акцию, в рамках которой игроки, проводящие наибольшее количество времени в игре, получат бонусные кристаллы и золотые монеты.
-- Мы хотим проверить текущие показатели по самым «вовлеченным» игрокам.

-- => Топ-25 по суммарному времени в игре (только сессии с окончанием, только пользователи 2022 года)
SELECT gs.id_user,
       SUM(gs.end_session - gs.start_session) AS time_session
FROM skygame.game_sessions AS gs
JOIN skygame.users AS u
    ON gs.id_user = u.id_user
WHERE end_session IS NOT NULL
  AND DATE_TRUNC('year', reg_date) = '2022-01-01'
GROUP BY gs.id_user
ORDER BY SUM(gs.end_session - gs.start_session) DESC
LIMIT 25;


-- Время позаниматься очисткой данных. Ищем проблемные записи.

-- => Общее число и доля сессий с пропущенным окончанием
SELECT SUM(CASE WHEN end_session IS NULL THEN 1 ELSE 0 END) AS total_count,
       SUM(CASE WHEN end_session IS NULL THEN 1.0 ELSE 0.0 END)/COUNT(*) AS total_share
FROM skygame.game_sessions;

-- => Доля проблемных сессий внутри каждой ОС (сколько % сессий на iOS битые, сколько % на Android)
SELECT SUM(CASE WHEN (gs.end_session IS NULL AND u.dev_type = 'ios') THEN 1.0 ELSE 0.0 END)/SUM(CASE WHEN u.dev_type = 'ios' THEN 1.0 ELSE 0.0 END) AS ios_share,
       SUM(CASE WHEN (gs.end_session IS NULL AND u.dev_type = 'android') THEN 1.0 ELSE 0.0 END)/SUM(CASE WHEN u.dev_type = 'android' THEN 1.0 ELSE 0.0 END) AS android_share
FROM skygame.game_sessions AS gs
JOIN skygame.users AS u
    ON gs.id_user = u.id_user;

-- => Распределение всех проблемных записей по ОС: какой % от всех битых сессий приходится на iOS, какой на Android
SELECT SUM(CASE WHEN (gs.end_session IS NULL) AND (u.dev_type = 'ios') THEN 1.0 ELSE 0.0 END)/SUM(CASE WHEN end_session IS NULL THEN 1 ELSE 0 END) AS ios_part,
       SUM(CASE WHEN (gs.end_session IS NULL) AND (u.dev_type = 'android') THEN 1.0 ELSE 0.0 END)/SUM(CASE WHEN end_session IS NULL THEN 1 ELSE 0 END) AS android_part
FROM skygame.game_sessions AS gs
JOIN skygame.users AS u
    ON gs.id_user = u.id_user;


-- Видим ли мы какие-то значимые изменения в структуре выручки с течением времени?

-- => Динамика выручки по месяцам и типам товаров (с джойном к истории цен)
SELECT DATE_TRUNC('month', mon.dtime_pay) AS month_buy,
       -- => Альтернативный вариант запроса: попытка сделать "широкую" таблицу с отдельными столбцами по типам
       -- SUM(CASE WHEN it.type = 'Ammo' THEN lp.price ELSE 0.0 END) AS "Ammo",
       -- SUM(CASE WHEN it.type = 'Currency' THEN lp.price ELSE 0.0 END) AS "Currency",
       -- SUM(CASE WHEN it.type = 'Materials' THEN lp.price ELSE 0.0 END) AS "Materials",
       -- SUM(CASE WHEN it.type = 'Transport' THEN lp.price ELSE 0.0 END) AS "Transport",
       -- SUM(CASE WHEN it.type = 'Weapon' THEN lp.price ELSE 0.0 END) AS "Weapon"
       it.type,
       SUM(mon.cnt_buy * lp.price)
FROM skygame.monetary AS mon
JOIN skygame.item_list AS it
    ON mon.id_item_buy = it.id_item
JOIN skygame.log_prices AS lp
    ON mon.id_item_buy = lp.id_item
    AND mon.dtime_pay >= lp.valid_from
    AND mon.dtime_pay < COALESCE(lp.valid_to, TO_DATE('01-01-2100','dd-mm-yyyy'))
GROUP BY month_buy, it.type
ORDER BY month_buy;


-- С 1 января 2023 года мы увеличили стоимость одного кристалла.
-- Повлияло ли изменение цены на этот показатель?
-- Повлияло ли изменение на суммарную выручку, которую мы получаем с кристаллов?

-- => Динамика для товара "Crystal": среднее число единиц в покупке и общая выручка по месяцам
SELECT DATE_TRUNC('month', mon.dtime_pay) AS month_buy,
       AVG(cnt_buy),
       SUM(mon.cnt_buy * lp.price)
FROM skygame.monetary AS mon
JOIN skygame.item_list AS it
    ON mon.id_item_buy = it.id_item
JOIN skygame.log_prices AS lp
    ON mon.id_item_buy = lp.id_item
    AND mon.dtime_pay >= lp.valid_from
    AND mon.dtime_pay < COALESCE(lp.valid_to, TO_DATE('01-01-2100','dd-mm-yyyy'))
WHERE it.name_item = 'Crystal'
GROUP BY month_buy, it.type
ORDER BY month_buy;


-- Надо ли нам наше маркетинговое воздействие распределять на все когорты поровну?
-- Возможно, какие-то когорты более «щедрые» на покупку игровых предметов.

-- => Средняя выручка на пользователя в месяц для каждой когорты (месяц регистрации) с нормировкой на "возраст"
SELECT avg_rev/EXTRACT('day' FROM ('04-28-2023' - cogort)/30) AS correct_avg,
       cogort
FROM (
    SELECT DATE_TRUNC('month', reg_date) AS cogort,
           COUNT(DISTINCT mon.id_user) AS cnt_users,
           SUM(cnt_buy * price) AS sum_rev,
           SUM(cnt_buy * price)/COUNT(DISTINCT mon.id_user) AS avg_rev
    FROM skygame.monetary AS mon
    JOIN skygame.users AS u
        ON u.id_user = mon.id_user
    JOIN skygame.log_prices AS lp
        ON mon.id_item_buy = lp.id_item
        AND mon.dtime_pay >= lp.valid_from
        AND mon.dtime_pay < COALESCE(lp.valid_to, TO_DATE('01-01-2100','dd-mm-yyyy'))
    WHERE u.reg_date < '04-01-2023'
    GROUP BY cogort
) AS t;


-- Факт. В ноябре и декабре 2022 года была опробована альтернативная стратегия привлечения клиентов.
-- Гипотеза:
-- в ноябре и декабре 2022 года из-за более дорогой и таргетированной рекламы мы приобрели более «лояльных» игроков, которые больше времени посвящают нашей игре.

-- => Простая помесячная динамика средней длины сессии
SELECT DATE_TRUNC('month', start_session) AS month,
       AVG(end_session - start_session) AS avg_sess
FROM skygame.game_sessions
WHERE end_session IS NOT NULL
GROUP BY month;

-- => Сравнение когорты ноябрь-декабрь 2022 с остальными пользователями (только сессии >5 минут)
WITH valid_sessions AS (
    SELECT user_c.id_user,
           user_c.cohort,
           gs.end_session - gs.start_session AS session_minutes
    FROM skygame.game_sessions AS gs
    JOIN (
        SELECT id_user,
               CASE WHEN reg_date BETWEEN '2022-11-01' AND '2022-12-31' THEN 'cohort_11_12'
                    ELSE 'other_cohorts' END AS cohort
        FROM skygame.users
    ) AS user_c
        ON gs.id_user = user_c.id_user
    WHERE (gs.end_session - gs.start_session) > INTERVAL '5 minutes'
)
SELECT cohort,
       COUNT(DISTINCT id_user) AS unique_users_count,
       COUNT(*) AS total_sessions,
       AVG(session_minutes) AS avg_session_minutes
FROM valid_sessions
GROUP BY cohort;


---- Рассчитаем показатели виральности

-- => Расчёт виральности (K-factor) и прогноз размера когорты с учётом виральности
WITH
invitations_stats AS (
    SELECT COUNT(DISTINCT r.id_user) AS sent_ref_users,
           COUNT(ref_reg) AS total_refs,
           COUNT(r.id_user)::FLOAT / (SELECT COUNT(DISTINCT id_user) FROM skygame.users) AS avg_refs
    FROM skygame.referral r
),
conversion_stats AS (
    SELECT SUM(ref_reg)/COUNT(ref_reg) AS regs_conv
    FROM skygame.referral r
),
avg_cohort_size AS (
    SELECT AVG(cs.cohort_size) AS historical_avg_cohort_size
    FROM (
        SELECT DATE_TRUNC('month', reg_date) AS cohort_month,
               COUNT(*) AS cohort_size
        FROM skygame.users
        WHERE DATE_PART('month', reg_date) <> 4
        GROUP BY cohort_month
    ) AS cs
)
SELECT i.avg_refs,
       c.regs_conv,
       (i.avg_refs * c.regs_conv) AS k_factor,
       a.historical_avg_cohort_size,
       a.historical_avg_cohort_size * (i.avg_refs * c.regs_conv) AS projected_cohort_size_with_virality
FROM invitations_stats AS i, conversion_stats AS c, avg_cohort_size AS a;


-- Допустим, лояльным считается тот пользователь, который пригласил как минимум троих друзей, из которых как минимум один в результате зарегистрировался в нашей игре.
-- Назовем данный критерий crit_invite.

-- => LMAU (лояльные уникальные пользователи в месяц) по критерию приглашений (>=3 отправлено, >=1 регистрация)
SELECT DATE_TRUNC('month', start_session) AS mm,
       COUNT(DISTINCT t.id_user) AS crit_invite
FROM (
    SELECT id_user
    FROM skygame.referral
    GROUP BY id_user
    HAVING COUNT(*) >= 3
       AND SUM(ref_reg) >= 1
) AS t
JOIN skygame.game_sessions AS gs
    ON t.id_user = gs.id_user
GROUP BY mm;


-- Допустим, лояльным считается тот пользователь, который заплатил суммарно больше 1000 рублей за всё время (не строго).
-- Назовем данный критерий crit_1000.

-- => LMAU по критерию выплат (суммарно >=1000 руб.)
WITH crit_1000 AS (
    SELECT id_user
    FROM skygame.monetary AS mon
    JOIN skygame.log_prices AS lg
        ON mon.id_item_buy = lg.id_item
        AND dtime_pay >= valid_from
        AND dtime_pay <= COALESCE(valid_to, '2100-01-01')
    GROUP BY id_user
    HAVING SUM(mon.cnt_buy * lg.price) >= 1000
)
SELECT DATE_TRUNC('month', start_session) AS mm,
       COUNT(DISTINCT gs.id_user) AS crit_invite
FROM skygame.game_sessions AS gs
JOIN crit_1000 AS cr
    ON cr.id_user = gs.id_user
GROUP BY mm;


-- Рассчитаем динамику LMAU для критерия лояльности crit_invite AND crit_1000.

-- => LMAU для лояльных, удовлетворяющих ОБОИМ условиям (AND)
WITH crit_invite AS (
    SELECT id_user
    FROM skygame.referral
    GROUP BY id_user
    HAVING COUNT(*) >= 3 AND SUM(ref_reg) >= 1
),
crit_1000 AS (
    SELECT id_user
    FROM skygame.monetary AS mon
    JOIN skygame.log_prices AS lg
        ON mon.id_item_buy = lg.id_item
        AND dtime_pay >= valid_from
        AND dtime_pay <= COALESCE(valid_to, '2100-01-01')
    GROUP BY id_user
    HAVING SUM(mon.cnt_buy * lg.price) >= 1000
)
SELECT DATE_TRUNC('month', start_session) AS mm,
       COUNT(DISTINCT gs.id_user) AS crit_invite_1000
FROM skygame.game_sessions AS gs
JOIN crit_invite ON crit_invite.id_user = gs.id_user
JOIN crit_1000  ON crit_1000.id_user  = gs.id_user
GROUP BY mm;


-- Рассчитаем динамику LMAU для критерия лояльности crit_invite OR crit_1000.

-- => LMAU для лояльных, удовлетворяющих ХОТЯ БЫ ОДНОМУ условию (OR)
WITH crit_invite AS (
    SELECT id_user
    FROM skygame.referral
    GROUP BY id_user
    HAVING COUNT(*) >= 3 AND SUM(ref_reg) >= 1
),
crit_1000 AS (
    SELECT id_user
    FROM skygame.monetary AS mon
    JOIN skygame.log_prices AS lg
        ON mon.id_item_buy = lg.id_item
        AND dtime_pay >= valid_from
        AND dtime_pay <= COALESCE(valid_to, '2100-01-01')
    GROUP BY id_user
    HAVING SUM(mon.cnt_buy * lg.price) >= 1000
)
SELECT DATE_TRUNC('month', start_session) AS mm,
       COUNT(DISTINCT gs.id_user) AS crit_invite_1000
FROM skygame.game_sessions AS gs
WHERE id_user IN (SELECT * FROM crit_invite)
   OR id_user IN (SELECT * FROM crit_1000)
GROUP BY mm;


-- Допустим, лояльным считается пользователь, который входит в топ-100 пользователей по средним выплатам за один месяц своей жизни. Тогда:

-- => Топ-100 пользователей по среднемесячным выплатам за всё время жизни
SELECT u.id_user,
       SUM(cnt_buy * price) AS revenue,
       reg_date,
       EXTRACT('day' FROM (SELECT MAX(dtime_pay) FROM skygame.monetary) - reg_date)/30 AS life_time,
       SUM(cnt_buy * price)/(EXTRACT('day' FROM (SELECT MAX(dtime_pay) FROM skygame.monetary) - reg_date)/30) AS avg_rev
FROM skygame.monetary AS mon
JOIN skygame.log_prices AS lg      
    ON mon.id_item_buy = lg.id_item
    AND dtime_pay >= valid_from
    AND dtime_pay <= COALESCE(valid_to, '2100-01-01')
JOIN skygame.users AS u
    ON u.id_user = mon.id_user
GROUP BY u.id_user, reg_date
ORDER BY avg_rev DESC
LIMIT 100;
