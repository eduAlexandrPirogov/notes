# Иммутабельные состояния вместо передачи по ссылке

## Пример 1

Было
```cpp
//Базовый алгоритм обхода графа (улиц) и выяснения, большой ли трафик на дорогах
//Перед проходом каждый раз помечаем каждую улицу как "не посещенную".
//Проблема в том, что проект использует потоки для работы над графом. Используются мьютексы и атомики.
void isTrafficToTough(Traffic& traffic, const string& fromStreet)
{
    //Dfs alg with modifications
};
```

Стало
```cpp
//Передаем граф как иммутабельную структуру (иммутабельную, в плане, что мы не меняем оригинал, а работаем с новым графом, как с отдельным объектом)
//Не беспокоимся о race condition, мы более не испольузем мьютексы. 
void isTrafficToTough(Traffic traffic, const string& fromStreet)
{
    //Dfs alg with modifications
};
```

## Пример 2

Было
```php
//Парсится файл с данными сервисов. Иногда нужно удалить тот или иной сервис, чтобы сошлись финансы.
//Но не всегда известно, что за сервис потребуется удалить. и приходится делать некий rollback
public function relationWithoutService(Relation& $relation, Servce $service, $targetFin) : Relation
{
   $relation->erase($servce);
   //Высчитываем финансы
   if($finances == $targetFin)
      return;
   else
   //Большой и сложный метод
    $relation->restore($service);
}
```

Стало
//Тут мы передаем отношение, как иммутабельное значение, чтобы избавиться от сложного метода, который "восстаналивает" отношение
//Мы теряем относительно скорости и памяти, но поскольку подобные задачи не частые, то этим можно пожертвовать, заметно упрощая жизнь
public function relationWithoutService(Relation $relation, Servce $service, $targetFin) : Relation
{
   $relation->erase($servce);
   //Высчитываем финансы
   if($finances == $targetFin)
      $this->eraseServiceStatus = self::CORRECT_SERVICE_WAS_ERASED;
   else
      $this->eraseServiceStatus = self::INCORRECT_SERVICE_WAS_ERASED
}


## Пример 3

Было
```cpp
```

Стало
```cpp
```
