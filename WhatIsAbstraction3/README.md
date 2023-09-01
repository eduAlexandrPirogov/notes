# Что такое абстракция-2

## Пример 1

Последние несколько дней я разрабаываю выпускной проект для яндекс-практикума - проект "автопарк", но немного переделан. Во-первых, там используются микросервисы
(изучаю параллельно "Создание микросервисов" от Ньюмэна). Во-вторых, немного адаптировал ТЗ.

Один из момент, который присутствует в данном проекте -- бронирование автомобилей: клиен может начать бронирование, просматривать доступные машины,
подтверждать бронирования или же отменять и заканчивать бронирование. Логика бронирования вынесена в отдельный микросервис "Booking". Сложность заключается в том, что 
на бронирование выделяется N минут, бронирование можно отменить только до того момента, когда бронирование было подтверждено, нельзя создать одному клиенту бронирование дважды.
Результаты действий относительно заказа должны сохраняться в БД. Как же я решал данную проблему?

У нас есть сущность заказ, которая имеет состояние и это состояние изменяется из одного в другого, причем имеюся ограничения. Этих сущностей заказа может быть множество, и для каждого
клиент они должны быть единственные. К тому же должны быть параллельные записи в БД относительно заказов.

"Поднимем" конкретную сущность, с ее свойствами изменений и мы получим...конечный автомат! Действительно, мы можем переходить из одного состояние в другое, а их некоторых -- нет.
Начальное состояние будет -- created, а конечно -- finished/canceled. 
Теперь, нам нужно подумать о том, чтобы клиент мог создавать заказ лишь единожды (он не может дважды начать создавать бронирование) и чтобы его заказ "висел" не больше N минут.
Для этого хорошо подойдет структура данных "хэш-таблица" или "словарь", а вот для отслеживания "времени жизнь" заказа отлично подойдет context.Background() из Go. 

Таким образом, у нас имеются следующие "поднятые" абстракции:
1) Конечный автомат, отражающий заказ
2) Хэш-таблица, отражающая отражение (извините за тавтологию) пользователя и его заказа.
3) context для отражения времени жизни заказа

Теперь, нужно "опустить" данные абстракции в наш прикладной мир. В данном случае, это дело незамысловатое: для конечного автомата мы присваниваем состояния и переходы из одного состояие в другой,
пригодные для заказа:

```go
const (
	StateCreated    = "created"  // Booking was created
	StateChoosedCar = "choosed"  // Car for created booking was choosed. Can be performed only from StateCreated
	StateApproved   = "approved" // Booking was approved. Can pe performed only from StateChoosedCar
	StateCanceled   = "canceled" // Booking was canceled. Can be performed from StateCreated and StateChoosedCar
	StateFinished   = "finished" // Booking was finished. Can be performed only from StateApproved
)

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
```

Далее, нужно отразить абстракцию хэш-таблицы <Клиент, Заказ>:

(код может быть намного лучше, обычно я его "улучшаю" в отдельно выделенный день):
```go
// btable struct that wraps map and mutes agains race condition
type btable struct {
	mutex sync.RWMutex
	fsms  map[int]*bookingfsm.BookingFSM
}

// Create creates new record in btable
//
// Pre-cond: given entity with set userID
//
// Post-cond: if instance doesn't exists in btable -- insert it to the table and returns nil.
// Otherwise returns error.
func (bt *btable) Create(b entity.Booking) error {
	if _, ok := bt.fsms[b.UserID]; ok {
		return fmt.Errorf("user already created booking")
	}

	bt.fsms[b.UserID] = bookingfsm.New(b.ID, b.UserID)
	return nil
}

// Choose transform bookingFSM to StateChoosedCar
//
// Pre-cond: given entity.Boking with set userID and CarID
//
// Post-cond: if respective record exists -- trying to make
// transformation and return result of transformation.
// If record doesn't exists then returns error
func (bt *btable) Choose(b entity.Booking) error {
	bt.mutex.RLock()
	defer bt.mutex.RUnlock()
	if val, ok := bt.fsms[b.UserID]; ok {
		bt.fsms[b.UserID].Book = b

		err := val.ChooseCar()
		if err != nil {
			log.Println(err)
			return err
		}
	}
	return nil
}

// Approve transform bookingFSM to StateApproved
//
// Pre-cond: given entity.Boking with set userID
//
// Post-cond: if respective record exists -- trying to make
// transformation and return result of transformation.
// If record doesn't exists then returns error
func (bt *btable) Approve(b entity.Booking) error {
	bt.mutex.RLock()
	defer bt.mutex.RUnlock()
	log.Printf("booking %v was approved", b)
	if val, ok := bt.fsms[b.UserID]; ok {
		bt.fsms[b.UserID].Book = b

		err := val.Approve()
		if err != nil {
			log.Println("err while approving booking: ", err)
			return err
		}

		log.Printf("deleting %v", b)
		delete(bt.fsms, b.UserID)
	}
	return nil
}

// Cancel transform bookingFSM to StateCanceled
//
// Pre-cond: given entity.Boking with set userID
//
// Post-cond: if respective record exists -- trying to make
// transformation and return result of transformation.
// If transformation was successfull then deletes record from btable
// If record doesn't exists then returns error
func (bt *btable) Cancel(b entity.Booking) error {
	bt.mutex.RLock()
	defer bt.mutex.RUnlock()
	if val, ok := bt.fsms[b.UserID]; ok {
		err := val.Cancel()
		if err != nil {
			log.Println(err)
			return err
		}

		log.Printf("deleting %v", b)
		delete(bt.fsms, b.UserID)
		return nil
	}
	return fmt.Errorf("not found")
}

// ExistsInTable checks if record with given entity.Booking is exists
//
// Pre-cond: given entity.Boking
//
// Post-cond: returns existence of given entity.Booking in btable
func (bt *btable) ExistsInTable(b entity.Booking) bool {
	bt.mutex.RLock()
	defer bt.mutex.RUnlock()
	log.Printf("checking booking %v ", b)
	_, ok := bt.fsms[b.UserID]
	return ok
}
```

Теперь, добавим conext для отслеживания состояния заказа:

```go


// Interface for btable
type BookingTabler interface {
	Approve(entity.Booking) error
	Cancel(entity.Booking) error
	Choose(entity.Booking) error
	Create(entity.Booking) error
	ExistsInTable(entity.Booking) bool
}

type watcher struct {
	s     db.BookingStorer
	table BookingTabler
}


// Create updates btable's record for given entity.Booking
// and update respective record in database/
//
// Pre-cond: given entity.Booking to Create and watcher.
// Updating entity.Booking must not exists in table
//
// Post-cond: if entity.Booking exists in table then returns error
// Otherwise given entity.Booking was added to btable and record
// was added to database
func Create(b entity.Booking, w *watcher) (int, error) {

	if w.table.ExistsInTable(b) {
		return -1, fmt.Errorf("booking already created")
	}

	id, dbErr := w.s.Created(b)
	if dbErr != nil {
		log.Println(dbErr)
		return -1, dbErr
	}

	b.ID = id
	tableErr := w.table.Create(b)
	if tableErr != nil {
		log.Println(tableErr)
		return -1, tableErr
	}

        // Контекст
	go func() {
		time.AfterFunc(time.Minute*1, func() {
			tableErr := w.table.Cancel(b)
			if tableErr == nil {
				w.s.Canceled(b)
			}
		})
	}()
	return id, nil
}
```

Мы "собрали" логику хэш-таблицы и конечного автомата совместно с логикой работы БД. 
Таким образом, у нас получилось, что имеется watcher, который "следит" за хэш-таблицей заказов и БД.
Насколько моя реализация отражает реальный мир? Мне кажется, что пример с конечным автоматом легко расширяется относительно состояний и переходов.
Хэш-таблица может расширяться относительно значения ключа (например, хранить не просто по номер пользователя, и а еще по иным метаданным).

Но что тут точно можео улучшить, это взаимодействие между хэш-таблицей и БД не просто собрав их посредством композиции у watcher, а организовать
message-passing с помощью каналов. То есть, мы "поднимаем" абстракцию взаимодействия между двумя сущностями (нам не важны какие они пока что) до 
message-passing protocol, определяем тип сообщений (будет ли это сообщение об ошибке или о призыве к действию), формируем картину и далее 
применяем (слово опускаем не нравится :) абстркцию к конкретному нашему случаю.



По итогу:
Поскольку термин "абстракция" - наш главный компаньон в разработке ПО, то желательно, чтобы он упростил нам путь в создании и поддержки. Термин сложный и неоднозначный, его не стоит путать с антиунификацией или с упаковкой,
дать определение однозначное - задача нетривиальная. Но, как мне кажется, абстракций именно об отражении и если бы меня попросили привести пример такого отражения, то незамедлительно привел бы в качестве примера
классические структуры данных. Например, у нас есть очередь с приоритетом, как структура данных, и она может отражаться как в очередь в поликлиннике, так и в очередь процессов в ОС. Хэш-таблица может отражаться, как 
множество пользователей в системе, так и виртуальная память. Конечный автомат может отражаться в различные конкретные части реального мира, как в моем примере.
Может возникнуть вопрос, мол "насколько долгоживующая/дейсвительна наша абстракция?". Ведь по мере изменений требований, может оказаться, что наша абстракция не подходит. Я бы для этого, помимо абстракции в коде, выявил бы абстракцию
для абстркции (мета-абстракцию), т.е. отражение около-универсума в множество абстракций, которые отражают конкретный мир. Тут мне вспомнилось, что "отношение между интерфейсом и реализацией -- многие-к-многим", и как мне 
кажется, такое утверждение будет верным и по отношению к абстркциям (многиие абстракции могут отражаться различные аспекты конктретного мира, а конкретный мир может удовлетворять многим абстракциям). Правда не уверен, если
конкретный мир удовлетворяет многим абстракциям, не будет ли это одна и та же абстракция (гомоморфизм вроде)? Если реальный мир удовлетворяем многим абстркциям, то возможно эти абстракции имеют общие свойства (если абстркции вообще
имеют свойства), которые присущи мета-абстракции. И вот выявив мета-абстракцию, думается мне, что тогда разработка ПО станет настолько гибкой, насколько это вообще возможно.

А если конкретно по этому материалу, то технику "поднять-применить" использовать буду регулярно, но для начала буду писать код без этой техники, а потом уже применять ее, чтобы видеть разницу между тем, чтобы была, и тем, что стало.
