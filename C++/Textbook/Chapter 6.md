# VI. Шаблоны

Шаблоны позволяют описать семейство функций, классов или других сущностей, поведение которых зависит от типов или значений, известных во время компиляции.

Шаблон — не готовая функция или класс, а правило, по которому компилятор создаёт конкретные специализации. Благодаря этому один алгоритм можно применять к разным типам без копирования исходного кода вручную.

## §6.1. Введение

### Шаблон функции

Обычная функция обмена для `int` работает только с `int`. Шаблон вводит параметр типа `T`:

```cpp
template <typename T>
void swap_values(T& first, T& second) {
    T temporary = first;
    first = second;
    second = temporary;
}
```

```cpp
int x = 1;
int y = 2;
swap_values(x, y);

std::string first = "one";
std::string second = "two";
swap_values(first, second);
```

Для первого вызова компилятор выводит `T = int`, для второго — `T = std::string`. Получаются разные специализации одной шаблонной функции.

Слова `typename` и `class` в объявлении типового параметра означают одно и то же:

```cpp
template <typename T>
struct First {};

template <class T>
struct Second {};
```

`class` здесь не требует, чтобы аргумент был классом: `Second<int>` корректен. `typename` часто яснее показывает, что параметр обозначает произвольный тип.

Шаблон может иметь несколько параметров:

```cpp
template <typename Key, typename Value>
struct Pair {
    Key key;
    Value value;
};

Pair<std::string, int> age{"Alice", 30};
```

### Вывод аргументов

Для шаблона функции компилятор обычно выводит аргументы из параметров вызова:

```cpp
template <typename T>
T max_value(T first, T second) {
    return first < second ? second : first;
}

int result = max_value(10, 20); // T = int
```

Один параметр `T` должен получить одно согласованное значение:

```cpp
// max_value(1, 2.5); // ошибка: T выводится и как int, и как double
```

При выводе шаблонных аргументов компилятор не перебирает произвольные пользовательские преобразования, чтобы привести оба аргумента к общему типу. Тип можно указать явно:

```cpp
double result = max_value<double>(1, 2.5);
```

Теперь `T` уже равен `double`, и `1` преобразуется в `double` как обычный аргумент готовой функции.

Можно явно указать только первые параметры, а остальные вывести:

```cpp
template <typename Result, typename Left, typename Right>
Result add(Left left, Right right) {
    return static_cast<Result>(left + right);
}

double result = add<double>(1, 2.5);
```

`Result` не встречается в типах параметров функции и не может быть выведен, поэтому задаётся явно. `Left` и `Right` выводятся как `int` и `double`.

### Инстанцирование и отложенные ошибки

Само определение шаблона не создаёт все возможные функции:

```cpp
template <typename T>
void print_size(const T& value) {
    std::cout << value.size() << '\n';
}
```

Пока не запрошена конкретная специализация, компилятор не знает, существует ли у `T` метод `size`.

```cpp
std::string text = "abc";
print_size(text); // корректно

// print_size(42); // ошибка при инстанцировании print_size<int>
```

Шаблон проверяется в несколько этапов. Синтаксические и не зависящие от параметров ошибки можно обнаружить сразу, а корректность зависимых операций — после подстановки конкретных аргументов.

Из-за этого определения шаблонов обычно помещают в заголовочные файлы. В месте инстанцирования компилятору нужно видеть полное определение, а не только объявление.

### Виды шаблонов

C++ поддерживает:

- шаблоны функций;
- шаблоны классов и структур;
- шаблоны псевдонимов типов — начиная с C++11;
- шаблоны переменных — начиная с C++14.

```cpp
template <typename T>
using Pointer = T*;

template <typename T>
constexpr bool is_pointer_v = false;

template <typename T>
constexpr bool is_pointer_v<T*> = true;
```

К каждой реально использованной комбинации аргументов относится отдельная специализация.

## §6.2. Перегрузка шаблонных функций

Шаблонные и обычные функции участвуют в одном процессе разрешения перегрузки:

```cpp
template <typename T>
void f(T) {
    std::cout << "template\n";
}

void f(int) {
    std::cout << "int\n";
}
```

```cpp
f(1);    // int
f(1.5);  // template
f<int>(1); // template
```

Для `f(1)` обе функции дают точное совпадение. Когда последовательности преобразований одинаково хороши, нешаблонная функция предпочтительнее шаблонной.

Запись `f<int>(1)` явно требует шаблон, поэтому обычная `f(int)` кандидатом не является.

### Качество преобразований важнее нешаблонности

Нешаблонная функция не выигрывает автоматически:

```cpp
template <typename T>
void g(T) {
    std::cout << "template\n";
}

void g(long long) {
    std::cout << "long long\n";
}

g(1);
```

> Вывод: `template`.

Шаблон выводит `T = int` и получает точное совпадение. Для обычной функции понадобилось бы преобразование `int -> long long`, поэтому она проигрывает. Предпочтение нешаблонной версии применяется только после сравнения качества преобразований и других правил выбора.

### Частичное упорядочивание шаблонов

Если подходят несколько шаблонов, компилятор пытается выбрать более специализированный:

```cpp
template <typename T, typename U>
void process(T, U) {
    std::cout << "different types allowed\n";
}

template <typename T>
void process(T, T) {
    std::cout << "same type\n";
}

process(1, 2);   // same type
process(1, 2.0); // different types allowed
```

Вторая версия применима к меньшему множеству вызовов — оба аргумента должны давать один `T`, — поэтому она считается более специализированной.

Перегрузки по значению и по ссылке часто оказываются неоднозначными:

```cpp
template <typename T>
void use(T);

template <typename T>
void use(T&);

int value = 0;
// use(value); // неоднозначность
```

Обе версии принимают lvalue без содержательного преобразования. Для временного значения `use(0)` версия с обычной `T&` не подходит, поэтому выбирается передача по значению.

Правила для `const T&`, `T&` и `T&&` зависят от категории значения и вывода `T`; нельзя свести их к утверждению, что константная ссылка всегда требует преобразования и всегда хуже. Например, для константного lvalue перегрузка `const T&` может быть единственной ссылочной версией, а для временного значения она связывается напрямую.

### Возвращаемый тип не создаёт перегрузку

Функции нельзя перегрузить только по возвращаемому типу:

```cpp
template <typename T>
int convert(T);

// template <typename T>
// double convert(T); // конфликтующее объявление
```

В месте вызова возвращаемый тип обычно не участвует в выборе перегрузки, поэтому компилятор не смог бы однозначно решить, какую функцию вызвать.

### Параметры по умолчанию

У шаблонного параметра может быть значение по умолчанию:

```cpp
template <typename Result = double, typename T>
Result half(T value) {
    return static_cast<Result>(value) / 2;
}

double first = half(5);
float second = half<float>(5);
```

`Result` берётся по умолчанию, если не указан явно, а `T` выводится из аргумента. В отличие от обычных параметров функции, в списке параметров шаблона функции параметр без значения по умолчанию может идти после параметра со значением, если его удаётся вывести.

## §6.3. Специализации шаблонов

Специализация задаёт отдельную реализацию шаблона для определённых аргументов.

### Полная специализация класса

```cpp
template <typename T>
struct Storage {
    T value;
};

template <>
struct Storage<bool> {
    unsigned char bits = 0;
};
```

`Storage<int>` использует первичный шаблон, а `Storage<bool>` — полную специализацию. Это отдельное определение класса: члены первичного шаблона не появляются в специализации автоматически.

Стандартный `std::vector<bool>` — известный пример специализированного поведения: для логических значений используется отдельная реализация.

### Частичная специализация класса

Часть аргументов можно оставить параметрами:

```cpp
template <typename T, typename U>
struct Relation {
    static constexpr bool same = false;
};

template <typename T>
struct Relation<T, T> {
    static constexpr bool same = true;
};
```

`Relation<int, double>` использует первичный шаблон, а `Relation<int, int>` — частичную специализацию.

Количество аргументов при использовании класса не меняется: у `Relation` по-прежнему два аргумента. В специализации один параметр `T` просто используется сразу в двух позициях.

Можно специализировать форму типа:

```cpp
template <typename T>
struct Traits;

template <typename T>
struct Traits<T*> {
    using Pointee = T;
};
```

Если подходят несколько частичных специализаций, выбирается более специализированная. Если ни одна не является однозначно более специализированной, инстанцирование завершается ошибкой.

### Функции: специализация и перегрузка

Функциональные шаблоны нельзя частично специализировать. Вместо этого создают перегрузку:

```cpp
template <typename T, typename U>
void print_pair(T, U) {
    std::cout << "general\n";
}

template <typename T>
void print_pair(T, T) {
    std::cout << "same types\n";
}
```

Полная специализация функции разрешена:

```cpp
template <typename T, typename U>
void print_pair(T, U) {
    std::cout << "primary\n";
}

template <>
void print_pair<int, double>(int, double) {
    std::cout << "int, double specialization\n";
}
```

Но полная специализация функции не является самостоятельной перегрузкой. Она принадлежит конкретному первичному шаблону. Сначала разрешается перегрузка между обычными функциями и первичными шаблонами, затем для выбранного шаблона проверяется подходящая полная специализация.

Порядок объявлений поэтому может быть существенным:

```cpp
template <typename T, typename U>
void f(T, U) {
    std::cout << "primary #1\n";
}

template <>
void f<int, int>(int, int) {
    std::cout << "specialization of #1\n";
}

template <typename T>
void f(T, T) {
    std::cout << "primary #2\n";
}

f(0, 0);
```

> Вывод: `primary #2`.

Перегрузка `f(T, T)` более специализирована и выигрывает. Специализация первого шаблона не участвовала в сравнении как отдельная функция.

Если специализацию объявить после второго первичного шаблона и она по правилам относится к `f(T, T)`, результат изменится:

```cpp
template <typename T, typename U>
void f(T, U) {
    std::cout << "primary #1\n";
}

template <typename T>
void f(T, T) {
    std::cout << "primary #2\n";
}

template <>
void f<int>(int, int) {
    std::cout << "specialization of #2\n";
}
```

`f(0, 0)` выберет второй первичный шаблон и затем его специализацию.

Из-за такой неочевидной связи для функций обычно предпочитают перегрузки, а специализации чаще используют для классов и переменных. Специализацию нужно объявить до первого использования, которое привело бы к неявному инстанцированию, во всех единицах трансляции, где это важно.

## §6.4. Нетиповые и шаблонные параметры

Параметр шаблона может быть не типом, а значением времени компиляции:

```cpp
template <typename T, std::size_t Size>
struct Array {
    T data[Size];
};

Array<int, 10> values;
```

`Array<int, 10>` и `Array<int, 20>` — разные типы. Значение `Size` доступно внутри определения и может определять размер массива или участвовать в `if constexpr`.

Аргумент должен быть допустимым constant template argument:

```cpp
constexpr std::size_t size = 10;
Array<int, size> first;

std::size_t runtime_size = 10;
// Array<int, runtime_size> second; // ошибка
```

`constexpr` не заставляет любое выражение вычислиться во время компиляции. Оно требует, чтобы объявленная сущность удовлетворяла правилам константного выражения в соответствующих случаях. Если инициализатор не подходит, компилятор сообщит об ошибке.

Нетиповой параметр не обязательно является числом, но на вводном уровне чаще всего используются целые значения и перечисления. Набор допустимых нетиповых аргументов зависит от версии стандарта.

### Шаблон как параметр

Шаблонный параметр может принимать другой шаблон:

```cpp
template <
    typename T,
    template <typename, typename> class Container = std::vector
>
class Stack {
    Container<T, std::allocator<T>> container;
};
```

Параметр `Container` ожидает шаблон с двумя типовыми параметрами. `std::vector` подходит: его второй параметр — аллокатор — имеет значение по умолчанию, но его можно указать явно.

Более гибкая запись принимает дополнительный пакет параметров:

```cpp
template <
    typename T,
    template <typename, typename...> class Container = std::vector
>
class Stack {
    Container<T> container;
};
```

Здесь `Container<T>` может использовать значения по умолчанию остальных параметров. Точные правила сопоставления template-template parameters менялись между версиями стандарта, поэтому при поддержке старых компиляторов явное описание всех параметров бывает переносимее.

Шаблонные параметры, как типовые и нетиповые, могут иметь значения по умолчанию.

## §6.5. Вычисления на шаблонах

Шаблонные специализации можно использовать для рекурсивных вычислений во время компиляции:

```cpp
template <int N>
struct Fibonacci {
    static constexpr unsigned long long value =
        Fibonacci<N - 1>::value + Fibonacci<N - 2>::value;
};

template <>
struct Fibonacci<1> {
    static constexpr unsigned long long value = 1;
};

template <>
struct Fibonacci<0> {
    static constexpr unsigned long long value = 0;
};
```

```cpp
static_assert(Fibonacci<10>::value == 55);
```

Первичный шаблон задаёт рекурсивный шаг, а полные специализации завершают рекурсию. Для `Fibonacci<10>` требуются специализации от `Fibonacci<10>` до базовых случаев. Одна и та же специализация не инстанцируется заново для каждого упоминания в пределах одного контекста компиляции.

Шаблон переменной сокращает запись:

```cpp
template <int N>
inline constexpr unsigned long long fibonacci_v = Fibonacci<N>::value;

static_assert(fibonacci_v<20> == 6765);
```

`inline` у шаблонной `constexpr`-переменной помогает ясно выразить допустимость определения в заголовочном файле.

Такие вычисления выполняются во время компиляции. Рекурсивный шаблон задаёт шаг, а специализации — условия остановки.

## §6.6. Зависимые имена и двухфазная трансляция

Имя внутри шаблона называется зависимым, если его смысл зависит от шаблонного параметра. До подстановки компилятор иногда не знает, обозначает ли оно тип, шаблон или значение.

### `typename`

```cpp
template <typename T>
struct Traits {
    using Value = int;
};

template <typename T>
void create_value() {
    typename Traits<T>::Value value = 0;
    std::cout << value << '\n';
}
```

`Traits<T>::Value` зависит от `T`. Слово `typename` сообщает парсеру, что зависимое квалифицированное имя нужно считать типом.

Без него конструкция могла бы разбираться как выражение:

```cpp
// Traits<T>::Value* pointer;
```

Последовательность токенов допускает интерпретацию как умножение значения `Traits<T>::Value` на переменную `pointer`. Компилятор обязан разобрать синтаксис ещё до знания `T`, поэтому тип нужно отметить явно.

`typename` требуется не для каждого имени с `::`. Если имя не зависит от параметра или контекст уже однозначно требует тип, дополнительные правила могут позволить его опустить. Но в объявлении локальной переменной через зависимый вложенный тип явная запись обычно наиболее понятна.

### Зависимый шаблон и слово `template`

```cpp
template <typename T>
struct Factory {
    template <std::size_t Size>
    using Array = std::array<T, Size>;
};

template <typename T>
void create_array() {
    typename Factory<T>::template Array<10> values{};
}
```

`typename` сообщает, что итоговое имя — тип, а `template` сообщает, что зависимый член `Array` является шаблоном. Без `template` символ `<` можно было бы разобрать как оператор сравнения.

То же правило применяется к вызову шаблонного метода через зависимый объект:

```cpp
template <typename T>
struct Worker {
    template <int Count>
    void run(int value) {
        std::cout << Count * value << '\n';
    }
};

template <typename T>
void execute(int value) {
    Worker<T> worker;
    worker.template run<5>(value);
}
```

Без `template` фрагмент мог бы интерпретироваться как сравнения `worker.run < 5` и `5 > value`.

### Члены зависимого базового класса

```cpp
template <typename T>
struct Base {
    int value = 0;
};

template <typename T>
struct Derived : Base<T> {
    void increment() {
        ++this->value;
    }
};
```

Неквалифицированный поиск при первом разборе шаблона не просматривает зависимые базовые классы. Теоретически специализация `Base<T>` может выглядеть иначе, поэтому простая запись `++value` не обязана найти член.

Сделать имя зависимым можно несколькими способами:

```cpp
++this->value;
++Base<T>::value;
```

Можно также ввести имя:

```cpp
using Base<T>::value;
```

`this->` часто удобнее для методов: оно откладывает поиск до инстанцирования и сохраняет обычную форму обращения к члену текущего объекта.

### Две фазы

Упрощённая модель двухфазной трансляции:

1. При определении шаблона проверяются синтаксис, не зависящие от параметров имена и доступная семантика.
2. При инстанцировании подставляются аргументы и проверяются зависимые выражения.

```cpp
void helper(int);

template <typename T>
void call(T value) {
    helper(0);     // независящий вызов: имя ищется при определении
    process(value); // зависимый вызов: участвует поиск при инстанцировании
}
```

Двухфазный поиск нужен, чтобы смысл шаблона не менялся непредсказуемо в зависимости от того, какие несвязанные объявления оказались между его определением и использованием.

## §6.7. Метафункции и type traits

Метафункция принимает типы или значения времени компиляции и возвращает тип либо константное значение.

### Проверка совпадения типов

```cpp
template <typename T, typename U>
struct IsSame : std::false_type {};

template <typename T>
struct IsSame<T, T> : std::true_type {};
```

```cpp
static_assert(IsSame<int, int>::value);
static_assert(!IsSame<int, double>::value);
```

`std::true_type` и `std::false_type` — специализации `std::integral_constant`, содержащие `value`, преобразование к `bool` и вызываемый оператор.

Шаблон переменной делает интерфейс короче:

```cpp
template <typename T, typename U>
inline constexpr bool is_same_v = IsSame<T, U>::value;
```

### Преобразование типа

```cpp
template <typename T>
struct RemoveReference {
    using type = T;
};

template <typename T>
struct RemoveReference<T&> {
    using type = T;
};

template <typename T>
struct RemoveReference<T&&> {
    using type = T;
};
```

```cpp
typename RemoveReference<int&>::type value = 0;
```

Псевдоним избавляет пользователя от `typename` и `::type`:

```cpp
template <typename T>
using RemoveReferenceT = typename RemoveReference<T>::type;

RemoveReferenceT<int&&> value = 0;
```

### Условный выбор типа

```cpp
template <bool Condition, typename TrueType, typename FalseType>
struct Conditional {
    using type = FalseType;
};

template <typename TrueType, typename FalseType>
struct Conditional<true, TrueType, FalseType> {
    using type = TrueType;
};
```

```cpp
using Result = typename Conditional<
    (sizeof(void*) == 8),
    std::uint64_t,
    std::uint32_t
>::type;
```

Это условный оператор на уровне типов: специализация выбирает один из двух типов по значению времени компиляции.

### Стандартная библиотека

Заголовок `<type_traits>` содержит готовые инструменты:

- `std::is_same`, `std::is_pointer`, `std::is_array`;
- `std::rank`, `std::extent`;
- `std::remove_reference`, `std::remove_const`, `std::decay`;
- `std::conditional` и другие базовые преобразования типов.

Начиная с C++14 преобразующие traits обычно имеют псевдонимы с суффиксом `_t`:

```cpp
std::remove_reference_t<T>
```

Начиная с C++17 логические traits имеют переменные с суффиксом `_v`:

```cpp
std::is_same_v<T, U>
```

### `if constexpr`

```cpp
template <typename T>
void print(const T& value) {
    if constexpr (std::is_pointer_v<T>) {
        std::cout << *value << '\n';
    } else {
        std::cout << value << '\n';
    }
}
```

Условие известно во время компиляции. После инстанцирования неподходящая ветвь отбрасывается, поэтому для обычного `int` выражение `*value` не обязано быть корректным.

Отброшенная ветвь всё равно должна синтаксически разбираться, а ошибки, не зависящие от шаблонных параметров, могут диагностироваться сразу. `if constexpr` не позволяет спрятать безусловно некорректный C++-код.

До C++17 обычный `if` проверял обе ветви как часть инстанцированной функции, даже если условие было константным. Для условного выбора реализации использовали перегрузки и специализации.

## §6.8. Вариативные шаблоны

Вариативный шаблон принимает пакет из произвольного количества параметров:

```cpp
template <typename... Types>
struct TypeList {};

TypeList<> empty;
TypeList<int, double, std::string> types;
```

`Types` — пакет типов, а `Types...` — его раскрытие в подходящем контексте. Пакет может быть пустым.

Функция может принимать соответствующий пакет значений:

```cpp
template <typename... Types>
void consume(Types... values) {
    // values — пакет параметров функции
}
```

Запись `Types... values` создаёт по одному параметру для каждого типа. При вызове `consume(1, 2.5, "text")` выводится пакет типов, соответствующий аргументам.

Передача `Types... values` по значению создаёт параметры соответствующих выведенных типов. Пакет можно раскрыть сразу в списке аргументов другой функции.

### Рекурсивная обработка пакета

До fold expressions пакет часто обрабатывали рекурсией:

```cpp
void print() {}

template <typename Head, typename... Tail>
void print(const Head& head, const Tail&... tail) {
    std::cout << head;

    if constexpr (sizeof...(Tail) != 0) {
        std::cout << ' ';
    }

    print(tail...);
}
```

```cpp
print(1, 2.5, "text");
```

> Вывод: `1 2.5 text`.

Каждый вызов отделяет первый аргумент `Head`, а остаток становится новым пакетом `Tail`. Нешаблонная `print()` завершает рекурсию.

### `sizeof...`

Оператор `sizeof...` возвращает количество элементов пакета:

```cpp
template <typename... Types>
constexpr std::size_t count_types() {
    return sizeof...(Types);
}

static_assert(count_types<int, double, char>() == 3);
```

Он не суммирует размеры типов; это именно количество аргументов.

Обычным синтаксисом C++20 нельзя напрямую взять «последний тип пакета» или обратиться к нему по индексу. Для этого пакет рекурсивно разбирают, преобразуют в `std::tuple` и используют `std::tuple_element`, либо применяют другие метафункции. Пакет должен раскрываться в контексте, который создаёт корректную последовательность типов, выражений или объявлений.

## §6.9. Fold expressions

Начиная с C++17 fold expression сворачивает пакет одним бинарным оператором.

```cpp
template <typename... Types>
constexpr bool all_pointers = (std::is_pointer_v<Types> && ...);

static_assert(all_pointers<int*, double*, char*>);
static_assert(!all_pointers<int*, double, char*>);
```

Для каждого элемента пакета вычисляется `std::is_pointer_v<Types>`, а результаты объединяются через `&&`.

Существуют четыре формы:

```cpp
(pack op ...)          // унарная правая свёртка
(... op pack)          // унарная левая свёртка
(pack op ... op init)  // бинарная правая свёртка
(init op ... op pack)  // бинарная левая свёртка
```

Например, для пакета `a, b, c`:

```text
(... + pack)       -> ((a + b) + c)
(pack + ...)       -> (a + (b + c))
(0 + ... + pack)   -> (((0 + a) + b) + c)
(pack + ... + 0)   -> (a + (b + (c + 0)))
```

Ассоциативность важна для вычитания, деления, конкатенации и пользовательских операторов.

Сумма с начальным значением работает и для пустого пакета:

```cpp
template <typename... Numbers>
auto sum(Numbers... numbers) {
    return (0 + ... + numbers);
}
```

```cpp
sum();       // 0
sum(1, 2, 3); // 6
```

Печать удобно свернуть оператором запятая:

```cpp
template <typename... Types>
void print_lines(const Types&... values) {
    ((std::cout << values << '\n'), ...);
}
```

Запятая задаёт последовательное вычисление слева направо.

Бинарная форма с `init` явно задаёт результат для пустого пакета. Поэтому `sum()` в примере возвращает `0`.

## §6.10. CRTP

CRTP — Curiously Recurring Template Pattern — передаёт производный тип как аргумент шаблона базы:

```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }

    static void static_interface() {
        Derived::static_implementation();
    }
};

class Concrete : public Base<Concrete> {
public:
    void implementation() {
        std::cout << "Concrete\n";
    }

    static void static_implementation() {
        std::cout << "static Concrete\n";
    }
};
```

```cpp
Concrete object;
object.interface();
Concrete::static_interface();
```

База знает точный производный тип на этапе компиляции и приводит `this` к `Concrete*`. Виртуальная функция и runtime-диспетчеризация не нужны.

Это называют статическим полиморфизмом. Компилятор видит конечную функцию, поэтому её проще встроить. Цена — типы `Base<First>` и `Base<Second>` различны, и хранить разные производные объекты через один обычный `Base*` не получится.

В момент объявления `Base<Concrete>` тип `Concrete` ещё не завершён, но указатель или ссылка на неполный тип допустимы. Операции, требующие членов `Concrete`, проверяются при инстанцировании соответствующих методов, когда тип уже определён.

CRTP используют для mixin-классов, общей реализации операторов, счётчиков объектов, compile-time interfaces и expression templates. Он не является универсальной заменой виртуальным функциям: динамический и статический полиморфизм решают разные задачи.

## §6.11. Expression templates

Обычная перегрузка операторов над большими контейнерами может создавать временные объекты:

```cpp
Vector result = first + second + third;
```

Наивная реализация сначала создаёт временный результат `first + second`, затем складывает его с `third`. Для больших векторов это означает дополнительную память и проходы по элементам.

Expression templates вместо немедленного вычисления строят объект, описывающий выражение. Конкретный элемент вычисляется только при обращении к его индексу.

### Базовый интерфейс выражения

```cpp
template <typename Derived>
class Expression {
public:
    double operator[](std::size_t index) const {
        return static_cast<const Derived&>(*this)[index];
    }

    std::size_t size() const {
        return static_cast<const Derived&>(*this).size();
    }
};
```

Это CRTP-база. Она перенаправляет операции конкретному узлу выражения без виртуальных функций.

Конечный узел хранит ссылку на настоящий вектор:

```cpp
class VectorReference : public Expression<VectorReference> {
    const std::vector<double>& values;

public:
    explicit VectorReference(const std::vector<double>& values)
        : values(values) {}

    double operator[](std::size_t index) const {
        return values[index];
    }

    std::size_t size() const {
        return values.size();
    }
};
```

Узел суммы хранит два подвыражения:

```cpp
template <typename Left, typename Right>
class Sum : public Expression<Sum<Left, Right>> {
    Left left;
    Right right;

public:
    Sum(Left left, Right right)
        : left(std::move(left)), right(std::move(right)) {
        if (this->left.size() != this->right.size()) {
            throw std::length_error("different vector sizes");
        }
    }

    double operator[](std::size_t index) const {
        return left[index] + right[index];
    }

    std::size_t size() const {
        return left.size();
    }
};
```

Оператор `+` создаёт описание суммы, но не проходит по элементам:

```cpp
template <typename Left, typename Right>
auto operator+(
    const Expression<Left>& left,
    const Expression<Right>& right
) {
    return Sum<Left, Right>{
        static_cast<const Left&>(left),
        static_cast<const Right&>(right)
    };
}
```

```cpp
std::vector<double> first{1.0, 2.0, 3.0};
std::vector<double> second{4.0, 5.0, 6.0};

auto expression =
    VectorReference{first} + VectorReference{second};

std::cout << expression[1] << '\n';
```

> Вывод: `7`.

Только запрос индекса `1` читает `first[1]` и `second[1]` и складывает их. При присваивании выражения настоящему вектору можно одним циклом вычислить каждый итоговый элемент без промежуточного контейнера.

Объекты выражения должны существовать достаточно долго для вычисления результата: если узел хранит ссылку на уже уничтоженный вектор, ссылка станет висячей. Полноценные библиотеки отдельно решают вопросы владения и времени жизни; пример выше показывает только основную идею ленивого вычисления.
