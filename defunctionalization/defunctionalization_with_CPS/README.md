# Дефункционализация с CPS

Предыдущая часть: https://raw.githubusercontent.com/eduAlexandrPirogov/notes/main/defunctionalization/CPS/README.md

Вопрос о применении дефункционализации совместно с CPS. 

Прежде, чем понять, в каких проектах данная комбинация может быть использовано, я определил, для чего нужно два подхода.
     Дефункционализация позволяет создать внутри программы свой язык программирования, путем определения функций первого порядка, которые
можно использовать в качестве аргумента к функциям высшего порядка. Это очень похоже на подход, когда мы имеем мапу, ключом которой является
строка, но вместо строки, мы передаем определенный тип данных enum -- мы строго определяем диапазон, в рамках которого разработчик
может действовать (не допускаем недопустимые состояния программы). Про сериализацию с RPC и распределенщину не могу сказать на данный момент
ничего.
     CPS -- стиль программирование [8], который позволяет абстрагировать поток управления. Для себя отмечу, что это хороший способ 
поддержки хорошего дизайна кодовой базы -- мы кобминириуем между собой чистые функции, но, когда результат нужно из или записать во
внешний мир, то мы неизбежно сталкиваемся с побочными эффектами [6]. Подобные цепочки вычислений можно записать в CPS, тем самым в одну 
строчку мы записываем поток управления. В таком виде очень напмоинает монады, когда мы можем передать в качестве продолжений значения-функций
в случае ошибки/удачного исполнения операции [5]. 

     Как я вижу использование совместное использование данных двух подходов? На самом деле выглядит все "на словах" элегантно -- мы определяем
продолжения в виде функций первого порядка конкретного модуля, и, на основе сформированного API, строим цепочки вычислений, причем, я так 
понимаю, что "финальные" продолжения (они же функции первого порядка) связанные с взаимодействием с внешним миром [2]. Конечно, подобные 
цепочки можно комбинировать с функциями высшего порядка.
     Например, в рабочем Erlang-проекте имеются модули, состоящие из чистых функций и модули, которые взаимодействуют с 
внешним миром, как например логгирование или работы с БД (не Мнезия). Модули, взаимодействующие с внешним миром устроены либо как 
gen_server'ы, либо как FSM-ки. Дефункционализация с Erlang легко исполняется с помощью встроенной apply, а CPS подразумевает в конечном 
итоге передача вычисленного значения продолжению, которое либо будет залоггировано, либо будет записано в БД. Попробую на отдельном форке
исполнить такое.
     То есть, на данный момент, я могу рассматривать данных подход как альтернативный вариант проектирования
(будет ли дефункционализация преобладать над рефункционализацией) и подход (будет ли использовано CPS или нет). И если с дефункци-цией 
для меня все просто, то с CPS вознкает ряд тем, в которых нужно разбиратьмя:
1. CPS очень часто упоминают совместно с асинхронными операциями, но везде ли это будет нужно? Например, в JavaScript используется
кооперативный шедулер, когда как в Go -- вытесняющий. Если я применю CPS с помощью этих языков, будет ли сохранена эффективность?[1], [4]
2. Насколько переход от рекурсивного к CPS будет целосообразен? [3] Например, мне мне нужно чистая фибоначи без продолжения. Стоит ли 
делать обертку над функцией фибоначи, которая будет принимать фибоначи-подобную функцию и продолжение? Насколько "тепло" это встретят
коллегци по цеху?
3. Насколько в целом целесообразно использование CPS в вашем рабочем языке программировании? Я рассмотрел пример CPS на Go, и мне показался
пример с БД неподходящим имхо [7].

Таким образом, я бы применял данный подход не столько к проектам, сколько к конкретным ситуациям, касающихся проектов. Если 
язык поддерживает функции, как объекты первого порядка, и функции высшего порядка, то вопрос де/ре-функционализации всегда имеет место быть.
По поводу CPS -- на данный момент лишь вижу как способ дизайна кода.

Источники:
1. https://stackoverflow.com/questions/50958042/continuation-passing-style-and-concurrency
2. https://vk.com/@lambda_brain-cps-2
3. https://vk.com/@lambda_brain-cps-3
4. https://www.educative.io/answers/what-are-the-synchronous-and-asynchronous-cps-in-nodejs
5. https://matt.might.net/articles/by-example-continuation-passing-style/
6. https://vk.com/@lambda_brain-repeatable-execution
7. https://github.com/Q69K/using-cps-in-golang
8. https://en.wikipedia.org/wiki/Continuation-passing_style
