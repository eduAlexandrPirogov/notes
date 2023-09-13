# 4 внутренних свойства хорошего кода

# По итогу
Прежде чем познакомиться с примерами, я сделаю важное предисловие.

Все проекты я "Трогал" последний раз недели две, поэтому обзор будет более объективным.
# Проект 1. Чтение и аггрегация метрик ОС.

## 1. Хороший код можно (и нужно) понимать модульно.
Что я понимаю под этим: насколько легко, глядя на единицу проекта (пакет, функция, структура/класс, строка), можно перечислить утверждений о нем  а) относительно его самого (внутреннее строение)
б) относительно его влияния на систему(как он вписывается в нее целиком).

Благодаря триплам Хоара, которые изменили мое мышление на "участок кода работает корректно только в том случае, если корректные входные данные". То, что это можно применять также по отношению
к строкам для меня в новинку, но, это логично выглядит. Благодаря материалу "SRP в каждой строчке" это принцип у меня появился и хорошо развит.
Исключения бывают связанны с лямбда-функциями в Go, которые занимают больше одной строки, но мне кажется, тут речь больше об одной инструкции. 

По итогу, поскольку я следую принципу "одна инструкция -- одна ответственность", применяю триплы хоара и стараюсь создавать линейный код, то, осмотря свой проект, он соответствует на 60% этому свойству.

## 2. Хороший код позволяет быстро восстановить замысел программиста.

Как можно измерить данную характеристику? Я бы предложил ответить на следующий вопрос "сколько мнений у вас сложилось при изучении единицы проектв?"
То есть, как многим дизайнам соответствует данный код?

Чем меньше вариантов дизайнов, тем лучше. Рассмотрим пример:

```go
// Trackers are used to collect all types of measures
//
// Trackers request measures to update own inner metrics
package trackers

// New Creates new instance of MetricTrackers
//
// Post-cond: Creates new instance of MetricsTracker
func New() MetricsTracker {
	...
}

type Trackerable interface {
  // InvokeTrackers Pre-cond:
  //
  // Post-cond: requests measures to update own metrics.
  // returns 0 if success otherwise we can return error_code
	InvokeTrackers()
}

// MetricsTracker Holds all measures in slice
type MetricsTracker struct {
	Metrics []metrics.Metricable
}

func (g *MetricsTracker) InvokeTrackers() error {
	for _, tracker := range g.Metrics {
		err := tracker.Read()
		if err != nil {
			return err
		}
	}
	return nil
}
``` 
Я бы выделил неудачное название интерфейса Trackerable, который имеет метод InvokeTrackers().Т.е Trackarable говорит о том, что тип, удовлетворяющий данному интерфейсу явно должен быть
чем-то единственным и не автономным (в плане, базовый кирпичик чего-то более комплексного), когда как InvokeTrackers() говорит о том, что он должен содержать множественные Trackers. 
Тут уже возникает неоднозначность и противоречние. Исправить можно следующим образом:

```
type TrackersInvoker interface {
  // Run invokes trackers to do job
  //
  // Post-cond: requests measures to update own metrics.
  // returns 0 if success otherwise we can return error_code
	Run()
}
```

Мы изменили с Trackerable до TrackersInvoker, тем самым дав понять, что интерфейс подразумвает работу с Trackers (множественное число), а также изменили метод с Invoke() на Run(), тем
самым дав понять, что мы запускаем некоторую работу над этими Trackers.

Насколько мой проект соответствует замыслу? Это одно из моих слабых мест по выбору хорошего имени для единицы программы, и такая ситуация возникает в каждом 4-ом компоненте в среднем.
Т.е. в каждом 4-ом компоненте у меня вонзикает желание переименовать на более подходящее по смысле.
Я бы сказал, что 25-30% единиц проекта могут легко дать понять замысел иному программисту.


## 3. Хороший код выражает замысел разработчика в одном месте и 4. Хороший код надёжен/робастен (несильные изменения требований подразумевают несильное изменение кода).

Что я понимаю под этим: насколько точно я выразил абстракцию в коде и насколько она автономна. Сразу допустим, что абстракция подобрана хорошо.
На Яндекс-Практикуме для сериализации метрик нам дали следующую структуру от которой мне становится не по душе:
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

С одной стороны, если придет некоторое изменение касательно структуры, то нам можно заявить, мол, "все изменения касательно сериализации метрик будут касательно только этой структуры".
Но я не согласен. Тут надо смотреть на первый пункт "понимать код модульно". Все изменения, которые коснуться данной структуре повлекут за собой изменение вне этого кода.
Поскольку через if-else мы проверяем mtype, то, изменив mtype, мы подвергаем изменениям все if-else, следовательно все, что касатется MType размазано по проекту.

Приведу аналогию, которая поможет лучше понять мою мысль. Допустим, мы делаем входные двери, которые работают по отпечатку пальца. 
В случае, если нам потребуется внести небольшое изменение в алгоритм распознавания пальца, нам не нужно ходить по квартирам, чтобы лезть в "биос" двери и "подгонять фичу".
И вот в случае метрик получается, что изменив MTypе, нам нужно изменить код не в едином месте.

Как я решил данную проблему. Во-первых, я задал себе условие, что это легаси-код, который нельзя изменять. Во-вторых, разделил логику для разных типов метрик.
В-третьих, создал модуль, который читает эту кашу Metrics, и выдает "кирпичик".

Получилось следующее: создал интерфейс Tupler, который отражает работус кортежами. Создал факторку, которая в зависимости от MType создает структуру Tuple, удовлетворяющая Tupler.
То есть, GaugeTuple, CounterTupler. Получились следующие плюсы:
1) Все изменение, касающееся изменений Metrics, будет касаться этих Metrics и факторки.
2) Поскольку я создал "кирпичик" алгебру Tuple, то моя остальная система не зависит от Metrics и факторки - она работает с кортежами! Алгебра моей системы будет корректна до тех пор, пока
подаются корректные кортежи!

Мне очень нравится подобным образом рефакторить проекты, посему мои текущие проекты соответствуют данному требованию более чем на 70%.

В данном случае объединил 3-ий пункт и 4-ый т.к. мне кажется, что 4-ый пункт есть прямое следствие 3-его.
3-ий пункт -- данный проект соответствует на 65%
4-ый пункт -- данный проект соответствует на 70%

Почему объеденил 3-ий и 4-ый пункт описано в итого.

# Проект 2. Микросервисный автопарк

## 1. Хороший код можно (и нужно) понимать модульно.

Тут ситуация аналогична с первым пунктом из первого примера.

## 2. Хороший код позволяет быстро восстановить замысел программиста.

Пересмотрев проект с данной точкой зрения, я, опять же, через 3-4-ый файл хотел переименовать пакет/функцию или рефакторнуть часть кода, поскольку понимал, что она либо не отражает моего замысла,
либо отражает, но размыто. Например:

```go
type watcher struct {
	s     db.BookingStorer
	table BookingTabler
}
```

Что за watcher? Почему он хранит db и интерфейс. Они взаимодействуют? Почему бы не применить паттерн "медиатор", дабы инкапсулировать взаимодействие?

А вот примеР, когда мне стало все предельно ясно за 5 секунд:
```go
/ bookingfsm represents FiniteStateMachine for booking.
// Book can be on on the next states: created, choosed, approved ,cancel, finished.
// Each of these states described below.
package bookingfsm

import (
	"booking-service/internal/entity"
	"context"

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
	return nil
}

// Finish tries to perfom transformation to StateFinished.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Finish() error {
	err := b.fsm.Event(context.Background(), StateFinished)
	if err != nil {
		return err
	}
	return nil
}

// Booking returns copy of entity.Booking
func (b *BookingFSM) Booking() entity.Booking {
	return b.Book
}
```

Мне даже на код не надо было смотреть, только на сигнатуры структур, функций и констант.
Я дал оценку 50% своему проекту.

## 3. Хороший код выражает замысел разработчика в одном месте и 4. Хороший код надёжен/робастен (несильные изменения требований подразумевают несильное изменение кода).

Для своего проекта я бы дал следующую оценку:
1) мой код выражает мой замысел в одном месте 50%
2) мой код надежен 50%

Удачный пример - вышеприведенная FSM. Все, что связанно с состоянием заказа и его изменением находится внутри этой FSM-ки. Код надежен и хорошо выражает замысел.

Неудачный пример:
```go

type watcher struct {
	s     db.BookingStorer
	table BookingTabler
}

// singletonWatcher is signleton
var singletonWatcher *watcher
var initOnce sync.Once

// GetInstance returns *watcher
// If watcher was created returns pointer to it
func GetInstance() *watcher {
	initOnce.Do(func() {
		singletonWatcher = new()
	})
	return singletonWatcher
}

// new returns pointer to the singleton of wather
func new() *watcher {
	return &watcher{
		s:     db.GetConnInstance(),
		table: btable.New(),
	}
}

// Approve updates btable's record for given entity.Booking
// and update respective record in database/
//
// Pre-cond: given entity.Booking to approve and watcher.
// Updating entity.Booking must exists in table
//
// Post-cond: if transformation was executed successfully
// the writes updates to database. First it's searching record in btable.
// If transformation was failed then cancel the booking
func Approve(b entity.Booking, w *watcher) error {
	if !w.table.ExistsInTable(b) {
		log.Debug().Msgf("cant approve not existing booking")
		return fmt.Errorf("cant approve not existing booking")
	}

	tableErr := w.table.Approve(b)
	if tableErr != nil {
		log.Debug().Msgf("btable err %v", tableErr)
		return tableErr
	}

	dbErr := w.s.Approved(b)
	if dbErr != nil {
		log.Debug().Msgf("db err %v", dbErr)
		return dbErr
	}

	return nil
}
```

Если честно, это не код, а набор букв. Легко ломается, легко сделать программу ошибочной. Не помню в каком состоянии писал это, но это никуда не годится.
