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

Мне импонирует нарратив, что:
1) код должен следовать спецификации
2) взаимосвязь между спецификацией и реализацией есть ключ в разработке антихрупкой ПО

Спецификации, хотя бы в виде пред и пост-условий, давно для меня стали естесвенной вещью в разработке. Совместно с тестами, выкат реализованных мною фич в прод с багами невероятно мал.
В первом занятии мы выяснили, что иметь некорректные предусловия при которых корректны пост-условия не позволительно.
Тут же выявилась важность в виде рекурсии [пред-условия|кусок кода|постусловие|пред-условие|кусок кода...]. Сама догма, которая говорит об влиянии послаблении и усилении пред/пост-условий понятна -- она очень похожа на LSP, но есть нюанс.
LSP в ООП смысле больше про то, чтобы составлять корректную иерархию (корректную, в смысле антихрупкости), где не нарушаются ослабления/усиления пред и пост-условий.

В данном же случае речь не столько про пред и пост-условия, сколько про области определений и значений, так как в первую очередь, набор компонентов, собранных с помощью композиции, представляет собой поток вычисления.
И тема с "наиболее слабое пред-условие -- наиболее сильное пост-условие" противополжна моему виденью разработки и противоположна она не в смысле "хорошее--плохое", а в более интересном.

Как создание программного кода мне видится сейчас и как я его применяю: создаю некоторую алгебру из элементарных операндов и операторов, расширяю ее и собираю компоненты с помощью элементов моей алгебры. Для простоты возьмем в пример сущность "функция".
Функция что-то принимает и отдает. Функция работает на какой-то области определений (пред-условия). Функция выдает четко определенное значение, которое соответствует пост-условию. Идея с усилением пост-условием мне нравится -- мы требуем получить как можно более конкретный и детерменированный результат.
А вот с областью определений я также считаю, что он должен быть как можно более конкретным (см. пример 3, где я использую не string, а четко определенный тип Extension, то есть требования выше). То есть мой нарратив таков, что чем более конкретные спецификации, тем лучше.

Теперь проведу пример-аналогию между данным материалом  и моим виденьем касательно спецификаций.
Представим, что мы создаем механизм для фильтрации воды. Вода фильтруется в несколько этапов (очищается от камней на одном фильтре, от песка на другом фильтре и так далее в виде пайплайна).

В соответствие с моим виденьем, я бы создавал API фильтра только для воды с определенными загрязнениями. Это требует меньше кода, так как для других жидкостей пред-условие не будет истинным.
Для моего механизма будет пред-условие "дана вода с камнями и песком", фильтру для очистки от камня будет дано предусловие "дана вода с камнями", а фильтру очистки песка будет дано предусловие "дана вода с песком".

Если посмотреть на эту же ситуации после изучения материала, мне это кажется следующий образом. Мы создаем первый фильтр с самым слабым пред-условием "дана жидкость". И уже инкапсулированная часть фильтра "отфильтровывает" все ненужное.
Затем отфильтрованное переходит на следующий этап и так далее.


В чем же разница между двумя подходами? 
Во-первых, спецификации: в первом случае дана конкретная вода (больше требований), а во-втором случае -- жидкость (меньше требований).
Во-вторых, влияние спецификация на реализацию (не забываем, что код должен следовать спецификации!):
   а) В-первом случае, я создаю более прямолинейный поток вычислений с конкретными входными и выходными значениями, где "нюансы" возникнуть не могут! Меньше компоненты программы, но их количество будет больше в будущем.
   б) Во-втором случае, все нюансы будут учитываться внутри программного компонента. Размер компонента программы больше, но они менее строгие, более полиморфны (кстати этот материал очень сильно перекликается в моем уме с материалом про полиморфный код).

В Erlang'e я настрадался в свое время от слабых пред-условий, которые ограничиваются либо ошибкой function clause, либо изучением цепочки вычислений, чтобы понять, что нужно передать функции. Возможно из-за этого и ослабление к пред-условием выглядит для меня пока что отталкивающим.
А вот касательно пост-условий, то тут я всецело согласен, что нужно получать на выходе настолько конкретный результат, насколько это возможно.
