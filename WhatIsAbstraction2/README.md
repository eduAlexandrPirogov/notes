Что такое абстракция

Пример 1.


Пример 5. Он получился насыщенным на рассуждения.
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
