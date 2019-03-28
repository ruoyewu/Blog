---
title: Android 数据持久化
date: 2018-05-16 23:00
tags:
	- android
---

对于计算机上的数据持久化而言，归根结底就是将数据保存在磁盘上，使得即使在断电的情况下数据仍然能够留存，不同的是使用不同存储方式时数据存储结构的不同，在 Android 开发中常用的数据持久存储方式大致有以下五个：

1.  SharedPreferences
2.  SQlite
3.  ContentProvider
4.  Socket
5.  文件直接存取

### SharedPreferences

SharedPreferences 是 Android 中一种比较轻量级的数据存储方式，其本质是通过 键-值 的方式将开发者要存储的数据保存在 xml 文件中，不过为了提高频繁存取的效率，SP 内部使用了内存缓存+硬盘存储的方式，SP 本身是一个单例，通过`Context.getSharePreferences()`获取，当获取到实例之后，SP 会首先将文件中的数据加载到内存，如果在这个过程中操作数据的话会被阻塞，完了之后所有的读取操作都是直接从内存读的，写操作有两步，分别将结果写会内存和硬盘，其中写会内存都是同步执行，对于协会磁盘而言，`commit()`是同步写，`apply()`是异步写。具体的可以参考 [Android SharePreferences](../android-sharedpreferences) 。

### SQLite

SQLite 是 Android 中自带的一个关系型数据库，是一个更适合存储大量应用产生数据的工具，如用户的聊天记录、笔记记录等，其基本的特性就是普遍的关系数据库的特性，更够很好的表达整个应用的数据存储结构、数据之间的关系等。在 Android 中使用 SQLite 需要 SQLiteOpenHelper 类，这是一个抽象类，其抽象方法为`onCreate()`和`onUpgrade()`，使用的时候需要继承 SQLiteOpenHelper 并实现这两个方法。

`onCreate()`会在数据库第一次使用的时候调用，主要用于做一些初始化的操作，如创建基本的表等。

`onUpgrade()`会在每次数据库升级的时候调用，数据库的版本号是在构造函数中传入的，当构造函数中的版本号大于当前的数据库版本号的时候，数据库就会自动升级到传入的版本号，并调用`onUpgrade()`方法，一般会在`onUpgrade()`方法中做数据的修改，如增加了表、修改了表的字段等。

在使用的时候需要使用 SQLiteOpenHelper 获得 SQLiteDataBase 对象，有两种方式，`getReadableDatabase()`和`getWritableDatabase()`，用户控制读写权限。

SQLiteDatabase 提供了增删改查四种方式，还提供了直接执行 SQL 语句的方法，以 insert 为例。

对应的方法为`insert()`，参数分别为：

1.  table: String
2.  nullColumnHack: String
3.  values: ContentValues

第一个参数为表名，第二个参数为 values 为空时的默认语句，第三个参数就是增加的行的数据。其具体的实现如下：

```java
public long insert(String table, String nullColumnHack, ContentValues values) {
    try {
        return insertWithOnConflict(table, nullColumnHack, values, CONFLICT_NONE);
    } catch (SQLException e) {
        Log.e(TAG, "Error inserting " + values, e);
        return -1;
    }
}

public long insertWithOnConflict(String table, String nullColumnHack,
        ContentValues initialValues, int conflictAlgorithm) {
    acquireReference();
    try {
        StringBuilder sql = new StringBuilder();
        sql.append("INSERT");
        sql.append(CONFLICT_VALUES[conflictAlgorithm]);
        sql.append(" INTO ");
        sql.append(table);
        sql.append('(');
        Object[] bindArgs = null;
        int size = (initialValues != null && !initialValues.isEmpty())
                ? initialValues.size() : 0;
        if (size > 0) {
            bindArgs = new Object[size];
            int i = 0;
            for (String colName : initialValues.keySet()) {
                sql.append((i > 0) ? "," : "");
                sql.append(colName);
                bindArgs[i++] = initialValues.get(colName);
            }
            sql.append(')');
            sql.append(" VALUES (");
            for (i = 0; i < size; i++) {
                sql.append((i > 0) ? ",?" : "?");
            }
        } else {
            sql.append(nullColumnHack + ") VALUES (NULL");
        }
        sql.append(')');
        SQLiteStatement statement = new SQLiteStatement(this, sql.toString(), bindArgs);
        try {
            return statement.executeInsert();
        } finally {
            statement.close();
        }
    } finally {
        releaseReference();
    }
}
```

可以看到，对于这种非 SQL 语句的方法，其内部还是通过 StringBuilder 将其转化成了 SQL 语句之后，才调用`SQLiteStatement.executeInsert()`完成 insert 操作。而直接使用`execSQL()`方法执行语句的过程如下：

```java
public int executeSql(String sql, Object[] bindArgs) throws SQLException {
    acquireReference();
    try {
        final int statementType = DatabaseUtils.getSqlStatementType(sql);
        if (statementType == DatabaseUtils.STATEMENT_ATTACH) {
            boolean disableWal = false;
            synchronized (mLock) {
                if (!mHasAttachedDbsLocked) {
                    mHasAttachedDbsLocked = true;
                    disableWal = true;
                    mConnectionPoolLocked.disableIdleConnectionHandler();
                }
            }
            if (disableWal) {
                disableWriteAheadLogging();
            }
        }
        try (SQLiteStatement statement = new SQLiteStatement(this, sql, bindArgs)) {
            return statement.executeUpdateDelete();
        } finally {
            // If schema was updated, close non-primary connections, otherwise they might
            // have outdated schema information
            if (statementType == DatabaseUtils.STATEMENT_DDL) {
                mConnectionPoolLocked.closeAvailableNonPrimaryConnectionsAndLogExceptions();
            }
        }
    } finally {
        releaseReference();
    }
}
```

其他的如删、改、查也都提供了两套方法，不过最终都会作为 SQL 语句提交给 SQLiteStatement 。

在数据库的操作中会经常使用到 ContentValus ，这个类是 ArrayMap 的包装类，其内部所有功能基本全由 ArrayMap 实现。

### ContentProvider

ContentProvider 是 Android 中的四大组件之一，是一个专注于数据共享的组件。虽然这里将 CotentProvider 作为数据持久化的一种方式，但是 ContentProvider 只是一个数据接口，它更多的是用于应用间的数据共享，一个应用可以通过 ContentProvider 作为自己的数据接口分享给其他应用，其他应用就可以使用 ContentProvider 操作本应用的数据，但是本应用还是要在 ContentProvider 中完成实际的数据操作，一般常用的方式有使用 SQLite 或者是文件等，也可以包装多种数据存储方案，根据请求方的数据请求指令选择合适的数据返回。

ContentProvider 是一个抽象类，其中的抽象方法有增删改查等，开发者使用的时候需要选择一种确切的数据存储方案，实现这些方法。使用 ContentProvider 的好处一般有俩，第一，能够提供丰富的数据接口，完成应用间的数据共享，第二，数据提供方能够有效的把握数据的控制权，其他应用的所有数据请求都是先提交给数据提供方（调用 ContentProvider 中的方法），然后由数据提供方完成实现的，所以就可以有效避免其他应用的数据污染。

### Socket

利用网络的方式做持久化，基本过程是将需要保存的数据通过网络发送给服务器，由服务器完成具体的数据存储，然后需要数据的时候再通过网络向服务器请求数据。

这种思想其实与上面的 ContentProvider 类似，它们获取/保存数据的时候都没有自己完成实际的工作，而是将这个任务提交给了其他能够完成数据持久化的应用，相比之下，Socket 方法是利用其它主机，ContentProvider 是利用其他应用。

### 文件直接存储

本质上来说，所有的数据持久化最终都要以文件存储的方式存储在磁盘中，包括上述的 SharedPreferences 和 SQLite ，都是以文件存储为基础的，在这个基础上，通过某种存储结构，达到更加方便、快捷的存储数据，通过这些方法，我们专注的是存取数据的时候数据的结构和数据的内容，而不需要关心这些数据到底是如何存储在磁盘上的，到底存储在哪个位置。

而使用文件直接存储的话，就要关心文件的路径，文件存储数据的时候的编码（二进制方式还是字节方式），总而言之要关心的数据变得多了，但是直接使用文件存储也有一定的好处，开发者可以自己控制数据存储的路径、存储的方式，在某些情况下可以完成更高效的数据存储。