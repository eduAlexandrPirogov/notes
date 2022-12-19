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
//Но не всегда известно, что за сервис потребуется удалить. Из-за того, что данные приходят по ссылке, то измененные уже никуда не годятся и 
//приходится добавлять костыли
public function relationWithoutService(Relation& $relation, Servce $service, $targetFin) : Relation
{
   $relation->erase($servce);
   //Высчитываем финансы
   return $finances == $targetFin;
}
```

Стало
```php
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
```

## Пример 3

Было
//Класс возвращал экземпляры комманд по ключу-строке. То есть у нас были единственные экземпляры комманд, которые расширивались
//Присутствовали костыли, когда словарь заполнялся -- некоторым командам нужны были аргументы в конструктор, некоторым нет.
//К тому же, некоторые команды, могли выполнять похожие команды далее, из-за чего, приходилось добавлять в существующую команду дополнительные методы
//и модифицировать текущий инстанс.
```cpp
typedef std::unordered_map<std::string, std::shared_ptr<Command>> CommandsMap;

/**
 * Class to Build Command Instances
*/
class CommandsBuilder
{
    std::unique_ptr<CommandsMap> commandsTable;
public:

    /**
     * Pre-condition:
     * Post-condition: created instance of CommandsBuilder
    */
    CommandsBuilder();

    // Pre-cond: name of command exists in conatiner
    // Post-cond: returns Command if exists; otherwise returns InvalidCommand
    std::shared_ptr<Command> build(char* commandLine[], int argc);
        /...

```

Стало
//Теперь мы работаем с иммутабельными значениями -- храним только функции в словаре, а функции возвращают новые экземпляры.
//В данном случае, я упростил заполнение словаря, а также моменты с созданием комманд с аргументами перенесены в соответствующие функции.
//Если нам понадобится новаая команда -- нам проще создать новую, нежели модифицировать оригинал
```cpp
typedef std::unordered_map<std::string, std::function<std::unique_ptr<Command>(std::unique_ptr<Arguments>&)>> CommandsMap;

/**
 * Class to Build Command Instances
*/
class CommandsBuilder
{
    std::unique_ptr<CommandsMap> commandsTable;
public:

    /**
     * Pre-condition:
     * Post-condition: created instance of CommandsBuilder
    */
    CommandsBuilder();

    // Pre-cond: name of command exists in conatiner
    // Post-cond: returns Command if exists; otherwise returns InvalidCommand
    std::unique_ptr<Command> build(char* commandLine[], int argc);
private:

    /**
     * Pre-condition: Given existing type of Command
     * Post-condition: If given command exists return new instance of Command; otherwise returns Invalid Command
    */

    //GetCommand to download files from dropbox
    std::function<std::unique_ptr<Command>(std::unique_ptr<Arguments>&)> createGetCommandFunction = [](std::unique_ptr<Arguments>& arguments)
    {
        return std::make_unique<GetCommand>(arguments);
    };

    //PutCommand to upload files from dropbox
    std::function<std::unique_ptr<Command>(std::unique_ptr<Arguments>&)> createPutCommandFunction = [](std::unique_ptr<Arguments>& arguments)
    {
        return std::make_unique<PutCommand>(arguments);
    };

    //InvalidCommand to give a user message if invalid command or arguments were inputed
    std::function<std::unique_ptr<Command>(std::unique_ptr<Arguments>&)> createInvalidCommandFunction = [](std::unique_ptr<Arguments>& arguments)
    {
        return std::make_unique<InvalidCommand>(arguments);
    };

    //HelpCommand to display avaible commands
    std::function<std::unique_ptr<Command>(std::unique_ptr<Arguments>&)> createHelpCommandFunction = [](std::unique_ptr<Arguments>& arguments)
    {
        return std::make_unique<HelpCommand>(arguments);
    };
    
    //...
};
```

----------------------------

Вывод:
В процессе перевода данных из мутабельных в иммутабельные, я выявил для себя следующие случаи, когда стоит использовать иммутабельные состояния:
1) если над объектом производятся манипуляции, но может понадобится сохранить исходное состояние объекта
2) в случае многопоточного программирования, если проще будет избавиться от мьютекстов и просто воспользоваться "копией" объекта
3) если над объектом происходят модификации, но было бы гораздо проще создать новый объект с заданными свойствами.
