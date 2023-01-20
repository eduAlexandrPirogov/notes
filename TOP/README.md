# Пишем безошибочный код

## Пример 1

Было:
```php
//Суть в том, что поочередно высчитывается turnover, proceeds, costPrice, profit на основе предыдущий вычислений (например, proceeds может быть высчитан только на основе turnover)
class ForecastManager
{
    private $finances;

    public function turnover()
    {
        ...
    }
    
    public function proceeds()
    {
        ...
    }
    
    public function costPrice()
    {
        ...
    }
    
    public function profit()
    {
        ... 
    };
}

public function forecast($forecastManager)
{
  //Потенциальная ошибка за счет того, что программист может вызвать функции не в верном порядке
   $forecastManager->turnover();
   $forecastManager->proceeds();
   $forecastManager->costPrice();
   $forecastManager->profit();
   ...
}
```
Стало:

```php
class Forecast 
{
    protected bool isCalculated;
    protected $finances;
    
    //Не лучшая  идея но все же
    public function retrieveFinances()
    {
        return $this->finances;
    }
};

class TurnoverForecast extends Forecast
{
    //Предоставляем некоторые стартовые значения
    public function __construct__($startFinances)
    {
    
    }
    
    public function calculate()
    {
      ...
    }
    
    public function isCalculated()
    {
       ...
    }
}

class ProceedsForecast extends Forecast
{
   public function __construct__(TurnoverForecast $turnover)
   {
      ...
   }
   
     public function calculate()
    {
      if($this->turnover->isCalculated())
      {
         ...
      }
    }
}

class CostPriceForecast extends Forecast
{
   public function __construct__(ProceedsForecast $proceeds)
   {
    ...
   }
   
   //Идентичная схема...
}

//Вызов расчета


public function calculate($finances)
{
   $turnover = new TurnoverForecast($finances);
   $turnover->calculate();
   
   $proceeds = new ProceedsForecast($turnover);
   $proceeds->calcualte();
   
   $costPrice = new CostPrice($proceeds);
   $costPrice->calculate();
   ...
}
//Ну или можно применить монады :)
```

Конечно, можно тут добавить иерархию классов, когда результат посчитался, например:

```php
class TurnoverForecast extends Forecast
{
    public function calculate()
    {
      ...
      return new CalculatedTurnoverForecast($this->finances);
    }
}

class CalculatedTurnoverForecast extends TurnoverForecast
{
   public function __construct__($finances)
   {
      ...
   };
}

class ProceedsForecast extends TurnoverForecast
{
    public function __construct__(CalculatedTurnoverForecast $turnover)
    {
        ...
    }
};
```

## Пример 2

Было:

Стало:

## Пример 3

Было:

Стало:
