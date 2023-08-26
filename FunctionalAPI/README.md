# Функциональные интерфейсы (с мутабельностью).

Очень давно не писал на ООП языках, нашел примеры лишь из старых проектов.
Даже в Go стараюсь на автомате писать функциональные API для тех или иных пакетов.

Например, последние несколько проектов включают в себя функциональное ядро. Ядро одного из проекта:
Пример 1
```go
// Package kernel handles job with MetricsState
// It is wrote with declarative paradigm
package kernel

import (
	"log"
	"memtracker/internal/kernel/tuples"
)

type Replicator interface {
	Write(data []byte)
}

type Storer interface {
	Write(tuple tuples.TupleList) (tuples.TupleList, error)
	Read(cond tuples.Tupler) (tuples.TupleList, error)
	Ping() error
}

// Write writes tupler to DB and return written state
//
// Pre-cond: given tuples to Write in given Storer
//
// Post-cond: if was written successfully returns NewTuple state and error nil
// Otherwise returns emtyTuple and error
func Write(s Storer, states tuples.TupleList) (tuples.TupleList, error) {
	newStates, err := s.Write(states)

	if err != nil {
		log.Printf("err while writing state %v", err)
		return tuples.TupleList{}, err
	}

	return newStates, nil
}

func Read(s Storer, state tuples.Tupler) (tuples.TupleList, error) {
	states, err := s.Read(state)
	if err != nil {
		log.Printf("err while reading state %v", err)
		return tuples.TupleList{}, err
	}

	return states, nil
}

// Ping checks is Storer is alive. Health check.
//
// Pre-cond: given Stoerer
//
// Post-cond: makes HealthCheck for given Storer.
// If alive -- return nil, otherwise returns error
func Ping(s Storer) error {
	return s.Ping()
}
```

Или вот еще один пример функционального API для работы с tuples. 
```go
package tuples

// ExtractString extracts string field from tuple
//
// Pre-cond: given field to extract and tuple to extract from
//
// Post-cond: extracts string value of fields.
// If field no exists or field is not string, return empty string
func ExtractString(field string, t Tupler) string {
	f, ok := t.GetField(field)
	if !ok {
		return ""
	}

	return f.(string)
}

// ExtractString extracts pointer to int64 field from tuple
//
// Pre-cond: given field to extract and tuple to extract from
//
// Post-cond: extracts pointer to int64  value of fields.
// If field no exists or field is not pointer to int64 , return nil
func ExtractInt64Pointer(field string, t Tupler) *int64 {
	f, ok := t.GetField("value")
	if !ok {
		return nil
	}
	return f.(*int64)
}

// ExtractString extracts pointer to float64 field from tuple
//
// Pre-cond: given field to extract and tuple to extract from
//
// Post-cond: extracts pointer to float64  value of fields.
// If field no exists or field is not pointer to float64 , return nil
func ExtractFloat64Pointer(field string, t Tupler) *float64 {
	f, ok := t.GetField("value")
	if !ok {
		return nil
	}
	return f.(*float64)
}
```


Попробуем переделать участки кода из старых проектов:

Пример 1
Было:
```cpp
class JsonContainerBuilder
{
public:
	JsonContainerBuilder();

	void parseFile(std::string& file_name);
	void appendKeyValue(std::string& key, std::string& value);


	JsonContainer* get_jsonContainer();
	MongoCollectionsContainer* getMongoCollectionsContainer();

	~JsonContainerBuilder();
private:

	Parser* parser = nullptr; // Äëÿ ïàðñèíãà ôàéëà
	JsonContainer* jsonContainer = nullptr; // Äëÿ õðàíåíèÿ ôàéëîâ
	MongoCollectionsContainer* mongoCollectionsContainer = nullptr; // Êîëëåêöèè äëÿ ñåðèàëèçàöèè è äåñåðèàëèçàöèè
};
```

Сделаем API класса, как функциональный API.
Стало:
```cpp

// API
JsonContainer parseFile(std::string& file_name, container JsonContainerBuilder);

JsonContainer appendKeyValue(std::string& key, std::string& value, container JsonContainerBuilder);

JsonContainer get_jsonContainer(container JsonContainerBuilder);

MongoCollectionsContainer getMongoCollectionsContainer(container JsonContainerBuilder);

// Это можно сделать также структурой=
class JsonContainerBuilder
{
public:
	JsonContainerBuilder();
	~JsonContainerBuilder();

	Parser parser;
	JsonContainer jsonContainer;
	MongoCollectionsContainer mongoCollectionsContainer;
};
```

Пример 2
Было:
```cpp
class Arguments
{
    protected:
        std::vector<std::string> args;
    public:
        Arguments() = default;

        /**
         * Pre-conditions: given container of inputed arguments for command
         * Post-conditions: created instance of Arguments
        */
        Arguments(const std::vector<std::string>& givenArgs);
        
        
        /**
        * Pre-conditions: given an index > 0 and < container.size()
        * Post-conditions: returns value stored at index
        */    
        inline const std::string& at(int index) noexcept
        {
            return args[index];
        }


        /**
        * Pre-conditions: 
        * Post-conditions: validting given arguments;
        */
        virtual bool validate()
        {
           for(auto s : args.args) {
              if (!v::validateStr(s)) {
                return false
              }
           }
            return true;
        };


        /**
        * Pre-conditions:
        * Post-conditions: return count of given arguments
        */
        const size_t size() const;

        std::string& operator[](int index)
        {
            return args[index];
        };
};
```

Метод size() мы можем получить напрямую обратившись к вектору.
Переопределения оператора нам ни к чему, поэтому останется вынести пару функций.

Стало:
```cpp

class Arguments
{
    public:
        std::vector<std::string> args;
        Arguments() = default;
};

 /**
* Pre-conditions: given Arguments and given an index > 0 and < container.size()
* Post-conditions: returns value stored at index
*/    
const std::string at(int index, Arguments args )
{
  if (index < 0 || args.args.size() < index) {
    return "";
  }
  return args.args.at(index);
}

/**
* Pre-conditions: given Arguments 
* Post-conditions: validting given arguments;
*/
bool validate(Arguments args)
{
    for(auto s : args.args) {
      if (!v::validateStr(s)) {
        return false
        }
     }
     return true;
};

```

Пример 3
Было:
```cpp
class Command
{
public:

    /**
     * Pre-conditions: given an RequestHandler pointer 
     * Post-conditions: Command will be executed
    */
    int execute(RequestHandler* handler);

    /**
     * Pre-conditions:
     * Post-conditions: validates inputed arguments of commands
    */
    int void validateArgs();

    /**
     * Pre-conditions:
     * Post-conditions: return string representation of command
    */
    string strForPrint();
};
```

Стало:


```cpp

 /**
* Pre-conditions: given an RequestHandler pointer and Command
* Post-conditions: Command will be executed
*/
int execute(RequestHandler* handler, c Command);

/**
* Pre-conditions: given Command
* Post-conditions: validates inputed arguments of commands
*/
int void validateArgs(c Command);

/**
* Pre-conditions: given Command
* Post-conditions: return string representation of command
*/
string strForPrint(c Command);

class Command {};
```

По итогу:
Если у меня имеется возможность писать функциональный API, то я его реализую. Но тут надо учитывать следующие факторы:
1) возможность реализовать API в конкретном ЯП
2) влияние на память реализации функционального API

О первом пунтке. Если говорить о Go, то там это происходит легко и естественно. Но вот применение такого подхода в C++ для меня стало проблемным, основная из которых -- экспорт. 
Мне приходиться делать публичными поля класса, чтобы получить доступ в функциональном стиле. В Go же я могу получить доступ к неэкспортируемым полям структуры внутри пакета.
Насколько эта проблемой является проблемой? Как мне кажется, в плюсах все функции будут экспортируемы. Да, можно создать namespace, отдельные файлы, но это выглядит как по мне 
слишком костыльно. То есть, данный метод желательно применять в соответствии с ЯП и его возможностями относительно экспорта.

О втором пунтке. Если говорить про память, то тут куда ситация куда интереснее. Если не используем какие либо контейнеры, а относимся к структурам, как к кортежам, то у нас вообще не будет узкий мест.
Данное утверждение я проверял на профайлере в Go, где ядро было функциональным. Разницы между "передачей по указателю" и фунцкиональным подходом не обнаружилось. 
Но вот ситуация с контейнерами куда интереснее. Например, так выглядит файл в одном из проектов реализации списка (приведен в самом низу). 
Хоть и стиль тут не функциональный, но по сути, мы каждый раз передаем новую версию списка, а разговор именно о копировании. 
Дело в том, что тут ситуация немного хуже и профайлер показал узкое место именно здесь в связи с постоянным копированием при стресс-тестах. 
Вопрос таков: стоит ли писать функциональную API для контейнеров? И да (из-за простоты тестирования и природы иммутабельных данных (не совсем чисто иммутабельных, а в данном контексте),
и нет, поскольку это дорого бьет по памяти.)
В Go есть варианты решения проблемы памяти, как персистентные структуры данных:
1) https://pkg.go.dev/golang.org/x/tools/internal/persistent
2) https://github.com/tobgu/peds

но ни разу не использовал, все как-то руки не доходят.

Для себя у меня стратегия такая относительно функционального API: если работаю не с контейнером и присутствует корректный механизм экспорта единиц программного кода, то пишу функциональный API,
иначе пишу императивный. Если разберусь, как в GO реализуются персистентные данные и действительно ли они эффективно, то требование о контейнерах отпадет.

```go
package tuples

import "encoding/json"

// TupleList is a container of instance that dispatches Tupler interface
type TupleList struct {
	tuples []Tupler
}

// Head return first elem of TupleList. If there is no elems return nil
//
// Pre-cond:
//
// Post-cond: if TupleList isn't empty returns first element of list
// Otherwise returns nil
func (t TupleList) Head() Tupler {
	if len(t.tuples) == 0 {
		return nil
	}
	return t.tuples[0]
}

// Head Tail returns new TupleList except of it head
//
// Pre-cond:
//
// Post-cond: returns new TupleList except of head if there are any elements in tail
// Otherwise returns empty TupleList
func (t TupleList) Tail() TupleList {
	if !t.Next() {
		return TupleList{}
	}
	t.tuples = t.tuples[1:]
	return t
}

// Next tells if there are elems in list
//
// Pre-cond:
//
// Post-cond: if TupleList has more than 0 elems returns true
// Otherwise returns false
func (t TupleList) Next() bool {
	return len(t.tuples) > 0
}

// Len return count of elems in TupleList
//
// Pre-cond:
//
// Post-cond: returns count of elems in TupleList
func (t TupleList) Len() int {
	if t.tuples == nil {
		return 0
	}
	return len(t.tuples)
}

// Add adds elem to TupleList
//
// Pre-cond: given Tupler element
//
// Post-cond: elem was added to TupleList. Returns new TupleList with given elem
func (t TupleList) Add(elem Tupler) TupleList {
	if t.tuples == nil {
		t.tuples = make([]Tupler, 0)
	}
	t.tuples = append(t.tuples, elem)
	return t
}

// HeadTail returns head and tail of TupleList
//
// Pre-cond:
//
// Post-cond: returns head and rest TupleList
func (t TupleList) HeadTail() (Tupler, TupleList) {
	return t.Head(), t.Tail()
}

// Merge merges two TupleLists into one
//
// Pre-cond:
//
// Post-cond: returns new TupleList that has elems of merging Tuplelists
func (t TupleList) Merge(toMerge TupleList) TupleList {
	t.tuples = append(t.tuples, toMerge.tuples...)
	return t
}

// AsSlice returns slice of elems in TupleList
//
// Pre-cond:
//
// Post-cond: returns slice of elems in TupleList
func (t TupleList) AsSlice() []Tupler {
	return t.tuples
}

// MarshalTupleList marshals all elems in TupleList
//
// Pre-cond: given TupleList and acc to store result
//
// Post-cond: returns marshaled elems
func MarshalTupleList(tail TupleList, acc []byte) []byte {
	slice := tail.AsSlice()
	if len(slice) == 1 {
		body, _ := json.Marshal(slice[0])
		return body
	}

	body, _ := json.Marshal(slice)
	return body
}
``` 
