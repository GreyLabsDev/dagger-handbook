# Component

## Внимание - Component может быть сложнее

### Передача внешних объектов в Component

Сам по себе компонент генерируется автоматически, под капотом создает все включенные в него модули и позволяет получить из модулей нужные объекты. Но, возможны случаи, когда мы не можем в модулях создавать объекты чисто без внешних зависимостей, которые эти модули не могут предоставить. Т.е. есть сущности, которые Dagger не может создавать самостоятельно и надо ему эти сущности передавать.

Например когда необходимо создавать объекты, которые требуют компоненты Android, тот же всеми любимый божественный Context, который задействован в множестве кейсов работы Android приложения.

Допустим необходимо избежать использования контекста напрямую, но при этом иметь возможность в любом месте получать строковые ресурсы, для чего есть класс:

```kotlin
class ResourceManager(context: Context) {
	
	fun getStingResource(@ResId id: Int): String {
		return context.resources.getString(id)
	}
}

@Module
class ReosurcesModule {
	@Privdes
	fun provideResourceManager(context: Context): ResourceManager {
		return ResourceManager(context)
	}
}
```

Такой класс не может быть создан Dagger самостоятельно, т.к. использует Context, и даже если подключить созданный модуль, то ничего не будет работать.

Но можно передать контекст в сам модуль:  

```kotlin
@Module
class ResourcesModule(private val context: Context) {
	@Privdes
	fun provideResourceManager(): ResourceManager {
		return ResourceManager(context)
	}
}
```

Остается только понять, как передать контекст с помощью Dagger. Для этого можно использовать Builder и при создании компонента указать нужный модуль, передав ему все необходимые объекты:

```kotlin
class App: Application() {
 
   lateinit var appComponent: AppComponent
 
   override fun onCreate() {
       super.onCreate()
       appComponent = DaggerAppComponent
           .builder()
           .resourcesModule(ResourcesModule(this))
           .build()
   }
}
```

Так же переданные в модуль объекты, могут предоставляться этим модулям для создания других зависимостей, таким образом можно передать подобный объект один раз и продолжать пользоваться им в других модулях. Немного исправленный пример:

```kotlin
@Module
class AndroidModule(private val context: Context) {
	@Privdes
	fun provideResourceManager(): ResourceManager {
		return ResourceManager(context)
	}
	@Privdes
	fun provideContext(): Context {
		return context
	}
}
```

### Более гибкая сборка Component через Builder или Factory

Сборку компонента можно кастомизировать создав собсвенный Builder, он описывается в интерфейсе и Dagger генерирует реализацию. Builder можно дополнять собственными методами, например для сборки модулей. Builder должен иметь описание метод, возвращающий инстанс самого компонента

```kotlin
@Module
class AndroidModule(private val context: Context) {
	@Privdes
	fun provideResourceManager(): ResourceManager {
		return ResourceManager(context)
	}
	@Privdes
	fun provideContext(): Context {
		return context
	}
}

@Component(
	modules = [
		DatabaseModule::class,
		NetworkModule::class
	]
)
interface AppComponent() {
	fun injectConsumerClass(consume: ConsumerClass)
	
	@Component.Builder
  interface ComponentBuilder {
		// Добавляем в компоненты нужный модуль
		fun androidModule(module: AndroidModule): ComponentBuilder
		// Создаем комонент
	  fun buildComponent(): AppComponent
  }
}

class App: Application {
	lateinit var appComponent: AppComponent
 
  override fun onCreate() {
      super.onCreate()
      appComponent = DaggerAppComponent
          .builder()
					.androidModule(this)
          .buildComponent()
   }
}

```

Создавая свой Builder можно передавать в компонент объекты, которые он не может создавать, тем самым можно передавать такие объекты без создания отдельного модуля, для этого используется аннотация **@BindsInstance**

```kotlin
interface AppComponent() {
	fun injectConsumerClass(consume: ConsumerClass)
	
	@Component.Builder
  interface ComponentBuilder {
		// Добавляем в компоненты объект напрямую
		**@BindsInstance**
		fun bindContext(context: Context): ComponentBuilder
		// Добавляем в компоненты нужный модуль
		fun someUselfulModule(module: SomeUsefulModule): ComponentBuilder
		// Создаем комонент
	  fun buildComponent(): AppComponent
  }
}

class App: Application {
	lateinit var appComponent: AppComponent
 
  override fun onCreate() {
      super.onCreate()
      appComponent = DaggerAppComponent
          .builder()
					.bindContext(this)
					.someUselfulModule(SomeUsefulModule())
          .buildComponent()
   }
}
```

Если нет желания наращивать методы в Builder, можно обойтись фабрикой, где все объекты для компонента можно передать в один метод его сборки. Для этого необходимо создать интерфейс такой фабрики, реализация будет сгенерирована при сборке, сам интерфейс помечается аннотацией **@Component.Factory.** Сам метод фабрики должен возвращать инстанс компонента

```kotlin
@Component
interface AppComponent() {
	fun injectConsumerClass(consume: ConsumerClass)
	
	@Component.Factory
  interface ComponentFactory {
		// Создаем фабричный метод для сборки компонента со всеми нужными объектами
		fun createComponent(context: Context, module: SomeUsefulModule): AppComponent
  }
}

class App: Application {
	lateinit var appComponent: AppComponent
 
  override fun onCreate() {
      super.onCreate()
      appComponent = DaggerAppComponent
          .factory()
					.createComponent(this, SomeUsefulModule())
   }
}
```