# О циклах по умному

Сначала приведу примеры из одного проекта, а далее будет вывод.

Все примеры использовали библиотеку для FP Go: https://github.com/repeale/fp-go?utm_campaign=awesomego&utm_medium=referral&utm_source=awesomego#map

===========================================================================================================================================================================================
# Пример 1
===========================================================================================================================================================================================
gRPC. Добавление объект в поток. Как разминочное, можно показать следующий простой пример:
Было:
```go
func (r RPCClient) buildAndSendCounters(m metrics.Metricable) {
	counters := buildMessageCounters("counter", m.AsMap())
	for _, c := range counters {
		if err := r.AddInStreamCounter(&c, r.counterStream); err != nil {
			log.Println(err)
		}
	}
}
```

Тут есть как и вложенный if, так и хотелось бы сделать этот простой кусок кода более читаемый. Мы поступим следующим образом:
1) Устраним if путем изминения логики: мы будем возвращать слайс ошибок, который единым образом можем обработать (вывести в термина или в файл)
2) Заменим for на reduce, который и будет возвращать ошибки:

Стало:
```go
func buildAndSendCounters(r RPCClient, m metrics.Metricable) []error {
	counters := buildMessageCounters("counter", m.AsMap())
	fp.Map(func(c Counter) error { return r.AddInStreamCounter(&c, r.counterStream) })(counters)
}
```

===========================================================================================================================================================================================
# Пример 2
===========================================================================================================================================================================================
Было:
```go
func buildMessageCounters(typ string, counters map[string]interface{}) []Counter {
	key := agent.ClientCfg.Hash
	res := make([]Counter, 0)
	for k, v := range counters {
		val := v.(float64)
		del := int64(val)
		toMarsal := Counter{
			Id:    k,
			Type:  typ,
			Delta: del,
			Hash:  crypt.Hash(fmt.Sprintf("%s:counter:%d", k, del), key),
		}
		res = append(res, toMarsal)
	}
	return res
}
```
Данный кусок кода имеет следующие недостатки:
1) нарушение SRP: тут и создание объекта, и хэширование и приведение к типу и собрать все в контейнер.
2) Интерфейс функции очень неудачен
3) ну и цикл for


Что мы сделаем:
1) Изменим интерфейс метода
2) Вынесем создание объекта в отдельную функцию
3) Замени for на Reduce
   
Стало:
```go

// Лямбда-функци можно было изменить на однострочную, но я придерживаюсь принципа "1 строчка кода -- одно действие".
func buildMessageCounters(counters []Counter) []Counter {
	res := fp.Reduce(func(acc []Counter, c Counter)[]Counter { 
		c := buildCounter((c.Name, c.Type, c.Val)) 
		return append(acc, c)
		}, []Counter{})(counters)
	return res
}

// Досконально изменил лоигку для этой части программы
func buildCounter(name string, typ string, val int64) Counter {
	key := agent.ClientCfg.Hash
	return Counter{
		Name:  name,
		Type:  typ,
		Val:   val,
		Hash:  crypt.Hash(fmt.Sprintf("%s:counter:%d", name, val), key),
	}
}
```

Если хочется писать в одну строчку подобные команды, то можно привести следующий пример:

===========================================================================================================================================================================================
# Пример 3
===========================================================================================================================================================================================
Было:
```go
func ConvertToMetrics(m []Metrics) (tuples.TupleList, error) {
	res := tuples.TupleList{}
	for _, metric := range m {
		tupl := metric.ToTuple()
		res = res.Add(tupl)
	}
	return res, nil
}

```

Стало:

```go
func ConvertToMetrics(m []Metrics) tuples.TupleList {
	return fp.Reduce(
		func(acc tuples.TupleList, metric Metrics) tuples.TupleList {
			return acc.Add(metric.ToTuple())
		}, tuples.TupleList{})(m)
}
```

TupleList я написал в функциональном стиле -- каждый раз возвращается новый экземпляр списка (сделал " в лоб", что не очень эффективно, но сделал для демонстрационных намерений).
Таким образом у нас замыкающих тип данных, который всегда возвращает TupleList при insert/delete/update и мы можем в одну строчку создавать конвейер операций, и
 в купе с reduce выглядит совсем замечательно :)

===========================================================================================================================================================================================
# Пример 4
===========================================================================================================================================================================================
Из материала GointNative очень воодушевило группировать for по смыслу:
Было:

```go
/ restore writes restored metric to DB
//
// Pre-cond: given slice of slice of bytes/ marshaled metrics
//
// Post-cond: unmarshal metric and writes it to DB
// If metric has type counter DB will contain last value of counter
func (d *DB) restore(bytes [][]byte) {
        // Read an unmarshal bytes into metrics and convert metrics to tuple
	for _, item := range bytes {

		var metric metrics.Metrics
		log.Printf("Unmarshaling %s", string(item))
		if err := json.Unmarshal(item, &metric); err != nil {
			log.Printf("err while unmarshal %v", err)
			continue
		}
		tuple := metric.ToTuple()
		log.Printf("Restoring %v", tuple)
		d.Journaler.Restored[metric.ID] = tuple
	}
        // Restore metrics into storage
	for _, tuple := range d.Journaler.Restored {
		//this increased handle time x2
		// TODO refactor
		res, err := kernel.Read(d.Storage, tuple)
		if !res.Next() || err != nil {
			kernel.Write(d.Storage, tuples.TupleList{}.Add(tuple))
		}
	}
}

```
Один цикл выполняет два действия, которые тяжело отследить. По мотивам лекции с С++, изменим код:

Стало:
(логирование убрал для демонстрационных целей, чтобы акцентировать внимание на изменение raw loops).

Пересмотрев логику кода, я разделил на 3 логических "блока":
1) Десериализация
2) Конвертация типа
3) Запись
```go
func (d *DB) restore(bytes [][]byte) {
	// deserialize written to file metrics
	unmarshaled := fp.Reduce(func(m []metrics.Metrics, item []byte) []metrics.Metrics {
		return append(m, unmarshalMetrics(item)...)
	}, []metrics.Metrics{})(bytes)

	// Write to restore section
	fp.Map(func(metric metrics.Metrics) error {
		tuple := metric.ToTuple()
		d.Journaler.Restored[metric.ID] = tuple
		return nil
	})(unmarshaled)

	// Write to storage
	fp.Map(func(tuple tuples.Tupler) error {
		res, err := kernel.Read(d.Storage, tuple)
		if !res.Next() || err != nil {
			kernel.Write(d.Storage, tuples.TupleList{}.Add(tuple))
		}
		return nil
	})
}

// Не стесняемся выносить участки кода в отдельные функции, чтобы можно было писать
// STL-подобные алгоритмы в одну строчку
func unmarshalMetrics(bytes []byte) []metrics.Metrics {
	var metric metrics.Metrics
	err := json.Unmarshal(bytes, &metric)
	if err != nil {
		return []metrics.Metrics{}
	}
	return []metrics.Metrics{metric}
}

```
С одной стороны, мы пришли от 2*N временной сложности к 3*N. Но, ощутимую потерю проивзодительности мы получим лишь при значения N > 1 000 000 000,
так что такой незначительное ухудшение производительности вполне допустимо в угоду читаемости кода.

===========================================================================================================================================================================================
# Пример 5
===========================================================================================================================================================================================
В Go с его идиоматикой для for/ if-else/ switch быстро создает код в котором много отступов.  Из-за чего получается следующее:
Было:
```go
func (d *DefaultHandler) verifyHash(metrcs []metrics.Metrics) error {
	var err error = nil
	for _, metric := range metrcs {
		switch metric.MType {
		case "counter":
			err = d.verifyCounterHash(metric)
		case "gauge":
			err = d.verifyGaugeHash(metric)
		default:
			err = errors.New("not implemeneted")
		}
	}
	return err
}
```

Я для себя выработал следующий паттерн:
Стало:
```go
// Выносим действия в мапу, тем самым устраняя switch
// методы DefaultHandler делаем функциями, которые принимают DefaultHandler
var action map[string]func(*DefaultHandler, metrics.Metrics) error = map[string]func(*DefaultHandler, metrics.Metrics) error{
	"counter": verifyCounterHash,
	"gauge":   verifyGaugeHash,
}

// Заменяем цикл for на Map + наш слварь с действиями
func (d *DefaultHandler) verifyHash(metrcs []metrics.Metrics) []error {
	return fp.Map(func(m metrics.Metrics) error {
		return action[m.MType](d, m)
	})(metrcs)
}
```

Вместо 2-ух кострукций мы теперь используем одну, которая легко расширяется (просто добавив ключ-значение в мапу!).
Читается легко, расширяется еще легче!

===========================================================================================================================================================================================
Итог:
При

