# CRUD приложение на Spring Boot 2

[Проект](https://github.com/eutkin/crud)

## Требования к окружению

* git
* Intellij Idea Ultimate (мы же не простые ребята)
* Плагины для идеи:
    
    * .ignore
    
    * lombok plugin
    
* maven (можно встроенный в идею)

## Инициализация проекта

Проект можно проинициализовать либо на [сайте](https://start.spring.io/), либо через ultimate idea.

Выбираем `New Project`:

![step1](img/crud-spring1.png)

Далее:

![step2](img/crud-spring2.png)

1. GroupId -- ваш идентификатор как автора. Если не знаете, что писать, пишите 
io.github.<ваш ник на гитхабе в нижнем регистре>
2. ArtifactId -- название проекта. Пишется в нижнем регистре, слова разделяются `-`
3. Версия Java. В большинстве компаний используется 8 версия, но 11 уже стабильна и для новых проектов можно 
использовать ее
4. Версия. Версия состоит из 4 частей: 
    
    * **1**.0.0-SNAPSHOT. Мажорная версия. Инкрементируется при масштабных изменениях, при сильных изменениях в API.
    
    * 1.**0**.0-SNAPSHOT. Минорная версия. Инкрементируется при добавлении новой фичи
    
    * 1.0.**0**-SNAPSHOT. Патч. Инкрементируется при исправлении бага
    
    * 1.0.0**-SNAPSHOT**. Квалифер. Альфа, бета, снапшот и так далее

5. Рут пакет.

Следующий шаг помогает выбрать зависимости:

![step3](img/crud-spring3.png)    
      
Рекомендуется набор зависимостей, как на скриншоте.

* **Lombok** -- кодогенератор 
* **Configuration Processor** -- позволяет интегрировать ваши конфигурационные параметры в мета файл конфигурации 
спринга. На практике эта информация нужна IDE, чтобы работало автодополнение и так далее.
* **Web** -- и так все понятно
* **REST Docs** -- документация к апи (swagger)
* **JPA** -- ORM (hibernate)
* **H2** -- встраиваемая бд, идеальный вариант для разработки
* **Actuator** -- предоставляет API, с помощью которого можно мониторить приложение
* **Security** -- модуль для авторизаций и всего такого. 

P.S.

Я забыл добавить Security на скрине, но переделывать мне лень. Поэтому добавьте сами в pom.xml в dependencies:
```xml
 <dependency>        
    <groupId>org.springframework.boot</groupId>           
    <artifactId>spring-boot-starter-security</artifactId>           
</dependency>
```
       
                
Получаем проинициализированный проект:

![step4](img/crud-spring4.png)

* **.mvn** -- здесь лежит встроенный мавен. Используется при настройке [CI](https://habr.com/ru/post/352282/).
* **src**
    
    * **main**
    
        * **java**  -- исходники
        
        * **resources** -- ресурсы. Конфиги и все то, что окажется в classpath
            
            * **static** -- используется только в веб. В папке лежат статические ресурсы, js, css, etc
            
            * **templates** -- используется только в веб. Лежат шаблоны для шаблонизаторов.

    * **test** -- папка с тестами
        
        * **java** -- исходники тестов
        
        * **resources** -- уже было, но только для тестов
        
* **.gitignore** -- специальный файл-конфиг, который позволяет не пихать в гитхаб лишнего
* **mvnw|mvnw.cmd** -- так называет мавен wrapper. Позволяет использовать мавен из папки .mvn. 
* **pom.xml** -- описание сборки для мавена

## DDL скрипт

Вспомним нашу ER-диаграмму:

![er](img/arch_pr8.png)                    
               
И напишем ddl скрипт, который будет задавать структуру базы данных. Используем синтаксис postgres'a, так как
в бою будет он. Чтобы H2 мог разобрать с диалектом postgres'a, добавляем в url параметр `MODE=PostgreSQL`.

Модифицируем `application.properties`:

```properties
# показываем генерируемый хибернейтом sql
spring.jpa.show-sql=true\
   
# включаем h2 консоль, что можно было посмотреть содержимое бд
spring.h2.console.enabled=true

# отключаем автогенерацию схемы по entity, так как схему будем создавать через sql
spring.jpa.hibernate.ddl-auto=none

# Нужно добавить MODE=PostgreSQL. Нормальный способов нет, поэтому пишем всю строку подключения и 
# и добавляем параметр туда
spring.datasource.url=jdbc:h2:mem:%s;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=false;MODE=PostgreSQL

# отключаем пока security
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

Добавляем в ресурсы файл `schema.sql`, содержащий следующий ddl скрипт

```postgresql
create table authors (
  author_id bigint       not null,
  name      varchar(255) not null,
  primary key (author_id)
);
create table genres (
  genre_id bigint not null,
  genre    varchar(255),
  primary key (genre_id)
);

create table publishers (
  publisher_id   bigint       not null,
  publisher_name varchar(255) not null,
  primary key (publisher_id)
);

create table books (
  book_id           UUID        not null,
  publish_date      date,
  short_description varchar(1000) not null,
  genre_id          bigint references genres(genre_id),
  publisher_id      bigint references publishers(publisher_id),
  primary key (book_id)
);

create table users (
  login        varchar(255) not null,
  display_name varchar(255),
  password     varchar(255),
  primary key (login)
);

create table booklists (
  booklist_id UUID not null,
  name        varchar(255),
  author      varchar(255) references users(login),
  primary key (booklist_id)
);

create table booklists_books (
  booklist_id binary not null references booklists(booklist_id),
  book_id     binary not null references books(book_id),
  primary key (booklist_id, book_id)
);

create table booklists_users (
  booklist_id UUID       not null references booklists(booklist_id),
  login       varchar(255) not null references users(login),
  primary key (booklist_id, login)
);

create table books_authors (
  book_id   UUID not null references books(book_id),
  author_id UUID not null references authors(author_id),
  primary key (book_id, author_id)
);
```           

## Создание моделей

Модели можно сгенерировать либо таблицам в базе (способ поищите сами, их много), либо написать руками.

Код моделей можете посмотреть на гитхабе, на примере разберем только одну. 

```java
@Entity  // Обязательная аннотация-маркер, что dto является сущностью
@Table(name = "booklists") // аннотация для указания имени таблицы и схемы
@Getter // ломбоковские аннотации, которые генерят геттеры и сеттеры
@Setter
public class Booklist {

    @Id //обязательная аннотация для поля-первичного ключа
    @GeneratedValue(generator = "org.hibernate.id.UUIDGenerator") //генерирует uuid автоматически при сохранении в бд
    @Column(name = "booklist_id") //аннотация для указания параметров маппинга и генерации схемы
    private UUID id;

    // не является обязательной, имя поля по умолчанию выступает как имя столбца
    private String name;

    @ManyToOne // аннотация для связи Много-к-Одному. У нескольких буклистов может быть только один автор
    @JoinColumn(name = "author") // имя стоблца, по которому можно сделать join для получения информации об авторе
    private User author;

    @ManyToMany // связь много-ко многим
    // здесь указывается колонки в таблице
    @JoinTable(name = "booklists_users",
        joinColumns = @JoinColumn(name = "booklist_id"), 
        inverseJoinColumns = @JoinColumn(name = "login"))
    private Set<User> users;

    @ManyToMany
    @JoinTable(name = "booklists_books", 
        joinColumns = @JoinColumn(name = "booklist_id"), 
        inverseJoinColumns = @JoinColumn(name = "book_id"))
    private Set<Book> books;

}
``` 

Надо еще бы указать `equals` и `hashCode` но сделаем это позже.       