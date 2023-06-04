# Правила хорошего code review

#1 Обозревать небольшое количество строк кода и #2 Придерживать темпа N строк кода в час. 

Придерживаюсь в некоторой степени.
Проводя code review, я измеряю обычно в количестве сущносостей на единицу времени. Например, 4-6 функций в полачас, 2-3 класса в час, при условии, что класс не более 200 строк, а функции -- не более 50-70.
Это позволяет мне сосредоточиться на конретных моментах, т.к. нашел, что просто "пробежать глазами" весь код крайне неэффективно. 

#3 Отвести достаточное количество времени времени для вдумчивого и неспешного просмотра.

Придеживаюсь.
Общее время -- час, где 45-50 минут -- изучение и 10-15 минут -- отдых. Пытался и целый день code review проводить и более мелкими шагами (по 20-25 минут), но вот 45 минут для меня наиболее удобным стал.

#4 Принуждение авторов кода к документированию/комментированию кода перед code review

Придерживаюсь, обычно требую следующее:
1) Комментарии по общему назначение модуля/класса и как он вписывает в систему
2) Пред и пост условия для методов/функций 

По-крайней мере, мне проще будет понять, что код семнатически соответсвтует комментариям, а если это не так, то я всегда могу написать "вот у тебя в комментариях вот так, а в коде -- другое...".

#5 Устанавливать количественно измеряемые цели в обзорах кода и фиксация метрик.

На работе этого не придерживался, ибо это никому не было нужно. А вот для себя, при ревью своего кода, стараюсь фиксировать следующие метрики:
1) Количество багов на сущность (функцию/класс)
2) Количество времени на фикс бага и причина бага (ушло столько-ко часов, а причина была ...)

С одной стороны уходит на полчаса-час в среднем больше на ревью, но появляется осознанность к написанию кода, это своего рода некоторый дневник.

#6 Использование чек-листов

Не придерживался, хотя это простая и гениальная идея, которую обязательно внесу в свой процесс.

#7 Убедиться, что баги действительно исправлены

В команде это обычно происходит в мессенджерах, где явно пишутся, что нужно исправить и исправил ли, плюс подтверждающие это тесты.
Для себя, оставляю коммендарии #TOFIX Pwhat_to_fix}, чтобы не забыть про них, плюс тесты подтверждающие это. 
То есть увеличивается время на написание тестов, на фикс ошибки и на обновление комментариев, если требуется, примерно на 10-15 минут.

#8. Формированию правильной культуры обзоров кода.

Не придерживались. На моем опыте, code review указывал на ошибки, касательно синтаксиса и фич языка программирования.
За замечания на code review тоже не прилетало "по шапке".

#9 "Большой брат" и влияние метрик.

На моем опыте это происходило следующем образом -- я связывался с менеджерои/старшим разработчиков, мы говорили о замечаниях, к чему они приводят и как их избегать. Все это происходило в дружеском тоне.
Из-за малого размера команды, в которой я работал, обычно все разговоры об классах ошибках происходили за "чаем". Никакие инструменты не использовались.

#10 Эффект эго

Придерживаюсь, т.к. однажды получил "по шапке" за некачественную работу и это сильно ударило по моему эго. С тех пор, за 1 час рабочего дня, пересматриваю код какой-либо и переделываю его.
Среднее количество улучшенного кода примерно 200-300 строк, но зависит от сложности и многих факторов.

#11 Облегчённые обзоры кода

Придерживаюсь в некоторой степени.
Использую только линтеры и статический анализ кода, не более. Далее использую git для рецензирования, ну и мессенджер, не более.
Слишком детальные обзоры отнимают силы и не поддерживались командой.