# Куда войти? Seize the block
[Telegram: Contact @kydavoiti](https://t.me/kydavoiti/17)

Сервисы, которые сейчас разрабатываются, часто являются интеграционный прослойкой между несколькими системами. Эта особенность налагает дополнительные условия на то, как сервисы должны работать с этим видом нагрузки. Здесь мы рассмотрим как разные виды приложений работают в условиях частого взаимодействия с устройствами *ввода-вывода* (далее *IO*). Если вам это интересно - добро пожаловать. Как всегда - обратная связь может быть отправлена мне в [Telegram: Contact @Im_Ilya](https://t.me/Im_Ilya) . 

> **DISCALAIMER**  
> Некоторые моменты могут быть и будут упрощены. Это может происходить в силу моей необразованности в данном вопросе или из-за моей лени 👿  
> Не считаю, что имеет смысл  рассматривать глубоко каждый из приведенных методов. Если интересно — я оставляю ссылки на материалы, а еще можно погуглить самому 👀  

## План рассказа
Будут рассмотрены приложения:
- Однопоточное приложение + блокирующий IO
- Многопоточное приложение + блокирующий IO 
- Однопоточное приложение + не блокирующий IO
- ~~Многопоточное приложение + не блокирующий IO~~ 
- Green-Threads

#### Пример решаемой задачи

Рассмотрим задачу на примере сервиса получения данных о пользователе. Данные пользователя — это *идентификатор*, *имя*, *фамилия*, *дата рождения*. Также имеются другие системы, которые предоставляют данные о нем: *серия* и *номер паспорта*, *адрес регистрации*, *история транзакций*. Сервис должен агрегировать эти данные по пользователю и отдавать клиенту.

#### План процесса обработки запроса на получение информации

- *Customer Info API* (*CI API*) — это сервис, которые будет отдавать пользователю агрегированные данные.
- *Customer Info DB* (*CI DB*) — это база данных, в которой хранятся основные данные пользователя: *идентификатор*, *имя*, *фамилия*, *дата рождения*. 
- *Customer Personal Info API* (*CPI API*) — это сервис, которые отдает персональный данные пользователя: *серия* и *номер паспорта*, *адрес регистрации*. Данные отдаются по *ID* пользователя.
- *Transaction History API* (*TH API*) — сервис, который отдает историю транзакций пользователя по его *ID*.

Чтобы сервис отдал полные данные — он должен обратиться в *Customer Info DB* (за время ß), откуда будет получена основная информация о пользователи. Она будет использована для получения остальных данных из *Customer Personal Info API* (за время ƒ), и *Transaction History API* (за время ∂).

> Прошу не судить архитектуру примера. В данном случае это не главное, суть во взаимодействии сервисов между собой.  

#### Аналогия с задачей из жизни

Каждый случай будет рассмотрен с точки зрения системы массового обслуживания клиентов. Заведение будет называться *ВамВдоволь*. Это стандартный кафе-ресторан. Процесс заказа будет состоять из нескольких шагов:
- Заказа изделия, которое нас интересует. 
	- В нашем случае это будет кофе и сэндвичи.
- Далее начинается готовка заказа.
	- Состав процесса приготовления заказа будет подробнее расписан в каждом случае.
- Заказ поступает клиенту, когда он будет готов целиком.

Аналогии будут раскрываться подробнее в ходе текста, чтобы в конце сложилась полная картина с ассоциациями, которые помогут запомнить основные принципы работы подходов.

##  Однопоточное приложение + блокирующий IO
Начать следует с самого простого случая работы. Это приложение, которое работает в 1 поток.

Рассмотрим процесс по шагам.

- В *Customer Info API* попадает новый запрос.
После поступления нового запроса сервис берет единственный,  выделенный под него поток, который будет выполнять поставленную ему задачу. Обработка текущего запроса не позволяет системе обрабатывать другие запросы. Другие потребители обязаны ждать окончания обработки текущего запроса.
- Совершается запрос к *Customer Info DB*.
- Далее в произвольном порядке совершается запрос в одну из систем, чтобы составить полный ответ:
	- *CPI API* для получения персональных данных.
	- *TH API* для получения транзакций пользователя.
Запросы могут быть сделаны в любом порядке, но обязательно после запроса в *CI DB*.

Время обработки одного запроса будет равно `ß + (ƒ + ∂) = ß + (∂ + ƒ)`. 

Поведение данного вида сервиса в любой момент времени просто предсказать. В тоже время работает он не оптимальным образом, что будет видно далее, в сравнении с другими способами обработки.

### Живой пример

Когда мы начали развивать свой бизнес *ВамВдоволь*, сначала мы работаем одни. Мы производим продукты сами и сами обслуживаем клиентов.

Мы работаем в небольшом киоске. У нас есть кофе машина, ингредиенты для его приготовления, а также все для производства вкусных сандвичей.

Процесс обслуживания клиента будет выглядеть таким образом: 
- Новый клиент делает заказ, общаясь с нами.
- После заказа мы идем готовить заказ.
	- Готовим кофе.
	- Готовим сандвичи.
Обработать их также можно в любом порядке. Но мы не можем это сделать до составления заказа и его оплаты.

Время обработки заказа клиента будет равно сумме времени готовки всех частей заказа. Мы еще не опытные, наши инструменты не лучшие на рынке, что делает этот процесс сложнее. Алгоритм понятен, оптимизировать можно, но в текущих реалиях это сложно, либо невозможно.

## Многопоточное приложение + блокирующий IO
Вторым по сложности является способ с блокирующим IO, только обработка уже происходит в нескольких потоках.

В данном случае каждый запрос к сервису выполняется в отдельном потоке.  Это позволяет избавить пользователей от нужды стоять в очереди, если чей-то запрос уже обрабатывается.

Также рассмотрим процесс по шагам:
 - В *Customer Info API* попадает новый запрос.
Каждый запрос в *CI API*  выполняется в одном из потоков приложения. Система может создавать новый поток на каждый запрос, либо переиспользовать потоки из некоторого пула потоков. У обоих подходов есть свои минусы и плюсы., которые будут рассмотрены ниже 
- Совершается запрос к *Customer Info DB*.
- Далее в произвольном порядке совершается запрос в одну из систем, чтобы составить полный ответ:
	- *CPI API* для получения персональных данных.
	- *TH API* для получения транзакций пользователя.
Запросы могут быть сделаны в любом порядке, но обязательно после запроса в *CI DB*. Опытные разработчики сразу заметят, что эти запросы можно будет сделать параллельно, запустив их в отдельных потоках.

Время обработки запроса будет: `ß + max(ƒ, ∂) = ß + max(∂, ƒ)`, если запросы будут выполняться параллельно. Эт быстрее ,чем однопоточный способ. Но не всегда приложение будет так работать, все зависит от того сколько у нас память либо потоков в пуле.

Этот способ является стандартным во многих языках и фреймворках ([*Tomcat*](https://tomcat.apache.org/tomcat-7.0-doc/config/executor.html), [Tomcat также обладает NIO конвектором](https://dzone.com/articles/understanding-tomcat-nio), который работает по подходам, описанным ниже). Если рассматривать случай с созданием нового потока на каждый запрос, то это будет работать только тогда, когда у нас бесконечное кол-во ядер процессора и памяти. В случае ограниченных ресурсов будут проблемы с количеством операцией [смены контекста](https://ru.wikipedia.org/wiki/Переключение_контекста) потока и проблемы с памятью, так как каждый поток имеет свой *stack* вызовов, который в [*Java* занимает *1 Мб* по умолчанию на *x64* архитектуре](https://www.oracle.com/java/technologies/hotspotfaq.html).

Также можно каждый запрос в *CPI API* и *TH API* делать в отдельном потоке из пула потоков. Можно пойти дальше и создать собственный пул для каждого клиента, это позволит менять его размеры отдельно, следовательно более гибко настраивать приложение под ваши нужды. С пулами потоков могут быть проблемы при больших нагрузках, так как потоки там могут просто закончиться. О том как выбрать размер пула потоков можно почитать в [интернете](https://engineering.zalando.com/posts/2019/04/how-to-set-an-ideal-thread-pool-size.html). 

### Живой пример
Аналогом такой системы будет развитие *ВамВдоволь* в сторону увеличения штата. Работники и качество инструментов оставят желать лучшего, но их станет больше.

У нас будет много касс, много рабочих, много кофеварок и много мест для приготовления сандвичей.

Процесс обслуживания клиента будет выглядеть примерно вот так: 
- Новый клиент делает заказ, общаясь с одним из работников за одной из касс.
- После заказа мы идем готовить заказ, либо отправляемся передать заказ кому-то другому, чтобы он его готовил.
	- Происходит готовка кофе.
	- Происходит готовка сандвичей. 
Затем работник, что принимал заказ собирает его и отдает клиенту.

Если далее проводить аналогии:
- Работники — это кол-во потоков в нашем приложении.
- Кассы — это ограничение параллелизма в работе принятия запросов сервисом.
- Инструменты приготовления чего-либо — это ограничения пула потоков для каждого клиента внешнего сервиса.

##  Однопоточное приложение + не блокирующий IO
Данный вид работы сервиса является более продвинутым, чем прошлый, так как он эффективнее расходует ресурсы приложения, чем создание новых потоков. Подробнее про эту схему работы можно прочитать [на примере Node.JS](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/) 

В данном случае будет использоваться возможность *nonblocking-io*, она позволяет вашему приложению не ждать ответа системы, а заниматься чем-то полезным. Приложение будет в определенный момент, когда ваша система сделает другую полезную работу, проверять у ОС появились ли для неё новости о работе с внешними системами, и приступит к обработке ответов, только в момент их появления, пока их нет, она сможет обрабатывать другие запросы.

Реализуется это с помощью [select / poll / epoll](https://habr.com/en/company/infopulse/blog/415259/) и шаблона [Event-Loop](https://en.wikipedia.org/wiki/Event_loop). Если описывать этот подход в паре слов, то приложение пишется как обработка событий о поступлении новых данных на чтение и событий о конце записи. Подробнее можно почитать об этом по ссылкам, приведенным выше в этой главе .

Стоит отметить, что не все операции ввода-вывода можно таким способом оптимизировать. Неблокирующий ввод сложно реализовать для файлов, так как доступ к ним может осуществляться в произвольном порядке, что не так для сокетов, через которые мы работает с внешними системами. Хотя есть варианты для [асинхронной работы с файлами](https://habr.com/en/company/badoo/blog/439972/), я не видел его использования в больших продуктах. Буду рад примерам 😄

Рассмотрим процесс по шагам:
 - В *Customer Info API* попадает новый запрос.
Создается задача на то, чтобы его обработать, которая будет взята в работу как только освободится наш поток выполнения. 
- Делаем запрос в *Customer Info DB*.
Когда наш поток дошел до этапа запроса в БД, он запускает операцию ввода-вывода, а сам уходит выполнять другую работу. Обработка результата будет выполнена по мере его готовности.
- Далее в произвольном порядке создаются задачи на работу с каждой из систем, чтобы составить полный ответ:
	- *CPI API* для получения персональных данных.
	- *TH API* для получения транзакций пользователя.
Тут сервис делает задачи, а после их результаты обрабатывает с помощью *Callback* (как это принято в [JS](https://learn.javascript.ru/callbacks)), либо иным способом. Способ работы с неблокирующим вводом может сильно отличаться в разных фреймворках.

Время обработки запроса будет: `ß + max(ƒ, ∂) = ß + max(∂, ƒ)`, так как запросы будут выполняться параллельно. Что дает нам преимущество над обычным однопоточным приложением, но идентично времени работы в многопоточный среде без нагрузки (когда потоков достаточно).

Если сравнивать  данный метод с прошлыми, то
- Он грамотно работает с одним потоком и он может выполнять полезную работу, пока происходят запросы ко внешним системам.
- У нас нет вопросов, которые нам задает второй способ. Не надо думать о том какой должен быть размер пула потоков и о сменах контекста, а просто пишем код, который сам по себе выполняется так, что его не надо масштабировать, он динамически адаптируется под нагрузку. 

### Живой пример
Данный случай является интересным. Он является альтернативой прошлому способу развития.

Мы работаем одни, но наше заведение стало очень технологичным, то есть мы решили расширяться не как во 2 случае, а вложившись в технологии.
Мы сделали приложение в котором пользователи могут заказывать, не отвлекая нас. Мы купили полностью автоматизированную кофеварку и машину для сандвичей, мы должны только ходить между ними и запускать готовку, а затем собирать её плоды, отдавая на пункт выдачи, где клиент сам заберет заказ.

Тут наша производительность напрямую зависит от наших инструментов, чем они лучше - тем мы производительней, мы на неё никак не влияем. Это можно сравнить с выбором реализации асинхронного взаимодействия с сокетом.

Процесс обслуживания клиента будет выглядеть следующим образом: 
- Новый клиент делает заказ в приложении.
- Заказ отображается для нас в очереди дел.
- Мы берем одну задачу из очереди.
В очереди могут быть различные задачи:
	- Поставить кофе.
		- Она может создать новую задачу, когда кофе приготовится: “Получить кофе”
	- Поставить сандвичи.
		- Она может создать новую задачу, когда кофе приготовится: “Получить кофе”
	- Отдать заказ на пункт выдачи.
- Клиент забирает заказ как он готов.
Как видно, мы почти не участвуем в работе, но она идет. Мы не будем простаивать, если заказов будет много, но так как работник один, мы скорее всего будем не успевать обслуживать систему, если посетителей много.

## Многопоточное приложение + не блокирующий IO
> Не вижу особого смысла останавливаться на этом пункте. Он будет являться почти полной копией прошлого, с одной разницей, что не 1 поток разбирает задачи, а несколько. Это поможет, если у нас многоядерная система. По этой теме можно почитать: [Understanding Reactor Pattern: Thread-Based and Event-Driven - DZone Java](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve), [Spring Webflux: EventLoop vs Thread Per Request Model - DZone Java](https://dzone.com/articles/spring-webflux-eventloop-vs-thread-per-request-mod)  
> В живом примере у нас просто будет несколько работников, работающий над одним пулом задач. То есть пулы переходят с уровня инструментов на уровни задач.  

## Green-Threads
Это, по-моему, самый сложный способ, из рассмотренных мной здесь. Но в свою очередь самый сложный способ предоставляет нам возможность писать код так как мы это делаем в 1 способе, используя преимущества не блокирующего ввода вывода. Такая схема работы применяется в Go [1](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html) [2](https://tpaschalis.github.io/goroutines-size/),  [Kotlin](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md). Также в [Python](https://docs.python.org/3/library/asyncio.html) используется похожий метод. 

Данный метод является способом снизить затраты на работу с потоками, сделав оптимизацию процесса переключения контекста. Она в них делается более осознанно, в моменты, когда должна быть сделана. Обычно за этим следит среда исполнения. Также там оптимизируются и другие аспекты потоков, но они не столь важны в нашем случае.

Рассмотрим процесс по шагам.
- В *Customer Info API* попадает новый запрос.
Его обработка запускается в новой корутине.
- Совершается запрос к *Customer Info DB*.
- Далее в произвольном порядке совершается запрос в одну из систем, чтобы составить полный ответ:
	- *CPI API* для получения персональных данных.
	- *TH API* для получения транзакций пользователя.
Запросы могут быть сделаны в любом порядке, но обязательно после запроса в *CI DB*. Также они могут быть запущены в отдельных корутинах, а потом объединены.

Время обработки запроса будет: `ß + max(ƒ, ∂) = ß + max(∂, ƒ)`, так как запросы будут выполняться параллельно. Что также быстрее стандартного способа и не имеет минусов многопоточного приложения.

Как видно, этот вариант почти ничем не отличается от блокирующего кода, вся разница от нас скрыта. Такой эффект называется [Structured concurrency](https://medium.com/@elizarov/structured-concurrency-722d765aa952).  Сама среда исполнения распределяет корутины по потокам ОС, так, чтобы потоки не стояли в состоянии ожидания, а работали, выполняя корутины. То есть *блокировки* (прерывания) есть на уровне корутин, но не на уровне потоков.  Потоки всегда работают, выполняя код, если им есть что выполнять 😃

### Живой пример
Данный метод сложно описать аналогий с реальным миром, так как он слишком синтетичен, в нем происходит полная автоматизация, которой просто сложно достигнуть в жизни.

Теоретически теперь есть бесконечных набор касс, которых когда надо — больше, когда не надо — меньше. У нас есть несколько работников, которые работают со своим инструментами. При этом у них столько инструментов, сколько надо для  обработки заказов, также все эти все инструменты умные и могут сами работать, но когда они работают — рабочий пропадает, а на кухне появляется другой рабочий, который дождался завершения своей готовки. *Важно ограничение*, у нас не могут находится сразу все работники на кухне, а только определенное кол-во, которое может там нормально работать. Это схоже с ограничением потоков в приложении, которые запускают на себе обработку корутин.

Процесс обслуживания клиента станет выглядеть примерно вот так: 
- Новый клиент делает заказ на кассе.
- Заказ делается одним из работников.
- Когда работник дошел до операции в которой ему не надо участвовать, он исчезает из кухни. На кухне появляется другой работник, который может продолжить работу, на которой он закончил, если она может быть продолжена, если нет — он не появляется.
Операции, которые не требуют работы работника:
	- Готовка кофе.
	- Готовка сандвичей.
	- Отдать заказ клиенту.
- Клиент забирает заказ как он будет готов.

Как видно, реализовать такое в жизни не является возможным (надеюсь пока что 😁, либо у меня не хватило фантазии на пример)

Как видите данная схема очень высокого уровня, что налагает дополнительную работу на её организацию.

## Вывод
Я  рассмотрел 4 способа работы с *вводом-выводом*. Из них явно плох только первые, остальные хороши в своих областях. Когда вам нужно применять один из них — зависит от задачи, поэтому важно понимать как они работают, с чем я вам и надеюсь, помог. Спасибо за внимание! Не забывайте писать с обратной связью 😉
