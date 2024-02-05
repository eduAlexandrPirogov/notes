# E2E ваше всё

Попробовал внедрить Е2Е тесты через следующие инструменты:
1) cypress
2) go-инструментарий

Cypress. 
Инструмент для упрощения ведения E2E. Через докер довольно просто настраивается и запускается, плюс наличие GUI. 
Из минусов, мне кажется в команду такое трудно будет внедрить (не все хотят писать даже юниты).

Go-инструментарий
В Go можно через _test.go файлы написать подобные E2E тесты с бОльшим размахом для себя, но много времени тратится на написание кода. 
Немного поэкспереминтировав, пошел дальше и написал отдельный скрипт, который тестит по сценарию (по URL оптравить json и ожидать такой-то ответ), обернул в Docker и закинул в github actions. 
Да, job-ы длятся намного дольше, плюс нужно поднимать БД, заполнять ее данными и прочее, но эффект приятный.
Причем на Go можно написать не просто посещение определенного URL, а сделать пайплайн из посещений URL и действий, чтобы проверить корректное состояние тестируемого субъекта на каждом шагу.

Теперь о самих впечатлениях E2E. 
Для меня любые тесты, будь-то интеграционные, юнит или новодобавленные в мой арсенал E2E, буду важны всегда. Да, Е2Е я рассматриваю с точки зрения можественного взаимодействия с API и проверки состояния тестируемого субъекта, 
и вроде это можно тестировать через юниты с моками и табами, но Е2Е подразумевает, как мне показалось, тестирование домейна взаимодействия пользователя с системой и последующей ее реакции, а не тестирование домейна входных данных тестируемного компонента.
Например, при тестировании сборщика метрик, где через API ему посылаются метрики и аггрегируются, я с Е2Е мог легко протестировать идемпотентность на уровне аттрибутов метрики и то, что она будет сохранена в БД в ожидаемом мною состоянии. 
Или же протестировал, что при передаче сохранении пачки метрик, нескольких операций чтении, они будут кэшироваться. Подобное не сделать через обычные тесты :)

Ещё бы отметил момент, касательно "краевых" случаев. Мне кажется, что в Е2Е нужно делать упор на "нестандартные" случаи -- например, нестандартный порядок обращения к API, нестандартные ситуации (что будет ,если пользователь обращается к API, когда
компонент, та же кафка, не работает) и прочее. Так получим бОльше выгоды от Е2Е. Мы определяем домейн действий, который можно разбить на дискретные временные отрезки и протестировать для каждого временного отрезка состояние системы. 
Нужно ли автоматизировать Е2Е. Я бы сказал, что этим должны заниматься QA. Не хочу сказать, что этим не должны пользоваться разработчики, но из-за сложности автоматизации и трат времени, я бы не оставил данный этап другой команде.

И последнее, по поводу разницы между интеграционными и Е2Е. Я бы сказал, что разница в виде взаимодействия: в интеграционных тестах речь больше об одном взаимодействии с системой, где проверяется работы нескольких внешних компонентов (БД, шина и прочее),
когда как Е2Е про проверку состояния и реакции системы на множество комплексных действий.