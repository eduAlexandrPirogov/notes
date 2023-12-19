# Дополнительная сложность -- мать всех запашков кода (2)

Я буду приводить части кода по дате написание (от самого старого, до самого нового).

# Пример 1. 01.06.2022. 

```cpp
void MongoWrapper::find(MongoSelectQuery& query, std::string& collection)
{
	MongoWrapper::query_result.clear();

	mongocxx::collection coll = MongoWrapper::db[collection];
	document doc{};

	auto filters = query.getWhere();
	short filters_size = filters.size();

	for (short i = 0; i < filters_size; i++)
	{
		doc << filters[i]->getFirst() << open_document;
		double val = MongoWrapper::convert_to_double(filters[i]->getSecond());
		if (val != -1.0)
			doc << filters[i]->getWhereOperator() << val;
		else
			doc << filters[i]->getWhereOperator() << filters[i]->getSecond();
		doc << close_document;
	}

	std::vector<MongoOrderParam*> orders = query.getOrderField();
	std::vector<MongoOrderParam*>::iterator order_it = orders.begin();

	document doc_test{};
	for (; order_it != orders.end(); order_it++)
	{
		doc_test << (*order_it)->getOrderField() << ((*order_it)->getType());
	}


	std::vector<std::string> project = query.getProjection();
	std::vector<std::string>::iterator project_it = project.begin();
	std::vector<std::string>::iterator project_end = project.end();

	document projection{};
	for (; project_it != project_end; project_it++)
	{
		projection << (*project_it) << 1;
	}

	auto opts = mongocxx::options::find{};
	opts.sort(doc_test.view());
	opts.projection(projection.view());
	mongocxx::cursor cursor = coll.find(doc.view(), opts);
	for (auto doc : cursor) {
		std::string t = bsoncxx::to_json(doc);
		Json* p = new Json{ t };
		MongoWrapper::query_result.push_back(p);
	}
}
```

-- Могу ли я чётко сформулировать, никуда больше не перескакивая, что этот код делает? Название метода говорит о том, что это связано с find-запросом mongo, 
но тут такое нарушение SRP, что название метода не соответствует действительности. Придется заглядывать в другие методы, дабы понять, что делает одна строка
во всей плеяде этого метода.
-- Похоже ли это всё на то, что действительно должно занимать "10 строк и 2 условия"? Однозначно нет.


# Пример 2. 01.12.2022

```cpp
void CommandParserWithoutConfig::parse()
{
    std::unique_ptr<ArgumentsFactory> argumentsFactory = std::make_unique<ArgumentsFactory>();
    arguments = std::move(argumentsFactory->createArgumentInstance(givenCommand, args));
    
    argumentsFactory->create_status() == true ? 
        current_parse_status = PARSE_STATUS_OK :
        current_parse_status = PARSE_STATUS_ERROR;
};

```

-- Могу ли я чётко сформулировать, никуда больше не перескакивая, что этот код делает? Создает новый экзмепляр типа Arguemnts на основе фабрики
и устанавливает статус создания экземпляра. С одной стороны мне не нужен никакой файл, чтобы понять смысл кода, но с другой стороны
стоит посмотреть код факторки, чтобы понять, а какие в принципе статусы создания возможны? 
-- Похоже ли это всё на то, что действительно должно занимать "10 строк и 2 условия"? Да.

# Пример 3. 06.01.2023

```go
// Package memtrack collects system's metrics
// To see avaible metrics see gauges.go
package memtrack

import (
	"log"
	"memtracker/internal/config/agent"
	"memtracker/internal/function"
	"memtracker/internal/memtrack/client/http"
	"memtracker/internal/memtrack/client/rpc"
	"memtracker/internal/memtrack/trackers"
	"memtracker/internal/metrics"
	"strconv"
	"time"
)

type ClientNet interface {
	Send(metrics []metrics.Metricable)
}

// clientType for different types of server.
var clientType map[bool]func() ClientNet = map[bool]func() ClientNet{
	true:  newRPC,
	false: newHTTP,
}

// newRPC returns new instance of RPC server
func newRPC() ClientNet {
	return rpc.New()
}

// newHTTP returns new instance of RPC server
func newHTTP() ClientNet {
	return http.NewClient()
}

// Collects all types of metrics
// Reads and updates metrics
type memtracker struct {
	MetricsContainer trackers.MetricsTracker
}

// Read metrics and send it to given with given http.Client
type httpMemTracker struct {
	Host           string
	PollInterval   int
	ReportInterval int
	client         ClientNet
	memtracker
}

// ReadAndSend Starts to read metrics
//
// readInterval -- how often read metrics
//
// sendInterval -- how often send metrics to server
//
// WARNING: Race condition appears
func (h httpMemTracker) ReadAndSend() {
	// TODO add here job queue
	readTicker := time.NewTicker(time.Second * time.Duration(h.PollInterval))
	sendTicker := time.NewTicker(time.Second * time.Duration(h.ReportInterval))
	for {
		select {
		case <-readTicker.C:
			go h.update()
		case <-sendTicker.C:
			go h.send()
		}
	}
}

// Sends metrics to given host
func (h httpMemTracker) send() {
	// #TODO add here worker pool
	h.client.Send(h.MetricsContainer.Metrics)
}

// updates values of tracking metrics
func (h httpMemTracker) update() {
	h.MetricsContainer.InvokeTrackers()
}

// NewHTTPMemTracker Creates new instance of HttpMemTracker
//
// Pre-cond: Given client instance and host = addr:port
//
// Post-cond: returns new instance of httpMemTracker
func NewHTTPMemTracker() httpMemTracker {
	cfg := agent.ClientCfg
	pollInterval := cfg.PollInterval[:len(cfg.PollInterval)-1]

	poll, err := strconv.Atoi(string(pollInterval))
	function.ErrFatalCheck("", err)

	reportInterval := cfg.ReportInterval[:len(cfg.ReportInterval)-1]

	report, err := strconv.Atoi(string(reportInterval))
	function.ErrFatalCheck("", err)

	client := clientType[agent.ClientCfg.RPC]()
	log.Printf("Client is %v", client)
	return httpMemTracker{
		PollInterval:   poll,
		ReportInterval: report,
		memtracker:     memtracker{trackers.New()},
		client:         client,
	}
}
```

-- Могу ли я чётко сформулировать, никуда больше не перескакивая, что этот код делает? Вполне. Да, мне помогает память, но наличие комментариев и 
небольшие методы способствуют этому. Да, можно вынести ReportIntevall и PollInterval в отдельный АТД, с чем соглашусь, но посколкьу логика проста, 
то я предпочел их не выносить, дабы не обращаться к другим файлам. 
-- Похоже ли это всё на то, что действительно должно занимать "10 строк и 2 условия"? В целом да. Я обращаю внимание на последний метод NewHTTPMemTracker(), где 
явно больше 10 строк, но поскольку одно объявление структуры занимает 5 строк, то в контексте Go я бы сказал, что это ОК. 

# Пример 4.  01.9.2023

```go
// bookingfsm represents FiniteStateMachine for booking.
// Book can be on on the next states: created, choosed, approved ,cancel, finished.
// Each of these states described below.
package bookingfsm

import (
	"booking-service/internal/entity"
	"context"
	"fmt"

	"github.com/looplab/fsm"
	"github.com/rs/zerolog/log"
)

const (
	StateCreated    = "created"  // Booking was created
	StateChoosedCar = "choosed"  // Car for created booking was choosed. Can be performed only from StateCreated
	StateApproved   = "approved" // Booking was approved. Can pe performed only from StateChoosedCar
	StateCanceled   = "canceled" // Booking was canceled. Can be performed from StateCreated and StateChoosedCar
	StateFinished   = "finished" // Booking was finished. Can be performed only from StateApproved
)

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

// Current return current state of BookingFSM
func (b *BookingFSM) Current() string {
	return b.fsm.Current()
}

// ChooseCar tries to perfom transformation to StateChoosedCar.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) ChooseCar() error {
	err := b.fsm.Event(context.Background(), StateChoosedCar)
	if err != nil {
		log.Debug().Msgf("%v", err)
		return err
	}
	fmt.Printf("Booking with id %d has choosen car. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Approve tries to perfom transformation to StateApproved
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Approve() error {
	err := b.fsm.Event(context.Background(), StateApproved)
	if err != nil {
		log.Debug().Msgf("%v", err)
		return err
	}
	fmt.Printf("Booking with id %d has approved. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Cancel tries to perfom transformation to StateCanceled.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Cancel() error {
	err := b.fsm.Event(context.Background(), StateCanceled)
	if err != nil {
		log.Debug().Msgf("cancel event called %v", err)
		return err
	}
	fmt.Printf("Booking with id %d has canceled. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Finish tries to perfom transformation to StateFinished.
// If transformation was performed successfully returns nil, otherwise returns error
func (b *BookingFSM) Finish() error {
	err := b.fsm.Event(context.Background(), StateFinished)
	if err != nil {
		return err
	}
	fmt.Printf("Booking with id %d has finished. Current fsm state: %s\n", b.Book.ID, b.fsm.Current())
	return nil
}

// Booking returns copy of entity.Booking
func (b *BookingFSM) Booking() entity.Booking {
	return b.Book
}
```

-- Могу ли я чётко сформулировать, никуда больше не перескакивая, что этот код делает? Да. Использование готовой, проверенной временем, FSM-ки позволяет мне 
заглянуть в код хоть через год без изучения других файлов мне не составит труда понять смысл пакета.
-- Похоже ли это всё на то, что действительно должно занимать "10 строк и 2 условия"? Да. 

Тут можно заявить, мол, "использовал готовое решение!". Да, но это, во-первых, проверенная библиотека, во-вторых, я не изобретаю велосипед. К тому же, в данной ситуации
это не просто библиотека под конкретную ситуацию, а готовая концепция.


# Пример 5. 01.10.2023

```go
// watcher wrapper aroung in memory btable and db connection
package watcher

type watcher struct {
	s     db.BookingStorer
	table BookingTabler
}

}

// Approve updates btable's record for given entity.Booking
// and update respective record in database/
//
// Pre-cond: given entity.Booking to approve and watcher.
// Updating entity.Booking must exists in table
//
// Post-cond: if transformation was executed successfully
// the writes updates to database. First it's searching record in btable.
// If transformation was failed then cancel the booking
func Approve(b entity.Booking, w *watcher) error {
	if !w.table.ExistsInTable(b) {
		log.Debug().Msgf("cant approve not existing booking")
		return fmt.Errorf("cant approve not existing booking")
	}

	tableErr := w.table.Approve(b)
	if tableErr != nil {
		log.Debug().Msgf("btable err %v", tableErr)
		return tableErr
	}

	dbErr := w.s.Approved(b)
	if dbErr != nil {
		log.Debug().Msgf("db err %v", dbErr)
		return dbErr
	}

	return nil
}

// Cancel updates btable's record for given entity.Booking
// and update respective record in database/
//
// Pre-cond: given entity.Booking to Cancel and watcher.
// Updating entity.Booking must exists in table
//
// Post-cond: if transformation was executed successfully
// the writes updates to database. First it's searching record in btable.
// If transformation was failed then cancel the booking
func Cancel(b entity.Booking, w *watcher) error {

	if !w.table.ExistsInTable(b) {
		log.Debug().Msgf("cant cancel not existing booking")
		return fmt.Errorf("cant cancel not existing booking")
	}

	tableErr := w.table.Cancel(b)
	if tableErr != nil {
		log.Debug().Msgf("%v", tableErr)
		return tableErr
	}

	dbErr := w.s.Canceled(b)
	if dbErr != nil {
		log.Debug().Msgf("%v", dbErr)
		return dbErr
	}

	return nil
}

// Choose updates btable's record for given entity.Booking
// and update respective record in database/
//
// Pre-cond: given entity.Booking to Choose and watcher.
// Updating entity.Booking must exists in table
//
// Post-cond: if transformation was executed successfully
// the writes updates to database. First it's searching record in btable.
// If transformation was failed then cancel the booking
func Choose(b entity.Booking, w *watcher) error {

	if !w.table.ExistsInTable(b) {
		log.Debug().Msgf("cant Choose not existing booking")
		return fmt.Errorf("cant Choose not existing booking")
	}

	tableErr := w.table.Choose(b)
	if tableErr != nil {
		log.Debug().Msgf("%v", tableErr)
		return tableErr
	}

	dbErr := w.s.CarChoosed(b)
	if dbErr != nil {
		log.Debug().Msgf("%v", dbErr)
		return dbErr
	}

	return nil
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
		log.Debug().Msgf("%v", dbErr)
		return -1, dbErr
	}

	b.ID = id
	tableErr := w.table.Create(b)
	if tableErr != nil {
		log.Debug().Msgf("%v", tableErr)
		return -1, tableErr
	}
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

// Finish updates btable's record for given entity.Booking
// and update respective record in database/
//
// Pre-cond: given entity.Booking to Finish and watcher.
// Updating entity.Booking must not exists in table
//
// Post-cond: if transformation was executed successfully
// the writes updates to database. The records MUST NOT EXISTS in btable.
// If it exists returns error otherwise returnes the result of transforamtion.
func Finish(b entity.Booking, w *watcher) error {
	if w.table.ExistsInTable(b) {
		log.Debug().Msgf("cant finish existing booking")
		return fmt.Errorf("cant finish existing booking")
	}

	dbErr := w.s.Finish(b)
	if dbErr != nil {
		log.Debug().Msgf("%v", dbErr)
		return dbErr
	}

	return nil
}
```

-- Могу ли я чётко сформулировать, никуда больше не перескакивая, что этот код делает? И да, и нет. Логика взаимодействия с БД довольна проста, но 
при работе с bookingTabler могут возникнуть вопросы по обработке ошибок и придется обращаться к другому файлу. Но комментарии значительно облегчают жизнь.
-- Похоже ли это всё на то, что действительно должно занимать "10 строк и 2 условия"? Из-за go-подхода с обработкой ошибок, на одну ошибку нужно будет 4 строку кода.
Да и количество условий растет из-за этого, так что такое себе вышло.

# По итогу

При обзоре относительно давно написанного кода, я заметил две основные тенденции, которые позволяют понять назначение кода:
1) наличие комментариев
2) небольшие программные компоненты

Комментарии сужают множество наших домыслов касательно кода, ведь одна реализация может соответствовать многим спецификациям. Особенно хорошо, если написано 
не просто, что делает компонент, а что нужно для его работы и что получится в итоге (пред и пост-условия). 
Что касательно небольших компонентов, то ситуация интереснее. С одной стороны, чем меньше компонент, тем больше композиции и собственных АТД, а значит больше файлов, в 
которых они расположены. Вроде бы придется перескакивать и изучать другие файлы. Но во многих IDE достаточно навестись на интересуемую вызываемую функцию, как 
нам в небольшой окне будут  выведена сигнатуры функции и комментарии. Речь тут больше скорее всего про изучение реализации вызываемого компонента, а не просто перех файла.
В идеале, конечно, когда вызываемая функция, с определенным типом данных и названием говорит, что будет она делать.

Мне нравится тенденция в мышлении, когда читая код, будь-то свой и чужой, у меня автоматически возникает мысль, что "можно сделать лучше" и вместе с этой мыслью имеется
арсенал по улучшению кодовой базе.
