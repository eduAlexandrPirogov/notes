# Избавляемся от условных конструкций

# Пример 1

Прохожу курсн на Яндексе, и там есть такой проект -- собираются два типа показателей counter и gauge, и нужно их передавать на сервер.
У нас имеется такая изначальная структура:
```go

// Serializable representation of metric
type Metrics struct {
	ID    string   `json:"id"`              //Metric name
	MType string   `json:"type"`            // Metric type: gauge or counter
	Delta *int64   `json:"delta,omitempty"` //Metric's val if passing counter
	Value *float64 `json:"value,omitempty"` //Metric's val if passing gauge
	Hash  string   `json:"hash,omitempty"`
}
```

И отдельные структуры для каждой измерительной метрики:
```
//counter
type PollCount counter // Count of times when Metrics was collected

type Polls struct {
	PollCount PollCount
}

//gauge
type FreeMemory gauge
type CPUutilization1 gauge
type TotalMemory gauge

type OpsUtil struct {
	FreeMemory      FreeMemory
	CPUutilization1 CPUutilization1
	TotalMemory     TotalMemory
}
```
Ситуация была сложная тем, что когда данные приходили на сервер, на две эти структуры был один json, которые делался из Metrics (см. выше).
Из-за этого по коду сервера постоянно были проверки:

if metric.type == "gauge" ...
else if metric.type == "counter" ...
else ...

Такие if-ы расползилсь по всем "уровням" архитектуры, и я решил эту проблему следующим образом.

Я подумал, что если у нас один json, структура которого известа, и она едина для всех типов метрик, то можно представить этот json в виде кортежа.
Создал интерфейс для поддержки кортежей:
```go

type Tupler interface {
	// ToTuple converts type that implements Tupler interface to Tuple
	ToTuple() Tupler

	// SetField adds k/v pair to Tuple.
	//
	// Pre-cond: given key to be set and value for key
	//
	// Post-cond: return new tuple with updated field value
	SetField(key string, value interface{}) Tupler

	// GetField returns value by key of tuple.
	//
	// Pre-cond: given key
	//
	// Post-cond: returns value of field.
	// If field is exists with key returns val and bool = true
	// Otherwise return nil and bool = false
	GetField(key string) (interface{}, bool)

	// Aggregate aggregates to tuples to union them
	//
	// Pre-cond: given tuple to aggregate with
	//
	// Post-cond: returns new Tupler and error
	// If success return Tuplers and error = nil
	// Otherwise return nil and error
	Aggregate(with Tupler) (Tupler, error)
}
```

Создал для каждого вида метрки соответствующие АТД, реализующие интерфейс Tupler

```go
// Struct to represent gauge metrics as tuple in FP style
type GaugeState struct {
	Name  string   `json:"id"`              //Metric name
	Type  string   `json:"type"`            // Metric type: gauge or counter
	Value *float64 `json:"value,omitempty"` //Metric's val if passing counter
	Hash  string   `json:"hash,omitempty"`
}

// Struct to represent Counter metrics as tuple in FP style
type CounterState struct {
	Name  string `json:"id"`              //Metric name
	Type  string `json:"type"`            // Metric type: gauge or counter
	Value *int64 `json:"delta,omitempty"` //Metric's val if passing counter
	Hash  string `json:"hash,omitempty"`
}
```

Поскольку на сервер приходит единый, json, то все if-ы уходят в "конструкто" то Tuple:
```go
// Serializable representation of metric
type Metrics struct {
	ID    string   `json:"id"`              //Metric name
	MType string   `json:"type"`            // Metric type: gauge or counter
	Delta *int64   `json:"delta,omitempty"` //Metric's val if passing counter
	Value *float64 `json:"value,omitempty"` //Metric's val if passing gauge
	Hash  string   `json:"hash,omitempty"`
}

func (m Metrics) ToTuple() tuples.Tupler {
	var tuple tuples.Tupler
	switch m.MType {
	case "counter":
		if m.Delta == nil {
			tuple, _ = createCounterState(m.ID, m.MType, "")
			return tuple
		}
		tuple, _ = createCounterState(m.ID, m.MType, fmt.Sprintf("%d", *m.Delta))
	case "gauge":
		if m.Value == nil {
			tuple, _ = createCounterState(m.ID, m.MType, "")
			return tuple
		}
		tuple, _ = createGaugeState(m.ID, m.MType, fmt.Sprintf("%.20f", *m.Value))
	}

	return tuple
}
```

То есть, на сервер приходит json, который анмаршалится в Metrics, а та в свою очередь, приводит инстант Metrics к соответствующемц виду кортежа.
Теперь, весь сервер у меня работает с полиморфным кортежем, так устранил около 20 if-ов.
Единственный минус, который можно отметить, это метод:
```go
GetField(key string) (interface{}, bool)
```

вот тут без if-а не обойтись, к сожалению таков идиоматизм в Go, но с Pattern matching вообще была бы красота :)
Изменил логику до уровня, где if-ы вообще не нужны.

# Пример 2

Касательно вышеприведенного примера. Я использовал в качестве кортежа интерфйс Tupler, а типы, реализующие этот интерфейсы обычно "оборачивались" над map[string]interface{}

Go-шники мне говорили, что писать switch с assert type -- норма:

```go
switch v := i.(type) {
	case int:
		fmt.Printf("Twice %v is %v\n", v, v*2)
	case string:
		fmt.Printf("%q is %v bytes long\n", v, len(v))
	default:
		fmt.Printf("I don't know about type %T!\n", v)
	}
```

Мне это очень не нравится, и я не хотел, чтобы клиент брал поле из типа, реализующий Tupler, и в добавок писал switch.
Изначально был метод, который возвращал поле по ключу, и это поле через if-ы надо было проверять:

```go
// GetField returns value by key of tuple.
//
// Pre-cond: given key
//
// Post-cond: returns value of field.
// If field is exists with key returns val and bool = true
// Otherwise return nil and bool = false
func (t Tuple) GetField(key string) (interface{}, bool) {
	if val, ok := t.Fields[key]; ok {
		return val, true // problem that interface{} must be asserted
	}
	return nil, false
}
```

Я решил пойти по-иному пути в данном случае, и, как мне кажется, он более правильный. 
Я добавил отдельные функции, которые достают определенный тип из Tupler'а:
```go
package tuples

// ExtractString extracts string field from tuple
//
// Pre-cond: given field to extract and tuple to extract from
//
// Post-cond: extracts string value of fields.
// If field no exists or field is not string, return empty string
func ExtractString(field string, t Tupler) string {
	f, ok := t.GetField(field)
	if !ok {
		return ""
	}

	return f.(string)
}

// ExtractString extracts pointer to int64 field from tuple
//
// Pre-cond: given field to extract and tuple to extract from
//
// Post-cond: extracts pointer to int64  value of fields.
// If field no exists or field is not pointer to int64 , return nil
func ExtractInt64Pointer(field string, t Tupler) *int64 {
	f, ok := t.GetField("value")
	if !ok {
		return nil
	}
	return f.(*int64)
}

// ExtractString extracts pointer to float64 field from tuple
//
// Pre-cond: given field to extract and tuple to extract from
//
// Post-cond: extracts pointer to float64  value of fields.
// If field no exists or field is not pointer to float64 , return nil
func ExtractFloat64Pointer(field string, t Tupler) *float64 {
	f, ok := t.GetField("value")
	if !ok {
		return nil
	}
	return f.(*float64)
}

```

То есть я полностью изменил с логику с "взять поле, удостовериться в его типе и его корректном значении", на "достать значение из поля". 
Тут вопрос в том еще, стоит ли бросать panic, или возвращать какое-то дефолтное значение (оставил 2-ое).

# Пример 3

Проблема следующего примера заключается в выборе записи реплики (синхронная или с задержкой, в зависимости от ReadInterval):
```go

// Journal writes data from channel to give file
// Works like replication/ db.log journal
type Journal struct {
	File         string
	WithRestore  bool
	ReadInterval int
	Restored     map[string]tuples.Tupler
	Channel      chan []byte
}

// Start make journal stats writing data to the given file in json format
//
// Pre-cond: j have file that can be modified
//
// Post-cond: data written to the file depending on the chosen mode
// There are two modes: synch mode writes permanently data to file
// delayed mode: writes data to the file once at the given period
// Returns nil if success started otherwise returns error
func (j Journal) Start() error {
	file, err := j.openWriteFile()
	if err != nil {
		return err
	}
	// unnecessary if-else
	if j.ReadInterval == 0 {
		go func() {
			j.writeSynch(file)
		}()
	} else {
		go func() {
			j.writeDelayed(file)
		}()
	}
	return nil
}

// readByTimer writes data once in a given period from channel
//
// Pre-cond: given file to write data
//
// Post-cond: data written to the file
func (j Journal) writeDelayed(file *os.File) {
	defer file.Close()
	writer := bufio.NewWriter(file)
	read := time.NewTicker(time.Second * time.Duration(j.ReadInterval))
	for {
		<-read.C
		for {
			if bytes, ok := <-j.Channel; ok {
				writer.Write(append(bytes, '\n'))
				writer.Flush()
			} else {
				break
			}
		}
	}
}

// NewWriter writes data every time when channel got new data
//
// Pre-cond: given file to write data
//
// Post-cond: data written to the file
func (j Journal) writeSynch(file *os.File) {
	defer file.Close()
	writer := bufio.NewWriter(file)
	for {
		if bytes, ok := <-j.Channel; ok {
			writer.Write(append(bytes, '\n'))
			writer.Flush()
		} else {
			break
		}
	}

}
```

Чтобы избавиться от if-ов, тут можно пойти несколькими путями:
1) Создать интерфейс Journaler, и сделать два отдельных вида SynchJournal и DelayedJournal
2) Создать АТД Journal, который будет встроен в SynchJournal или DelayedJournal у каждого из которых будет соответствующих метод write
3) Создать функции, которые будут добавлены в структуру в зависимости от данного интервала

Реализуем 3-ий способ, так как мне кажется от более простым:
Стало:
```go

// Newjournal returns new instance of Journal
func NewJournal() Journal {
	cfg := server.JournalCfg
	readInterval := cfg.ReadInterval[:len(cfg.ReadInterval)-1]
	read, err := strconv.Atoi(string(readInterval))
	if err != nil {
		log.Fatalf("%v", err)
	}
        // От if можно было бы избавиться, если бы Ticker при 0 не выбрасывал panic...
	if read == 0 {
		return Journal{
			File:        cfg.StoreFile,
			WithRestore: cfg.Restore,
			write:       writeSynch, //function-value
			Restored:    map[string]tuples.Tupler{},
			Channel:     make(chan []byte),
		}
	}

	return Journal{
		File:        cfg.StoreFile,
		WithRestore: cfg.Restore,
		//function with closure
		write: func(read int) func(in chan []byte, file *os.File) {

			return func(in chan []byte, file *os.File) {
				defer file.Close()
				writer := bufio.NewWriter(file)
				read := time.NewTicker(time.Second * time.Duration(read))
				for {
					<-read.C
					writeTo(in, writer)
				}

			}
		}(read),
		Restored: map[string]tuples.Tupler{},
		Channel:  make(chan []byte),
	}
}

// Journal writes data from channel to give file
// Works like replication/ db.log journal
type Journal struct {
	File        string
	WithRestore bool
	write       func(in chan []byte, file *os.File)
	Restored    map[string]tuples.Tupler
	Channel     chan []byte
}

// Start make journal stats writing data to the given file in json format
//
// Pre-cond: j have file that can be modified
//
// Post-cond: data written to the file depending on the chosen mode
// There are two modes: synch mode writes permanently data to file
// delayed mode: writes data to the file once at the given period
// Returns nil if success started otherwise returns error
func (j Journal) Start() error {
	file, err := j.openWriteFile()
	if err != nil {
		return err
	}

	go j.write(j.Channel, file)
	return nil
}

// NewWriter writes data every time when channel got new data
//
// Pre-cond: given file to write data
//
// Post-cond: data written to the file
func writeSynch(in chan []byte, file *os.File) {
	defer file.Close()
	writer := bufio.NewWriter(file)
	for {
		writeTo(in, writer)
	}

}

func writeTo(in chan []byte, writer *bufio.Writer) {
	for {
		if bytes, ok := <-in; ok {
			writer.Write(append(bytes, '\n'))
			writer.Flush()
		} else {
			break
		}
	}
}

```

В данном случае, в конструкторе у меня будет создавать соответствующая функция для записи.
Если интервал будет представлять из себя некоторую логику (допустим, будут по дня, или день-два-день), то тогда я бы создал мапу с объектом интервала и значением функции, 
и из мапы, в зависимости от типа интервала, брал нужную функцию записи.

Стало:


# Пример 4

Ещё одна фишка, чтобы убрать if-ы, которая мне очень нравится, и она довольно гибкая.
Встречал частенько, когда тип объекта сравнивают через if-ы, когда полиморфно это сделать нельз.
```php
public function fillAggregation() {
   foreach($this->aggregated_services as $row) {
        if ($this->finances->getType() == "Turnover") {
	    //...
	} else if ($this->finances->getType() == "Proceeds) {
	   //...
	} else {
	   ...
	}
   }
}
```

В таких случаях я следовал "изменить логику на такую, чтобы убрать if-ы, и сделал класс-контейнер, которые wraps мапу, ключом которой и являются типы.

```php
public function fillAggregation() {
   foreach($this->aggregated_services as $row) {
        $type = $this->finances->getType()
	$aggr = this->aggregate($this->typeContainer[$type])
	//...
   }
}
```

Мапа очень хороша, потому что она покрывает не только бинарные случаи (true/false), но и интервалы.
В итоге получилась логика из "над каждой строкой, если она типа А, то делай а1, если Б, то б1..." в "получить из контейнера по ключу А соответствущий а1 и выполнить действие над ним).

# Пример 5

Ну и последний пример, который хорошо удаляет if-ы, это создание собственных АТД.
Была задача, что рассчеты финансов нужно проводить от текущего месяца + 3 предыдущих (-1, -2, -3 месяц). Если в прошлом месяце от текущего (-1 месяц) не было данных, то производить рассчеты от (-2, -3,- 4месяцев).

Изначально было сделано через if-ы, поскольку нужно было сделать "быстрее":

```php
public function calculate(array $monthAgo) {
    if $monthAgo[0] != null {
       // Берем запрос, который считал за последние три месяца  
    } else {
       // Берем запрос, который считал за -2, -3, -4 месяца
    }
} 
```

Потом нужно было добавить условие до -3, -4, -5 месяцев, плюс запросы были с ORM по 20 строк, из-за чего функция разрасталась быстро.

В итоге, решил сделать АТД MonthAgo и Offset, которые возвращали нужную мне дату отступа:

```php
// Wrapper
class MonthAgo {
     public function __construct__(array $monthAgo) {
          $this->convert($monthAgo)
     }
     
     public function FirstOffset() {
         return $this->first
     }
     
     public function SecondOffset() {
         return $this->second
     }
     
     //...
}

public function calculate(array $monthAgo) {
    $this->MonthAgo = new MonthAgo($monthAgo)
    $first = $this->MonthAgo->FirstOffset()
    $second = $this->MonthAgo->SecondOffset()
    
    // Выполняем запрос с соответствуюищми отступами даты
} 
```
То есть, я изменил логику  с "проверить, есть ли в k-ом месяца данные, если нет, то проверить в k+1-ом месяце...), до "конвертировать пришедшые "сырые" данные и получить каждый отступ.

