# Subcomponent

## Внимание - Component может быть не один (и не два)

и даже не три, а сколько понадобится

### Разделение на Subcomponent

Для чего - принцип разделения ответственности, более гибкое и осознанное разделение модулей, более гибкое управление временем жизни созданных объектов.

Кратко и по делу:

1. Component является “родительским” объектом по отношению к Subcomponent, который в нем создается
2. Subcomponent имеет доступ ко всем модулям родительского компонента и может использовать их в создании собственных объектов
3. Subcomponent позволяет разделить основной компонент, чтобы избежать его разрастания при увеличении размера кодовой базы
4. Subcomponent может создаваться в любом родительском компоненте, главное чтобы в родительском компоненте были модули с зависимостями, которые нужны для создания объектов в Subcomponent

Например глобально в приложении может быть основной компонент и сабкомпоненты для каждой Activity или Fragment, который будут предоставлять инжект этих классов со всеми необходимыми зависимости. Подобный подход можно реализовать напрямую через сабкомпоненты как в примере ниже либо использовать автоматическую генерацию сабкомпонента с помощью [HasAndroidInjector + @ContributesAndroidInjector](https://dagger.dev/dev-guide/android.html)

Простой пример разделения на Subcomponent (что более громоздко, по сравнению с более современным решением с AndroidInjection)

Есть два ConsumerActivity, которым необходимо создать собственные компоненты, в роли такого класса могут быть любые объекты, где может понадобиться много разных зависимостей. Subcomponent описывается так же как и главный компонент, только для его обозначения используется аннотация **@Subcomponent,** после чего созданный сабкомпонент должен быть предоставлен через родительский компонент.

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

@Module
class MainModuleTwo() {
	@Provides
	provide mainActivitTwoPresenter(database: Database): MainActivityTwoPresenter {
		return MainActivityTwoPresenter(database)
	}
}

// Создаем сабкомпоненты под каждую Activity
@Subcomponent(
	modules = [MainModuleOne::class]
)
interface ActivityOneComponent {
	fun getMainActivityPresenter(): MainActivityOnePresenter
}

@Subcomponent(
	modules = [MainModuleTwo::class]
)
interface ActivityTwoComponent {
	fun getMainActivityPresenter(): MainActivityTwoPresenter
}

// Основной/корневой компонент
@Component(
	modules = [
		DatabaseModule::class,
		UserApiModule::class,
	]
)
interface AppComponent() {
	// Предоставляем нужные сабкомпоненты
	fun getActivityOneComponent(): ActivityOneComponent
	fun getActivityTwoComponent(): ActivityTwoComponent
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
 
			 val activityComponent = (application as App).appComponent.getActivityOneComponent()
			 presenter = activityComponent.getMainActivityPresenter()
  }
}

class ConsumerActivityTwo(): Acitivity {

	lateinit var presenter: MainActivityTwoPresenter

	override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContentView(R.layout.activity_two)
			 
			 val activityComponent = (application as App).appComponent.getActivityTwoComponent()
			 presenter = activityComponent.getMainActivityPresenter()
  }
}
```

В итоге получается вот такое отношение модулей и компонетнов

**AppComponent** - содержит модули, создающие необходимые объекты для сборки конечных классов

**ActivityOneComponent**, **ActivityTwoComponent** - создаются внутри AppComponent, имеют доступ ко всему, что содают модули из AppComponent

**MainModuleOne**, **MainModuleTwo** - находятся внутри сабкомпонентов и используя доступные модули из родительского AppComponent, могут создавать необходимые для Activity презентеры и любые другие классы.

### Subcomponent - Builder/Factory

Аналогично компоненту, сабкомпонент может свой фабричный метод сборки или Builder

Такой подход может пригодиться, когда сабкомпонент, например, необходимо проинициализировать с какими-то входными данными, которые нельзя получить через Dagger или нужная кастомизация для инициализации сабкомпонента в разных конфигурациях

В коде ниже расширим возможности ранее созданного примера

```kotlin

@Module
class MainModuleOne() {
	@Provides
	provide mainActivityOnePresenter(
		userApi: UserApi, context: Context
	): MainActivityOnePresenter {
		return MainActivityOnePresenter(userApi, context)
	}
}

@Subcomponent(
	modules = [MainModuleOne::class]
)
interface ActivityOneComponent {

	@Subcomponent.Builder
  interface Builder {
      @BindsInstance fun activity(context: Context): Builder
      fun build(): ActivityOneComponent
  }

	@Subcomponent.Factory
  interface Factory {
      @BindsInstance fun create(context: Context): MainComponent
  }

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

	fun getActivityOneComponentBuilder(): ActivityOneComponent.Builder
	fun getActivityOneComponentFactory(): ActivityOneComponent.Factory
}

class ConsumerActivityOne(): Acitivity {

	lateinit var presenter: MainActivityOnePresenter

	override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContentView(R.layout.activity_one)
 
			 // Для наглядности оба варианта, сначала Builder, потом Factory
			 val activityComponent = (application as App).appComponent
																										.getActivityOneComponentBuilder()
																										.context(this)
																										.build()

			 val activityComponent = (application as App).appComponent
																										.getActivityOneComponentFactory()
																										.create(this)

			 presenter = activityComponent.getMainActivityPresenter()
  }
}
```

### Бесит все получать через component.get()? Здесь тоже можно использовать Inject!

Порядок действий почти такой же как инжектом других объектов, за исключением того, что нужно явно дать понять главному компоненту из какого сабкомпонента брать билдер или фабрику, для этого нужно создать отдельный модуль главного компонента в котором будет наш сабкомпонент

```kotlin
@Module
class MainModuleOne() {
	@Provides
	provide mainActivityOnePresenter(
		userApi: UserApi, context: Context
	): MainActivityOnePresenter {
		return MainActivityOnePresenter(userApi, context)
	}
}

@Subcomponent(
	modules = [MainModuleOne::class]
)
interface ActivityOneComponent {

	@Subcomponent.Factory
  interface Factory {
      @BindsInstance fun create(context: Context): MainComponent
  }

	fun getMainActivityPresenter(): MainActivityOnePresenter
}

// Тот самый модуль для главного компонента, который 
// позволяет использовть Inject для передачи билдера/фабрики
// нужного сабкомпонента
@Module(
	sumcomponents = [ActivityOneComponent::class]
)
class AcitivityModule() {}

// Основной/корневой компонент
@Component(
	modules = [
		DatabaseModule::class,
		UserApiModule::class,
		AcitivityModule::class
	]
)
interface AppComponent() {
	
	fun injectConsumerActivity(activity: ConsumerActivityOne)
}

class ConsumerActivityOne(): Acitivity {
	
	@Inject
	lateinit var activityComponentFactory: ActivityOneComponent.Factory
 
	lateinit var presenter: MainActivityOnePresenter

	override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContentView(R.layout.activity_one)
			
			 (application as App).appComponent.injectConsumerActivity(this)
 
			 // Для наглядности оба варианта, сначала Builder, потом Factory
			 val activityComponent = activityComponentFactory.create(this)

			 presenter = activityComponent.getMainActivityPresenter()
  }
}
```

Кроме удобства @inject + SubcomponentModule полезны тем, что это помогает избежать лишней кодогенерации, т.к. если прописывать get() напрямую, то Dagger всегда будет генерировать класс сабкомпонента, если же работать с @Inject , то классы сабкомпонентов будут создаваться только если используются дальше в коде