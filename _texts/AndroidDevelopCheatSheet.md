---
collection: texts
layout: post
title:  "Android开发备忘录"
date:   2016-12-16 
category: Android
---
积累一些Android开发时遇到的坑点。虽然有些问题比较基础，但是开发调试的时候也会难以察觉，较容易中招。

[String --> null](#stringNull)  
[数据库事务](#databseTransaction)  
[缩放事件](#scaleEvent)  

<h2 id = "stringNull">String --> null</h2>

#### 坑爹事件
开发Android数据库时，封装了sqlite SQL语句。Column的对象构造函数是这样婶儿滴。

```java
public Column(String name, String type, String constraints) {
        content = " " + name + " " + type + constraints;
}
```

然后有些情况会将constraints设置为null。然后SQL语句会出现类似 " stauts INTEGER null" ,这样的情况。

#### 分析
当`String`变量为空时（不是"")。于其他字符串拼接或者打印输出时，会转换成"null"。

```java
String empty = null;
String a = "abc" + null;
```
`a` 的值为 "abcnull"


<h2 id = "databseTransaction">数据库事务</h2>

#### 坑爹事件
开发Android数据库时, 在插入函数中增加了数据库事务语句。当时是这样滴

```java
public long insertTask(TaskBean taskBean) {
    SQLiteDatabase db = dbHelper.getWritableDatabase();

    ContentValues values = new ContentValues();
    values.put(TaskBean.CREATETIME, taskBean.createTimeMs);
    values.put(TaskBean.UPDATETIME, taskBean.updateTimeMs);

    db.beginTransaction();
    long id = db.insertOrThrow(DatabaseHelper.TABLE_TASK, null,values);
    db.endTransaction();

    taskBean.id = id;
    return id;
}
```
然后发现每次id 返回都是1，而且根本就没有数据插入。

#### 分析
` db.beginTransaction()` 开启一个事务，` db.endTransaction()`结束一个事务， **同时检查事务的标志是否为成功，如果程序执行到endTransaction()之前调用了setTransactionSuccessful() 方法设置事务的标志为成功则提交事务，如果没有调用setTransactionSuccessful() 方法则回滚事务。**
所以在结束事务之前需要执行`setTransactionSuccessful()`。

<h2 id = "scaleEvent">缩放事件</h2>

#### 坑爹事件
做一个view的缩放手势检测，然后对其进行缩放。检测到TouchEvent ，但是监听不到缩放手势，源代码如下：

```java
editArea.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        int action = MotionEventCompat.getActionMasked(event);
        mScaleGestureDetector.onTouchEvent(event);
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                break;

            case MotionEvent.ACTION_POINTER_DOWN:
                scaling = true;
                break;

            case MotionEvent.ACTION_UP:
                    if (!mEditFlag && !scaling) {
                        dosomething();
                    }
                }
                scaling = false;

                default:
                    return false;
        }
        return false;
    }
});

```

#### 分析
原因是如果在接收`ACTION_DOWN`的时候返回`false`的话，是不能接收后续的`MotionEvent`。
所以要在接收`ACTION_DOWN`的时候返回`true`。或者在`onInterceptTouchEvent()`中实现事件监听。
