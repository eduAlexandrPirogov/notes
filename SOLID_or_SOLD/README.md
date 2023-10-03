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
Кажется, что пример простой и искусственный? Хорошо, вот ещё один пример:

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
	s     db.BookingStorer
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

watcher имеет огромные два встроенных интерфейса, которые не прибавляют гибкости и устойчивости, и , замени их на обычные структуры, разницы не было бы.
Можно сделать вот так:

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
	s     db.BookingStorer
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
