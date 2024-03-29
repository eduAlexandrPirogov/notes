# Модули важнее всего-2

Пример функционального интерфейса с помощью F-ограниченного полиморфизма.


```go
// You can edit this code!
// Click here and start typing.
package main

import "fmt"

type LinkedStructure interface {
	Push(int)
}

// Functional interface
type Cloner[T any] interface {
	Clone() T
}

//

type stack struct {
	Elems []int
}

func (s *stack) Push(elem int) {
	s.Elems = append(s.Elems, elem)
}

func (s stack) Size() int {
	return len(s.Elems)
}

func (s stack) Clone() stack {
	clon := stack{
		Elems: s.Elems,
	}
	return clon
}

type dict struct {
	Elems map[int]int
}

func (d *dict) Push(elem int) {
	d.Elems[elem] = elem
}

func (d dict) Size() int {
	return len(d.Elems)
}

func (d dict) Clone() dict {
	cloned := dict{Elems: d.Elems}
	return cloned
}

// F-Bounded constraint
func CloneAny[T Cloner[T]](c T) T {
	return c.Clone()
}

func main() {
	s := stack{Elems: []int{1, 2, 3}}
	s1 := CloneAny(s)
	fmt.Println(s1)

	d := dict{Elems: map[int]int{}}
	d.Push(1)

	d1 := CloneAny(d)
	fmt.Println(d1)

}
```

В Go можно лишь эмулировать F-ограничение. В основном это помогает работать с единым типом объекта, который удовлетворяет интерфейсу (а ограничение в этом помогают, но работает это в Go с 1.18),
чтобы вернув экземпляр, мы не проверяли его принадлежность интерфейсу. Вроде штука полезная, но довольно тривиальная -- мы работаем с одним типом объекта. Но есть более интересная ситуация,
где показывает мощь подобной техники, который я не нашел на просторах интернета.

Представим, что у нас имеется иерархия структур данных: стэки, очереди, словари и т.д. Мы хотели бы получить среди множества экземпляров данной иерархии ту, которая имеет наибольший вес 
(в данном случае, пусть будет кол-во элементов, возвращает метод Size() int). Звучит довольно просто, ведь мы можем собрать все экзмепляры в контейнер, который принимает интерфейс и сравнить. Но!
Тут возникает такой момент, что некоторые структуры могут быть вообще несравниваемые, например, они бесконечные. Пример притянут за уши, постараюсь сформулировать формально:
У нас иерархия И1, множество экземпляров {Э}, принадлежащие И1, и вот среди {Э}, мы хотели бы сравнить только те, которые обладают свойством С1.

Можно опять же создать метод, принимающий в качестве параметра И1, нагородить кучу if-ов...но лучше использовать F-Bounded полиморфизм. Суть заключается в том, что мы создаем функциональные 
интерфейс, который (!) отражает свойство нашей иерархии И1. В нашем случае, это будет кол-во элементов, но это можно быть что-угодно. В Go нам достаточно реализовать подобные методы, чтобы
они удовлетворяли интерфейсу, спасибо interface dispatch. Вы можете справедливо заметить, что "можно же нагородить подобных интерфейсов дестяки штук, что на счет SRP?". Так вот, поскольку
мы определяем функциональные интерфейсы, отражающие свойства иерархии, то по сути мы лишь добавляем команды к экземплярам! Они ничего не делают, а лишь возвращают некоторое утверждение об 
экземпляре! 

Конкретный пример:
```go
// You can edit this code!
// Click here and start typing.
package main

import "fmt"

type LinkedStructure interface {
	Push(int)
}

// Мы можем добавить любые свойства, это может быть не Size, а, например,
// фактическая занимаемая память
type Comparer[T LinkedStructure] interface {
	LinkedStructure 
	Size() int
}

// !!! Мы сравниваем ту структуру, котрая может быть сравнима (удовлетворяем Comparer)
func CompareAny[T Comparer[LinkedStructure], S Comparer[LinkedStructure]](c T, c1 S) LinkedStructure {
	fmt.Printf("T %v, S %v", c, c1)
	if c.Size() > c1.Size() {
		return c
	}
	return c1
}

func ReturnNonNil(t1, t2 LinkedStructure) LinkedStructure {
	if t1 == nil {
		return t2
	}
	return t1
}

type Cloner[T any] interface {
	Clone() T
}

func CloneAny[T Cloner[T]](c T) T {
	return c.Clone()
}

// Сравним стак и мапу
func main() {
	s := stack{Elems: []int{1, 2, 3}}
	s1 := CloneAny(s)
	s1.Push(4)

	d := dict{Elems: map[int]int{}}
	d.Push(1)

	d1 := CloneAny(d)
	fmt.Println(d1)

	fmt.Println(CompareAny(&s, &d))

}

type stack struct {
	Elems []int
}

func (s *stack) Push(elem int) {
	s.Elems = append(s.Elems, elem)
}

func (s stack) Size() int {
	return len(s.Elems)
}

func (s stack) Clone() stack {
	clon := stack{
		Elems: s.Elems,
	}
	return clon
}

type dict struct {
	Elems map[int]int
}

func (d *dict) Push(elem int) {
	d.Elems[elem] = elem
}

func (d dict) Size() int {
	return len(d.Elems)
}

func (d dict) Clone() dict {
	cloned := dict{Elems: d.Elems}
	return cloned
}

```

По итогу, мы не просто сравнить экземпяры одной иерархии, но также "отфильтровать" их по свойствам, без всяких if-ов, type assertion и прочего, что уменьшает вложенность кода.
Данный пример можно расширить -- например, мы можем создать контейнер для экземпяров, удовлетворяющий некоторому свойству, создать аналогичую функцию CompareAny, и функционально проходить по
этому контейнеру! Получив экземпляр нам не надо его type assert'ить. 

Конечно, звучит прекрасно, но давайте поговорим о минусах:
1) Нужно встраивать ограничиваемый интерфейс, как в случае с :
```go
type Comparer[T LinkedStructure] interface {
   // Обязательно добавляем LinkedStructure
	LinkedStructure 
	Size() int
}
```
2) Громоздкий синтакс: представим, что нам нужно сравнить структуры данных, которые Comparer, которые Countable, которые... и вот после уже 3-4 таких свойств уместить дженерики на одной строке
не получится
3) Я уверен, что эту идею можно выразить куда-более лучше. Как мне кажется, иерархии различают между собой по свойствам (в программировании -- по инвариантам), и получается, что
иерархии стэков должна быть отделена от иерархии мап. Тут приходит заключение, что реализация и стэка, и мапы может удовлетворять нескольким интерфейсам, и вот идея работать с определенными
типами данных, когда мы хотим обработать некоторое их свойство, то проще было бы передать экземпляр некоторому модулю, который бы на основе экземпялры и выдал нужный интерфейс для обработки!
Звучит неслаженно, но мысли таким

==============================================================================================================================================================================================

Уже не касается материала, а просто мысли.
Я подумал, буду ли использовать данную технику, как и все изученные до этого? Скорее всего, как делаю это обычно, на параллельной ветке в гите. Дело в том, что языкам нужен такой инструмент, 
который скажет, что техника Т1 (допустим тот же ограниченный полиморфизм) и техника Т2 (мой самописный код) -- они эквивалентны. То есть, я хочу убедиться, что операции, выполняемые над типами в
случае Т1, и, выполняемые над типами в случае Т2 -- эквивалентны. Тогда, я бы мог использовать все изученные техники без проблем. А пока, отношусь ко всему со скепсисом и приходиться доверять
своему опыту и рассуждениям :)
