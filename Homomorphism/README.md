Обобщение проектных абстракций

1.

Пример 1.

```cpp
class Command
{
public:

    /**
     * Pre-conditions: given an RequestHandler pointer
     * Post-conditions: Command will be executed
    */
    virtual void execute(RequestHandler* handler) = 0;

    /**
     * Pre-conditions:
     * Post-conditions: validates inputed arguments of commands
    */
    virtual void validateArgs() = 0;

    /**
     * Pre-conditions:
     * Post-conditions: return string representation of command
    */
    virtual string strForPrint() = 0;
};

class InfoCommand : public Command 
{
public:
   // Prints info to stdout
   //
   //
   virtual void stdout() = 0;
}

class HelpCommand : public InfoCommand
{
    COMMANDS commandHelp;
    std::unique_ptr<Arguments> args;
public:
    HelpCommand();
    HelpCommand(std::unique_ptr<Arguments>& givenArgs);
    HelpCommand(COMMANDS command);
    HelpCommand(COMMANDS command, const std::string& srcPath, const std::string& dstPath);
    HelpCommand(COMMANDS command, Arguments& args);

    void execute(RequestHandler* handler);
    string strForPrint();
    void validateArgs();
    void stdout() = 0;
};

class InvalidCommand : public Command
{
    std::unique_ptr<Arguments> args;
public:
    InvalidCommand() = default;
    InvalidCommand(std::unique_ptr<Arguments>& givenArgs);
    virtual void execute(RequestHandler* handler);
    virtual string strForPrint();
    virtual void validateArgs();
   
};

class GetCommand : public Command
{
    COMMANDS commandGet;
    std::unique_ptr<Arguments> args;
public:
    GetCommand() = delete;
    GetCommand(const GetCommand&) = delete;

    GetCommand(std::unique_ptr<Arguments>& givenArgs);
    GetCommand(COMMANDS command, std::unique_ptr<Arguments>& givenArgs);
    virtual void execute(RequestHandler* handler);
    virtual string strForPrint();
    virtual void validateArgs();
};

class PutCommand : public Command
{
    COMMANDS commandPut;
    std::unique_ptr<Arguments> args;
public:
    PutCommand() = delete;
    PutCommand(std::unique_ptr<Arguments>& givenArgs);
    virtual void execute(RequestHandler* handler);
    virtual string strForPrint();
    virtual void validateArgs();
};
```
Получается иерархия:
           Command
         /        \
        /          \
       /            \
commnad to execute   InfoCommand
(invalid, put,get)       \
                          \
                           \
                           HelpCommand
                           
Тут скорее можно избавиться от InfoCOmmand. Этот "слой" был добавлен, чтобы отделить инфнормативные команды, которые просто выводят информацию, от исполняющих, 
которые влияют на состояние системы. Можно на самом деле HelpCommand наследовать не от Command, а от какого-нибудь Logger/Printer, который имеет один 
метод virtual void stdout() = 0. 
Плюс, то, что семантика у выполняющихся и информирующихся команд одинаковая, их спецификации различны и лучше разделить их.

Пример 2.

```cpp

/**
 * Class to Validate types of Arguments
*/
class Rule
{
    public:
       /** 
         * Validates form of arguments
         * Pre-conditions: 
         * Post-conditions: validates given arguments
        */
        virtual const RULESTATUS validate() = 0;
};

class ArgumentsRule : public Rule
{
public:
    virtual const ARGRULESTATUS verifyFlags() = 0;
}

class PathRule : public ArgumentsRuleRule
{
    std::string localPath;
    PATHRULESTATUSES currentParseStatus;
    std::unordered_map<PATHRULESTATUSES, const char*> argumentsMessage;
    public:
        PathRule() = delete;
        /**
         * Pre-conditions: given name of existing file
         * Post-conditions: instance of PathRule created
        */
        PathRule(const std::string& localPath);

        /** 
         * Validates form of arguments
         * Pre-conditions: 
         * Post-conditions: validates given arguments
        */
        const RULESTATUS validate();


        /** 
         * Validates form of flags
         * Pre-conditions: 
         * Post-conditions: validates given arguments
        */
        const ARGRULESTATUS verifyFlags() = 0;
};

class RequestRule : public Rule
{
public:
        /** 
         * Validates form of request
         * Pre-conditions: 
         * Post-conditions: validates given arguments
        */
        const RULESTATUS validate();
}
```
Иерархия получается таковой:
                      Rule -- validate()
                     /    \
                    /      \
                   /        \
          ArgumentsRule     RequestRule
            /
           /
          /
      PathRule

Иерархия правил для валидаций соответствующих сущностей. ArgumentsRule можно в целом убрать и наследовать PathRule от Rule и добавить соответствующие доп методы
в PathRule. На момент написания кода казалось, что это сделает иерархию проще для восприятия, а также для будущего дополнения валидации аргументов (командной строки).

Вообще, если бы нужно было добавить такие изменения, то валидация, например флагов аргументов выглядела бы одинаковой для всех аргументов строки. Это можно вывести
как один отдельный тип:

FlagValidator:
   ValidateFlags(flags []flag) = 0;


2. Вышеприведенные примеры можно переписать с использованием interface dispatch на Go:

Первый пример:
```go
// Executer makes object execute task
//
// Pre-cond:
//
// Post-cond: if executes correct then returns error = 0
// Otherwise returns error 
type Executer interface {
    func Execute() error
}

// InfoReporter buuild string message for events
//
// Pre-cond:
//
// Post-cond: returns built message.
type InfoReporter {
    func Report() string
}
``` 

Проблема первого примера с командами могла быть таковой, что одна команда могла быть как исполняющей, так и информативной (например исполнить запрос и сразу доложить результат).
Хотя лучше это сделать через два отдельный метода (точнее, команду и запроса).
В любом случае, тут у нас получается, что любые объекты, будь то команды различных видов, или например, обработчик запросов, которые никак не связаны между собой,
образуют так называемую "область", "гомоморфические типы" по операциям.
Они хоть и влияют по разному на свое внутреннее состояние, но идея одна: они вернут либо nil, в случае успеха, либо error в ином.

======
Перепишем пример 2.
При первом приближении
```go
type Validater interface {
    // Validate validates params and they are correct, return nil otherwise error
    func Validate() error
}
```

Если мы все же хотим разделить тип по операциями более явно/конкретно, то можно сделать так:

```go
type CmdArgValidator interface {
    Validater
    func ValidateFlags() error
}

type RequestValidator interface {
    Validater
    func ValidateRequest() error
}
```

Go за счет interface embedding позволяет легко создавать подобные гомоморфизмы не только операциям, но и семантически. Вроде и Cmd аргументы и RequestValidator
валидирует значение, но, это, хоть и одна логика, но спецификации лучше разделить :)

3. Сразу на ум приходит Dependency Injection. Причем в Go они ком
