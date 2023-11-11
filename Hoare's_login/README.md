# Логика Хоара для программистов

# Пример 1

Начнем с простого. Пакет-аналог std плюсов:
```go
// std hold interfaces for all data structures that used in this project

package std

// Iterator incapsulate movement inside data structure elements in one way
type Iterator[T any] interface {
	// Next tells is there are elements to iterate
	//
	// if there are elements to iterate then return true
	// otherwise returnes false
	Next() bool

	// Curr returnes current element of iterator
	Curr() T
}

type Linked[T any] interface {
	// Add adds given elem at the tail
	//
	// Pre-cond: given element to add
	//
	// Post-cond: list's tail now is equal to given element
	PushBack(item T)

	// PopFront removes head of the list and returns item
	//
	// Pre-cond:
	//
	// Post-cond: if list isn't empty then returns value of head and true
	// and moves head to the next elem
	// Otherwise returns default value and false
	PopFront() (T, bool)

	// Iterator creates new instance of Iterator to inspect elements
	//
	// Pre-cond:
	//
	// Post-cond: returned instance of iterator
	Iterator() Iterator[T]
}

// AsSlice put all data structure's elemes into slice
//
// Pre-cond: given data structure's iterator
//
// Post-cond: returned slice of data structure's elements
func AsSlice[T any](i Iterator[T]) []T {
	res := make([]T, 0)
	for i.Next() {
		res = append(res, i.Curr())
	}
	return res
}
```

Обратите внимание на сигнатуру функции AsSlice. 
Используется данный интерфейс повсеместно, и если нужно получить слайс с помощью AsSlice, то код выглядит так:

```go
//l -- полученный инстанс, удовлетворяющий интерфейсу std.Linked
slice := std.AsSlice(l.Iterator())
```

Да, получить элементы контейнера в виде слайса -- задача простая, но моей целью было сделать так, чтобы функцию AsSlice и все контейнеры std 
нельзя было использовать неправильно! 

В соответствие с пост-условиями получается следующее:
1) Участок AsSlice исполняется лишь тогда, когда передается Iterator
2) Результат будет получен лишь после удовлетворения пунтка 1

Но есть нюанс, который заключается в реализации инстанса Iterator'a. Да, спецификации для интерфейса имеются, но криво написанный итератор может вернуть
пустой слайс непустого контейнера. Таким образом, спецификации для AsSlice можно написать следующим образом:

```go
// AsSlice put all data structure's elemes into slice
//
// Pre-cond: given data structure's iterator
//
// Post-cond: returned data structure representation as slice   <--------
func AsSlice[T any](i Iterator[T]) []T {
	res := make([]T, 0)
	for i.Next() {
		res = append(res, i.Curr())
	}
	return res
}
```

# Пример 2

А вот неудачный пример:

```go
// NewWriter writes data every time when channel got new data
//
// Pre-cond: given file to write data
//
// Post-cond: data written to the file
func (j *Journal) WriteSynch(file *os.File) {

	j.mux.Lock()
	defer j.mux.Unlock()
	defer file.Close()
	writer := bufio.NewWriter(file)
	for {
		if bytes, ok := <-j.Channel; ok {
			bytes = append(bytes, '\n')
			writer.Write(bytes)
			writer.Flush()
		} else {
			break
		}
	}
}
```

Во-первых, несмотря на корректное предусловие, возможна ситуация, когда данные могут быть не записаны в файл: 
поскольку происходит взаимодействие с ОС, там могут быть самые различные примеры.
Также может быть в теории передан некорректный дескриптор файла, что тоже в спецификациях не отражается.

Сложность рассуждения о данном участке кода добавляет тот факт, как используется данная функция:

```go
go j.writeSynch(file)
```
где просто передается дескриптор какого-то файла (проверок до этой строки никаких не было).

Получился совершенно хрупкий участок кода, который ломается при любом дуновении ветра...

# Пример 3

```go
const xRealIP = "X-Real-IP"

// IsContainerSubnetHTTP checks if request's ip satisfy's subnet
//
// Pre-cond: Given pointer to not nil http.Request
//
// Post-cond: returns a boolean which tells if requests satisfy subnet
func IsContainerSubnetHTTP(r *http.Request) bool {
  if r == nil {
     return false
  }
	cameIP := r.Header.Get(xRealIP)
	ip, parseErr := netip.ParseAddr(cameIP)
	subnet, parsePrefixErr := netip.ParsePrefix(server.ServerCfg.Subnet)
	return parseErr == nil && parsePrefixErr == nil && subnet.Contains(ip)
}
```

Простая проверка на принадлежность IP-запроса к подсети. 

Используется в middleware:

```go
func SubnetValidate(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if ok := verifier.IsContainerSubnetHTTP(r); !ok {
			w.WriteHeader(http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

Возможна ли ситуация, когда при некорректных пред-условий истинны пост-условия? Нет, так как уже обработан вариант с nil (о чем сказано в спецификациях),
плюс использование хорошо отлаженных готовых библиотек (проверил на тестах).

# Пример 4

# Пример 5
