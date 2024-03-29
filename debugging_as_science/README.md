# Отладка как наука

# Пример 1

   Проект с курса Яндекса. Система записывает значение метрик на сервер. Метрики сохраняются в мапе, которые позже сериализуются (в моем случае, записывались в файл в виде JSON). 
Если сервер перезапускался с флагом RESTORE, то все данные из файла десериализовались на сервер. Теперь к проблеме.
   После создания новой фичи, одни из автотестов начали падать, говоря, что после записи метрики, при попытке получить ее значение, метрика не найдена (ожидалось 200, а получал 404). Назовем этот случай Т1.
Как я пытался отлаживал данный баг: была гипотеза, что при записи метрики, алгоритм как-то неверно записывал метрику (например, использовал в качестве ключа не имя метрики, а иное поле) из-за чего метрику нельзя было найти.
Данная гипотеза была опровергнута тестами, которые успешно "проходили данных сценарий". Проблема была в том, что сервер возвращал 404 только одном случае при одном сценарии, но этот сценарий успешно обрабатывали тесты, из-за чего я 
начал "гадать" где-же кроется ошибка. Стоит сказать, что автотесты со случаем Т1 проходили до определенного момента, т.е. Т1 мог успешно пройти часть автотестов, а после автотеста А1 падал.
   Попробуем решить данную проблему, задавая не конкретные вопросы (почему вернул именно 404), а широкие. Например, можно задать вопрос не "почему сервер возвращает именно 404", а "автотесты падают в момент, когда была добавлена поддержка флагов (RESTORE)".
Мы откинем все связанное не с флагами, поскольку: 
1) более ранние автотесты подтверджают корректную работу системы
2) добавление флагов -- точка, где автотесты ломаются, и, что важно, при добавлении флагов, логика сервера записи/чтения не менялась.
Теперь, вместо многих конкретных причин, связанных с записью и чтением, нужно проверить, как работает система с включенным и выключенным флагом. В случае, если флаг был выключен, то автотесты проходили, иначе -- падали. 
То есть, при включенном флаге RESTORE данные не появлялись на сервере.

Ошибка заключалась в следующем -- автотесты с включенным флагом "роняли" сервер и проверяли, что ранее записанные данные десериализуются на сервер, и, поскольку я неверно написал логику десериализации, данные не десериализовались на сервер, из-за чего, 
при попытке чтения данных, возвращалась 404. До этого, я потратил несколько часов на поиск ошибки в логие чтения/записи и несколько часов на "гадания", но "откидавая ненужные варианты" я буквально за три вопроса пришел бы к источнику проблемы.
То есть вопросы:
1) С какого момента появилась ошибка?
2) Какие изменения внес момент в систему, когда появилась ошибка?
3) В каком конкретном состоянии находится программа (на 1-ом уровне), при котором ошибка возникает?

привели к источнику пробенмы.

# Пример 2

   Проект с прошлой работы. Сервис рассчитывал финансовые показатели на основе ежедневно обновляемых исходных данных и рассчитанные показатели отражались в tableau. Однажды показатели конкретного сервиса выдавали неверное значение (было сильное расхождение с подтвержденными данными) в некоторые дни месяца.
В остальные дни данные были корректны. Требовалось определить, была ли это ошибка программы или это было просто поведение метода расчета. Забегая вперед, могу сказать, ошибка была выявлена методом "перебора всевозможных вариантов" и ошибка заключалась в коде и 
постановке: иногда приходили данные, которые выходили за инварианты, указанные в постановке (например, один из коэффициентов, который должен был быть в диапазоне от 1 до 10, мог быть 0, отрицательным или многобольше 10), в коде, в случае если приходили невалидные данные, 
подобные коэффициенты и показатели имели некоторый дефолтное значение (вот только проблема, никак не отражалось в БД, что было ли рассчет на основе дефолтных значений, что означает, что исходные данные были невалидны). Ситуацию осложняло, что некоторые данные обновлялись  в БД, а не записывались новые, из-за чего отследить было крайне сложно ошибку. Кстати, следовало бы больше логгировать, поскольку одни и те же данные могли отличаться, подобная избыточность информации очень помогла. Кстати, следовало бы больше логгировать, поскольку одни и те же данные могли отличаться, подобная избыточность информации очень помогла. 
   В общем, как можно было найти проблему в данном случае. Будем говорить об ошибке в тот день, когда итоговые показатели сильно расходились. 
1) На основе каких исходных данных были сделаны рассчеты? В случае, если исходные данные валидны, то ошибка определенно была бы в коде. Изучив все исходные данные для конкретного сервиса, была обнаружена несостыковка между постановкой
и реальными данными. 
2) Как программа работает с невалидными исходными данными? Поскольку я создал АТД, которые принимали дефолтные значению в случае несоответствия пред-условиям, нужно было определить, насколько дефолтные значения влияют на процент расхождения.
По мере бурных обсуждений, проб и проверок, были установлены иные значения, но это еще не все
3) Действительно ли новопринятые изменения не нарушают логику программы, не противоречат первым двум вопросом? Данный вопрос решался методом наблюдения -- просто изучали, как поведет себя метод.
   Через призму времени кажется, что проблема решалась легко. Но я провел несколько дней, ища баг не там где нужно, поскольку не рассмотрел вариант "на основе каких данных делались рассчеты?".



# Пример 3

   Сервис с прошлой работы по парсингу сводной таблицы партнеров, договоров, тарифов и т.п. Суть состояла в том, чтобы сделать из csv файла сиды для миграций. Сделал парсер следующим образом -- реализовал реляционную модель, которая включала
отношения, кортежи и ограничения. При чтении файла создавались отношения, отражающие отношения для миграций, и, при чтении строки, отношение в парсере заполнялось в соответствующим кортежом из csv. В случае, если отношение содержало FK, и кортежа с данным PK не находилось, то кортеж из csv не заносился в файл. Теперь к ошибке.
   По мере обновления файла, в один момент парсер перестал корректно связывать некоторые данные (например, тип связи многие-ко-многим и подобные) . Опять же, с помощью дебаггера (который после отключил и 1.5 года уже программирую без него)
проблема была пофикшена. Источником бед оказался логика представления отношений -- у меня это было множество, при котором нарушались связи (а вообще нарушилось просто порядок id, но это было важно на тот момент ), и вот а при мультимножестве 
все работало, как хотели видеть. 
   Как проблему можно было обнаружить быстрее. На вопрос "корректны ли исходные данные" ответ получить сложно, поскольку файл содержал несколько тысяч строк, а ответ от поставщика файла ждать долго. Примем, что исходные данные корректны.
Можно задать вопрос "в чем разница между парсинга файла версии 1 и парсинга файла версии 2, в котором все ломалось". Таким образом, можно выявить, что основная разница в связях между отношениями. Для дальнейшей отладки, нужно принять во 
внимание, что нарушение связи между отношением может быть вследствии неверных данных в связуемых отношениях, посему напрашивается вопрос "в чем разница отношения А, которое получилось в результате парсинга файла версии1 от отношения 
А, которое получилось в результате парсинга файла версии 2". Если разница есть, то источник проблемы фактически решен, но в нашем случае разницы не оказалось, и, следовательно, если ошибка не в связуемых отношениях, то в отношении-связи.
Изучив вопрос "в чем разница между отношением-связи Б, полученное в результате парсинга файла версии 1, и между отношением-связи Б, полученное в результате парсинга файла версии 2", было выявлено, что в данных в миграций оказалось меньше 
в первом случаее, нежели во втором. Таким образом, у нас куда-то "пропадают" данные, причем данные, которые идентичны в некоторых атрибутах. Приняв во внимание, что отношение в коде -- это множества, то нужно определить, 
"действительно гипотеза, что данные теряются в следствие использования множества". По итогу, так действительно и оказывается, что часть данных, теряется, поскольку множество не содержало дубликатов. Стоит сказать, что дубликаты определялись
по определенному набору атрибутов.

# По итогу
Интересное направление рассуждения, при котором поиск проблему идет не в ключе "есть N вариантов источника проблем", а "проблемы нет в M вариантах". На самом деле, за свою карьеру. во время отладки я искал ошибки бОльшую часть времени
ища их там где нет, что сказывалось на времени решения задачи. Верно ли, что если ошибка заключается в П, то в не П ошибки не существует? Если П является фактической причины ошибки, и, принять во внимание, что П -- это некоторое пространство,
то все, что не входил в П -- не является причиной ошибки. Возможно, в некоторых случаях, когда имеется характеристика ошибки, то на основе этих характеристик можно выделять из юниверсума неподходящие варианты, которые помогут в дальнешейм
решении проблемы.
