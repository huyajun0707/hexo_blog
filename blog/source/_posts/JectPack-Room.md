---
title: JectPack之Room
tags: 
  - jectpack
categories:
  - [jectpack] 
top_image: https://rmt.dogedoge.com/fetch/fluid/storage/bg/dojm2h.png?w=1920&q=100&fmt=webp
excerpt: 使用方法类似于greenDao，默认支持子线程操作，如果需要在主线程中处理，可进行相应设置，可以与liveData，一同使用。默认情况下使用了SQLite3.7的新特性.也就是预写日志(WAL). 
---
使用方法类似于greenDao，默认支持子线程操作，如果需要在主线程中处理，可进行相应设置，可以与liveData，一同使用。
注意：默认情况下使用了SQLite3.7的新特性.也就是预写日志(WAL). 在使用可视化工具查看表内容时需要将.db|.db-wal|.db-shm同时导出。

[![ta8yrV.jpg](https://s1.ax1x.com/2020/06/03/ta8yrV.jpg)](https://imgchr.com/i/ta8yrV)

# 使用
## 添加依赖
**java**

```bash
//添加依赖
// support
implementation "android.arch.persistence.room:runtime:1.0.0"
annotationProcessor "android.arch.persistence.room:compiler:1.0.0"
```

**kotlin**

```bash
//kotlin
def room_version = "2.2.0"
implementation "androidx.room:room-runtime:$room_version"
kapt "androidx.room:room-compiler:$room_version"
// optional - Kotlin Extensions and Coroutines support for Room
implementation "androidx.room:room-ktx:$room_version"
```

##  创建实体类

```bash
@Entity(tableName = "tb_student",//表名
                indices = @Index(value = {"name"}, unique = true))
//        ,//定义索引
//        foreignKeys = {@ForeignKey(entity = ClassEntity.class,
//        parentColumns = "id",
//        childColumns = "class_id")})//定义外键
public class StudentEntity {
    @PrimaryKey(autoGenerate = true)//定义主键
    private long id;
    @ColumnInfo(name = "name")
    private String name;
    @Ignore
    private String ignoreText;
    @ColumnInfo(name = "class_id")
    private String class_id;
    @ColumnInfo(name = "sex")
    private int sex;
}
```

### @Entity
默认情况下，使用类名作为数据库表名，也可通过tableName来指定

**indices**定义索引，unique = true 表示表内该字段为唯一

foreignKeys 定义外键关联<br>
- entity:指定外键类名<br>
- parentColumns：指定外键属字段<br>
- childColumns:指定<br>
- onDelete = ForeignKey.CASCADE  <br>

### @PrimaryKey
为主键  autoGenerate表示为自增
### @ColumnInfo(name ="")
设置对应数据库的字段名，默认为字段名
### @Ignore 
创建数据库时忽略该字段

除了使用@PrimaryKey作为主键之外，还可以通过在@Entity中指定

```bash
Entity(primaryKeys = {"name"})
public class StudentEntity {
  
    @ColumnInfo(name = "name")
    private String name;
    @Ignore
    private String ignoreText;
    @ColumnInfo(name = "class_id")
    private String class_id;
    @ColumnInfo(name = "sex")
    private int sex;
}
```

### @Embedded 
对象嵌套,可以将对象分解为表中的一个字段，嵌入式字段还可以包含其他嵌入式字段，也可以通过设置前缀来保持唯一。

```bash
public class StudentEntity {
    @PrimaryKey(autoGenerate = true)//定义主键
    private long id;
    private String name;
    @Ignore
    private String ignoreText;
    private String class_id;
    private int sex;

    @Embedded(prefix="pre_")//这样该表中多了一个pre_post_code的字段
    public Address address;
}

public class Address{
    public String street; 
    public String state; 
    public String city; 
    @ColumnInfo(name = "post_code") 
    public int postCode;    
}
```
## 定义Dao类

```bash
@Dao
public interface StudentDao {

    @Query("SELECT * FROM tb_student")
    List<StudentEntity> getAll();

    @Query("SELECT * FROM tb_student WHERE id IN (:ids)")
    List<StudentEntity> getAllByIds(long[] ids);

    @Insert
    void insert(StudentEntity... entities);

    @Delete  //可以返回一个int值，表示从数据库中删除的行数
    void delete(StudentEntity entity);

    @Update
    void update(StudentEntity entity);

}

//insert与update可以执行事务操作，可以定义处理事务冲突的操作
public @interface Insert {
   
    @OnConflictStrategy  //默认为策略冲突就退出事务 
    int onConflict() default OnConflictStrategy.ABORT;
}
public @interface OnConflictStrategy { 
    //策略冲突就替换旧数据 
    int REPLACE = 1; 
    //策略冲突就回滚事务 
    int ROLLBACK = 2; 
    //策略冲突就退出事务 
    int ABORT = 3; 
    //策略冲突就使事务失败 
    int FAIL = 4; 
    //忽略冲突 
    int IGNORE = 5; 
}
```
## 定义数据库
每个注有@Entity的类都表示为一张表，在@Database(entities={Entity.class来指定})


```bash
@Database(entities = {ClassEntity.class,StudentEntity.class},version = 1)
public abstract class RoomDemoDatabase extends RoomDatabase {

    public abstract ClassDao classDao();
    public abstract StudentDao studentDao();

}
//在@Database中

public @interface Database {
      //添加要建表的类
    Class[] entities();
    //指定数据库版本
    int version();
    //可以数据库的结构文件导出到指定目录
    boolean exportSchema() default true;
}

build.gradle中定义
//指定room.schemaLocation生成的文件路径
javaCompileOptions {
    annotationProcessorOptions {
        arguments = ["room.schemaLocation": "$projectDir/schemas".toString()]
    }
}
```

## 数据库实例

```bash
final RoomDemoDatabase database =  Room.databaseBuilder(getApplicationContext(),
        RoomDemoDatabase.class, "database_name.db")//数据库名称
        .addCallback(new RoomDatabase.Callback() {
            //第一次创建数据库时调用，但是在创建所有表之后调用的
            @Override
            public void onCreate(@NonNull SupportSQLiteDatabase db) {
                super.onCreate(db);
            }

            //当数据库被打开时调用
            @Override
            public void onOpen(@NonNull SupportSQLiteDatabase db) {
                super.onOpen(db);
            }
        })
        .allowMainThreadQueries()//允许在主线程查询数据
        .addMigrations()//迁移数据库使用
        .fallbackToDestructiveMigration()//迁移数据库如果发生错误，将会重新创建数据库，而不是发生崩溃
        .build();
```

别外创建数据库实例时还可以设置操作模式

setJournalMode()

```bash
public enum JournalMode {
    //默认值，api>=16 采用WAL模式
    //如果api<16 采用TRUNCATE模式
    AUTOMATIC,

    /**
     * Truncate journal mode.
     */
    TRUNCATE,
    //WAL模式 先缓存数据到wal,当到达一定数据再缓存到db中
    WRITE_AHEAD_LOGGING
}
```

如果是TUNCATE模式

[![ta86bT.md.png](https://s1.ax1x.com/2020/06/03/ta86bT.md.png)](https://imgchr.com/i/ta86bT)

如果是WRITE_AHEAD_LOGGING模式

[![ta8gVU.md.png](https://s1.ax1x.com/2020/06/03/ta8gVU.md.png)](https://imgchr.com/i/ta8gVU)

WAL模式机制

https://www.sqlite.org/wal.html

##  类型转换器类
TypeConverte 可以将类中指定字段的转换成数据库可以保存的类型

```bash
public class Converters {
    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```

# 数据库升级
## 加字段
比如给StudenEntity加一字段

```bash
public class StudentEntity {
    @PrimaryKey(autoGenerate = true)//定义主键
    private long id;
    private String name;
    @Ignore
    private String ignoreText;
    private long class_id;
    private int sex;

    @Embedded
    public Address address;

    private int age;//新添加字段
}
```
## 版本升级
数据库版本加1  version +1 

```bash
@Database(entities = {ClassEntity.class,StudentEntity.class},version = 2,exportSchema = true)
public abstract class RoomDemoDatabase extends RoomDatabase {

    public abstract ClassDao classDao();
    public abstract StudentDao studentDao();

}
```

### 添加迁移配置

```bash
static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE tb_student ADD COLUMN age INTEGER NOT NULL DEFAULT 0");
    }
};

Room.databaseBuilder(getApplicationContext(),
                RoomDemoDatabase.class, "database_name.db")//数据库名称
                .addMigrations(MIGRATION_1_2)//迁移数据库
```

如果没有添加Migration只指定版本号加1，而且调用了fallbackToDestructiveMigration，所有表被丢弃，会丢失所有数据。
SQLITE只支持 alter table ... add column, 不支持 drop column.

ALTER TABLE只能添加新的字段，如果更改字段的类型，需要创建临时表进行迁移。
addMigrations可以添加多个，可以跳跃式升级，Room会一个一个的触发所有migration。

### 输出模式

```bash
defaultConfig { //指定room.schemaLocation生成的文件路径
    javaCompileOptions {
        annotationProcessorOptions {
            arguments = ["room.schemaLocation": "$projectDir/schemas".toString()]
        }
    }
}
```

[![taO3As.md.png](https://s1.ax1x.com/2020/06/03/taO3As.md.png)](https://imgchr.com/i/taO3As)


可以查看到数据库的历次升级情况，在每次数据库的升级过程中，都会导出一个Schema文件，这是一个json文件，里面包含了数据库的所有基本信息。有了这些文件，能知道数据库的历史变更情况

## room的优点：
1. SQL查询在编译时就会验证 - 在编译时检查每个@Query和@Entity等，这就意味着没有任何运行时错误的风险可能会导致应用程序崩溃（并且它不仅检查语法问题，还会检查是否有该表）
2. sql 查询直接关联到 Java 对象。
3. 耗时操作主动要求异步处理。Room 会在执行 db 操作时判断是不是在 UI 线程，Room 会要求放到异步线程去做，否则会直接 crash 掉
4. 基于注解编译时自动生成代码。
5. API 设计符合 Sql 标准。方便扩展进行各种 db 操作

# 扩展
## 与LiveData一起使用


```bash
@Query("select * from tb_book")
fun getBooksByLiveData(): LiveData<Array<Book>>
```


## 支持rxjava2

```bash
dependencies {
    def room_version = "2.2.0"
    implementation "androidx.room:room-rxjava2:$room_version"
}
```

```bash
@Query("select count(*) from tb_book")
fun getTotalCount(): Flowable<Integer>
```

## 支持paging

```bash
dependencies {
    implementation "androidx.paging:paging-runtime-ktx:2.1.0"
}
```

```bash
@Query("SELECT * FROM tb_book")
fun getAllBook(): DataSource.Factory<Int, Book>
//初始化
fun getRefreshLiveData(): LiveData<PagedList<Book>> =
        LivePagedListBuilder(DbDataHelper.getInstance(getApplication()).dbDataBase.bookDao().getAllBook(), PagedList.Config.Builder()
                .setPageSize(PAGE_SIZE)                         //配置分页加载的数量
                .setEnablePlaceholders(ENABLE_PLACEHOLDERS)     // 当item为null是否使用PlaceHolder展示
                .setInitialLoadSizeHint(PAGE_SIZE)              //初始化加载的数量
                .build()).build()
```


### 特点：
无限滚动 分页效果,也可支持网络请求数据的分页，需要自定义义DataSource


> 参考：
> 
> 外键约束
> 
> https://www.jianshu.com/p/e9adae4e2de0?utm_campaign=haruki&utm_content=note&utm_medium=reader_share&utm_source=qq
> 
> paging
> 
> https://juejin.im/post/5cd677d1518825691f48155b