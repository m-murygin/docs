# ООП

## Table of Contents
* [Наследование](#наследование)
* [Приведение](#приведение)
* [Перегрузка](#перегрузка)
* [Виртуальные методы](#виртуальные-методы)
* [Таблица виртуальных методов.](#таблица-виртуальных-методов)
* [Особенности виртуальных методов](#особенности-виртуальных-методов)
* [Полиморфизм](#полиморфизм)
* [Принцип подстановки Барбары Лисков](#принцип-подстановки-барбары-лисков)
* [Модификаторы доступа при наследовании](#модификаторы-доступа-при-наследовании)
* [Множественное наследование](#множественное-наследование)

## Наследование
1. Наследование - это механизм, позволяющий создавать производные классы, расширяя уже существующие.
1. Внутри объекта класс-наследника хранится экземпляр родительского класса
    ```
    | Person       |
    | name_ | age_ |

    | Student             |
    | name_ | age_ | uni_ |
    ```
1. При создании производного класса сначала вызывается конструктор родительского класса.
    ```cpp
    struct Person {
        Person(string name, int age)
            : name_(name), age_(age)
        {}
    };

    struct Student : Person {
        Student (string name, int age, string uni)
            : Person(name, age), uni_(uni)
        {}
    };
    ```
1. После деструктора `Student` вызывается деструктор `Person`.

## Приведение
1. Для произвольных классов определены следующие приведения:
    ```cpp
    Student s("Alex", 21, "Oxford");
    Person & l = s; // Student & => Person &
    Person *r = &s; // Student * => Person *
    ```
1. Поэтому объекты класса-наследника могут присваиваться объектам родительского класса:
    ```cpp
    Student s("Alex", 21, "Oxford");
    Person p = s; // Person("Alex", 21);
    ```
1. При этом копируются только поля класса-родителя (срезка). Т.е. в данном случае вызывается конструктор копирования `Person (Person const &p)`, который ничего не знает про `uni_`.

## Перегрузка
1. В С++ можно определить несколько функций с одинаковым именем, но разными параметрами.
1. При вызове функции по имени будет произведен поиск наиболее подходящей функции.
1. Правила перегрузки:
    1. Если есть точное совпадение, то используется оно.
    1. Если нет функции, которая могла бы подойти с учётом преобразований, выдаётся ошибка.
    1. Есть функции подходящие с учётом преобразований:
        1. Расширение типов (из меньшего типа получаем больший)
            * `char, signed char, short => int`
            * `unsigned char, unsigned short => int/unsigned int`
            * `float => double`
        1. Стандартные преобразования
            * `double` => `int`
            * указатель на производный клас => указатель на базовый класс
            * указатель на int => указатель на void
        1. Пользовательские преобразования - самые дорогие преобразования, т.к. самые медленные
            * в классе `a` определён конструктор от типа `b`
1. В случае нескольких вариантов функции одна из них должна быть строго лучше других. Т.е. по каждому параметру она должна иметь преобразование не хуже чем у других функций и хотя бы по одному параметру преобразование лучше чем у других функций.
1. Перегрузка выполняется на этапе компиляции.
1. Не стоит злоупотреблять неочевидными перегрузками!

## Виртуальные методы
1. Переопределение методов
    ```cpp
    struct Person {
        string name () const { return name_; }
        ...
    };
    struct Professor : Person {
        string name () const {
            return "Prof. " + Person::name();
        }
        ...
    };
    ```
    * При переопределении методов вызвается метод по типу указателя/ссылки на объект:
        ```cpp
        Professor pr("Stroustrup");
        cout << pr.name() << endl; // Prof. Stroustrup
        Person *p = &pr;
        cout << p.name() << endl; // Stroustrup
        ```
1. Виртуальные методы
    ```cpp
    struct Person {
        virtual string name () const { return name_; }
        ...
    };
    struct Professor : Person {
        string name () const {
            return "Prof. " + Person::name();
        }
        ...
    };
    ```
    * При определении виртуальных методов вызывается тот метод, по типу значения (а не указателя) на которое ссылается указатель/ссылка
        ```cpp
        Professor pr("Stroustrup");
        cout << pr.name() << endl; // Prof. Stroustrup
        Person *p = &pr;
        cout << p.name() << endl; // Prof. Stroustrup
        ```
1. Определение абстрактоного метода указывается `= 0` в конце определения.
    ```cpp
    struct Person {
        virtual string occupation() const = 0;
    }
    ```
1. Если мы хотим чтобы от нашего класса можно было унаследоваться, то нужно определять виртуальный деструктор
    * без виртуального деструктора
        ```cpp
        struct Person {
            ...
        };

        struct Student : Person {
            ...
        private:
            string uni_;
        };

        int main () {
            Person * p = new Student ("Alex", 21, "Oxford");
            delete p; // здеcь вызывается деструктор типа Person, и как следствие не очищаются поля дочернего класса => получаем утечку памяти
        }
        ```

    * с виртуальным деструктором
        ```cpp
        struct Person {
            ...
            virtual ~Person() {}
        };

        struct Student : Person {
            ...
        private:
            string uni_;
        };

        int main () {
            Person * p = new Student ("Alex", 21, "Oxford");
            delete p; // вызывается деструктор типа Student
        }
        ```
## Таблица виртуальных методов.
1. Динамический полиморфизм реализуется при помощи таблиц виртуальных методов.
1. Таблица заводится для каждого _полиморфного_ класса.
1. Объекты полиморфных классов содержат указатель на таблицу виртуальных методов соответствующего класса (вначале сегмента памяти объекта)
    ```
    | Person               |
    | vptr | name _ | age_ | uni_ |
    | Student                     |
    ```
1. Вызов виртуально метода - это вызов метода по адресу из таблицы (в коде сохраняется номер метода в таблице)
    ```cpp
    p->occupation(); // p->vptr[1]();
    ```
1. Пример:
    ```cpp
    struct Person {
        virtual ~Person() {}
        virtual string occupation() = 0;
    };

    struct Teacher : Person {
        string occupation () { ... }
        virtual string course () { ... }
    };

    struct Professor : Teacher {
        string occupation () { ... }
        virtual string thesis() { ... }
    };
    ```

    Person

    | index | Method     | adress |
    | ----- | ---------- | ------:|
    | 0     | ~Person    | 0x1111 |
    | 1     | occupation | **0x0000** |

    Teacher

    | index | Method     | adress |
    | ----- | ---------- | ------:|
    | 0     | ~Teacher   | 0x2222 |
    | 1     | occupation | 0x3333 |
    | 2     | course     | **0x4444** |

    Professor

    | index | Method     | adress |
    | ----- | ---------- | ------:|
    | 0     | ~Professor | 0x5555 |
    | 1     | occupation | 0x6666 |
    | 2     | course     | **0x4444** |
    | 3     | thesis     | 0x7777 |

## Особенности виртуальных методов
1. В при вызове виртуального метода в конструкторе\деструкторе происходит вызов метода текущего класса, даже если в одном из дочерних классов этот метод переопределён.

    ```cpp
    struct Person {
        Person(string const name) : name_(name) {}
        virtual ~Person() {}
        virtual string name() const {
            return name_;
        }
    private:
        string name_;
    };

    struct Teacher : Person {
        Teacher(string const name) : Person(name) {
            cout << name();
        }
    };

    struct Professor : Person {
        string name () {
            return "Prof. " + name_;
        }
    };

    Professor p("Stroustrup"); // "Stroustrup"
    ```

    * Такое поведение позоляет предотвратить доступ к неинициализированным полям. Т.к. конструкторы вызываются от базового к дочернему (Person => Teacher => Professor), то из конструктора Teacher мы не должны иметь доступ к полям Professor

    * Это реализовано следующим образом:
        * При вызове конструктора Person указатель на таблицу виртуальных методов для этого объекта указывает на таблицу виртуальных методов класса Person
        * При вызове следующего конструктора (Teacher) указатель начинает указывать на таблицу виртуальных методов класса Teacher и т.д.

1. Можно переопределить private виртуальный метод
    ```cpp
    struct NetworkDevice {
        void send(void * data, size_t size) {
            log("start sending");
            send_impl(data, size);
            log("end");
        };
    private:
        virtual void send_impl(void * data, size_t size) {}
    };

    struct Router : NetworkDevice {
    private
        void send_impl(void * data, size_t size) {}
    }
    ```

    * Таким образом мы гарантируем что код работающий с `NetworkDevice` при посылке информации по сети всегда будет записывать информацию в логи.

1. Реализация чистых виртуальных методов. Для чистого виртуального метода можно написать реализацию, но эта реализация не попадёт в таблицу вируальных функций. Такую реализацию можно использовать как реализацию по умолчанию, которую обязательно нужно переопределить в наследниках.
    ```cpp
    struct NetworkDevice {
        virtual void send(void * data, size_t t) = 0;
    };

    void NetworkDevice::send(void * data, size_t size) { ... }

    struct Router : NetworkDevice {
        void send (void * data, size_t size) {
            // не виртуальный метод
            // вызываем как реализацию по умолчанию
            NetworkDevice::send(data, size);
        }
    };
    ```
1. _Интерфейс_ - это абстрактный класс, у которого отсутствуют поля, а все методы являются чистыми виртуальными.
    ```cpp
    struct IClonable {
        virtual ~IClonable() {}
        virtual IClonable * clone() const = 0;
    };
    ```

## Полиморфизм
1. **Полиморфизм** - возможность единообразно обрабатывать разные типы данных
1. **Перегрузка функций** - выбор функции происходит в момент компиляции на основе типов аргументов функции, _статический полиморфизм_.
1. **Виртуальные методы** - выбор метода происходит в момент выполнения на основе типа объекта, у которого вызывается виртуальный метод.

## Принцип подстановки Барбары Лисков
1. Существующие формулировки:
    * Функции работающие с базовым классом, должны иметь возможность работать с подклассами не зная об этом
    * Поведение наследумых классов не должно противоречить поведению, заданному базовым классом
    * Подкласс не должен требовать от вызывающего кода большше, чем базовый класс, и не должен предоставлять вызывающему коду меньше, чем базовый.
1. К примеру имеем классы `Shape`, `Rectanble`, `Square`. Кажется логичным унаследовать таким образом `Shape => Rectangle => Square`. Но если `Square` наследуется от `Rectangle` то мы нарушаем принцип подстановки. К примеру у `Rectangle` есть метод `setWidth`. Аналогичный метод для `square` должен будет установить и `width` и `height` => поведение класса наследника отличается от поведения базового класса => нарушается принцип подставновки.

## Модификаторы доступа при наследовании
1. При наследовании можно использовать модификаторы доступа: `public`, `protected`, `private`.
    * `public` наследование - информацию о том что класс `B` является наследником класса `A` (_ссылку\указатель на `B` можно приветсти к ссылке\указателю на `A`, по ссылке класса `B` можно вызывать методы класса `A`_) известна внутри класса `B`, внутри всех наследников класса `B` и во внешнем коде.
        ```cpp
        class B : public A {};
        ```
    * `protected` - информацию о том что класс `B` является наследником класса `A`  известна внутри класса `B` и внутри всех наследников класса `B`
    * `private` - информацию о том что класс `B` является наследником класса `A`  известна внутри класса `B`.
1. Структуры по умолчанию наследуются с модификатором доступа `public`, классы с модификатором доступа `private`.
1. **Важно**: отношение наследования в терминах ООП задаётся только public наследованием.
1. В большинстве случаем `private` и `protected` наследование можно заменить агрегированием.
1. Использование `private` и `protected` наследований целесообразно, если необходимо не только агрегировать другой класс, но и переопределить его виртуальные методы.

## Можественное наследование.
1. В `C++` разрешено множественное наследование
    ```cpp
    struct Person {};
    struct Student : Person {};
    struct Worker : Person {};
    struct WorkingStudent : Student, Worker {};
    ```
    * В приведённом выше примере в объекте класса `WorkingStudent` будет содержаться две копии объекта `Person`. Причём эти данные не будут синхронизироваться. Более того при попытке вызывать функцию класса `Person` получим неоднозначное поведение, т.к. данная фунция определена дважды.
1. Стоит избегать наследования реализации более чем от одного класса, вместо этого использовать интерфейсы.
    ```cpp
    struct IWorker {};
    struct Worker : Person, IWorker {};
    struct Student : Person {};
    struct WorkingStudent : Student, IWorker {};
    ```
