# ISP с т.зр. ФП

C давних времен, когда писал на Java и PHP, где иметь несколько методов в интерфейсе считалось нормой,
я "заразился" этим до того, как разрабатывал в ФП. Из-за этого в Go частенько проявляются большие интерфейсы.
По поводу естественного совместного использования ISP и SRP будет в выводе.
# Пример 1

Частый у меня плохой пример с интерфейсами, когда работаю с БД и делаю интерфейс для CRUD над сущностью:

```go

type BookingStorer interface {
	// Approved executes QueryBookingApproved
	//
	// Pre-cond: given booking with set ID to approve
	//
	// Post-cond: query was executed. If successfull, returns nil. Otherwise returns error

	Approved(entity.Booking) error
	// Canceled executes QueryBookingCanceled
	//
	// Pre-cond: given booking with set ID to cancel
	//
	// Post-cond: query was executed. If successfull, returns nil. Otherwise returns error
	Canceled(entity.Booking) error

	// CarChoosed executes QueryBookingCarChoosed
	//
	// Pre-cond: given booking with set ID to ChooseCAr and CarID
	//
	// Post-cond: query was executed. If successfull, returns nil. Otherwise returns error

	CarChoosed(entity.Booking) error
	// Created executes QueryBookingCreate
	//
	// Pre-cond: given booking with set userID
	//
	// Post-cond: query was executed. If successfull, returns id of newly created record.
	// Otherwise returns -1 and error

	Created(entity.Booking) (int, error)
	// Finish executes QueryBookingFinish
	//
	// Pre-cond: given booking with set userID and set ID
	//
	// Post-cond: query was executed. If successfull returns nil.
	// Otherwise returns error
	Finish(entity.Booking) error
}
```

Из-за этого интерфейсы получаются широкими и их редко где можно переиспользовать. Да, в контексте взаимодействия с БД этот
подход можно оправдать, но все же лучше сделать несколько небольших  интерфейсов.

```go
type BookingApprover interface {
  // Approved executes QueryBookingApproved
	//
	// Pre-cond: given booking with set ID to approve
	//
	// Post-cond: query was executed. If successfull, returns nil. Otherwise returns error

	Approved(entity.Booking) error
}


type BookingCreater interface {
	// Created executes QueryBookingCreate
	//
	// Pre-cond: given booking with set userID
	//
	// Post-cond: query was executed. If successfull, returns id of newly created record.
	// Otherwise returns -1 and error
	Created(entity.Booking) (int, error)
}  

...
```

Почему так будет лучше? Во-первых, в первом варианте я создаю жесткую зависимость к БД (то есть только одна БД может работать при  таком раборе методов).
Во-вторых, это не практично, так как БД, обычно меняются, а если и меняются, то миграции с одной БД на другую происходит намного сложнее,,
Нежели реализацией интерфейса.


# Пример 2 

А вот это как продолжение к первому примеру. То есть было:

```go
// Interface for btable
type BookingTabler interface {
	Approve(entity.Booking) error
	Cancel(entity.Booking) error
	Choose(entity.Booking) error
	Create(entity.Booking) error
	ExistsInTable(entity.Booking) bool
}
```

А можно ничего не делать и просто взять новое готовое решение из первого примера:

```go
type BookingApprover interface {
  // Approved executes QueryBookingApproved
	//
	// Pre-cond: given booking with set ID to approve
	//
	// Post-cond: query was executed. If successfull, returns nil. Otherwise returns error

	Approved(entity.Booking) error
}


type BookingCreater interface {
	// Created executes QueryBookingCreate
	//
	// Pre-cond: given booking with set userID
	//
	// Post-cond: query was executed. If successfull, returns id of newly created record.
	// Otherwise returns -1 and error
	Created(entity.Booking) (int, error)
}  

Забавно, но, казалось, что изначально это два разных, как мне казалось по смыслу интерфейса, но, декомпозироваав и обобщив, их легко повторного использовать!


# Пример 3

Последний пример неудачно использования интерфейса, который можно разделить на более мелкие.
Было:
```go
// Metricalbes entities should update own metrics by Read() errpr method
type Metricable interface {
	Read() error
	String() string
	AsMap() map[string]interface{}
}
```

Стало:
```go
// Metricalbes entities should update own metrics by Read() errpr method
type Metricable interface {
	Read() error
}

type ToTupler interface {
	AsMap() map[string]interface{}
}
```

Первый пример неудачный из-за того, что клиенту, использующему метода Read() не всегда потребуется AsMap(). 


# Вывод
Забавно, что с помощью этого материала я понял суть "Пишем правильный полиморфный код". Я думал над тем, чтобы писать маленькие интерфейсы, но после Java и их функциональных интерфейсов для лямбда-функций, сложилось неверное
впечатление об интерфейсах. Да и без interface dispatch, через кучу явно дописанных implements I1, I2,.., выглядит в ООП языках не очень, но это можно решить через дженерики, с прошлого занятия.
Единственное, что я не понимаю, так это темы касательно SPR и ISP, поскольку она мне кажется достаточно очевидной, если мы будем следовать ISP ровно в таком виде. 
