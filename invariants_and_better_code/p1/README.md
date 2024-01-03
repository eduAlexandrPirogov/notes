# Инварианты и качественный код (1)

Поскльку в Go нет assert'ов в смысле С++ (https://golang.org/doc/faq#assertions), то вместо них я написал предикат, который в ложных случаях выдает панику.

# Пример 1

Пример, где создавалась карта заказов. Заказ может быть в одном из нескольких состояний 

```go
const (
	StateCreated    = "created"  // Booking was created
	StateChoosedCar = "choosed"  // Car for created booking was choosed. Can be performed only from StateCreated
	StateApproved   = "approved" // Booking was approved. Can pe performed only from StateChoosedCar
	StateCanceled   = "canceled" // Booking was canceled. Can be performed from StateCreated and StateChoosedCar
	StateFinished   = "finished" // Booking was finished. Can be performed only from StateApproved
)
```

Первая реализация заключалась в использовании структуры заказа Booking, которая хранила переменную текущего состояния.
```go
type Booking struct {
    current string
}
```

При изменения статуса заказа, нередка ситуация, когда нужно было удостовериться в том, что состояние изменено на желаемое, что
порождало множество if-ов. Со временем появились заказы, которые включали себя лишь часть из вышеприведенных состояний. В структуру добавил структуру Set, которая
хранила допустимое состояние для конкретного Booking. Это добавило в код не только проверки на желаемое состояние после изменений, но и 
проверки на то, что измененное состояние входит в множество допустимых состояний конкретного экземпляра Booking.

Все эти заказы хранились в структуре BookingTable, отражающая ИД пользователя и его заказ:
```go
type BookingTable struct {
	mutex sync.RWMutex
	fsms  map[int]*booking.Booking
}
```

И если требовалось, чтобы в карте BookingTable у пользователя изменилось состояние заказа, то это требовало множество проверок if-ов, что в свою очередь 
повышало цикломатическую сложность кода.

## Уберем логику проверки на структуру данных, в которой в принципе невозможно неверное состояние.

Для представления заказа и его состояний лучше всего использовать FSM-ку.
В случае, если потребуется для определенных заказов "особенные" состояний и их переходы, то можно в метод New передать карты fsm.Events.

```go
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
```

Таким образом, нам вообще не потребуется проверять текущее состояние заказа в силу работу (и коректной реализации) конечного автомата.

## Добавим ассерты

Добавлять ассерты в FSM не стоит, поскольку мы подразумеваем, что ее реализация корректна. Мы вернемся к BookingTable и рассмотрим один из методов:

```go
// Approve tries to perfom transformation to StateApproved
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Approve() error {
	err := b.fsm.Event(context.Background(), StateApproved)
	if err != nil {            <------------------------------------------------------------------- изменим положение дел здесь
		log.Debug().Msgf("%v", err)
		return err
	}
	log.Debug.Msgf("Booking with id %d has approved. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}
```

И усилим ассертом проверку err на nil:

```go
// Approve tries to perfom transformation to StateApproved
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Approve() error {
  var err error
	err = b.fsm.Event(context.Background(), StateApproved)
  f.assert(err, nil) <---------------------------------------------------------------------------- заменили проверку на кастомный ассерт
	return err
}
```

Но есть нюанс касательно возврата ошибок и assert.

# Пример 2


# Пример 3
