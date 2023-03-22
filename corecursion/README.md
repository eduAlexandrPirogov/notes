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

Полный код и gist --> :
