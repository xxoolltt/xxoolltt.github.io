---
collection: texts
layout: post
title:  "Android架构组件"
date:   2017-6-2 
category: Android
---

翻译原文 [Andorid and Architecture](https://android-developers.googleblog.com/2017/05/android-and-architecture.html?utm_source=Android+Weekly&utm_campaign=293483b394-android-weekly-258&utm_medium=email&utm_term=0_4eb677ad19-293483b394-338078437) 

对于在大范围内设备上构建应用，Android 操作系统提供了强力的基础设施。话虽如此，我们还是收到了开发者的反馈，问题类似于复杂的生命周期，以及确实是合适的应用架构以至于难以编写健壮的应用。

我们需要让编写健壮性的应用更加简单，更加有趣。让开发者专注于他们可以进行创新的地方。现在我们将发布一份Android应用架构指导，以及预览一下架构组件。不同于重复造轮子，我们也认可一些流行Android 库的工作。

### Opinions not Prescriptions

我们指导开发Android应用的方法不止一种。我们提供一系列的指导，使你通过与Android独特的交互方式来搭建Android应用的架构。Android框架提供了与操作系统交互的APIs比如说Activities。但这些只是进入应用程序的入口，而不是应用架构的模块。框架组件并不会强制你的把数据模型从UI组件中分离出去，也不会提供一个清晰的方法解决不依赖生命周期进行数据固化的问题。

### 模块

Android 架构组件组合实现了一个健全的应用架构，同时他们又独立的解决了开发者的痛点。组件的第一部分将会帮助你：

- 自动管理activity与fragment声明周期，以此避免内存以及资源的泄漏。
- 将Java数据对象固化到SQLite 数据库中。

### 生命周期组件

新的生命周期组件构建了应用主要部件到生命周期事件的联系，移除了显性依赖路径。

一个典型的Android观察者模型将会在`onStart()`开始观察并且在`onStop`解除观察。这听起来很简单，但有的时候你会有多个异步请求，并各自管理着他们组件的生命周期。很容易忘记某个触发事件。生命周期组件会帮你解决这个问题。

#### Lifecycle ,LifecycleOwner,LifecycleObserver

核心类是`Lifecycle`。它使用一个枚举来表示当前的生命生命周期状态，对应着一个生命周期事件的美剧变量，以此来跟踪所关联的组件。

![生命周期](https://4.bp.blogspot.com/-K_rXa64Hg84/WRziq3GIogI/AAAAAAAAEMc/VCSWhNXQjPk7dtw0nxVpb6XVLSZxZHdxACLcB/s640/Screen%2BShot%2B2017-05-17%2Bat%2B4.53.59%2BPM.png)

`LifecycleOwner`是一个接口，在`getLifeCycle()`方法返回生命周期对象。`LifecycleObserver`是一个类，通过在方法上添加注解来监控组件的生命周期。把他们放在一起，就可以创建一个既可以监控生命周期时间，又能查询当前生命周期状态的`lifecycle-ware`组件，

```java
public class MyObserver implements LifecycleObserver {
  public MyObserver(Lifecycle lifecycle) {
    // Starts lifecycle observation
    lifecycle.addObserver(this);
   ...
  }
  public void startFragmentTransaction() {
     // Queries lifecycle state
     if (lifecycle.getState.isAtLeast(STARTED)) {
        // perform transaction
     }
  }
  
  // Annotated methods called when the associated lifecycle goes through these events
  @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
  public void onResume() {
  }
  @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
  public void onPause() {
  }
}
MyObserver observer = new MyObserver(aLifecycleOwner.getLifecycle());
```

#### LiveData

'LiveData'是一个持有被观察`lifecycle-aware`数据的类。你的UI代码订阅用于改变的底层代码，并且确保观察者：
 
- 当生命周期在激活状态时（`STARTED` or `RESUMED`)，得到数据的更新。
- 当`LifecycleOwner`被销毁时，被移除
- 当`LifecycleOwner`由于配置或者后端启动被重启时，更新到最新数据

这有助于消除多种方式的内存泄漏，通过避免更新被停止的应用来减少崩溃。

`LiveData`可以有多个观察者，每个`LiveData`关联到一个拥有生命周期的对象比如说`Fragmeng`或者`Activity`。

#### ViewModel

ViewModel 是一个帮助类，通过保存了`Activity`或`Fragment`的UI数据来分离视图数据与UI控制逻辑。`ViewModel`会在`Activity/Fragment`存活的时候尽可能的保留，甚至是在配置改变的时候被摧毁并重建。这使得重建`Activity/Fragment`实体时，`ViewModel`的UI数据依然有效。使用`LiveData`包装UI数据存储在`ViewModel`中，将会为数据提供一个可观察的`lifecycle-aware`。`LiveData`掌握着通知方面的事情，通知`ViewModel`确保数据被适当的保存。

#### 数据持久化

使用`Room library` 简化了数据的持久化。`Room`提供了一个对象映射抽象层，运用SQLite的功能来流畅的访问数据库。
框架的核心是提供内建的支持与原生SQL打交道。尽管这些API很强大，但是他们太过于底层，并且需要大量时间来优化使用：

- 使用原生SQL查询时，没有编译时验证。
- 当数据库改变时，需要手动的升级SQL。这样浪费时间，并可能会一起错误。
- 需要写大量的模板代码，将SQL查询 转化为Java数据类。

Room 通过提供一个抽象层来解决这些问题

##### Database ,Entity and Dao
Room中有三个主要的组件：
- Entity 代表数据库中的一个表，在Java数据对象上添加一个注解。
- DAO(Data Access Object) 定义了访问数据的方法，使用注解来将方法绑定到SQL
- Database 是一持有类，使用注解定义entity 的列表和数据库版本。这个类定义里DAO的列表。同时也是与底层数据库链接的主要接口。
![](https://2.bp.blogspot.com/-JOfLef-C7b4/WRzl-wLKqLI/AAAAAAAAEM4/9aHbKY7OHM4j1cTfAQcLHLi4nS5ngOU3QCLcB/s640/Screen%2BShot%2B2017-05-17%2Bat%2B5.08.05%2BPM.png)

使用Room，需要对数据类进行注解，创建一个包含这些实体的数据库，并且定义一个访问SQL以及修改数据的DAO类。
```java
@Entity
public class User {
    @PrimaryKey
    private int uid;
    private String name;
    // Getters and Setters - required for Room
    public int getUid() { return uid; }
    public String getName() { return name; }
    public void setUid(int uid) { this.uid = uid; }
    public void setName(String name) { this.name = name; }
}

@Dao
public interface UserDao {
    @Query("SELECT * FROM user")
    List getAll();
    @Insert
    void insertAll(User... users);
}

@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```
### 应用架构指导

架构组件是被独立设计的，但是组合使用会让应用的架构更加高效。
更多内容请看[Guide to App Architecture](https://developer.android.com/topic/libraries/architecture/guide.html)
