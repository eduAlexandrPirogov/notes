# Раздутый код. часть 2

Пример 1

```go
func (h httpMemTracker) ReadAndSend(readInterval time.Duration, sendInterval time.Duration) {
 readTicker := time.NewTicker(readInterval)
 sendTicker := time.NewTicker(sendInterval)
 for {
  // Раздутый код. Тут лучше бы разделить два таймера на два разных метода отдельных.
  select {
  case <-readTicker.C:
   h.update()
  case <-sendTicker.C:
   h.send()
  }
 }
}
```


Пример 2

```go
func (p *PollCount) Read() error {
   newVal := p.Poll + p.Incr()
   //Раздутый код
   // Вроде и значимая проверка, но подобное и в отдельную функцию не хочется оформлять
   // Мелочь, но из-за этого приходится в голове подобную деталь
   oldVal := p.Poll
   if newVal < oldVal {
       return errors.New("Int overflow")
   } else {
       p.Poll = newVal
       return nil
   }
}
```

Пример 3

```cpp
void MongoWrapper::findMany(MongoQueryBuilder& query, std::string& collection)
{
  //...
	auto order = doc_test.view();
	auto opts = mongocxx::options::find{};
	opts.sort(order);
	mongocxx::cursor cursor = coll.find(doc.view(), opts);
	for (auto doc : cursor) {
    // раздутый код. Закидываем результат в контенйер и логируем. Тут лучше было бы сделать механизм логгирования другим и убрать отсюда.
		std::string t = bsoncxx::to_json(doc);
		Json* p = new Json{ t };
		MongoWrapper::query_result.push_back(p);
    MongoWrapper::log(p);
	}
}
```

Пример 4

```cpp
...
for (int i = 0; i < yLinesCount; i++)
{

   double valueAtLineY = normalizedToValue.TransformPoint({ 0, normalizedLineY }).m_y;
   
   auto text = wxString::Format("%.2f", valueAtLineY);
   text = wxControl::Ellipsize(text, dc, wxELLIPSIZE_MIDDLE, chartArea.GetLeft() - labelsToChartAreaMargin);
   
   
   // Раздутый кодю. Если закинуть в одну функцию, то она будет использоваться только здесь и это будет такой себе строительный блок
   // За один поток пытаемся сделать две несвязанные операции
   double tw, th;
   gc->GetTextExtent(text, &tw, &th);
   gc->DrawText(text, chartArea.GetLeft() - labelsToChartAreaMargin - tw, lineStartPoint.m_y - th / 2.0);
}
```

Пример 5

```php
public function forecastForSaleServices($services) {
   foreach($services as $service => $turnover)
   {
       $firstPart = $turnover[0]
       // Раздутый код. Не две семантически независимые операции, которые нельзя закинуть в одну функцию, т.к. будет недостаточно автономной
       $fullPrice = $this->add($turnover)
       $part = $this->partion($firstPart)
       ...
   }
}
```
