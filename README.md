## 1. Выведите 10 самых популярных брендов и моделей, имеющие больше всего просмотров.

```
SELECT 
  model_brand, COUNT (*) cnt_watch
FROM (
  SELECT 
      CASE
        WHEN ym_pv_URL LIKE '%/new/%' THEN REGEXP_EXTRACT(ym_pv_URL, r'\w*/new/(\w*/\w*)/n\d+') 
        ELSE REGEXP_EXTRACT(ym_pv_URL, r'\w*/used/(\w*/\w*)/u\d+') 
      END model_brand
    FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
    WHERE 
      (REGEXP_CONTAINS(ym_pv_URL,r'\w*/new/\w*/\w*/n\d+') OR REGEXP_CONTAINS(ym_pv_URL,r'\w*/used/\w*/\w*/u\d+')) 
      AND ym_pv_URL NOT LIKE '%?%'   
  ) model_brand_not_agg
GROUP BY model_brand
ORDER BY cnt_watch DESC
LIMIT 10;

-- если хотим посмотреть самые популярные марки и модели в разрезе новые/с пробегом

WITH model_brand_cnt AS (
  SELECT model_brand, new_used, COUNT(*) cnt_watch
  FROM (
    SELECT 
      CASE
        WHEN ym_pv_URL LIKE '%/new/%' THEN REGEXP_EXTRACT(ym_pv_URL, r'\w*/new/(\w*/\w*)/n\d+') 
        ELSE REGEXP_EXTRACT(ym_pv_URL, r'\w*/used/(\w*/\w*)/u\d+') 
      END model_brand, 
      CASE
        WHEN REGEXP_CONTAINS(ym_pv_URL,r'\w*/new/\w*/\w*/n\d+') THEN 'new'
        ELSE 'used'
      END new_used
    FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
    WHERE 
      (REGEXP_CONTAINS(ym_pv_URL,r'\w*/new/\w*/\w*/n\d+') OR REGEXP_CONTAINS(ym_pv_URL,r'\w*/used/\w*/\w*/u\d+')) 
      AND ym_pv_URL NOT LIKE '%?%'   
  ) model_brand_not_agg
  GROUP BY model_brand, new_used
)
SELECT model_brand, new_used, cnt_watch
FROM (
  SELECT model_brand, new_used, cnt_watch,
       ROW_NUMBER() OVER (PARTITION BY new_used ORDER BY cnt_watch DESC) rnk
  FROM model_brand_cnt
) car_with_rnk
WHERE rnk <= 10
```
*_Возможно, более справедливо было бы использовать вместо ROW_NUMBER функцию DENSE_RANK, чтобы выводить топ с учетом тех, кто имеет равное количество просмотров, но не попадает в список, потому что находится ниже по алфавиту._

![image](https://github.com/user-attachments/assets/2da1f0e4-4485-40b2-b262-22a0b71fe563)

**Выводы:**
- на основе графика можно заметить, что автомобили с пробегом пользуются большей популярностью, нежели новые;
- при этом абсолютным лидером по количеству просмотров является автомобиль с пробегом chevrolet/niva; 
- также стоит внимания то, что только несколько авто занимают лидирующие позиции как в сегменте машин с пробегом, так и новых (lada/granta, skoda/rapid).

## 2. Посчитайте среднюю длину сессии.
```
SELECT ROUND(AVG(session_time) / 60, 2) avg_session_time_min
FROM
(
  SELECT 
    ym_pv_clientID,
    TIMESTAMP_DIFF(MAX(ym_pv_datetime), MIN(ym_pv_datetime), SECOND) session_time
  FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
  GROUP BY ym_pv_clientID
  ORDER BY session_time DESC   
) t1; 

-- введем условие, если никаких переходов на другие страницы не происходило больше, чем за 30 мин., то данный промежуток времени будем считать неактивным

WITH durations AS (
  SELECT 
    ym_pv_clientID,
    COALESCE(
      TIMESTAMP_DIFF(
        LEAD(ym_pv_dateTime) OVER (PARTITION BY CAST(ym_pv_clientID AS STRING) ORDER BY ym_pv_dateTime), 
        ym_pv_datetime, 
        SECOND
      )
    , 0) duration
  FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test` 
  ORDER BY ym_pv_clientID      
)

SELECT ROUND(AVG(total_duration) / 60, 2) avg_session_time_min, ROUND(AVG(total_duration_with_limit) / 60, 2) avg_session_time_min_with_limit
FROM
(
  SELECT ym_pv_clientID, SUM(duration) total_duration, 
         (SELECT SUM(duration) 
         FROM durations d2
         WHERE duration <= 1800 
         AND d1.ym_pv_clientID = d2.ym_pv_clientID
         GROUP BY ym_pv_clientID) total_duration_with_limit
  FROM durations d1
  GROUP BY ym_pv_clientID     
) t1;
```

![image](https://github.com/user-attachments/assets/7328064e-a6df-4b19-a586-07b0f29fa646)

**Выводы:**
- если рассчитывать среднее время сессии на основе всех данных, то получаем почти 38 мин.;
- при этом стоит учитывать те случаи, когда пользователь открывает страницу и бездействует. поэтому я рассчитала второй вариант, в котором поставила следующее условие: если пользователь находится на странице дольше 30 мин., то это время не берем в учет. в связи с этим среднее время по второму варианту сильно меньше первого (2,5 мин.).

## 3. Сколько страниц автомобилей (страница конкретного автомобиля имеет следующий вид и обычно называется VDP – vehicle details page) было просмотрено с мобильных устройств и с компьютера.

```
SELECT 
  CASE
    WHEN ym_pv_deviceCategory = 1 THEN 'computer'
    ELSE 'mobile'
  END ym_pv_deviceCategory, 
  COUNT(*) cnt_vdp
FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
WHERE  
  (REGEXP_CONTAINS(ym_pv_URL,r'\w*/new/\w*/\w*/n\d+') OR REGEXP_CONTAINS(ym_pv_URL,r'\w*/used/\w*/\w*/u\d+')) 
  AND ym_pv_URL NOT LIKE '%?%'
GROUP BY ym_pv_deviceCategory;

-- если хотим дополнительно посмотреть, с каких браузеров смотрят страницы с мобильных устройств и с компьютера
SELECT 
  CASE
    WHEN ym_pv_deviceCategory = 1 THEN 'computer'
    ELSE 'mobile'
  END ym_pv_deviceCategory, 
  ym_pv_browser,
  COUNT(*) cnt_vdp
FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
WHERE  
  (REGEXP_CONTAINS(ym_pv_URL,r'\w*/new/\w*/\w*/n\d+') OR REGEXP_CONTAINS(ym_pv_URL,r'\w*/used/\w*/\w*/u\d+')) 
  AND ym_pv_URL NOT LIKE '%?%'
GROUP BY ym_pv_deviceCategory, ym_pv_browser
ORDER BY ym_pv_deviceCategory, cnt_vdp DESC
```

![image](https://github.com/user-attachments/assets/9549b709-af09-4787-a8d7-64978fa21597)

**Выводы:**
- можем заметить, что приблизительно 3/4 просмотров приходятся на мобильные устройства;
- также стоит отметить, что в разрезе просмотров с компьютеров различные виды браузеров находятся в примерно равном соотношении - chrome имеет незначительное преимущество относительно yandex_browser, от которого недалеко расположен firefox. только opera в группе отстающих;
- в разрезе просмотров с мобильных устройств ситуация иная и chromemobile имеет весомое преимущество над остальными, за которым следует yandex_browser. можно сделать вывод, что среди мобильных устройств встроенные поисковые системы менее популярны, чем привычные chrome и yandex.

## 4. С каких источников трафика пришли клиенты.

```
SELECT ym_pv_lastTrafficSource, COUNT(*) number_of_visits
FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
GROUP BY ym_pv_lastTrafficSource
ORDER BY number_of_visits DESC;

-- если хотим узнать, из какого источника клиенты приходят на сайт в первый раз чаще всего 

SELECT ym_pv_lastTrafficSource, COUNT(*) cnt_clients
FROM
(
  SELECT ym_pv_clientID, MIN(ym_pv_dateTime) first_dt
  FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test` 
  GROUP BY ym_pv_clientID
) t1
JOIN 
(
  SELECT ym_pv_clientID, ym_pv_lastTrafficSource, ym_pv_dateTime
  FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test` 
) t2
ON t1.ym_pv_clientID = t2.ym_pv_clientID AND t1.first_dt = t2.ym_pv_dateTime
GROUP BY ym_pv_lastTrafficSource
ORDER BY cnt_clients DESC
```

![image](https://github.com/user-attachments/assets/badbc651-950a-463a-a7cf-c54fae53d577)

**Выводы:**
- можно заметить, что чаще всего пользователи переходят на сайт по прямым ссылкам;
- поскольку на сайте, например, происходит поиск подходящей модели авто по различным фильтрам и изучение страниц конкретных авто, источник internal также очень распространен;
- также одним из наиболее распространенных является органический трафик, на основе чего можно предположить, что сайт содержит качественный контент и является удобным для пользователей, благодаря чему поисковик не опускает сайт в результатах выдачи;
- при этом через остальные источники трафика клиенты почти не заходят. на основе этого можно сделать вывод, что компания не особо вкладывается в свое продвижение через разного рода рекламные форматы.

## 5. Одна из основных метрик на сайте – VDP views. В нашем случае это отношение просмотров страниц автомобиля ко всем просмотрам. Посчитайте это отношение для уникальных пользователей в разрезе мобильных устройств и компьютера, а также добавьте разрез по источникам трафика

```
-- также в столбце rate_visit_device вывела распределение долей внутри групп в разрезе мобильных устройств и компьютера
SELECT 
  ym_pv_deviceCategory, 
  ym_pv_lastTrafficSource, 
  ROUND(COUNT(*) / SUM(COUNT(*)) OVER() * 100, 2) rate_visit,
  ROUND(COUNT(*) / SUM(COUNT(*)) OVER(PARTITION BY ym_pv_deviceCategory) * 100, 2) rate_visit_device
FROM (
  SELECT 
    DISTINCT ym_pv_clientID, 
    ym_pv_URL, 
    CASE
      WHEN ym_pv_deviceCategory = 1 THEN 'computer'
      ELSE 'mobile'
    END ym_pv_deviceCategory, 
    ym_pv_lastTrafficSource
  FROM `artsofte-test-433808.Artsofte_Test.Artsofte_Test`
  WHERE  
    (REGEXP_CONTAINS(ym_pv_URL,r'\w*/new/\w*/\w*/n\d+') OR REGEXP_CONTAINS(ym_pv_URL,r'\w*/used/\w*/\w*/u\d+')) 
    AND ym_pv_URL NOT LIKE '%?%'
) t1
GROUP BY ym_pv_deviceCategory, ym_pv_lastTrafficSource
ORDER BY ym_pv_deviceCategory, rate_visit DESC;
```

![image](https://github.com/user-attachments/assets/c4011f31-3de5-4a87-a54c-db80f08729af)

**Выводы:**
- чаще всего пользователи просматривают страницы авто с мобильных устройств (соотношение с компьютером приблизительно 2/1), при этом переходя в основном по прямой ссылке;
- в разрезе просмотров с компьютера ситуация немного отличается и здесь чаще просматривают страницу авто через внутренние переходы по сайту;
- стоит отметить, что доля органического трафика для мобильных устройств значительно выше, чем для компьютеров. следовательно, через общий поиск до конкретной страницы авто с мобильных устройств заходят намного чаще, чем через компьютер;
- нераспространенными источниками трафика являются recommend для мобильных устройств и referral для компьютеров, что отчасти подтверждается графиком из 4 задания, по которому видно, что такие типы трафика не являются основными.

## 6. Опишите своими словами, за что отвечает столбец ym_pv_notBounce.
возможно, данный столбец отвечает за то, дошел ли клиент до какого-либо целевого действия (например, покупки авто) или отвалился. если 0, то отвалился, если 1 - то нет.

## Какие выводы у вас сформировались на основе тех данных, которые вы получили? Опишите 2 гипотезы и каких данных вам не хватает для их проверки.
1. моя первая гипотеза связана с тем, что средняя длина сессии дольше 2,5 мин., но меньше 38 мин.:
    - введенное ограничение на то, чтоб не учитывать те интервалы времени, которые дольше 30 мин., весьма условно, но однозначно необходимо не брать в расчет те сессии, которые сильно дольше;
    - поэтому для начала необходимо выяснить - компания считает сессию неактивной, если она продлилась дольше скольки мин.?
    - в своих расчетах я убирала такие сеансы полностью, но если бы, к примеру, у нас были данные последнего клика на странице, то мы бы брали в учет такие сеансы, но уже не полностью, а до момента последнего клика + возможно, какой-то установленный интервал. поэтому среднее время сессии стало бы дольше.

2. низкое количество просмотров с таких источников трафика, как ad, social, refferal, recommend, связано с низким уровнем вложений в рекламные форматы:
    - для этого необходимо изучить маркетинговые расходы, их структуру;
    - здесь же можно предположить, что стратегия рекламного продвижения реализована некачественным образом;
    - для того, чтобы это проверить, будут необходимы данные для расчета конверсии, например, данные о показах, кликах и т.д.
