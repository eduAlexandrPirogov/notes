# Интерфейс компактнее реализации

## Призрачные состояния

### Пример 1.

```php
public function balanceFactRaw($factRaw, $restOfFacts)
{
  //Ghost state
   $coeff = coeffOfResfOfFacts($restOfFacts);
   //Ghost state
   $factCorrelCoef = //формула для расчета другого типа коэффициента c переменной $coeff
   //...
   $factRaw *= $coeff;
   return $factRaw;
};
```

$coeff и $factCorrelCoef в каком-то смысле недетерминированы, т.к. они зависят от множества данных, которые могут приходят из БД.
Для такой функции невозможно написать тесты. Лучше вынести коээфициенты за пределы функции.

### Пример 2.

Проблема может заключаться в том, что другой разработчик изменит значени $sum на иное, что сломает рассчеты для других сервисов
```php
public function calculateFactTypeForecast(array $rows)
{
   //ghost state
   $sum = 0;
   //ghost state
   $countedDays = 0;
   foreach($rows as $row)
   {
      $row = (array)$row;
      if(!is_null($row["..."])
      {
         $sum +=...
         $counted += ...
      }
   }
}
```

Тут лучше сделать версию accumulate/fold, где будет явно указываться и $sum и $countedDays в параметрах.

## Погрешности/неточности

### Пример 1.

```php
class Correlater
{

  //Неточность, так как 1)  какой диапазон допустимых значений имеется для $coeff; 2) при указании значения по-дефолту, мы двигаемся от конкретной
  //"узкой" реализации (в проекте именно так и происходит)
  public function correlateForecastType($coeff = 34.55)
  {
     //...
     //Неточность. Несмотря на то
     $precisionForecast = //formula .. * $coeff;
  }

}
```

### Пример 2.

Случаи, когда имеем класс-"контейнер", который хранит определенный объект, хотя потенциально он может хранить любой тип объекта.

class Container 
{
public:

  void insert(Object A);
  void erase(Object A);
}

В данном случае лучше использовать template. Если же, мы хотим хранить только определенный типы, то тут можно прибегнуть к:
1) Специализации шаблона
2) Явного указания хранимого типа данных в шаблоне

```cpp
template<typename T = SpecificObjects>
class Container 
{
public:
  //Pre-cond: given an element of SpecificObject type
  //Post-cond: element instance was added to Container
  void insert(T t);
  
  //Pre-cond: given an element of SpecificObject type and elem exists in Container
  //Post-cond: searcing element returned
  T erase(T t);
}
```

### Пример 3.

Использование конкретного типf данных, вместо иерархии типа данных.

```cpp
class Writer
{
public:
  //Неточность
  //Pre-cond: Given a Loger
  //Post-cond: Action was loged
  void LogAs(ConcreteLoger* loger);
}
```

Сказано, что мы можем использовать Логгер, но при этом добавить только конкретный объект (хотя в теории, это может быть верхушка иерархии).
Из множества логгеров мы используем конкретный. Тут лучше добавить абстрактный класс, от которого наследуется ConcreteLoger.

```cpp
class Writer
{
public:
  //Неточность
  //Pre-cond: Given a pointer of object of type AbstractLogger
  //Post-cond: Action was loged
  void LogAs(AbstractLoger* loger);
}
```


## Интерфейс явно не должен быть проще реализации

### Пример 1.

Представим, что мы имеем множество/иерархию элементов t клааса T , часть которых требуется обработать одним способом,а остальную часть -- другим.
Т.е. имеется следующий метод:

```php
public function handleSpecificObjects(array $objects)
{
   //Если объект типа А, то делаем так, иначе...
}
```

В таком случае можно:
1. Добавить метод с подобным именем, но другими параметрами (ad-hoc polymorhism)
2. Добавить параметр лямбды, которая нужны образом будет обрабатывать объекты

```php
public function handleSpecificObjectsA(array $objectsA)
{
   //Обрабатываем объекты A
}


public function handleSpecificObjectsB(array $objectsB)
{
   //Обрабатываем объекты B
} 
```

На плюсах это выглядело было следующим образом:

```cpp
void handleSpecificObjects(std::list<A*>& objects)
{
}

void handleSpecificObjects(std::list<B*>& objects)
{
}
```

### Пример 2.

Имеется вариант, когда нам нужно обработать некоторый диапазон элементов контейнера:

```cpp
void handelSpecificObjects(Container* container, int min, int max)
{
    if (min < container->min() || max > container->max())return;
    //...
};
```
В данном случае, хоть и можно указать комментариями, что min > 0 а max < container->max(), но тут есть решение следующее:

```cpp
void handelSpecificObjects(ContainerRange* range)
{
    //...
};
```
То есть мы принимает на вход оболочку для работы с диапазоном в Container'e. При этом, ContainerRange принимает на вход не конкретные числа, а итератор
от Container. Хоть интерфейс и стал сложнее, но мы устранили возможное ошибочное поведение.

### Пример 3.

Бывают случаи, когда интерфейс, который нельзя сделать компактнее/проще через его сигнатуру. Например, если мы вспомним структуру данных "динамических массив",
то, при добавлении элемента, если динамический массив заполнен более, чем на k%, объем для хранения данных увеличивается.

Проблема в том, что мы никак не можем указать это через интерфейс, вне зависимости от данных Типов данных. 
Можно конечно добавить в метод push_back, параметр-флаг, или отдельный метод, который расшираяет объем массива, но лучше сделать это через комментарии в виде пред/пост-условий:

```cpp
//Pre-cond: Given an element to add
//Post-cond: Element was added to array. If array has k% of elements then array increase own capacity.
void vector::push_back(T elem)
{
   //....
};
```

То есть, мы "ужесточяем" границы за счет:
1. Комментариев с пред и пост условиями
2. Явного указания типа данных (это могут быть классы-wrapper'ы или ad-hoc polymorphism)
