# Дополнительная сложность -- мать всех запашков кода

Ознакомившись с каталогом CodeSmells, пока искал примеры для материала, заметил, что каждый раз, когда натыкаюсь на CodeSmell, у меня невольно нахмуриваются брови, мол "что значит это строка/функция...?"

# Пример 1. Duplicated Code

С функционального программирования у меня вошло в привычку использовать вместо raw for-loops такие конструкции, как map/fold/reduce.
Каждый раз, когда мне встречается raw for-loops, мои брови начинают хмуриться.

Было:
```go
// restore writes restored metric to DB
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

Не обращаясь же к данному примеру, мое мнение касательно работы с циклами и вынесение их в отдельную конструкции таково:
1) итерация контейнера -- рекурсивное действие (берем текущий элемент и оставшиеся)
2) воспроизводим действие над текущим элементом (лямбда-функция)

И таким образом мы устраняем DuplicatedCode. Обратите внимание, что я делаю акцент на создании самостоятельной конструкции, а не просто вынесении общений части кода,
в каталоге об этом тоже написано, что просто вынесение общего кода не всегда лучшая идея:
https://wiki.c2.com/?DuplicatedCode#:~:text=In%20SmalltalkLanguage%2C%20I%20think%20duplicated%20code%20is%20more%20apparent%20than%20in%20other%20languages%20(and%20certainly%20easier%20to%20refactor%20away).%20In%20Java%2C%20though%2C%20I%20find%20that%20duplicated%20code%20isn%27t%20always%20obvious%20and%20sometimes%20impossible%20to%20refactor%20out.

# Пример 2. ValueObject

CodeSmell говорит, что в классе/структуре слишком много атрибутов. 
Ещё с курса по ООП во мне укоренилась привычка, что АТД не должен содержать бОлее двух атрибутов.
Этот "запашок" обычно выправляется у меня итеративно, когда я понимаю, что переменные можно вынести в отдельный объект.

Было:
```go

type MongoQuery struct {
   FieldsProjection []string
   FieldsFilter []string
   FieldOrder string
   DocsLimit int
   DocsSkip int
}
```

Стало:
```go

type Filter struct {
  Fields []string
  Prjections []string
}

type Offset struct {
     Limit int
     Skip int
}

// Наша структура хранит теперь лишь два атрибута
type MongoQuery struct {
    Filter
    Offset
}

// А Sort мы вынесем вообще из MongoQuery, поскольку это опция, которую можно вызвать без MongoQuery
type Sort struct {
    Field string
    Order string
}
```

Для меня это стало настолько естественным, что во времена, когда я писал на ООП языках, у меня хоть и было большое количество АТД с минимумом атрибутов (за что получал критику), но они все легко тестировались и 
было легко понять, за что отвечает конкретный АТД.

Но!

В контексте Go я бы на данный момент отметил некоторые исключения. Например, часто можно увидеть в Go структуры с 5-10 полями. И, как мне кажется, это можно не считать "запашком" до тех пор, пока структура
реализует функцию кортежа или модель для сериализации/десериализации данных.

# Пример 3. Same Name Different Meaning

Один из самых часто встречающихся "запашков", которые напрягают.
Например, приходит запрос на получение товаров GetGoods. В проекте имеется слов, который принимает запрос с методом GetGoods и отправляет запрос на выполнение метода слою БД с таким же методом GetGoods.
Да, по пакетам/названия класса можно их отличить, но при поиске слова ctrl+shift+f по всему проекту выдаются все GetGoods, неважно какую задачу они выполняют

Один из примерв
Было:
```go
func Read(s Storer, state tuples.Tupler) (tuples.TupleList, error) {
	states, err := s.Read(state)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```

Стало:
```go
func Read(s Storer, state tuples.Tupler) (tuples.TupleList, error) {
	states, err := s.Retrieve(state) // <-------------------------------------Read changed for Retreive
	if err != nil {
		return tuples.TupleList{}, err
	}

	return states, nil
}
```

Я считаю это важным. Часто методы слоя, который принимает запросы, имеют название, семантически обозначающие действие: GetGoods, UpdateGoods, AddGoods
Слой БД же должен отличаться, например: не GetGoods, а ReadGoods/RetreiveGoods. Не UpdateGoods, а ChangeGoods и т.п.
Мы не только облегчим себе жизнь с поиском и заменой, но и ясно будет, какой метод относится к какому слою.

# Пример 4. Long Method Smell


Довольно не редка ситуация, где в одном методе выполняются разные по уровню архитектуры логики.

Было:
```go
// Делаем запрос к БД и обрабатываем список
func (p *Postgres) ReadGauges(cond string) (tuples.TupleList, error) {
	query := fmt.Sprintf("SELECT * from READ_METRIC('gauge', '%s')", cond)
	rows, err := p.conn.Query(context.Background(), query)
	if err != nil {
		return tuples.TupleList{}, err
	}

	defer rows.Close()

	var res = tuples.TupleList{}
	for rows.Next() {
		var toScan metrics.GaugeState
		err := rows.Scan(&toScan.Name, &toScan.Type, &toScan.Value)
		if err != nil {
			return tuples.TupleList{}, err
		}
		res = res.Add(toScan)
	}
	return res, nil
}

}
```

Стало:
```go
func (p *Postgres) ReadGauges(cond string) (pgx.Row, error) {
	query := fmt.Sprintf("SELECT * from READ_METRIC('gauge', '%s')", cond)
	rows, err := p.conn.Query(context.Background(), query)
	return rows, err
}

// Совершенно в другом месте, отдельном от слоя БД, созщдать метод обработки полученных строк
func HandleGaugesRows(cond string) (tuples.TupleList, error) {
     rows, err := kernel.ReadGauges(cond)
     defer rows.Close()

	  var res = tuples.TupleList{}
	  for rows.Next() {
		  var toScan metrics.GaugeState
		  err := rows.Scan(&toScan.Name, &toScan.Type, &toScan.Value)
		  if err != nil {
			  return tuples.TupleList{}, err
		  }
		  res = res.Add(toScan)
	}
	return res, nil
}
```

Опять же из курса по ООП, если метод состоит из более чем, условно, 30 строк, то явно метод не соответствует SRP.
Разбиение метода на более мелкие есть путь к соблюдению SRP и созданию автономных единиц программных компонентов.

# Пример 5. Verb Subject

Буквально на новой работе обнаружил этот запашок оставленный мною :)

Было:
```go
type AdminHandler struct {}
                  // replace-verb, CreatedDateField -- subject
func (a AdminHandler) replaceCreatedDateField(dateField string) string{
   if dateField == "created" || dateField == "createdDate" {
      return "created_at"
   }

   return dateField 
}
```

Лучше вынести этот метод в отдельный пакет

Стало:

```go
package replacement

func ReplaceCreatedDateField(dateField string) string{
   if dateField == "created" || dateField == "createdDate" {
      return "created_at"
   }

   return dateField 
}
```

Таким образом я из структуры AdminHandler убрал ответственность за изменение строковых значений (не тем он должен заниматься).
В целом, когда в АТД имеется VerbSubject метод, то явно это метод будет делегировать Verb какому-то Subject'у, либо будет изменять этот Subject сам, что не всегда будет соответствовать SRP.

Такая маленькая фича, но полезная, поможет разделять сущности в проекте.

# По итогу

Интересный каталог запашков, некоторые из которых решили давние мои проблемы, как VerbSubject. Что понравилось, что под некоторыми случаями написаны достоинства и недостатки устранения запашков.
Применять данные приемы в качестве "шлифовки" кода я бы точно взял не все. Например, Classes with too few instance variables мне кажется не таким уж запашком, особенно в контексте Go.
Или  например InstanceofInConditionals тоже мне кажется, не всегда будет "запашком". Но сам факт, что на подобные моменты стоит обратить внимания -- здравый.
Да, один из главных плюсов таких приемов, что они обоснованы (как в положительную, так и в отрицательную сторону).
Интересно, можно ли натренировать какую-нибудь LLV, чтобы подобные запашки сама убирала...

На новой работе, как окончательно внедрюсь в коллектив, попробую протащить подобные вещи. Кодовой базы много и шлифовать там есть что. 
