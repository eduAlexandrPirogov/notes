# Инварианты и качественный код (1)

Поскольку в Go нет assert'ов в смысле С++ (https://golang.org/doc/faq#assertions), то вместо них я написал предикат, который в ложных случаях выдает панику.

# Пример 1

Пример, где создавалась карта заказов. Заказ может быть в одном из нескольких состояний 

```go
const (
	StateCreated    = "created"  // Booking was created
	StateChoosedCar = "choosed"  // Car for created booking was choosed. Can be performed only from StateCreated
	StateApproved   = "approved" // Booking was approved. Can pe performed only from StateChoosedCar
	StateCanceled   = "canceled" // Booking was canceled. Can be performed from StateCreated and StateChoosedCar
	StateFinished   = "finished" // Booking was finished. Can be performed only from StateApproved
)
```

Первая реализация заключалась в использовании структуры заказа Booking, которая хранила переменную текущего состояния.
```go
type Booking struct {
    current string
}
```

При изменения статуса заказа, нередка ситуация, когда нужно было удостовериться в том, что состояние изменено на желаемое, что
порождало множество if-ов. Со временем появились заказы, которые включали себя лишь часть из вышеприведенных состояний. В структуру добавил структуру Set, которая
хранила допустимое состояние для конкретного Booking. Это добавило в код не только проверки на желаемое состояние после изменений, но и 
проверки на то, что измененное состояние входит в множество допустимых состояний конкретного экземпляра Booking.

Все эти заказы хранились в структуре BookingTable, отражающая ИД пользователя и его заказ:
```go
type BookingTable struct {
	mutex sync.RWMutex
	fsms  map[int]*booking.Booking
}
```

И если требовалось, чтобы в карте BookingTable у пользователя изменилось состояние заказа, то это требовало множество проверок if-ов, что в свою очередь 
повышало цикломатическую сложность кода.

## Уберем логику проверки на структуру данных, в которой в принципе невозможно неверное состояние.

Для представления заказа и его состояний лучше всего использовать FSM-ку.
В случае, если потребуется для определенных заказов "особенные" состояний и их переходы, то можно в метод New передать карты fsm.Events.

```go
// BookingFSM is a wrapper for looplabfsm
type BookingFSM struct {
	Book entity.Booking
	fsm  *fsm.FSM
}

// Created new BooingFSM instance
//
// Pre-cond: given id and userID
//
// Post-cond: instance was created and pointer to it was returned
func New(id, userID int) *BookingFSM {
	return &BookingFSM{
		Book: entity.Booking{
			ID:     id,
			UserID: userID,
		},
		fsm: fsm.NewFSM(
			StateCreated,
			fsm.Events{
				{Name: StateCreated, Src: []string{StateCreated}, Dst: StateCreated},
				{Name: StateChoosedCar, Src: []string{StateCreated}, Dst: StateChoosedCar},
				{Name: StateApproved, Src: []string{StateChoosedCar}, Dst: StateApproved},
				{Name: StateCanceled, Src: []string{StateCreated, StateChoosedCar}, Dst: StateCanceled},
				{Name: StateFinished, Src: []string{StateApproved}, Dst: StateFinished},
			},
			fsm.Callbacks{},
		),
	}
}
```

Таким образом, нам вообще не потребуется проверять текущее состояние заказа в силу работу (и коректной реализации) конечного автомата.

## Добавим ассерты

Добавлять ассерты в FSM не стоит, поскольку мы подразумеваем, что ее реализация корректна. Мы вернемся к BookingTable и рассмотрим один из методов:

```go
// Approve tries to perfom transformation to StateApproved
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Approve() error {
	err := b.fsm.Event(context.Background(), StateApproved)
	if err != nil {            <------------------------------------------------------------------- изменим положение дел здесь
		log.Debug().Msgf("%v", err)
		return err
	}
	log.Debug.Msgf("Booking with id %d has approved. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}
```

И усилим ассертом проверку err на nil:

```go
// Approve tries to perfom transformation to StateApproved
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Approve() error {
  var err error
	err = b.fsm.Event(context.Background(), StateApproved)
  f.assert(err, nil) <---------------------------------------------------------------------------- заменили проверку на кастомный ассерт
	return err
}
```

Но есть нюанс касательно возврата ошибок и assert.

# Пример 2

Реализация драйвера для Postgres'а, суть которого довольна проста, я пропущу момент с авторизацией и объясню момент, связанный с выполнение запросов.
Клиент отправляет некоторую команду, состоящую из тэга и payload'а, а сервер в ответ, присылает поток байтов, в зависимости от запроса клиента.
Ответ от сервера состоит из тега, мета-информации на основе которых происходит чтение payload'а. 

Когда я делал этот драйвер за 100 строк кода, то для декодирования сообщения от сервера написал следующий кусок кода:

```go
// receive reads stream from backend
func receive(r *bufio.Reader) {
	tag, _ := r.ReadByte()
	for tag != 90 {
		readMsg(r)
		tag, _ = r.ReadByte()
	}
	readMsg(r)
}

// readMsg reads message part after tag
func readMsg(r *bufio.Reader) []byte {
	n := readMsgLen(4, r)
	msg := make([]byte, n)
	io.ReadFull(r, msg)
	return msg
}

// readMsgLen reads len of response message from backend
func readMsgLen(n int, r *bufio.Reader) int {
	lenPart := make([]byte, n)
	io.ReadFull(r, lenPart)
	return int(binary.BigEndian.Uint32(lenPart)) - 4
}
```

Код довольно простой, но не покрывает 100% ответов от сервера, как например, авторизация, где может прийти лишь один тэг. Да и ситуация с чтением
количества байтов payload'а сообщения от сервера довольно щепетильная -- если мы прочитаем неточное число байтов (больше или меньше), то, в лучшем случае,
вернем некорректные результаты, а в худшем -- программа будет ожидать до тех пор, пока не прочитает все байты, когда их нет, т.е. "зависнет".
Что тут можно сделать? Изучив документацию протокола общения между клиентом и сервера Postgres'а, можно разделить сообщения от сервера на группы:
1) сообщения результатов авторизации
2) сообщения результатов запросов
3) иные соощбения (создание процедур, экспорт данных, ошибок и т.д.)

Для каждого типа сообщения можно создать соответствующий интерфейс:

```go
type AuthDecoder interface {
    VerifyAuth() Status
}

type QueryDecoder interface {
   ReadNext() bool
}

type ErrorDecoder interface {
    Error()
}
```

Для каждого сообщения можно создать отдельную структуру, реализующую соответствующий интерфейс и с соответствующими костантами мета-информации:

```go
package QueryResponse

const   TAG = 'Q'
const   META_INFO_LEN = 24
const ...

type QueryResponse struct {
	Rows []Row
	Header Header
	rowLen int
	r *bufio.Reader
}

func (q QueryResponse) ReadNext() bool {
   // Код для декодирования
}
```

Поскольку Тэг всегда приходит первом, то подойдет  хорошо паттерн фабрика -- мы подаем в метод Create параметр тэг, и, в зависимости от тэга, нам возвращается соответствующий экземпляр.
А при декодировании, чтобы избежать зависания в случае неверного количества считывания байтов, очень хорошо подойдет assert совместно с context'ом, например по тайм-ауту:

```go
func (q QueryResponse) ReadNext(ctx context.Context) bool {
	read := readMsgLen(ctx, lenPart)
	return read > 0
}

func readMsgLen(ctx context.Context, n int) int {
	lenPart := make([]byte, n)
	io.ReadFull(r, lenPart)
	return int(binary.BigEndian.Uint32(lenPart)) - 4
}
```

и извне обработать сигнал cancel() от context'а. В случае, если время вышло, то выкинуть панику:

```go
case <-ctx.Done():
	panic()
}
```

...

Обратите внимание, каков был первый вариант код декодирования сообщений и сколько пришлось привнести для того, чтобы декодирование стало безопаснее: добавили паттерн, контексты, интферфейсы,
для каждого сообщения будет по структуре. 


# Пример 3

Последний пример приведу из кода Docker'а. Есть следующая структура, у которой я убрал часть полей:

```go
// Container holds the structure defining a container object.
type Container struct {
	StreamConfig *stream.Config
	// embed for Container to support states directly.
	*State          `json:"State"`          // Needed for Engine API version <= 1.11
	Root            string                  `json:"-"` // Path to the "home" of the container, including metadata.
	BaseFS          containerfs.ContainerFS `json:"-"` // interface containing graphdriver mount
	...

	// MountLabel contains the options for the 'mount' command 
	MountLabel             string
	ProcessLabel           string
	RestartCount           int
	MountPoints            map[string]*volumemounts.MountPoint
	...

	// Fields here are specific to Unix platforms                                                                    
	AppArmorProfile string
	HostnamePath    string
	...

	// Fields here are specific to Windows
	NetworkSharedContainerID string            `json:"-"`
	SharedEndpointList       []string          `json:"-"`
	LocalLogCacheMeta        localLogCacheMeta `json:",omitempty"`
}
```

Обращаю внимание на MountLabel. Для работы с ними в репозитории есть следующие методы:

```go
// AddMountPointWithVolume adds a new mount point configured with a volume to the container.
func (container *Container) AddMountPointWithVolume(destination string, vol volume.Volume, rw bool) {
	volumeParser := volumemounts.NewParser()
	container.MountPoints[destination] = &volumemounts.MountPoint{
		Type:        mounttypes.TypeVolume,
		Name:        vol.Name(),
		Driver:      vol.DriverName(),
		Destination: destination,
		RW:          rw,
		Volume:      vol,
		CopyData:    volumeParser.DefaultCopyMode(),
	}
}

// UnmountVolumes unmounts all volumes
func (container *Container) UnmountVolumes(volumeEventLog func(name, action string, attributes map[string]string)) error {
	var errors []string
	for _, volumeMount := range container.MountPoints {
		if volumeMount.Volume == nil {   		// <--------------------------------- !!!
			continue
		}

		if err := volumeMount.Cleanup(); err != nil {
			errors = append(errors, err.Error())
			continue
		}

		attributes := map[string]string{
			"driver":    volumeMount.Volume.DriverName(),
			"container": container.ID,
		}
		volumeEventLog(volumeMount.Volume.Name(), "unmount", attributes)
	}
	if len(errors) > 0 {
		return fmt.Errorf("error while unmounting volumes for container %s: %s", container.ID, strings.Join(errors, "; "))
	}
	return nil
}
```

Я бы вынес здесь работу с mount'ами в отдельный пакет, не столько, чтобы разбить код на ответственности, сколько устранить недопустимого состояния:

```go
type Mount struct {
	MountLabel             string
	ProcessLabel           string
	RestartCount           int
	MountPoints            map[string]*volumemounts.MountPoint
}
```

Опасность тут кроется в указателях мапы MountPoints. Зачем в методе UnmountVolumes нужна проверка на nil? Ещё при прохождении лайв-кодинга в Яндекс у меня выработалась привычка, что
если в мапе в качестве ключа может выступать некое "нулевое значение" (ноль в случае, если в качестве ключа выступают числа; nil, если в качестве ключа выступают указатели т.д.), то если ключ равен
этому нулевому значению, мы удаляем этот ключ. В данном случае, это выглядело бы примерно так:

```go
func (m* Mount) unmount(id string) {
    if m.MountPoints[id] == nil {
       delete(m.MountPoints, id)
    }
}
```

Мы не просто вынесли часть кода в отдельную структуру, но и сделали одно из возможных ее состояний невозможным, тем самым устранив множество проверок на nil.

Вернемся к методу UnmountVoluems и строке, выполняющей проверку на nil:

```go
	for _, volumeMount := range container.MountPoints {
		if volumeMount.Volume == nil {   		// <--------------------------------- !!!
			continue
		}
```

Вот тут ассерт с паникой подошел бы идеально. Раз уж мы вынесли работу с Mount'ами в отдельную структуру и подразумеваем, что эта структура не может хранить нулевые указатели, то асссерт будет лишним тому подтверждением
(подтверждением нашим рассуждениям).


```go
	for _, volumeMount := range container.MountPoints {
		f.AssertNotNil(volumeMount)
```


# По итогу

Начну пожалуй с ассертов. Ассерты я пытался использовать ещё года 2 назад, работая джуном, но лишь на локальной разработке. Они присутствуют почти в любом языке или же их легко реализовать самому на свой вкус.
Механизм, казалось бы, тривиальный, но у него есть важная особенность, которая помогает строить качественный и устойчивый код.  И речь не о том, что приложение упадет и мы поймем, в чем дело. Вспомним материал "4 признака 
качественног кода", где шла речь о том, что каждая строка не просто должна следовать SRP, но ее пост-условие будет пред-условием для следующей. Так вот ассерты в данных ситуациях заходят, что называется "на ура". 
Очень много ошибок ловил, используя эти ассерты. 
Но использовать ассерты на стейдже...это больше выносится на обсуждение с командой. В командах, где больше 100 человек, мне кажется, это подход не подойдет, а вот где команды по 10 человек -- самое то. Но это уже домыслы, не более.

Теперь касательно создания структур данных без недопустимого состояния. Во-первых, мне было сложно найти примеры, поскольку бОльшая часть проектов заключается в "перетаскивании json'чиков" и там, несмотря на то, что пишут код 
с явным состоянием, применение этой техники будет избыточным (обычно файлы не превышают 2 тысяч строк кода). Подобное разбиение привнесет файлы по 100 строк кода и порой придется слишком часто перескакивать по файлам.
Но вот что касается разработки не продуктовой, то этот метод обязателен к применению. В целом, если говорить он непродуктовой разработки, то тут идет тут работы больше с абстракциями: мы продумываем, какие компоненты
будут отражать ту или иную часть внешнего мира в кодовой базе, как они будут взаимодействовать между собой. И чем дольше мы думаем над этим, тем больше декомпозируем наши компоненты, причем стараемся делать их такими, чтобы
они были как можно более устойчивыми, безопасными. Несмотря на то, что количество файлом растет, но в данном случае это, как мне кажется будет оправданым. 

И пугает тут больше не рост количества файлов, сколько способ взаимодействия наших компонентов. Рассмотрим тот же пример с отрезком и сдвигом. Что если, мы будем делать сдвиг не на дискретные единицы, а в процентах?
Будем ли мы вносить "СдвигВПроцентах" в отдельный компонент? Будет ли Сдвиг в Процентах работать совместно со Сдвигом в Единицах (например сдвиг на 20% + 1)? 
Представим, что мы создали два компонента СдвигПроценты и СдвигЕдинцицы. Тогда я бы думал не только над тем, как Отрезок будет взаимодействовать с компонентами Сдвига, но и над тем, как Сдвиги будут взаимодействовать между собой.
И почему-то мне кажется, что ООП в этих моментах будет выглядеть слабо...
