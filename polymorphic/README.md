# Правильный полиморфный код

Никак не мог понять, как бОльшее количество информации о функции, типе позволит выполнять над ними большее количество операций.
Но пример с ToDoList и доводами в пользу полиморфного кода, вроде понял. 
Тут важный момент с количеством смыслов и реализаций. Во-первых, полиморфный код и правда сужает количество реализации при наличии бОльшего смыслов (семантики).
Допустим, у нас имеется сервис, который принимает разные метрики, и закидывает их в базу данных.
Как я это сделал изначально:
Пример 1.
```go
type Metric struct {
  ...
}

type DB struct {
   storage map[string]Metric
}

func (db* DB) Read(metric Metric) {
   ...
}


func (db* DB) Write(metric Metric) {
   ...
}

func (db* DB) Update(metric Metric){
   ...
}

func InsertMetricHandler(...) {
    Metric := ...
    DB.Write(Metric).
}
```

Тут есть момент с мономорфным кодом, а именно Metric и работаем именно с ней. Так понимаю, что поскольку у нас есть вся информация о метриках, 
то мы можем делать большее количество операций над ними за счет того, что 
1) мы знаем что наша "БД" работает с метрики
2) мы знаем, как эти метрики устроены, мы должны приводить полученные данные к метрикам путем различных манипуляций

Чтобы сделать код полиморфным, я бы сделал не просто метрики более общими, а хранимые данные более обищми -- таким образом наша БД не будет знать "знать" о внешнем
мире ничего.

Это код можно сделать более полиморфным следующим образом:
```go
type Storagable interface{
    Serialize() string // бинарное представление, которые легко можно записать
    Deserialize(string) Storagable //из сериализуемого значения получаем Storagable
}

type Metric struct {
  ...
}

func (m * Metric) Serialize() string {
   //...
}

func (m* Metric) Deserialize(string) Storagable {
   //...
}

type DBable interface {
   Read(s Storagable)
   Write(s Storagable)
   Update(s Storagable)
}


func (db* DB) Read(s Storagable){
    res := s.Serialize()
    // Далее ищем по полученному res.
    // Поскольку все данные у нас сериализованны, то иных операций мы делать и не может,
    // кроме как искать их в таком виде
}


func (db* DB) Write(s Storagable) {
   res := s.Serialize()
   // записываем это значение в БД коим-то образом
}

func (db* DB) Update(s Storagable){
    res := db.Read(s)
    //и обнаовляем результат
}

func InsertHandler(...) {
    Metric := ...
    DB.Write(Metric).
}
```

В итоге, тут получается такая ситуация, что внешним миром для нашей БД выступает Storagable. 
Storagable может быть что угодно, и, важный момент тут такой, что у нас получается по факту единственные операции над Storagable -- сериализовать и десериализовать.
Иных вариантов развитий событий быть не может. 
Интерфейс для метрик может дополнять свои методы, благо interface embedding в go это позволяет сделать легко. 

Получается следующая картина: формируем Storagable элемент --> закидываем в DB --> DB сериализует его и записывает.
С мономорфным кодом, мы формируем ТУ метрика, которая ПОДОЙДет к нашей DB (Наша DB зависит от внешего мира, получается?), БД проводит дополнительные манипуляции
с метркиами.

То есть я пониманию следующим образом:
Мономорфный код дает нам больше информации, что дает больше поводов держать в голове некоторые детали, над которыми мы должны выполнять доп операции.
Хоть и реализаций у нас меньше, но это выполняется за счет ограничений и тех самых деталей о типе. Тут скорее низкоуровневое мышление над кодом
(мы думаем над взаимодействием конкретный типах, на не о взаимодействиях структур данныХ)

Полиморфный код дает меньше информации, дает нам, казалось бы, больше свободы. Но тут и есть тот самый момент, когда, как в случае с БД, мы говорим, что
ты можешь использовать любой тип данный Storagable", но одновременно с этим, кроме как сериализации и десериализации мы делать-то и ничего не можем.
Уверен, тут со стороны дискретной математики и мат логики можно объяснить красивее -- есть объект. И есть два отношения -- универсум по отношению к объекту
и объект по отношению к универсому. В общем тут есть ещё пища для размышлений.

По крайней мере я понял, за счет чего полимфорный код дает меньше пространства для реализаций.
 

---

Пример 2.
Приложение десктоп, в котором огромное количество различных панелей
Было
```cpp
void ComparePanel::updateCompareView(std::map<std::string, int>& colored_items)
{
	 // Обновление панеля сравнения
};

void CollectionStructurePanel::updateCollectionStructreView(MongoCollection& collection)
{
	// 
}

void ChoicePanel::updateCollection(std::map<std::string, Collection> schemas)
{
   //Обновление схемы коллекций
}

```

Тут можно и сократить кодовую базу, и за счет полиморфного кода, уменьшить количество разлчных реализаций для однотипных панелей

Стало:
Можно создать интерфейс, который задаст "тон" метода, а также полиморфнуть коллекции.
```cpp

class Panel extends wxPanel {
public:
  // Обновляет панель и возвращает код успеха операции.
  virtual int UpdateCollection(Container) int = 0;
}

class Container {
  public:
     //Задаем операции над контейнеорм
     void Insert(Item item)
     void Find(Item item)
     // И так далее
}
```

Суть такова. Начну с коллекций. В приложении много вещей связанные с коллекциями, у которых довольно схожее поведение. Мы оформляем это в отдельный класс, причем 
Container лучше сделать template'ом -- хранящиеся Item's не будут знать о внешнем Containere, а Container -- о хранимых предметах.
Панели, которые работаю с контейнерами, также имеют "полиморфное" поведение -- мы задали "тон", (лучше возвращать не INT, а заданный enum-поведения для обновления
панелей), который вроде бы и подразумевает различные смыслы, но реализаций тут немного.


Ещё частая ситуация на прошлой работе, когда для разных групп сервисов нужно было считать по-разному различные финансы, из-за чего я создавал различные
SQL процедуры

Пример 3.
Код приблизительный, т.к. доступа к коду на прошлой работе не имею
```SQL

CREATE OR REPLACE SERVICE1_FORECAST(turnover int, coeff float)
  // Курсор для того, чтобы получить основные данные для сервиса
BEGIN
 // Длинная портянка рассчетов для финансов сервиса
 turnover := ...
 proceeds := ...
 //...
END


CREATE OR REPLACE SERVICE2_FORECAST(fact int, proceeds int, date datetime)
  // Курсор для того, чтобы получить основные данные для сервиса
BEGIN
 // Длинная портянка рассчетов для финансов сервиса
 turnover := ...
 fact_accepted := ...
END
```

Суть в том, что для каждого сервиса были общие алгоритмы рассчета тех или иных функций, например факт согласованный или оборот.
Вместо того, чтобы для каждого сервиса писать процедуру, можно было бы написать по 2 функции для составляющих этих рассчетов
1) функцию для рассчета факта согласованного
2) функцию для рассчета оборота

И большинство функций для сервисов схлопывались бы в единую -- берем запросом нужные сервисы и применяем для них нужные чистые функции. Причем, 
на ФП было бы намного лучше с foldl

Стало бы:
```SQL

CREATE OR REPLACE FACT_ACCEPTED(proceeds int, coeff int)
RETURNS integer
  LANGUAGE plpgsql AS
$func$
BEGIN
   fact_accepted := ...
   RETURN fact_accepted;
END
$func$;


CREATE OR REPLACE PROCEEDS_SERVICES_2(costprice int, turnover int)
RETURNS integer
  LANGUAGE plpgsql AS
$func$
BEGIN
   proceeds := ...
   RETURN proceeds;
END
$func$;
```

Выглядит слишком абстрактно, но суть такова, чтобы сделать чистые функции, которые не "смотрят" на внешний мир -- им все равно, для каких сервисов они считают 
показатели, они просто получают входные данные и формируют выходные. Полиморфность заключается в том, что смысл может быть применен к различным
конкретным сервисам, но вот пространства для реализации у нас почти нет иной. Да, это не ФП, но если рассматривать как строительный блок, то очень даже хорошо.

Пример 4.

Рассмотрми пример на ФП.
В случае ФП, мы можем использовать чистые функции, дабы сделать наш код более полиморфным. В ФП по моим ощущениям с этим дела обстоят проще.
Мне нужно было протестировать код для чата:
1) Подключение пользователей
2) Регистрация пользователей
3) Отправка сообщений

Суть первых тестов состояла в последующем переборе списка входных данных и исполнения над ними определенных действий
Было:
```erl
-module(dbtest).
-export([
    conn_test/0,
    reg_test/0,
    start/0
]).

start()->
    reg_test(users()),
    conn_test(users()),
    msgs_test(users(), users()),
    disconn_test(users()).

users()->[alex, ivan, sasha, misha, andrei, kristina, sveta, maria    ].

reg_test()->iter_users(users(), reg_fun()).
conn_test()->iter_users(users(), conn_fun()).
disconn_test()->iter_users(users(), disconn_fun()).
msgs_test()-> iter_msgs(users(), users()).


reg_test([H|T])when T == [] -> dbusers:register(Elem);
reg_test([H|T])->
    dbusers:register(Elem),
    reg_test(T).
    

conn_test([H|T])when T == [] -> dbconn:connect(Elem);
conn_test([H|T])->
    dbconn:connect(Elem),
    conn_test(T).

disconn_test([H|T])when T == [] -> dbconn:disconnect(Elem);
disconn_test([H|T])->
    dbconn:disconnect(Elem),
    disconn_test(T).
    
    
msgs_test([From|Users], [To|Rest]) when Rest == [] -> 
    dbmsgs:send(hello, From, To),
    msgs_test(Users, users());
msgs_test([From|Users], [To|Rest]) when Users == [] ->dbmsgs:send(hello, From, To);
msgs_test([From|Users], [To|Rest])-> 
    dbmsgs:send(hello, From, To),
    msgs_test([From|Users], Rest).
```

Можно обощить проход цикла одной функции с конркетной лямбда.

```erl
-module(dbtest).
-export([
    conn_test/0,
    reg_test/0,
    start/0
]).

start()->
    reg_test(),
    conn_test(),
    disconn_test(),
    conn_test(),
    msgs_test().

users()->[alex, ivan, sasha, misha, andrei, kristina, sveta, maria    ].

reg_test()->iter_users(users(), reg_fun()).
conn_test()->iter_users(users(), conn_fun()).
disconn_test()->iter_users(users(), disconn_fun()).
msgs_test()-> iter_msgs(users(), users()).


reg_fun()->fun (Elem)->dbusers:register(Elem)end.
conn_fun()->fun (Elem)->dbconn:connect(Elem) end.
disconn_fun()->fun (Elem)->dbconn:disconnect(Elem) end.

# Полиморфный код
iter_users([H|T], Fun)when T == [] -> Fun(H);
iter_users([H|T], Fun)->
    Fun(H),
    iter_users(T, Fun).

iter_msgs([From|Users], [To|Rest]) when Rest == [] -> 
    dbmsgs:send(hello, From, To),
    iter_msgs(Users, users());
iter_msgs([From|Users], [To|Rest]) when Users == [] ->dbmsgs:send(hello, From, To);
iter_msgs([From|Users], [To|Rest])-> 
    dbmsgs:send(hello, From, To),
    iter_msgs([From|Users], Rest).
```

Пример 5
