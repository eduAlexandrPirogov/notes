# Извлекаем пользу из сторонних зависимостей

# Пример 1. Драйверы Mongo

На текущей работе идет переход с легаси монго драйвера globalsign на официальный mongo-driver. При исполнения запроса, что методы Find в пакете globalsign, что метод Find в пакете mongo-driver
в качестве запроса принимает тип данных interface{}. Меня сразу смутило отсутствие типизации и насколько переход на новый драйвер не сломает систему.

Разбирался в тот момент я долго, но для краткости стоит понимать, что запросы чтения в монге это по сути json с фильтром и json с проекцией. В фильтре можно устанавливать в качестве искомых значений 
конкретный тип данных: целочисленные, строки, дата и т.п. Но есть и комплексные типы данных, такие как регулярные выражения. Вообще, регулярки в монге можно задавать в сыром виде, то есть в виде строки с
конкретным синтаксисом или в виде объекта RegExp. 
Представление подобных комплексных объектов среди различных драйверов требует особого внимания. Например, как представляется RegExp в globalsign:

```go
ElementRegEx                  byte = 0x0B
```

а как в mongo-driver

```go
Regex            Type = 0x0B  // отдельный тип данных Type
```

Естественно, если мы строим запросы на основе регулярок, то регулярки из globalsign не будут работать в запросе Find из mongo-driver. (но что забавно, обратное неверно, globalsign понимает регулярки из mongo-driver).
Таким образом, однажды часть запросов на чтения на стейдже у нас просто перестала работать, когда перевели метод на новый драйвер)

# Пример 2. Posgres driver

Исследовать зависимости можно по поиску по словам. Например, в Go можно найти все паники, дабы неправильно используемая зависимость не уронила целое приложение. Например, работа с бесконечностями в Postgres:

```go
// If EnableInfinityTs is called with negative >= positive, it will panic.
// Calling EnableInfinityTs after a connection has been established results in
// undefined behavior.  If EnableInfinityTs is called more than once, it will
// panic.
func EnableInfinityTs(negative time.Time, positive time.Time) {
	if infinityTsEnabled {
		panic(infinityTsEnabledAlready)
	}
	if !negative.Before(positive) {
		panic(infinityTsNegativeMustBeSmaller)
	}
	infinityTsEnabled = true
	infinityTsNegative = negative
	infinityTsPositive = positive
}
```

Нет ничего плохого в выдаче паники, но проблема в том, что функция экспортируема и установить некорректное значение достаточно легко, если не вдаваться в документацию.
Можно было бы ещё привести пример с работой чтения и декодирования запросов, когда выполняем select, где происходит очень много копирований из слайса в слайс, но 
это крайне сложно как либо обойти.

# Пример 3. Kafka Writer

Некоторые библиотеки "оборачивают" низкоуровневый функционал в более простой высокоуровневый. Это и хорошо,т.к. в большинстве случаев достаточно простых решений, но стоит иметь ввиду, как они устроены под капотом.
Например библиотека для записи в кафку имеет Writer, который упрощает запись в шину. Сама по себе структура изобилует количеством настраиваемых параметров, но и без них все работает хорошо. Но знать, какие параметры установлены
по-дефолту стоит знать, и как поведение предполагается:

Например, что происходит, если попытка записи была неудачной:
```go
for attempt, maxAttempts := 0, ptw.w.maxAttempts(); attempt < maxAttempts; attempt++ {
		if attempt != 0 {
			stats.retries.observe(1)
			// TODO: should there be a way to asynchronously cancel this
			// operation?
			//
			// * If all goroutines that added message to this batch have stopped
			//   waiting for it, should we abort?
			//
			// * If the writer has been closed? It reduces the durability
			//   guarantees to abort, but may be better to avoid long wait times
			//   on close.
			//
			delay := backoff(attempt, ptw.w.writeBackoffMin(), ptw.w.writeBackoffMax())
			ptw.w.withLogger(func(log Logger) {
				log.Printf("backing off %s writing %d messages to %s (partition: %d)", delay, len(batch.msgs), key.topic, key.partition)
			})
			time.Sleep(delay)
		}

		ptw.w.withLogger(func(log Logger) {
			log.Printf("writing %d messages to %s (partition: %d)", len(batch.msgs), key.topic, key.partition)
		})

		start := time.Now()
		res, err = ptw.w.produce(key, batch)

		stats.writes.observe(1)
		stats.messages.observe(int64(len(batch.msgs)))
		stats.bytes.observe(batch.bytes)
		// stats.writeTime used to report the duration of WriteMessages, but the
		// implementation was broken and reporting values in the nanoseconds
		// range. In kafka-go 0.4, we recylced this value to instead report the
		// duration of produce requests, and changed the stats.waitTime value to
		// report the time that kafka has throttled the requests for.
		stats.writeTime.observe(int64(time.Since(start)))

		if res != nil {
			err = res.Error
			stats.waitTime.observe(int64(res.Throttle))
		}

		if err == nil {
			break
		}

		stats.errors.observe(1)

		ptw.w.withErrorLogger(func(log Logger) {
			log.Printf("error writing messages to %s (partition %d, attempt %d): %s", key.topic, key.partition, attempt, err)
		})

		if !isTemporary(err) && !isTransientNetworkError(err) {
			break
		}
	}

```

Тут даже важен не сам код, сколько комментарии, которые отражают мысли создателей библиотеки, о которых стоит иметь ввиду. 


# По итогу

Я изначально просмотрел реализацию обычных повседневно используемых библиотек, которые не обращаются к внешнему миру. В большинстве своем это просто обертки низкоуровневых операций, которые предоставляют удобные интерфейс использования.
В таких зависимостях не нашлось каких-то изощренных моментов, которые нельзя найти в документации.
А вот что касаетсяф зависимостей для работы с внешним миром -- тут все куда более интересно. Например, мне кажется важным понимание работы драйверов с СУБД. И если нельзя оптимизировать какой-либо момент, то как минимум нужно быть
осведомленным об исключительных ситуациях, паниках и прочем, дабы не уронить сервис. В случае с кафкой желательно понимать логику комплексных компонентов в случае, если параметры выставлены по-дефолту, тем самым избежав "сюрпризов".
На самом деле большинство библиотек в Go предлагают тесты и бенчмарки на которых можно изучить поведение зависимости и быстродействие. Этих условных 20% как мне кажется будет для использования зависимости. Да и в Go довольно все просто.

Изучение кода зависимостей больше актуально к Java/C++/C, у которых не так развита экосистемы из коробки, как у Go. В Java могут возникунть проблемы с использованием памяти (это я лишь слышал), в плюсах -- работа с указателями и выстрелы в ногу.

