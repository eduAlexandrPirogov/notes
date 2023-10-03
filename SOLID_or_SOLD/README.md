# SOLID или SOLD?

Поскольку Go мой основной язык по количеству кода, на котором я пишу, то можно было бы сказать, что реализовать следующее:

```
class GameEngine<T> where T : IFindByName, IFindById {
private readonly T Team;
```

в Go нельзя, т.к. пока ещё не поддерживает объединение интерфейсов с дженериками. 
Источник: https://github.com/golang/go/issues/49054#issuecomment-946337356

Но тем интереснее реализовать некоторое подобие объединения интерфейсов. Я пытался применить разного рода приемы и пришел к тому, что будет далее.
В целом повествование будет таковым: проблема больших интерфейсов, предложенный способ решения, и то, что у меня получилось по итогу.

Интерфейсы, которые владеют более чем два метода, меня оттаргивают. Одни из самых важных, как по мне причин:
1) Они не добавляют гибкости, а наоборот приводят к засорению кодовой базы. Например, имеется интерфейс I с методами m1, m2, ..., mn. 
Нередко подобные интерфейсы появляются в различных методах/функциях в качестве параметра. Кажется, что теперь любой экземпляр, реализующий
интерфейс, теперь может передаваться в качестве параметра. Но, при расширении интерфейса, придется расширять экземпляр класса, тем самым
классы становятся "широкими" в плане методов. Зачастую это приводит также к нарушению SRP.
2) Тестировать невозможно из-за создания моков и стабов.

Из-за подобного в проектах, где я работаю с БД, нередка такая картина:

```go
// Комментарии убраны для читабельности
type BookingStorer interface {
	Approved(entity.Booking) error
	Canceled(entity.Booking) error
	CarChoosed(entity.Booking) error
	Created(entity.Booking) (int, error)
	Finish(entity.Booking) error
}
```

При тестировании, замучался писать заглушки для подобных интерфейсов.
К тому же, может придти задача, чтобы добавить в BookingStorer метод CarChoosedWithCar(car.Car) error. Интерфейс станет еще жирнее, а тесты придется дополнять.

Хорошо, проблема выделена, последствия понятны, но что с этим можно сделать конкретно в Go?
Как было сказано ранее, к сожалению в Go пока что отсутствует возможность объединения интерфейсах через дженерики. И все же есть способ имитровать подобное поведение.
Для начала приведу часть компонентов, которые используют данные интерфейс:

```go
package kernel

func ChooseCarBooking(b entity.Booking, s db.BookingStorer) error {
	return s.CarChoosed(b)
}
```

Начнем с простого. При тестировании данной функции, мне не хочется имплементировать все методы кроме CarChoosed(entity.Booking). Можно сделать вот так:

```go
package kernel

func ChoosrCarBooking(b entity.Booking, s interface{
   CarChoosed(entity.Booking)
}) error {
   return s.CarChoosed(b)
}

Мы просто заменили s db.BookingStorer на interface{ CarChoosed(entity.Booking)}. Теперь, мне не требуется реализовывать весь клад методов для тестирования, да и выглядит опрятно.
Объединение довольно просто имитировать:
Было:
```go
// EventSender -- интерфейс
func RegisterManager(m client.Manager, e events.EventSender) (client.Manager, error) {
	err :=  e.RegisterManager(m)
 	e.StoreError(err)
	return err
}
```

Стало:
```go
// EventSender -- интерфейс
func RegisterManager(m client.Manager, e interface{
    RegisterManager(client.manager) error
    StoreError(error)
}) (client.Manager, error) {
	err :=  e.RegisterManager(m)
 	e.StoreError(err)
	return err
}
```

Насколько предложенный мною метод имитации множествонного наследования/объединения интерфейсов хорош? На самом деле, областей применения у него не так много. Рассмотрим следующий пример:

Было:
```go
func RegisterManager(m client.Manager, c client.ManagerHandler, s db.ActionStorer) (client.Manager, error) {
	err :=  c.RegisterManager(m)
 	s.StoreError(err)
	return err
}
```

```go

// Interface for btable
type BookingTabler interface {
	Approve(entity.Booking) error
	Cancel(entity.Booking) error
	Choose(entity.Booking) error
	Create(entity.Booking) error
	ExistsInTable(entity.Booking) bool
}

type watcher struct {
	// interface
	s     db.BookingStorer
	// interface
	table BookingTabler
}

func Approve(b entity.Booking, w *watcher) error {
	if !w.table.ExistsInTable(b) {
		return fmt.Errorf("cant approve not existing booking")
	}

	tableErr := w.table.Approve(b)
	if tableErr != nil {
		return tableErr
	}

	dbErr := w.s.Approved(b)
	if dbErr != nil {
		return dbErr
	}

	return nil
}
```

Имеется watcher интерфейс, и, для его тестирования, хотелось бы использовать вышеприведенный метод. 
Казалось бы, для этого достаточно вложенного интерфейса в параметрах Approve, что-то вроде следующего:

```go
func Approve(b entity.Booking, w interface{
	// Nested interface
	table interface{
      		ExistsInTable(entity.Booking)
   	}
	// Nested interface
   	s interface{
    		Approve(entity.Booking)
   	}
}) error {
	if !w.table.ExistsInTable(b) {
		return fmt.Errorf("cant approve not existing booking")
	}

	tableErr := w.table.Approve(b)
	if tableErr != nil {
		return tableErr
	}

	dbErr := w.s.Approved(b)
	if dbErr != nil {
		return dbErr
	}

	return nil
}
```

но в Go так нельзя :)

```go
func someFunc(x interface{
   y interface {Call()} // !INVALID
})() {
   x.y.Call()
}
``` 

Ну и ещё пример кода, где я все-таки использовал подход с интерфейсами:

Было
```go
func Read(s Storer, state tuples.Tupler, l logg.Logger) (tuples.TupleList, error) {
	l.InfoWrite(states)
	states, err := s.Read(state)
	l.InfoWrite(states, err)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```

Стало
```go
func Read(s Storer, state tuples.Tupler, l interface{
    LogWrite(x ...any)
}) (tuples.TupleList, error) {
	l.LogWrite(states)
	states, err := s.Read(state)
	l.LogWrite(states, err)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```

Если потребуется вести лог не только для Info, но и для Warn/Error, то мне не составит труда добавить эти методы в интерфейс.


По итогу.
Сразу скажу, что я никогда не был фанатом IoC и использовать его не приходилось (да и как-то виделось это мне костылем, нежели техникой).
Сидя на Go у меня возникают большие интерфейсы и я не знал как с ними бороться. Сама идея использования дженериков с объединением интерфейсов очень привлекательна, 
но хоть на Go она недоступна в полной мере, я бы все-таки использовал прием с интерфейсами, которые объявлены только в параметрах функций. 
Да, отсутствует возможность создать вложенные интерфейсы внутри объявленных интерфейсов как параметр функции, но тут уже надо пересмотреть дизайн компонента в целом.

Тут может быть хорошее замечание, мол "почему бы не создать просто множество маленьких интерфейсов около функций" -- для чтение данных из БД условный ReaderDB и прочее.
Я замечу, что при создании подобных маленьких интерфейсов, экземпляры, имплементирующие их, будут от этих интерфейсов зависеть, поскольку при их расширении, придется расширять и экземпляры.
А вот при подобном "объединении" интерфейсов, которые функциональны (т.е. имеют лишь один метод, термин взят из Java), мы этим не страдает.

И да, стоит сказать, что зачастую интерфейсы имеют больше двух методов, и условный АТД список мы не будем разбивать на отдельные интерфейсы Pusher, Poper и подобные.
Но, как мне кажется, это уже путаница между парадигмами -- интерфейс для АТД, и интерфейс для некоторой функциональности все-таки разные интерфейсы.
Как в примере с GameEngine, мы в первую очередь ориентировались на интерфейс, который давал некоторую функциональность, в случае с АТД списком, мы ориентируемся на интерфейс взаимодействия с компонентом с состоянием.
