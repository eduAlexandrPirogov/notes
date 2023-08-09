# О циклах по умному

# Пример 1

Было:

```go
func buildAndSendCounters(r RPCClient, m metrics.Metricable) {
	counters := buildMessageCounters("counter", m.AsMap())
	for _, c := range counters {
		r.AddInStreamCounter(&c, r.counterStream)
	}
}
```

Стало:

```go
func buildAndSendCounters(r RPCClient, m metrics.Metricable) {
	counters := buildMessageCounters("counter", m.AsMap())
	fp.Map(func(c Counter) error { return r.AddInStreamCounter(&c, r.counterStream) })(counters)
}
```

# Пример 2

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

Стало:

```go
func buildMessageCounters(counters []Counter) []Counter {
	res := fp.Reduce(func(acc []Counter, c Counter)[]Counter { 
		c := buildCounter((c.Name, c.Type, c.Val))
		return append(acc, c)
		}, []Counter{})(counters)
	return res
}

func buildCounter(name string, typ string, val int64) Counter {
	key := agent.ClientCfg.Hash
	return Counter{
		Id:    name,
		Type:  typ,
		Delta: val,
		Hash:  crypt.Hash(fmt.Sprintf("%s:counter:%d", name, val), key),
	}
}
```

# Пример 3

Было:

```go
/ restore writes restored metric to DB
//
// Pre-cond: given slice of slice of bytes/ marshaled metrics
//
// Post-cond: unmarshal metric and writes it to DB
// If metric has type counter DB will contain last value of counter
func (d *DB) restore(bytes [][]byte) {
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

Стало:

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

func unmarshalMetrics(bytes []byte) []metrics.Metrics {
	var metric metrics.Metrics
	err := json.Unmarshal(bytes, &metric)
	if err != nil {
		return []metrics.Metrics{}
	}
	return []metrics.Metrics{metric}
}

```


# Пример 4

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


# Пример 5

Было:

```go
```

Стало:

```go
```
