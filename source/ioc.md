# Spring IoC

## Контейнер бинов и способы его конфигурации

### Вступление

Что такое бин (`bean`)? Бин - это класс, жизненным циклом которого управляет контейнер спринга.

Основная задача контейнера бинов заключается в хранении бинов и управлении их жизненным циклом.

Способ, с помощью которого контейнер управляет жизненным циклом, задаётся пользователем через _конфигурацию_ бинов 
либо самим контейнером способом по умолчанию. 

Единственное необходимое условие включения класса в качестве 
бина под управлением контейнера: пользователь явно должен указать Spring'у, какие классы будут им 
управляться.

Делегирование управления жизненным циклом класса позволяет избавиться от большой части рутинного кода,
полноценно следовать принципу инвертирования зависимостей и сосредоточить внимание на бизнес-коде, а не на 
его инфраструктурной части.

Все бины приложения, как пользовательские, так и служебные, называются контекстом приложения и хранятся 
в `ApplicationContext`.

### Конфигурация контейнера бинов

Однако, чтобы эффективно управлять жизненным циклом, спрингу необходимы подсказки от пользователя. Эти подсказки
описываются в _конфигурации бинов_ в виде декларативного описания: как создавать объект, в какие поля
необходимо внедрить зависимости, какие методы вызывать после создания и перед уничтожением объекта и т.д.

Конфигурации могут иметь зависимости друг на друга посредством импортирования. С помощью этого механизма можно
иметь конфигурации всех типов, импортировав их в одну рутовую конфигурацию, через которую будет 
инициализироваться Spring контекст.

### Виды конфигурации бинов 
Есть несколько способов это сделать, давайте их перечислим:

* XML конфигурация. Описание бинов лежит в xml файле с использованием специального синтаксиса. Считается
устаревшим, используется только стариками, которые не хотят идти в ногу со временем. 
    * Плюсы: 
        * можно перегружать в рантайме
    * Минусы: 
        * можно перегружать в рантайме
        * большой объем, многословный синтаксис
        * нет типизации, но спасает IDE
* Java-class конфигурация. Описание бинов находится в java классе с использованием различных 
аннотаций.
    * Минусы: 
        * теперь вместо кучи xml пишем кучу кода
        * нельзя перегружать в рантайме 
    * Плюсы:
         * типизация, сложно ошибиться
         * код писать привычней и приятней, чем xml
* Groovy скрипт конфигурация. Описание бинов лежит в groovy скрипте с использованием DSL. Появился 
со Spring 4.
    * Минусы:
        * им никто не пользуется
        * требует груви зависимостей
    * Плюсы:
        * есть типизация
        * можно перегружать в рантайме (ха-ха, прощай xml)
        * немногословный синтаксис (привет, java-конфигурация)         
* Kotlin конфигурация. Появилась в Spring 5. Бины описываются с помощью DSL, написанном на котлине. Тоже скрипт. По сути 
всё то же самое, что и груви, только типизация не динамическая, а статическая, что не может не радовать.
Must have при написании проектов Spring 5 + Kotlin.

### Примеры

Xml конфигурация:

```xml
<?xml version = "1.0" encoding = "UTF-8"?>

<beans xmlns = "http://www.springframework.org/schema/beans"
   xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation = "http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

   <bean id = "converterBean" class = "NumberToCharacterConverter" destroy-method="close">
   </bean>
   
   <bean id="someService" class="SomeServiceImpl" init-method="init" scope="prototype">
     <constructor-arg>converter</constructor-arg>
   </bean>

</beans>
```

Java-class конфигурация:     
```java
@Configuration
public class Beans {
    
    @Bean(destroyMethod = "close")
    public Converter<Number, Character> converterBean() {
        return new NumberToCharacterConverter();
    }
    
    @Bean(initMethod = "init")
    @Scope(BeanDefinition.SCOPE_PROTOTYPE)
    public SomeService someService(Converter<Number, Character> converter) {
        return new SomeServiceImpl(converter);
    }
}
```        

Groovy конфигурация:

```groovy
beans {
    converterBean(NumberToCharacterConverter) {bean ->
        bean.destroyMethod = "close" 
    }
    
    someService(SomeServiceImpl, converter : converterBean) {bean ->
        bean.initMethod = "init" 
        bean.scope = "prototype"     
    }
    
}
```

Kotlin конфигурация:

```kotlin
fun beans() = beans {

    bean<NumberToCharacterConverter>("converterBean")
    
    bean<SomeService>("someService") {
        SomeServiceImpl(ref())
    }
}
```
(Да, я не знаю, где в котлине destroy, init методы, так как даже примеры оказалось найти не так просто.)

Отметим ещё способ объявления бина через аннотации. Технически это сложно назвать способом конфигурации,
так как он довольно сильно отличается от других способов. Идея этого способа состоит в том, чтобы в 
самом классе расставить аннотации и указать на него спрингу. Спринг считывает определение бина из этих
аннотаций, и класс попадает по управление контейнером.

```java
@Component
public class NumberToCharacterConverter implements Converter<Number, Character> {
 
    @Override
    public Character convert(Number source) {
    	return (char) source.shortValue();
    }
    
    @PreDestroy
    private void destroy() {
        // Это приватный метод-деструктор. Да, destroy методы и init могут быть приватными.
        // Он нужен, чтобы закрывать различные ресурсы.
    }
}

@Service // Это псевдоним для @Component. Он нужен, чтобы подчеркнуть, что в классе находится бизнес-логика.
public class SomeServiceImpl implements SomeService {
    
    private final Converter<Number, Character> converter;
    
    public SomeServiceImpl(Converter<Number, Character> converter) {
        this.converter = converter;
    }
    
    @PostConstruct
    public void init() {
        // Это так называемый второй конструктор. 
        // Он нужен, чтобы выполнить какие-то инициализирующие действия уже после
        // инжекта всех зависимостей.
    }
    
    public void businessLogic() {
        converter.convert(1);
    }
}
```

### Резюме

Как видно из примеров, мы можем указать спрингу имя бина (какой именно класс мы передаем под его 
управление), описать его зависимости, _init_, _destroy_ методы (выполняются после инициализации и 
до уничтожения соответственно).

Далее мы рассмотрим непосредственно правило внедрения зависимостей.

## Инверсия зависимостей со Spring IoC

### Инициализация контекста  

В данной части рассмотрим работу со Spring IoC (inversion of control) - одной из реализаций принципа 
Dependency Inversion.

Как же работать с контекстом спринга? Для начала его необходимо проинициализировать. 

Способ его инициализации зависит от способа конфигурации. 

Далее мы будем рассматривать все примеры на Java конфигурации, потому что это самый распространенный пример
конфигурации (процентов так 
[95%](http://lurkmore.to/95%25_%D0%BD%D0%B0%D1%81%D0%B5%D0%BB%D0%B5%D0%BD%D0%B8%D1%8F_%E2%80%94_%D0%B8%D0%B4%D0%B8%D0%BE%D1%82%D1%8B) 
в относительно новых проектах). Остальные варианты рассмотрите самостоятельно.

```java
@Configuration
public class BeansConfiguration {
    
}

public class Main {
    
    public static void main(String[] args){
      final ApplicationContext ctx = new AnnotationConfigApplicationContext(BeansConfiguration.class);
    }
}
```

В данном примере, `BeansConfiguration` - это рутовый класс java-class конфигурации контекста.

В переменной `ctx` получим инициализированный контекст приложения.

### Вводная

Определимся с несколькими терминами:

* *Зависимость класса A типа B* — класс А имеет поле типа B (где B должен быть обязательно интерфейсом, 
за редким исключением).

* *Обязательная зависимость* — зависимость, без которой основная задача класса не выполнима. Например, если
некоторый сервис оперирует данными, то зависимость от репозитория будет считаться обязательной.

* *Опциональная зависимость* — зависимость, которая не вносит какой-либо весомый вклад в выполнении задачи класса, а 
является вспомогательной, например, какой-либо фильтр, транслятор исключений и так далее. Без данной зависимости
сервис все равно способен выполнить свою бизнес-задачу. 

* *Автосвязывание* — если у класс А есть зависимость от типа В и есть только один бин с этим типом, то Spring
автоматически внедрит зависимость, подставив бин типа В.

Для примера опишем несколько классов и интерфейсов. Из предыдущих лекций вы должны помнить, что любой 
класс, у которого есть поведение (то есть, в его методах заключена логика), должен реализовывать интерфейс,
в котором данная логика имеет формальный вид. Приступим:

```java
public interface Repository<T, ID> {} // (1)

public class FileStoreRepository implements Repository<Author, Long> {} // (2)

public class JpaRepository implements Repository<Author, Long> {} // (3)
```
```java
public interface AuthorService {} // (4)
```
```java
public interface PrettyPrinter {}  // (5)

public class CapitalizePrettyPrinter implements PrettyPrinter {}  // (6)
```

Для простоты мы не указываем методы в интерфейсах. Посмотрим на пример использования: 

```java
public class AuthorServiceImpl implements AuthorService {  // (7)

    private final Repository<Author, Long> repository; // (8)

    private PrettyPrinter prettyPrinter; // (9)

    private String fieldDependency; // (10)

    public AuthorServiceImpl(Repository<Author, Long> repository) { // (11)
        this.repository = requiredNonNull(repository, "Repository must be not null"); // (12)
    }

    @Override
    public Author getAuthor(Long authorId) {
        Author author = repository.findOne(authorId);

        if (prettyPrinter != null) {  
            String prettyName = prettyPrinter.doNameAsPretty(author.getName());
            author.setName(prettyName);
        }
        return author;
    }

    public void setPrettyPrinter(PrettyPrinter prettyPrinter) {  // (13)
        this.prettyPrinter = prettyPrinter;
    }
}
```

```java
@Configuration
public class BeansConfiguration { // (14)
    
}

public class Main {
    
    public static void main(String[] args){
      final ApplicationContext ctx = new AnnotationConfigApplicationContext(BeansConfiguration.class);
    }
}
```

Классы (2) и (3) представляют собой репозитории для манипуляции сущностями авторов в файловой системе и базе данных 
соответственно. Сервисы (4) и (5) инкапсулируют в себе некоторую бизнес-логику, которую реализуют классы (6) и (7).

Класс (14) является java конфигурацией.

Перед примерами работы с конфигурацией, разберем класс `AuthorServiceImpl` (7).

Согласно терминам из начала лекции: 

* Зависимость `repository` (8) является обязательной зависимостью, чье наличие является крайне важной. Поэтому
она является финальным полем, инициализируется через конструктор и дополнительно проверяется на null (12). 
* Зависимость `prettryPrinter` (9) является опциональной, так как она не влияет на выполнение бизнес-задачи 
сервисом. Для ее внедрения служит `setter` (13).
* Зависимость `fieldDependency` (10) нужна, чтобы показать как пример внедрения через поле. Как мы увидим это ниже,
то через java конфиг нельзя сделать это легко и непринужденно. Несмотря на то, что это очень популярный способ 
внедрения, этот способ является в корне неверным и не рекомендуется самим Pivotal (разработчиком и вендором спринга).

Наша задача - передать контроль за жизненным циклом перечисленных классов контексту спринга. 

### Конфигурация через java-class

#### Пример 1

Разберём следующий пример. Сервис `AuthorService` (4) при вызове метода `getAuthor(authorId)` должен возвращать
сущность автора. Также необходимо выводить авторов в лог в определенном формате. Авторы должны храниться в файловой системе. Наша задача - настроить конфигурацию спринга таким 
образом, чтобы получить приложение, соответствующее условиям задачи. 


Идём в конфигурацию (14) и добавляем необходимые бины:

```java
@Configuration
public class BeansConfiguration { // (14)
    
    @Bean(name = "authorService") // (15)
    public AuthorService authorService(
            Repository<Author, Long> repository, // (16)
            @Autowired(required = false /* 17 */ ) PrettyPrinter prettyPrinter // (18) 
        ) {
        AuthorService authorService = new AuthorServiceImpl(repository);
        if (prettyPrinter != null) {
            authorService.setPrettyPrinter(prettyPrinter); // (19)
        }
        return authorService; // (20)
    }   
    
    @Bean
    public Repository<Author, Long> fileRepository() { // (21)
        return new FileStoreRepository();
    }
    
    @Bean
    public PrettyPrinter prettyPrinter() { // (22)
        return new CapitalizePrettyPrinter();
    }
}
```

Теперь разберём пример чуть подробнее. В методе (15), который в спринге называется `factory-method`, создаётся
`authorService`. Для его создания нам надо получить его зависимости. Чтобы получить ссылки на другие бины, которые нужны
для создания сервиса, мы пишем их как аргументы (16),(18) factory метода. Так как `PrettyPrinter` (5) является 
опциональной зависимостью, то мы подсказываем спрингу (17), чтобы он не выдавал ошибку, если данного бина не будет.

После этого мы можем получить через спринг-контекст бин `authorService` и автора из файловой системы:

```java
public class Main {
    
    public static void main(String[] args){
      final ApplicationContext ctx = new AnnotationConfigApplicationContext(BeansConfiguration.class);
      AuthorService authorService = ctx.getBean(AuthorService.class);
      Author authorFromFile = authorService.getAuthorId(1L);
    }
}
```

В итоге мы получили искомое приложение, написав спринг конфигурацию.

#### Пример 2

По мере развития нашего проекта, его аудитория разрослась, и теперь мы больше не можем позволить себе хранить авторов в файлах.
Бизнес хочет аналитику, и в этот чёрный день вам создали тикет мигрировать в базу данных. Так же мы решили отказаться
от вывода авторов в лог, то есть от `PrettyPrinter` (5). 

Опустим перенос существующих данных. Нам необходимо изменить способ доступа к данным и вместо файлов использовать 
базу данных. Для этих целей нам пригодится `JpaRepository` (3), который так же реализует наш интерфейс `Repository`. 
Поэтому изменим спринг конфигурацию таким образом, чтобы вместо файлового репозитория использовался 
jpa репозиторий:

```java
@Configuration
public class BeansConfiguration { // (14)
    
    @Bean(name = "authorService") // (15)
    public AuthorService authorService(
            Repository<Author, Long> repository, // (16)
            @Autowired(required = false /* 17 */ ) PrettyPrinter prettyPrinter // (18) 
        ) {
        AuthorService authorService = new AuthorServiceImpl(repository);
        if (prettyPrinter != null) {
            authorService.setPrettyPrinter(prettyPrinter); // (19)
        }
        return authorService; // (20)
    }   
    
    @Bean
    public Repository<Author, Long> jpaRepository() { // (23)
        return new JpaRepository();
    }
    
}
``` 

Мы удалили бин `fileRepository` (21) и `prettyPrinter` (22) за ненадобностью и добавили новый бин
`jpaRepository` (23). Если внимательно посмотреть на конфигурацию `authorService` (15), то можно заметить, что
метод не изменился, так как мы заранее предусмотрели возможность отсутствия `prettyPrinter` и ссылаемся не
на конкретный репозиторий, а на интерфейс.

#### Резюме 

Мы рассмотрели на двух примерах небольшой период жизни, за который проект подвергся 
изменениям. Очень важный момент в этих примерах состоит в том, что, несмотря на изменения требований, менялась
только конфигурация спринга, а ранее написанный код (1) - (13) не менялся. 

Это происходит потому, что мы использовали такие SOLID принципы, как принцип единой ответственности и принцип
открытости-закрытости, согласно которым каждый класс выполяет одну задачу и закрыт для модификаций, но открыт 
для расширения.  


### Конфигурация через аннотации или "Spring, сделай все за меня, братишка"


Теперь посмотрим, как всё будет выглядеть, если сконфигурировать спринг через аннотации над самими классами:

```java
public interface Repository<T, ID> {} // (1)

@Repository // (2)
public class FileStoreRepository implements Repository<Author, Long> {} // (3)

public class JpaRepository implements Repository<Author, Long> {} // (4)
```
```java
public interface AuthorService {} // (5)
```
```java
public interface PrettyPrinter {}  // (6)

@Component // (7)
public class CapitalizePrettyPrinter implements PrettyPrinter {}  // (8)
```

Для простоты мы не указываем методы в интерфейсах. Посмотрим на пример использования: 

```java
@Service // (9)
public class AuthorServiceImpl implements AuthorService {  // (10)

    private final Repository<Author, Long> repository; // (11)

    private PrettyPrinter prettyPrinter; // (12)

    @Autowired // (13)
    private String fieldDependency; // (14)

    public AuthorServiceImpl(Repository<Author, Long> repository) { // (15)
        this.repository = requiredNonNull(repository, "Repository must be not null"); // (16)
    }

    @Override
    public Author getAuthor(Long authorId) {
        Author author = repository.findOne(authorId);

        if (prettyPrinter != null) {  // (13)
            String prettyName = prettyPrinter.doNameAsPretty(author.getName());
            author.setName(prettyName);
        }
        return author;
    }

    @Autowired(required = false) // (17)
    public void setPrettyPrinter(PrettyPrinter prettyPrinter) {  // (18)
        this.prettyPrinter = prettyPrinter;
    }
}
```

```java
@Configuration
@ComponentScan("здесь имя пакета, где лежат ваши бины") // (19)
public class BeansConfiguration { // (20)
    
}

public class Main {
    
    public static void main(String[] args){
      final ApplicationContext ctx = new AnnotationConfigApplicationContext(BeansConfiguration.class);
    }
}
```

Посмотрим, что изменилось:

* В `FileStoreRepository` (3) добавили аннотацию `@Repository` (2), которая говорит спрингу, что жизненный цикл
данного класса передают под управлению спринга. Данная аннотация является аналогом аннотации `@Component`, которую
рассмотрим чуть ниже. В отличии от неё, аннотацией `@Repository` помечаются бины, которые осуществляют операции по 
манипулированию сущностями в хранилищах.

* Над `CapitalizePrettyPrinter` (8) появилась аннотация `@Component` (7). Этой аннотацией помечаются все классы, 
которые мы хотим сделать бинами. Этой аннотацией следуют помечать те классы, которые не имеют отношения к хранению
данных или бизнес-логике. Например, какие-то инфраструктурные классы.

* Над сервисом `AuthorServiceImpl` (10) появилась аннотация `@Service` (9). Этой аннотацией помечаются все бины,
которые инкапсулируют в себе бизнес-логику. 

Теперь разберём, как показать спрингу, какие зависимости мы хотим внедрить.

Зависимость от `Repository` (11) мы получаем через конструктор (15); здесь ничего дополнительно указывать не надо,
т.к. чтобы создать объект сервиса, спрингу необходимо внедрить зависимость, указанную в аргументах
конструктора.

Зависимость от `PrettryPrinter` (12) мы получаем через сеттер (18), так как данная зависимость является 
опциональной. Для автосвязывания мы используем аннотацию `@Autowired` с флагом required = false (17), чтобы спринг
не выдавал ошибку в случае отсутствия бина.

Чтобы внедрить зависимость в поле `fieldDependency` (14), мы также используем данную аннотацию (13). Однако, ещё раз 
повторю, внедрять зависимости через поле запрещено (если это, конечно, не тесты).

Последним шагом укажем в конфигурации (20) аннотацию `@ComponentScan` (19), которая подскажет спрингу, в 
каком пакете искать наши бины.

