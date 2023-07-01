# Группировка в функциях и файлах

Пример 1

```go
func (m *MemStats) Read() error {
	var runtimeMemStat runtime.MemStats
	runtime.ReadMemStats(&runtimeMemStat)
	{
		m.Alloc = Alloc(runtimeMemStat.Alloc)
		m.BuckHashSys = BuckHashSys(runtimeMemStat.BuckHashSys)
		m.Frees = Frees(runtimeMemStat.Frees)
		m.GCCPUFraction = GCCPUFraction(runtimeMemStat.GCCPUFraction)
		m.GCSys = GCSys(runtimeMemStat.GCSys)
		m.HeapAlloc = HeapAlloc(runtimeMemStat.HeapAlloc)
		m.HeapIdle = HeapIdle(runtimeMemStat.HeapIdle)
		m.HeapInuse = HeapInuse(runtimeMemStat.HeapInuse)
		m.HeapObjects = HeapObjects(runtimeMemStat.HeapObjects)
		m.HeapReleased = HeapReleased(runtimeMemStat.HeapReleased)
		m.HeapSys = HeapSys(runtimeMemStat.HeapSys)
		m.LastGC = LastGC(runtimeMemStat.LastGC)
		m.Lookups = Lookups(runtimeMemStat.Lookups)
		m.MCacheInuse = MCacheInuse(runtimeMemStat.MCacheInuse)
		m.MCacheSys = MCacheSys(runtimeMemStat.MCacheSys)
		m.MSpanInuse = MSpanInuse(runtimeMemStat.MSpanInuse)
		m.MSpanSys = MSpanSys(runtimeMemStat.MSpanSys)
		m.Mallocs = Mallocs(runtimeMemStat.Mallocs)
		m.NextGC = NextGC(runtimeMemStat.NextGC)
		m.NumForcedGC = NumForcedGC(runtimeMemStat.NumForcedGC)
		m.NumGC = NumGC(runtimeMemStat.NumGC)
		m.OtherSys = OtherSys(runtimeMemStat.OtherSys)
		m.PauseTotalNs = PauseTotalNs(runtimeMemStat.PauseTotalNs)
		m.StackInuse = StackInuse(runtimeMemStat.StackInuse)
		m.StackSys = StackSys(runtimeMemStat.StackSys)
		m.Sys = Sys(runtimeMemStat.Sys)
		m.TotalAlloc = TotalAlloc(runtimeMemStat.TotalAlloc)
		m.RandomValue = RandomValue(rand.Float64())
	}
	return nil
}
```

Пример 2

Пример 3
