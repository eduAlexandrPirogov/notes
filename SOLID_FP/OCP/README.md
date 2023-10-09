# OCP с т.зр. ФП

# Пример 1

```go
package function

type onSuccessFunction func()

func CompareStringsDo(check string, with string, actionOnTrue onSuccessFunction) {
	if check == with {
		actionOnTrue()
	}
}

func CompareStringsDoOthewise(check string, with string, actionOnTrue onSuccessFunction, actionOnFalse onSuccessFunction) {
	if check == with {
		actionOnTrue()
		return
	}

	actionOnFalse()
}

func CompareBoolssDo(check bool, with bool, actionOnTrue onSuccessFunction) {
	if check == with {
		actionOnTrue()
	}
}
```

# Пример 2



# Пример 3



# Пример 4

