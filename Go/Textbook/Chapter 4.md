# IV. Структуры, методы и интерфейсы

В Go нет классов и наследования классов, однако есть пользовательские типы, структуры, методы, embedding и интерфейсы. Из этих средств строятся инкапсуляция, композиция и полиморфное поведение.

## §4.1. Структуры и методы

### §4.1.1. Объектная модель Go

Go не требует помещать состояние и поведение в класс. Обычно программа разделяет понятия:

- `struct` объединяет данные;
- defined type придаёт данным собственный смысл;
- методы связывают операции с типом;
- интерфейс описывает требуемое поведение;
- пакет определяет границу видимости и инкапсуляции;
- embedding продвигает поля и методы, но не создаёт отношение «подкласс — базовый класс».

Такой стиль часто называют объектно-ориентированным, но он основан на композиции и неявном удовлетворении интерфейсов, а не на иерархиях наследования.

### §4.1.2. Объявление структуры

Структура — последовательность именованных полей, каждое из которых имеет свой тип:

```go
type User struct {
	Name   string
	Age    int
	Active bool
}
```

`User` — defined type с underlying type `struct { ... }`. Поля являются частью структуры типа: их имена, типы, порядок и tags влияют на идентичность безымянного struct type.

Соседние поля одного типа можно объединить:

```go
type Point struct {
	X, Y float64
}
```

#### Нулевое значение

Нулевое значение структуры состоит из нулевых значений полей:

```go
var user User

fmt.Println(user.Name)   // ""
fmt.Println(user.Age)    // 0
fmt.Println(user.Active) // false
```

Полезный Go-тип по возможности проектируют так, чтобы его zero value уже был корректен. Например, `bytes.Buffer`, `sync.Mutex` и `strings.Builder` можно объявить без конструктора. Но это рекомендация, а не требование языка: некоторым типам нужна явная инициализация.

#### Доступ к полю

```go
user.Name = "Ann"
fmt.Println(user.Name)
```

Выражение `user.Name` называется selector expression. Для указателя на структуру разыменование выполняется неявно:

```go
p := &user
p.Age = 30 // сокращение для (*p).Age = 30
```

Если `p == nil`, чтение или запись `p.Age` вызывает panic.

### §4.1.3. Литералы структур

Именованный литерал указывает поля явно:

```go
user := User{
	Name:   "Ann",
	Age:    30,
	Active: true,
}
```

Пропущенные поля получают нулевые значения. Порядок keyed elements не важен.

Позиционная форма перечисляет значения строго в порядке полей:

```go
point := Point{10, 20}
```

В одном литерале нельзя смешивать позиционные и именованные элементы. Для публичных структур почти всегда предпочтительна именованная форма: добавление или перестановка полей тогда реже ломает вызывающий код.

Из другого пакета позиционный литерал структуры с неэкспортируемыми полями использовать нельзя. Именованным литералом также нельзя задавать чужое неэкспортируемое поле.

#### Указатель на литерал

```go
user := &User{Name: "Ann"}
```

Это создаёт значение структуры и возвращает указатель на него. Запись эквивалентна по смыслу:

```go
user := new(User)
user.Name = "Ann"
```

`new(User)` только выделяет обнулённое значение; он не вызывает специальный конструктор, поскольку автоматических конструкторов в Go нет.

#### Безымянная структура

```go
config := struct {
	Host string
	Port int
}{
	Host: "localhost",
	Port: 8080,
}
```

Anonymous struct удобна для локального одноразового значения, тестовых таблиц и промежуточных данных. Если смысл типа повторяется или является частью API, defined type обычно понятнее.

#### Promoted field в литерале

Ключом struct literal может быть имя корректно выбираемого promoted field:

```go
type Address struct {
	City string
}

type Customer struct {
	Address
}

customer := Customer{
	City: "Paris",
}
```

Синтаксиса `Address.City: "Paris"` нет: ключом остаётся identifier `City`, но при проверке он трактуется как selector поля структуры. Тот же объект можно записать вложенным литералом `Customer{Address: Address{City: "Paris"}}`.

Нельзя одновременно задать embedded структуру целиком и поле внутри неё: `Customer{Address: Address{}, City: "Paris"}` содержит перекрывающиеся ключи. Путь к promoted field также не должен проходить через embedded pointer.

### §4.1.4. Копирование и сравнение структур

Присваивание копирует всю структуру по значению:

```go
first := User{Name: "Ann", Age: 30}
second := first
second.Age = 31

fmt.Println(first.Age)  // 30
fmt.Println(second.Age) // 31
```

Копирование поля не обязательно копирует данные за ним. Если структура содержит срез, map, указатель, channel, function или interface, копируется само значение поля. Например, две копии среза могут ссылаться на один underlying array:

```go
type Document struct {
	Data []byte
}

first := Document{Data: []byte{1, 2, 3}}
second := first
second.Data[0] = 99

fmt.Println(first.Data[0]) // 99
```

Для полной независимой копии нужна операция, учитывающая каждый ссылочный компонент. Универсальной встроенной deep copy в Go нет.

#### Сравнимость

Структура сравнима, если сравнимы все её поля:

```go
type Coordinate struct {
	X int
	Y int
}

fmt.Println(Coordinate{1, 2} == Coordinate{1, 2}) // true
```

Структура со срезом, map или function несравнима. Сравнимая структура может быть ключом map.

#### Порядок и padding

Порядок полей влияет на memory layout и возможный padding для выравнивания. Компилятор гарантирует порядок полей, но конкретные размеры и выравнивание могут зависеть от архитектуры. Их можно исследовать через `unsafe.Sizeof`, `Alignof` и `Offsetof`, однако строить переносимую бизнес-логику на предположениях о padding нельзя.

Перестановка полей иногда уменьшает размер структуры, но является преждевременной оптимизацией без измерений. Она также может ухудшить логическую группировку данных.

#### Запрет бесконечного размера

Тип не может содержать себя непосредственно только через структуры и массивы:

```go
// type Node struct {
//     Next Node // ошибка: бесконечный размер
// }
```

Косвенная ссылка через тип фиксированного размера допустима:

```go
type Node struct {
	Value int
	Next  *Node
}
```

Срезы, maps, channels, functions и interfaces также имеют фиксированное представление значения и могут разрывать такую рекурсию.

#### Пустая структура и blank fields

`struct{}` имеет размер 0 и единственное значение. Её используют там, где важен сам факт наличия, а payload не нужен, например в `map[T]struct{}`. Runtime может давать нескольким zero-size values одинаковый адрес или разные адреса: равенство адресов таких объектов не гарантируется.

Поле с именем `_` участвует в layout и в определении сравнимости типа, но к нему нельзя обратиться selector. При сравнении двух структур значения blank fields игнорируются. Такие поля иногда используют для padding или намеренного запрета сравнения, однако это специализированный низкоуровневый приём.

### §4.1.5. Tags полей

После типа поля можно записать строковый tag:

```go
type User struct {
	Name string `json:"name" validate:"required"`
	Age  int    `json:"age,omitempty"`
}
```

Язык сам не придаёт tag семантики. Его читают пакеты через reflection; например, `encoding/json` использует часть `json`. Общепринятый формат — пары `key:"value"`, разделённые пробелами.

Tag является частью идентичности безымянного struct type, хотя при некоторых преобразованиях между структурами tags игнорируются по специальным правилам. Изменять tag во время выполнения нельзя.

`go vet` помогает находить некорректный синтаксис tags. Экспортируемость поля по-прежнему важна: один лишь `json` tag не заставит `encoding/json` сериализовать неэкспортируемое поле.

### §4.1.6. Defined types и методы

Метод — функция с дополнительным receiver parameter:

```go
type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

Вызов:

```go
r := Rectangle{Width: 10, Height: 5}
fmt.Println(r.Area()) // 50
```

Receiver `r` — обычный параметр с особой синтаксической позицией. В Go не используют обязательные имена `this` или `self`; обычно выбирают короткое устойчивое сокращение типа.

#### Где разрешено объявить метод

Receiver base type должен быть defined type, объявленным в том же пакете. Он не может быть interface type или pointer type:

```go
type Celsius float64

func (c Celsius) Fahrenheit() float64 {
	return float64(c)*9/5 + 32
}
```

Методы можно объявлять не только для структур, но и для defined numeric, string, slice, map и других типов.

Нельзя добавить метод к `string`, `time.Time` или другому типу из чужого пакета. Обычно объявляют собственный тип, оборачивают чужой либо создают обычную функцию.

#### Type definition и alias

```go
type UserID int // новый defined type
type ID = int   // alias существующего типа
```

`UserID` имеет собственную идентичность и может получить методы. Alias не создаёт новый тип. Нельзя использовать alias как способ добавить методы к чужому non-local type.

Новый defined type не наследует методы исходного defined type, даже если underlying types одинаковы:

```go
type MyTime time.Time // методы time.Time не переносятся
```

### §4.1.7. Value receiver и pointer receiver

Метод с value receiver получает копию значения:

```go
func (r Rectangle) Scale(factor float64) {
	r.Width *= factor
	r.Height *= factor
}
```

Изменения затрагивают только локальную копию `r`. Чтобы изменить исходное значение, нужен pointer receiver:

```go
func (r *Rectangle) Scale(factor float64) {
	r.Width *= factor
	r.Height *= factor
}
```

```go
r := Rectangle{Width: 10, Height: 5}
r.Scale(2)
fmt.Println(r.Width, r.Height) // 20 10
```

#### Автоматическое взятие адреса

Если значение адресуемо, компилятор преобразует вызов pointer-receiver method:

```go
r.Scale(2) // (&r).Scale(2)
```

Это удобство синтаксиса вызова, но оно не меняет method set типа `Rectangle`. Неадресуемое значение преобразовать так нельзя:

```go
func makeRectangle() Rectangle { return Rectangle{} }

// makeRectangle().Scale(2) // ошибка: результат вызова не адресуем
```

Элемент map также неадресуем:

```go
items := map[string]Rectangle{"x": {Width: 1}}
// items["x"].Scale(2) // ошибка
```

Нужно получить значение, изменить и записать обратно либо хранить указатели.

#### Автоматическое разыменование

Value-receiver method можно вызвать через указатель:

```go
p := &r
fmt.Println(p.Area()) // смысл (*p).Area()
```

Для вызова создаётся копия разыменованного значения. Если `p == nil`, такое неявное разыменование вызывает panic ещё до входа в value-receiver method.

#### Nil pointer receiver

Pointer-receiver method технически может быть вызван с nil receiver:

```go
type Node struct {
	Value int
	Next  *Node
}

func (n *Node) Len() int {
	if n == nil {
		return 0
	}
	return 1 + n.Next.Len()
}
```

Это безопасно только потому, что метод проверяет `n` до разыменования. Язык не выполняет такую проверку автоматически. Нужно документировать, допустим ли nil receiver; большинство типов его не принимает.

#### Как выбирать receiver

Pointer receiver обычно выбирают, если:

- метод изменяет receiver;
- структура велика или её нежелательно копировать;
- структура содержит `sync.Mutex` или другое значение, которое нельзя копировать после использования;
- экземпляры должны иметь устойчивую идентичность;
- другие методы типа уже используют pointer receiver.

Value receiver подходит, если значение небольшое, логически неизменяемое и естественно копируется, как `time.Time`.

Не следует выбирать pointer receiver только из убеждения, что любой указатель всегда быстрее: escape analysis, locality и стоимость разыменований требуют измерения. Для одного типа обычно сохраняют последовательный выбор, хотя язык не запрещает смешивание.

### §4.1.8. Method sets

Method set определяет, какие методы учитываются при удовлетворении интерфейса и какие method expressions допустимы.

Для defined type `T`:

| Тип | Method set |
|---|---|
| `T` | методы с receiver `T` |
| `*T` | методы с receiver `T` и `*T` |

Пример:

```go
type Counter int

func (c Counter) Value() int { return int(c) }
func (c *Counter) Add(delta int) { *c += Counter(delta) }
```

`Counter` имеет в method set только `Value`, а `*Counter` — `Value` и `Add`.

Отсюда важное различие: вызов `c.Add(1)` может компилироваться благодаря автоматическому `&c`, но значение `Counter` не удовлетворяет интерфейсу с методом `Add`. Указатель `*Counter` удовлетворяет.

```go
type Adder interface {
	Add(int)
}

var c Counter
// var a Adder = c  // ошибка
var a Adder = &c    // допустимо
```

#### Method value

```go
r := Rectangle{Width: 10, Height: 5}
area := r.Area
fmt.Println(area())
```

Выражение `r.Area` сохраняет receiver в момент создания. Для value receiver сохраняется копия значения; для pointer receiver — копия указателя.

```go
r := Rectangle{Width: 2, Height: 3}
area := r.Area
r.Width = 100
fmt.Println(area()) // 6: receiver был скопирован раньше
```

#### Method expression

```go
area := Rectangle.Area
fmt.Println(area(Rectangle{Width: 2, Height: 3}))
```

`Rectangle.Area` имеет тип `func(Rectangle) float64`: receiver превращён в первый явный аргумент. Для pointer method используется `(*Rectangle).Scale`.

## §4.2. Интерфейсы

### §4.2.1. Интерфейс как контракт поведения

Basic interface перечисляет методы, которые нужны потребителю:

```go
type Speaker interface {
	Speak() string
}
```

Тип реализует интерфейс неявно — достаточно иметь требуемый method set:

```go
type Dog struct {
	Name string
}

func (d Dog) Speak() string {
	return d.Name + ": woof"
}

var speaker Speaker = Dog{Name: "Rex"}
fmt.Println(speaker.Speak())
```

Нет ключевого слова `implements`, регистрации или ссылки конкретного типа на интерфейс. Благодаря этому интерфейс часто объявляет пакет-потребитель, которому нужно поведение, а не пакет-производитель типа.

#### Сигнатура должна совпадать точно

Имена методов и типы параметров и результатов должны совпадать. Похожая сигнатура недостаточна:

```go
type Loader interface {
	Load(string) ([]byte, error)
}
```

Метод `Load(string) []byte` или `Load(any) ([]byte, error)` интерфейс не реализует. Go не применяет covariance или contravariance к method signatures.

Имена параметров и результатов частью function type не являются, поэтому они могут различаться.

#### Compile-time assertion

```go
var _ Speaker = Dog{}
```

Если `Dog` перестанет удовлетворять `Speaker`, компилятор укажет на эту строку. Для pointer receiver:

```go
var _ io.Writer = (*MyBuffer)(nil)
```

Такое объявление не создаёт полезную runtime-переменную и обычно не выделяет объект.

### §4.2.2. Значение интерфейса

Для рассуждений интерфейсное значение удобно представлять парой:

```text
(dynamic type, dynamic value)
```

```go
var s Speaker
s = Dog{Name: "Rex"}
```

Статический тип `s` всегда `Speaker`. В этот момент dynamic type — `Dog`, а dynamic value — конкретный `Dog`.

Это смысловая модель, не обещание о точном runtime layout или числе machine words.

Через интерфейс доступны только методы его статического типа:

```go
type Dog struct{}

func (Dog) Speak() string { return "woof" }
func (Dog) Fetch()        {}

var s Speaker = Dog{}
s.Speak()
// s.Fetch() // ошибка: Speaker не объявляет Fetch
```

Dynamic dispatch выбирает реализацию `Speak` по dynamic type. Чтобы получить иное поведение, интерфейсу можно присвоить другой реализующий тип.

#### Присваивание копирует concrete value

```go
dog := Dog{Name: "Rex"}
var s Speaker = dog
dog.Name = "Max"
fmt.Println(s.Speak()) // Rex: woof
```

В интерфейс была скопирована value `Dog`. Если присвоить `&dog`, dynamic value будет указателем, и изменения объекта станут видны через него.

### §4.2.3. Nil interface и typed nil

Нулевое интерфейсное значение содержит отсутствие и dynamic type, и dynamic value:

```go
var s Speaker
fmt.Println(s == nil) // true
```

Если внутрь положить типизированный nil pointer, dynamic type уже существует:

```go
var dog *Dog = nil
var s Speaker = dog

fmt.Println(s == nil) // false
```

Интерфейс концептуально равен `(*Dog, nil)`, а не `(nil, nil)`. Вызов `s.Speak()` войдёт в pointer-receiver method либо вызовет panic из-за неявного разыменования value method; результат зависит от method set и реализации метода.

API, возвращающий interface, должен возвращать настоящий `nil`, а не typed nil:

```go
func findSpeaker() Speaker {
	var dog *Dog
	if dog == nil {
		return nil
	}
	return dog
}
```

Та же ловушка относится к `error`, рассмотренному в предыдущей главе.

#### Вызов метода nil interface

```go
var s Speaker
// s.Speak() // panic
```

У nil interface нет dynamic type, по которому можно выбрать реализацию.

### §4.2.4. Empty interface и `any`

Пустой интерфейс не требует методов:

```go
interface{}
```

Предопределённое `any` — точный alias этой записи:

```go
var value any
value = 42
value = "Go"
value = User{Name: "Ann"}
```

`any` не отключает статическую типизацию. Статический тип переменной — `any`, и без проверки dynamic type нельзя выполнять операции конкретного значения.

Использование `any` оправдано, когда данные действительно могут иметь произвольную форму: форматирование, декодирование динамического формата, reflection boundary. Если набор вариантов известен, конкретный тип или узкий интерфейс обычно сохраняет больше проверок компилятора.

#### Срез `[]T` не является `[]any`

```go
values := []int{1, 2, 3}
// var items []any = values // ошибка
```

Представления и операции элементов различаются. Нужно преобразовать каждый элемент:

```go
items := make([]any, len(values))
for i, value := range values {
	items[i] = value
}
```

### §4.2.5. Type assertions

Assertion извлекает concrete value или проверяет другой interface:

```go
var value any = "Go"
text := value.(string)
```

Если dynamic type не равен `string`, single-result form вызывает panic. Безопасная comma-ok form:

```go
text, ok := value.(string)
if !ok {
	fmt.Println("value is not a string")
}
```

При неудаче `text` получает zero value типа `string`.

Для concrete target требуется идентичность dynamic type, а не просто одинаковый underlying type:

```go
type Name string

var value any = Name("Ann")
_, stringOK := value.(string) // false
_, nameOK := value.(Name)     // true
```

Если target сам interface, assertion проверяет, реализует ли его dynamic type:

```go
writer, ok := value.(io.Writer)
```

#### Assertion не является conversion

`value.(T)` работает только для interface expression и проверяет сохранённое dynamic value. `T(value)` — conversion по правилам типов. Это разные операции.

### §4.2.6. Type switch

Type switch выбирает ветвь по dynamic type интерфейса:

```go
func describe(value any) string {
	switch value := value.(type) {
	case nil:
		return "nil"
	case int:
		return fmt.Sprintf("int: %d", value)
	case string:
		return fmt.Sprintf("string: %q", value)
	case fmt.Stringer:
		return value.String()
	default:
		return fmt.Sprintf("other: %T", value)
	}
}
```

Синтаксис `.(type)` допустим только в guard type switch. В case с одним типом переменная получает этот тип. В case с несколькими типами она сохраняет тип исходного interface:

```go
switch value := value.(type) {
case int, int64:
	// value имеет тип исходного interface, а не int или int64
}
```

Порядок может быть важен, если cases являются интерфейсами: первый совпавший case выполняется. Один concrete dynamic type способен реализовать несколько интерфейсов.

Type switch полезен на границе разнородных данных, но длинная проверка каждого известного concrete type иногда означает, что вместо полиморфизма через поведение построен ручной variant dispatch.

### §4.2.7. Сравнение интерфейсных значений

Интерфейсные значения сравнимы. Два равны, если их dynamic types идентичны и dynamic values равны, либо оба nil:

```go
var a any = 10
var b any = 10
fmt.Println(a == b) // true
```

Но если одинаковый dynamic type несравним, сравнение вызывает panic:

```go
var a any = []int{1, 2}
var b any = []int{1, 2}
// fmt.Println(a == b) // panic: []int несравним
```

Сравнение такого интерфейса с `nil` безопасно и сообщает только, пуст ли сам interface.

Эта особенность важна для `map[any]V`: интерфейсный key type допустим, но вставка dynamic value несравнимого типа вызывает panic.

## §4.3. Агрегация и композиция

### §4.3.1. Агрегация и композиция через поля

В языке нет отдельных ключевых слов `aggregation` и `composition`. Это способы моделирования отношения «содержит».

При композиции внешний объект обычно владеет внутренней частью и управляет её временем жизни:

```go
type Engine struct {
	Power int
}

type Car struct {
	engine Engine
}
```

При агрегации внешний объект обычно лишь ссылается на независимо существующий объект:

```go
type Department struct {
	manager *Employee
}
```

Для языка оба примера — обычные поля. Семантика владения определяется API, документацией и логикой программы. GC не выражает ownership: он лишь сохраняет достижимые объекты.

#### Именованное поле и явная делегация

```go
type Logger struct{}

func (Logger) Log(message string) {
	fmt.Println(message)
}

type Service struct {
	logger Logger
}

func (s Service) Start() {
	s.logger.Log("starting")
}
```

Именованное поле делает зависимость и маршрут вызова явными. Внешний тип сам решает, какие операции внутреннего объекта публиковать.

### §4.3.2. Embedded fields

Поле без явного имени называется embedded field:

```go
type Service struct {
	Logger
}
```

Имя поля получается из unqualified type name, здесь `Logger`. Поэтому можно обращаться и явно:

```go
service.Logger.Log("starting")
```

А selector метода может быть promoted:

```go
service.Log("starting") // сокращённый selector
```

Продвижение не копирует метод и не меняет его receiver. Вызов `service.Log` фактически вызывает метод embedded `Logger`; внутри метода receiver имеет тип `Logger`, а не `Service`.

#### Что можно embed

Embedded field записывается как type name `T` либо как `*T`, где для pointer form `T` должен быть именованным non-interface type. Сам `T` не может быть pointer type.

```go
type Cache struct{}

type Server struct {
	Cache  // value embedding
	*Logger // pointer embedding
}
```

Нельзя одновременно embed типы с одинаковым результирующим именем поля, например `T` и `*T`.

#### Value embedding и pointer embedding

При `T` внешний struct содержит значение `T`; его zero value включает zero value `T`.

При `*T` внешний struct содержит указатель. Его zero value содержит nil, и вызов promoted method может привести к panic, если метод или автоматическое разыменование не допускают nil:

```go
type App struct {
	*Logger
}

var app App
// app.Log("x") // вероятный panic: embedded pointer равен nil
```

Pointer embedding позволяет разделять один внутренний объект и уменьшает копирование, но добавляет nil-состояние и aliasing.

### §4.3.3. Продвижение и конфликты имён

Selector ищет поле или метод на наименьшей глубине embedding. Если на этой минимальной глубине найден ровно один кандидат, он promoted.

```go
type Address struct {
	City string
}

type Customer struct {
	Address
}

c := Customer{Address: Address{City: "Paris"}}
fmt.Println(c.City)         // promoted field
fmt.Println(c.Address.City) // явный путь
```

Прямое поле или метод внешнего типа имеет глубину 0 и скрывает одноимённый promoted member:

```go
type Profile struct{ Name string }

type User struct {
	Profile
	Name string
}
```

`user.Name` обозначает поле `User.Name`, а embedded поле остаётся доступно как `user.Profile.Name`.

#### Неоднозначность

```go
type Left struct{ ID int }
type Right struct{ ID int }

type Pair struct {
	Left
	Right
}
```

`pair.ID` запрещён: на одинаковой минимальной глубине два кандидата. Сама структура допустима, и неоднозначность разрешается явным путём:

```go
fmt.Println(pair.Left.ID)
fmt.Println(pair.Right.ID)
```

Аналогичное правило действует для promoted methods. Конфликт может означать, что внешний тип не удовлетворяет интерфейсу, пока не объявит собственный однозначный метод.

#### Method sets при embedding

Если `S` embed значение `T`:

- method sets `S` и `*S` включают promoted methods с receiver `T`;
- method set `*S` дополнительно включает promoted methods с receiver `*T`.

Если `S` embed `*T`, method sets и `S`, и `*S` включают promoted methods с receivers `T` и `*T`.

Это формальные правила удовлетворения интерфейсов. Возможность написать отдельный вызов через адресуемое значение не должна подменять проверку method set.

#### Embedding не является наследованием

Если `Dog` embed `Animal`, значение `Dog` не становится значением типа `Animal`:

```go
type Animal struct{ Name string }
type Dog struct{ Animal }

dog := Dog{Animal: Animal{Name: "Rex"}}
// var animal Animal = dog // ошибка
var animal Animal = dog.Animal
```

Нет автоматического upcast, virtual override и отношения subtype между структурами. Метод embedded типа, вызванный через внешний, не получает внешний объект и не вызывает «переопределённый» метод внешнего типа виртуально.

```go
type Base struct{}

func (Base) Name() string { return "base" }
func (b Base) Print()     { fmt.Println(b.Name()) }

type Derived struct {
	Base
}

func (Derived) Name() string { return "derived" }
```

`Derived{}.Print()` печатает `base`: receiver метода `Print` — `Base`. Для заменяемого поведения явно используют interface, callback или поле-стратегию.

### §4.3.4. Встраивание интерфейсов

Небольшие интерфейсы можно объединять:

```go
type Reader interface {
	Read([]byte) (int, error)
}

type Writer interface {
	Write([]byte) (int, error)
}

type ReadWriter interface {
	Reader
	Writer
}
```

Type set `ReadWriter` — пересечение требований: concrete type должен реализовать и `Read`, и `Write`.

Если embedded interfaces содержат метод с одинаковым именем, его signatures должны быть идентичны. Идентичный метод включается один раз; конфликтующие signatures делают объявление недопустимым.

Embedding интерфейсов расширяет требования. Это противоположно ошибочной интуиции о «union возможностей»: значений, удовлетворяющих более широкому контракту, становится меньше.

### §4.3.5. Интерфейс как embedded field структуры

Интерфейс можно хранить именованным полем:

```go
type Service struct {
	logger Logger
}
```

Или embed:

```go
type Service struct {
	Logger
}
```

Во втором случае методы dynamic value promoted к `Service`, поэтому внешний тип может сам удовлетворить тому же интерфейсу. Но zero value поля `Logger` равен nil; вызов promoted method без инициализации вызовет panic.

Embedding interface иногда удобно для decorator или adapter, но оно не должно скрывать обязательную зависимость. Именованное поле и явный forwarding method часто яснее:

```go
func (s Service) Log(message string) {
	s.logger.Log("service: " + message)
}
```

Явный метод позволяет изменить, ограничить или дополнить поведение, а не автоматически публиковать весь контракт embedded interface.

## §4.4. Инкапсуляция и зависимости

### §4.4.1. Инкапсуляция на уровне пакета

В Go доступность определяется первым Unicode-символом identifier:

- `Name`, `User`, `NewClient` экспортируются из пакета;
- `name`, `user`, `newClient` доступны только внутри того же пакета.

Нет отдельных модификаторов `public`, `private` и `protected`. Граница инкапсуляции — пакет, а не отдельная структура: любой файл того же пакета имеет доступ к неэкспортируемым полям.

Promotion через embedding не меняет доступность имени: неэкспортируемое поле или метод не становится экспортируемым. `unsafe` способен обходить многие гарантии представления, поэтому языковая инкапсуляция — средство построения API и проверки компилятором, а не криптографическая граница безопасности.

```go
package account

type Account struct {
	balance int64
}

func (a Account) Balance() int64 {
	return a.balance
}
```

Код другого пакета может вызвать `Balance`, но не обратиться к `balance` напрямую.

#### Getter и setter

Идиоматичное имя getter обычно совпадает с существительным без префикса `Get`:

```go
func (a Account) Balance() int64 { return a.balance }
```

Setter, если он действительно нужен:

```go
func (a *Account) SetOwner(owner string) {
	a.owner = owner
}
```

Не нужно механически создавать setter для каждого поля. Операция предметной области лучше сохраняет инварианты:

```go
func (a *Account) Deposit(amount int64) error
func (a *Account) Withdraw(amount int64) error
```

#### Инкапсуляция изменяемых коллекций

Getter может случайно раскрыть внутренний срез или map:

```go
func (u *User) Roles() []string {
	return u.roles // вызывающий сможет менять элементы
}
```

Если требуется защитить состояние, возвращают копию:

```go
func (u *User) Roles() []string {
	return slices.Clone(u.roles)
}
```

Это shallow copy. Для элементов-указателей или вложенных коллекций может понадобиться более глубокая политика копирования. Иногда копирование слишком дорого, и API предоставляет итератор, read-only операции или явно документирует shared ownership.

### §4.4.2. Конструкторы и инварианты

Конструктор — обычная функция по соглашению:

```go
func NewAccount(owner string) (*Account, error) {
	if owner == "" {
		return nil, errors.New("owner is empty")
	}

	return &Account{owner: owner}, nil
}
```

Для одного основного типа пакета используют `New`; для вариантов — `NewClient`, `NewReader` и подобные имена. Специальной перегрузки конструкторов нет.

Конструктор полезен, когда нужно:

- проверить аргументы;
- установить обязательные зависимости;
- нормализовать данные;
- скрыть concrete representation;
- подготовить map, channel или другой компонент, чей nil state непригоден.

Если zero value корректен, конструктор может быть не нужен.

#### Возвращать значение или указатель

Указатель естественен для изменяемого объекта с идентичностью, большой структуры или скрытой синхронизации. Значение удобно для маленького immutable-like типа.

Сам факт использования конструктора не требует возвращать указатель:

```go
func NewPoint(x, y float64) Point {
	return Point{X: x, Y: y}
}
```

#### Неэкспортируемый concrete type

```go
type notifier struct{}

func (notifier) Notify(string) error { return nil }

func NewNotifier() Notifier {
	return notifier{}
}
```

Так пакет полностью скрывает representation. Цена — вызывающий получает только методы интерфейса, а добавление метода в возвращаемый интерфейс может сломать внешние реализации и mocks. Часто лучше вернуть экспортируемый concrete type и принять узкий интерфейс на стороне потребителя.

### §4.4.3. Ограничение реализации интерфейса

Неэкспортируемый метод в интерфейсе обычно не позволяет типам из других пакетов реализовать его:

```go
package token

type Token interface {
	String() string
	isToken()
}
```

Метод `isToken` другого пакета имеет другую package identity, даже при таком же написании. Это создаёт «закрытый» набор реализаций для обычного внешнего кода.

Но такой контракт намеренно уменьшает расширяемость и усложняет тестовые подделки. Его применяют, когда пакет действительно должен контролировать все реализации, а не просто по привычке.

Другая граница — каталог `internal`: пакет внутри него можно импортировать только из разрешённого дерева каталогов. Это ограничивает зависимости на уровне пакетов, а не отдельных методов.

### §4.4.4. Проектирование интерфейсов

#### Интерфейс принадлежит потребителю

Если функции нужны только `Read` и `Close`, ей незачем принимать огромный concrete client. Потребитель может объявить ровно необходимый контракт:

```go
type ReadCloser interface {
	Read([]byte) (int, error)
	Close() error
}
```

Это уменьшает связанность и облегчает тестирование. Производителю не нужно заранее угадывать все интерфейсы, которым удовлетворит его тип.

#### Не создавать интерфейс заранее

Интерфейс из одного concrete implementation без реальной точки замены часто добавляет имена и косвенность, но не абстракцию. Сначала полезно определить поведение потребителя; интерфейс появляется там, где он отделяет действительно изменяемую зависимость.

#### Изменение публичного интерфейса

Добавление метода — breaking change для всех внешних типов, которые его реализуют. Удаление метода расширяет type set, но может менять ожидания пользователей. Публичные интерфейсы поэтому стараются делать маленькими и устойчивыми.

Embedding большого интерфейса автоматически наследует эту стоимость эволюции. Иногда лучше перечислить минимально нужные методы.

#### Pointer to interface почти не нужен

Интерфейсное значение уже хранит dynamic value, которое может быть указателем:

```go
func Use(r io.Reader) {}
```

`*io.Reader` означает указатель на саму interface variable, а не «интерфейс, содержащий указатель». Он нужен лишь в редких случаях, когда функция должна заменить interface variable вызывающей стороны. Обычно значение интерфейса передают напрямую и при необходимости возвращают новое.

### §4.4.5. Dependency inversion на практике

Высокоуровневая логика объявляет минимальное поведение:

```go
type UserStore interface {
	Find(id string) (User, error)
}

type WelcomeService struct {
	users    UserStore
	notifier Notifier
}

func (s WelcomeService) Welcome(id string) error {
	user, err := s.users.Find(id)
	if err != nil {
		return fmt.Errorf("find user: %w", err)
	}
	return s.notifier.Notify("Welcome, " + user.Name)
}
```

Конкретные database и messaging clients адаптируются к этим интерфейсам. В тесте можно передать небольшой fake:

```go
type userStoreFunc func(string) (User, error)

func (f userStoreFunc) Find(id string) (User, error) {
	return f(id)
}
```

Так полиморфизм служит разделению ответственности, а не имитации классовой иерархии.

## §4.5. Полиморфизм

### §4.5.1. Полиморфизм через интерфейсы

Полиморфная функция зависит от поведения, а не от конкретного представления:

```go
type Notifier interface {
	Notify(message string) error
}

func SendWelcome(n Notifier, name string) error {
	return n.Notify("Welcome, " + name)
}
```

Разные типы реализуют контракт независимо:

```go
type EmailNotifier struct{}

func (EmailNotifier) Notify(message string) error {
	fmt.Println("email:", message)
	return nil
}

type SMSNotifier struct{}

func (SMSNotifier) Notify(message string) error {
	fmt.Println("sms:", message)
	return nil
}
```

```go
SendWelcome(EmailNotifier{}, "Ann")
SendWelcome(SMSNotifier{}, "Bob")
```

Компилятор проверяет удовлетворение интерфейсу, а runtime выбирает реализацию метода по dynamic type.

#### Коллекция разных реализаций

```go
notifiers := []Notifier{
	EmailNotifier{},
	SMSNotifier{},
}

for _, notifier := range notifiers {
	_ = notifier.Notify("system update")
}
```

Срез хранит interface values одного статического типа с разными dynamic types.

#### Интерфейс не обязан быть большим

```go
type Flusher interface {
	Flush() error
}
```

Однометодные интерфейсы хорошо комбинируются и легко реализуются. Частое правило проектирования: принимать интерфейсы, достаточные для конкретной операции, а возвращать полезные concrete types. Это не абсолютный закон — публичный interface result нужен, когда именно он является устойчивым контрактом или реализации намеренно скрыты.

### §4.5.2. Function values, adapters и decorators

#### Function value

Если нужна единственная операция без состояния или семейства методов, interface может быть избыточен:

```go
type Formatter func(string) string

func PrintName(name string, format Formatter) {
	fmt.Println(format(name))
}
```

Named function type тоже может иметь методы и удовлетворять интерфейсам.

#### Adapter и decorator

Adapter превращает существующий API в требуемый интерфейс:

```go
type NotifyFunc func(string) error

func (f NotifyFunc) Notify(message string) error {
	return f(message)
}
```

Любую подходящую функцию теперь можно передать как `Notifier`.

Decorator хранит другой интерфейс и добавляет поведение:

```go
type LoggingNotifier struct {
	next Notifier
}

func (n LoggingNotifier) Notify(message string) error {
	fmt.Println("sending notification")
	return n.next.Notify(message)
}
```

Оба паттерна естественно выражаются композицией, без базового класса.

### §4.5.3. Типичные ошибки

#### Ожидание наследования

Embedding не делает внешний тип допустимым там, где нужен embedded concrete type, и не создаёт virtual methods. Для заменяемости определяют интерфейс.

#### Неправильный receiver

Value receiver не сохраняет присваивания полям в исходной структуре. Pointer receiver отсутствует в method set значения `T`, даже если отдельный вызов на адресуемой переменной компилируется.

#### Случайная копия mutex

```go
type SafeCounter struct {
	mu sync.Mutex
	n  int
}
```

Такой тип не следует копировать после начала использования. Все его методы должны иметь pointer receiver; `go vet` с анализатором `copylocks` помогает находить опасные копии.

#### Слишком широкий interface

Интерфейс на десятки методов тяжело реализовать, подделать и развивать. Принимать следует контракт, реально используемый функцией.

#### Возврат typed nil

Interface с dynamic type и nil pointer не равен nil. Возвращающий код должен явно вернуть `nil`.

#### Type assertion без проверки

Single-result assertion уместна, только если несоответствие действительно означает нарушенный инвариант. На внешних данных используют comma-ok или type switch.

#### Утечка внутреннего состояния

Неэкспортируемое поле не защищает коллекцию, если getter возвращает общий срез, map или pointer. Нужно выбрать и документировать ownership.

#### Интерфейс ради mock

Абстракция должна отражать нужды production-кода. Тестируемость — преимущество узкого настоящего контракта, но не причина копировать весь API concrete type в интерфейс.

### §4.5.4. Практическая последовательность проектирования

1. Описать данные небольшим concrete type.
2. Сделать zero value полезным либо предоставить обычный конструктор.
3. Скрыть поля, если прямое изменение может нарушить инварианты.
4. Выбрать value или pointer semantics и применять их последовательно.
5. Использовать именованные поля для явного владения зависимостями.
6. Применять embedding только когда продвижение является частью желаемого API.
7. Объявлять маленький интерфейс рядом с потребителем поведения.
8. Проверять method sets, особенно различие `T` и `*T`.
9. Продумать nil и ownership вложенных ссылочных данных.
10. Добавить compile-time interface assertions там, где связь является важным намерением.

### §4.5.5. Итоговая модель

Структура объединяет поля и копируется как значение. Методы принадлежат defined type из того же пакета; receiver остаётся обычным передаваемым значением или указателем. Method set формально определяет удовлетворение интерфейсам, а удобные автоматические `&` и `*` при вызове не отменяют его правил.

Композиция и агрегация выражаются полями. Embedded field дополнительно продвигает selectors и методы, но не создаёт наследование, upcast или virtual override. Конфликты разрешаются глубиной selector либо явным путём.

Интерфейс задаёт type set через требуемое поведение. Реализация неявна, interface value имеет статический interface type и динамические type и value, а nil interface отличается от интерфейса с typed nil. Type assertions и switches позволяют безопасно исследовать dynamic type.

Инкапсуляция строится вокруг пакета, экспортируемых имён и продуманного API. Полиморфизм возникает через небольшие интерфейсы и function values; выбор механизма зависит от того, требуется ли заменяемый объект с набором методов или единственный callback.
