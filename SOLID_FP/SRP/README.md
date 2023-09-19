# SRP с т.зр. ФП

Для начала приведу примеры с комментариями, а затем вывод.

# Пример 1

В драйвере для Postgres'a имеется строитель, который создает экземпляры message.Messager.
Происходит это следующим образом: считывается поток байтов, чтобы отправить это базе данных, и сообщение строится по частям.
Приняв весь поток, вызываем Build,  которые создает нужный формат байтов, но прежде, чем их отправить, нужно им проставить длину сообщения в байтах.
Было:
```go
func (s *StartupBuilder) Build() message.Messager {
  binary.BigEndian.PutUint32(s.startup.S, uint32(conf.PROTOCOL))
  s.startup.S = append(s.startup.S, '\x00')

  // Проставляет длину байта
  l := make([]byte, 4)
  binary.BigEndian.PutUint32(l, uint32(len(s.startup.S)+4))
  s.startup.S = append(l, s.startup.S...)

  return s.startup
}
```

Проставить длину сообщений нужно для всех типов сообщений драйверов. Вынесес это как в отдельную функцию:

Стало:
```go
//package bytes
// AssignBytesLength addes to the front of buffer bytes count including self
// It uses 32 bits for length
//
// Pre-cond: given slice of byte
//
// Post-cond: return slice of bytes with set length
func AssignBytesLength(buffer []byte) []byte {
  l := make([]byte, 4)
  binary.BigEndian.PutUint32(l, uint32(len(buffer)+4))

  buffer = append(l, buffer...)
  return buffer

// package builder
func (s *StartupBuilder) Build() message.Messager {
  binary.BigEndian.PutUint32(s.startup.S, uint32(conf.PROTOCOL)) // #1
  s.startup.S = append(s.startup.S, '\x00')                      // #2
  s.startup.S = bytes.AssignBytesLength(s.startup.S)
  return s.startup
}

}
```

Функция AssignBytesLength отвечает только за установку длины сообщений и более ничего.
Что касается метода Build(), то он не только стал чище, но и явно прослеживается логика создания сообщения.
Может возникнуть вопрос, почему не вынести строчки #1 и #2 в отдельную функцию, ведь Build(), получается,
отвечает не только за создания, ну и установку версии, что нарушает, казалось бы SRP.

Но установка протокола -- единственный случай для всех сообщений, во-первых, и выносить в отдельный компонент не имеет смысла.
Во-вторых, ответственность Build() заключается в создании формата байтов, готовых к отправке, и поскольку для StartupMessage нужно установить
 не только длину, но и протокол, то Build вполне будет соответствовать SRP.


# Пример 2

А теперь рассмотрим пример, в котором нарушил SRP.
Суть трех методов заключается в считывании потока байтов от Postgres'a.
Ответ состоит из:
1) Тэга
2) Размера сообщения в байтах
3) Сообщения
4) Тэг
5) Размер сообщения
...

То есть, для начала нужно считать тэг, затем размер сообщения и само сообщение:
```go
func (d *dial) receive() (message.Messager, error) {
  tag, _ := d.r.ReadByte()
  msgBuilder := factory.Build(tag)
  res, err := d.rcvStream(tag, msgBuilder)
  return res, err
}

// Считываем стрим по тэгам и создаем соответствующее сообщение
func (d *dial) rcvStream(tag byte, b builder.MessageBuilder) (message.Messager, error) {
  for tag != 90 {
    msg := d.rcvMsg(tag)
    b.AssignMessage(msg)

    tag, _ = d.r.ReadByte()
  }
  d.rcvMsg(tag)
  return b.Build(), nil
}

// счиитываем сообщение
func (d *dial) rcvMsg(tag byte) []byte {

  lenPart := make([]byte, 4)
  io.ReadFull(d.r, lenPart)
  n := binary.BigEndian.Uint32(lenPart)

  msg := make([]byte, n-4)
  io.ReadFull(d.r, msg)
  return msg
}
```

Начнем с rcvMsg. Он считывает и длину сообщения, и само сообщения. Можно ли вынести считывание длины сообщение в отдельный метод, например, readMsgLen()?
С одной стороны это явно укажет другому разработчику, что для начала нужно считать длину сообщения. Но если сделать так, то, по-хорошему, нужно
будет добавить метод readMsg(n int), который будет считывать n байтов сообщения, и получается, что rcvMsg будет состоять из двух строк, плюс,
добавится два метода.
Я больше склоняюсь к оставить в таком виде, т.к. протокол сообщений Postgres'a явно разделяет тэги и сообщения, посему чтение длины сообщения и самого сообщения
можно оставить в таком виде.

rcvStream -- метод, который отвечает за формирование сообщения по мере чтения байтов. Т.е он делегирует чтение rcvMsg и создание объекта b builder.MessageBuilder.
Все-таки тут нарушается SRP, т.к я мог спокойно считать все байты, которые передаются по протоколу (грубо говоря по сети), и затем, на основе считанных байтов 
создать сообщение. И rcvStream не должен создавать инстанс сообщения.


То есть  схема такова:
1) Считываем поток байтов по сети и по мере поступления формируем инстанс сообщения

А должна схема такова:
1) Считали поток байтов по сети
2) Сохранили в буфере
3) На основе буфера создали сообщение


# Пример 3 и 4

Мне очень нравится в Go создавать пакеты, в которых имеются лишь функции. 
SRP следовать в таком случае довольно просто. Например, если нужно валидировать строку, то:
```go
package validator

import "regexp"

const titleReg = "^[a-z](?:_?[a-z0-9]+)*$"

// ValidateTitle valides given string
//
// Pre-cond: given title. Title must contain only letters and numbers
// Title can't start with numbers
//
// Post-cond: return true if title is valid
// Otherwise return false
func ValidateTitle(title string) bool {
  b, err := regexp.MatchString(titleReg, title)
  if err != nil {
    return false
  }

  return b
}
```


Или же, проверить подсеть:
```go
package verifier

import (
  "log"
  "memtracker/internal/config/server"
  "net/http"
  "net/netip"

  "google.golang.org/grpc/metadata"
)

const xRealIP = "X-Real-IP"

func IsContainerSubnetHTTP(r *http.Request) bool {
  cameIP := r.Header.Get(xRealIP)
  ip, parseErr := netip.ParseAddr(cameIP)
  subnet, parsePrefixErr := netip.ParsePrefix(server.ServerCfg.Subnet)
  return parseErr == nil && parsePrefixErr == nil && subnet.Contains(ip)
}

func IsContainerSubnetGRPC(md metadata.MD) bool {
  if server.ServerCfg.Subnet == "" {
    return true
  }

  cameIP := md.Get(xRealIP)
  if len(cameIP) == 0 {
    return false
  }

  ip, parseErr := netip.ParseAddr(cameIP[0])

  subnet, parsePrefixErr := netip.ParsePrefix(server.ServerCfg.Subnet)
  log.Println(ip)
  return parseErr == nil && parsePrefixErr == nil && subnet.Contains(ip)
}
```

В моем мозгу уже выстроился паттерн, что, если нужно обработать примитивный тип данных, то для этого нужно просто создать функцию и все.

По итогу:
