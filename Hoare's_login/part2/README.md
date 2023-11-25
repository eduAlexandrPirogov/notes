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

Рассмотрим примеры, где достаточно изменить лишь спецификации, не меняя реализацию. 
Было:
```erl
// Pre-cond: given ChannelID and additional parameters for request to make request to the channel
// Post-cond: return the result of request
make_request(ChId, ParamList) ->
    metric:counter(?'A1S.METRIC.HTTP.ACCEPTED', [ChId]),
    StopWatch = metric:stop_watch_millis(),
    case (catch channel:call(ChId,{request,ParamList},infinity)) of
        {ok, Response} ->
            metric:timer_milli(?'A1S.METRIC.HTTP.SUCCESS', StopWatch(), [ChId]),
            {html, Response};
        {error, Error} ->
            metric:timer_milli(?'A1S.METRIC.HTTP.FAILED', StopWatch(), [ChId]),
            {html, "ERROR: " ++ Error};
        _ ->
            metric:timer_milli(?'A1S.METRIC.HTTP.FAILED', StopWatch(), [ChId]),
            {html, "ERROR: System error"}
    end.
```

Пред-условия в данном сложно как-либо ослабить, так как к ним предъявление наименее строгие требования на данный момент.
Они могли содержать "Дать айди СУЩЕСТВУЮЩЕГО канала" или иметь какой-то определенный вид параметров.

Постуловия же следуюет уточнить, а именно вид результата: 

Стало:
```erl
// Pre-cond: given ChannelID and additional parameters for request to make request to the channel
// Post-cond: returns tuple with {ok, Response} or {html, ErrorMsg}	 <-------------------------
make_request(ChId, ParamList) ->
    metric:counter(?'A1S.METRIC.HTTP.ACCEPTED', [ChId]),
    StopWatch = metric:stop_watch_millis(),
    case (catch channel:call(ChId,{request,ParamList},infinity)) of
        {ok, Response} ->
            metric:timer_milli(?'A1S.METRIC.HTTP.SUCCESS', StopWatch(), [ChId]),
            {html, Response};
        {error, Error} ->
            metric:timer_milli(?'A1S.METRIC.HTTP.FAILED', StopWatch(), [ChId]),
            {html, "ERROR: " ++ Error};
        _ ->
            metric:timer_milli(?'A1S.METRIC.HTTP.FAILED', StopWatch(), [ChId]),
            {html, "ERROR: System error"}
    end.
```

# Пример 5

Было:
```erl
// Pre-cond: given decode type ('7bit', gsm or known encoding other type), message to decode and udhLen
// Post-cond: returned decoded message
decode('7bit',M,UdhLen) ->
    BinMsg = if is_binary(M) -> M; true -> list_to_binary(M) end,
    LeadingCount = case UdhLen rem 7 of
                       0 -> -1;
                       L -> L
                   end,
    M1 = add_leading_zeros(LeadingCount,BinMsg),
    ClosingCount = case size(M1) rem 7 of
                       0 -> -1;
                       C -> 7-C
                   end,
    M2 = add_closing_zeros(ClosingCount,M1),
    M3 = from_7bit(M2),
    M4 = cut_off_closing_zeros(M3),
    cut_off_leading_zeros(LeadingCount+1,M4);
decode(gsm,M,_) ->
    {_, UtfMsg} = gsm_to_utf16(if is_binary(M) -> M; true -> list_to_binary(M) end),
    list_to_binary(iconv:convert("UTF-16BE", "UTF-8", binary_to_list(UtfMsg)));
decode(Enc,M,_) ->
    Msg = if is_binary(M) -> binary_to_list(M); true -> {error, M} end,
    list_to_binary(iconv:convert(?ENCODING_NAME(Enc), "UTF-8", Msg)).
```

Предусловия были ослаблены с конкретного типа кодирования до обобщенного, то к предусловиям предъявляется меньше требований.
А вот постусловия были усилие с "декодированного сообщения", до "бинарное репрезентация сообщения" (в Erlang'e это всегда список битов).
К тому же были обработаны вариантЫ, когда дан некорректный тип декодирования.

Стало:
```erl
// Pre-cond: given desired decode type, message to decode and udhLen
// Post-cond: returned binary representation of decoded message
// If given incorrect decode type, returns error with given M
decode('7bit',M,UdhLen) ->
    BinMsg = if is_binary(M) -> M; true -> list_to_binary(M) end,
    LeadingCount = case UdhLen rem 7 of
                       0 -> -1;
                       L -> L
                   end,
    M1 = add_leading_zeros(LeadingCount,BinMsg),
    ClosingCount = case size(M1) rem 7 of
                       0 -> -1;
                       C -> 7-C
                   end,
    M2 = add_closing_zeros(ClosingCount,M1),
    M3 = from_7bit(M2),
    M4 = cut_off_closing_zeros(M3),
    cut_off_leading_zeros(LeadingCount+1,M4);
decode(gsm,M,_) ->
    {_, UtfMsg} = gsm_to_utf16(if is_binary(M) -> M; true -> list_to_binary(M) end),
    list_to_binary(iconv:convert("UTF-16BE", "UTF-8", binary_to_list(UtfMsg)));
decode(Enc,M,_) ->
    Msg = if is_binary(M) -> binary_to_list(M); true -> M end,
    list_to_binary(iconv:convert(?ENCODING_NAME(Enc), "UTF-8", Msg)).
```

# Вывод


