# Как выбесить интервьюера (2)

# Случай первый

Собеседование в ОЗОН.
На данном собесе я не смог понять, что от меня хочет услышать интервьюер  на двух секциях.

Первая секция -- проектирование реляционной СУБД. Попросили спроектировать схему из 3-ех отношений. На момент собеседования я полгода не затрагивал тему проектирования 
реляционных схем, поэтому немного подзабыл, но на "мышечной памяти" спроектировал без каких либо проблем -- в моем варианте отсутсвовали возможные варианты null, 
не было булевых атрибутов и хорошо разбил отношения на различные связи. Да, стоит отметить, что перед решением задачи, я уточнил, нужна ли им модель или схема (они просили модель).

Предложив свое решение, меня спросили "а все ли в порядке с моей моделью?  Может чего-то не хватает?". И минуты 3 я копался у себя в голове, но не мог понять, что требуется улучшить.
Ограничения на уникальность имелись, ссылочная целостнасть отражалась, нормализация поддерживалась. По итогу меня спросили "а где Foreign Key?". Меня настолько ввел в ступор данный вопрос,
что не знал, как ответить на него, ведь в ER-модели ссылочная целостность отражается между отношениями всегда, если она имеется между ними. Ей Богу, такие вопрос "а может что-то не так?" 
сбивают с толку больше, нежели помогают.

...

Позже началась секция System Design. Для меня она была впервые и, прочитав книгу по подготовку к собесам по System Design и, посмотрев конференции, в целом понимал, что ожидать.
Интервьюеры дали следующее задание: у них есть сервис, который синхронно отправляет данные другому сервису, из-за чего потенциально возникает bottle neck. Вопрос: что будешь делать?
Я быстро дал ответ на асинхронщину и то, как можно синхронизировать этот процесс. Далее меня осыпали вопросами типа "а что если упадет сервис/бд" и т.д на которые с горем пополам давал ответы.
И последний вопрос данной секции был следующим: "а что если упадет все?". Ну.....не проектировать так, чтобы все падало :)
Ответить я на него не смог, поскольку:
1) мне не дали домен возможностей, т.е. что я мог применять, а что не мог.
2) для меня странно проектировать систему, которая может так вот падать и думать в такой вот ситуации было в новинку.


# Случай второй

Не помню, куда было собеседование, на на техническом интервью на Go была следующая задача. Представьте, у вас есть большой массив данных. Как в Go вы параллельно обработаете эти данные,
чтобы это было "по Go-шному". 

Задача была простой и я с ходу дал два паттерна для задачки:
1) worker pool
2) fanin->fanout

Но интервьюер сказал, что есть "еще более трушный Go-подход". Минут 5 я ломал голову и не понимал, что от меня требуется, после чего ответил "возможно я не знаю про какой-то паттерн, 
который подойдет здесь лучше, чем перечисленные мною варианты".

Ответ от интервьюера был таковым: нужно использовать каналы и из каналов горутины берут таски.
Занавес.



# По итогу

Случаи, когда мне не понятен вопрос не редок, но никогда не стеснялся задавать много уточняющих вопросов. Это показывает меня с хорошей стороны, да и шансов на корректный ответ будет у меня больше.
Да, проводить собесы со стороны интерьюера тоже задача не из легких и нужно уметь навести на верную мысль кандидата, дабы не сбить его с толку. Два вышеперечисленных случая как раз из той 
песни, когда вопросы меня наоборот сбивали с толку и не мог понять, что хотят от меня услышать.
Тут ещё играет роль мышление кандидата. Например, получив задачу, у меня в голове возникает несколько десятков решений и, по мере уточняющих вопросов, я постепенно отсекаю неверные варианты.
Бывают ситуации, когда в моем множестве решений остается 2-3 варианта, мне дают такое условие, которое отсекают все мои варианты, и я сижу с пустым множеством решением.

За последние 3 месяца у меня было с десяток собеседований. Мне встречались вопросы, ответ на которые я не мог дать. И вот, как мне показалось, лучшим вариантом полностью объяснить свое
виденье задачи, обосновать свое решение и сказать о нем. Вы в любом случае получите правильный ответ (и тем самым восполните пробел знаний), но и покажете себя, как кандидата рассуждающего.