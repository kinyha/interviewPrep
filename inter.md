---
created: 2024-12-13 14:30
modified: 2024-12-13 14:30
type: tech-note
status:
tags:
  - 
  - 
  - 
  - 
  - 
aliases:
  - Java Interview Questions
  - Kotlin Interview Questions
  - Spring Interview Questions
sr-due: 2025-09-26
sr-interval: 3
sr-ease: 250
cssclasses:
---
#review
# 📝 Core Interview Questions - Java/Kotlin/Spring/DB

## 🎯 Цель
Структурированный сборник ответов на популярные вопросы для подготовки к техническому интервью по Java/Kotlin/Spring/DB стеку.
jk
jhkhkj


## Java Questions

### 🧠 Устройство памяти в Java
Память в JVM делится на Stack (для примитивов и ссылок) и Heap (для объектов). Heap разделён на Young Generation (Eden, Survivor spaces) и Old Generation. Метаданные классов хранятся в Metaspace.

### 🔄 Методы класса Object
Базовый класс содержит важнейшие методы: equals(), hashCode(), toString(), clone(), wait(), notify(), notifyAll(), finalize(). Каждый класс наследует эти методы.

### #️⃣ HashCode возвращает int
Использование int обеспечивает баланс между производительностью и распределением значений. 32 бита достаточно для эффективной работы хеш-таблиц.

### 🔒 Synchronized
Механизм для потокобезопасности. Synchronized метод блокирует весь объект, блок synchronized позволяет более гранулярную блокировку конкретных участков кода.

### 🔄 Dynamic vs Static Binding
Dynamic binding происходит во время выполнения (runtime) для реализации полиморфизма. Static binding происходит во время компиляции для перегруженных методов.

### 📦 ClassLoader
Bootstrap ClassLoader → Extension ClassLoader → Application ClassLoader.

### 🔄 Any vs Object
Any в Kotlin является суперклассом для всех non-nullable типов, тогда как Object в Java - для всех классов. Any имеет только три метода: equals(), hashCode(), toString().

### 🛠 JVM Annotations
- @JvmOverloads - генерирует перегруженные методы для функций с параметрами по умолчанию
- @JvmStatic - делает функции/свойства companion object доступными как статические методы Java

### 📦 Scope Functions
let, run, with, apply, also - специальные функции для выполнения блока кода в контексте объекта. Различаются контекстным объектом (this/it) и возвращаемым значением.

### 🔧 Extension Functions
Позволяют добавлять методы к существующим классам без их модификации. Компилируются в статические методы с параметром-receiver.

### 🏰 Sealed Classes vs Enum
Sealed классы позволяют создавать ограниченную иерархию классов, где каждый подкласс может иметь своё состояние. Enum ограничен фиксированным набором экземпляров.

### 🎯 Companion Object
Заменяет статические члены Java. Может реализовывать интерфейсы и содержать инициализацию, которая в Java делалась бы в статических блоках.

## Database Questions

### 🔌CallableStatement
Интерфейс JDBC для вызова хранимых процедур базы данных. В отличие от PreparedStatement, позволяет получать OUT параметры и работать с возвращаемыми значениями процедур. Синтаксис вызова: {call procedure_name(?, ?)}

### SQL JOINs
Операции объединения данных из нескольких таблиц:
- INNER JOIN - только строки с совпадениями в обеих таблицах
- LEFT JOIN - все строки из левой таблицы и совпадения из правой
- RIGHT JOIN - все строки из правой таблицы и совпадения из левой
- FULL JOIN - все строки из обеих таблиц
- CROSS JOIN - декартово произведение таблиц

### 🔌 JDBC Core
Основные классы: DriverManager, Connection, Statement, PreparedStatement, ResultSet. Используются для взаимодействия с базой данных.

### 📊 PreparedStatement vs Statement
PreparedStatement кэширует план выполнения, защищает от SQL-инъекций, позволяет использовать параметры. Statement выполняет простые запросы без параметров.

### 🔒 Уровни изоляции
- Read Uncommitted: возможно чтение "грязных" данных
- Read Committed: только зафиксированные данные
- Repeatable Read: гарантирует повторяемость чтения
- Serializable: полная изоляция транзакций

### 🗃 Hibernate Cache Levels
- First Level Cache: уровень сессии
- Second Level Cache: уровень SessionFactory
- Query Cache: кэширование результатов запросов

## Spring Questions

### 💉 Dependency Injection
@Autowired не рекомендуется из-за сложности тестирования и явной зависимости от Spring. Лучше использовать конструктор-инжекцию.

### 🔄 Bean Initialization
Порядок можно контролировать через @DependsOn, @Order, Ordered interface. По умолчанию порядок не гарантирован.

### 🎮 RestController vs Controller
@RestController включает @ResponseBody, автоматически сериализуя возвращаемые объекты в JSON/XML. @Controller требует явного указания.



