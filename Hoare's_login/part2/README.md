# Логика Хоара для программистов-2

# Пример 1

Метод для декодирования байтов от Postgres'a. 
Принимал массив размером 50 байтов и отдавал слайс декодированого сообщения
(в зависимости от тега сообщение декодировалось по-разному).
Было:
```go
// Pre-cond: given array of received bytes of postgres protocol msg
// Post-cond: returns slice of decoded payload
func (s* Socket) receive(bytes [50]byte) []byte{
  tag := bytes[0]
  msg := s.readPayload(tag, bytes)
  return msg
}
```

Можно ослабить пред-условия с массива, до слайса. Это наиболее слабая форма потока байтов. (Сила свойства в данном сулчае есть количество входящих байтов).
Что касается пост-условия, то кажется, что по аналогии с предусловием, можно "усилить" со слайса до массива с фиксированным размером.
Но у в данном случае стоит посмотреть в другом направлении. Что если тэг сообщения передается некорректный?
В данном случае, лучше усилить, пост-условие дополнительным полем error, тем самым предотвращая последующую логику кода.

Стало:
```go
// Pre-cond: given slice of received bytes of postgres protocol msg
// Post-cond: returns slice of trim'd decoded payload
func (s* Socket) receive(bytes []byte) ([]byte, error) {
  tag := bytes[0]
  msg, err := s.readPayload(tag, bytes)
  return msg, err
}
```

# Пример 2



Было:
```go
// Write writes tupler to DB and return written state
//
// Pre-cond: given TupleList to Write in given Storer
//
// Post-cond: if was written successfully returns TupleList with writen tuples and error nil
// Otherwise returns empty Tuple and error
func Write(s Storer, states tuples.TupleList) (tuples.TupleList, error) {
	newStates, err := s.Write(states)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return newStates, nil
}
```

Ослабим предусловие c конкретного TupleList, до итерируемого интерфейса, что расширит пул входящих значений, не нарушая предыдущую логику.
А вот постусловие можно оставить как есть. Его невозможно усилить, так как это и так конкретный тип.
Если же потребуется, чтобы метод возвращал не TupleList, а TupleMap, то я бы написал для этого отдельную функцию, или можно обойтись дженериками с F-полиморфизмом.

Стало:
```go
// Write writes tupler to DB and return written state
//
// Pre-cond: given iterable container of tuples to Write in given Storer
//
// Post-cond: if was written successfully returns New TupleList instance of written tuples and error nil
// Otherwise returns emtyTuple and error
func Write(s Storer, states std.Iterable) (tuples.TupleList, error) {
	newStates, err := s.Write(states)
	if err != nil {
		return tuples.TupleList{}, err
	}

	return newStates, nil
}
```

# Пример 3

Пример поинтереснее.

Было:
```go
// package parser holds table of avaible parsers for parsing
package parser

import (
	"log"
	"product-parser/comparator"
	"product-parser/parser/csv"
	"product-parser/parser/json"
)

type Extension string

const (
	JSON Extension = ".json"
	CSV  Extension = ".csv"
)

type Parser[T any] interface {
	// Parses given text and unmarshal into T instance
	Parse(text string) (T, error)
}

// New return instance of parser depending on given file extension
//
// Pre-cond: given extension Extension of file
// Post-condL: returns instance of parser depends on given extension
// or programm exits
func New[T any](ext Extension) Parser[T] {      // <-------------------------------------------- Интересующая функция
	avaibleParsers := parsers[T]()
	parser, ok := avaibleParsers[ext]
	if !ok {
		log.Fatalf("given unsupported extension %s", ext)
	}

	return parser
}

func parsers[T any]() map[Extension]Parser[T] {
	return map[Extension]Parser[T]{
		CSV:  csv.New[T](),
		JSON: json.New[T](),
	}
}
```

Нас интересует функция New(). 
Тут можно ослабить пост-условие с типа Extension, до string, тем самым позволяя передавать любые расширения, которые не присутствуют явно в константах.
Это как вариант, но в данном случае я оставил бы предусловия без изменений.

А вот что касается пост-условий, ситуация интереснее. Дело в том, что тут:
1) я нарушаю Go-идиому, возвращая интерфейс
2) он ещё и дженерик

С одной стороны это гибкость кода и сокращение кодовой базы (в пакете не присутствуют NewCSVParser, NewJSONParser...).
Можно ли тут усилить пост-условие, при этом сохраняя возвращение интерфейса? Можно, за счет F-bound полиморфизма:

Стало:
```go
// New return instance of parser depending on given file extension
//
// Pre-cond: given string extension of file
// Post-cond: if extension is exists, returns instance of Parser that
// implements comparator.Comparable
func New[T comparator.Comparable[T]](ext string) Parser[T] {
	avaibleParsers := parsers[T]()
	parser, ok := avaibleParsers[Extension(ext)]
	if !ok {
		log.Fatalf("given unsupported extension %s", ext)
	}

	return parser
}


type Comparable[T Comparable[T]] interface {
	Greater(with Comparable[T]) Comparable[T]
}
```

За счет чего произошло усиление пост-условия? Конкретизация/усиление требования для возвращаемого значения.

# Пример 4

Было:
```go
```

Стало:
```go
```

# Пример 5

Было:
```go
```

Стало:
```go
```

# Вывод

