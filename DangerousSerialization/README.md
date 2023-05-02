Опасность сериализации данных

1)
a) был проект, где также важные структура данных (финансовые отчеты по срвисам) были 
лишь в сериализованном виде с помощью openapi, а сами отчеты формировались с помощью процедур.
обращение к отчетам, при локальной разработке с postman происходил через id-ки. Проект был
написан на php laravel. Про структуры данных сложно сказать, так как классы в основном 
отражали модели в БД, не более.

b) На текущей работе, ситуация аналогичная, и специальных структур данных для отражения
сущностей не имеется (даже банальных wrap-классов). На своей карьере не наблюдал
эксплицитного отражентя данных, но и работаю не в больших командах.

2)
Как можно избегать импользование id с REST? Я бы подходил к вопросу со следующей логикой...
Если требуются id-ки, то вам нужно както идентифицировать сущность, значит, нужен иной  
способ идентификации. Как мне кажется, тут идет "конитивный баг" от реляционных СУБД, где
id воспринимается как идентификатор (изучив реляционную теорию Кейта, это кажется это совершенно
неправильным). То есть, тут вопрос не столько в том, как "обойти нюанс Rest'a", сколько, как отразить реальный прикладной мир в наш мир модели данных. 
Например, нужно однозначно идентифицировать пользователя, то можно использовать комбинацию
имя+почта. 

Но, что если нам потребуется отслеживать последовательность действий, как непример 
"создал объект, удалил объект" без id? А вот тут уже вопрос структур данных.
Если нужно удалить только что созданный объект (сделать undo) то стэк решит вопрос.
Да, пример притянут за уши, но что если вы формируете сложный объект (автомобиль собираете)?
Тут, чтобы нн обращаться с id аатомобиля, достаточно FSM, а обращение к ней будет о
идентификатора пользователя.

Тут на самом деле в зависимости от контекста сделать грамотнее можно по всякому.
Да и рассуждаю я эфимерно, поскольку я видел только Rest с id в рабочих проектах.
Но сама идея и проблемы связанные с id я увидел в ребочих проектах