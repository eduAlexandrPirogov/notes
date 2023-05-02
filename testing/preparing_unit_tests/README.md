# Как правильно готовить юнит-тесты

Пример 1
Суть теста заключается в удостоверение, что некорректные тип метрики не будет сохраняться в БД. 
Логика такова -- имется лишь один верный для метрики Gauge тип = gauge. Все остальные не подходят.

Было:
```go
func TestIncorrectGaugeUpdateHandler(t *testing.T) {
	values := []float64{-1.1, 0, 1.1}
	data := []Payload{
		{
			StatusCode: http.StatusBadRequest,
			Metric: metrics.Metrics{
				ID:    "",
				MType: "gauge",
				Delta: nil,
				Value: &values[0],
			},
		},
		{
			StatusCode: http.StatusNotImplemented,
			Metric: metrics.Metrics{
				ID:    "1",
				MType: "",
				Delta: nil,
				Value: &values[0],
			},
		},
		{
			StatusCode: http.StatusNotImplemented,
			Metric: metrics.Metrics{
				ID:    "1",
				MType: "gauge1",
				Delta: nil,
				Value: &values[0],
			},
		},
		{
			StatusCode: http.StatusBadRequest,
			Metric: metrics.Metrics{
				ID:    "1",
				MType: "gauge",
				Delta: nil,
				Value: nil,
			},
		},
	}

	server := createTestServer()
	defer server.Close()
	for _, actual := range data {
		t.Run("Incorrect gauge", func(t *testing.T) {
			js, err := json.Marshal(actual.Metric)
			if err != nil {
				t.Errorf("got error while marshal json %v", err)
			}

			resp, err := executeUpdateRequest(server, js)
			require.Nil(t, err)
			defer resp.Body.Close()

			assert.EqualValues(t, actual.StatusCode, resp.StatusCode)
		})
	}
}
```

Думал, что Fuzz придется настривать, как для С++, но в Go он уже встроен и его легко интегрировать.
Таким образом, с помощью фазз темтирования, мы легко проверяем истинность свойства "Mtype != 'gauge' => http.StatusCode = http.NotImplemented'

Now:
```go

func FuzzIncorrectTypeGaugeUpdateHandler(t *testing.F) {
	values := []float64{-1.1, 0, 1.1}
	cases := []string{"", "qwe", "!123wd", "~$@!#% cwerq23"}
	for _, edge := range cases {
		t.Add(edge)
	}

	server := createTestServer()
	defer server.Close()

	t.Fuzz(func(t *testing.T, orig string) {
		actual := Payload{
			StatusCode: http.StatusNotImplemented,
			Metric: metrics.Metrics{
				ID:    "",
				MType: orig,
				Delta: nil,
				Value: &values[0],
			},
		}
		js, err := json.Marshal(actual)
		if err != nil {
			t.Errorf("got error while marshal json %v", err)
		}

		resp, err := executeUpdateRequest(server, js)
		require.Nil(t, err)
		defer resp.Body.Close()

		assert.EqualValues(t, actual.StatusCode, resp.StatusCode)
	})

}

```

Пример 2

Суть данного теста заключается в том, что если мы обновляем метрику типа counter, то значение равно текущее + новое.
Был тест опять же написан с пощоью табличных тестов:
```go

func TestCorrectCounterUpdateHandler(t *testing.T) {
	deltas := []int64{0, 1, 2, 3, 4, 5}
	data := []Payload{}
	for i := range deltas {
		data = append(data, Payload{
			StatusCode: http.StatusOK,
			Metric: metrics.Metrics{
				ID:    "some",
				MType: "counter",
				Delta: &deltas[i],
				Value: nil,
			},
		})
	}
	server := createTestServer()
	defer server.Close()
	var updatedMetric int64 = 0
	for _, actual := range data {
		updatedMetric += *actual.Metric.Delta
		t.Run("Correct counter", func(t *testing.T) {
			js, err := json.Marshal(actual.Metric)
			if err != nil {
				t.Errorf("got error while marshal json %v", err)
			}

			resp, err := executeUpdateRequest(server, js)
			require.Nil(t, err)
			defer resp.Body.Close()

			var respJs metrics.Metrics
			buffer, err := io.ReadAll(resp.Body)
			if err != nil {
				t.Errorf("got error while reading body %v", err)
			}
			err = json.Unmarshal(buffer, &respJs)
			assert.Nil(t, err)

			assert.EqualValues(t, actual.StatusCode, resp.StatusCode)
			assert.Greater(t, resp.ContentLength, int64(0))
			require.EqualValues(t, updatedMetric, *respJs.Delta)
			assert.Equal(t, true, true)

		})
	}
}
```

С фазз-тестированием мы можем проверить легко истинность нашего свойства:

```go

func FuzzCorrectCounterUpdateHandler(f *testing.F) {
	deltas := []int64{0, 5, 10, 100, 5000, 10000}
	for _, delta := range deltas {
		f.Add(delta)
	}

	server := createTestServer()
	defer server.Close()

	var updatedMetric int64 = 0
	f.Fuzz(func(t *testing.T, orig int64) {
		log.Println(orig)
		updatedMetric += orig
		actual := Payload{
			StatusCode: http.StatusOK,
			Metric: metrics.Metrics{
				ID:    "some",
				MType: "counter",
				Delta: &orig,
				Value: nil,
			},
		}

		js, err := json.Marshal(actual.Metric)
		if err != nil {
			t.Errorf("got error while marshal json %v", err)
		}

		resp, err := executeUpdateRequest(server, js)
		require.Nil(t, err)
		defer resp.Body.Close()

		var respJs metrics.Metrics
		buffer, err := io.ReadAll(resp.Body)
		if err != nil {
			t.Errorf("got error while reading body %v", err)
		}
		err = json.Unmarshal(buffer, &respJs)
		assert.Nil(t, err)

		assert.EqualValues(t, actual.StatusCode, resp.StatusCode)
		assert.Greater(t, resp.ContentLength, int64(0))
		require.EqualValues(t, updatedMetric, *respJs.Delta)
		assert.Equal(t, true, true)
	})
}
```

Пример 3.

Данный способ можно применять и в обратную сторону. 
Нижеприведенный тест проверяет, что созданный запрос с помощью кортежа с корректными параметрамаи отработает корректно. 
Проверяемый метод сопоставляет данный кортеж ({"name": "some", "type":"counter"}) с имеющимися в БД.
То есть, мы проверили лишь свойство корректного исполнения:

```go

func TestReadByParams(t *testing.T) {
	db := MemStorage{Metrics: filledDB()}
	expectedMetrics := db.Metrics

	for _, expectedType := range expectedMetrics {
		for _, expected := range expectedType {
			t.Run("expected", func(t *testing.T) {
				mname, _ := expected.GetField("name")
				mtype, _ := expected.GetField("type")

				query := tuples.NewTuple()
				query.SetField("name", mname.(string))
				query.SetField("type", mtype.(string))

				_, err := db.Read(query)

				assert.Nil(t, err)
				//assert.EqualValues(t, 1, len(actuals))
			})
		}
	}
}
```

Но зачастую стоить проверять также и то, что вышеприведенное использование -- единственное корректное, а остальные случаи использовнаия неверны.
Например, можно написать тест, который отправляет кортеж не с полем name, а любым другим полем. 
То есть, мы проверяем, что мы для корректного исполнения может использовать в качестве фильтрации атрибут "name" и никакой иной:

```go

func FuzzReadByIncorrectParams(f *testing.F) {
	db := MemStorage{Metrics: filledDB()}
	expectedMetrics := db.Metrics

	cases := []string{"123123", "", "1qe324s", "~@!$#@^)&%", "_+=/'doimq[g]"}
	for _, edge := range cases {
		f.Add(edge)
	}

	f.Fuzz(func(t *testing.T, orig string) {
		for _, expectedType := range expectedMetrics {
			for _, expected := range expectedType {
				mname, _ := expected.GetField("name")
				mtype, _ := expected.GetField("type")

				query := tuples.NewTuple()
				query.SetField(orig, mname.(string))
				query.SetField("type", mtype.(string))

				_, err := db.Read(query)

				assert.NotNil(t, err)
			}
		}
	})
}
```

Пример 4

Обычно рассматриваются варианты "корректное" и "некорректное" исполнения тестируемого объекта, и для каждого пишутся соответствующие тесты.
Но бывают случаи, когда мы полностью исключаем некорректное поведения. Например, в проекте, я создал собственнйы тип Кортеж (Tuple), в который можно 
преобразовать любой тип данных. Плюс, у каждого типа данных имеется обратный метод FromTuple.
Аналогично работает для некоторых типов и маршалинг/анмаршалинг, т.е. мы можем свободны маршалить и анмаршалить разлчиные типы между собой.

И у нас получается следующее свойство: если мы маршалим тип А (который может быть преобразован в Tuple) и демаршалим в тип Б, 
то поля типа А должны быть равны полям типа Б.
Тут уже сработал метод пристального взгляда, когда глядя на реализованные методы типов А и Б вытегает данное свойство.
Плюс добавим фазз тестирования, чтобы убедиться, что наше свойство не зависит от содержимого значений типов А и Б.

Тут интересный момент таков, что мы не можем проверить неправильную работу :), до и оно нам не нужно
Новый тест получился таковым:
```go
func FuzzCorrectCreateCounterMetricStateFromCounterMetric(f *testing.F) {
	//Arrange
	deltas := []int64{0, 100, 500, 999999, 234524}
	names := []string{"asds", "aqweqweewewq", "wfffffsa", "eqwersa", "c"}

	for i := 0; i < len(names); i++ {
		f.Add(names[i], deltas[i])
	}
	f.Fuzz(func(t *testing.T, n string, d int64) {
		metric := Metrics{
			ID:    n,
			MType: "counter",
			Delta: &d,
		}
		body, _ := json.Marshal(metric)
		log.Printf("%s", body)
		var sut CounterState

		//Act
		err := json.Unmarshal(body, &sut)

		//Assert
		assert.Nil(t, err)
		assert.EqualValues(t, metric.ID, sut.Name)
		assert.EqualValues(t, metric.MType, sut.Type)
		assert.EqualValues(t, metric.Delta, sut.Value)
	})
```

Пример 5

Следующий метод тестирует, что метод Read, если ему подали запрос на чтение всех данных, вернет все данные. Тут момент такой, что тест работает корреткно,
но он проверяет каждое значение, что не совсем верно. Тут нужно выделить свойство.
Например, если мы создадим заглушку БД с некоторыми данными, то можно выделить следующее свойство:
Чтение всех элементов из заглушки даст нам количество элементов, равное числу элементов в заглушке:

Был тест таков:
```go
func TestReadAllByParam(t *testing.T) {
	db := MemStorage{Metrics: filledDB()}
	cases := []string{"gauge", "counter"}

	for _, expectedType := range cases {
		t.Run(expectedType, func(t *testing.T) {
			query := tuples.NewTuple()
			query.SetField("name", "*")
			query.SetField("type", expectedType)

			actuals, _ := db.Read(query)

			for actuals.Next() {
				actualType, ok := actuals.Head().GetField("type")
				assert.True(t, ok)
				assert.EqualValues(t, expectedType, actualType.(string))
				actuals = actuals.Tail()
			}
		})
	}
}
```

Теперь проверим данное свойство, сравниваем число возвращенных элементов:
```go
func TestReadAllByParam(t *testing.T) {
	db := MemStorage{Metrics: filledDB()}
	cases := []string{"gauge", "counter"}

	for _, expectedType := range cases {
		t.Run(expectedType, func(t *testing.T) {
			query := tuples.NewTuple()
			query.SetField("name", "*")
			query.SetField("type", expectedType)

			actuals, _ := db.Read(query)

			assert.Equal(t, len(db.Metrics[expectedType]), actuals.Len())
		})
	}
}

```

=========================================

Подобный прием я использовал ранее, но немного в ином исполнении: при тестировании метода, рисовал дерево решения, от корня которого было две ветви:
корректное и некорректное исполнение.
Затем, от корректного исполнения рисовал ветви, которые характеризовали способ корректного исполнения (правильный поданный тип, значение) и соответствующие
варианты для некорректного исполнения. Отличие в том, что у меня было не на логическом уровне (не выделял свойства), точнее даже, было у меня все более конкректно
и низкоуровнево. Сейчас же, немного попрактиковавшись, я уже могу выделять более высокоуровневые свойство корректноести тестируемых фич.
