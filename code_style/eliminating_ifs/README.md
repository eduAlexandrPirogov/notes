# Избавляемся от условных конструкций

# Пример 1

Было:
```go

// Serializable representation of metric
type Metrics struct {
	ID    string   `json:"id"`              //Metric name
	MType string   `json:"type"`            // Metric type: gauge or counter
	Delta *int64   `json:"delta,omitempty"` //Metric's val if passing counter
	Value *float64 `json:"value,omitempty"` //Metric's val if passing gauge
	Hash  string   `json:"hash,omitempty"`
}

type PollCount counter // Count of times when Metrics was collected

type Polls struct {
	PollCount PollCount
}

//----
type FreeMemory gauge
type CPUutilization1 gauge
type TotalMemory gauge

type OpsUtil struct {
	FreeMemory      FreeMemory
	CPUutilization1 CPUutilization1
	TotalMemory     TotalMemory
}
```

Стало:

```go

type Tupler interface {
	// ToTuple converts type that implements Tupler interface to Tuple
	ToTuple() Tupler

	// SetField adds k/v pair to Tuple.
	//
	// Pre-cond: given key to be set and value for key
	//
	// Post-cond: return new tuple with updated field value
	SetField(key string, value interface{}) Tupler

	// GetField returns value by key of tuple.
	//
	// Pre-cond: given key
	//
	// Post-cond: returns value of field.
	// If field is exists with key returns val and bool = true
	// Otherwise return nil and bool = false
	GetField(key string) (interface{}, bool)

	// Aggregate aggregates to tuples to union them
	//
	// Pre-cond: given tuple to aggregate with
	//
	// Post-cond: returns new Tupler and error
	// If success return Tuplers and error = nil
	// Otherwise return nil and error
	Aggregate(with Tupler) (Tupler, error)
}


func (m Metrics) ToTuple() tuples.Tupler {
	var tuple tuples.Tupler
	switch m.MType {
	case "counter":
		if m.Delta == nil {
			tuple, _ = createCounterState(m.ID, m.MType, "")
			return tuple
		}
		tuple, _ = createCounterState(m.ID, m.MType, fmt.Sprintf("%d", *m.Delta))
	case "gauge":
		if m.Value == nil {
			tuple, _ = createCounterState(m.ID, m.MType, "")
			return tuple
		}
		tuple, _ = createGaugeState(m.ID, m.MType, fmt.Sprintf("%.20f", *m.Value))
	}

	return tuple
}
```


# Пример 2

Было:

Стало:


# Пример 3

Было:

Стало:


# Пример 4

Было:

Стало:


# Пример 5

Было:

Стало:
