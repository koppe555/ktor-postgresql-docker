# ktor_website

## フォルダ構成

### build.gradle.kts
→依存関係が記されているファイル

### main/resources
→設定ファイルが入っているディレクトリ

### main/kotlin
→ソースコードが入っているディレクトリ


使用しているプラグイン
- plugins/Routing.kt
- plugins/Templating.kt


### main/resouces/static
static配下にあるファイルはktorが自動でマッピングしてくれる。

```
fun Application.configureRouting() {

    routing {
        static("/static") {
            resources("files")
        }
    }
}

```


## databeの処理の流れ

Application.ktでデータベースを起動。
```
fun Application.module() {
    DatabaseFactory.init(environment.config)
}
```

initの引数のenviroment.configは `main/resource/application.conf`


```
fun init(config: ApplicationConfig) {
    val driverClassName = config.property("storage.driverClassName").getString()
    val jdbcURL = config.property("storage.jdbcURL").getString() +
            (config.propertyOrNull("storage.dbFilePath")?.getString()?.let {
                File(it).canonicalFile.absolutePath
            } ?: "")
    val database = Database.connect(createHikariDataSource(url = jdbcURL, driver = driverClassName))
    transaction(database) {
        SchemaUtils.create(Articles)
    }
}
```






```
storage {
    driverClassName = "org.h2.Driver"
    jdbcURL = "jdbc:h2:file:"
    dbFilePath = build/db
    ehcacheFilePath = build/ehcache
}

```





## FreeMaker

JAVA系のテンプレートエンジン。pluginで追加している。

plugins/Templating.kt

```
fun Application.configureTemplating() {
    install(FreeMarker) {
        templateLoader = ClassTemplateLoader(this::class.java.classLoader, "templates")
        outputFormat = HTMLOutputFormat.INSTANCE
    }
    routing {
    }
}
```


## exposed
ORマッパー。JetbrainsがKotlinで作ってる。


## DAO
デザインパターンの一つ。データベースの操作に関する部分をビジネスロジックから切り離す。


kotlin/com/example/DatabaseFactory.kt
```
package com.example.dao

import com.example.models.*
import kotlinx.coroutines.*
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.*
import org.jetbrains.exposed.sql.transactions.experimental.*

object DatabaseFactory {
    fun init() {
        val driverClassName = "org.h2.Driver"
        val jdbcURL = "jdbc:h2:file:./build/db"
        val database = Database.connect(jdbcURL, driverClassName)
    }
}

```

driverClassNameとjbdcURLをハードコーディングしてるが設定ファイルに記述することも可能
https://ktor.io/docs/configurations.html#configuration-file





### Facade
ファサードと読む。Javaのデザインパターンの一つ。建物を正面から見た窓口という意味。
その名の通り、連続する一連の処理を窓口を提供するパターン。

一連の処理で複雑なことをしていても、Facadeパターンを使う側は呼び出し口だけしか見えていなく、中の処理を気にしなくてよい。

このリポジトリの実装だと、DBへのアクセスの窓口をinterfaceで実装し、実際にDBにアクセスする具体的な処理はImpl内部に記述している。

```
package com.example.dao

import com.example.models.*

interface DAOFacade {
    suspend fun allArticles(): List<Article>
    suspend fun article(id: Int): Article?
    suspend fun addNewArticle(title: String, body: String): Article?
    suspend fun editArticle(id: Int, title: String, body: String): Boolean
    suspend fun deleteArticle(id: Int): Boolean
}
```

ここで定義したDAOFacadeを `DAOFacadeImpe` と `DAOFacadeCacheImpl` で継承してる。


https://geechs-job.com/tips/details/42


### H2databse

H2と呼ばれているDB。javaで作られている。
https://github.com/h2database/h2database

めちゃくちゃ手軽でテストDBとして用いられることが多い。


DatabaseFactory.ktで定義されてる。
```
object DatabaseFactory {
    fun init() {
        val driverClassName = "org.h2.Driver"
        val jdbcURL = "jdbc:h2:file:./build/db"
        val database = Database.connect(jdbcURL, driverClassName)
    }
}
```

### HikariCP
コネクションプールライブラリ。

動作が安定していてjava系のライブラリの中で一番早いらしい。
https://openstandia.jp/oss_info/hikaricp/#:~:text=HikariCP%EF%BC%88%E3%83%92%E3%82%AB%E3%83%AA%E3%82%B7%E3%83%BC%E3%83%94%E3%83%BC%EF%BC%89%E3%81%AF%E3%80%81,%E6%9C%89%E5%8A%B9%E3%81%AA%E3%82%BD%E3%83%AA%E3%83%A5%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A8%E3%81%AA%E3%82%8A%E3%81%BE%E3%81%99%E3%80%82

### コネクションプール
DBとデータのやり取りをするときのコネクションをプールしておくこと。プールしたコネクションを繰り返し使うことで、切断、再接続の負荷を軽減する。



`DatabaseFactory.kt`
データベースを立ち上げるときに `createHikariDataSource` 経由にすることでコネクションをプールできる。

```

object DatabaseFactory {
    fun init(config: ApplicationConfig) {
        val driverClassName = config.property("storage.driverClassName").getString()
        val jdbcURL = config.property("storage.jdbcURL").getString() +
                (config.propertyOrNull("storage.dbFilePath")?.getString()?.let {
                    File(it).canonicalFile.absolutePath
                } ?: "")
        val database = Database.connect(createHikariDataSource(url = jdbcURL, driver = driverClassName))
        transaction(database) {
            SchemaUtils.create(Articles)
        }
    }

    private fun createHikariDataSource(
        url: String,
        driver: String
    ) = HikariDataSource(HikariConfig().apply {
        driverClassName = driver
        jdbcUrl = url
        maximumPoolSize = 3
        isAutoCommit = false
        transactionIsolation = "TRANSACTION_REPEATABLE_READ"
        validate()
    })
```

`maximumPoolSize = 3`
確保最大プール数を指定。

`transactionIsolation`
分離レベルの設定
>TRANSACTION_REPEATABLE_READ - このレベルは、トランザクションが読み込むデータが他のトランザクションによって修正されず、読込トランザクションがデータを修正してコミットしない限り変更されないことを保証します。


### Ehcache
javaのライブラリ。キャッシュ機能を提供する。



Routing.ktでdaoを定義、`DAOFacadeCacheImpl` → `DAOFacadeImpl` の順番でアクセスして、キャッシュ上に存在してなかったら `DAOFacadeImpl` を通してクエリが実行される。クエリ実行後は結果をキャッシュする。


```
    val dao: DAOFacade = DAOFacadeCacheImpl(
        DAOFacadeImpl(),
        File(environment.config.property("storage.ehcacheFilePath").getString())
    ).apply {
        runBlocking {
            if(allArticles().isEmpty()) {
                addNewArticle("The drive to develop!", "...it's what keeps me going.")
            }
        }
    }
```

最初に一度 `allArticles` を呼んでキャッシュしてる。


`DAOFacadeCacheImpl`
```
override suspend fun article(id: Int): Article? {
    return articlesCache[id]
        ?: delegate.article(id)
            .also { article -> articlesCache.put(id, article) }
}
```

`DAOFacadeImpl`
```
override suspend fun article(id: Int): Article? = dbQuery {
    Articles
        .select { Articles.id eq id }
        .map(::resultRowToArticle)
        .singleOrNull()
}
```
キャッシュ問い合わせ、なければDB問い合わせ、結果を保持。
# ktor-postgresql-docker
