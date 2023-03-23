# Ко-рекурсивные программы

Пример 1.

Имеется функция, на вход которой подается данные футбольные статистик. Нужно сгруппировать их по сезону.
И выдать сгруппированный список
Было:
```go

type Stats struct {
	Season  string
	Age     string
	Squad   string
	Country string
	Comp    string
	PlayingTime
	Perfomance
	Expected
	Progression
	Per90Minutes
}

func grpByYears(records []stats.Stats) []stats.Stats {
	var res = make([]stats.Stats, 0)
	currSeason := ""
	var grpdHead = make([]stats.Stats, 0)
	for i := 0; i < len(records); i++ {
		if currSeason != "" && records[i].Season != currSeason {
			res = append(res, grpdHead[0])
			grpdHead = make([]stats.Stats, 0)
			currSeason = ""
		} else {
			if len(grpdHead) == 0 {
				grpdHead = append(grpdHead, records[i])
			} else {
				grpdHead[0] = grpdHead[0].Add(records[i])
			}
			currSeason = grpdHead[0].Season
		}

	}
	return res
}
```

Перепишем это в ко-рекурсивную программу. Подумаем как со стороны рекурсивной, так и со стороны ко-рекурсивной

1) Со стороны рекурсивной сформируем шаблон функции, пропустив первые три шага
Если получаем пустой список, то возвращаем пустой список
Если получаем список, обрабатываем голову и вызываем функцию для оставшегося списка
Шаблон:
```
fun ([]) -> []
fun ([H|T]) -> ... H ... fun (T)
```

Со стороны ко-рекурсивной программы
1) Когда выход пуст? Когда вход пуст
2) Если выход не пуст, чему равна голова списка? Агреггированной по году элементом stats.Stats
3) Из каких данных рекурсивно формируется хвост? Из элементов, не агрегиррованных по году

Шаблон:
```
fun list | null list = []
          | ... x ... fun y
          where x = head of list and y tail
```

Перенесем все это в код:
Стало:
```go

func grpByYears(records []stats.Stats) []stats.Stats {
	//Когда выход пуст? Когда вход пуст
	if len(records) == 0 {
		empty := make([]stats.Stats, 0)
		return empty
	}
	// Если выход не пуст, чему равна голова списка? Агреггированной по году элементом stats.Stats
	// Из каких данных рекурсивно формируется хвост? Из элементов, не агрегиррованных по году
  
	head, tail := split(records) // split -- порождающая рекурсия
	return append(head, grpByYears(tail)...)
}

func
```

Также рассмотрим порождающую рекурсию split:

Было:
```go

func split(head stats.Stats, tail []stats.Stats) ([]stats.Stats, []stats.Stats) {
	// Когда выходы пусты? Когда tail пуст
	if len(tail) == 0 {
		headAsList := []stats.Stats{head}
		return headAsList, tail
	}
	// Если вход не пуст, как формируется значение головы? Аггрегируется head с текущей головой
	// Из каких данных рекурсивно формируется хвост? Из элементов, не агрегиррованных по году

	var head = make([]stats.Stats, 0)
	var aggrStats = stats.Stats{}
	var tail = make([]stats.Stats, 0)
	var currSeason = ""
	for k, v := range records {
		if v.Season != currSeason && currSeason != "" {
			tail = append(tail, records[k:]...)
			head = append(head, aggrStats)
			break
		}
		aggrStats = aggrStats.Add(v)
		currSeason = v.Season
	}

	return head, tail
}
```

Рассмотрми с двух сторон: program follows input structure and program follows output structures

Общий шаблон для program follows input structure:
```
fun val, [] -> val, []
fun val, [H|T] ->  ...h(val, H) ... fun(T)
```

Шаблон для  program follows output structures:

```
// Когда выходы пусты/вычисление завершается?
// 1) Когда tail пуст (null list) и 2) когда сезон отличается от головы списка и при этом он не пустой (val == x && val != null)
fun val, list | null list | (val == x && val != null) -> []
          | ... val x ... fun y
          where x = head of list and y tail, val -- some value
```

Получилось следующее:
```go
func split(head stats.Stats, tail []stats.Stats) ([]stats.Stats, []stats.Stats) {
	// Когда выходы пусты/вычисление завершается?
	// 1) Когда tail пуст и 2) когда сезон отличается от головы списка и при этом он не пустой
	if len(tail) == 0 || (head.Season != tail[0].Season && head.Season != "") {
		headAsList := []stats.Stats{head}
		return headAsList, tail
	}

	// Если вход не пуст и сезоны не отличаются, как формируется значение головы?
	// Аггрегируется head с текущей головой
	head = head.Add(tail[0])

	// Из каких данных рекурсивно формируется хвост? Из элементов, не агрегиррованных по году
	return split(head, tail[1:])
}
```

Полный код и gist --> : https://gist.github.com/eduAlexandrPirogov/eb1b870ea1cc30d4d3bf450c0709662d


========================================================================================================================
========================================================================================================================

Пример 2.

Преобразование объектов Row, хранящий строки CSV файла в объект Seeds, отражающий сиды.
Формируются строки только содержащие значение value.

Было:
```go
func (r *Relation) Search(value string) []Seed {
	var res = make([]Seed, 0)

	for _, v := range r.Rows {
		for _, str := range v {
			if str == value {
				res = append(res, v.ToSeed())
				continue
			}
		}
	}
	return res
}
```

```go

func (r *Relation) Seeds(value string) []Seed {
	return r.recSeeds(r.Rows, value)
}

func (r *Relation) recSeeds(tail []Row, value string) []Seed {
	//Когда выход пуст? Когда вход пуст
	if len(tail) == 0 {
		return []Seed{}
	}
	// Если выход не пуст, как формируется голова? Голова формирует сиды из головы строк, которые имеют value
	head, tail := makeSeed([]Seed{}, tail, value)
	// Из каких данных рекурсивно формируется хвост? Из оставшихся строк
	return append(head, r.recSeeds(tail, value)...)
}

func makeSeed(head []Seed, tail []Row, value string) ([]Seed, []Row) {
	//Когда выход пуст? Когда вход пуст
	if len(tail) == 0 {
		return head, tail
	}
	// Если выход не пуст, как формируется голова? Row, содержащий value
	// Из каких данных формируется рекурсивно хвост? Хвост равен оставшимся строкам
	if isContainValue(tail[0], value) {
		head = append(head, tail[0].ToSeed())
	}
	return makeSeed(head, tail[1:], value)
}

func isContainValue(tail Row, value string) bool {
	// Когда выход ложен?
	// Когда вход пуст
	if len(tail) == 0 {
		return false
	}
	// Когда выход истинен?
	// Когда значение головы == value
	if tail[0] == value {
		return true
	}
	// Если выход не пуст и значение головые не равно value,  как формируется голова? Никак, проверяем оставшийся список
	// Из каких данных формируется рекурсивно хвост? Хвост равен оставшимся строкам
	return isContainValue(tail[1:], value)
}

```

========================================================================================================================
========================================================================================================================

Пример 3.

Было:
```go
// Функция аггрегирует пользователей с суммарными сообщениями за день
// Возвращает пользователя с максимальным количеством сообщений
func UsersWithMaxMsgs() []UserMsgStats {
	res := make([]UserMsgStats, 0)
	for _, v := range s.Users {
		groupedUsers := MaxMsgsByDate(v)
		maxUser := Max(groupedUsers)
		res = append(res, maxUser)
	}
	return res
}

// Формирует карту с аггрегироваными пользователям и сообщениями по дате
func MaxMsgsByDate(users []UserMsgStats) map[string]UserMsgStats {
	user := make(map[string]UserMsgStats, 0)
	for _, v := range users {
		if v, ok := user[v.Name]; !ok {
			user[v.Name] = v
		} else {
			user[v.Name] = user[v.Name].AddMsgs(v)
		}
	}
	return user
}

// Возвращает пользователя с максимальным количеством сообщений
func Max(groupedUsers map[string]UserMsgStats) UserMsgStats {
	max := UserMsgStats{}
	for _, v := range groupedUsers {
		if max.MsgCount < v.MsgCount {
			max = v
		}
	}
	return max
}

```

Стало:
```go

// map[time]list -> [{date, u1, msgc1}, {date, u2, msgc2}, {date, u3, msgc3}, ...]
// Когда список пуст? Когда входная структура данных пуста

// Если список не пуст, как формируется голова? Вставляется пользователь с аггрегированными сообщениями
// при этом, проверяется его количество сообщений с уже существующими пользователями

// Как аггрегируются сообщения? Для каждого пользователя по дате считается суммарное количество сообщений
// На выходе выдается пользователь с наибольшим кол-во сообщений

// Как рекурсивно формируется хвост списка? По мапе не получится сделать хвост
// Преобразовать ключи в список и рекурсивно пройтись по ним

func topUsers(stats map[time.Time][]UserMsgStats) []UserMsgStats {
	//Преобразовываем в рекурсивную структуру данных
	keys := make([]time.Time, 0, len(stats))
	for k := range stats {
		keys = append(keys, k)
	}
	return recTopUsers(keys)
}

func recTopUsers(dates []time.Time) []UserMsgStats {
	// Когда выход пуст? Когда вход пуст
	if len(dates) == 0 {
		return []UserMsgStats{}
	}
	// Если выход не пуст, как мы формируем голову? Аггрегируем данные за дату
	// Из каких данных формируется рекурсивно хвост? Оставшийся список
	head, tail := MaxMsgs(dates[0]), dates[1:]
	return append([]UserMsgStats{head}, recTopUsers(tail)...)
}

func MaxMsgs(date time.Time) UserMsgStats {
	users := make(map[string]UserMsgStats)
	return UserMaxMsgs(users, s.Users[date])[0]
}

func UserMaxMsgs(users map[string]UserMsgStats, tail []UserMsgStats) []UserMsgStats {
	// Когда завершаем вычисления? Когда список пуст
	if len(tail) == 0 {
		//Тут проще будет сделать циклом через мапу, хотя можно также преобразовать в список
		max := UserMsgStats{}
		for _, v := range users {
			if max.MsgCount < v.MsgCount {
				max = v
			}
		}
		return []UserMsgStats{max}
	}

	// Если список не пуст, как формируем голову? Берем голову списка
	// Из каких данных рекурсивно строится хвост? Из оставшейся части списка
	head, tail := tail[0], tail[1:]
	if _, ok := users[head.Name]; !ok {
		users[head.Name] = head
	} else {
		users[head.Name] = users[head.Name].AddMsgs(head)
	}
	return UserMaxMsgs(users, tail)
}
```

========================================================================================================================
========================================================================================================================

Новый способ построения программ, ко-рекурсивных, приятно удивил. Простая вещь.
Из оригинальной работы How to Build co-programms взял на заметку способ формирования шаблона функций, когда мы описываем
шаблон как "программа следует входным данным" и как "программа следует выходным данным". 

С данным способом решил несколько задач на codewars, которые не мог решить смотря с ракурса "ко-рекурсивных программ".
