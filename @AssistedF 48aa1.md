# @AssistedFactory

Или еще один способ дать Dagger объекты, которые он не может создать сам, при этом не создавая для этого отдельные модули или сабкомпоненты

Кратко и по делу - [здесь](https://dagger.dev/dev-guide/assisted-injection.html)

Полезно использовать для передачи конфига, как пример. Работает так же как @Inject constructor, только дополнительно к этому позволяет добавлять свои параметры к поля конструктора

Для такого параметра, который мы хотим передать - создается интерфейс фабрики с аннотацией @AssistedFactory, в котором описываем методы создания объекта, в параметрах метода указываем только те параметры класса, которые помечены аннотацией @Assisted, после сборки будет сгенерирован класс, и можно обращаться к этому классу передавая в метод фабрики нужные аргументы

Например есть класс для работы с сетью, где необходимо использовать набор заголовков в формате Map<String, String>, и эта конфигурация может быть разных вариантов, не 2, не 3 (Иначе можно было бы использовать Named или свой Qualifier)

@AssistedInject constructor - для конструктора нужного класса

@Assisted - для поля, которое нужно изменять

Для @Assisted параметра можно задавать значение по умолчанию

```kotlin
// Предположим, что Client передается в модуле компонента

class UserApi @AssistedInject constructor(
	client: Client, 
	@Assisted	headers: Map<String,String> = mapOf("is_dager" to true)
) {
}

@AssistedFactory
interface UserApiFactory {
   fun create(headers: Map<String,String>): ServerApi
}

class SomeUserRepository @Inject constructor(
	private val userApiFactory: UserApiFactory
) {

	private lateinit var userApi: UserApi

	fun configApi(headers: Map<String,String>) {
		userApi = userApiFactory.create(headers)
	}
}
```

Если нужно изменять несколько параметров одного типа, то необходимо явно прописать в фабрике для Dagger имена параметров

```kotlin
class UserApi @AssistedInject constructor(
	client: Client, 
	@Assisted	headers: Map<String,String>,
	@Assisted	testHeaders: Map<String,String>
) {
}

@AssistedFactory
interface UserApiFactory {
  fun create(
		@Assisted("headers") headers: Map<String,String>,
		@Assisted("testHeaders") testHeaders: Map<String,String>
	): ServerApi
}

class SomeUserRepository @Inject constructor(
	private val userApiFactory: UserApiFactory
) {

	private lateinit var userApi: UserApi

	fun configApi(
		headers: Map<String,String>,
		testHeaders: Map<String,String>
	) {
		userApi = userApiFactory.create(headers, testHeaders)
	}
}
```