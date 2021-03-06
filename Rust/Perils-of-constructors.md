Опасности конструкторов
Aleksey Kladov
https://matklad.github.io/2019/07/16/perils-of-constructors.html

Один из моих любимых постов из блогов о Rust -  [Things Rust Shipped Without авторства Graydon Hoare](https://graydon2.dreamwidth.org/218040.html). Для меня отсутствие в языке любой фичи, способной выстрелить в ногу, обычно важнее выразительности. В этом слегка философском эссе я хочу поговорить о моей особенно любимой фиче, отсутствующей в Rust - о конструкторах.

# Что такое конструктор?

Конструкторы обычно используются в ОО языках. Задача конструктора - полностью инициализировать объект, прежде чем остальной мир увидит его. На первый взгляд, это кажется действительно хорошей идеей:

1. Вы **устанавливаете инварианты** в конструкторе.
2. Каждый метод заботится о **сохранении** инвариантов.
3. Вместе эти два свойства значат, что можно думать об объектах как об инвариантах, а не как о конкретных внутренних состояниях.

Конструктор здесь играет роль индукционной базы, будучи единственным способом создать новый объект.

К сожалению, в этих рассуждениях есть дыра: сам конструктор наблюдает объект в незаконченном состоянии, что и создает множество проблем.

# Значение this

Когда конструктор инициализирует объект, он начинает с некоторого пустого состояния. Но как вы определите это пустое состояние для произвольного объекта?

Наиболее легкий способ сделать это - присвоить всем полям значения по умолчанию: false для bool, 0 для чисел, null для всех ссылок. Но такой подход требует, чтобы все типы имели значения по умолчанию, и вводит в язык печально известный null. Именно по этому пути пошла Java: в начале создания объекта все поля имеют значения 0 или null.

При таком подходе будет очень сложно избавиться от null впоследствии. Хороший пример для изучения - Kotlin. Kotlin использует non-nullable типы по умолчанию, но он вынужден работать с прежде существующей семантикой JVM. Дизайн языка хорошо скрывает этот факт и хорошо применим на практике, но **несостоятелен**. Иными словами, используя конструкторы, есть возможность обойти проверки на null в Kotlin.

Главная характерная черта Kotlin - поощрение создания так называемых "первичных конструкторов", которые **одновременно** объявляют поле и присваивают ему значение прежде, чем будет выполняться какой-либо пользовательский код:

```kotlin
class Person(
  val firstName: String,
  val lastName: String
) { ... }
```

Другой вариант: если поле не объявлено в конструкторе, программист должен немедленно инициализировать его:

```kotlin
class Person(val firstName: String, val lastName: String) {
    val fullName: String = "$firstName $lastName"
}
```

Попытка использовать поле перед инициализацией запрещена статически:

```kotlin
class Person(val firstName: String, val lastName: String) {
    val fullName: String
    init {
        println(fullName) // ошибка: переменная должна быть инициализирована
        fullName = "$firstName $lastName"
    }
}
```

Но, имея немного креативности, любой может обойти эти проверки. Например, для этого подойдет вызов метода:

```kotlin
class A {
    val x: Any
    init {
        observeNull()
        x = 92
    }
    fun observeNull() = println(x) // выводит null
}

fun main() {
    A()
}
```

Также подойдет захват this лямбдой (которая создается в Kotlin следующим образом: { args -> body }):

```kotlin
class B {
    val x: Any = { y }()
    val y: Any = x
}

fun main() {
    println(B().x) // выводит null
}
```

Примеры вроде этих кажутся нереальными в действительности (и так и есть), но я находил подобные ошибки в реальном коде (правило вероятности 0-1 Колмогорова в разработке ПО: в достаточно большой базе любой кусок кода почти гарантированно существует, по крайней мере, если не запрещен статически компилятором; в таком случае он почти точно не существует).

Причина, по которой Kotlin может существовать с этой несостоятельностью, та же, что и в случае с ковариантными массивами в Java: в рантайме все равно происходят проверки. В конце концов, я бы не хотел усложнять систему типов Kotlin, чтобы сделать вышеприведенные случаи некорректными на этапе компиляции: учитывая существующие ограничения (семантику JVM), отношение цена/польза проверок в рантайме намного лучше таковой у статических проверок.

А что, если язык не имеет разумного значения по умолчанию для каждого типа? Например, в C++, где определенные пользователем типы не обязательно являются ссылками, вы не можете просто присвоить null каждому полю и сказать, что это будет работать! Вместо этого в C++ используется специальный синтаксис для установления начальных значений полям: списки инициализации:

```cpp
#include <string>
#include <utility>

class person {
  person(std::string first_name, std::string last_name)
    : first_name(std::move(first_name))
    , last_name(std::move(last_name))
  {}

  std::string first_name;
  std::string last_name;
};
```

Так как это специальный синтаксис, остальная часть языка работает с ним небезупречно. Например, сложно поместить в списки инициализации произвольные операции, так как C++ не является фразированным языком (expression-oriented language) (что само по себе нормально). Чтобы работать с исключениями, возникающими с списках инициализации, необходимо использовать еще одну [невразумительную фичу языка](https://en.cppreference.com/w/cpp/language/function-try-block).

# Вызов методов из конструктора

Как намекают примеры из Kotlin, все разлетается в щепки, как только мы пытаемся вызвать метод из конструктора. В основном, методы ожидают, что объект, доступный через this, уже полностью сконструирован и корректен (соответствует инвариантам). Но в Kotlin или Java ничто не мешает вам вызывать методы из конструктора, и таким образом мы можем случайно оперировать полусконструированным объектом. Конструктор обещает установить инварианты, но в то же время это самое простое место их возможного нарушения.

Особенно странные вещи происходят, когда конструктор базового класса вызывает метод, переопределенный в производном классе:

```kotlin
abstract class Base {
    init {
        initialize()
    }
    abstract fun initialize()
}

class Derived: Base() {
    val x: Any = 92
    override fun initialize() = println(x) // выводит null!
}
```


Просто подумайте об этом: код произвольного класса выполняется **до** вызова его конструктора! Подобный код на C++ приведет к еще более любопытным результатам. Вместо вызова функции производного класса будет вызвана функция базового класса. Это имеет немного смысла, потому что производный класс еще не был инициализирован (помните, мы не можем просто сказать, что все поля имеют значение null). Однако если функция в базовом классе будет чистой виртуальной, ее вызов приведет к UB.

# Сигнатура конструктора

Нарушение инвариантов - не единственная проблема конструкторов. Они имеют сигнатуру с фиксированным именем (пустым) и типом возвращаемого значения (сам класс). Это делает перегрузки конструкторов сложными для понимания людьми.

> Вопрос на засыпку: чему соответствует std::vector&lt;int&gt; xs(92, 2)?
>
> a. Вектору двоек длины 92
>
> b. [92, 92]
>
> c. [92, 2]

Проблемы с возвращаемым значением возникают, как правило, тогда, когда оказывается невозможно создать объект. Вы не можете просто вернуть Result&lt;MyClass, io::Error&gt; или null из конструктора!

Это часто используется в качестве аргумента в пользу того, что использовать C++ без исключений сложно, и что использование конструкторов вынуждает также использовать исключения. Однако, я не думаю, что этот аргумент корректен: фабричные методы решают обе эти проблемы, потому что они могут иметь произвольные имена и возвращать произвольные типы. Я считаю, что следующий паттерн иногда может быть полезен в ОО языках:

- Создайте один **приватный** конструктор, который принимает значения всех полей в качестве аргументов и просто присваивает их. Таким образом, такой конструктор работал бы как литерал записи (record literal) в Rust. Он также может проверять любые инварианты, но он не должен делать что-то еще с аргументами или полями.

- для публичного API предоставляются публичные фабричные методы с подходящими названиями и типами возвращаемых значений.

Похожая проблема с конструкторами заключается в том, что они специфичны, и поэтому нельзя их обобщать. В C++ "есть конструктор по умолчанию" или "есть копирующий конструктор" нельзя выразить проще, чем "определенный *синтаксис* работает". Сравните это с Rust, где эти концепции имеют подходящие сигнатуры:

```rust
trait Default {
    fn default() -> Self;
}

trait Clone {
    fn clone(&self) -> Self;
}
```

# Жизнь без конструкторов

В Rust есть только один способ создать структуру: предоставить значения для всех полей. Фабричные функции, такие как общепринятый new, играют роль конструкторов, но, что самое важное, они не позволяют вызывать какие-либо методы до тех пор, пока у вас на руках нет хотя бы более-менее корректного экземпляра структуры.

Недостаток этого подхода заключается в том, что любой код может создать структуру, так что нет единого места, такого как конструктор, для поддержания инвариантов. На практике это легко решается приватностью: если поля структуры приватные, то эта структура может быть создана только в том же модуле. Внутри *одного* модуля совсем нетрудно придерживаться соглашения "все способы создания структуры должны использовать метод new". Вы даже можете представить расширение языка, которое позволит помечать некоторые функции атрибутом #[constructor], чтобы синтаксис литерала записи был доступен только в помеченных функциях. Но, опять же, дополнительные языковые механизмы мне кажутся излишними: следование **локальным** соглашениям требует мало усилий.

> Лично я считаю, что этот компромисс выглядит точно также и для контрактного программирования в целом. Контракты вроде "не null" или "положительное значение" лучше всего кодируются в типах. Для сложных инвариантов просто писать assert!(self.validate()) в каждом методе не так уж и сложно. Между этими двумя паттернами есть немного места для #[pre] и #[post] условий, реализованных на уровне языка или основанных на макросах.

# А что насчет Swift?

Swift - еще один интересный язык, на механизмы конструирования в котором стоит посмотреть. Как и Kotlin, Swift - null-безопасный язык. В отличие от Kotlin, проверки на null в Swift более сильные, так что в языке используются интересные уловки для смягчения урона, вызванного конструкторами.

*Во-первых*, в Swift используются именованные аргументы, и это немного помогает с "все конструкторы имеют одинаковое имя". В частности, два конструктора с одинаковыми типами параметров - не проблема:

```swift
Celsius(fromFahrenheit: 212.0)
Celsius(fromKelvin: 273.15)
```

*Во-вторых*, для решения проблемы "конструктор вызывает виртуальный метод класса объекта, который еще не был полностью создан" Swift использует продуманный протокол двухфазной инициализации. Хотя и нет специального синтаксиса для списков инициализации, компилятор статически проверяет, чтобы тело конструктора имело правильную и безопасную форму. Например, вызов методов возможно только после того, как все поля класса и его потомков проинициализированы.

*В-третьих*, на уровне языка есть поддержка конструкторов, вызов которых может завершиться неудачей. Конструктор может быть обозначен как nullable, что делает результат вызова класса вариантом. Конструктор также может иметь модификатор throws, который лучше работает с семантикой двухфазной инициализации в Swift, чем с синтаксисом списков инициализации в C++.

Swift удается закрыть в конструкторах все дыры, на которые я пожаловался. Это, однако, имеет свою цену: [глава, посвященная инициализации](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html) одна из самых больших в книге по Swift.

# Когда конструкторы действительно необходимы

Вопреки всему я могу придумать как минимум две причины, по которым конструкторы не могут быть замещены литералами записей, такими как в Rust.

*Во-первых*, наследование в той или иной степени вынуждает язык иметь конструкторы. Вы можете представить расширение синтаксиса записей с поддержкой базовых классов:

```rust
struct Base { ... }

struct Derived: Base { foo: i32 }

impl Derived {
    fn new() -> Derived {
        Derived {
            Base::new()..,
            foo: 92,
        }
    }
}
```

Но это не будет работать в типичном макете объектов (object layout) ОО языка с простым наследованием! Обычно объект начинается с заголовка, за которым следуют поля классов, от базового до самого производного. Таким образом, префикс объекта производного класса является корректным объектом базового класса. Однако, чтобы такой макет работал, конструктору необходимо выделять память под весь объект за один раз. Он не может просто выделить память только под базовый класс, а затем присоединить производные поля. Но такое выделение памяти по кускам необходимо, если мы хотим использовать синтаксис записи, где мы могли бы указывать значение для базового класса.

*Во-вторых*, в отличие от записей, конструкторы имеют ABI, хорошо работающий с размещением подобъектов объекта в памяти (placement-friendly ABI). Конструктор работает с указателем на this, который указывает на область памяти, которую должен занимать новый объект. Что самое важное, конструктор может с легкостью передавать указатель в конструкторы подобъектов, позволяя тем самым создавать сложные деревья значений "на месте". В противовес этому, в Rust конструирование записей семантически включает довольно много копий, и здесь мы надеемся на милость оптимизатора. Это не совпадение, что в Rust еще нет принятого рабочего предложения относительно размещения подобъектов в памяти!
