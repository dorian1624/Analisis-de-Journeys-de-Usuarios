# Embudo de conversión
## Parte 1
## Objetivo: agrupar las conversiones del embudo por país (country) y detectar en que etapa del funnel se pierde más a los usuarios.

### Se creo una CTE por evento con el rango de fechas deseado

WITH first_visit AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name ='first_visit' AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
select_item AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name IN ('select_item', 'select_promotion') AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_to_cart AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name ='add_to_cart' AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
begin_checkout AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name ='begin_checkout' AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_shipping_info AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name ='add_shipping_info' AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_payment_info AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name ='add_payment_info' AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
purchase AS(
    SELECT DISTINCT user_id
    FROM mercadolibre_funnel
    WHERE event_name ='purchase' AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
)
### Se unieron las CTEs anclando en signup y cuenta usuarios por etapa y se agruparon mediante la variable country

funnel_counts AS(
SELECT
    fv.country,
    COUNT(fv.user_id)  AS usuarios_first_visit,
    COUNT(si.user_id)  AS usuarios_select_item,
    COUNT(a.user_id)   AS usuarios_add_to_cart,
    COUNT(bc.user_id)  AS usuarios_begin_checkout,
    COUNT(asi.user_id) AS usuarios_add_shipping_info,
    COUNT(api.user_id) AS usuarios_add_payment_info,
    COUNT(p.user_id)   AS usuarios_purchase
FROM first_visits fv
LEFT JOIN select_item si        ON fv.user_id = si.user_id  AND fv.country = si.country
LEFT JOIN add_to_cart a         ON fv.user_id = a.user_id   AND fv.country = a.country
LEFT JOIN begin_checkout bc     ON fv.user_id = bc.user_id  AND fv.country = bc.country
LEFT JOIN add_shipping_info asi ON fv.user_id = asi.user_id AND fv.country = asi.country
LEFT JOIN add_payment_info api  ON fv.user_id = api.user_id AND fv.country = api.country
LEFT JOIN purchase p            ON fv.user_id = p.user_id   AND fv.country = p.country
GROUP BY fv.country
)

###  A partir de los conteos por etapa del embudo, se calculo el porcentaje de conversión desde la etapa inicial (first_visit) hacia cada etapa.

SELECT 
    country,
    usuarios_select_item       * 100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_select_item,
    usuarios_add_to_cart       * 100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_add_to_cart,
    usuarios_begin_checkout    * 100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_begin_checkout,
    usuarios_add_shipping_info * 100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_add_shipping_info,
    usuarios_add_payment_info  * 100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_add_payment_info,
    usuarios_purchase          * 100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_purchase
FROM 
    funnel_counts

## Parte 2
## Objetivo: Para cada país, contar usuarios activos acumulados desde su registro, en el rango 2025-01-01 → 2025-08-31, al día 7, día 14, día 21 y día 28.

### Objetivo: SOLO CONTEOS acumulados por país (sin porcentajes)

SELECT
    country,
    COUNT(DISTINCT CASE WHEN day_after_signup >= 7  AND active = 1 THEN user_id END) AS users_d7,
    COUNT(DISTINCT CASE WHEN day_after_signup >= 14 AND active = 1 THEN user_id END) AS users_d14,
    COUNT(DISTINCT CASE WHEN day_after_signup >= 21 AND active = 1 THEN user_id END) AS users_d21,
    COUNT(DISTINCT CASE WHEN day_after_signup >= 28 AND active = 1 THEN user_id END) AS users_d28
FROM 
    mercadolibre_retention
WHERE 
    activity_date BETWEEN '2025-01-01' AND '2025-08-31'
GROUP BY 
    country
ORDER BY 
    country
    
### Objetivo: convertir los conteos del Task 1 en porcentajes por país

SELECT
    country,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 7  AND active = 1 THEN user_id END)  * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 1) AS retention_d7_pct,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 14 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 1) AS retention_d14_pct,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 21 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 1) AS retention_d21_pct,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 28 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 1) AS retention_d28_pct
FROM 
    mercadolibre_retention
WHERE 
    activity_date BETWEEN '2025-01-01' AND '2025-08-31'
GROUP BY 
    country
ORDER BY 
    country;

### Se asigna a cada usuario su cohorte de registro en formato YYYY-MM

SELECT 
    user_id,
    MIN(signup_date) AS signup_date,
    TO_CHAR(DATE_TRUNC('month', MIN(signup_date)), 'YYYY-MM') AS cohort
FROM 
    mercadolibre_retention 
    
GROUP BY 
    user_id
LIMIT 5

### CTE para cohorte

WITH cohort AS (
    SELECT
        user_id,    
        TO_CHAR(DATE_TRUNC('month', MIN(signup_date)), 'YYYY-MM') AS cohort
    FROM 
        mercadolibre_retention
    GROUP BY 
        user_id
),
activity AS (
### CTE activity: tomar columnas claves de mercadolibre_retention y añadir el cohort 

SELECT 
        r.user_id,
        c.cohort,
        r.day_after_signup,
        r.active
FROM 
        mercadolibre_retention r
LEFT JOIN 
        cohort c ON c.user_id = r.user_id
WHERE 
        r.activity_date BETWEEN '2025-01-01' AND '2025-08-31'
)
### Select final con los conteos exactos por día acumulado X / tamaño de cohorte -> % redondeado

SELECT 
    cohort,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 7 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id),0),1) AS D7,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 14 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id),0),1) AS D14,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 21 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id),0),1) AS D21,
    ROUND(COUNT(DISTINCT CASE WHEN day_after_signup >= 28 AND active = 1 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id),0),1) AS D28
FROM 
    activity
GROUP BY 
    cohort
ORDER BY 
    cohort;

# Hallazgos y oportunidades de mejora

<img width="504" height="435" alt="image" src="https://github.com/user-attachments/assets/b596538f-5cf8-4189-b53d-effa22f7da7f" />

ORDER BY 
    conversion_purchase DESC
