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

привели к источнмку пробенмы.

Пример 2



Пример 3

