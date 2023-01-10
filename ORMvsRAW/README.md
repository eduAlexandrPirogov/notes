# ORM vs SQL RAW

Дисклеймер.
У меня нашлось только два примера, хотя требовалось три, т.к. обычно не использую ORM и нашел вот два типа примеров.

Ниже приведены два примера перевода из ORM в Raw SQL.

Как работает ORM в Laravel: в построении запросов участвует класс Builder. Модели используют Builder (и многие другие traits'ы), чтобы предоставлять удобные методы:
find, select, get и т.п.
При построении запроса, в Builder'e в основном имеются проверки на:
1) что за модели участвуют
2) проверка на типы данных (постоянные if instance of...).

Сам по себе QueryBuilder -- просто абстркция для построения SQL запросов, т.е. он преобразует отраженные модели в таблицы, а методы -- в грамматику SQL и не особо тяжелый.

# Example 1

Старый:

Данный код берет коллекцию путешествий совместно с иными коллекциями для заданных путешествий и машинки.
Далее проводятся манипуляции с датой и временем и небольшие преобразования значений.
```php
public function retreiveTravels(Request $request, Vehicle $vehicle){
             WorkingList::select(['timezone', 'working_lists.id'])->with('assign_list')
            ->with('enterprise_vehicle')
            ->with('enterprises')
            ->whereIn('working_lists.id', $workingId)
            ->where('enterprise_vehicle.vehicle_id', '=', $vehicle)->get()->toArray();
            //...
}
```
Rows count: > 10,000
Time executed: 3.82 s

New:

SQL Raw Запрос.
```php
public function retreiveTravels(Request $request, Vehicle $vehicle){
        $enterprises = \DB::select("select * from working_lists wl 
        join assign_list al on al.id = wl.assign_id 
        join enterprise_vehicle ev on ev.vehicle_id = al.vehicle_id
        join enterprises e on e.id = ev.enterprise_id
        where wl.id in ('".$workingId."') and ev.vehicle_id = ".$vehicle);
        //...
}

// А еще лучше обернуть в отдельную БД функцию
```
Time executed: 1.8 s

# Example 2

Old:
Берем менеджера и соответствующие машинки. With -- замена join, синтаксический сахар.
```php
public function managerWithVehicles(){
       // $withVehicles = Manager::with('vehicle')->where('id', '=', Auth::guard('api')->user()->id)->get();      
        return new ManagerResource($withVehicles);
}
```
Rows count: > 20,000
Time executed:2.4s

New:
Аналогичный SQL Raw запрос.
```php
public function managerWithVehicles(){
         $id = Auth::guard('api')->user()->id;
        $withVehicles = \DB::select("select v.* from managers m 
        join enterprise_manager em on em.manager_id = m.id
        join enterprise_vehicle ev on ev.enterprise_id = ev.enterprise_id
        join vehicles v on v.id = ev.vehicle_id
        where m.id = $id")->toArray();   
        return new ManagerResource($withVehicles);
}
```

Rows count: > 20,000
Time executed: 1.7s

------------------------------

При работе с БД, мне доводилось в основном создавать функции в Postgres, которые возвращали результат запроса (или сам запрос). Функция принимала в качестве:
1) Проекцию на столбцы
2) Условия сортировки

ORM хорош для элементарных запросах (например with), но когда запросы становятся сложнее, то код быстро "уродуется" и, как выяснилось, падает в производительности.
Писать SQL Raw запросы в коде тоже не лучшая идея, посему их можно перенести в файл, а лучше, как я делаю и как мне кажется верным, переносить в функции Базы Данных.

Плюс данного подхода в том, что мы:
1) Явно задаем спецификации -- у нас по проекту не размазаны ORM или SQL запросы, имеется подобие интерфейса БД к которому мы можем обращаться за данными.
2) Кодовая база становится чище
3) оптимизатор запросов сделает запрос быстрее
4) Все манипуляции над данными лучше производить в БД (но не берусь утверждать), а не тратить циклы процессора на сервере :)
5) Мы даем клиенту-разработчику минимум свободы (запрещаем то, что нельзя).

Но тут имеются и минусы:
1) Пагинация. Проще воспользоваться готовым инструментом, нежели писать пагинацию с нуля. Плюс вопрос в том, как пагинацию реализовать без готового инструмента.
2) Количество функций возрастает в Postgres'e (в Oracle есть хотя бы пакеты). Можно писать метафункции, которые обращаются в БД за списком функций, но это чревато ошибками и довольно тяжело кодировать (либо писать новую функцию с дублирующим кодом, либо изменять существующую).

То есть, если брать вышеприведенные примеры, то желательно эти запросы обернуть в функции БД.

Ну и пара примеров, как я использую запросы к БД:
