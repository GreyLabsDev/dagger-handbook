# Component Dependencies

Кроме создания Subcomponent, Dagger позволяет связывать друг с другом независимые компоненты и передавать зависимости из одного компонента в другой, достаточно просто указать в целевом компоненте от каких других компонентов он хочет получать объекты. После этого класс компонента будет сгенерирован с учетом того, какие компоненты в нем указаны как dependencies и их нужно будет указать в его билдере. Так же надо явно прописать какие объекты может предоставлять наружу компонент-зависимость.

Отличие от Subcomponent в связи между компонентами:

1. При работе с Subcomponent, созданные сабкомпоненты сами могут найти все необходимые модули и получить объекты из них, все генерится автоматически
2. При работе с Component Dependencies необходимо явно указывать в интерфесе компонента-зависимости, какие объекты он может предоставить, т.к. подключенный к нему таким способом другой компонент ничего не знает о его модулях

```kotlin
@Module
class DatabaseModule {
	@Provides
	fun provideDatabaseInstance() {
		return Database()
	}
}

@Module
class UserApiModule {
	@Provides
	fun provideUserApi() {
		return UserApi()
	}
}

// Создаем модули с зависимостями для нужных Activity,
// зависимости будут предоставляться через новые сабкомпоненты
@Module
class MainModuleOne() {
	@Provides
	provide mainActivityOnePresenter(userApi: UserApi): MainActivityOnePresenter {
		return MainActivityOnePresenter(userApi)
	}
}

// Создаем сабкомпоненты под каждую Activity
@Сomponent(
	modules = [MainModuleOne::class],
	// Указали компонент, откуда хотим получать зависимости
	dependencies = [AppComponent::class]
)
interface ActivityOneComponent {
	fun getMainActivityPresenter(): MainActivityOnePresenter
}

// Основной/корневой компонент
@Component(
	modules = [
		DatabaseModule::class,
		UserApiModule::class,
	]
)
interface AppComponent() {
	// Предоставляем объекты, необходимые для другого компонента
	fun getUserApi(): UserApi
}

class App: Application() {
 
   lateinit var appComponent: AppComponent
 
   override fun onCreate() {
       super.onCreate()
       appComponent = DaggerAppComponent.create()
   }
}

class ConsumerActivityOne(): Acitivity {

	lateinit var presenter: MainActivityOnePresenter

	override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContentView(R.layout.activity_one)
 
			 // Получаем сгенерированный билдер нужного компонента 
			 // и передаем в него компонент, от которого он зависит
			 val activityComponent = ActivityOneComponent.Builder()
																	.appComponent((application as App).appComponent)
																	.build()

			 presenter = activityComponent.getMainActivityPresenter()
  }
}
```

Связь организовать проще, но таком варианте компоненты общаются друг с другом стандартным способом, как если бы один компонент просто предоставлял классу-потребителю нужные объекты