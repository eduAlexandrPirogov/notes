# Dependency inversion principle (DIP)

Смотря на DIP с такой т.зр., я могу продемонстрировать две тенденции, которые интуитивно удовлетворяют DIP:

# Пример 1

Первая тенденция заключается в использовании функций высшего порядка:

```go
package function

// Helper function-values for applying
type onSuccessFunction func()

func CompareStringsDo(check string, defaulValue string, actionOnTrue onSuccessFunction) {
	if check != defaulValue {
		actionOnTrue()
	}
}

func CompareStringsDoOthewise(check string, defaulValue string, actionOnTrue onSuccessFunction, actionOnFalse onSuccessFunction) {
	if check != defaulValue {
		actionOnTrue()
		return
	}

	actionOnFalse()
}

func CompareBoolssDo(check bool, defaulValue bool, actionOnTrue onSuccessFunction) {
	if check != defaulValue {
		actionOnTrue()
	}
}

func CompareIntsDo(check int, with int, actionOnTrue onSuccessFunction) {
	if check != with {
		actionOnTrue()
	}
}
```

Таким образом, мой пакет не будет зависеть от какой-либо реализации, плюс никакая реализация не будет влиять на внутренее
состояние пакета (его просто нет :).
Можно сделать функцию с дженериками для сравниваемых типов и дженериками для функций-аргументов, но, как неоднократно повторял,
я считаю подобные оптимизации выполнять тогда, когда это действительно на порядок улучшит кодовую базу.

# Пример 2

Вторая тенденция заключается в передаче интерфейсов (не в смысле ФП) в качестве аргументов:

```go
func ChooseCarBooking(b entity.Booking, s db.BookingStorer) error {
	return s.CarChoosed(b)
}
```

А если вспомнить занятие по "SOLID или SOLD", то в Go вообще красота получается:

```go
func ChooseCarBooking(b entity.Booking, s interface{
   CarChoosed(entity.Booking)
}) error {
   return s.CarChoosed(b)
}
```

за счет того, что, лямбда-интерфейс более полиморфен (опять же вспоминаем занятие про правильный полиморфизжм), нежели обычный,
данная функция ни-коим образом не зависит от деталей реализации, это прямой аналог функциям высшего порядка.

# Пример 3

В данном примере я волен передавать любые экземпляры, удовлетворяющие интерфейсу Metricable. Метод Read() считывает
используемые ресурсы. Таким образом, при  потребности добавить новые экземпляры, которые будут считывать новый ресурс, мне 
достаточно реализовать этот метод, а сбор метрик будет происходить вне зависимости от реализации новой метрики.
По сути получилось внедрение функционального интерфейса.

```go
// Metricalbes entities should update own metrics by Read() errpr method
type Metricable interface {
	Read() error
}

// MetricsTracker Holds all measures in slice
type MetricsTracker struct {
	Metrics []metrics.Metricable
}

// InvokeTrackers Pre-cond:
//
// Post-cond: requests measures to update own metrics.
// returns 0 if success otherwise we can return error_code
func (g *MetricsTracker) InvokeTrackers() error {
	// #TODO add here grp and errgrp
	var grp sync.WaitGroup
	grp.Add(len(g.Metrics))
	for _, tracker := range g.Metrics {
		go func(tracker metrics.Metricable) {
			defer grp.Done()
			err := tracker.Read()
			if err != nil {
				log.Println(err)
			}
		}(tracker)
	}
	grp.Wait()
	return nil
}
```

Но лучше добавление экземпляров Metricable делать через явный метод.

```go
func (g *MetricsTracker) Add(m metrics.Metricable) error {
    ...
}
```

# Пример 4

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

type StoreWriter interface {
	Write(tuple tuples.TupleList) (tuples.TupleList, error)
}


type StoreReader interface {
	Read(cond tuples.Tupler) (tuples.TupleList, error)
}


// Write writes tupler to DB and return written state
//
// Pre-cond: given tuples to Write in given Storer
//
// Post-cond: if was written successfully returns NewTuple state and error nil
// Otherwise returns emtyTuple and error
func Write(s StoreWriter, states tuples.TupleList) (tuples.TupleList, error) {
	newStates, err := s.Write(states)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return newStates, nil
}

func StoreReader(s Storer, state tuples.Tupler) (tuples.TupleList, error) {
	states, err := s.Read(state)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```

Тут функции просто принимают в качестве аргумента интерфейсы. Вообще интерфейсы так хорошо ложатся в парадигму ФП, что, как мне кажется, 
такой подход используется уже в других функциональных языках. И я прекрасно понимаю, что это не лямбда-функции, это нечто более интересное, 
с чем я пока не сталкивался, но кажется невероятно мощным интерфейсом.

# Пример 5

Я очень люблю реализовывать в разных язык подход std в С++, где DIP реализован с функциональной т.зр. как самый лучший и наглядный пример.

```go
// std hold interfaces for all data structures that used in this project

package std

// Iterator incapsulate movement inside data structure elements in one way
type Iterator[T any] interface {
	// Next tells is there are elements to iterate
	//
	// if there are elements to iterate then return true
	// otherwise returnes false
	Next() bool

	// Curr returnes current element of iterator
	Curr() T
}

type Linked[T any] interface {
	// Add adds given elem at the tail
	//
	// Pre-cond: given element to add
	//
	// Post-cond: list's tail now is equal to given element
	PushBack(item T)

	// PopFront removes head of the list and returns item
	//
	// Pre-cond:
	//
	// Post-cond: if list isn't empty then returns value of head and true
	// and moves head to the next elem
	// Otherwise returns default value and false
	PopFront() (T, bool)

	// Iterator creates new instance of Iterator to inspect elements
	//
	// Pre-cond:
	//
	// Post-cond: returned instance of iterator
	Iterator() Iterator[T]
}

// AsSlice put all data structure's elemes into slice
//
// Pre-cond: given data structure's iterator
//
// Post-cond: returned slice of data structure's elements
func AsSlice[T any](i Iterator[T]) []T {
	res := make([]T, 0)
	for i.Next() {
		res = append(res, i.Curr())
	}
	return res
}
```

Подобные дают, как мне кажется, хорошую практику создавать не просто код, а формальную эко-систему. Ну например:

```go
package kernel

import (
	"autopark-service/internal/autopark/car"
	"autopark-service/internal/db"
	"autopark-service/internal/std"
)

// CreateBrand stores brand in storage
//
// Pre-cond: given brand entity and entity implementing BrandStorer interafce
//
// Post-cond: executes command to store given brand in given BrandStorer
// returns nil if command executed successfully otherwise error
func CreateBrand(b car.Brand, s db.BrandStorer) error {
	return s.StoreBrand(b)
}

// ReadBrands return list from storage
//
// Pre-cond: given brand entity and entity implementing BrandStorer interafce
//
// Post-cond: executes command to read brands by given brand-pattern from given BrandStorer
// returns nil if command executed successfully otherwise error
func ReadBrands(pattern car.Brand, s db.BrandStorer) (std.Linked[car.Brand], error) {
	return s.ReadBrands(pattern)
}
```

В моем случае ядро приложения будет работать исключительно с типами данных, а не конкретными реализациями (разве что не считать примитивные типы данных).
Kernel работает с типом BrandStorer и возвращает тип Linked. С помощью Iterator'a вы проходите через любую реализацию Linked. 
С помощью дженериков, функциональных интерфейсов код становится невероятно гибким и менее связным. Ну не красота ли?)

# Вывод

DIP с т.зр. ФП интуитивно пишется на ходу из-за разницы между парадигмами ФП и ООП в составляющих их элементах: в ФП базовый компонент есть функция, у которой,
имеется лишь тип и реализация, когда как у ООП есть класс/структура, ее экземпляр и интерфейс. В ФП меньше подобных "разделений абстракций". 
Я ещё раз повторюсь, что не фанат пихать во всех методы, функции и конструкторы интерфейсы, просто потому что DIP. Как мне кажется, DIP должен тесно идти вместе 
с темой функциональных интерфейсов, лишь потому что они формируют экосистему/алгебру. И когда система формирует алгебру, ее проще разрабатывать.

С ФП это просто, потому что там есть функции: вы их можете принимать, возвращать и исполнять, т.е. работаете в замкнутой экосистеме. 
В ООП, если бездумно использовать интерфейсЫ, то получится не экосистема, а просто менее связный код, да и то с вопросами. 

Вероятнее всего моя точка зрения неверна или неподходящая под ООП, но DIP должен быть рассмотрен совместно с ISP и они должны формировать целую алгебру.
