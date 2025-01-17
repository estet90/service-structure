# Идеальная структура сервиса
Привет! Хотелось бы поделиться с вами ~~своей мудростью~~ примером структуры, которая подойдёт практически для любых сервисов, вне зависимости от того, гоняете ли вы JSON-ы в микросервисах или разрабатываете монолитное решение со сложной бизнес-логикой.  
Хотя данный материал рассчитан на бэкенд-разработчиков, он может быть интересен и полезен не только им. То, что будет описано далее, достаточно просто для понимания, однако понимать и делать - это не одно и то же. И я бы не стал писать эту статью, если бы не сталкивался на разных проектах с одними и теми же проблемами.  
Я пишу на Java. Этот язык, как и многие другие, даёт возможность разделять исходный код на пакеты и модули. Эти концепции я буду использовать в дальнейшем для описания структуры.

### Не хочу читать статью, хочу узнать, как мне теперь структурировать сервисы, и вернуться к ~~просмотру мемов~~ работе   
**тут будет картинка**  

## Что не так
Хорошо, когда в команде есть договорённости по структурированию кода. Плохо, что они не всегда являются достаточно подробными и однозначными. Часто ли приходя на новый проект вам приходилось видеть подоную структуру пакетов?  
```
ru.superbank.superservice:
    controller:
        CustomerController
        MoneyController
    domain:
        Customer
        Money
    service:
        CustomerService
        MoneyService
    repository:
        CustomerRepository
        MoneyRepository
```
Думаю, что часто. И выглядит вроде неплохо, всё аккуратно разложено по пакетам, понятно, где точка входа, где лежит обработчик, и как он ходит в базу. А если в контроллере ещё и логики нет, можно считать что вам крупно повезло! Но такое возможно только на старте проекта. 

### Пакет service
Со временем в контроллерах добавляются методы, в классах из пакета **service** появляются новые обработчики, и они начинают очень быстро увеличиваться в размерах. Для того, чтобы понимать, что в них вообще происходит, уже нужно писать документацию (в моём случае javadoc). Но т.к. даже описание сигнатуры и назначения методов может занимать много места, для этого можно выделить интерфейсы. Т.о. структура пакета **service** будет выглядеть примерно так:
```
ru.superbank.superservice:
    service:
        CustomerService
        CustomerServiceImpl
        MoneyService
        MoneyServiceImpl
```
или даже так:
```
ru.superbank.superservice:
    service:
        CustomerService
        MoneyService
        impl:
            CustomerServiceImpl
            MoneyServiceImpl
```
По мере роста проекта в нём кроме интеграций с базой появляются интеграции с другими сервисами, кэшами, брокерами сообщений. Куда обычно кладут интеграции? Конечно же в пакет **service**! И конечно не зыбывают сделать по интерфейсу для каждой интеграции.
```
ru.superbank.superservice:
    service:
        CustomerService
        MoneyService
        impl:
            CustomerServiceImpl
            MoneyServiceImpl
        kafka:
            KafkaService
            impl:
                KafkaServiceImpl
        dictionaty:
            DictionaryRestService
            impl:
                DictionaryRestServiceImpl
```
По мере усложенения логики возникает соблазн переиспользовать отдельные фрагменты. И это хорошо, все же знают про принцип DRY. А с учётом того, что у нас уже есть интерфейсы и подробная документация по сигнатуре и назначанию методов, мы можем считать, что проблем у нас не будет. Но это не так. Основная проблема в том, что классы из пакеты **service** могут вызывать друг друга, а сами цепочки вызовов быть очень длинными. А если документация не была вовремя актуализирована, можно получить, неприятную ситуацию, когда вызываемые методы делают не только то, что в указано в документации. Упрощённый пример из реальной жизни:
```java
class BusinessService {

    KafkaService kafkaService;
    /**
     * выполнить какое-то действие и отправить ответ в топик
     */
    public void process(Request request) {
        Response response = innerProcess(request);   
        kafka.sendResponse(response);
    }
}

interface KafkaService {

    /**
     * отправка сообщения в топик
     */
    void sendResponse(Response response)
}

class KafkaServiceImpl implements KafkaService {

    KafkaSender kafkaSender;
    SaveResponseService saveResultService;

    void sendResponse(Response response) {
        KafkaMessage message = toKafkaMessage(response);
        kafkaSender.send(message);
        saveResultService.save(response);
    }
}
```
Тут мы ожидали, что сообщение будет отправлено в топик, но судя по реализации оно ещё и сохраняется. В данной ситуации это может быть и нужно, но переиспользовать такой метод точно не стоит.

### Пакет domain
Он же **dto**, он же **model**, суть от этого не меняется. Пока проект небольшой, всё хорошо, и там лежат сущности для работы с базой (вы же используете подход contract-first и генерируете dto для контроллеров и различных интеграций?). Но по мере роста проекта там появляется много разных классов относящихся не только к базе. Это могут быть как запросы-ответы для внешних сервисов (а ведь их можно было генерировать!), так и различные вспомогательные DTO, которые используются, например, для агрегации данных. На одном проекте я видел сервис-монолит, который разбили на модули. Один из модулей так и назывался - **domain**. Там лежали ВООБЩЕ ВСЕ сущности, которые так или иначе исопльзовались в сервисе. И ориентироваться в нём было очень сложно.

### Пакет controller
Вроде всё логично, в пакете лежат классы и методы, которые являются точкой входа в приложение. Но приложение может иметь не только синхронный API. Ваш сервис может получать сообщения из различных брокеров, запускать какие-то задачи по расписанию, в нём может быть организован асинхронный обмен сообщениями внутри самого приложения. Где будут лежать классы обработчики для всего этого вы уже догадались.

## Что с этим делать

### API сервиса в отдельном пакете
В первоначальном варианте отдельного пакета удостоились только контроллеры, но, как было сказано выше, не только они могут быть точкой входа в ваше приложение. Всё это должно быть выделено в отдельный пакет:
```
ru.superbank.superservice:
    entrypoint:
        controller:
            CustomerController
        consumer:
            KafkaConsumer
        scheduler:
            TaskScheduler
```
Эти классы не должны содержать логики, только первичная обработка запроса (например валидация).  

### Отдельный обработчик для каждого метода API
Да, именно так. Отдельный класс для каждой "ручки" с ОДНИМ публичным методом. Что это нам даёт? Мы не только избавляемся от классов, в которых может быть 1000 и более строк, содержащих в себе логику для обработки нескольких методов API, но и от необходимости делать интерфейсы ради документации. Взамен мы получим на порядок больше файлов, но их уже можно структурировать, используя подпакеты:
```
ru.superbank.superservice:
    logic:
        customercontroller:
            CreateCustomerOperation
            UpdateCsutomerOperation
```
Будет ли пакет называться **logic**, **handler** или как-то ещё, не важно. Главное, чтобы такой пакет был, и классы в нём **не вызывали друг друга**.  
Не всегда можно вместить всю обработку в один класс, особенно если в зависимости от параметров запроса она может иметь разные сценарии, либо саму обработку логично разбить на шаги. В этом случае можно добавить отдельные классы-обработчики, которые будут вызываться из основного класса, выполняющего роль оркестратора.

### Разделение бизнес-логики и интеграций
Все интеграции должны быть вынесены в отдельный слой и разделены по соответствующим пакетам. Работа с БД - тоже интеграция, т.е. структура должна выгдлядеть примерно так:
```
ru.superbank.superservice:
    integration:
        repository:
            CustomerRepository
            CustomerRepositoryAdapter
        rest:
            DictionaryRestClient
            DictionaryRestClientAdapter
        kafka:
            KafkaSender
            KafkaSenderAdapter
```
При именовании почти всех таких классов принято использовать суффикс Service. Исключением являются только классы для работы с БД, они могут иметь суффиксы Repository, Dao, какие-то ещё. Сейчас мы видим, что классов с суффиксом Service нет вообще, зато появились какие-то Adapter-ы. Звучит знакомо. Точно, это же что-то из гексагональной архитектуры! Только вместо портов Sender, Client и Repository, которые по сути ими и являются.  
Адаптеры вмещают в себя часть бизнес-логики, они формируют запрос к нужному ресурсу, вызывают соответствующий порт и при необходимости валидируют ответ. Они, в отличие от классов-обработчиков, могут иметь по несколько публичных методов, но точно так же **не должны вызывать друг друга**.  
Если у сервиса много интеграций, таких классов так же будет немало. И как их компановать внутри пакета **integration**, решать вам.  
Порты выполняют отправку сформированного адаптерами запроса и обработку ошибок. Пример реализации:
```java
class CustomerRepositoryAdapter {

    CustomerRepository customerRepository;

    public List<Customer> getCustomers(CustomerIdsHolder holder) {
        List<Long> customerIds = holder.getCustomerIds();
        List<Row> customerRows = customerRepository.getCustomersByIds(customerIds);
        if (customerRows.isEmpty()) {
            throw new BusinessException()
        }
        return toCustomers(customerRows);
    }
}

class CustomerRepository {

    DbClient dbClient;

    public List<Row> getCustomersByIds(List<Long> customerIds) {
        try {
            return dbClient.select("SELECT customer WHERE id IN :customerIds")
        .bind("customerIds", customerIds)
        } catch (SqlExceltion e) {
            throw new DbException(e);
        }
    }
}
```
Адаптеры как правило работают только с соответствующим им портом. Исключением являются адаптеры для БД, т.к. порой в одном методе можно собрать ответ из нескольких методов разных репозиториев, чтобы потом отдать агрегированные данные.  
Ни в адаптерах, ни в портах не могут инициироваться транзакции. Они должны начинаться на уровень выше, в обработчиках, т.к. во-первых транзакция может затронуть несколько адаптеров, а во-вторых методы адаптеров могут переиспользоваться и не факт, что в каждом случае транзакция будет нужна.

### Наш любимый пакет "service"
Теперь у нас есть операции, клиенты, сендеры, репозитории и прочие адаптеры. Но мы, разработчики, привыкли к классам типа Service! И самое лучшее применение для таких классов - логика, которая может быть переиспользована, либо логическое объединение интеграций. Добавим же, наконец, пакет **service**:
```
ru.superbank.superservice:
    service:
        CacheService
        AggregationService
```
Эти классы, в отличие от обработчиков, могут содержать более одного публичного метода. В них могут быть инициированы транзакции. И конечно же **они не должны вызывать друг друга**. 

### Отдельные пакеты для моделей там, где они используются
Вы же не используете одни и те же DTO и для базы и для внешнего API? Тогда рано или поздно таких классов может стать очень много. Чтобы не запутаться в моделях в большом проекте, их нужно держать там, где они используются. Т.е. представления таблиц из БД должны лежать там же, где происходит работа с базой (пакет **integration.repository**), DTO запросов и ответов от внешнего сервиса там, где он вызывается (**integration.rest**).  
Не стоит так же забывать про маппинг одних сущностей в другие. Часто такие преобразования делают в классах, содержащих бизнес-логику или внешний вызов. Это не только ухудшает читаемость кода, но и усложняет отладку, т.к. при таком подходе состояние объекта может меняться в разных местах. Именно поэтому маппинг должен выделяться в отдельные классы, на которые, кстати, можно написать отдельные юнит-тесты.  А ещё из маппера, в отличие от большинства описанных ранее классов, можно вызвать другой маппер.  
Пример структуры с мапперами и DTO:
```
ru.superbank.superservice:
    entrypoint:
        controller:
            domain:
                GetCustomersResponse
            mapper:
                GetCustomersResponseMapper
    integration:
        repository:
            domain:
                Customer
            mapper:
                CustomerMapper
```

## Что это нам даёт
### Единообразная структура
Новый разработчик не будет страдать, глядя на 100 сервисов, написанных по-разному. Он без труда найдёт точку входа в сервис и сразу будет видеть, какие интеграции у него есть. При наличии хорошей документации на вики новый разработчик сможет достаточно быстро править баги и вносить небольшие доработки (добавить пару полей, новую интеграцию) в уже существующие методы. И ему не придётся для этого лопатить весь проект. Проверено на себе. 6,5 лет назад я, будучи начинающим специалистом, пришёл на такой проект. И очень быстро, не вникая глубоко в бизнес, начал делать фичи. Описанная структура отчасти повторяет то, что я там тогда подсмотрел (а потом внедрил на 2 проектах), немного отличается неймингом и является продолжением тех самых идей, но лучше масштабируется.

### Лучшая читаемость кода
Из классов обработчиков вынесено всё лишнее, и они содержат только бизнес-логику. Если при этом давать понятные названия методам, можно читая код сверху вниз понять, что происходит при обработке, даже если вы этот код в первый раз видите. Все преобразования вынесены в отдельные классы, что так же повышает читаемость кода.

### Абстрации одного уровня не зависят друг от друга
Меняя код в одном обработчике, вы гарантировано не сломаете другой. Это важное отличие от "плоской" структуры, где сервисы могут как попало вызывать друг друга, и одна из основных причин использовать такой подход.

### Проще писать тесты и ориентироваться в них
Когда у вас есть CustomerService на 1000 строк, вам придётся либо сделать класс CustomerServiceImpl на 5000 строк, либо делать по одному классу на каждый его метод, чтобы по-честному его протестировать (не процента покрытия ради, а пользы для). Если же для каждого обработчика есть отдельный класс, то для него можно сделать и отдельный тест-класс. Хотите сделать интеграционный тест? Вызовите в тест-классе соответствующий метод контроллера и добавьте кроме проверки ответа проверку вызовов соответствующих методов (для Java это делается с помощью `@Spy`). Хотите написать юнит-тест? Замокайте все вызываемые адаптеры (а лучше вызываемые в них порты) и проверьте, как происходит их вызов. В случае, когда условный Service1 вызывает Service2, который вызывает Service3 и т.д. намного сложнее понять, где использовать `@Spy`/`@Mock`.