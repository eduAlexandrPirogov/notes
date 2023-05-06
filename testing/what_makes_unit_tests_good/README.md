#

#1

Was
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

	actual := len(sut.Journaler.RestoredRecords())
	assert.Greater(t, actual, 0)
}
```

Now
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

	actual := len(sut.Journaler.RestoredRecords())
	assert.Greater(t, actual, 0)
}
```

#2

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

```go

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

#3
