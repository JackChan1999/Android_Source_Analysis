## SQLite源码解析

## 1. SQLiteOpenHelper
——封装管理数据库的创造和版本管理类

主要封装了数据库的创建和获取的方法，一般继承该类实现onCreate()、onUpdate()方法，在onCreate创建数据库，在onUpdate进行数据库升级操作。其中还有onConfigure()、onDowngrade()、onOpen()方法，将会在下面获取数据库对象分析进行解析

- 数据库的获取：

两个方法：getReadableDatabase()、getWritableDatabase()。需要注意的一点是这两个方法都加锁，是线程安全的。这两个方法最终调用getDatabaseLocked(boolean writable)：

```java
private SQLiteDatabase getDatabaseLocked(boolean writable) {
    if (mDatabase != null) {

        if (!mDatabase.isOpen()) {  // 判断数据库是否已经关闭
            // Darn!  The user closed the database by calling mDatabase.close().
            mDatabase = null;
        } else if (!writable || !mDatabase.isReadOnly()) { //判断数据库是否符合要求，如果数据库可读可写则返回，即!mDatabase.isReadOnly()一直为true
            // The database is already open for business.
            return mDatabase;
        }
    }

    // 正在初始化中
    if (mIsInitializing) {
        throw new IllegalStateException("getDatabase called recursively");
    }

    SQLiteDatabase db = mDatabase;
    try {
        mIsInitializing = true;

        if (db != null) { // 数据库不为null，需要重新开启读写数据库使得符合要求
            if (writable && db.isReadOnly()) {
                db.reopenReadWrite();
            }
        } else if (mName == null) {
            db = SQLiteDatabase.create(null);
        } else {
            try {
                if (DEBUG_STRICT_READONLY && !writable) {
                    final String path = mContext.getDatabasePath(mName).getPath();
                    db = SQLiteDatabase.openDatabase(path, mFactory,
                            SQLiteDatabase.OPEN_READONLY, mErrorHandler);
                } else {
                    // 通过mContext.openOrCreateDatabase创建数据库，其实还是调用SQLiteDatabase.openDatabase(..)创建数据库
                    db = mContext.openOrCreateDatabase(mName, mEnableWriteAheadLogging ?
                            Context.MODE_ENABLE_WRITE_AHEAD_LOGGING : 0,
                            mFactory, mErrorHandler);
                }
            } catch (SQLiteException ex) {
                if (writable) {
                    throw ex;
                }
                Log.e(TAG, "Couldn't open " + mName
                        + " for writing (will try read-only):", ex);
                final String path = mContext.getDatabasePath(mName).getPath();
                db = SQLiteDatabase.openDatabase(path, mFactory,
                        SQLiteDatabase.OPEN_READONLY, mErrorHandler);
            }
        }

        // 调用onConfigure
        onConfigure(db);

        final int version = db.getVersion();
        if (version != mNewVersion) {
            if (db.isReadOnly()) {
                throw new SQLiteException("Can't upgrade read-only database from version " +
                        db.getVersion() + " to " + mNewVersion + ": " + mName);
            }

            db.beginTransaction();
            try {
                // 当第一次创建数据库时DataBase的版本为0，会调用onCreate()方法
                if (version == 0) {
                    onCreate(db);
                } else {  // 判断数据库版本升降级，调用相应方法
                    if (version > mNewVersion) {
                        onDowngrade(db, version, mNewVersion);
                    } else {
                        onUpgrade(db, version, mNewVersion);
                    }
                }
                db.setVersion(mNewVersion);
                db.setTransactionSuccessful();
            } finally {
                db.endTransaction();
            }
        }

        // 调用onOpen()方法
        onOpen(db);

        if (db.isReadOnly()) {
            Log.w(TAG, "Opened " + mName + " in read-only mode");
        }

        mDatabase = db;
        return db;
    } finally {
        mIsInitializing = false;
        // 数据库创建失败时，进行close操作
        if (db != null && db != mDatabase) {
            db.close();
        }
    }
}
```

#### onCreate()、onUpdate()、onConfigure()、onDowngrade()、onOpen()方法的调用规则：

- onConfigure()：

当第一次调用getReadableDatabase()或者getWritableDatabase()会调用onConfigure()，如果第一是获取到只读形式的数据库，当转换成可写形式数据库时会再次调用onConfigure()。

- onCreate()

mDatabase第一次创建时会调用onCreate()

- onUpdate() / onDowngrade()

在版本改变时会调用相应的onUpdate()或onDowngrade()方法，

- onConfigure()

至于onOpen()的调用规则同onConfigure()。

那么onConfigure()和onOpen()方法可以干嘛呢，从api文档可以看到：

- 可以在onConfigure**开启SQLite的WAL模式**，以及**设置外键的支持**；

- 而onOpen()方法是说明数据库已经打开，可以进行一些自己的操作，但是**需要通过SQLiteDatabase#isReadOnly方法检查数据库是否真正打开了**

- 开启WAL方法：setWriteAheadLoggingEnabled(boolean enabled)

WAL支持读写并发，是通过将修改的数据单独写到一个wal文件中，默认在到达一个checkpoint时会将数据合并入主数据库中
至于关于WAL的详细介绍和分析可以参见SQLite3性能深入分析](http://blog.xcodev.com/posts/sqlite3-performance-indeep/)

## 2. SQLiteDatabase

#### open

获取SQLiteDatabase对象，从上面可以看到getReadableDatabase()、getWritableDatabase()是通过SQLiteDatabase.openDatabase(..)创建数据库，那么其中包含那些细节呢？

```java
public static SQLiteDatabase openDatabase(String path, CursorFactory factory, int flags,
        DatabaseErrorHandler errorHandler) {
    SQLiteDatabase db = new SQLiteDatabase(path, flags, factory, errorHandler);
    db.open();
    return db;
}
```

可以看到new一个SQLiteDatabase对象，并调用open()，再返回该数据库对象，先看open()函数：

open():

```java
private void open() {
    try {
        try {
            openInner();
        } catch (SQLiteDatabaseCorruptException ex) {
            onCorruption();
            openInner();
        }
    } catch (SQLiteException ex) {
        // ....
    }
}

private void openInner() {
    synchronized (mLock) {
        assert mConnectionPoolLocked == null;
        mConnectionPoolLocked = SQLiteConnectionPool.open(mConfigurationLocked);
        mCloseGuardLocked.open("close");
    }

    synchronized (sActiveDatabases) {
        sActiveDatabases.put(this, null);
    }
}

// 可以看到调用SQLiteConnectionPool.open(mConfigurationLocked)：
public static SQLiteConnectionPool open(SQLiteDatabaseConfiguration configuration) {
    if (configuration == null) {
        throw new IllegalArgumentException("configuration must not be null.");
    }

    // Create the pool.
    SQLiteConnectionPool pool = new SQLiteConnectionPool(configuration);
    pool.open(); // might throw
    return pool;
}
// 可以看到其中是创建一个SQLiteConnectionPool，并且调用open操作：

// Might throw
private void open() {
    // Open the primary connection.
    // This might throw if the database is corrupt.
    mAvailablePrimaryConnection = openConnectionLocked(mConfiguration,
            true /*primaryConnection*/); // might throw

   // ...
}

// 可以看到创建了主连接mAvailablePrimaryConnection：
private SQLiteConnection openConnectionLocked(SQLiteDatabaseConfiguration configuration,
        boolean primaryConnection) {
    final int connectionId = mNextConnectionId++;
    return SQLiteConnection.open(this, configuration,
            connectionId, primaryConnection); // might throw
}

// 调用了SQLiteConnection.open()创建主连接：
static SQLiteConnection open(SQLiteConnectionPool pool,
        SQLiteDatabaseConfiguration configuration,
        int connectionId, boolean primaryConnection) {
    SQLiteConnection connection = new SQLiteConnection(pool, configuration,
            connectionId, primaryConnection);
    try {
        connection.open();
        return connection;
    } catch (SQLiteException ex) {
        connection.dispose(false);
        throw ex;
    }
}

private void open() {
    mConnectionPtr = nativeOpen(mConfiguration.path, mConfiguration.openFlags,
            mConfiguration.label,
            SQLiteDebug.DEBUG_SQL_STATEMENTS, SQLiteDebug.DEBUG_SQL_TIME);

    setPageSize();
    setForeignKeyModeFromConfiguration();
    setWalModeFromConfiguration();
    setJournalSizeLimit();
    setAutoCheckpointInterval();
    setLocaleFromConfiguration();

    // Register custom functions.
    final int functionCount = mConfiguration.customFunctions.size();
    for (int i = 0; i < functionCount; i++) {
        SQLiteCustomFunction function = mConfiguration.customFunctions.get(i);
        nativeRegisterCustomFunction(mConnectionPtr, function);
    }
}

// 可以看到最终调用了nativeOpen打开一个主数据库连接，并且设置各自sqlite的属性。
```

**创建流程：**
![sqlite_open](http://img.blog.csdn.net/20160604104353792)

可以看出，创建一个数据库对象，会创建一个数据库连接池，并且会创建出一个主连接  

数据库连接池用于管理数据库连接对象

而数据库连接SQLiteConnection则在其中包装了native的sqlite3对象，数据库sql语句最终会通过sqlite3对象执行可以看出，创建一个数据库对象，会创建一个数据库连接池，并且会创建出一个主连接  

数据库连接池用于管理数据库连接对象

而数据库连接SQLiteConnection则在其中包装了native的sqlite3对象，数据库sql语句最终会通过sqlite3对象执行

#### insert

那么接下来就可以对数据库进行一些CRUD操作

先分析一下insert()和insertOrThrow()插入函数：

```java
// 最终会调用insertWithOnConflict
public long insertWithOnConflict(String table, String nullColumnHack,
        ContentValues initialValues, int conflictAlgorithm) {
    acquireReference();
    try {
        StringBuilder sql = new StringBuilder();
        // 构造insert SQL语句

        // 创建SQLiteStatement对象，并调用executeInsert()
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

// SQLiteStatement::executeInsert():
public long executeInsert() {
    acquireReference();
    try {
        return getSession().executeForLastInsertedRowId(
                getSql(), getBindArgs(), getConnectionFlags(), null);
    } catch (SQLiteDatabaseCorruptException ex) {
        onCorruption();
        throw ex;
    } finally {
        releaseReference();
    }
}

// getSession()调用的是mDatabase.getThreadSession()，获取到SQLiteSession对象：

// SQLiteSession::executeForLastInsertedRowId():
public long executeForLastInsertedRowId(String sql, Object[] bindArgs, int connectionFlags,
        CancellationSignal cancellationSignal) {
   // 验证判断

   // 获取一个数据库连接
    acquireConnection(sql, connectionFlags, cancellationSignal); // might throw
    try {
        // 执行sql语句
        return mConnection.executeForLastInsertedRowId(sql, bindArgs,
                cancellationSignal); // might throw
    } finally {
        releaseConnection(); // might throw
    }
}

// SQLiteConnection::executeForLastInsertedRowId():
public long executeForLastInsertedRowId(String sql, Object[] bindArgs,
        CancellationSignal cancellationSignal) {
    if (sql == null) {
        throw new IllegalArgumentException("sql must not be null.");
    }

    final int cookie = mRecentOperations.beginOperation("executeForLastInsertedRowId",
            sql, bindArgs);
    try {
        final PreparedStatement statement = acquirePreparedStatement(sql);
        try {
            throwIfStatementForbidden(statement);
            // 绑定数据参数
            bindArguments(statement, bindArgs);
            applyBlockGuardPolicy(statement);
            attachCancellationSignal(cancellationSignal);
            try {
                // 调用native执行sql语句
                return nativeExecuteForLastInsertedRowId(
                        mConnectionPtr, statement.mStatementPtr);
            } finally {
                detachCancellationSignal(cancellationSignal);
            }
        } finally {
            releasePreparedStatement(statement);
        }
    } catch (RuntimeException ex) {
        mRecentOperations.failOperation(cookie, ex);
        throw ex;
    } finally {
        mRecentOperations.endOperation(cookie);
    }
}
```
**流程图：**
![insert](http://img.blog.csdn.net/20160604105614763)

这里有几个需要注意一下：

- SQLiteSession：
```java
private final ThreadLocal<SQLiteSession> mThreadSession = new ThreadLocal<SQLiteSession>() {
    @Override
    protected SQLiteSession initialValue() {
        return createSession();
    }
};
```

每个线程都拥有自己的SQLiteSession对象。多个线程进行数据操作的时候需要注意和处理保持数据的原子性

- SQLiteStatement

SQLiteStatement类代表一个sql语句，其父类为SQLiteProgram，从上面可以看到，insert操作会先构造出SQLiteStatement，其构造方法：

```java
SQLiteProgram(SQLiteDatabase db, String sql, Object[] bindArgs,
        CancellationSignal cancellationSignalForPrepare) {
    mDatabase = db;
    mSql = sql.trim();

    int n = DatabaseUtils.getSqlStatementType(mSql);
    switch (n) {
        case DatabaseUtils.STATEMENT_BEGIN:
        case DatabaseUtils.STATEMENT_COMMIT:
        case DatabaseUtils.STATEMENT_ABORT:
            mReadOnly = false;
            mColumnNames = EMPTY_STRING_ARRAY;
            mNumParameters = 0;
            break;

        default:
            boolean assumeReadOnly = (n == DatabaseUtils.STATEMENT_SELECT);
            SQLiteStatementInfo info = new SQLiteStatementInfo();
            db.getThreadSession().prepare(mSql,
                    db.getThreadDefaultConnectionFlags(assumeReadOnly),
                    cancellationSignalForPrepare, info);
            mReadOnly = info.readOnly;
            mColumnNames = info.columnNames;
            mNumParameters = info.numParameters;
            break;
    }

    // 参数初始化操作
}
```

可以看到其会调用SQLiteSession::prepare()操作，又是转发到SQLiteConnection::prepare()操作，进行SQL语法预编译，并会返回行列信息到SQLiteStatementInfo中。

再看下插入函数`public long executeForLastInsertedRowId(String sql, Object[] bindArgs,        CancellationSignal cancellationSignal)`通过前面的`SQLiteStatement`将sql语句和参数组成sql并传递进来，通过`PreparedStatement acquirePreparedStatement(String sql)`获取`PreparedStatement`对象，再通过`nativeExecuteForLastInsertedRowId(                          mConnectionPtr, statement.mStatementPtr)`native方法执行sql语句。

在获取`PreparedStatement`的时候，可以看到PreparedStatement通过一个mPreparedStatementCache来进行缓存操作，具体是一个`LruCache<String, PreparedStatement>`来完成sql的缓存

#### replace、delete

同理的操作有replace()、replaceOrThrow、delete、updateupdateWithOnConflict、execSQL等函数。

读者可按照前面思路分析

#### query

现在重点分析一下SQLiteDatabase的查询操作：

从源码可以看出查询操作最终会调用rawQueryWithFactory():

```java

public Cursor rawQueryWithFactory(
        CursorFactory cursorFactory, String sql, String[] selectionArgs,
        String editTable, CancellationSignal cancellationSignal) {
    acquireReference();
    try {
        SQLiteCursorDriver driver = new SQLiteDirectCursorDriver(this, sql, editTable,
                cancellationSignal);
        return driver.query(cursorFactory != null ? cursorFactory : mCursorFactory,
                selectionArgs);
    } finally {
        releaseReference();
    }
}
```

可以看出先构造出SQLiteDirectCursorDriver，再调用其query操作：

```java

// SQLiteDirectCursorDriver::query():
public Cursor query(CursorFactory factory, String[] selectionArgs) {
    final SQLiteQuery query = new SQLiteQuery(mDatabase, mSql, mCancellationSignal);
    final Cursor cursor;
    try {
        query.bindAllArgsAsStrings(selectionArgs);

        if (factory == null) {
            cursor = new SQLiteCursor(this, mEditTable, query);
        } else {
            cursor = factory.newCursor(mDatabase, this, mEditTable, query);
        }
    } catch (RuntimeException ex) {
        query.close();
        throw ex;
    }

    mQuery = query;
    return cursor;
}
```

流程图：
![query](http://img.blog.csdn.net/20160604125914330)

可以看出先构造出SQLiteQuery，在构造出SQLiteCursor，并返回SQLiteCursor对象。

所以得到的Cursor的原型是SQLiteCursor类，你会发现没有其他操作，那么查询数据是在哪里呢？


SQLiteCursor分析：

```java
public final boolean moveToFirst() {
    return moveToPosition(0);
}

public final boolean moveToPosition(int position) {
    // Make sure position isn't past the end of the cursor
    final int count = getCount();
    if (position >= count) {
        mPos = count;
        return false;
    }

    // Make sure position isn't before the beginning of the cursor
    if (position < 0) {
        mPos = -1;
        return false;
    }

    // Check for no-op moves, and skip the rest of the work for them
    if (position == mPos) {
        return true;
    }

    boolean result = onMove(mPos, position);
    if (result == false) {
        mPos = -1;
    } else {
        mPos = position;
    }

    return result;
}

public int getCount() {
    if (mCount == NO_COUNT) {
        fillWindow(0);
    }
    return mCount;
}

private void fillWindow(int requiredPos) {
    clearOrCreateWindow(getDatabase().getPath());

    try {
        if (mCount == NO_COUNT) {
            int startPos = DatabaseUtils.cursorPickFillWindowStartPosition(requiredPos, 0);
            mCount = mQuery.fillWindow(mWindow, startPos, requiredPos, true);
            mCursorWindowCapacity = mWindow.getNumRows();
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "received count(*) from native_fill_window: " + mCount);
            }
        } else {
            int startPos = DatabaseUtils.cursorPickFillWindowStartPosition(requiredPos,
                    mCursorWindowCapacity);
            mQuery.fillWindow(mWindow, startPos, requiredPos, false);
        }
    } catch (RuntimeException ex) {
        // Close the cursor window if the query failed and therefore will
        // not produce any results.  This helps to avoid accidentally leaking
        // the cursor window if the client does not correctly handle exceptions
        // and fails to close the cursor.
        closeWindow();
        throw ex;
    }
}

protected void clearOrCreateWindow(String name) {
    if (mWindow == null) {
        mWindow = new CursorWindow(name);
    } else {
        mWindow.clear();
    }
}
```

到这里你会发现CursorWindow，那这个对象是干嘛的呢？从文档上看可以发现其保存查询数据库的缓存，那么数据是缓存在哪的呢？先看器构造器：

```java
public CursorWindow(String name) {
    // ...
    mWindowPtr = nativeCreate(mName, sCursorWindowSize);

    // ..
}
```

nativeCreate通过JNI调用CursorWindow.cpp的create():

```java
status_t CursorWindow::create(const String8& name, size_t size, CursorWindow** outCursorWindow) {
    String8 ashmemName("CursorWindow: ");
    ashmemName.append(name);

    status_t result;
    // 创建共享内存
    int ashmemFd = ashmem_create_region(ashmemName.string(), size);
    if (ashmemFd < 0) {
        result = -errno;
    } else {
        result = ashmem_set_prot_region(ashmemFd, PROT_READ | PROT_WRITE);
        if (result >= 0) {
            // 内存映射
            void* data = ::mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, ashmemFd, 0);
            // ...
    }
    *outCursorWindow = NULL;
    return result;
}
```

可以看到查询数据是通过创建共享内存来保存的，但是数据在哪里被保存了呢？

继续分析上面SQLiteCursor:: fillWindow()函数：
mQuery.fillWindow(mWindow, startPos, requiredPos, true);

其最终会调用SQLiteConnection::executeForCursorWindow，也是通过JNI调用cpp文件将查询数据保存到共享内存中。

至于共享内存的知识点，可以参考[ Android系统匿名共享内存Ashmem](http://blog.csdn.net/luoshengyang/article/details/6666491)

## 总结

经过上面分析，关于数据库的操作应该有了大致的了解：
![sqlite](http://img.blog.csdn.net/20160604134640551)

当然里面也有些地方也是可以加以改善，取得更好的效率。

而且你会发现SQLiteConnection里面包含许多native方法，通过jni与sqlite进行交互，除了在android提供的sqlite库的基础上优化之外，也可以基于SQLiteConnection，甚至是完全使用c++来实现数据库的封装也是可以的

最后，如果本文有哪些地方不足或者错误，还请指出，谢谢。
