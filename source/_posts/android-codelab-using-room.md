---
title: [Android Codelabs] 使用Room
date: 2021-10-17 01:00:31
tags:
- Android
- 数据持久化
- Jetpack
---

[Room](https://developer.android.com/jetpack/androidx/releases/room)是[Jetpack](https://developer.android.com/jetpack)的成员之一，是一个数据持久化的组件，作用跟`SQLite`类似，但提供了更易于使用、便捷开发的编程逻辑。



本文将通过以下codelabs学习和使用Room。

* [Introduction to Room and Flow](https://developer.android.com/codelabs/basic-android-kotlin-training-intro-room-flow)
* [Persist data with Room](https://developer.android.com/codelabs/basic-android-kotlin-training-persisting-data-room)
* [Android Kotlin Fundamentals: Create a Room Database](https://developer.android.com/codelabs/kotlin-android-training-room-database)
* [Android Room with a View - Kotlin](https://developer.android.com/codelabs/android-room-with-a-view-kotlin)



本文不实际探讨关于Room的深入原理。

<!-- more -->

## 整体背景

使用Room的整体结构参照下图：

![Room arch](https://developer.android.google.cn/images/training/data-storage/room_architecture.png)

从图中看到，应用通过Database获取到DAO示例；通过DAO与数据库进行交互，如CRUD。可以看出，对App来说，比较重要的应该就是这个DAO。



那Entities是什么呢？可以理解成是对象。一个对象对应着数据库中的一条数据。



## 使用步骤

总结下Room的使用步骤：

1. 抽象对象，设计对应的Entity类。一般用`data class`。
2. 定义DAO类，定义数据库与App交互的接口。一般用`interface`。
3. 定义Database类，继承自`RoomDatabase`，主要用于提供DAO对象。一般用`abstract class`。



这就是使用Room的最小步骤了。但在项目中使用，肯定是不满足仅仅如此的，通常我们还需要一个“最佳实践”。



## 最佳实践

### 使用`Repository`中间层

在实际的项目中，通常会使用一个`Repository`来作为App与Database中间的桥梁。



考虑以下场景：当需要访问xxx资源时，预期是如果数据被未拉取到本地，或者是远端存在更新，那么希望从远端获取数据并保存到本地。如果本地存在数据，且已是最新，则直接返回。



很容易想到的方法是在业务代码中直接实现，不过完全可以把这部分封装在获取数据的类中。这样业务方调用就很清晰明了了。



## 实例

这里保存书籍信息为例，实例介绍Room的使用。

```kotlin
// 1. 先抽象对象，定义Entity
// 假设我们需要访问书籍对象，获取它的书名、作者、页数、价格信息
// file: Book.kt
@Entity(tableName = "book") // 数据库中的表名
data class Book(
    @PrimaryKey @ColumnInfo(name = "name") val name: String, // 定义主键，在数据库中的名字为name，类型为String
    @ColumnInfo(name = "author") val author: String, // 定义一个键，名为author
    @ColumnInfo(name = "pages") val pages: Int,
    @ColumnInfo(name = "price") val price: Double,
)

// 2. 定义DAO，定义业务与数据库交互的操作
// file: BookDao.kt
@Dao
interface BookDao {
    @Query("SELECT * FROM book") // 这里是用到的SQL语句
    fun getAllBooks(): Flow<List<Book>> // 获取数据库中的所有书，异步操作，是用来了Kotlin Flow

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertBook(vararg book: Book) // 插入书籍，Room自动完成book到数据库条目的转换。onConflict定义了当遇到相同条目时的处理策略

    @Query("DELETE FROM book")
    suspend fun deleteAll() // 删除所有书籍
}

// 3. 定义Database
// file：BookRoomDatabase.kt
@Database(entities = [Book::class], version = 1, exportSchema = false)
abstract class BookRoomDatabase : RoomDatabase() {
    abstract fun bookDao(): BookDao
  
    companion object {
        @Volatile
        private var INSTANCE: BookRoomDatabase? = null
        // 返回单例数据库实例
        // 通常，创建一个新的数据库的开销较大，并且毫无必要创建多个数据库实例。
        // 所以在使用时，最好定义一个接口，仅创建一次。
        fun getDatabase(context: Context, scope: CoroutineScope): BookRoomDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    BookRoomDatabase::class.java,
                    "book_database" // 数据库的名字
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}

// 4. [optional] 定义Repository类（存储库）
// Repository通常以DAO作为其构造参数，在业务实际操作数据的过程中，
// 应通过它提供的接口访问数据（底层对象不一定是数据库，还可以是网络资源）
// file：BookRepository.kt
class BookRepository(private val bookDao: BookDao) {
    val allBooks: Flow<List<Book>> = bookDao.getAllBooks()

    @WorkerThread
    suspend fun insert(book: Book) {
        bookDao.insertBook(book)
    }

    companion object {
        private const val TAG = "BookRepository"
    }
}

// 如何使用
// 数据库通常仅用单例，可以在Application或者单例类中懒加载创建其实例。
// file：MyApplication.kt
class MyApplication: Application() {
    // 数据库实例
    val bookDb: BookRoomDatabase by lazy {
        BookRoomDatabase.getDatabase(this)
    }
    // 存储库实例
    val bookRepository: BookRepository by lazy {
        BookRepository(bookDb.bookDao())
    }
}

// 在MainActivity中，结合ViewModel使用
// 先定义ViewModel
// file：BookViewModel.kt
class BookViewModel(private val repository: BookRepository): ViewModel() {
    val allBooks: LiveData<List<Book>> = repository.allBooks.asLiveData()

    fun insert(book: Book) = viewModelScope.launch {
        repository.insert(book)
    }

    companion object {
        private const val TAG = "BookViewModel"
    }

    // 使用ViewModelProvider提供的Factory来创建ViewModel实例，
    // 避免ViewModel在Activity的生命周期中被重复创建
    class BookViewModelFactory(private val repository: BookRepository): ViewModelProvider.Factory {
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            if (modelClass.isAssignableFrom(BookViewModel::class.java)) {
                return BookViewModel(repository) as T
            }
            throw IllegalArgumentException("Unknown view model class")
        }

    }
}
class MainActivity: AppCompatActivity() {
    // 注意：必须使用viewModels方式创建ViewModel实例，否则会出问题，无法从数据库读or写数据
    private val mBookViewModel: BookViewModel by viewModels {
        BookViewModelFactory((application as MyApplication).bookRepository)
    }
  
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
        // 当数据有变化，并且Activity位于前台时，将会收到Callback
        mBookViewModel.allBooks.observe(this, Observer { books ->
            books?.let {
                it.forEach { book ->
                    Log.d(TAG, "$book")
                }
            }
        })
    }
}
```



