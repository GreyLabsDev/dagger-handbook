# AndroidInjection

Или если лень создавать сабкомпоненты руками

Что почитать - [документация](https://dagger.dev/dev-guide/android.html) (если есть понимание сабкомпонентов и компонентов)

Разделение на сабкомпоненты помогает лучше управлять жизненным циклом зависимостей и разделять компоненты по их зонам ответственности, например

AppComponent

ActivityComponent

ServiceComponent

и т.д.

Dagger 2 позволяет создавать компоненты и сабкомпоненты, привязывая их к нужным нам классам, но все это нужно писать руками, тем более когда они связаны с компонентами системы по типу Acivity/Service, например как выглядит работа с компонентом в Activity

```kotlin
@Module
class UserModule {
	@Provides
	fun provideUserConfig(): UserConfig {
		return UserConfig()
	}
}

// Создаем сабкомпонент MainActivity
@Subcomponent(
	modules = [UserModule::class]
)
interface MainActivityComponent {
	// Создаем inject метод для MainActivity
	fun inject(activity: MainActivity)
	
	// Создаем билдер сабкомпонента для предоставления MainActivity
	@Subcomponent.Builder
  interface Builder {
      @BindsInstance fun activity(activity: MainActivity): Builder
      fun build(): ActivityOneComponent
  }
}

@Component)
interface AppComponent() {
	// Предоставляем билдер для компонента MainActivity
	fun getMainActivityComponentBuilder(): MainActivityComponent.Builder
}

class App: Application() {
 
   lateinit var appComponent: AppComponent
 
   override fun onCreate() {
       super.onCreate()
       appComponent = DaggerAppComponent.create()
   }
}

class MainActivity: Activity() {
  @Inject 
	lateinit var userConfig: UserConfig

  @Override
  public fun onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
		(application as App).appComponent
        .getMainActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this)
  }
}
```

На каждую Acitivity/Fragment будет приходиться достаточно много однотипного и повторяющегося кода, решение дает пакет dagger.android

Последние версии Dagger 2 имеют возможность автоматически создавать самбкомпоненты для платформенных классов

Activity, Fragment, Service, ContentProvide, BroadcastReveiver

Если необходимо иметь контроль сабкомпонента и дополнять его функционалом, то аналогичный пример выше при работе с Android Injection будет выглядеть вот так

```kotlin
@Module
class UserModule {
	@Provides
	fun provideUserConfig(): UserConfig {
		return UserConfig()
	}
}

// Создаем сабкомпонент MainActivity и обозначаем интерфейс фабрики
// Используя AndroidInjector<MainActivity> для компонента 
// и AndroidInjector.Factory<MainActivity> для фабрики
@Subcomponent(
	modules = [UserModule::class]
)
interface MainActivityComponent: AndroidInjector<MainActivity> {	
	@Subcomponent.Factory
  interface Factory: AndroidInjector.Factory<MainActivity> {}
}

// Создаем модуль, который добавит новый сабкомпонент в главный компонент
// и свяжет с ним фабрику сабкомпонента для MainActivity
@Module(subcomponents = MainActivityComponent::class)
abstract class MainActivityModule {
  @Binds
  @IntoMap
  @ClassKey(MainActivity.class)
  abstract fun bindYourAndroidInjectorFactory(
		factory: MainActivityComponent.Factory
	): AndroidInjector.Factory<Any>
}

@Component(
	modules = [MainActivityModule::class]
)
interface AppComponent() {
	// Добавляем inject для Application класса 
	fun inject(app: App)
}

// Чтобы использовть AndroidInjection необходимо обозначить Application класс
// интерфейсом HasAndroidInjector, заинжектить в класс будущую реализацию 
// DispatchingAndroidInjector<Any> (будет сгенерирована Dagger)
// и реализовать метод fun androidInjector(), который должен возвращать
// тот самый DispatchingAndroidInjector<Any>
class App: Application(), HasAndroidInjector {
 
   @Inject
   lateinit var dispatchingAndroidInjector: DispatchingAndroidInjector<Any>
 
   override fun onCreate() {
       super.onCreate()
       AppComponent.create().inject(this)
   }
	
	 override fun androidInjector(): AndroidInjector<Any> {
		 return dispatchingAndroidInjector
	 }
}

// После описания работы с AndroidInjection, достаточно 
// вызвать AndroidInjection.inject(this) в нужной Activity,
// чтобы начать работать с ее компонентом
class MainActivity: Activity() {
  @Inject 
	lateinit var userConfig: UserConfig

  @Override
  public fun onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
		AndroidInjection.inject(this)
  }
}
```

Если в сабкомпонентах нет необходимости управлять ими и настраивать, и достаточно просто создавать их инстансы, то работа с ними будет еще проще:

```kotlin
@Module
class UserModule {
	@Provides
	fun provideUserConfig(): UserConfig {
		return UserConfig()
	}
}

// Создаем модуль, который добавит новый сабкомпонент в главный компонент
// и свяжет с ним фабрику сабкомпонента для MainActivity, просто обозначая 
// метод провайда MainActivity аннотацией @ContributesAndroidInjector
// все остальное будет сгенерировано автоматически
@Module(
	modules = [UserModule::class]
)
interface MainActivityModule {
	@ContributesAndroidInjector
	fun mainAcitivityInjector(): MainActivity
}

@Component(
	modules = [MainActivityModule::class]
)
interface AppComponent() {
	// Добавляем inject для Application класса 
	fun inject(app: App)
}

// Чтобы использовть AndroidInjection необходимо обозначить Application класс
// интерфейсом HasAndroidInjector, заинжектить в класс будущую реализацию 
// DispatchingAndroidInjector<Any> (будет сгенерирована Dagger)
// и реализовать метод fun androidInjector(), который должен возвращать
// тот самый DispatchingAndroidInjector<Any>
class App: Application(), HasAndroidInjector {
 
   @Inject
   lateinit var dispatchingAndroidInjector: DispatchingAndroidInjector<Any>
 
   override fun onCreate() {
       super.onCreate()
       AppComponent.create().inject(this)
   }
	
	 override fun androidInjector(): AndroidInjector<Any> {
		 return dispatchingAndroidInjector
	 }
}

// После описания работы с AndroidInjection, достаточно 
// вызвать AndroidInjection.inject(this) в нужной Activity,
// чтобы начать работать с ее компонентом
class MainActivity: Activity() {
  @Inject 
	lateinit var userConfig: UserConfig

  @Override
  public fun onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
		AndroidInjection.inject(this)
  }
}
```

Аналогичным образом система работает с Fragment

Принцип работы в момент вызова **AndroidInjection.inject(this) - этот метод получает инстанс** DispatchingAndroidInjector из Application класса, дальше идет поиск фабрики сабкомпонента для указанной Activity/Fragment, создает сабкомпонент для указанной  Activity/Fragment и вызывает сгенерированный метод inject(acitivity: MainActivity)

Таким образом сабкомпонент, фабрика сабкомпонента и метод inject для целевого класса создаются автоматически и их не нужно описывать напрямую
