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

# Пример 4



```go
```

# Пример 5

```go
```

# Вывод
