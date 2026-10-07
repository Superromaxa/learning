# Тест по C++

## Условия

- Используется стандарт **C++20**.
- Каждый фрагмент рассматривается как отдельная программа или отдельный независимый вопрос.
- Все необходимые стандартные заголовки считаются подключёнными, если вопрос не проверяет знание заголовков.
- Предупреждения компилятора не считаются ошибками компиляции.
- Если вопрос содержит код, нужно определить одно из следующего:
  - программа корректна и имеет определённый результат;
  - результат не специфицирован однозначно;
  - присутствует implementation-defined поведение;
  - возникает ошибка компиляции (**CE**);
  - возникает неопределённое поведение (**UB**).
- Ответ необходимо объяснять. Правильное слово без объяснения причины может оцениваться только частично.
- Для вывода программы нужно указывать точный результат, если он определён стандартом.

## Оценивание

- Задания 1–45: по **2 балла**.
- Задания 46–47: по **5 баллов**.
- Максимум: **100 баллов**.

Примерная интерпретация результата:

| Баллы | Предварительный уровень |
|---:|---|
| 0–24 | Начинающий |
| 25–44 | Junior |
| 45–59 | Junior+ / Middle− |
| 60–75 | Middle |
| 76–89 | Strong Middle |
| 90–100 | Кандидат на Senior-уровень |

Высокий балл сам по себе не подтверждает Senior-уровень: дополнительно оцениваются качество объяснений, проектные решения, умение видеть ограничения задачи и писать поддерживаемый код.

---

# Часть I. Переменные, выражения и управляющие конструкции

## 1. Объявления переменных

Какие из объявлений корректны? Для некорректных объясните причину.

```cpp
int first = 10;
int second{2.5};
const int third;
unsigned int fourth = -1;
auto fifth = 3.0f;
```

Для корректных объявлений укажите тип и значение переменной.

## 2. Инициализация фундаментальных типов

Определите состояние каждой переменной сразу после объявления:

```cpp
int a;
int b{};
int c = 0;
static int d;
int* p = new int;
int* q = new int{};
```

Какие значения можно безопасно прочитать? Не забудьте указать необходимое освобождение памяти.

## 3. Целочисленные операции

Что выведет программа?

```cpp
int a = 7;
int b = 3;

std::cout << a / b << ' '
          << a % b << ' '
          << static_cast<double>(a) / b;
```

## 4. Цикл `for`

Что выведет программа?

```cpp
for (int i = 0; i < 5; ++i) {
    if (i % 2 == 0) {
        std::cout << i;
    }
}
```

Как изменить цикл, чтобы числа выводились через пробел без пробела после последнего числа?

## 5. Цикл `while`

Что выведет программа и чему будет равен `x` после цикла?

```cpp
int x = 1;

while (x < 20) {
    x *= 2;
    std::cout << x << ' ';
}
```

## 6. `break` и `continue`

Что выведет программа?

```cpp
for (int i = 0; i < 8; ++i) {
    if (i == 2) {
        continue;
    }

    if (i == 6) {
        break;
    }

    std::cout << i << ' ';
}
```

Объясните, выполняется ли `++i` после `continue`.

## 7. Сокращённое вычисление

Является ли программа корректной? Что она выведет?

```cpp
int* pointer = nullptr;

if (pointer != nullptr && *pointer == 10) {
    std::cout << "found";
} else {
    std::cout << "not found";
}
```

Что изменится, если поменять операнды `&&` местами?

## 8. Области видимости

Что выведет программа?

```cpp
int x = 1;

{
    int x = 2;
    {
        int y = x + 1;
        std::cout << x << ' ' << y << ' ';
    }
}

std::cout << x;
```

Какая переменная скрывает какую и когда заканчивается время жизни каждой локальной переменной?

## 9. Передача по значению

Что выведет программа?

```cpp
void increment(int value) {
    ++value;
}

int main() {
    int value = 5;
    increment(value);
    std::cout << value;
}
```

Измените только объявление параметра функции так, чтобы результатом было `6`.

## 10. Выбор контейнера

Для каждого случая выберите наиболее подходящий стандартный контейнер и кратко объясните выбор:

1. Последовательность элементов с быстрым доступом по индексу и добавлением в конец.
2. Множество уникальных значений, которое должно храниться отсортированным.
3. Таблица «ключ — значение» со средним временем поиска `O(1)`, порядок элементов не важен.
4. Очередь обработки задач в порядке поступления.
5. Структура, из которой всегда извлекается элемент с наибольшим приоритетом.

---

# Часть II. Контейнеры, функции, ссылки и память

## 11. `std::vector`

Что выведет программа?

```cpp
std::vector<int> values{1, 2, 3};

values.push_back(4);
values[1] = 10;
values.pop_back();

for (int value : values) {
    std::cout << value << ' ';
}
```

## 12. `operator[]` и `at`

Чем различаются следующие обращения?

```cpp
std::vector<int> values{1, 2, 3};

int first = values[10];
int second = values.at(10);
```

Опишите поведение каждой строки и возможность обработать проблему через `try-catch`.

## 13. `std::string`

Что выведет программа?

```cpp
std::string text = "abc";

text.push_back('d');
text += "ef";
text.erase(1, 2);

std::cout << text << ' ' << text.size();
```

Входит ли завершающий нулевой символ в `size()`?

## 14. `std::map::operator[]`

Что выведет программа и сколько элементов будет в `values` после выполнения?

```cpp
std::map<std::string, int> values;

values["one"] = 1;
std::cout << values["two"] << ' ';
std::cout << values.size();
```

Как проверить наличие ключа, не создавая новый элемент?

## 15. `std::set`

Что выведет программа?

```cpp
std::set<int> values{4, 1, 3, 1, 2, 4};

for (int value : values) {
    std::cout << value << ' ';
}
```

Почему повторяющиеся значения не появляются в результате?

## 16. Инвалидация итераторов

Корректен ли код?

```cpp
std::vector<int> values{1, 2, 3};
auto iterator = values.begin();

values.push_back(4);
std::cout << *iterator;
```

От чего зависит ответ? Предложите не менее двух способов написать код безопасно.

## 17. Ссылки и обмен значений

Что будет после вызова каждой функции?

```cpp
void first_swap(int x, int y) {
    int temporary = x;
    x = y;
    y = temporary;
}

void second_swap(int& x, int& y) {
    int temporary = x;
    x = y;
    y = temporary;
}

int a = 1;
int b = 2;
```

Рассмотрите отдельно `first_swap(a, b)` и `second_swap(a, b)`.

## 18. Константность указателей

Для каждой строки определите, корректна ли она:

```cpp
int x = 1;
int y = 2;

const int* first = &x;
int* const second = &x;
const int* const third = &x;

*first = 10;
first = &y;

*second = 10;
second = &y;

*third = 10;
third = &y;
```

## 19. Массив и указатель

Предположим, что `sizeof(int) == 4`, а `sizeof(int*) == 8`. Что выведет программа?

```cpp
void print_size(int values[]) {
    std::cout << sizeof(values) << ' ';
}

int main() {
    int values[5]{};

    std::cout << sizeof(values) << ' ';
    std::cout << sizeof(values) / sizeof(values[0]) << ' ';
    print_size(values);
}
```

Почему тип параметра `int values[]` не сохраняет размер исходного массива?

## 20. Управление динамической памятью

Найдите все проблемы:

```cpp
int* create_value() {
    int* pointer = new int{42};
    return pointer;
}

int main() {
    int* first = create_value();
    int* second = first;

    delete first;
    std::cout << *second << '\n';
    delete second;
}
```

Перепишите код с использованием стандартного средства владения ресурсом.

---

# Часть III. Нетривиальное поведение языка

## 21. Самоинициализация

Классифицируйте программу и подробно объясните каждый этап разбора объявления:

```cpp
int main() {
    int x = x;
    std::cout << x;
}
```

Является ли строка `int x = x;` синтаксической ошибкой? Когда имя `x` начинает быть видимым? Можно ли предсказать вывод?

## 22. Знаковое переполнение

Классифицируйте программу:

```cpp
for (int i = 0; i < 300; ++i) {
    std::cout << i << ' ' << i * 12345678 << '\n';
}
```

Начиная с какого `i` произведение перестаёт помещаться в обычный 32-битный `int`? Какие предположения имеет право делать оптимизатор?

## 23. Порядок вычисления аргументов

Для C++20 классифицируйте вызов и опишите все допустимые результаты:

```cpp
void print(int first, int second) {
    std::cout << first << ' ' << second;
}

int i = 0;
print(i++, i++);
```

Чему гарантированно равен `i` после вызова?

## 24. Неупорядоченные изменения

Классифицируйте выражение:

```cpp
int i = 1;
int result = i++ + ++i;
```

Можно ли объяснять результат только приоритетом и ассоциативностью операторов?

## 25. Время жизни временного объекта

Какие ссылки корректно использовать и как долго существуют соответствующие объекты?

```cpp
const std::string& first = std::string("abc");

const std::string& make_reference() {
    return std::string("xyz");
}

const std::string& second = make_reference();
```

## 26. Ссылка на локальную переменную

Классифицируйте код:

```cpp
int& get_value() {
    int value = 10;
    return value;
}

int main() {
    int result = get_value();
}
```

Обязан ли компилятор отказать в сборке? В какой момент возникает проблема?

## 27. Перегрузка по значению и ссылке

Какую функцию выберет компилятор?

```cpp
void f(int) {
    std::cout << "value";
}

void f(int&) {
    std::cout << "reference";
}

int value = 0;
f(value);
```

## 28. Двумерный массив

Почему следующий вызов некорректен?

```cpp
void process(int** values);

int matrix[3][5]{};
process(matrix);
```

Как должен выглядеть параметр функции, принимающей такой массив? Объясните различие между `int**` и `int (*)[5]`.

## 29. `reinterpret_cast`

Классифицируйте код:

```cpp
long long bits = 5757;
double& value = reinterpret_cast<double&>(bits);
std::cout << value;
```

Создаёт ли `reinterpret_cast` объект типа `double`? Изменится ли ответ, если `sizeof(long long) == sizeof(double)`?

---

# Часть IV. Классы, наследование и полиморфизм

## 30. Доступ к закрытому члену

Скомпилируется ли программа?

```cpp
class Counter {
    int value = 0;

public:
    void increment() {
        ++value;
    }
};

int main() {
    Counter counter;
    counter.value = 10;
}
```

Почему компилятор не разрешает обращение, даже если устройство класса очевидно? Как корректно предоставить изменение значения при необходимости?

## 31. Порядок инициализации полей

Классифицируйте программу:

```cpp
class Example {
    int first;
    int second;

public:
    explicit Example(int value)
        : second(value), first(second) {}

    void print() const {
        std::cout << first << ' ' << second;
    }
};

int main() {
    Example example{10};
    example.print();
}
```

В каком порядке на самом деле инициализируются поля?

## 32. Срезка объекта

Что произойдёт в каждом вызове?

```cpp
struct Base {
    virtual std::string name() const {
        return "Base";
    }

    virtual ~Base() = default;
};

struct Derived : Base {
    std::string name() const override {
        return "Derived";
    }
};

void by_value(Base object) {
    std::cout << object.name() << ' ';
}

void by_reference(const Base& object) {
    std::cout << object.name() << ' ';
}

Derived object;
by_value(object);
by_reference(object);
```

## 33. Виртуальная и невиртуальная функция

Что выведет программа?

```cpp
struct Base {
    void first() const {
        std::cout << "Base::first ";
    }

    virtual void second() const {
        std::cout << "Base::second";
    }
};

struct Derived : Base {
    void first() const {
        std::cout << "Derived::first ";
    }

    void second() const override {
        std::cout << "Derived::second";
    }
};

Derived object;
Base& base = object;

base.first();
base.second();
```

## 34. Несовпадение `const`

Скомпилируется ли класс `Derived`?

```cpp
struct Base {
    virtual void f() const {}
};

struct Derived : Base {
    void f() override {}
};
```

Что изменится, если удалить `override`? Будет ли `Derived::f` виртуальным переопределителем `Base::f`?

## 35. Невиртуальный деструктор

Классифицируйте программу:

```cpp
struct Base {
    ~Base() {
        std::cout << "~Base";
    }
};

struct Derived : Base {
    std::unique_ptr<int> value = std::make_unique<int>(42);

    ~Derived() {
        std::cout << "~Derived ";
    }
};

Base* pointer = new Derived;
delete pointer;
```

Достаточно ли сказать, что здесь только утечка памяти?

## 36. Закрытый переопределитель

Скомпилируется ли вызов и что он выведет?

```cpp
struct Base {
    virtual void f() const {
        std::cout << "Base";
    }
};

struct Derived : Base {
private:
    void f() const override {
        std::cout << "Derived";
    }
};

Derived object;
Base& base = object;
base.f();
```

На каком этапе проверяется доступ и для какого объявления?

## 37. Виртуальная функция и аргумент по умолчанию

Что выведет программа?

```cpp
struct Base {
    virtual void f(int value = 1) const {
        std::cout << "Base " << value << '\n';
    }
};

struct Derived : Base {
    void f(int value = 2) const override {
        std::cout << "Derived " << value << '\n';
    }
};

Derived object;
Base& base = object;

base.f();
object.f();
```

Почему тело функции и значение аргумента выбираются по разным правилам?

## 38. Виртуальный вызов из конструктора

Что выведет программа? Безопасно ли ожидать вызов `Derived::f`?

```cpp
struct Base {
    Base() {
        f();
    }

    virtual void f() const {
        std::cout << "Base";
    }
};

struct Derived : Base {
    int value = 42;

    void f() const override {
        std::cout << value;
    }
};

Derived object;
```

---

# Часть V. Шаблоны, SFINAE и лямбды

## 39. Шаблон и обычная перегрузка

Что выведет программа?

```cpp
template <typename T>
void f(T) {
    std::cout << "template";
}

void f(long long) {
    std::cout << "long long";
}

f(1);
```

Почему наличие нешаблонной функции ещё не гарантирует её выбор?

## 40. Зависимое имя

Почему в функции требуется `typename`?

```cpp
template <typename T>
struct Traits {
    using Value = int;
};

template <typename T>
void create() {
    typename Traits<T>::Value value = 0;
}
```

Как компилятор мог бы разобрать строку без `typename`?

## 41. Fold expression

Что вернёт каждый вызов?

```cpp
template <typename... Values>
auto sum(Values... values) {
    return (0 + ... + values);
}

auto first = sum();
auto second = sum(1, 2, 3, 4);
auto third = sum(1.5, 2, 3);
```

Какой тип будет у каждого результата?

## 42. SFINAE через `std::enable_if`

Какие вызовы скомпилируются и какая перегрузка будет выбрана?

```cpp
template <typename T,
          std::enable_if_t<std::is_integral_v<T>, int> = 0>
void classify(T) {
    std::cout << "integral";
}

template <typename T,
          std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
void classify(T) {
    std::cout << "floating";
}
```

Рассмотрите отдельно:

```cpp
classify(1);
classify(1.0);
classify('a');
classify("text");
```

Почему неудачная подстановка не обязана завершать компиляцию всей программы?

## 43. Detection idiom

Определите значения выражений `HasSize<T>::value`:

```cpp
template <typename T, typename = void>
struct HasSize : std::false_type {};

template <typename T>
struct HasSize<T, std::void_t<
    decltype(std::declval<const T&>().size())
>> : std::true_type {};
```

```cpp
HasSize<std::vector<int>>::value
HasSize<std::string>::value
HasSize<int>::value
HasSize<int[10]>::value
```

Объясните, в какой момент исключается специализация для неподходящего типа.

## 44. Захват по значению и `mutable`

Что выведет программа и чему будет равен внешний `value` в конце?

```cpp
int value = 10;

auto next = [value]() mutable {
    return ++value;
};

std::cout << next() << ' ';
std::cout << next() << ' ';
std::cout << value;
```

Зачем здесь требуется `mutable`?

## 45. Время жизни и преобразование лямбд

Рассмотрите две части независимо.

### Часть A

Классифицируйте код:

```cpp
auto make_counter() {
    int value = 0;

    return [&value]() mutable {
        return ++value;
    };
}

auto counter = make_counter();
std::cout << counter();
```

Как изменить захват, чтобы возвращённая лямбда была безопасной?

### Часть B

Какие присваивания корректны?

```cpp
auto first = [](int value) {
    return value * 2;
};

int offset = 3;
auto second = [offset](int value) {
    return value + offset;
};

int (*first_pointer)(int) = first;
int (*second_pointer)(int) = second;
```

Объясните различие между типами двух замыканий.

---

# Часть VI. Практические задания

## 46. Частоты слов — 5 баллов

Напишите функцию:

```cpp
std::vector<std::pair<std::string, std::size_t>>
word_frequencies(const std::vector<std::string>& words);
```

Требования:

1. Для каждого уникального слова нужно посчитать количество вхождений.
2. Результат должен быть отсортирован по убыванию частоты.
3. При одинаковой частоте слова должны идти в лексикографическом порядке.
4. Входной вектор изменять нельзя.
5. Укажите временную и дополнительную пространственную сложность решения.

Пример:

```cpp
{"red", "blue", "red", "green", "blue", "red"}
```

Ожидаемый формат результата:

```cpp
{
    {"red", 3},
    {"blue", 2},
    {"green", 1}
}
```

## 47. Полиморфные фигуры — 5 баллов

Спроектируйте классы `Shape`, `Circle` и `Rectangle`.

Требования:

1. `Shape` должен быть абстрактным классом.
2. Для каждой фигуры должны быть доступны:
   - площадь;
   - строковое имя типа фигуры;
   - полиморфное копирование через `clone()`.
3. Объекты должны храниться в:

```cpp
std::vector<std::unique_ptr<Shape>>
```

4. Напишите функцию глубокого копирования всего вектора.
5. Напишите функцию вычисления суммарной площади.
6. Не используйте ручные `new` и `delete` вне реализации стандартных умных указателей.
7. Объясните:
   - зачем нужен виртуальный деструктор;
   - почему вектор нельзя просто скопировать;
   - какую гарантию относительно ресурсов даёт решение при исключении во время копирования.

---

# Требования к оформлению ответа

Для заданий с кодом используйте следующий формат:

```text
Номер задания:
Классификация: корректно / CE / UB / unspecified / implementation-defined
Результат:
Объяснение:
```

Для практических заданий приведите компилируемый код и кратко обоснуйте выбор контейнеров, владение ресурсами и сложность.
