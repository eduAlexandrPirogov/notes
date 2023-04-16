Супергибкий код

Изначально подумал, что супергибкий код будет достигаться за счет интересных конструкций и приемов в языке программировании, но тут оказалось все намного интереснее.

Пример 1.

Прохожу яндекс курс на Go и есть пример, который позволил бы использовать "журналирование для типов".
Изначально дана такая структура:
```go
type Metrics struct {
	ID    string   `json:"id"`              //Metric name
	MType string   `json:"type"`            // Metric type: gauge or counter
	Delta *int64   `json:"delta,omitempty"` //Metric's val if passing counter
	Value *float64 `json:"value,omitempty"` //Metric's val if passing gauge
}
```
Обозначим это, как типа данных А.

Если MType == counter, то Value = nil, если MType == gauge, то Delta == nil.
Эти метрики собираются на сервисе и передаются сервису, где они аггрегируются и сохраняются в БД.
Из-за наличия nil в одном из полей (Delta или Value) в коде постоянно проверялись проверки на типы.
Чтобы избавиться от постоянных проверок, я создал следующую структуру данных, которая имитирует кортеж в ФП, на сторое сервера:

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

И соответствующие для каждой из типа метрки структуры, которые реализуют интерфейс Tupler:

```go
// Struct to represent Counter metrics as tuple in FP style
type CounterState struct {
	Name  string `json:"id"`              //Metric name
	Type  string `json:"type"`            // Metric type: gauge or counter
	Value *int64 `json:"delta,omitempty"` //Metric's val if passing counter
	Hash  string `json:"hash,omitempty"`
}

// Struct to represent gauge metrics as tuple in FP style
type GaugeState struct {
	Name  string   `json:"id"`              //Metric name
	Type  string   `json:"type"`            // Metric type: gauge or counter
	Value *float64 `json:"value,omitempty"` //Metric's val if passing counter
	Hash  string   `json:"hash,omitempty"`
}
```

И у каждой этой этой структуры есть соответствующий метод аггрегирования, как например для CounterState:

```go
// Aggregate counterState with given counterState
//
// Pre-cond: given tupler to aggregate with
//
// Post-cond: if given TODO
func (c CounterState) Aggregate(with tuples.Tupler) (tuples.Tupler, error) {
	val, ok := c.GetField("value")
	if !ok || val == nil {
		return nil, errors.New("value must exists for writing metric")
	}

	if with == nil {
		return c, nil
	}

	val, ok = with.GetField("value")
	if !ok || val == nil {
		return c, nil
	}

	val64 := val.(*int64)
	newVal := *(c.Value) + *(val64)
	c.Value = &newVal
	return c, nil
}
```
Обозначим эти структуры данных Б, как альтернативные А. 

И вот теперь подходим к теме, как приемы "журналирования" могли бы мне делать этот код с мЕньшей головной болью. 
Поскольку что структура А, что структура Б -- агрегируемые, для структуры А можно фиксировать изменения в журнале по типу:

```json
receive {"name": "poll", "type":"counter", "delta":1}
select {"name": "poll", "type":"counter", "delta":3}
aggregate 
{"name": "poll", "type":"counter", "delta":1}
{"name": "poll", "type":"counter", "delta":3}
result {"name": "poll", "type":"counter", "delta":4}
```

Обозначим это как снимок С.
И вот теперь к сути дела. Если я захочу изменить структуру с А на Б, при этом, чтобы А и Б были free objects (по отношению друг к другу),
то я могу либо написать код, также прожурналировать действия для Б, записать в другой лог и сравнить журнал А с журналом Б, либо, как мне кажется более сильный,
но более сложный прием:
1) Написать код для АТД Б
2) Написать код машины времени, которая берем логи из снимка С.
3) Преобразует записи в значения типа Б (unmarshal, например)
4) Я не зря написать тип операции перед логами. Для того, чтобы воспроизвести работу операций на значениями типа А, можно воспроизвести операции на типами Б и с
помощью assert сравнить результат значение. Или же можно пойти в другом порядка, отката -- мы пишем обратную функцию для aggregate, 
и в обратном порядке воспроизводим операции над значениями типа Б.

Как это поможет выстраивать более гибкий код? Я вижу следующим образом.
Допустим нам дано множество вхожных данных Х. Для Х может быть множество абстракций (точнее, отображений), как эти Х можно отражать во что-то ( в бизнес логику).
Гибкость кода заключается, как мне теперь кажется, не просто в передаче различных объектов одному интерфейсу 
(на примере рассматриваем разные структуры из разных сервисов), а в заменене АТД и способах их взаимодействия таким образом,
чтобы исходная версия и новая были идентичны логическому потоку вычислений (но не в коде!!!).

Чтобы мы могли преобразовать одну композицию АТД и их взаимодействия в другую, и так, чтобы они были равны (пусть будут равны по логике), мы должны на что-то опираться.
И вот тут в дело вступает журналирование и отладка, на основе которых мы можем ориентироваться.

Если выразить чисто математически, то я вижу это следующим образом. 
Допустим, у нас имеется график функции для двухмерного пространства. График -- это результат нашего журнала операций.
Как известно, существуют множество функций и их комбинации, которые смогут создать идентичный график.
И вот задача программиста, как мне видется, находить такие комбинации функций "АТД и их взамосвязи", чтобы их взаимодзаменять.

Ну вот как-то так :) 

Пример 2.

Пример 3.
