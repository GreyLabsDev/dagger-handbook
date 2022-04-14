# Lifecycle

## Сколько живут компоненты

**Главный и корневой компонент -** который создается на уровне Application - живет столько, сколько живет приложение

**Сабкомпоненты внутри Activity/Fragment** - при создании сабкомпонент, например внутри  Activity, тот будет жить пока в памяти живет сама Activity, которая хранит на него ссылку

Компонент каждый раз создает новы экземпляр сабкомпонента, если Activity была уничтожена, то при повторном открытии получит новый экземпляр своего сабкомпонента

## Используем @Scope

По умолчанию Dagger создает объекты по типу фабрики, при каждом обращении в новом классе будет получен новый инстанс одного и того же объекта, если мы в модуле просто пометили метод создания объекта как @Provides

Но кроме постоянного пересоздания объектов есть и другие варианты управления жизненным циклом зависимостей, что позволяет не только экономить память и вовремя очищать ее, но и наоборот иметь объекты, которые будут жить постоянно и после первого создания во всех классах будет предоставляться один и тот же инстанс нужного объекта.

## @Singleton

Стандартная аннотация, без дополнительных настроек его применение в AppComponent обозначает, что такой объект будет жить, пока не уничтожен Application 

## Singleton + CustomScope

Основная идея такой комбинации - связать время жизни объекта с временем жизни компонента или сабкомпонента.

Чтобы это сделать, необходимо:

1. Создать собсвенную аннотацию с неймингом, отражающим создаваемй scope жизни объекта
2. Пометить созданной аннотацией компонент, который предоставляет объекты
3. Пометить созданной аннотацией создаваемые объекты напрямую через @Inject constructor() или пометить этой аннотацией @Provides методы внутри модулей для привязки времени жизни создаваемых в них объектов к времени жизни компонента

Если все сделано правильно, Dagger работает в режиме проверки наличия ранее созданного инстанса объекта, который был отмечен нужным скоупом, и если тот уже существует, то при предоставлении этого объекта классу-потребителю, он отдаст ранее созданный экземпляр, вместо того, чтобы создавать его заново.

ВАЖНО - Аннотацией кастомного скоупа должны быть помечены как объект, так и компонент, который его предоставляет

```kotlin
// Создаем аннотацию для своего скоупа жизни под сабпомонент ActivityOneComponent
@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ActivityOneScope

@Module
class DatabaseModule {
	
	// Почемаем скоуп внутри модуля
	@ActivityOneScope
	@Provides
	fun provideDatabaseInstance() {
		return Database()
	}
}

// Создаем модули с зависимостями для нужных Activity,
// зависимости будут предоставляться через новые сабкомпоненты
@Module
class MainModuleOne() {
	@Provides
	provide mainActivityOnePresenter(db: Database, userApi: UserApi): MainActivityOnePresenter {
		return MainActivityOnePresenter(db, userApi)
	}
}

@Module
class UserApiModule {
	@Provides
	fun provideUserApi() {
		return UserApi()
	}
}

// Почемаем скоуп у компонента 
@ActivityOneScope
@Subcomponent(
	modules = [MainModuleOne::class, DatabaseModule::class,]
)
interface ActivityOneComponent {
	fun getMainActivityPresenter(): MainActivityOnePresenter
}

// Основной/корневой компонент
@Component(
	modules = [
		UserApiModule::class
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
```

ВАЖНО 2 **-** Есть возможность заставить сабкомпонент создавать обхект со скоупом его родительского компонента, достаточно просто обозначить одним и тем же скоупом родительский компонент и объект, который предоставляется в дочернем, тогда время жизни объекта поменяется и наш сабкомпонент будет обращаться к своему компоненту за инстансом этого объекта

## MULTISCOPE??

Один объект может создаваться в разных скоупах, и его инстансы будут иметь время жизни, равное времени жизни сабкомпонентов с этими скоупами. Для этого в модулях каждого из сабкомпонентов необходимо создать @Provides метод, который будет создавать экземпляр объекта и пометить его нужным скоупом. Грубо говоря - каждый компонент будет создавать объект самостоятельно через свой модуль и управлять временем жизни этого объекта

Допустим есть два сабкомпонента для ServiceActivity и ProfileActivity и нужно создавать один и тот же объект внутри скоупов этих Activity

```kotlin
class DialogManager(activity: Activity) {}

@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ServiceScope

@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ProfileScope

@Module
class ServiceModule {
	@ServiceScope
	@Provide
	fun provideDialogHelper(activity: Activity) {
		return DialogHelper(activity)
	}
}

@ServiceScope
@Subcomponent(
	modules = [ServiceModule::class]
)
interface ServiceComponent {
}

@Module
class ServiceModule {
	@ProfileScope
	@Provide
	fun provideDialogHelper(activity: Activity) {
		return DialogHelper(activity)
	}
}

@ProfileScope
@Subcomponent(
		modules = [ProfileModule::class]
)
interface ProfileComponent {
}
```

Дополнительно есть возможность использовать множественный скоупы есть объект создается через @Inject constructor

Но два скоупа пометить нельзя одновременно, для этого нужно создать еще одну общую аннотацию, за которой можно сделать необходимое разделение, итоговый объект должен иметь только одну обобщающую аннотацию, а сабкомпоненты и обобщающую и свою собственную

```kotlin
@ActivityScope
class DialogManager(activity: Activity) {}

// Обобщающая аннотация
@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ActivityScope

@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ServiceScope

@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ProfileScope

@ActivityScope
@ServiceScope
@Subcomponent(
	modules = [ServiceModule::class]
)
interface ServiceComponent {
}

@ActivityScope
@ProfileScope
@Subcomponent(
		modules = [ProfileModule::class]
)
interface ProfileComponent {
}
```