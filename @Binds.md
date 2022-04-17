# @Binds

Что почитать - [вот](https://medium.com/mobile-app-development-publication/dagger-2-binds-vs-provides-cbf3c10511b5), [вот еще](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f), [документация](https://dagger.dev/dev-guide/faq.html) 

Кратко - связывает реализацию и абстракцию, генерирует меньше кода, обычно выглядит так:

```kotlin
interface ServiceApi {
	fun load()
}

class UserApi: ServiceApi {
	override fun load() {}
}

@Module
interface UserModule {
	@Binds
	abstract fun provideUserApi(userApi: UserServiceApi): ServiceApi
}
```

@Binds методы должны быть абстрактными, на вход принимают только один параметр, который должен реализовывать интерфейс, возвращаемый этим методом, не может находиться напрямую в одном интерфейсе модуля вместе с @Provides

Но есть очень хочется совместить @Binds и @Provides, то @Provides методы должны быть статичными:

```kotlin
@Module
interface UserModule {
	@Binds
	abstract fun provideUserApi(userApi: UserServiceApi): ServiceApi

	companion object {
		@Provides
		fun provideUserConfig(configData: ConfigData, userKeys: UserKeys): UserConfig {
			return UserConfig(configData, userKeys)
		}
	}
}
```