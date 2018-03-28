
# Compile time DIのススメ

## モチベーション
そもそも凝ったDI手法なんて必要なくて、コンポーネントの依存関係の整理したいをシンプルに実現したい

## 目的
特に何も考えずに Guice使ってた人に考える機会になれば嬉しい
Compile time DIもありだなくらいに思ってもらえたらいい  

## PlayのDI事情
JSR330に基づいたRuntime DIと Compile time DIの両方をサポート
デフォルトは Guice
基本は constructor Injection

## DIの本来の目的
インターフェイスを使ってコードを書いておくと後で定義した実装を関連づけてプログラムが動く仕組み

## Runtime DIの利点 (Guice)
- メリット  
ボイラープレートコードを抑える

- デメリット
  - compile時に依存関係の構築を検証できない
  - アノテーションhell
  - 既存コードに依存が増える 


## Manual DIの利点
- メリット  
型安全　(コンパイル時に依存関係が満たされることを検証できる)  
プレーンな Scalaと コンストラクタパラメータを使うだけ  
runtimeでのreflectionが必要ないので、僅かかもしれないが起動時間が短くなる  
柔軟性が高い  
DIコンテナを必要としない  

- デメリット  
コード量/ボイラープレート増える
コンパイル時間が伸びる 

## Compile Time DIの候補

####- Thin Cake Pattern  
標準の言語機能だけで実現できるが、  
マニュアルで依存関係を定義するの辛い、ボイラープレートが増える

####- Reader Monad
Top of constructorsでDIする手法ではないので今回は話さない

####- macroを使ったauto wiring tool
MacWire

## MacWireについて
- MacWireとは？  
Scala Macroを使ってインスタンスを作成するコードを生成してくれる  
具体的な動作については下記参照  
https://github.com/adamw/macwire#how-wiring-works


- 簡単な例

```
import com.softwaremill.macwire._

class UserDao(db: DB) {
  def store(user: User): Unit = ???
}

trait DBModule {
  lazy val db = new DB("localhost", 3306)
  lazy val userDao = wire[UserDao]
}
```

wireしてるところがマクロで下記に展開

```
lazy val userDao = new UserDao(db)
```

## lazy valでの宣言
val で定義するとインスタンスが初期化される前に参照しようとするとnullになるので、  
lazy val で定義するとオンデマンドで適切な初期化順序を自動で計算されるそう


## DIの主要なケースでのmacwireの例
一部 Guiceでの方法と比較しながら、DIでの主要なケースをmacwireでどんな感じで  
実現できるか簡単な例を挙げる


### ある型に対して1つの実装を関連づける

- Guice (Linked binding)

```
bind(classOf[UserService]).to(classOf[UserServiceImpl])
```
- MacWire  

```
lazy userSrvice: UserService = wire[UserServiceImpl]
```

### ある型に具体的なインスタンスを生成し、バインドする
- Guice ->  Instance Bindings or Provides Methods or ProviderBindings  
インスタンス生成の複雑度によって使い分けられる3つの方法が提供されている

Instance Bindings

```
bind(classOf[A]).toInstance(new A)
```

ProviderBindings

```
class UserServiceProvider extends Provider[Service] {
  override def get(): Service = new UserService
}

class ServiceModule extends AbstractModule {
  override protected def configure() {
    bind(classOf[UserService]).toProvider(classOf[UserServiceProvider])
  }
}
```

- MacWire -> Factory Method

```
lazy val a = new A
```

```
class A()
class C(a: A)
object C {
  def create(a: A) = new C(a)
}

trait Module {
  lazy val a = wire[A]
  lazy val c = wireWith(C.create _)
}
```

### 型パラメータを使ったDI
かなり雰囲気

- Guice -> TypeLiteral

```
bind(new TypeLiteral[UserRepository[JDBCCotext]]).to(classOf[UserRepositoryOnJDBC])
```

- MacWire

```
lazy val repository:UserRepository[JDBCCotext] = wire[UserRepositoryOnJDBC]
```

### Scope
- Guice  
デフォルトでは値を要求される度に、新しいインスタンス作成  
`@Singleton`, `@SessionScoped`, `@RequestScoped`などで  
インスタンスのライフタイムを変更できる

- MacWire  
`lazy val`で宣言 -> シングルトン  
`def` -> 要求される度に新しいインスタンスを作成して返す  
`Scope trait`を使って requestやsessionの様な独自のscopeを定義できる

 
### Testing
基本的には constructor Injection なので、簡単にmockをさせる

テスト時にテスト用のDBに切り替えたりできる

```
class UserDao(dbSetting: DB) {
  def insert(): Unit = ???
}

trait DBModule {
  lazy val dbSetting = new DB("localhost", 3306)
  lazy val userDao = wire[UserDao]
}

trait DBModuleMock extends DBModule with MockitoSugar {

  override lazy val dbSetting = mock[DB]
}

class UserDaoSpec extends FlatSpec with DBModuleMock {
  it should "insert" in {
    userDao.insert()
    ...
  }
}
```


### ApplicationLoaderという存在

#### ApplicationLoaderとは?
(アプリケーション自体で構築できないようなコンテキストを与えられて)アプリケーションをインスタンス化する役割  
PlayのDevモードでは、ApplicationLoaderは一度インスタンス化され、アプリケーションが再ロードされるたびに1回呼び出さる    
ApplicationLoader内で コンポーネントの依存関係を解決できるようにしてあげる必要がある

Guiceの場合は、GuiceApplicationBuilderっていうヘルパーが提供されていて  
reference.confに標準で設定されてたので,特に意識しなくて済んだ

```
class CustomApplicationLoader extends ApplicationLoader {

  override def load(context: ApplicationLoader.Context): Application = {
    new Components(context).application
  }

  class Components(context: Context)
      extends BuiltInComponentsFromContext(context)
      with AuthenticatorServiceModule
      with QueryModule
      with CommandModule
      with AhcWSComponents
      with CustomHttpFiltersComponents {

    val routePrefix: String = "/"

    override def router: Router = wire[Routes]

    //setup logger
    LoggerConfigurator(context.environment.classLoader).foreach {
      _.configure(context.environment, context.initialConfiguration, Map.empty)
    }
    
    wire[scalikejdbc.PlayInitializer]
  }

  trait CustomHttpFiltersComponents extends SecurityHeadersComponents with AllowedHostsComponents {

    def httpFilters: Seq[EssentialFilter] = Seq(securityHeadersFilter, allowedHostsFilter)
  }

}
```

#### Playでの開発をサポートするライブラリとの関係
基本 Guiceをサポートしているものが多い

- play-wsの例   
WSClientのインスタンスを作るための Component を用意してくれている  

```
/**
 * AsyncHttpClient WS API implementation components.
 */
trait AhcWSComponents {

  def environment: Environment

  def configuration: Configuration

  def applicationLifecycle: ApplicationLifecycle

  def materializer: Materializer

  def executionContext: ExecutionContext

  lazy val wsClient: WSClient = {
    implicit val mat = materializer
    implicit val ec = executionContext
    val asyncHttpClient = new AsyncHttpClientProvider(environment, configuration, applicationLifecycle).get
    new AhcWSClientProvider(asyncHttpClient).get
  }
}
```

```
class CustomApplicationLoader extends ApplicationLoader {

  override def load(context: ApplicationLoader.Context): Application = {
    new Components(context).application
  }

  class Components(context: Context)
      extends BuiltInComponentsFromContext(context)
      with AhcWSComponents {
    	....  
   }
}
```


- scalikejdbc.PlayInitializerの例

PlayInitializerが必要とする `ApplicationLifecycle`と`Configuration `を解決してあげれば良い

```
PlayInitializer @Inject() (
    lifecycle: ApplicationLifecycle,
    configuration: Configuration
) { ... }
```

`BuiltInComponentsFromContext`というContextから必要なコンポーネントを取り出してくれるヘルパーがあるので簡単

```
abstract class BuiltInComponentsFromContext(context: ApplicationLoader.Context) 
  extends BuiltInComponents {
  
  lazy val environment = context.environment
  lazy val sourceMapper = context.sourceMapper
  lazy val webCommands = context.webCommands
  lazy val applicationLifecycle: ApplicationLifecycle = context.lifecycle
	...
}
```

```
class CustomApplicationLoader extends ApplicationLoader {

  override def load(context: ApplicationLoader.Context): Application = {
    new Components(context).application
  }

  class Components(context: Context)
      extends BuiltInComponentsFromContext(context) {
    	....  
    	wire[scalikejdbc.PlayInitializer]
   }
}

```

### デメリット
コンパイル時間が伸びる  
experimentalな機能(macro)に依存している


#### その他の機能
- using implicit parameters
- Qualifiers(Tagged type)
- Akka integration  

...etc

## まとめ
MacWireでの compile time DIも懸念点はあるものの、
Thin cake patternのデメリットであった、実装やボイラープレートを多く必要とせずに compile time DIを実現できるのはとても良い  
機能も充実しており、使い方もシンプル  

