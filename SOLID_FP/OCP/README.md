# OCP с т.зр. ФП

# Пример 1

Я любитель создать в Go функциональный пакет с функциями высшего порядка. Они редко изменяются, в основном дополняются, 
применимы на протяжении всего проекта, да и читабельность кода повышают. Не припомню случая, чтобы мне приходилось
изменять какую-либо функция:

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

Последнее время стараюсь уделять время на чтения кода на Go на Github'е, и мне удивительно странно, почему так редко используются функции высшего порядка, ведь
для добавления функциональности, нам достаточно написать лямбда-функцию, а не создавать новый пакет/изменять текущую реализация.

# Пример 2

Помимо функций высшего порядка, у меня имеется привычка писать пакеты в стиле std плюсов. Возможно я ошибаюсь и не имею достаточного
опыта работы с std C++, но тендеция std там такова, что редко изменяется какой-либо компонент/алгоритм, а вносится новая его версия и со 
временем старый компонент/алгоритм удаляется из нового стандарта.
Основной мой мотив таков следования данной тендеции таков, что это
1) перенимать опыт у разработчиков плюсов по поддержанию проекта (в данном случае языка С++)
2) перенимать стиль написания компонентов

Самый простой пример, но очень понятный:
```go

package std

type Iterator[T any] interface {
	Next() bool
	Curr() T
}

type Linked[T any] interface {
	PushBack(item T)
	PopFront() (T, bool)
	Iterator() Iterator[T]
}
```

Можно заявить, мол "это же паттерн итератор!". Не знаю, что появилось раньше: итераторы в С++ или паттерн "итератор", но реализация
концепции итераторов в плюсах заложили в моей голове мысли, что при изменении способа прохода по контейнеру, не нужно менять его внутреннюю реализацию,
а достаточно создать, подчеркиваю, дополнить, итератор (т.е. не нужно и итератор существующий менять).
 

# Пример 3

Это скорее будет очевидная вещь, но DI служит хорошей техников для поддержания OCP. 
```go
// Storer, tuple.Tupler, loggo.Logger -- интерфейсы
func Read(s Storer, state tuples.Tupler, l loggo.Logger) (tuples.TupleList, error) {
	l.InfoWrite(state)
	states, err := s.Read(state)
	l.InfoWrite(states, err)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```

Да, первоначально DI про ответвстенность создания объекта, но я вижу это тут аналогии с лямбда-функциями: 
у нас есть параметр с определенным типом, и, как мы можем передать в качестве аргумента лямбда-функцию, которая удовлетворяет типа-параметру функции,
также работает и с интерфейсами. То есть, если компонент, будь-то интерфейс или функция, имеет тот же тип, что и параметр, то это хороший способ поддерживать OCP.

Хороший будет контраргумент моего примера, если спросить "что если интерфейс, например, loggo.Logger изменится -- не нужно ли нам переписывать текущую функцию, тем самым
нарушая ОCP?". Я бы ответил, что
1) 100% ОСР не всегда получится, но есть более крутой ответ
2) Можно вспомнить прошлое занятие HW, про SOLID or SOLD, и сделать лямбда-интерфейс (название ниоткуда не взял, сам придумал):
```go
func Read(s Storer, state tuples.Tupler, l interface{
    LogWrite(x ...any)
}) (tuples.TupleList, error) {
	l.InfoWrite(state)
	states, err := s.Read(state)
	l.InfoWrite(states, err)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
``` 

Да, что в первом случае, что во втором мы будем изменять реализацию, но, в случае заданным типом интерфейса, мы должны либо менять сам тип интерфейс, либо 
дополнять закрытый для изменения пакет новой функций, а вот во втором случае, нам достаточно изменять лямбда-интерфейс, тем самым минимизируя нарушения ОСР!


# Пример 4

Может сложиться впечатление, что у меня получается всегда следовать OCP. Но это не так.

Пересмотрев код проектов, вспомним фсм-ку для поддержания состояния заказов:
Как есть:
```go
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

//Далее методы для взаимодействия с экземпляром BookingFSM
```

Дело в том, что если у нас появится надобность добавить новое состояние или новый переход состояний, то неизбежно придется изменять сам закрытый для изменения пакет bookingfsm.
А на данный момент BookingFSM имеет заранее жестко закодированный предопределенный набор состояний. 
Можно подправить, добавив механизм гибкого добавления состояний и переходов -- у нас будет Builder, который собирает состояния и переходы из BuildPart() и Build() возвращает 
экземпляр BookingFSM. Но....

Стоит ли выносить механизм гибкого добавления состояний и переходов, тем самым сохраняя OCP? Зависит от многих параметров.
Если состояние заказа не менялось, или меняется редко, то изменять существующий компонент, нарушая OCP, видется мне как здравый смысл. Создавая Builder'ы или иной механизм, для 
гибкого формирования состояний в данном случае кажется как "из пушки по воробьям".

А вот если состояние заказа изменяется часто, или для типа клиента заказ может иметь разные состояния, то тогда вариант с Builder'ом выглядит очень хорошим:

```go
type BookingFSMBuilder interface {
    BuildState(state.State) error
    BuildTransition(from state.State, to state.State) error
    Build() state.State
}
```

И мы просто добавляем билдеры для каждой FSM

# Итого

Мне нравится точка зрения OCP со стороны "черного ящика" -- она прямолинейна и ее легче применять при повседневной разработке, спрашивая себя "нужно ли мне изменять черный ящик, чтобы добавить новую фичу?".
Взгляд с точки зрения ФП лучше демонстрирует OCP, нежели ООП, но справедливости ради стоит сказать, что в ФП сложнее нарушать OCP. В ООП мы работаем с ячейкой (ячейки не в смысле ячейка в stateful модели, а ячейка, как
некоторая АТД, изменяемая со временем сущность), структура которой может измениться, и в таком случае мы столкнемся с диллемой: изменять код для работы с ячейкой, нарушая OCP или создавать новый код для работы с новой
ячейкой, но дублируя функционал?

А нужно ли вообще следовать OCP? Да, так как от вашего компонента зависят иные программные сущности, и, изменяя компонент, вы рискуете сломать систему. Нужно ли следовать на 100%? Как уже было сказано ранее,
нужно здраво оценивать ситуация, возможные решения и их последствия и затем решать, применять или нет. Это касается не только OCP, но все же.
