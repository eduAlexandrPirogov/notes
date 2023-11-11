# Логика Хоара для программистов

# Пример 1

Начнем с простого. Пакет-аналог std плюсов:
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

Обратите внимание на сигнатуру функции AsSlice. 
Используется данный интерфейс повсеместно, и если нужно получить слайс с помощью AsSlice, то код выглядит так:

```go
//l -- полученный инстанс, удовлетворяющий интерфейсу std.Linked
slice := std.AsSlice(l.Iterator())
```

Да, получить элементы контейнера в виде слайса -- задача простая, но моей целью было сделать так, чтобы функцию AsSlice и все контейнеры std 
нельзя было использовать неправильно! 

В соответствие с пост-условиями получается следующее:
1) Участок AsSlice исполняется лишь тогда, когда передается Iterator
2) Результат будет получен лишь после удовлетворения пунтка 1

Но есть нюанс, который заключается в реализации инстанса Iterator'a. Да, спецификации для интерфейса имеются, но криво написанный итератор может вернуть
пустой слайс непустого контейнера. Таким образом, спецификации для AsSlice можно написать следующим образом:

```go
// AsSlice put all data structure's elemes into slice
//
// Pre-cond: given data structure's iterator
//
// Post-cond: returned data structure representation as slice   <--------
func AsSlice[T any](i Iterator[T]) []T {
	res := make([]T, 0)
	for i.Next() {
		res = append(res, i.Curr())
	}
	return res
}
```

# Пример 2

А вот неудачный пример:

```go
// NewWriter writes data every time when channel got new data
//
// Pre-cond: given file to write data
//
// Post-cond: data written to the file
func (j *Journal) WriteSynch(file *os.File) {

	j.mux.Lock()
	defer j.mux.Unlock()
	defer file.Close()
	writer := bufio.NewWriter(file)
	for {
		if bytes, ok := <-j.Channel; ok {
			bytes = append(bytes, '\n')
			writer.Write(bytes)
			writer.Flush()
		} else {
			break
		}
	}
}
```

Во-первых, несмотря на корректное предусловие, возможна ситуация, когда данные могут быть не записаны в файл: 
поскольку происходит взаимодействие с ОС, там могут быть самые различные примеры.
Также может быть в теории передан некорректный дескриптор файла, что тоже в спецификациях не отражается.

Сложность рассуждения о данном участке кода добавляет тот факт, как используется данная функция:

```go
go j.writeSynch(file)
```
где просто передается дескриптор какого-то файла (проверок до этой строки никаких не было).

Получился совершенно хрупкий участок кода, который ломается при любом дуновении ветра...

# Пример 3

```go
const xRealIP = "X-Real-IP"

// IsContainerSubnetHTTP checks if request's ip satisfy's subnet
//
// Pre-cond: Given pointer to not nil http.Request
//
// Post-cond: returns a boolean which tells if requests satisfy subnet
func IsContainerSubnetHTTP(r *http.Request) bool {
  if r == nil {
     return false
  }
	cameIP := r.Header.Get(xRealIP)
	ip, parseErr := netip.ParseAddr(cameIP)
	subnet, parsePrefixErr := netip.ParsePrefix(server.ServerCfg.Subnet)
	return parseErr == nil && parsePrefixErr == nil && subnet.Contains(ip)
}
```

Простая проверка на принадлежность IP-запроса к подсети. 

Используется в middleware:

```go
func SubnetValidate(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if ok := verifier.IsContainerSubnetHTTP(r); !ok {
			w.WriteHeader(http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

Возможна ли ситуация, когда при некорректных пред-условий истинны пост-условия? Нет, так как уже обработан вариант с nil (о чем сказано в спецификациях),
плюс использование хорошо отлаженных готовых библиотек (проверил на тестах).

# Пример 4

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

Используется везде, где требуется получить значение кортежа по ключу:
```go
func writeGauges(p *postgres, state tuples.Tupler) (tuples.TupleList, error) {
	var val *float64
	var mname, mtype string

	// You can expose/fold brackets for better code reading
	{
		val = tuples.ExtractFloat64Pointer("value", state)                             <----------------------------
		if val == nil {
			return tuples.TupleList{}, errors.New("value must exists while writing")
		}

		mname = tuples.ExtractString("name", state)					<----------------------------	
		mtype = tuples.ExtractString("type", state)					<----------------------------
	}

	rows, err := p.conn.Query(context.Background(), WriteMetric, mtype, mname, *val)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return p.assembleGaugeState(rows)
}
```

Тут сложно найти несостыковки, разве что если передать nil...но проверка на nil имеется до момента Экстракта значения по ключа.
Вопрос в том, стоит ли внести проверки на nil в функции пакета tuples? Скорее да, так как возможна ситуация в теории, что это функции вызовут с nil,
и спецификации не говорит ничего о nil.

# Пример 5

Последний большой пример, связанный с fsm:
```go
// bookingfsm represents FiniteStateMachine for booking.
// Book can be on on the next states: created, choosed, approved ,cancel, finished.
// Each of these states described below.
package bookingfsm

import (
	"booking-service/internal/entity"
	"context"
	"fmt"

	"github.com/looplab/fsm"
	"github.com/rs/zerolog/log"
)

const (
	StateCreated    = "created"  // Booking was created
	StateChoosedCar = "choosed"  // Car for created booking was choosed. Can be performed only from StateCreated
	StateApproved   = "approved" // Booking was approved. Can pe performed only from StateChoosedCar
	StateCanceled   = "canceled" // Booking was canceled. Can be performed from StateCreated and StateChoosedCar
	StateFinished   = "finished" // Booking was finished. Can be performed only from StateApproved
)

// BookingFSM is a wrapper for looplabfsm
type BookingFSM struct {
	Book entity.Booking
	fsm  *fsm.FSM
}

// Created new BooingFSM instance
//
// Pre-cond: given id and userID
//
// Post-cond: instance was created and pointer to it was returned
func New(id, userID int) *BookingFSM {
	return &BookingFSM{
		Book: entity.Booking{
			ID:     id,
			UserID: userID,
		},
		fsm: fsm.NewFSM(
			StateCreated,
			fsm.Events{
				{Name: StateCreated, Src: []string{StateCreated}, Dst: StateCreated},
				{Name: StateChoosedCar, Src: []string{StateCreated}, Dst: StateChoosedCar},
				{Name: StateApproved, Src: []string{StateChoosedCar}, Dst: StateApproved},
				{Name: StateCanceled, Src: []string{StateCreated, StateChoosedCar}, Dst: StateCanceled},
				{Name: StateFinished, Src: []string{StateApproved}, Dst: StateFinished},
			},
			fsm.Callbacks{},
		),
	}
}

// Current return current state of BookingFSM
func (b *BookingFSM) Current() string {
	return b.fsm.Current()
}

// ChooseCar tries to perfom transformation to StateChoosedCar.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) ChooseCar() error {
	err := b.fsm.Event(context.Background(), StateChoosedCar)
	if err != nil {
		log.Debug().Msgf("%v", err)
		return err
	}
	fmt.Printf("Booking with id %d has choosen car. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Approve tries to perfom transformation to StateApproved
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Approve() error {
	err := b.fsm.Event(context.Background(), StateApproved)
	if err != nil {
		log.Debug().Msgf("%v", err)
		return err
	}
	fmt.Printf("Booking with id %d has approved. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Cancel tries to perfom transformation to StateCanceled.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Cancel() error {
	err := b.fsm.Event(context.Background(), StateCanceled)
	if err != nil {
		log.Debug().Msgf("cancel event called %v", err)
		return err
	}
	fmt.Printf("Booking with id %d has canceled. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Finish tries to perfom transformation to StateFinished.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Finish() error {
	err := b.fsm.Event(context.Background(), StateFinished)
	if err != nil {
		return err
	}
	fmt.Printf("Booking with id %d has finished. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Booking returns copy of entity.Booking
func (b *BookingFSM) Booking() entity.Booking {
	return b.Book
}
```

Корректность спецификаций fsm-ки не требует, как мне кажется, моих доказательств)

И где используется, например, метод Choose:

```go
// Choose transform bookingFSM to StateChoosedCar
//
// Pre-cond: given entity.Boking with set userID and CarID
//
// Post-cond: if respective record exists -- trying to make
// transformation and return result of transformation.
// If record doesn't exists then returns error
func (bt *btable) Choose(b entity.Booking) error {
	bt.mutex.RLock()
	defer bt.mutex.RUnlock()
	if val, ok := bt.fsms[b.UserID]; ok {
		bt.fsms[b.UserID].Book = b

		err := val.ChooseCar()
		if err != nil {
			log.Warn().Msgf("%v", err)
			return err
		}
	}
	return nil
}
```

А вот то, как используется эта фсм-ка, показывает, что если мы используется только корректно некоторый компонент системы, отражаем это в спецификациях, то у нас 
В принципе невозможна некорректное выполнение спецификации.

Метод Choose btable говорит, что пытается перевести fsm в следующиее состояние, и поскольку спецификации fsm никак не могут быть нарушены, то спецификации
метода Choose btable также не могут быть нарушены в данном случае, так как они полностью зависят от результата работы fsm.


# Вывод

Хорошее напоминание о том, что пост-условие может быть истинным если и только если истинно пред-условие. В ФП с этим как-то проще, где можно писать чистые функции
и явно разделять чистые функции от нечистых. В ООП ситуация осложняется тем, что обращение к внешнему миру происходит где-то внутри мутабельного состояние, которое
тяжело отследить, да и спецификации могут этого "не донести" до клиента. Посему, было бы неплохо явно разграничивать отвественность за обращение во-внешний мир. Это первое.

Второе, что я заметил, что все пред и постусловия у меня были написаны до этого (вошло в привычку). И, как мне показалось, их стоит писать не везде, а именно,
где происходит взаимодействие с внешним миром. По-крайней мере это должно быть написано в документации. В коде я бы оставил лишь общее назначение куска кода.

И да, когда мне тяжело написать хорошую спецификацию для участка кода, то тут хорошо подойдет fuzz-тестирование :) 
