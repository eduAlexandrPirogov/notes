# Когда наследование лучше композиции?

Статья: Reframing the Liskov Substitution Principle through the Lens of Testing

2. Идея того, что тесты для супертипа должны проходить также для suptyp'а очень интересна, и, изучив статью "Reframing the Liskov Substitution Principle through
the Lens of Testing" понял, почему это хорошо и просто работает. Думал об этом раньше, но, из-за того, что я терялся в понятии между "более сильным" и "более слабым" пред и пост условий, когда
речь не шла об диапазонах, как-то забил на возможность замены в тестах экземпляр родительского класса (оказывается, я нарушал LSP). Помимо тестов, можно использовать assert'ы для
предусловий suptype, взятые из supertype предусловий supertype, как было приведено в статье.

====================================================================================================================================================================================

3. Если брать в учет, что наследование стоит использовать лишь тогда, когда полезно использовать полиморфизм подтипов, то ситуация выглядит следующей: 

====================================================================================================================================================================================
Пример 1.
На одной из работ частенько присылали файлы, которые парсились и возникла идея написать библиотеку "базовый парсер" от которого наследовались дочерние парсеры для каждой из ситуаций.
Основной довод был в том, что бОльшие части кода могли повторяться (как, например, конфигурации парсера, способ итерации, сепараторы). В итоге родилась такая иерархия:

```php
//Отдельный внутрикорпоративный репозиторий с либой
class CompanyNameBaseParser{
    protected $config;
    protected $iterator;
    
    public function Iterate() {
         //...
    }
}
``` 

Остальные парсеры для каждого из проекты выглядели так:
```php
class ServicesReportFromT1 extends CompanyNameBaseParser {

}

class QuartalProfits extends CompaneNameBaseParser {

}

```

Наследование помогало здесь в определенной ситуации, мол для каждого случая писать конфиги, итераторы, из тысячи строк которых 80% буду похожими не хотелось бы, а 
если нужно иное поведение, то достаточно переопределить метод. Но тут есть проблемы:
1) Сам  CompanyNameBaseParser в своем экземпляры нигде не использовался, он просто "висит" в кодовой базе.
2) От конфигурации CompanyNameBaseParser зависят дочерние  парсеры, и, в случае изменения конфига, придется "перелопатить" дочерние парсеры. Это, кстати и произошло, из-за чего запрещалось
изменять базовый парсер, т.к. другие могли сломаться и из-за этого приходилось писать свои с нуля.
К тому же, если проверять LSP через тесты родительского классА, то дочерние классы их не пройдут "поскольку для разных парсеров нужны разные массивы строк, и от одних данных к другим тесты
будут валиться".

Обсудив минусы, что тут можно сделать. В целом, я бы разделил для каждого парсера отдельную репу, потому что"несмотря на то, что логика парсеров во многом схожа, реализация у них должна быть у каждого своя. Я попытался несколько раз придумать, как лучше организовать иерархию между парсерами, чтобы использовать преимущества повторного кода, но вот как-то выходили все время криво.
Об использовании наследования написал свои мысли в самом "Итого". 

====================================================================================================================================================================================
====================================================================================================================================================================================

Пример 2.
На текущей работе после изучение материала я сплошь и рядом вижу, когда используют наследование вместо того, что использовать композицию или более простой вариант. 
Ниже приведу несколько примеров и комменарии:

а) Payload отображает сообщения, которые передаются по протоколу. Через поиск обнаружил, что сам Payload нигде не используется. Зато напиханы if instance of Payload ... else if instaince of ...
```java
public class Payload {
     protected String payload;

     public String getPayload() {
           return this.payload;
     }
}
 
public class StatusPayload extends Payload {
    protected Integer status;
    protected LocalDateTime date;
    protected int parts;

    public void setStatus(Integer status) {
        this.status = status;
    }

    public void setDate(LocalDateTime date) {
        this.date = date;
    }

    public void setParts(int parts) {
        this.parts = parts;
    }  
}

public class RespPayload extends Payload {
      //дополнительные поля 
}
```
Как бы я исправил: 
1) Создал бы интерфейс Paylodable, с единственным методом String ToRaw() -- чтобы получать raw-payload и импементировал каждый вид Paylod'a. В данном случае мы избавляемся от instance of
и заранее получаем raw-payload, поскольку формируем его внутри каждого экземпляра класса.
2) Создал бы для каждой возможной части мапу, где ключ -- перечисление из частей сообщений (date, status и т.д.), а в качестве значения -- функция, возвращащая строковае значение 
(например, для date -> "Date: <Value>"). При получении Payload'a от определенного канала, мы заполняем мапу посредством команд, а затем посредством запроса проходил по мапе и формируем Payload.
В идеале создать АТД на мапу и делегировать ей добавление и считывание значений.

В любом из вариантов мы избавимся от нескольких классов, которые нигде в коде полиморфно не используются.

б)  Опять же, нигдк полиморфно не используются данные классы, их конкретные инстансы создаются через new(). И, как мне кажется, везде, где используются нечистые методы, то лучше не использовать
наследование, а максимально изолировать их (если же мы наследуем нечистые функции, то не соблюдаем LSP, и в большинстве своем переопределяем везде методы).
```java
public class DefaultBillRenderer {
    // Рендер контента на оснвое файла
    public void render(File file) throws IOException {
        //...
    }
}

public class HtmlBillRender extends DefaultBillRendere {
    @Override
    public void render(File file) throws IOException {
        // Своя реализация
    }
}

public class CsvBillRenderer extends DefaultBillRendere {
    @Override
    public void render(File file) throws IOException {
        // Своя реализация
    }
}
```

Тут вообще лучше обойтись без композиции и наследования и создать для каждого случая свой класс. 
Хоть и логика у них, казалось бы одинаковая (рендер), но реализация должна быть качественно разной.

====================================================================================================================================================================================
====================================================================================================================================================================================

Пример 3.

Удачное применение наследования. Это был первый раз, когда я писал внутрикорпоративный проект сам, и тогда же прошел курс по АТД, часть 2, где воодушевился использованием
наследования. Этот проект для экстраполяции финансовых показателей, который уже не раз приводил в отчетах, но тут будет его самая первая версия.
Первоначально спецификация звучала так: нужно для сервисов рассчитать с помощью данных формул/алгоритмов экстраполяцию данных. 
На основе ТЗ, мой проект содержал классы, отражающие способы вычисления:

```php
class ForecastDefault {

    public function Turnover() {

    }

    public function CostPrice() {

    }

    public function Profit() {

    }
}

class ForecastByDate extends ForecastDefault {
    public function CorrelateByDate(Interval $interval) {

    }
}

class ForecastWithCorrelation extends ForecastDefault {
     public function CorrelateWith(Correlation $corr) {

     }
}
```

Свой подход я объяснял так, мол, наследование тут хорошо, поскольку алгоритм экстраполяции имеется общий у них (используя методы Turnover(), CostPrice(), Profit()), 
но для некоторых сервисов нужны дополнительные действия, и повторное использование кода тут подходит в самый раз. В случае чего, я всегда мог переопределить наследуемый метод, 
если требовались изменения.

Рассмотрим данный случай с точки зрения полиморфизм подтипов. Хоть в том проекте мне это и не понадобилось, но порассуждать на эту тему стоит. Допустим, мне нужно было бы для определенного
сервиса все способы экстраполяции, чтобы, например, узнать, какой из способ дает наименьшее расхождение с согласованными данными. Функция может иметь вид:

```php
class Comparator {
    //Return 1 if $first is better than the $second one, otherwise -1
    public function Compare(Service $service, ForecastDefault $first, ForecastDefault $second){
    }
}
```

Рассмотрим случай с точки зрения полиморфизма подтипов. Для сервиса S я мог применить любой подход из переданных наследуемых методов, будь то родительский ForecastDefault, или же
любой дочерний. Важно отметить, что я "угадал" на тот момент с полиморфизмом подтипов, поскольку не знал про тестирование для дочерних классов через родительские тесты. 
Смотря сейчас на этот проект, учитывая данные на тот момент пред и постусловия (их либо не было вовсе, либо они были элементарные), то я никак не нарушил правило сужения/расширения 
пост и предусловий соответственно. 
1) Получается такая картина, что мог передать любой класс из иерархии в метод Compare(...), поскольку сохраняю равноправность (о равноправности я пишу ниже).
2) Я использовал лучшие стороны наследования: повторное использование кода, и полиморфизм подтипов.

====================================================================================================================================================================================
====================================================================================================================================================================================

Итого: наследование действительно мощный инструмент, но использовать его ради того, чтобы просто "не дублировать" код -- не лучшая идея.
Я воспринимаю наследование следующим образом -- это инструмент "оптимизации" (не только кодовой базы, но и структуры, за счет ковариантности и полиморфизма) проекта. Подобно тому, как мы 
используем контейнеры, чтобы "сложить"  взаимосвязанные сущности, наследование также помогает связть сущности. Но так называемая "преждевременная оптимизация наследованием" чревата проблемами, 
т.к. наследование подразумевает сильную связность между сущностями -- изменение родительского класса влияет на дочерние. Лишь в тот момент, когда кодовая база сформировалась и видны общее 
поведение сущностей, их кодовая база и уверенность в том, что наследование действительно поможет, то тогда стоит задуматься на "оптимизацией".
Я несколько раз вступал в дебаты, когда утверждал, что "лучше небольшое дублирование кода", нежели "хрупкая иерархия". Объясняю это следующим образом: допустим у нас имеется объект О1, который
относится к иерархии И1. Мы вправе утверждать, что О1 может принадлежать и к другой иерархии И2, и вообще к множеству других иерархий (например, утка может принадлежать к иерархии уток, но также
может принадлежать к иерархии игрушек). Если мы присваем например О1 к И1, а потом приходит требование, которое меняет О1 таким образом, что оно не может принадлежать к И1, нам приходится 
"вынимать" О1 из И1 и менять везде код, где используется данная иерархия. 
И наибольшая проблема заключается в том, что подобных иерархий -- множество, а реализуем мы, по крайней мере, одну.

Если же мы используем наследование, то LSP -- есть некое доказательство того, что О1 действительно принадлажит некоторой иерархии. Этому способствуют предикаты пред и пост-условий.
Таким образом мы ссужаем универсум до некоторого конкретного множества иерархий, к которым может принадлежать О1. Проблема заключается в том, что имеются "подводные камни", связанные с 
сужением и расширением условий, идя вниз по иерархии (с диапазонами все просто, но как быть с множеством типов?), 
но вот благодаря статье, можно это проверять через тесты, что является очень даже хорошой идеей.
