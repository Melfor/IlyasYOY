# Куда Войти? Интерфейсы
[Telegram: Contact @kydavoiti](https://t.me/kydavoiti/21)

Привет! В этом тексте я поделюсь тем, почему я считаю, что всегда надо пользоваться интерфейсами в **Java**, может это и применимо будет к другим языкам и технологиям, но ответы исходят из опыта работы с **Java** + **Spring**. Кому интересно, добро пожаловать! Как всегда, пишите мне в [Telegram](https://t.me/Im_Ilya) свою обратную связь и шарьте, если узнали и согласны 😄. 

> **DISCLAIMER** Я не хочу никого взорвать, просто говорю свое мнение. Можете думать ,что я не прав, я никому не навязываю, спросите моих коллег 🙂 Местами шутки — просто шутки.  

## Почему я это написал ✍️
Все связано с ревью и тем, что я часто не только пишу код, но и читаю его. Также время от времени данная тема начала подниматься у меня в разговорах с друзьями, поэтому думаю, что ответ надо зафиксировать. Надеюсь это упростит мне работу в будущем, либо поможет кому-то еще.

## Когда стоит использовать интерфейсы? 🙈
Я считаю, что использование интерфейсов всегда оправдано. Если вам говорят, что интерфейсы можно не использовать, то им просто лень 😅.
Далее пройдемся по пунктам, почему это делать стоит, так будет проще.

### Это отделяет ваш контракт от реализации ➗
Звучит просто. Все скажут *Dependency Inversion*, но не только.  
Безусловно — это *DI*, но когда вы не выделяете интерфейс это мешает вам в будущем увидеть, что у сервиса разросся контракт. 

```java
public interface ApplicationMetaDataExtractor {
  MetaData extractFrom(Application application);
  MetaData extractFrom(ApplicationWithAttachments application);
  MetaData extractAllFor(User user);
  // Можно еще много придумать...
}
```

Обычно классы пилят, когда они большие, а за большим кол-вом несвязных методов следят не так часто. Может быть класс из 10 методов, но ни все совсем минорные. В данном случае я считаю, что имеет смысл сделать класс реализацией нескольких интерфейсов. Если у вас будет выделен отдельный интерфейс, то вы увидите, что он начал обретать очертания *God Object*. Следовательно данный пункт упрощает вам рост и в будущем. 

Важно следить не только за размером класса, но и за его контрактом!

### Способствуем проектированию 📈
Я проектирую, я инженер 👨‍💻. Не всегда.

Когда ты делаешь класс и начинаешь в него писать код, ты забываешь о том, за что отвечает класс, а когда уже решил задачу, оказывается поздно об этом думать, вчера уже фича в релизе. Подумай о контракте сразу, это сэкономит тебе время в будущем росте.

Когда ты делаешь быстро, сначала подумай, времени править не будет!

> Я не говорю, что преждевременная оптимизация архитектуры это хорошо, но надо понимать слабые и сильные её места, а также делать её прозрачнее, чтобы дальше было проще.  

### Проще читать код 📖
Мой код — самый читаемый 😎! Неа, у нас всех — неа 😭.

Самый простой пункт. Код написанный с интерфейсами [читается](https://github.com/spring-cloud/spring-cloud-openfeign/blob/master/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/support/PageableSpringEncoder.java) в *WebUI* проще, чем код в одном классе. В *Web* не навигаций, которые есть у тебя в *IDEA*. Спасибо автору за код, я привел ссылочку, можете найти его на *GitHub* и задонатить ему. `@Override` позволяет почти мгновенно вычленить важные методы из класса, что я очень ценю, спасибо, интерфейс!
```java
/**
 * Provides support for encoding spring Pageable via composition.
 *
 * @author Pascal Büttiker
 */
public class PageableSpringEncoder implements Encoder {
	private final Encoder delegate;
	/**
	 * Page index parameter name.
	 */
	private String pageParameter = "page";
	/**
	 * Page size parameter name.
	 */
	private String sizeParameter = "size";
	/**
	 * Sort parameter name.
	 */
	private String sortParameter = "sort";

	/**
	 * Creates a new PageableSpringEncoder with the given delegate for fallback. If no
	 * delegate is provided and this encoder cant handle the request, an EncodeException
	 * is thrown.
	 * @param delegate The optional delegate.
	 */

	public PageableSpringEncoder(Encoder delegate) {
		this.delegate = delegate;
	}

  // ОЧЕНЬ ПОЛЕЗНЫЕ МЕТОДЫ, ЗАЧИТАЛСЯ БЫ!!!

	public void setPageParameter(String pageParameter) {
		this.pageParameter = pageParameter;
	}

	public void setSizeParameter(String sizeParameter) {
		this.sizeParameter = sizeParameter;
	}

	public void setSortParameter(String sortParameter) {
		this.sortParameter = sortParameter;
	}

	@Override // А вот и ты наконец!
	public void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException {
	  // ...
  }

	private void applySort(RequestTemplate template, Sort sort) {
    // ...
	}

	protected boolean supports(Object object) {
     // ...
	}
}
```

Время ⌛ — деньги 💴, спроси у своего менеджера сколько ты им стоишь, поймешь, почему тебе не повышают ЗП 🧧.

### Это заставляет вас думать 🤔
Я и так думаю , я же программист🧑‍💻, не думают на заводе 🏭! Нет, не думаешь.

Этот пункт завершает все прошлые, так как относится ко всем им одновременно, в тоже время ни к одному. Все мы думаем, когда пишем код как лучше решать ту или иную задачу. Без извлечения контракта в интерфейсы мы не думаем **В ПЕРВУЮ** очередь о том, что должен делать класс, а думаем о том, как проблему решить. Безусловно без понимания как решить проблему жить сложно, особенно решать эту проблему, но давайте будем честны перед самими собой. Никто (абсолютно никто) из нас не запускает ракеты в космос. От нас требуется складывать, вычитать, умножать, делить в лучшем случае. Обычно мы работаем с 3 сущностями: `boolean`, `String`, `Number` — сравнивая их между собой по правилам, что придумали не мы. Но еще чаще ты читаешь как кто-то другой сделал это до тебя и проклинаешь как все сложно и запутано. Поэтому я предлагаю сконцентрироваться не на том, чтобы решить *оч* сложную проблему, а на том, чтобы сделать так, чтобы решение было легко воспринимать, эта задача **НАМНОГО** сложнее `if-else` условий, что ты так ценишь и принесет больше денег заказчику, сэкономив на времени таких, как ты, которые будут в этом разбираться потом.

Возлюби ближнего своего!

### Само собой разумеется 
Я не буду упоминать, что когда мне говорят что код расширяем, а в нем из интерфейсов только *Spring Data* — то это смешно.

Ha-ha, classic!

### Я не считаю AoP причиной
Можно долго говорить, но [тут](https://habr.com/en/post/347752/) почитайте, если интересно. Автор молодец, сделал то, что многим лень.
> Я получил ответ на свой вопрос, а также сделал отметку в памяти, что (как минимум при использовании Spring 5), несмотря на утверждения документации, прокси объекты с большей вероятностью будут создаваться с помощью CGLib.  

Это пригодится тебе на интервью!

## Заключение
Спасибо, что дочитали мой, довольно токсичный текст, надеюсь никого не обиде, пишете мне ЛС , я могу долго об этом говорить, наверное это значит, что тема не имеет нормального заключения. Спасибо за внимание! 
