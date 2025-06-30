# -
изучить влияние характеристик игроков и их игровых персонажей 

/* Проект «Секреты Тёмнолесья»
 * Цель проекта: изучить влияние характеристик игроков и их игровых персонажей 
 * на покупку внутриигровой валюты «райские лепестки», а также оценить 
 * активность игроков при совершении внутриигровых покупок
 * 
 * Автор: Маницын Дмитрий
 * Дата: 26.06.25
*/

-- Часть 1. Исследовательский анализ данных
-- Задача 1. Исследование доли платящих игроков

-- 1.1. Доля платящих пользователей по всем данным:
-- Напишите ваш запрос здесь
SELECT
    COUNT(*) AS общее_количество_игроков,
    COUNT(CASE WHEN payer = 1 THEN 1 END) AS количество_платящих_игроков,
    ROUND(COUNT(CASE WHEN payer = 1 THEN 1 END) * 100.0 / COUNT(*), 2) AS доля_платящих_игроков_от_общего_количества
FROM fantasy.users;

-- 1.2. Доля платящих пользователей в разрезе расы персонажа:
-- Напишите ваш запрос здесь
SELECT
    r.race AS раса_персонажа,
    COUNT(DISTINCT CASE WHEN u.payer = 1 THEN u.id END) AS платящие_игроки,
    COUNT(DISTINCT u.id) AS всего_игроков,
    ROUND(COUNT(DISTINCT CASE WHEN u.payer = 1 THEN u.id END) * 100.0 / NULLIF(COUNT(DISTINCT u.id), 0), 2) AS доля_платящих_процентов
FROM fantasy.race r
LEFT JOIN fantasy.users u ON r.race_id = u.race_id
GROUP BY r.race;

-- Задача 2. Исследование внутриигровых покупок
-- 2.1. Статистические показатели по полю amount:
-- Напишите ваш запрос здесь
SELECT
    COUNT(*) AS общее_число_покупок,
    SUM(amount) AS суммарная_стоимость,
    MIN(amount) AS минимальная_стоимость,
    MAX(amount) AS максимальная_стоимость,
    AVG(amount) AS среднее_значение,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS медиана,
    STDDEV(amount) AS стандартное_отклонение
FROM fantasy.events
WHERE amount > 0;

-- 2.2: Аномальные нулевые покупки:
-- Напишите ваш запрос здесь
SELECT
    COUNT(*) AS нулевые_покупки,
    COUNT(*) * 100.0 / NULLIF((SELECT COUNT(*) FROM fantasy.events), 0) AS доля_процентов
FROM fantasy.events
WHERE amount = 0;

-- 2.3: Популярные эпические предметы:
-- Напишите ваш запрос здесь
WITH filtered_events AS (
    SELECT e.*
    FROM fantasy.events e
    WHERE e.amount > 0 -- исключаем нулевые покупки
),
total_sales AS (
    SELECT COUNT(*) AS total_count
    FROM filtered_events
),
item_stats AS (
    SELECT
        i.item_code,
        i.game_items,
        COUNT(*) AS продажи_абсолютно,
        COUNT(*) * 1.0 / (SELECT total_count FROM total_sales) AS доля_от_всех_продаж,
        -- Количество уникальных игроков, купивших этот предмет:
        COUNT(DISTINCT e.id) * 1.0 / (SELECT COUNT(DISTINCT id) FROM filtered_events) AS доля_игроков
    FROM filtered_events e
    JOIN fantasy.items i ON e.item_code = i.item_code
    GROUP BY i.item_code, i.game_items
)
SELECT 
    item_code,
    game_items,
    продажи_абсолютно,
    ROUND(доля_от_всех_продаж * 100, 2) AS доля_процентов,
    ROUND(доля_игроков * 100, 2) AS доля_игроков
FROM item_stats
ORDER BY продажи_абсолютно DESC;

-- Часть 2. Решение ad hoc-задачbи
-- Задача: Зависимость активности игроков от расы персонажа:
-- Напишите ваш запрос здесь:

WITH player_purchases AS (
    SELECT
        u.id AS user_id,
        u.race_id,
        u.payer,
        COUNT(e.transaction_id) FILTER (WHERE e.amount > 0) AS total_purchases,
        SUM(e.amount) FILTER (WHERE e.amount > 0) AS total_amount
    FROM fantasy.users u
    LEFT JOIN fantasy.events e ON u.id = e.id AND e.amount > 0
    GROUP BY u.id, u.race_id, u.payer
),
race_stats AS (
    SELECT
        r.race_id,
        r.race,
        COUNT(DISTINCT p.user_id) AS total_players,
        COUNT(DISTINCT CASE WHEN p.payer = 1 THEN p.user_id END) AS paying_players
    FROM fantasy.race r
    LEFT JOIN player_purchases p ON r.race_id = p.race_id
    GROUP BY r.race_id, r.race
)
SELECT
    rs.race,
    rs.total_players AS общее_игроков,
    rs.paying_players AS платящих_игроков,
    CASE WHEN rs.total_players > 0 THEN ROUND((rs.paying_players * 100.0 / rs.total_players)::numeric, 2) ELSE 0 END AS доля_платящих,
    
    -- Среднее количество покупок на платящего игрока:
    COALESCE(ROUND(AVG(p.total_purchases)::numeric, 2), 0) AS среднее_количество_покупок,
    
    -- Средняя стоимость одной покупки на платящего игрока:
    CASE WHEN SUM(p.total_purchases) > 0 THEN ROUND((SUM(p.total_amount) / NULLIF(SUM(p.total_purchases), 0))::numeric, 2) ELSE 0 END AS средняя_стоимость_одной_покупки,
    
    -- Средняя суммарная стоимость всех покупок на платящего игрока:
    CASE WHEN COUNT(DISTINCT p.user_id) > 0 THEN ROUND((SUM(p.total_amount) / NULLIF(COUNT(DISTINCT p.user_id), 0))::numeric, 2) ELSE 0 END AS средняя_суммарная_стоимость
FROM race_stats rs
LEFT JOIN player_purchases p ON rs.race_id = p.race_id AND p.payer = 1
GROUP BY rs.race, rs.total_players, rs.paying_players;



WITH stats AS (
  SELECT 
    AVG(amount) AS avg_amount,
    STDDEV(amount) AS std_amount
  FROM fantasy.events 
  WHERE amount > 0
)
SELECT
  COUNT(*) AS abnormal_purchases_count,
  MAX(amount) AS max_abnormal_amount
FROM fantasy.events, stats
WHERE amount > (avg_amount + 3 * std_amount);

-- проверка аномалий 

SELECT COUNT(*) 
FROM fantasy.events 
WHERE amount > 8080.60

SELECT 
    COUNT(*) AS тотал_аномалий,
    AVG(amount) AS среднее_аномалий,
    MIN(amount) AS мин_аномалий,
    MAX(amount) AS макс_аномалий
FROM fantasy.events
WHERE amount > 8080.60;



