---
collection: texts
layout: post
title:  "Android 版《Effictive Java》"
date:   2016-12-16 
category: Android
---
翻译原文 [Effictive Java for Android](https://medium.com/rocknnull/effective-java-for-android-cheatsheet-bf4e3433889a#.9x1qerwl7) 

《Effictive Java》是一本公认的Java最重要的书籍之一，该书介绍了如何编写长期可维护的代码，同时又兼顾效率。
Android应用是使用Java进行开发，那是否意味着这本书中的建议都可以应用到Android开发中？并未如此。
有些人认为书中的大部分建议无法应用到Android开发世界中。
因为Android没有优化Java的所有特性（比如说：枚举，本地化等）或是手机设备的限制（比如 Dalvik/ART 与桌面JVM的差异），书中的一部分内容无法应用到Android开发中。
尽管如此,书中的大部分范式还是可以经过少许的修改或者直接使用，用以构建更加健康的、清晰的、可维护的代码基础。
  
这篇博客将尝试关注书中那些我认为对于Android开发比较重要的条目。

#### Force non-instantiability
如果你不希望对象被`new`创建，可以使用私有的构造函数。对于那些只有静态方法的工具类特别有用。

```java
class MovieUtils {
    private MovieUtils() {}
    static String titleAndYear(Movie movie) {
        [...]
    }
}
```

#### Static Factories
不使用`new`以及构造函数，而是使用静态工厂方法（以及私有构造函数）。
这些工厂方法被赋予名字，不要求每次都返回一个新的实例，并且可以根据要求来返回不同的子类型。

```java
class Movie {
    [...]
    public static Movie create(String title) {
        return new Movie(title);
    }
}
```

#### Builder
当你有一个对象的构造函数包含3个以上的参数时，使用`Builder`来构建对象。
这样做有些繁琐，但是更容易扩展并且可读性更好。如果你要创建一个数据类（value class), 那么考虑下使用`Auto Value`。

```java
class Movie {
    static Builder newBuilder() {
        return new Builder();
    }
    static class Builder {
        String title;
        Builder withTitle(String title) {
            this.title = title;
            return this;
        }
        Movie build() {
            return new Movie(title);
        }
    }

    private Movie(String title) {
    [...]    
    }
}
// Use like this:
Movie matrix = Movie.newBuilder().withTitle("The Matrix").build();
```

#### Avoid mutability
不可变,是说对象在其整个生命周内保持一致。
对象中所有的必要数据都在创建时提供，这种方式有简洁性，线程安全，可分享(shareability)等优点。

```java
class Movie {
    [...]
    Movie sequel() {
        return Movie.create(this.title + " 2");
    }
}
// Use like this:
Movie toyStory = Movie.create("Toy Story");
Movie toyStory2 = toyStory.sequel();
```
把所有的类都设计成不可变是不可能的。尽管如此，还是要把类尽可能的设计成不可变（比如，`private final`域以及`final` 类）。
手机上创建类会有更多开销，所以要适量。

#### Static member classes
如果你定义的内部类不依赖于外部类，不要忘记把它定义成静态的。
否则的话，内部类的每个实例都会包含一个外部类的引用。

```java
class Movie {
    [...]
    static class MovieAward {
        [...]
    }
}
```

#### Generics(almost)everywhere
我们应该感谢Java提供的类型安全（请参见JavaScript）。
尽可能的避免使用原始类型或类。
泛型提供了在编译时确保类型安全的机制。

```java
// DON'T
List movies = Lists.newArrayList();
movies.add("Hello!");
[...]
String movie = (String) movies.get(0);

// DO
List<String> movies = Lists.newArrayList();
movies.add("Hello!");
[...]
String movie = movies.get(0);
```
不要忘记可是在方法的参数以及返回中使用泛型

```java
// DON'T
List sort(List input) {
    [...]
}

// DO
<T> List<T> sort(List<T> input) {
    [...]
}
```
再复杂一点的情况，你可以使用通配符来扩展类型范围。

```java
// Read stuff from collection - use "extends"
void readList(List<? extends Movie> movieList) {
    for (Movie movie : movieList) {
        System.out.print(movie.getTitle());
        [...]
    }
}

// Write stuff to collection - use "super"
void writeList(List<? super Movie> movieList) {
    movieList.add(Movie.create("Se7en"));
    [...]
}

```

#### Return empty
当不得不返回没有数值的`list`或者`collection`时，不应该返回`null`。
返回一个空集合，使其指向一个比较简单接口（不需要编写文档或者使用非空返回注解功能），这样避免了引发 `NullPointException`。
尽量返回同一个集合，而不是创建一个新的。

```java
// Read stuff from collection - use "extends"
void readList(List<? extends Movie> movieList) {
    for (Movie movie : movieList) {
        System.out.print(movie.getTitle());
        [...]
    }
}

// Write stuff to collection - use "super"
void writeList(List<? super Movie> movieList) {
    movieList.add(Movie.create("Se7en"));
    [...]
}
```

#### No + String 
需要链接少量的`String`时，可以使用`+`。 当需要链接大量的字符串变量时，不要使用，因为性能很差。
这个时候使用`StringBuilder`来代替。

```java
String latestMovieOneLiner(List<Movie> movies) {
    StringBuilder sb = new StringBuilder();
    for (Movie movie : movies) {
        sb.append(movie);
    }
    return sb.toString();
}
```

#### Recoverable exceptions
我个人不喜欢在检查到错误是抛出异常，但是你要这样做的话，要保证异常是可检测的，并且异常的捕获者可以修复这个错误。

```java
String latestMovieOneLiner(List<Movie> movies) {
    StringBuilder sb = new StringBuilder();
    for (Movie movie : movies) {
        sb.append(movie);
    }
    return sb.toString();
}
```

#### 总结
这份列表并不是书中建议的全部列表，也不是简短的深度解读。
这是一份备忘录。
