# OCP с т.зр. ФП

# Пример 1

```go
package function

type onSuccessFunction func()

func CompareStringsDo(check string, with string, actionOnTrue onSuccessFunction) {
	if check == with {
		actionOnTrue()
	}
}

func CompareStringsDoOthewise(check string, with string, actionOnTrue onSuccessFunction, actionOnFalse onSuccessFunction) {
	if check == with {
		actionOnTrue()
		return
	}

	actionOnFalse()
}

func CompareBoolssDo(check bool, with bool, actionOnTrue onSuccessFunction) {
	if check == with {
		actionOnTrue()
	}
}
```

# Пример 2

```go
func Read(s Storer, state tuples.Tupler, l loggo.Logger) (tuples.TupleList, error) {
	l.InfoWrite(states)
	states, err := s.Read(state)
	l.InfoWrite(states, err)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```


# Пример 3

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

```

# Пример 4

Как есть:
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
```

Как можно улучшить:
```go
```

# Итого
