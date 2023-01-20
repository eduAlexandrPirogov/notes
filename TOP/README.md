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

UI дипломного проекта.
Было:

```cpp
//Панель содержала иные виджеты. При вызове методов FillViewResult и FillViewAggrResult изменялась таблица отражения результатов (виджеты, которые
//содержали таблица результатов различались. Неправильное состояние заключалось в том, что при некотором одном запросе к БД, результат можно было применить к определенному типу таблицы
//
class ReportMainPanel : public wxPanel
{
public:
	ReportMainPanel(wxWindow* parent);
	~ReportMainPanel();

	void AddSavedQuery(std::string& title);
    //Default sql result
	void FillViewResult(std::vector<Json*>& query_result);
    
    //Result with aggregated components
    void FillViewAggrResult(std::vector<Json*>& query_result);
private:

    //Класс отражающий результаты запросов
	ListView* query_result_view = nullptr;

	QueryHistoryPanel* query_history_panel = nullptr;
};

//Результаты отображал класс, который внутри себя менял тип таблиц для отражения результатов

class ListView : public wxPanel
{
   public:
      //...
      
       //Внутри куча вызово для изменения отражения таблицы результатов
	void setDefaultQueryResult(std::vector<Json*>& query_result);
    
      //Внутри куча вызово для изменения отражения таблицы результатов
    void setAggrQueryResult(std::vector<Json*>& query_result);
}
```
Тут легко привести программу к краху, если дать не тот массив результатов в неподходящий тип таблицы, или добавить в уже существующую таблицу дополнительные неподходящие данные

Стало:
Сделаем класс ListView иммутабельным и наследуем от него два нужных типа нам класса для отражения соответствующего типа таблиц:

```cpp
class ListView : public wxPanel
{
   public:
      //...
      
    virtual void fillTableByQueryResult(std::vector<Json*>& query_result) = 0;
}

class  DefaultListView : public ListView
{
   public:
      void fillTableByQueryResult(std::vector<Json*>& query_result);
}

class AggrListView : public ListView
{
   public:
      void fillTableByQueryResult(std::vector<Json*>& query_result);
}
```

Теперь, для того, чтобы отразить тот или иной тип результатов, будет создан новый соответствующий виджет.

```cpp
class ReportMainPanel : public wxPanel
{
public:
	ReportMainPanel(wxWindow* parent);
	~ReportMainPanel();

    //Default sql result
	void FillViewResult(std::vector<Json*>& query_result)
    {
       query_result_view = new DefaultListView(this);
       listView->fillTableByQueryResult(query_result);
       this->update();
    }
    
    //Result with aggregated components
    void FillViewAggrResult(std::vector<Json*>& query_result)
     {
       query_result_view = new AggrListView(this);
       listView->fillTableByQueryResult(query_result);
       this->update();
    }
private:

    //Класс отражающий результаты запросов
	ListView* query_result_view = nullptr;

	QueryHistoryPanel* query_history_panel = nullptr;
};
```

## Пример 3

Разделяем

Было:

Стало:
