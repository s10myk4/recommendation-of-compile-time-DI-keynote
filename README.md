
# Compile time DIのススメ

## モチベーション
そもそも凝ったDI手法なんて必要なくて、コンポーネントごとの依存関係を整理したいをシンプルに実現したい

## 目的
Compile time DIありだな くらいに思ってもらえたらいい

## 前提の環境
Play2.6系 scala2.12系

scala2.10、11系
Play2.4, 2.5でも利用可能

## PlayFWにおけるDI
Play 2.6は下記の2つのDI手法をサポートしている

- JSR330に基づいたRuntime DI  
Guiceはサポートしている1つ  

- Compile time DI  

## DIの本来の目的
TODO
インターフェイスを使ってコードを書いておくと後で定義した実装に沿ってプログラムが動きます

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

## Compile Time DIの候補
####- Thin Cake Pattern  
標準の言語機能だけで実現できるが、  
マニュアルで依存関係を定義するの辛い、ボイラープレートが増える

####- Reader Monad
TODO Top of constructorsでDIする手法ではないので今回は話さない

####- macroを使ったautowiring tool
MacWire

## 上記の問題を解決しつつ、コンポーネントごとの依存関係を簡潔に整理するようなDIがしたいのう
### MacWireを使って依存関係を自動配線する
- MacWireとは？  

TODO: ある程度どのようなscala APIを利用して依存関係を解決しているのかを説明したい
- wireはどのように動いているのか
http://di-in-scala.github.io/ (How wire works)

scala.Macros を使ってIFと実装の関連付けを自動で解決してくれる

簡単な例

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

### MacWireを使うとどれくらい既存のコードに依存を加えずにDIできるかをサンプルコードを使って示す
  - アノテーション書かなくてもMacro?が依存関係を解決してくれるよ
  - DILifeCycle何とかに依存してないから、DIを意識せずに単体テストがかける
  - Compile timeなので穏やかな気持ちで開発できる(いざ動かしたらDIのRuntime Error)



## Guice DIの主要な機能とmacWireの比較

### ある型に対して1つの実装を関連づける

- Guice (Linked binding)

```
bind(classOf[UserService]).to(classOf[UserServiceImpl])
```
- macwire  

```
lazy userSrvice: UserService = wire[UserServiceImpl]
```

### ある型に具体的なインスタンスを生成し、バインドする
- Guice ->  Instance Bindings or Provides Methods or ProviderBindings
https://qiita.com/saka1_p/items/8dfdeccce1f856214bae#providerbindings

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

- macwire -> Factory Method

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

### 型パラメータ使ったDI
TODO かなり雰囲気

- Guice -> TypeLiteral

```
bind(new TypeLiteral[UserRepository[JDBCCotext]]).to(classOf[UserRepositoryOnJDBC])
```

- macwire

```
lazy val repository:UserRepository[JDBCCotext] = wire[UserRepositoryOnJDBC]
```

### Scope
- Guice
デフォルトでは値を要求される度に、新しいインスタンス作成  
`@Singleton`, `@SessionScoped`, `@RequestScoped`などで  
インスタンスのライフタイムを変更できる

- macwire  
`lazy val`で宣言 -> シングルトン  
`def` -> 要求される度に新しいインスタンスを作成して返す  
`Scope trait`を使って requestやsessionの様な独自のscopeを定義できる
 
 
TODO これ必要ないかも
### IFに対して複数の実装がある場合

1.  1つのモジュール内で、コンディショナルな条件によって実装を切り替える

```
trait DBModule {
  ...

  lazy val userDao: UserRepository = {
    if (config.isTest) wire[UserRepositoryOnMemory]
    else wire[UserRepositoryOnJDBC]
  }

  def config: Config
}
```

 
### Testing

基本的にはコンストラクタDIなので、
依存しているIFの実装はコンストラクタから簡単に切り替えられる

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


### ApplicationLoaderと付き合う

#### ApplicationLoaderとは?
(与えられたコンテキストの)アプリケーションをインスタンス化する役割  
アプリケーションが動くのに必要な全てのコンポーネントが、依存の関係を解決してインスタンス化される  
PlayのDevモードでは、ApplicationLoaderは一度インスタンス化され、アプリケーションが再ロードされるたびに1回呼び出されます。  

Guiceの場合は、GuiceApplicationBuilderっていうヘルパーが提供されているので特に意識しなくて済んだ

ApplicationLoader内で コンポーネントの依存関係を解決できるようにしてあげる必要がある

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
基本 JSR330に基づいたRuntime DIをサポートしているものが多い

- play-wsの例   
WSClientのインスタンスを作るための Component を用意してくれている  
もしなければHogeProviderにInjectされている依存を見て Componentを作ればいい

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
) { ...
}
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

### 懸念点
コンパイルタイムにどれくらい影響があるか($お金で解決する)
エクスペリメンタルな機能(macros)に依存している


#### その他の機能
- Tagged type
- using implicit parameters
- Qualifiers(Tagged type)
- Akka integration  


...etc

## まとめ

TODO 大方やりたいことに対して十分な機能があること + メリット + 懸念点

macwireでの compile time DIも懸念点はあるものの、
Thin cake patternのデメリットであった、ボイラープレートを多く必要とせずに compile time DIを実現できるのは良い
apiも充実しており、使い方もシンプルである
runtimeでのreflectionが必要ないので、僅かかもしれないが起動時間が短くなる
プレーンなScalaによって実現できる(no annotations)


DIの手法にとらわれずに、シンプルにコンポーネントごとの依存がちいさくなる様な設計に注力しましょう。


