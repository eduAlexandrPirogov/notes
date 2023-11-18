# Ошибка модульного рассуждения

# Пример 1
Первое, что пришло в голову, так это код на Erlang'e, где я постоянно дергаю коллег, чтобы понять, что может случиться при вызове функции.

```erl
process_dr(Pdu) ->
    Msg = params:get(short_message,Pdu),
    [
        Id,
        DoneDate,
        Stat,
        DrErrorCode
    ] = case re:run(Msg,"^id:([^\\s]+) sub:.* done date:(\\d+) stat:(\\w+) err:(\\d+)",
        [global,{capture,all_but_first,list}]) of
            {match,[List]} ->
                List;
            _ ->
                [undefined, make_dr_submit_date(os:timestamp()), undefined, 0]
        end,
    NewId = params:get(receipted_message_id, Id, Pdu),
    State = message_state_to_atom(params:get(message_state, dr_string_to_state(Stat), Pdu)),
    NEC =
        case params:get(network_error_code,Pdu) of
            {network_error_code, _, ErrorCode} when ErrorCode > 0 ->
                [{error_code, ErrorCode}];
            _ ->
                try
                    IntCode = list_to_integer(DrErrorCode),
                    if
                        IntCode > 0   -> [{error_code, IntCode}];
                        true          -> []
                    end
                catch
                    _ -> [];
                    _:_ -> []
                end
        end,
    %% MCC+MNC от поставщика Infobip
    MCCMNC = case params:get(5142,Pdu) of  %% <-------------<-------------<-------------<-------------<-------------<-------------< Почему используется именно 5142, какой семантический смысл и как это влияет на вызов функции
                 B when is_binary(B) ->
                     [{mccmnc,lists:filter(fun(0)->false;(_)->true end,binary_to_list(B))}];
                 _ ->
                     []
             end,
    %% Идентификатор оператора NetworkId от поставщика Infobip
    NetworkId = case params:get(5130,Pdu) of %% <-------------<-------------<-------------<-------------<-------------<-------------< Почему используется именно 5130, какой семантический смысл и как это влияет на вызов функции
                    Bin when is_binary(Bin) ->
                        <<NetId:32, _/bits>> = Bin,
                        [{network_id, integer_to_list(NetId)}];
                    _ ->
                        []
                end,
    [{id,NewId}, {done_date,DoneDate}, {state,State}] ++ NEC ++ MCCMNC ++ NetworkId.
```

Я выделил две строки кода, где присутствуют литералы. Проблема, с которой столкулся несколько месяцев назад, что некоторые корректные по PDU-шки могли возвращать отличное от ожидаемого значения.
Как выяснилось, константы означали ключи в мапе, с которой работали другие процессы и один из процессов удалял ключи 5130 и 5142. Проблема заключалась в том, что поиск во всем проекте не помог и 
к мапе другие процессы обращались через зарегестрированное имя. Потребовалось дергать коллег и вникать в суть проблемы в деталях. 

# Пример 2

```go
// Parse given string and unmarshal into given T instance
// Firstly it parses Header of CSV file and returns error "header was created"
// Then Parse Unmarshal given text to T instance
//
// Pre-cond: given string
//
// Post-cond: unmarshal given text. Return instance and error status
func (c *CSV[T]) Parse(text string) (T, error) {

	var fail T
	var success []T
	if c.Header == nil {
		c.Header = []byte(text)
		return fail, fmt.Errorf("header was created")
	}

	toUnmarshal := c.buildCSVInput(text)
	err := csvutil.Unmarshal(toUnmarshal, &success)
	if err != nil {
		return fail, err
	}
	return success[0], nil
}

func (c *CSV[T]) buildCSVInput(text string) []byte {
	toUnmarshal := make([]byte, 0, len(c.Header)+len(text)+1)
	toUnmarshal = append(toUnmarshal, c.Header...)
	toUnmarshal = append(toUnmarshal, '\n')
	toUnmarshal = append(toUnmarshal, []byte(text)...)
	return toUnmarshal
}
```

Суть программы заключалась в построчном чтении CSV файла и демаршализации строк в структуру. Если демарашлизация успешно, возвращалась структура и error = nil.
Метод Parse(text string) (T, error) рассчитан на то, что некоторый цикл извне будет вызывать его (и решение удачное).
Проблема заключается в том, что для более широких значений (например CSV файл не имеет первую строку-заголовок, а лишь значения) этот алгоритм будет работать некорректно.
Или если заголовок может состоять из нескольких строк.
То есть возможны набор значений, при  которых предусловия в целом верны, но программный компонент ошибочен.


# Пример 3


```go
// Read property of CounterState for given value
//
// Pre-cond: given existing key
//
// Post-cond: returns corresponding value and true, otherwise empty string and false
func (g GaugeState) GetField(key string) (interface{}, bool) {
	switch key {
	case "name":
		return g.Name, true
	case "type":
		return g.Type, true
	case "value":
		if g.Value == nil { <-------------------------------------------------------------
			return nil, false
		}
		return g.Value, true
	case "hash":
		return g.Hash, true
	default:
		return "", false
	}
}
```

GaugeState содержит в себе мапу, которая в качестве значения содержит пустой интерфейс. Обратите внимание на выделенную стрелкой строку кода.
Что если GaugeState может хранить в качяестве значения nil? То есть ключ имеется и по ключу значение = nil. 
В данном случае при  корректных спецификациях метод вернет флаг false, мол значение не присутствует, когда это ложно :)

# Вывод

Думал ещё привести примери с PHP кодом, но динамическая типизация сама по себе штука небезопасная и стоит быть готовым к излишнему изучению реализации модуля.  А если ещё и слабая типизация, то дела обстоят ещё "веселее".
С Erlangoм, где можно возвращать разные значение (например, кортеж и список) разработка становится по большей части изучения кода в деталях, поскольку нужно написать все сопоставления для функции-клиента.

Говоря про статические языки с сильной типизацией, то, как мне видится, во избежание ошибок модульного рассуждения, стоит прибегнуть к:
1) Уточнению, происходит ли конвертация одного типа в другой в программном компоненте. В зависимости от используемого инструмента конвертации могут быть различные результаты (например, строка с вещественным числом как будет конвертирована в float32 и т.п.)
2) Задавать конкретную область определения (как в случае со вторым примером).

В счет программных компонентов, которые обращаются во внешний мир не беру, так как это, как мне кажется немного другая тема для рассуждений.
А в целом, помимо "предусловия верны -> пост условия верны", "предусловия не верны -> пост условия не верны", нужно брать в учет "предусловия не верны -> пост условия верны". Такое можно проверить fuzzing для примитивных типов данных. 

Ещё, как мне кажется, определить ошибку модульного рассуждения можно следующим образом: при изучении реализации используемого программного компонента, мы задаемся вопросом "что" было использовано (декларативно), а не "как/почему" (императивно).
