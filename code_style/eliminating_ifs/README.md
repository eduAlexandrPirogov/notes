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


# Пример 2

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

Стало:
```go
```

# Пример 3

Было:

Стало:


# Пример 4

Было:

Стало:


# Пример 5

Было:

Стало: