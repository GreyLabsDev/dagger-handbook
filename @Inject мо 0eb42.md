# @Inject может больше

## Что еще может @Inject

Базовый вариант использования аннотации, это инъекция готовых объектов в класс - потребитель, инъекция в поле класса.

**Автоматическое создание объекта без модуля -** Так же с помощью @Inject Dagger может внедрять созданные им объекты в конструктор класса, если сборка таковых объектов описана в соответствующих модулях и эти модули подключены к компоненту, который поставляет зависимости.

В таком случае нет необходимости создавать модуль, который создает нам инстанс нужного класса, все сгенерируется автоматически

```kotlin
// Помечаем конструктор аннотацией @Inject, 
// прописываем необходимые параметры и Dagger автоматически 
// добавит эти объекты при создании UserApi
class UserApi @Inject constuctor(
	private val httpClient: HttpClient,
	private val networkConfig: NetworkConfig
) {
}

@Module
class NetworkModule {
	@Provides
	fun provideHttpClient(): HttpClient {
		return HttpClient()
	}

	@Provides
	fun provideNetworkConfig(): NetworkConfig {
		return NetworkConfig()
	}
}

@Component(
	modules = [
		DatabaseModule::class,
		NetworkModule::class
	]
)
interface DaggerComponent() {
	fun injectConsumerClass(consume: ConsumerClass)
}

class ConsumerClass() {

	@Inject
	lateinit var userApi: UserApi
	
	init {
		daggerComponent.injectConsumerClass(this)
	}
}
```

Такой вариант подходит для создания объектов с простым конструктором, если мы имеем доступ к конструктору этого объекта (полностью управляем им, а не пользуемся например библиотечным классом, доступ к которому закрыт)

**Внедрение разных реализаций в конструктор -** Дополнительно при работе с **@Inject constucto**r можно указывать свои кастомные Qualifier аннотации, чтобы внедрять разные реализации одного класса

```kotlin
@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class Test

@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class Prod

class UserApi @Inject constuctor(
	private val httpClient: HttpClient,
	@Test private val networkConfig: NetworkConfig
) {
}

@Module
class NetworkModule {
	@Provides
	@Prod
	fun provideProdNetworkConfig(prodSettings: NetSettings): NetworkConfig {
		return NetworkConfig(prodSettings)
	}

	@Provides
	@Test
	fun provideTestNetworkConfig(testSettings: NetSettings): NetworkConfig {
		return NetworkConfig(testSettings)
	}
}
```

**Выполнение нежного метода при создании объекта или инжекте в объект** - если в классе обозначить любой метод аннотацией **@Inject,** то этот метод будет выполнен:

1. После создания объекта, если для него используется **@Inject constuctor**
    
    ```kotlin
    class UserApi @Inject constuctor(
    	private val httpClient: HttpClient,
    	@Test private val networkConfig: NetworkConfig
    ) {
    	
    	// Выполняется после создания класса UserApi
    	@Inject
    	fun setupClient() {
    		httpClient.setup(networkConfig)
    	}
    }
    ```
    
2. После инжекта в объект, если он является потребителем зависимостей
    
    ```kotlin
    class ConsumerClass() {
    
    	@Inject
    	lateinit var userApi: UserApi
    	
    	init {
    		daggerComponent.injectConsumerClass(this)
    	}
    	
    	// Выполняется после injectConsumerClass(this)
    	@Inject
    	fun setupUserApi() {
    		userApi.setupClient()
    	}
    }
    ```
    

### @Inject для конструктора - спасение?

Не всегда и не везде. Для простого графа зависимостей, для простых в создании классов, для классов, которые выполняют простые действия не имеют большой зоны ответственности и много зависимостей - можно использовать.

Но есть очевидная боль - не всегда понятно что и откуда берется/создается для класса с инжектом в конструктор, иногда сложно распутать цепочку зависимостей, если нет конкретного модуля, где класс собран и приходится самому изучать всю иерархию, чтобы понять откуда были взяла объекты, помещенные в его конструктор.

Так же такой способ не работает с классами, закрытыми для модифицирования/расширения, например если это класс внешнего модуля или библиотеки.

Дополнительно - [рассуждения](https://proandroiddev.com/dagger-and-inject-on-constructors-do-or-dont-9d97e7c93f84) на эту тему.