# Что делает тесты хорошими

#Пример 1

Тест изначально проверял, что записываемые данные в БД реплицировались, и тест был завязан на конкретной реализации БД и Journaler'a.
Из-за этого, тест походил больше на интеграционный поскольку нужно было сделать множество методов чтобы проверить корректность функции.

Было:
```go
func TestWriteWithReplicate(t *testing.T) {
	sut := MemStorageDB() // Concrete DB with concrete journaler
	}

	metric := metrics.Metrics{
		ID:    "Some val",
		MType: "gauge",
		Delta: nil,
		Value: nil,
	}

	tl := tuples.TupleList{}
	tl.Add(metric.ToTuple())

	sut.Storage.Write(tl)
	sut.Storage.Clear() // Mock method to imitate new DB instance

	sut.Journaler.Restore()
	actual, _ := sut.Storage.Read(t1)

	assert.Equals(t, metric, actual)
}
```

Мы хотим выявить, что при записи в БД, запись реплицируется. Это можно проверить следующим свойством:
Если записывается значение в БД
И
Значение также реплицируется
То
Количество записанных значений будет равно количество реплицируемых.

Стало:
```go
// Own mock object
type MockJournal struct {
	Restored [][]byte
}

func (m MockJournal) Start() error {
	return nil
}

func (m MockJournal) Restore() ([][]byte, error) {
	return [][]byte{}, nil
}

func (m MockJournal) RestoredRecords() map[string]tuples.Tupler {
	res := make(map[string]tuples.Tupler)
	for _, v := range m.Restored {
		res[string(v)] = nil
	}
	return nil
}

func (m MockJournal) Write(write []byte) {
	m.Restored = append(m.Restored, write)
}

func TestWriteWithReplicate(t *testing.T) {
	sut := DB{
		Storage: MemStorageDB(),
		//Mock object to test abstract effect of replicate to journal
		Journaler: MockJournal{
			Restored: make([][]byte, 0),
		},
	}

	metric := metrics.Metrics{
		ID:    "Some val",
		MType: "gauge",
		Delta: nil,
		Value: nil,
	}

	tl := tuples.TupleList{}
	tl.Add(metric.ToTuple())

	sut.Storage.Write(tl)

	actual := len(sut.Journaler.RestoredRecords()) // Checking abstract Affect
	assert.Greater(t, actual, 0)
}
```

Обратите внимание на две последние строки теста. Я не проверяю, как записывались значения и чему они равны, поскольку в зависимости от реализации,
реплицируемое значение может сериализоваться вообще на стороннем сервисе, доступ на чтение к которому мы не имеем, и проверить значения не может.
Но мы проверяем свойство абстрактного эффекта, что, записав значение в Базу (пустую!), то значение также реплицировалось.

#Пример 2

Рассмотрим следующий юнит-тест (опять же, он должен быть, как мне кажется, интеграционным, но имеем, что имеем), который тестирует, что 
при записи метрики Counter, мы удостоверяемся, что запись в API работает корректно. 
Опять же, строчка `server := createTestServer() //server with concrete DB realisation` реализует конкретный сервер с конкретной БД и Репликатором.

Было:
```go
func TestCorrectCounterMetric(t *testing.T) {
	deltas := []int64{1, 2, 3, 4, 5, 6, 7, 8}
	var expectedCounter int64 = 0
	data := []Payload{}
	for i := range deltas {
		data = append(data, Payload{
			StatusCode: http.StatusOK,
			Metric: metrics.Metrics{
				ID:    "PollCount",
				MType: "counter",
				Delta: &deltas[i],
				Value: nil,
			},
		})
		expectedCounter += deltas[i]
	}

	server := createTestServer() //server with concrete DB realisation
	defer server.Close()
	sum := int64(0)
	for i, actual := range data {
		sum += deltas[i]
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

			assert.EqualValues(t, sum, *respJs.Delta)
		})
	}
}
```

Исправил следующим образом, разделив тест на два теста. Почему я сделал именно так, описываю в самом низу в выводе.
А пока обозначу такую логику -- если поступила корректная метрика, то сервер вернет положительный ответ (201 status code). БД -- это побочный эффект.

Я разделил тест на следущих два, который отправляет корректный объект, чтобы:
1) Удостовериться, что получу ожидаемый код
2) удостовериться, что будет воспроизведен абстрактный эффект записи в БД.
```go

//удостовериться, что будет воспроизведен абстрактный эффект записи в БД.
func TestCorrectWriteCounterMetric(t *testing.T) {
	deltas := []int64{1, 2, 3, 4, 5, 6, 7, 8}
	var expectedCounter int64 = 0
	data := []Payload{}
	for i := range deltas {
		data = append(data, Payload{
			StatusCode: http.StatusOK,
			Metric: metrics.Metrics{
				ID:    "PollCount",
				MType: "counter",
				Delta: &deltas[i],
				Value: nil,
			},
		})
		expectedCounter += deltas[i]
	}

	server, h := createTestServer()
	defer server.Close()
	sum := int64(0)
	for i, expected := range data {
		sum += deltas[i]
		t.Run("Correct counter", func(t *testing.T) {
			js, err := json.Marshal(expected.Metric)
			if err != nil {
				t.Errorf("got error while marshal json %v", err)
			}

			resp, err := executeUpdateRequest(server, js)
			require.Nil(t, err)
			defer resp.Body.Close()

			assert.Equal(t, expected.StatusCode, resp.StatusCode)

			assert.Greater(t, h.DB.Storage.Len(), 0) //abstract effect. Storage of DB was increased if it was empty
		})
	}
}

//Удостовериться, что получу ожидаемый код
func TestCorrectCounterMetric(t *testing.T) {
	deltas := []int64{1, 2, 3, 4, 5, 6, 7, 8}
	var expectedCounter int64 = 0
	data := []Payload{}
	for i := range deltas {
		data = append(data, Payload{
			StatusCode: http.StatusOK,
			Metric: metrics.Metrics{
				ID:    "PollCount",
				MType: "counter",
				Delta: &deltas[i],
				Value: nil,
			},
		})
		expectedCounter += deltas[i]
	}

	server, _ := createTestServer()
	defer server.Close()
	sum := int64(0)
	for i, expected := range data {
		sum += deltas[i]
		t.Run("Correct counter", func(t *testing.T) {
			js, err := json.Marshal(expected.Metric)
			if err != nil {
				t.Errorf("got error while marshal json %v", err)
			}

			resp, err := executeUpdateRequest(server, js)
			assert.EqualValues(t, expected.StatusCode, resp.StatusCode)
		})
	}
}
```

#Пример 3

В продолжение примера 2. В примере два побочный эффект проверялся на пустой БД.
Но каков будет побочный эффект, если мы запишем множество объектов. Должна ли БД перезаписать это, или создать новые объекты?
Это зависит от требований, но суть в том, что изначально, я в тесте проверял КАКОЕ значение записывает/перезаписывает БД, вместо того, чтобы проверить эффект:

Было:
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
	server, _ := createTestServer()
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

			// Проверяем КАК бд записала значение
			assert.EqualValues(t, actual.StatusCode, resp.StatusCode)
			assert.Greater(t, resp.ContentLength, int64(0))
			require.EqualValues(t, updatedMetric, *respJs.Delta)
			assert.Equal(t, true, true)
		})
	}
}

```

В моем случае, нужно было перезаписывать текущее значение. Значит и стоит проверять соответствующий побочный эффект, например:

Стало:
```go
//При записи нескольких однотипных объектов, старый объект перезаписывается (рамзер БД не увеличивается)
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
	server, h := createTestServer()
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
			defer resp.Body.Close()

			buffer, err := io.ReadAll(resp.Body)
			

			//При записи нескольких однотипных объектов, старый объект перезаписывается (рамзер БД не увеличивается)
			assert.Equal(t, 1, h.DB.Storage.Len())
		})
	}
}
```


# Пример 4

Рассмотрим ещё пример из Docker Engine. Я подчеркнул моменты, на которые стоит обратить внимание.
Тест проверяет ошибочность запуска daemon'a на винде, что при работе сервиса возникают побочные эффекты связанные с обработкой ошибок.

Строчка  `m, err := mgr.Connect()` завязана на конкретную реализацию windows. 


```go
func TestEnsureServicesExistErrors(t *testing.T) {
	m, err := mgr.Connect() //----------------------------------------------------------------------------------------------------------------------------------------------------------------
	if err != nil {
		t.Fatal("failed to connect to service manager, this test needs admin")
	}
	defer m.Disconnect()
	s, err := m.OpenService(existingService) 
	if err != nil {
		t.Fatalf("expected to find known inbox service %q, this test needs a known inbox service to run correctly", existingService)
	}
	defer s.Close()

	for _, testcase := range []struct {
		input         []string
		expectedError string
	}{
		{
			input:         []string{"daemon_windows_test_fakeservice"},
			expectedError: "failed to open service daemon_windows_test_fakeservice",
		},
		{
			input:         []string{"daemon_windows_test_fakeservice1", "daemon_windows_test_fakeservice2"},
			expectedError: "failed to open service daemon_windows_test_fakeservice1",
		},
		{
			input:         []string{existingService, "daemon_windows_test_fakeservice"},
			expectedError: "failed to open service daemon_windows_test_fakeservice",
		},
	} {
		t.Run(strings.Join(testcase.input, ";"), func(t *testing.T) {
			err := ensureServicesInstalled(testcase.input) //abstract effect-----------------------------------------------------------------------------------------------------------------------------
			if err == nil {
				t.Fatalf("expected error for input %v", testcase.input)
			}
			if !strings.Contains(err.Error(), testcase.expectedError) {
				t.Fatalf("expected error %q to contain %q", err.Error(), testcase.expectedError)
			}
		})
	}
}
```

Поскольку работа докера сильно зависит, на какой ОС использует, то тут такой подход мне кажется ненадежным. Во-первых, я не смогу запустить адекватно тесты на Ubuntu, тесты, предназначенные на винде.
Во-вторых, как будет работать дкоер c Windows 10 и Windows XP? 
Конечно пример искуственный, но суть в том, что изменив среду, наши тесты могут легко поломаться. Причем тесты вышеупомянутые проверяет конкретные ошибки (конечно, в Go стиле, но речь немного о другом).

Как бы изменил:
```go
type MockMgr struct {
    MockHandle windows.MockWindows
}

// Далее методы для MockWindows

//Сам тест
func TestEnsureServicesExistErrors(t *testing.T) {
	m, err := mgr.Connect() //Mock
	if err != nil {
		t.Fatal("failed to connect to service manager, this test needs admin")
	}
	defer m.Disconnect()
	s, err := m.OpenService(existingService) 
	if err != nil {
		t.Fatalf("expected to find known inbox service %q, this test needs a known inbox service to run correctly", existingService)
	}
	defer s.Close()

	for _, testcase := range []struct {
		input         []string
		expectedError string
	}{
		{
			input:         []string{"daemon_windows_test_fakeservice"},
			expectedError: "failed to open service daemon_windows_test_fakeservice",
		},
		{
			input:         []string{"daemon_windows_test_fakeservice1", "daemon_windows_test_fakeservice2"},
			expectedError: "failed to open service daemon_windows_test_fakeservice1",
		},
		{
			input:         []string{existingService, "daemon_windows_test_fakeservice"},
			expectedError: "failed to open service daemon_windows_test_fakeservice",
		},
	} {
		t.Run(strings.Join(testcase.input, ";"), func(t *testing.T) {
			err := ensureServicesInstalled(testcase.input) //abstract effect-----------------------------------------------------------------------------------------------------------------------------
			// Проверяем наличие асбатрктного эффекта (возникновение ошибки
			if err == nil {
				t.Fatalf("expected error for input %v", testcase.input)
			}
			// А вот это бы я совсем убрал, т.к. это должен проверять соответствующий модуль обработки ошибок
			/*if !strings.Contains(err.Error(), testcase.expectedError) {
				t.Fatalf("expected error %q to contain %q", err.Error(), testcase.expectedError)
			}*/
		})
	}
}
```

# Пример 5

