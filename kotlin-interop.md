---
type: doc
layout: reference
category: "Совместимость с Java"
title: "Вызов Kotlin из Java"
---

# Вызов Kotlin из Java

Код на языке Kotlin можно легко вызывать из Java.

## Свойства

Свойство в Kotlin компилируется в следующие элементы в Java:

 * В геттер с именем, вычисляемым путём добавления префикса `get`;
 * В сеттер с именем, вычисляемым путём добавления префикса `set` (только для изменяемых (`var`) свойств);
 * В приватное поле с тем же именем, что и у свойства (только для свойств с `backing fields).

Например, `var firstName: String` будет скомпилирован в следующие Java инструкции:

``` java
private String firstName;

public String getFirstName() {
    return firstName;
}

public void setFirstName(String firstName) {
    this.firstName = firstName;
}
```

Если свойство начинается с префикса `is`, будет использована другая система именования: имя геттера будет
таким же, как и имя свойства, а имя сеттера будет получено путём замены `is` на `set`.
Например, для свойства `isOpen`, геттер будет объявлен как `isOpen()` а сеттер будет объявлен как `setOpen()`.
Это правило применяется для свойств любого типа, не только для `Boolean`.

## Функции с областью видимости на уровне пакета

Все функции и свойства, объявленные в файле `example.kt` внутри пакета `org.foo.bar`, включая функции-расширения,
компилируются в статические методы в Java с именем `org.foo.bar.ExampleKt`.

``` kotlin
// example.kt
package demo

class Foo

fun bar() {
}

```

``` java
// Java
new demo.Foo();
demo.ExampleKt.bar();
```

Имя сгенерированного класса в Java может быть изменено с помощью аннотации `@JvmName`:

``` kotlin
@file:JvmName("DemoUtils")

package demo

class Foo

fun bar() {
}

```

``` java
// Java
new demo.Foo();
demo.DemoUtils.bar();
```

Наличие нескольких файлов, которые будут сгенерированы в Java класс с одинаковым названием (одинаковый пакет и имя класса,
или одинаковое значение аннотации @JvmName обычно является ошибкой ). Хотя компилятор может сгенерировать единственный Java
класс-фасад, который будет иметь указанное имя и содержать все объявления which has the specified name из файлов с таким именем.
Для включения генерации такого фасада добавтье аннотацию @JvmMultifileClass во всех файлах.

``` kotlin
// oldutils.kt
@file:JvmName("Utils")
@file:JvmMultifileClass

package demo

fun foo() {
}
```

``` kotlin
// newutils.kt
@file:JvmName("Utils")
@file:JvmMultifileClass

package demo

fun bar() {
}
```

``` java
// Java
demo.Utils.foo();
demo.Utils.bar();
```

## Поля экземпляра

Если вам нужно указаит свойство Kotlin как поле в Java, вам необходимо аннотировать его аннотацией `@JvmField`.
Поле будет иметь такую-же область видимости, что и базовое свойство. Вы можете аннотировать свойство `@JvmField`
если оно имеет backing field, не приватное, не имеет модификаторов `open`, `override`, или `const` и это не делегированное свойство.

``` kotlin
class C(id: String) {
    @JvmField val ID = id
}
```

``` java
// Java
class JavaClient {
    public String getID(C c) {
        return c.ID;
    }
}
```

[Late-Initialized](properties.html#late-initialized-properties-and-variables) properties are also exposed as fields. 
The visibility of the field will be the same as the visibility of `lateinit` property setter.

## Статические поля

Свойства Kotlin, объявленные в именованном объекте или объекте-компаньоне, будут иметь статические backing fields
или в этом именованном объекте или в классе, содержащем объект-компаньон.

Обычно эти поля приватные, но они могут быть объявлены следующими способами:

 - аннотация `@JvmField`;
 - модификатор `lateinit`;
 - модификатор `const`.
 
Аннотирование такого свойства с `@JvmField` делает его статическим полем с такой же областью видимости как и само свойство.

``` kotlin
class Key(val value: Int) {
    companion object {
        @JvmField
        val COMPARATOR: Comparator<Key> = compareBy<Key> { it.value }
    }
}
```

``` java
// Java
Key.COMPARATOR.compare(key1, key2);
// public static final field in Key class
```

[Свойства с отложенной инициализацией](properties.html#late-initialized-properties-and-variables) в объекте, или в объекте-компаньоне
имеют статический backing field с той же областью видимости, что и сеттер свойства.

``` kotlin
object Singleton {
    lateinit var provider: Provider
}
```

``` java
// Java
Singleton.provider = new Provider();
// public static non-final field in Singleton class
```

Свойства аннотированные с `const` (как в классах, так и на верхнем уровне) преобразуются в следующие статические поля в ava:

``` kotlin
// file example.kt

object Obj {
    const val CONST = 1
}

class C {
    companion object {
        const val VERSION = 9
    }
}

const val MAX = 239
```

В Java:

``` java
int c = Obj.CONST;
int d = ExampleKt.MAX;
int v = C.VERSION;
```

## Статические методы

Как упоминалось выше, Kotlin представляет функции уровня пакета как статические методы.
Kotlin также может генерировать статические методы для функций, определенных в именованных объектах или объектах-компанбонах, если вы аннотируете эти функции как `@ JvmStatic`.
Если вы используете эту аннотацию, компилятор будет генерировать как статический метод в охватывающем классе объекта, так и метод экземпляра в самом объекте.
Например:

``` kotlin
class C {
    companion object {
        @JvmStatic fun foo() {}
        fun bar() {}
    }
}
```

Теперь, `foo()` это статический метод в Java, хотя `bar()` не статический:

``` java
C.foo(); // работает
C.bar(); // ошибка: не статический метод
C.Companion.foo(); // остался метод экземпляра
C.Companion.bar(); // работает только так
```

То же самое для именованных объектов:

``` kotlin
object Obj {
    @JvmStatic fun foo() {}
    fun bar() {}
}
```

В Java:

``` java
Obj.foo(); // works fine
Obj.bar(); // error
Obj.INSTANCE.bar(); // works, a call through the singleton instance
Obj.INSTANCE.foo(); // works too
```

Аннотация `@JvmStatic` также может быть применена к свойствам объекта или объекта-компаньона,
делая их геттеры и сеттеры статическими членами в этом объекте, или классе, содержащем объект-компаньон.

## Область видимости

Область видимости в Kotlin преобразуется в Java следующим способом:

* `private` члены компилируются в `private` члены;
* `private` объявления верхнего уровня компилируются в объявления с областью видимости на уровне пакета;
* `protected` остаётся `protected` (учтите, что разрешает доступ к защищённым членам из других классов в том же пакете,
а Kotlin - нет, то есть Java классы имеют более широкий доступ к коду);
* `internal` становятся `public` в Java. Члены `internal` классов переименовываются, чтобы,
уменьшить вероятность их случайного использования сJava и разрешить перегрузку для членов с той же сигнатурой, которые не видят
друг друга в Kotlin;
* `public` остаётся `public`.

## KClass

Иногда вам нужно вызвать метод Kotlin с параметром типа `KClass`.
Автоматическое преобразование из класса «Class» в «KClass» отсутствует, поэтому вам нужно сделать это вручную, вызывая эквивалентное
свойство расширения класса <T> .kotlin`:

```kotlin
kotlin.jvm.JvmClassMappingKt.getKotlinClass(MainView.class)
```

## Обработка конфликтов сигнатур с @JvmName

Иногда у нас есть именованная функция в Kotlin, для которой нам нужно другое имя в JVM байткоде.
Самый яркий пример такого случая это *стирание типа *:

``` kotlin
fun List<String>.filterValid(): List<String>
fun List<Int>.filterValid(): List<Int>
```

Эти две функции не могут быть определены в одном месте, потому что их подписи JVM одинаковы: `filterValid (Ljava / util / List;) Ljava / util / List;`.
Если мы действительно хотим, чтобы они имели одинаковое имя в Kotlin, мы можем аннотировать один (или оба) из них с помощью `@ JvmName` и указать другое имя в качестве аргумента:


``` kotlin
fun List<String>.filterValid(): List<String>

@JvmName("filterValidInt")
fun List<Int>.filterValid(): List<Int>
```

Из Kotlin они будут доступны под одним и тем же именем `filterValid`, но из Java это будут доступны как `filterValid` и `filterValidInt`.

Тот же трюк применяется, когда нам нужно иметь свойство `x` вместе с функцией `getX()`:


``` kotlin
val x: Int
    @JvmName("getX_prop")
    get() = 15

fun getX() = 10
```


## Генерация перегруженных методов

Обычно, если вы пишете функцию Kotlin со значениями параметров по умолчанию, она будет видна на Java только как функция со всеми параметрами. Если вы хотите видеть все варианты перегруженной функции в вызывающем коде Java, вы можете использовать
аннотацию `@JvmOverloads`.

Аннотации также работают для конструкторов, статических методов и т.д. Их нельзя использовать для абстрактных методов, включая методы
объявленные в интерфейсах.

``` kotlin
class Foo @JvmOverloads constructor(x: Int, y: Double = 0.0) {
    @JvmOverloads fun f(a: String, b: Int = 0, c: String = "abc") {
        ...
    }
}
```

Для каждого параметра со значением по умолчанию это создаст одну дополнительную перегрузку, которая включает этот параметр и
все не имеет остальных. В этом примере будет сгенерирован следующий код:


``` java
// Constructors:
Foo(int x, double y)
Foo(int x)

// Methods
void f(String a, int b, String c) { }
void f(String a, int b) { }
void f(String a) { }
```

Обратите внимание, что, как описано в [Вторичные конструкторы] (classes.html # secondary-constructors), если класс имеет значение по умолчанию для всех параметров конструктора, для него будет создан публичный конструктор без аргументов. Это работает даже
если аннотация `@ JvmOverloads` не указана.


## Проверяемые исключения

Как мы уже упоминали выше, в Kotlin нет проверяемых исключений.
То есть нормально, когда Java-сигнатура Kotlin-функции не объявляет исключений, которые выбрасывает.
Таким образом, если функция в Kotlin выглядит следующим образом:

``` kotlin
// example.kt
package demo

fun foo() {
    throw IOException()
}
```

И мы хотим вызвать её из Java и поймать исключение:

``` java
// Java
try {
  demo.Example.foo();
}
catch (IOException e) { // error: foo() does not declare IOException in the throws list
  // ...
}
```

мы получаем сообщение об ошибке из компилятора Java, потому что `foo ()` не объявляет `IOException`.
Чтобы обойти эту проблему, используйте аннотацию `@ Throws` в Kotlin:

``` kotlin
@Throws(IOException::class)
fun foo() {
    throw IOException()
}
```

## Null-безопасность

При вызове функций Kotlin из Java никто не мешает нам передавать *null* {: .keyword} в качестве ненулевого параметра.
Вот почему Kotlin генерирует проверки времени выполнения для всех публичных функций, которые ожидают не-null значений.
Таким образом мы сразу получаем «NullPointerException» в Java-коде.

## Обобщения вариантов?

Когда классы Kotlin используют [declaration-site variance] (generics.html # declaration-site-variance), есть два
варианта того, как их будет видно из Java кода. Допустим, у нас есть следующий класс и две функции, которые его используют:

``` kotlin
class Box<out T>(val value: T)

interface Base
class Derived : Base

fun boxDerived(value: Derived): Box<Derived> = Box(value)
fun unboxBase(box: Box<Base>): Base = box.value
```

Наивный способ трансляции этих функций в Java был бы следующим:
 
``` java
Box<Derived> boxDerived(Derived value) { ... }
Base unboxBase(Box<Base> box) { ... }
``` 

Проблема в том, что в Kotlin мы можем сказать `unboxBase (boxDerived ("s"))`, но в Java это было бы невозможно, потому что в Java класс `Box` *инвариантен* в своем параметре `T`, и поэтому `Box <Derived>` не является подтипом `Box <Base>`. Чтобы он работал на Java, нам нужно было бы определить `unboxBase` следующим образом:
  
``` java
Base unboxBase(Box<? extends Base> box) { ... }  
```  

Здесь мы используем Java *generic wildcards* (`? Extends Base`) для эмуляции declaration-site variance через use-site, потому что в Java это есть.

Чтобы API-интерфейсы Kotlin работали в Java, мы генерируем `Box <Super>` как `Box <? extends Super> `для ковариантно определенных` Box`
(или `Foo <? super Bar>` для контравариантно определенных `Foo`), когда он появляется *как параметр*. Когда это возвращаемое значение,
мы не создаем подстановочные знаки, так как в противном случае клиенты Java будут иметь дело с ними (и это противоречит общему
стилю кодирования Java). Поэтому функции из нашего примера фактически переводятся следующим образом:
  
``` java
// return type - no wildcards
Box<Derived> boxDerived(Derived value) { ... }
 
// parameter - wildcards 
Base unboxBase(Box<? extends Base> box) { ... }
```

ПРИМЕЧАНИЕ: когда тип аргумента является окончательным, обычно нет смысла создавать подстановочный знак, поэтому `Box <String>` всегда
   `Box <String>`, независимо от того, какую позицию он занимает.

Если нам нужны подстановочные знаки, в которых они не генерируются по умолчанию, мы можем использовать аннотацию `@ JvmWildcard ':

``` kotlin
fun boxDerived(value: Derived): Box<@JvmWildcard Derived> = Box(value)
// is translated to 
// Box<? extends Derived> boxDerived(Derived value) { ... }
```

С другой стороны, если нам не нужны подстановочные знаки, где они сгенерированы, мы можем использовать `@ JvmSuppressWildcards`:

``` kotlin
fun unboxBase(box: Box<@JvmSuppressWildcards Base>): Base = box.value
// is translated to 
// Base unboxBase(Box<Base> box) { ... }
```

ПРИМЕЧАНИЕ. `@ JvmSuppressWildcards` может использоваться не только для аргументов отдельного типа, но и для всех объявлений, например
функций или классов, заставляя все подстановочные знаки внутри них подавляться.

### Перевод типа Nothing
 
Тип [`Nothing`] (exceptions.html # the-nothing-type) является особым, потому что он не имеет естественного аналога в Java. Действительно, каждый ссылочный тип Java, включая
`java.lang.Void`, принимает значение« null »как значение, а` Nothing` даже не принимает этого. Таким образом, этот тип не может быть точно
представленных в мире Java. Вот почему Kotlin генерирует необработанный тип, где используется аргумент типа `Nothing`:

``` kotlin
fun emptyList(): List<Nothing> = listOf()
// is translated to
// List emptyList() { ... }
```
