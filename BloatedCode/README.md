# Раздутый код

# Пример 1

Было:

```php
class FinanceRefinder
{
  public function refindFins(Finance $finances, $method, $coeff)
  {
      //В зависимости от параметров происходили различные методы
      if($coeff > 1.11 && $method instanceof StdMethod)
          ...
      elseif (..)
  }
```

Стало:

```php
class FinanceRefinder
{
  private Finance $finances; // Внутренняя структура данных
  private array $coeffMap; //Коэффициенты и методы
  
  public function refindFincStdMethod($coeff)
  {
  
     $this->finances = $this->coeffMap[$coeff]()
  }
}
```

В первом варианте, приходилось помнить особенности, при каких коэффициентах нужно передавать какие методы (методы были представлены в виде классов с единственным методом)
Хоть и была документация, но это все равно приходилось держать в голове.
Исправил это тем, что сделал отдельный АТД FinanceRefinder, сделал его полями Finance и словарь, которая для диапазона коэффициентов хранит соответствующий метод. Теперь,
если явно не придет тикет на изменение методов расчета и коэффициентов, то нам не стоит думать об этих нюансах.

# Пример 2

Было:

```php
//Считывает пропаршенные данные и группирует по указанному клиенту
class DataProcessing {

  const CLIENTS_TO_GROUP = [0,1...];

  public function groupClientsServices($clientId)
  {
    
     if($clientId == 0)
        $this->groupUp($clientId); //возвращает группированный массив 
     elseif($clientId == 1)
     ....
     else 
      throw new Exception('Incorrect client to group!\n');
  };
}
```
Казалось бы простой пример, но из-за этого часто возникали ошибки, когда хозяин кода уходил в отпуск, поскольку неизвестно было каких можно передавать клиентов, поскольку
они зависят от массива CLIENT_TO_GROUP, а это могло меняться несколько раз за неделю, из-за чего во многих местах нужно было это учитывать.

Стало (точнее, как бы изменил):

```php
class AvaibleToGroupClients
{
   private array $clientsId;
   
   //add method and lookup method
   public function add($client)
   {
   
   }
   
   //Try find. If exists -- returns true otherwise false
   public function lookup($client) : bool
   {
   
   }
   
   public function retrieve()
   {
   
   }
}

class DataProcessing {

  private AvaibleToGroupClients $avaibleClientToGroup;
  
  public function __construct__($avaibleClientToGroup)
  {
    $this->avaibleClientToGroup = avaibleClientToGroup;
  }

  public function groupClientsServices()
  {
    foreach($avaibleClientToGroup as ...)
    ...
  };
}
```

Суть в том, чтобы заполнять за раз нужных нам к группировке клиентов, т.е. задавать это не статически, а динамически, и после передавать АТД для обработки.
Тем самым, нам в разных участках кода нужно лишь создать АТД AvaibleToGroupClients, который заполнит себя сам (или с помощью .env) и его обработывает соответсвующий класс DataProcessing.
Почему метод обработки не включать в сам AvaibleToGroupClients? В теории у нас могут быть различные обработки и плодить их в этой АТД -- нарушение SRP, к тому же
можно использовать трейты или паттерн посетитель.

# Пример 3

Было:

```php
//$output file может быть null, $orderFile зависит от $order (могут быть null также). из-за тогО, что $cleintsToCut -- просто массив, можно указать не тех клиентов и итерации будут лишними
public function cutByOptions($inputFile, array $datesToCut, array $clientToCut, $order, $orderField,  $outputfile)
{
    
}
```

Получается, что мы, если не будем полмнить о нюансах бизнес логики и данного метода, то можем замедлить работу методы в разы!

Стало:


```php
// Определяем АТД для файла, который нужно будет обрезать
class CutOperand
{
   private array $givenFile;
   
   
}

// Определяем АТД для дат, которые нужно будет обрезать
class DatesToCut
{
   private array $dates;
} 

// Определяем АТД для клиентов, которые нужно будет обрезать
class ClientsToCut
{
  private array $cleints;
}

class CutOperator
{
   public function datesToCut(DatesToCut $dates) : CutOperand
   {
    
   }
   
   public function clientsToCut(ClientsToCut $clients) : CutOperand
   {
   
   }
   
   public function orderAsc(OrderField $field) : CutOperand
   {
   
   }
   
   
   public function orderDesc(OrderField $field) : CutOperand
   {
   
   }
}
```

Идея в том, чтобы динамически заполнять нужно АТД DatesToCut, ClientsToCut, из них делать выборку существующих элементов (чтобы лишних раз не итерировть файл для несуществующих данных).
Затем эти данные подаются CutOperand'у. Тут минимум ошибок, поскольку у нас CutOperand обладает свойством замкнутости.
