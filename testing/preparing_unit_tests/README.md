# Как правильно готовить юнит-тесты

#1

Was:
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

#2

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

Now:
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

#3

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

Now():
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
#4

```go


func FuzzCorrectCreateCounterMetricStateFromCounterMetric(f *testing.F) {
	//Arrange
	deltas := []int64{0, 100, 500, 999999, 234524}
	names := []string{"s", "a", "w", "e", "c"}

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

#5

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
