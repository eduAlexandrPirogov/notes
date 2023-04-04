Что такое абстракция

Один из самых сложных материалов, который мне давался, посему я отражал дополнительно свои мысли под примерами.

Пример 1.

Суть опишу в конце, интересно, совпала ли она с абстракцией, выраженной в коде:
Handler.go file
```go
type MetricsStorer interface {
	// Reads all metrics and returns their string representation
	Read() []byte
  
	// Writes metric in store
	//
	// Pre-cond: given correct type name and value of metric
	//
	// Post-cond: stores metric in storage. If success error equals nil
	Write(mtype string, mname string, val string) ([]byte, error)
}

type MetricsHandler interface {
	RetrieveMetrics(w http.ResponseWriter, r *http.Request)
	UpdateHandler(w http.ResponseWriter, r *http.Request)
}

type DefaultHandler struct {
	DB MetricsStorer
}

// RetrieveMetric return all contained metrics
func (d *DefaultHandler) RetrieveMetrics(w http.ResponseWriter, r *http.Request) {
	body := d.DB.Read()
	w.Write(body)
}

// UpdateHandler saves incoming metrics
//
// Pre-cond: given correct type, name and val of metrics
//
// Post-cond: correct metrics saved on server
func (d *DefaultHandler) UpdateHandler(w http.ResponseWriter, r *http.Request) {
	   res, err := d.DB.Write(...)
	   w.Write(res)
	}
}
```
database.go file:
```go
type Storable interface {
  // Write writes metric to DB.
  //
  // Pre-cond: given correct params for corresponding metric.
  //
  // Post-cond: if params are corrects -- writes metric to DB.
  // Returns marshaled metric and nil error
  // Otherwise return empty slice of byte and error
  Write(mtype, mname, val string) ([]byte, error)
  
  // Read all metrics that stored in DB
  //
  // Post-cond: returns slice of marshaled metrics
     Read() []byte
	
}

type DB struct {
	Storage   Storable
	Journaler journal.Journal
}
```

Два интерфейса, Storable и MetricsStorer представляют собой абстракцию записи полученных метрик по API в DB. 
Нам нужно как-то записать метрики в базу и понять успешно ли мы записали.
Цепочка вызовов операции записи: MetricsStorer.Write(mtype string, mname string, val string) ([]byte, error) --> Storable.Write(mtype, mname, val string) ([]byte, error). Можно понять, что полученные параметры преобразуются в слайт байтов, которые затем записываются. В данном случае возвращаются две переменные --
слайт байтов и результат успешной операции записи.
Если рассуждать о методе данном, то можно прийти к нескольким рассуждениям, я приведу лишь пару:
1) Данные string преобразуются в слайс байтов и записываются в базу, возвращается этот слайт и nil error, самый простой случай.
2) Если слайс байтов, записываемые Storable уже имеются в базе, то возвращаем этот слайт байтов и ошибку not found
3) Если параметры не верны, то даже не пытаемся преобразовать их в слайс, а сразу возвращаем пустой слайт с соответствующей ошибкой.

В зависимости от реальных данных, конкретных спецификаций по данным, мы можем применить одну из вышеприведенных вариантов. 
При этом, мы точны в рассуждениях -- успех операции записи возвращается переменной error, а записываемые данные -- слайсом байтов.
операция

Рассмотрми теперь операцию записи. 
Цепочка вызовов операции записи: MetricsStorer.Read() []byte --> Storable.Read() []byte. Вот в данном случае, абстракция чтения не так точна.
Если, база имеет ошибку, которая в последствии обрабатывается и возвращается пустой слайт, как нам понять, была ли это ошибка или просто нет данных?
Тут, как и вслучае с абстркцией записи данных, лучше добавить в возвращаемый результат error, который будет указывать на успех операции.
MetricsStorer.Read() ([]byte, error) --> Storable.Read() ([]byte, error). Абстракция становится однозначной и довольно точной.

Данная абстракция чтения и записи показывает, что данный участок кода обладает таким свойством, что операция либо совершается успешно (error = nil) 
и возвращается результат записи ([]byte), получился шаблон.

По итогу, у нас получается вот такая простая абстракция чтения и записи данных.

====================================================================================================================================================

Пример 2.

Следующий пример. Опять же сначала код, попробуйте понять по цепочке вызовов суть абстркции, а дальше я приведу истинное значение.
Суть заключается в том, что имеются метрики, которые собирают значение CPU, StackSys, и эти метрики отправляются куда-то.
```go
// Metricalbes entities should update own metrics by Read() errpr method
// Read make metrics update own value. Returns nil if success otherwise error
type Metricable interface {
	// Read makes metrics update own values
	//
	// Pre-cond:
	//
	// Post-cond: if read finished successfully and error return nil then metrics was updated
	Read() error
	
	// AsMap returns own metrics as map where key is name of metric
	//
	// Pre-cond:
	//
	// Post-cond: returns new map (immutable) of own metrics
	AsMap() map[string]interface{}
}

// Trackarable collects metrics
// InvokeTrackers make stored metrics update own values
type Trackerable interface {
	// InvokeTracker makes trackers update own values
	//
	// Pre-cond:
	//
	// Post-cond: if error == nil then metrics updated own values
	// Otherwise metrics wasn't updated
	InvokeTrackers() error
}
```

Итак, имеется Metricable, которая что-то считывает Read(), который может завершиться неудачно (error). Также имеется метод, который выдает нам мапу,
ключ у которой строка а значение -- что угодно. Поскольку AsMap() не принимает аргументов, то экземпляр, реализующий данный интерфейс, должен преобразовать
собственные значения в мапу.
Также имеется Trackerable, который заставляет метрики обновить собственные значения. Поскольку InvokeTrackers() не имеет параметров,
то экземпляр, реализующий данный интерфейс, должен invoke собственный тип метрики Metricable. 
Получается следующая цепочка вызовов: Trackerable.InvokeTrackers() error --> Metricable.AsMap() map[string]interface{} --> Metricable.Read() error. 

Насколько точна абстракция относительно обновления метрик? Операция Read() возвращает лишь статус успеха операции. 
Metricable просто перезаписывает свои старые значение при выполнении Read(), и, если не произошло фатальных ошибок, то error всегда будет nil.
Относительно Trackerable.InvokeTracker() ситуация аналогичная.

Получился шаблон по чтению, сбору и обновлению метрик.
На самом деле комментарии даже лучше подправить, написать, что работа идет со всем, что можно измерить, а не конкретно с метриками.

====================================================================================================================================================

Пример 3.

Шаблон по работе с ТГ-ботом и взаимодействие с БД. Вот тут уже приведу неточную абстракцию и приведу примеры, как ее улучшить.

```go
type DB interface {
 RegisterUser(chatId int64, username string) error
 ReadTasks(username string) ([]db.Task, error)
 ReadUsers() ([]db.User, error)
 AddTask(username string, task string) error
 RmTask(username string, task uint64) error
 UserStats(username string) (db.Stats, error)
}

type TgCommandHandler interface {
 ProcessCommand(isWaitingInput *bool, update *tgbotapi.Update, bot *tgbotapi.BotAPI)
 ProcessAddTask(msg tgbotapi.MessageConfig, update *tgbotapi.Update) tgbotapi.MessageConfig
 ProcessRegister(msg tgbotapi.MessageConfig, update *tgbotapi.Update) tgbotapi.MessageConfig
 ProcessTaskList(msg tgbotapi.MessageConfig, update *tgbotapi.Update) tgbotapi.MessageConfig
}

type TgBot struct {
 Token string
 //Change to interface
 Bot *tgbotapi.BotAPI
 //Change to interface
 Scheduler scheduler.Scheduler
 Storage   DB
}

//Некоторые участки кода, к которым придется перенаправляться. Такие участки раскиданы по файлам
func (t *TgBot) processCommand(isWaitingInput *bool, update *tgbotapi.Update, bot *tgbotapi.BotAPI) {
 // Create a new MessageConfig. We don't have text yet,
 // so we leave it empty.

 msg := tgbotapi.NewMessage(update.Message.Chat.ID, "")
 // Extract the command from the Message.
 switch update.Message.Command() {
 case "addtask":
  *isWaitingInput = true
  msg = t.processAddTask(msg, update)
 case "tasklist":
  msg = t.processTaskList(msg, update)
 case "begin":
  msg = t.processRegister(msg, update)
 case "stats":
  msg = t.processStats(msg, update)
 default:
  msg.Text = "Unknown command"
 }
 SendMessage(msg, bot)
}

func (t *TgBot) processCommand(isWaitingInput *bool, update *tgbotapi.Update, bot *tgbotapi.BotAPI) {
 // Create a new MessageConfig. We don't have text yet,
 // so we leave it empty.

 msg := tgbotapi.NewMessage(update.Message.Chat.ID, "")
 // Extract the command from the Message.
 switch update.Message.Command() {
 case "addtask":
  *isWaitingInput = true
  msg = t.processAddTask(msg, update)
 case "tasklist":
  msg = t.processTaskList(msg, update)
 case "begin":
  msg = t.processRegister(msg, update)
 case "stats":
  msg = t.processStats(msg, update)
 default:
  msg.Text = "Unknown command"
 }
 SendMessage(msg, bot)
}
```

Суть заключается в том, что в зависимости от команды, будет вызвана соответствующая функция БД. Шаблон  "команда бота --> команда БД" получился
неточным, а абстракция -- не лучшей. Во-первых, тут нет комментариев. Во-вторых, приходится постоянно перенаправляться, чтобы убедиться в своих домыслах.
В третьих, заголовок функций интерфейсов мало о чем говорит.

Теперь по порядку. Чтобы сделать абстракцию "обработка команды ТГ --> соответствующая обработка БД", нужно переработать часть "обработка команды ТГ".
Действительно, интерфейс DB, который отвечает за взаимодействие с БД, получился довольно точным, осталось лишь добавить комментарии:
```go
type DB interface {
 // RegisterUser registers user in system.
 // 
 // Pre-cond: given chatID and unique Username of user
 //
 // Post-cond: If user wasn't registere, then DB register him and error = nil
 // Otherwise returns corresponding erro
 RegisterUser(chatId int64, username string) error

 // RmTask remove task for given username
 //
 // Pre-cond: given registered username and existing task id
 //
 // Post-cond: if user is registered and task is exists with given id 
 // Task is removed and error = nil
 // Otherwise return error
 RmTask(username string, task uint64) error
 
 // UserStats collects stats and writes it to the struct db.Stats
 //
 // Pre-cond: given registered username
 //
 // Post-cond: if username is registered, returns Stats with filled values and error = nil
 // Otherwise returns empty struct and error
 UserStats(username string) (db.Stats, error)
}
```

Судя по сообщениям мы понимаем шаблон взаимодействия с БД. В случае обновления мы возвращаем статус успеха. В случае чтения, мы 
возвращаем результат и также статус усупеха.

Переработаем часть шаблона "обработка команд ТГ". 
Во-первых, возвращаемое значение MessageConfig (просто сообщение, аналогично, если бы мы возвращали string), не говорит нам ни о чем.
Во-вторых, если убрать параметры, нужные лишь для API Telegram'а, то об интерфейсе мало что известно, лишь назначение операций.
Проблема в там, что в ProcessAddTask мы можем добавить логику ProcessRegister и никак не поймем это!

Что тут можно сделать?
1) Изменить результат возвращаемого значения
2) Изменить параметры методов
3) Добавить комментарии

Важная ремарка. Кажется, что я спускаюсь в реализации, но нет. Я рассматриваю вариации шаблонов, какие абстракции можно использовать, код выступает
в роли эскиза/псевдокода.
```go
type TgCommandHandler interface {
  ProcessCommand(isWaitingInput *bool, update *tgbotapi.Update, bot *tgbotapi.BotAPI)
  ProcessAddTask(msg tgbotapi.MessageConfig, update *tgbotapi.Update) tgbotapi.MessageConfig
  ProcessRegister(msg tgbotapi.MessageConfig, update *tgbotapi.Update) tgbotapi.MessageConfig
  ProcessTaskList(msg tgbotapi.MessageConfig, update *tgbotapi.Update) tgbotapi.MessageConfig
}
```

Получилось:
```go
type TgCommandHandler interface {
  //ProcessCommand handles commands that came from user.
  // Return bool if commands user inputed a command and reply with some message
  //
  // Pre-cond: given update of tgbotApi
  //
  // Post-cond: bool = true if we waiting for user respone otherwise bool = false
  // error = nil if nothing happned. Otherwise if error !=nil then bool = false
  ProcessCommand(update *tgbotapi.Update) (bool, error)
  
  // ProcessAddTask handle command of adding task
  //
  // Pre-cond:
  //
  // Post-cond: handle user response and return instance of msgs.AddTaskMessage.
  // msgs.AddTaskMessage contains corresponding message.
  ProcessAddTask(update *tgbotapi.Update) msgs.AddTaskMessage
  
   // ProcessTaskList handle command of adding task
  //
  // Pre-cond:
  //
  // Post-cond: handle user response and return instance of msgs.TaskListMessage.
  // msgs.TaskListMessage contains corresponding message and reply keyboard with tasks
  ProcessTaskList(update *tgbotapi.Update) msgs.TaskListMessage
}
```

Теперь у нас получается следущая картина абстркции "обработка команд ТГ -> обработка БД". Тут хоть и не обойтись без перенаправления, но 
это будет сделано в рамках одного файла. Обработка команд ТГ теперь возвращает некоторый шаблон ответов, посмотрев который (опять перенаправление),
мы можем понять суть ситуации. Главное, что мы избавились от неоднозначности: в первой версии мы возвращали на всех Message.
Сейчас же мы видим иную ситуацию. Соответствующие варианты на обертки сообщений раскрывают картину лучше и четче.
Присутствует однозначное отражение, что дает прибавляет точность.


====================================================================================================================================================

Пример 4.

```go
type Storable interface {
    // Replicate sends to ReplWriter messgae to replicate data
    //
    // Pre-cond: given a pointer to ReplWriter
    //
    // Post-cond: signal to replicate data was sent to replWriter
    // If succes than returns number of replicated bytes and error = nil
    // Otherwise returns -1 and error
    Replicate(r *ReplWriter) (int, error)
}

//ReplWriter receive data and writes it to dest.
type ReplWriter interface {
    // Receive reads incoming data to replicate
    //
    // Pre-cond: given data to replicate
    //
    // Post-cond: RelWriter runs thread that accepts incoming data
    // If succes than returns number of read bytes and error = nil
    // Otherwise returns -1 and error
    Receive(data []byte) (int, error)
    
    // Write writer all incomed data. Dest can be file, other db, etc...
    //
    // Pre-cond: Given Destinationer that can write data to destionation
    //
    // Post-cond: data was written to dest. 
    // If succes than returns number of written bytes and error = nil
    // Otherwise returns -1 and error
    Write(d Destinationer) (int, error)
}

type Destinationer interface {
    // Write writes given data
    //
    // Pre-cond: given data to write
    //
    // Post-cond: writes data to destination. 
    // If succes than returns number of written bytes and error = nil
    // Otherwise returns -1 and error.
    Write(data []byte) (int, error)
}
```

Данный кусок кода демонстрирует абстракцию репликации данных. 
Шаблон: "Уведомить репликатор --> Принять данные --> Записать данные в пункт назначения--> Уведомить об успехе операции".

Насколько точно отражают абстракцию репликации данных данный кусок кода? Успех операции отражает error (получается уже сам по себе как шаблон). Возможно и не следует
возвращать int, количество записанных байтов.
Если смотреть на цепочку вызовов, то получится следующее:
"Storable.Replicate(r *ReplWriter) (int, error) --> ReplWriter.Receive(data []byte) (int, error) --> ReplWriter.Write(d Destinationer) (int, error) --> Destination.Write(data []byte) (int, error)"
Вопрос касательно двух методов ReplWriter'a: Receive и Writer -- насколько точно они отражают абстракцию? Если с Write все довольно однозначно и точно, то
Receive немного запутывает -- зачем возвращать int. В общем, лучше убрать Receive и сделать цепочку вызовов вот такой:

"Storable.Replicate(r *ReplWriter) (int, error) --> ReplWriter.Write(d Destinationer) (int, error) --> Destination.Write(data []byte) (int, error)"
Проблема первой цепочки в том, что Receiver и Write не коммутативны + Receive не добавляет точности.

====================================================================================================================================================

Пример 5.

Рассмотрми абстракцию volum'ов из Docker'а:

```go// Driver is for creating and removing volumes.
type Driver interface {
	// Name returns the name of the volume driver.
	Name() string
	// Create makes a new volume with the given name.
	Create(name string, opts map[string]string) (Volume, error)
	// Remove deletes the volume.
	Remove(vol Volume) (err error)
	// List lists all the volumes the driver has
	List() ([]Volume, error)
	// Get retrieves the volume with the requested name
	Get(name string) (Volume, error)
	// Scope returns the scope of the driver (e.g. `global` or `local`).
	// Scope determines how the driver is handled at a cluster level
	Scope() string
}

// Volume is a place to store data. It is backed by a specific driver, and can be mounted.
type Volume interface {
	// Name returns the name of the volume
	Name() string
	// DriverName returns the name of the driver which owns this volume.
	DriverName() string
	// Path returns the absolute path to the volume.
	Path() string
	// Mount mounts the volume and returns the absolute path to
	// where it can be consumed.
	Mount(id string) (string, error)
	// Unmount unmounts the volume when it is no longer in use.
	Unmount(id string) error
	// CreatedAt returns Volume Creation time
	CreatedAt() (time.Time, error)
	// Status returns low-level status information about a volume
	Status() map[string]interface{}
}
```

Исходя из кода можно сделать вывод, что это отражение взаимодействия с Volume'ми докера. 
Причем это можно говорить даже не зная, что такое докер: например 
// List lists all the volumes the driver has
List() ([]Volume, error) говорит нам о том, где хранятся вольюмы.
Также отображение говорит, что взаимодействие происходит с контейнерами не посредством именно Driver'-a, это лишь контейнер, а именно с самим
экземпляром Volum'а


====================================================================================================================================================

3. По поводу определения Дейкстры  "быть на новом семантическом уровне, где можно быть парадоксально точны". Для себя выделил такой момент, что
абстракция заключается не в каких-то конкретных фичах языка, будь-то интерфейс или абстрактный класс (это просто инструменты, попытки направить абстакцию
в нужное русло, область),а в том, что это некоторый мета-язык, отражающий положение мира в коде. Хороший пример был с преобразованием чисел и работу транзисторов.
Вообще, абстракцию составляет не сами фичи языка, а их применение, насколько точно комбинация функций, классов, интерфейсов, пакетов, комментариев 
отразит логику в код. Причем, получается, что тем меньше "размер" подобных абстракций, тем более точно отражают они картину.

В общем, для себя я буду применять следующие техники:
1) Спрашивать себя, как абстракция Х отражается в Y, а затем в код
2) Насколько точна абстракция (определение коммутативности абстракций мне дается сложнее)
3) Корректировать себя, что я работаю с абстракцией, а не с кодом, посредством выявление в участке кода множество абстракций (суть в том, что абстракций может быть бесконечно много) 

====================================================================================================================================================














Пример 6. Он получился насыщенным на рассуждения.
Рассмотрим следующие фрагменты кода, которые реализуют такую логику: на API поступают метрики, которые нужно сохранить в БД, а также прочитать их.
Метрики состоят из имени. типа и значения.

Выглядит все следующим образом:
handlers file:
```go

type MetricsStorer interface {
	// Reads all metrics and returns their string representation
	Read() []byte
  
	// Read metrics with given type and name.
	//
	// Pre-cond: Given correct mtype and mname
	//
	// Post-cond: Returns suitable metrics according to given mtype and mname
	ReadByParams(mtype string, mname string) ([]byte, error)

	// Writes metric in store
	//
	// Pre-cond: given correct type name and value of metric
	//
	// Post-cond: stores metric in storage. If success error equals nil
	Write(mtype string, mname string, val string) ([]byte, error)
}

type MetricsHandler interface {
	RetrieveMetrics(w http.ResponseWriter, r *http.Request)
	RetrieveMetric(w http.ResponseWriter, r *http.Request)
	UpdateHandler(w http.ResponseWriter, r *http.Request)

	RetrieveMetricJSON(w http.ResponseWriter, r *http.Request)
	UpdateHandlerJSON(w http.ResponseWriter, r *http.Request)
}

type DefaultHandler struct {
	DB MetricsStorer
}

// RetrieveMetric return all contained metrics
func (d *DefaultHandler) RetrieveMetrics(w http.ResponseWriter, r *http.Request) {
	...
}

// RetrieveMetric returns one metric by given type and name
func (d *DefaultHandler) RetrieveMetric(w http.ResponseWriter, r *http.Request) {
	...
}

// UpdateHandler saves incoming metrics
//
// Pre-cond: given correct type, name and val of metrics
//
// Post-cond: correct metrics saved on server
func (d *DefaultHandler) UpdateHandler(w http.ResponseWriter, r *http.Request) {
	...
	}
}

```

database file:
```go
type Storable interface {
  // Write writes metric to DB.
  //
  // Pre-cond: given correct params for corresponding metric.
  //
  // Post-cond: if params are corrects -- writes metric to DB.
  // Returns marshaled metric and nil error
  // Otherwise return empty slice of byte and error
	Write(mtype, mname, val string) ([]byte, error)
  // Read all metrics that stored in DB
  //
  // Post-cond: returns slice of marshaled metrics
	Read() []byte
  // Read metric that stored in DB by given params.
  //
  // Pre-cond: given correct params for corresponding metric.
  //
  // Post-cond: if params are corrects and metrics is stored in DB returns slice of marshaled metric and nil error
  // if params are correct and metric doesn't stored -- return empty slice and nil error
  // Otherwise return empty slice of byte and error
	ReadByParams(mtype, mname string) ([]byte, error)
}

type DB struct {
	Storage   Storable
	Journaler journal.Journal
}
```

Рассмотрим, как интерфейсы MetricsStorer и Storable образуют абстракцию, которая отражает взаимодействие метрик с БД. 
Семантическим доменом в обоих случаях будет код операции (error) и результат записи/чтения. Также будем оценивать абстракцию по точности.
Абстракция (совокупность  MetricsStorer и Storable)  отражает процесс сохранения в БД полученных "сырых данных" и последующего их чтения.

Насколько эта абстракция точна?
Если следовать рекомендации, что абстракции должны быть минимальны и информативны, то можно было бы в качестве результата оставить только []byte, но есть проблема.
Например, если мы вызовем функцию ReadByParams которая вернет пустой слайс, как понять, что данных нет или произошла ошибка? 
Если же говорить об операции Read, которая возвращает только пустой слайс, то тут можно обойтись без ошибки, поскольку при чтении не может произойти никаких
ошибок, мы просто считываем все значения сохраненных метрик.

Тут есть проблема в абстракции, которую обнаружил во время написание сего текста, что выглядит она двусмысленно. Например, в коде есть такие строчки:
```go
func (d *DB) ReadByParams(mtype, mname string) ([]byte, error) {
	return d.Storage.Select(mtype, mname)
}
````
зачем нужна тогда абстракция Storable, если MetricsStorer в некоторых случаях просто делегирует работу?
Обозначу абстракцию хранения метрика как А, которая состоит из двух небольших абстракций а1 и а2.
В данном примере у меня получилось, что а1 делегирует работу а2. То есть, если представить а1 как функцию, которая принимает в качестве параметра метрику,
то можно записать это следующим образом: 
a1(m1) = a1(a2(m2)).
Не знаю как корректно сказать, но нарратив такой, что как будто я добавил лишний ненужный слой. Да, это перенаправление, но...

В общем, получилась отвратительная архитектура. Можно улучшить ее следующим образом:
1) метод Write интерфейса Storable преобразовать в Write([]byte) error. Тогда у нас будет следующая цепочка
MetricsStorer.Write(mtype string, mname string, val string) ([]byte, error) --> Storable.Write([]byte) error. 
2) метод ReadByParams(mtype, mname string) ([]byte, error) преобразовать в  ReadByParams([]byte) ([]byte, error). Тогда у нас будет следующая цепочка
MetricsStorer.ReadByParams(mtype string, mname string, val string) ([]byte, error) --> Storable.ReadByParams([]byte) error. 
Но здесь усложнится логика поиска маршализованных значений.

```go
type MetricsStorer interface {
	// Reads all metrics and returns their string representation
	Read() []byte
	// Read metrics with given type and name.
	//
	// Pre-cond: Given correct mtype and mname
	//
	// Post-cond: Returns suitable metrics according to given mtype and mname
	ReadByParams(mtype string, mname string) ([]byte, error)
	// Writes metric in store
	//
	// Pre-cond: given correct type name and value of metric
	//
	// Post-cond: stores metric in storage. If success error equals nil
	Write(mtype string, mname string, val string) ([]byte, error)
}

type Storable interface {
  // Write writes metric to DB.
  //
  // Pre-cond: given correct params for corresponding metric.
  //
  // Post-cond: if params are corrects -- writes metric to DB.
  // Returns marshaled metric and nil error
  // Otherwise return empty slice of byte and error
	Write([]byte) error
  // Read all metrics that stored in DB
  //
  // Post-cond: returns slice of marshaled metrics
	Read() []byte
  // Read metric that stored in DB by given pattern.
  //
  // Pre-cond: given correct params for corresponding metric.
  //
  // Post-cond: if params are corrects and metrics is stored in DB returns slice of marshaled metric and nil error
  // if params are correct and metric doesn't stored -- return empty slice and nil error
  // Otherwise return empty slice of byte and error
	ReadByParams([]byte) ([]byte, error)
  
}

```

Надо подумать....
