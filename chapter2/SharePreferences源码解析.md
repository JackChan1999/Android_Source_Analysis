## **1 前言**

在我们开发[Android](http://lib.csdn.net/base/android)过程中数据的存储会有很多种解决方案，譬如常见的文件存储、[数据库](http://lib.csdn.net/base/mysql)存储、网络云存储等，但是Android系统为咱们提供了更加方便的一种数据存储方式，那就是SharePreference数据存储。其实质也就是文件存储，只不过是符合XML标准的文件存储而已，而且其也是Android中比较常用的简易型数据存储解决方案。

我们在这里不仅要探讨SharePreference如何使用，还要探讨其源码是如何实现的；同时还要在下一篇博客讨论由SharePreference衍生出来的Preference相关Android组件实现，不过有意思的是我前几天在网上看见有人对google的Preference有很大争议，有人说他就是鸡肋，丑而不灵活自定义，有人说他是一个标准，很符合设计思想，至于谁说的有道理，我想看完本文和下一篇文章你自然会有自己的观点看法的，还有一点就是关于使用SharePreference耗时问题也是一个争议，分析完再说吧，那就现在开始分析吧（基于API 22源码）。

## **2 SharePreferences基本使用实例**

在Android提供的几种数据存储方式中SharePreference属于轻量级的键值存储方式，以XML文件方式保存数据，通常用来存储一些用户行为开关状态等，也就是说SharePreference一般的存储类型都是一些常见的数据类型（PS：当然也可以存储一些复杂对象，不过需要曲线救国，下面会给出存储复杂对象的解决方案的）。

在我们平时应用开发时或多或少都会用到SharePreference，这里就先给出一个常见的使用实例，具体如下：

```java
public class MainActivity extends ActionBarActivity {
    private SharedPreferences mSharedPreferences;
    private SharedPreferences mSharedPreferencesContext;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initTest();
    }

    private void initTest() {
        mSharedPreferencesContext = getSharedPreferences("Test", MODE_PRIVATE);
        mSharedPreferences = getPreferences(MODE_PRIVATE);

        SharedPreferences.Editor editor = mSharedPreferencesContext.edit();
        editor.putBoolean("saveed", true);
        Set<String> set = new HashSet<>();
        set.add("aaaaa");
        set.add("bbbbbbbb");
        editor.putStringSet("content", set);
        editor.commit();

        SharedPreferences.Editor editorActivity = mSharedPreferences.edit();
        editorActivity.putString("name", "haha");
        editorActivity.commit();
    }
}
```

运行之后adb进入data应用包下的shared_prefs目录可以看见如下结果：

```
-rw-rw---- u0_a84   u0_a84        108 2015-08-23 10:34 MainActivity.xml
-rw-rw---- u0_a84   u0_a84        214 2015-08-23 10:34 Test.xml1212
```

其内容分别如下：

```xml
at Test.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <boolean name="saveed" value="true" />
    <set name="content">
        <string>aaaaa</string>
        <string>bbbbbbbb</string>
    </set>
</map>

at MainActivity.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="name">haha</string>
</map>
```

可以看见SharePreference的使用还是非常简单easy的，所以不做太多的使用说明，我们接下来重点依然是关注其实现原理。

## **3 SharePreferences源码分析**

### **3-1 从SharePreferences接口说起**

其实讲句实话，SharePreference的源码没啥深奥的东东，其实质和ACache类似，都算时比较独立的东东。分析之前我们还是先来看下SharePreference这个类的源码，具体如下：

```java
//你会发现SharedPreferences其实是一个接口而已
public interface SharedPreferences {
    //定义一个用于在数据发生改变时调用的监听回调
    public interface OnSharedPreferenceChangeListener {
        //哪个key对应的值发生变化
        void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
    }

    //编辑SharedPreferences对象设定值的接口
    public interface Editor {
        //一些编辑存储基本数据key-value的接口方法
        Editor putString(String key, String value);
        Editor putStringSet(String key, Set<String> values);
        Editor putInt(String key, int value);
        Editor putLong(String key, long value);
        Editor putFloat(String key, float value);
        Editor putBoolean(String key, boolean value);
        //删除指定key的键值对
        Editor remove(String key);
        //清空所有键值对
        Editor clear();
        //同步的提交到硬件磁盘
        boolean commit();
        //将修改数据原子提交到内存，而后异步提交到硬件磁盘
        void apply();
    }

    //获取指定数据
    Map<String, ?> getAll();
    String getString(String key, String defValue);
    Set<String> getStringSet(String key, Set<String> defValues);
    int getInt(String key, int defValue);
    long getLong(String key, long defValue);
    float getFloat(String key, float defValue);
    boolean getBoolean(String key, boolean defValue);
    boolean contains(String key);

    //针对preferences创建一个新的Editor对象，通过它你可以修改preferences里的数据，并且原子化的将这些数据提交回SharedPreferences对象
    Editor edit();
    //注册一个回调函数，当一个preference发生变化时调用
    void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
    //注销一个之前(注册)的回调函数
    void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
}
```

很明显的可以看见，SharePreference源码其实是很简单的。既然这里说了SharePreference类只是一个接口，那么他一定有自己的实现类的，怎么办呢？我们继续往下看。

### **3-2 SharePreferences实现类SharePreferencesImpl分析**

我们从上面SharePreference的使用入口可以分析，具体可以知道SharePreference的实例获取可以通过两种方式获取，一种是Activity的getPreferences方法，一种是Context的getSharedPreferences方法。所以我们如下先来看下这两个方法的源码。

先来看下Activity的getPreferences方法源码，如下：

```java
    public SharedPreferences getPreferences(int mode) {
        return getSharedPreferences(getLocalClassName(), mode);
    }123123
```

哎？可以发现，其实Activity的SharePreference实例获取方法只是对Context的getSharedPreferences再一次封装而已，使用getPreferences方法获取实例默认生成的xml文件名字是当前activity类名而已。既然这样那我们还是转战Context（其实现在ContextImpl中，至于不清楚Context与ContextImpl及Activity关系的请先看这篇博文，[点我迅速脑补](http://blog.csdn.net/yanbober/article/details/45967639)）的getSharedPreferences方法，具体如下：

```java
//ContextImpl类中的静态Map声明，全局的一个sSharedPrefs
private static ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>> sSharedPrefs;

//获取SharedPreferences实例对象
public SharedPreferences getSharedPreferences(String name, int mode) {
    //SharedPreferences的实现类对象引用声明
    SharedPreferencesImpl sp;
    //通过ContextImpl保证同步操作
    synchronized (ContextImpl.class) {
        if (sSharedPrefs == null) {
            //实例化对象为一个复合Map，key-package，value-map
            sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
        }
    //获取当前应用包名
        final String packageName = getPackageName();
        //通过包名找到与之关联的prefs集合packagePrefs
        ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
        //懒汉模式实例化
        if (packagePrefs == null) {
            //如果没找到就new一个包的prefs，其实就是一个文件名对应一个SharedPreferencesImpl，可以有多个对应，所以用map
            packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
            //以包名为key，实例化的所有文件map作为value添加到sSharedPrefs
            sSharedPrefs.put(packageName, packagePrefs);
        }

        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                //nice处理，name传null时用"null"代替
                name = "null";
            }
        }
    //找出与文件名name关联的sp对象
        sp = packagePrefs.get(name);
        if (sp == null) {
            //如果没找到则先根据name构建一个File的prefsFile对象
            File prefsFile = getSharedPrefsFile(name);
            //依据上面的File对象创建一个SharedPreferencesImpl对象的实例
            sp = new SharedPreferencesImpl(prefsFile, mode);
            //以key-value方式添加到packagePrefs中
            packagePrefs.put(name, sp);
            返回与name相关的SharedPreferencesImpl对象
            return sp;
        }
    }
    //如果不是第一次，则在3.0之前（默认具备该mode）或者mode为MULTI_PROCESS时调用reload方法
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        //重新加载文件数据
        sp.startReloadIfChangedUnexpectedly();
    }
    //返回SharedPreferences实例对象sp
    return sp;
}
```

我们可以发现，上面方法中首先调运了getSharedPrefsFile来获取一个File对象，所以我们继续先来看下这个方法，具体如下：

```java
    public File getSharedPrefsFile(String name) {
        //依据我们传入的文件名字符串创建一个后缀为xml的文件
        return makeFilename(getPreferencesDir(), name + ".xml");
    }

    private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                //获取当前app的data目录下的shared_prefs目录
                mPreferencesDir = new File(getDataDirFile(), "shared_prefs");
            }
            return mPreferencesDir;
        }
    }
```

可以看见，原来SharePreference文件存储路径和文件创建是这个来的。继续往下看可以发现接着调运了SharedPreferencesImpl的构造函数，至于这个构造函数用来干嘛，下面会分析。

好了，到这里我们先回过头稍微总结一下目前的源码分析结论，具体如下：

前面我们有文章分析了Android中的Context，这里又发现ContextImpl中有一个静态的ArrayMap变量sSharedPrefs。这时候你想到了啥呢？无论有多少个ContextImpl对象实例，系统都共享这一个sSharedPrefs的Map，应用启动以后首次使用SharePreference时创建，系统结束时才可能会被垃圾回收器回收，所以如果我们一个App中频繁的使用不同文件名的SharedPreferences很多时这个Map就会很大，也即会占用移动设备宝贵的内存空间，所以说我们应用中应该尽可能少的使用不同文件名的SharedPreferences，取而代之的是合并他们，减小内存使用。同时上面最后一段代码也及具有隐藏含义，其表明了SharedPreferences是可以通过MODE_MULTI_PROCESS来进行夸进程访问文件数据的，其reload就是为了夸进程能更好的刷新访问数据。

好了，还记不记得上面我们分析留的尾巴呢？现在我们就来看看这个尾巴，可以发现SharedPreferencesImpl类其实就是SharedPreferences接口的实现类，其构造函数如下：

```java
final class SharedPreferencesImpl implements SharedPreferences {
    ......
    //构造函数，file是前面分析data目录下创建的传入name的xml文件，mode为传入的访问方式
    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        //依据文件名创建一个同名的.bak备份文件，当mFile出现crash的会用mBackupFile来替换恢复数据
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        //将文件从flash或者sdcard异步加载到内存中
        startLoadFromDisk();
    }
    ......
    //创建同名备份文件
    private static File makeBackupFile(File prefsFile) {
        return new File(prefsFile.getPath() + ".bak");
    }
    ......
    private void startLoadFromDisk() {
        //同步操作mLoaded标志，写为未加载，这货是关键的关键！！！！
        synchronized (this) {
            mLoaded = false;
        }
        //开启一个线程异步同步加载disk文件到内存
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                synchronized (SharedPreferencesImpl.this) {
                    //新线程中在SharedPreferencesImpl对象锁中异步load数据，如果此时数据还未load完成，则其他线程调用SharedPreferences.getXXX方法都会被阻塞，具体原因关注mLoaded标志变量即可！！！！！
                    loadFromDiskLocked();
                }
            }
        }.start();
    }
}
```

好了，到这里你会发现整个SharedPreferencesImpl的构造函数很简单，那我们就继续分析真正的异步加载文件到内存过程，如下：

```java
    private void loadFromDiskLocked() {
        //如果已经异步加载直接return返回
        if (mLoaded) {
            return;
        }
        //如果存在备份文件则直接使用备份文件
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
        ......
        Map map = null;
        StructStat stat = null;
        try {
            //获取Linux文件stat信息，Linux高级C中经常出现的
            stat = Os.stat(mFile.getPath());
            //文件至少是可读的
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    //把文件以BufferedInputStream流读出来
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    //使用系统提供的XmlUtils工具类将xml流解析转换为map类型数据
                    map = XmlUtils.readMapXml(str);
                } catch (XmlPullParserException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (FileNotFoundException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
        }
        //标记置为为已读
        mLoaded = true;
        if (map != null) {
            //把解析的map赋值给mMap
            mMap = map;
            mStatTimestamp = stat.st_mtime;//记录时间戳
            mStatSize = stat.st_size;//记录文件大小
        } else {
            mMap = new HashMap<String, Object>();
        }
        //唤醒其他等待线程（其实就是调运该类的getXXX方法的线程），因为在getXXX时会通过mLoaded标记是否进入wait，所以这里需要notify
        notifyAll();
    }
```

OK，到此整个Android应用获取SharePreference实例的过程我们就分析完了，简单总结下如下：

创建相关权限和mode的xml文件，异步同步锁加载xml文件并解析xml数据为map类型到内存中等待使用操作，特别注意，在xml文件异步加载未完成时调运SharePreference的getXXX及setXXX方法是阻塞等待的。由此也可以知道，一旦拿到SharePreference对象之后的getXXX操作其实都不再是文件读操作了，也就不存在网上扯蛋的认为多次频繁使用getXXX方法降低性能的说法了。

分析完了构造实例化，我们回忆可以知道使用SharePreference可以通过getXXX方法直接获取已经存在的key-value数据，下面我们就来看下这个过程，这里我们随意看一个方法即可，如下：

```java
    public boolean getBoolean(String key, boolean defValue) {
        //可以看见，和上面异步load数据使用的是同一个对象锁
        synchronized (this) {
            //阻塞等待异步加载线程加载完成notify
            awaitLoadedLocked();
            //加载完成后解析的xml数据放在mMap对象中，我们从mMap中找出指定key的数据
            Boolean v = (Boolean)mMap.get(key);
            //存在返回找到的值，不存在返回设置的defValue
            return v != null ? v : defValue;
        }
    }
```

先不解释，我们来关注下上面方法调运的awaitLoadedLocked方法，具体如下：

```java
    private void awaitLoadedLocked() {
        ......
        //核心，这就是异步阻塞等待
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```

哈哈，不解释，这也太赤裸裸的明显了，就是阻塞，就是这么任性，没辙。那我们继续攻占高地呗，get完事了，那就是set了呀。

### **3-3 SharePreferences内部类Editor实现EditorImpl分析**

还记不记得set是在SharePreference接口的Editor接口中定义的，而SharePreference提供了edit()方法来获取Editor实例，我们先来看下这个edit()方法吧，如下：

```java
    public Editor edit() {
        //握草！这也和异步load用的一把锁
        synchronized (this) {
            //阻塞等待，不解释吧，向上看。。。
            awaitLoadedLocked();
        }
    //异步加载OK以后通过EditorImpl创建Editor实例
        return new EditorImpl();
    }
```

可以看见，SharePreference的edit()方法其实就是阻塞等待返回一个Editor的实例（Editor的实现是EditorImpl），那我们就顺藤摸瓜一把，来看下这个EditorImpl这个类，如下：

```java
    public final class EditorImpl implements Editor {
        //创建一个mModified的key-value集合，用来在内存中暂存数据
        private final Map<String, Object> mModified = Maps.newHashMap();
        //一个是否清除preference的flag
        private boolean mClear = false;

        ......//省略类似的putXXX方法
        public Editor putBoolean(String key, boolean value) {
            //同步锁操作
            synchronized (this) {
                //将我们要存储的数据放入mModified集合中
                mModified.put(key, value);
                //返回当前对象实例，方便这种模式的代码写法：putXXX().putXXX();
                return this;
            }
        }
        //不用过多解释，同步删除mModified中包含key的数据
        public Editor remove(String key) {
            synchronized (this) {
                mModified.put(key, this);
                return this;
            }
        }
        //不解释，要清楚所有数据则直接置位mClear标记
        public Editor clear() {
            synchronized (this) {
                mClear = true;
                return this;
            }
        }
        ......
    }
```

好了，到此你可以发现Editor的setXXX及clear操作仅仅只是将相关数据暂存到内存中或者设置好标记为，也就是说调运了Editor的putXXX后其实数据是没有存入SharePreference的。那么通过我们一开始的实例可以知道，要想将Editor的数据存入SharePreference文件需要调运Editor的commit或者apply方法来生效。所以我们接下来先来看看Editor类常用的commit方法实现原理，如下：

```java
public boolean commit() {
    //1.先通过commitToMemory方法提交到内存
    MemoryCommitResult mcr = commitToMemory();
    //2.写文件操作
    SharedPreferencesImpl.this.enqueueDiskWrite(
            mcr, null /* sync write on this thread okay */);
    try {
        //阻塞等待写操作完成，UI操作需要注意！！！所以如果不关心返回值可以考虑用apply替代，具体原因等会分析apply就明白了。
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    }
    //3.通知数据发生变化了
    notifyListeners(mcr);
    //4.返回写文件是否成功状态
    return mcr.writeToDiskResult;
}
```

我去，小小一个commit方法做了这么多操作，主要分为四个步骤，我们先来看下第一个步骤，通过commitToMemory方法提交到内存返回一个MemoryCommitResult对象。分析commitToMemory方法前先看下MemoryCommitResult这个类，具体如下：

```java
// Return value from EditorImpl#commitToMemory()
//也是内部类，只是为了组织数据结构而诞生，也就是EditorImpl.commitToMemory()的返回值
private static class MemoryCommitResult {
    public boolean changesMade;  // any keys different?
    public List<String> keysModified;  // may be null
    public Set<android.content.SharedPreferences.OnSharedPreferenceChangeListener> listeners;  // may be null
    public Map<?, ?> mapToWriteToDisk;
    public final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);
    public volatile boolean writeToDiskResult = false;

    public void setDiskWriteResult(boolean result) {
        writeToDiskResult = result;
        writtenToDiskLatch.countDown();
    }
}
```

回过头现在来看commitToMemory方法，具体如下：

```java
// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
    //啥也不说，先整一个实例化对象
    MemoryCommitResult mcr = new MemoryCommitResult();
    //和SharedPreferencesImpl共用一把锁
    synchronized (SharedPreferencesImpl.this) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            //有多个未完成的写操作时复制一份，但是我们不知道用来干啥？？？？？？？
            mMap = new HashMap<String, Object>(mMap);
        }
        //构造数据结构，把通过SharedPreferencesImpl构造函数里异步加载的文件xml解析结果mMap赋值给要写到disk的Map
        mcr.mapToWriteToDisk = mMap;
        //增加一个未完成的写opt
        mDiskWritesInFlight++;
    //判断有没有监听设置
        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            //创建监听队列
            mcr.keysModified = new ArrayList<String>();
            mcr.listeners =
                    new HashSet<android.content.SharedPreferences.OnSharedPreferenceChangeListener>(mListeners.keySet());
        }
        //再加一把自己的锁
        synchronized (this) {
            //如果调运的是Editor的clear方法，则这里commit时这么处理
            if (mClear) {
                //如果从文件里加载出来的xml不为空
                if (!mMap.isEmpty()) {
                    //设置数据结构中数据变化标志为true
                    mcr.changesMade = true;
                    //清空内存中xml数据
                    mMap.clear();
                }
                //处理完毕，标记复位，程序继续执行，所以如果这次Editor中如果有写数据且还未commit，则执行完这次commit之后不会清掉本次写操作的数据，只会clear以前xml文件中的所有数据
                mClear = false;
            }
        //mModified是调运Editor的setXXX零时存储的map
            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                //删除需要删除的key-value
                if (v == this || v == null) {
                    if (!mMap.containsKey(k)) {
                        continue;
                    }
                    mMap.remove(k);
                } else {
                    if (mMap.containsKey(k)) {
                        Object existingValue = mMap.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    //把变化和新加的数据更新到SharePreferenceImpl的mMap中
                    mMap.put(k, v);
                }
            //设置数据结构变化标记
                mcr.changesMade = true;
                if (hasListeners) {
                    //设置监听
                    mcr.keysModified.add(k);
                }
            }
            //清空Editor中零时存储的数据
            mModified.clear();
        }
    }
    //返回重新更新过mMap值封装的数据结构
    return mcr;
}   
```

到此我们Editor的commit方法的第一步已经完成，根据写操作组织内存数据，返回组织后的[数据结构](http://lib.csdn.net/base/datastructure)。接下来我们继续回到commit方法看下第二步—-写到文件中，其核心是调运SharedPreferencesImpl类的enqueueDiskWrite方法实现。具体如下：

```java
//按照队列把内存数据写入磁盘，commit时postWriteRunnable为null，apply时不为null
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    //创建一个writeToDiskRunnable的Runnable对象
    final Runnable writeToDiskRunnable = new Runnable() {
        public void run() {
            synchronized (mWritingToDiskLock) {
                //真正的写文件操作
                writeToFile(mcr);
            }
            synchronized (SharedPreferencesImpl.this) {
                //写完一个计数器-1
                mDiskWritesInFlight--;
            }
            if (postWriteRunnable != null) {
                //等会apply分析
                postWriteRunnable.run();
            }
        }
    };
    //判断是同步写还是异步
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    //commit方式走这里
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (SharedPreferencesImpl.this) {
            //如果当前只有一个写操作
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            //一个写操作就直接在当前线程中写文件，不用另起线程
            writeToDiskRunnable.run();
            //写完文件就返回
            return;
        }
    }
    //如果是apply就在线程池中执行
    QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
}
```

可以发现，commit从内存写文件是在当前调运线程中直接执行的。那我们再来看看这个写内存到磁盘方法中真正的写方法writeToFile，如下：

```java
    // Note: must hold mWritingToDiskLock
    private void writeToFile(MemoryCommitResult mcr) {
        if (mFile.exists()) {
            if (!mcr.changesMade) {
                //如果文件存在且没有改变的数据则直接返回写OK
                mcr.setDiskWriteResult(true);
                return;
            }

            if (!mBackupFile.exists()) {
                //如果要写入的文件已经存在，并且备份文件不存在时就先把当前文件备份一份，因为如果本次写操作失败时数据可能已经乱了，所以下次实例化load数据时可以从备份文件中恢复
                if (!mFile.renameTo(mBackupFile)) {
                    Log.e(TAG, "Couldn't rename file " + mFile
                          + " to backup file " + mBackupFile);
                    //命名失败直接返回写失败了
                    mcr.setDiskWriteResult(false);
                    return;
                }
            } else {
                //备份文件存在就把源文件删掉，因为要写新的
                mFile.delete();
            }
        }

        try {
            //创建mFile文件
            FileOutputStream str = createFileOutputStream(mFile);
            if (str == null) {
                //创建失败直接返回写失败了
                mcr.setDiskWriteResult(false);
                return;
            }
            //把数据写入mFile文件
            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
            //彻底同步到磁盘文件中
            FileUtils.sync(str);
            str.close();
            //设置文件权限mode            
            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
            //和刚开始实例化load时一样，更新文件时间戳和大小
            try {
                final StructStat stat = Os.stat(mFile.getPath());
                synchronized (this) {
                    mStatTimestamp = stat.st_mtime;
                    mStatSize = stat.st_size;
                }
            } catch (ErrnoException e) {
                // Do nothing
            }
            // Writing was successful, delete the backup file if there is one.
            //写成功了mFile那就把备份文件直接删掉，没用了。
            mBackupFile.delete();
            //设置写成功了，然后返回
            mcr.setDiskWriteResult(true);
            return;
        } catch (XmlPullParserException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        } catch (IOException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        }
        // Clean up an unsuccessfully written file
        if (mFile.exists()) {
            //上面如果出错了就删掉，因为写之前已经备份过数据了，下次load时load备份数据
            if (!mFile.delete()) {
                Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
            }
        }
        //写失败了
        mcr.setDiskWriteResult(false);
    }
```

回过头可以发现，上面commit的第二步写磁盘操作其实是做了类似数据库的事务操作机制的（备份文件）。接着可以继续分析commit方法的第三四步，很明显可以看出，第三步就是回调设置的监听方法，通知数据变化了，第四步就是返回commit写文件是否成功。

总体到这里你可以发现，一个常用的SharePreferences过程已经完全分析完毕。接下来我们就再简单说说Editor的apply方法原理，先来看下Editor的apply方法，如下：

```java
public void apply() {
    //有了上面commit分析，这个雷同，写数据到内存，返回数据结构
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
        public void run() {
            try {
                //等待写文件结束
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException ignored) {
            }
        }
    };

    QueuedWork.add(awaitCommit);
    //一个收尾的Runnable
    Runnable postWriteRunnable = new Runnable() {
        public void run() {
            awaitCommit.run();
            QueuedWork.remove(awaitCommit);
        }
    };
    //这个上面commit已经分析过的，这里postWriteRunnable不为null，所以会在一个新的线程池调运postWriteRunnable的run方法
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    //通知变化
    notifyListeners(mcr);
}
```

看到了吧，其实和commit类似，只不过他是异步写的，没在当前线程执行写文件操作，还有就是他不像commit一样返回文件是否写成功状态。

### **3-4 SharePreferences源码分析总结**

**题外趣事：** 记得好像是去年有一次我用SharePreferences存储了几个boolean值，由于开发调试，当时我直接进入系统data目录下应用的xml存储文件夹，然后执行了删除操作；接着我没有重启应用，直接打断点调试，握草！奇迹的发现SharePreferences调运get时竟然拿到了不是初始化的值。哈哈，其实就是上面分析的，加载完后是一个静态的map，进程没挂之前一直用的内存数据。

通过上面的实例及源码分析可以发现：

- SharedPreferences在实例化时首先会从sdcard异步读文件，然后缓存在内存中；接下来的读操作都是内存缓存操作而不是文件操作。
- 在SharedPreferences的Editor中如果用commit()方法提交数据，其过程是先把数据更新到内存，然后在当前线程中写文件操作，提交完成返回提交状态；如果用的是apply()方法提交数据，首先也是写到内存，接着在一个新线程中异步写文件，然后没有返回值。
- 由于上面分析了，在写操作commit时有三级锁操作，所以效率很低，所以当我们一次有多个修改写操作时等都批量put完了再一次提交确认，这样可以提高效率。

可以发现，在简单数据行为状态存储中，Android的SharedPreferences是一个安全而且不错的选择。

## **4 SharePreferences进阶项目解决方案**

其实分析完源码之后也就差不多了。这里所谓的进阶项目解决方案只是曲线救国的2B行为，只是表明还有这么一种方案，至于项目中是否值得提倡那就要综合酌情考虑了。

### **4-1 SharePreferences存储复杂对象的解决案例**

这个案例完全有些多余（因为看完这个例子你会发现还不如使用github上大神流行的ACache更爽呢！），但是也比较有意思，所以还是拿出来说说，其实网上实现的也很多。

我们有时候可能会涉及到存储一个自定义对象到SharedPreferences中，这个怎么实现呢？标准的SharedPreferences的Editor只提供几个常见类型的put方法呀，其实可以实现的，原理就是Base64转码为字符串存储，如下给出我的一个工具类，在项目中可以直接使用：

```java
/**
 * @author 工匠若水
 * @version 1.0
 * http://blog.csdn.net/yanbober
 * 保存任意类型对象到SharedPreferences工具类
 */
public final class ObjectSharedPreferences {
    private Context mContext;
    private String mName;
    private int mMode;

    public ObjectSharedPreferences(Context context, String name) {
        this(context, name, Context.MODE_PRIVATE);
    }

    public ObjectSharedPreferences(Context context, String name, int mode) {
        this.mContext = context;
        this.mName = name;
        this.mMode = mode;
    }

    /**
     * 保存任意object对象到SharedPreferences
     * @param key
     * @param object
     */
    public void setObject(String key, Object object) {
        SharedPreferences preferences = mContext.getSharedPreferences(mName, mMode);

        ObjectOutputStream objOutputStream = null;
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        try {
            objOutputStream = new ObjectOutputStream(outputStream);
            objOutputStream.writeObject(object);
            String objectVal = new String(Base64.encode(outputStream.toByteArray(), Base64.DEFAULT));
            SharedPreferences.Editor editor = preferences.edit();
            editor.putString(key, objectVal);
            editor.commit();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (outputStream != null) {
                    outputStream.close();
                }
                if (objOutputStream != null) {
                    objOutputStream.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 从SharedPreferences获取任意object对象
     * @param key
     * @param clazz
     * @return
     */
    public <T> T getObject(String key, Class<T> clazz) {
        SharedPreferences preferences = this.mContext.getSharedPreferences(mName, mMode);
        if (preferences.contains(key)) {
            String objectVal = preferences.getString(key, null);
            byte[] buffer = Base64.decode(objectVal, Base64.DEFAULT);
            ByteArrayInputStream inputStream = new ByteArrayInputStream(buffer);
            ObjectInputStream objInputStream = null;
            try {
                objInputStream = new ObjectInputStream(inputStream);
                return (T) objInputStream.readObject();
            } catch (StreamCorruptedException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            } finally {
                try {
                    if (inputStream != null) {
                        inputStream.close();
                    }
                    if (objInputStream != null) {
                        objInputStream.close();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return null;
    }
}
```

好了，没啥解释的，依据需求自己决定吧，这只是一种方案而已，其替代方案很多。

### **4-2 SharePreferences跨进程访问解决案例**

Android系统有自己的一套安全机制，当应用程序在安装时系统会分配给他们一个唯一的userid，当该应用程序需要访问文件等资源的时候必须要匹配userid。默认情况下[安卓](http://lib.csdn.net/base/android)应用程序创建的各种文件（SharePreferences、数据库等）都是私有的（在`/data/data/[APP PACKAGE NAME]/files`），其他程序无法访问。在3.0以后必须在创建文件时显示的指定了Context.MODE_WORLD_READABLE或者Context.MODE_WORLD_WRITEABLE才能被其他进程（应用）访问。

这个案例也就类似于Android的ContentProvider，可以跨进程访问，但是其原理是给权限然后多进程文件访问而已。具体我们看如下一个案例，一个用来类比当服务端，一个用来类比当客户端，如下：

**服务端：**

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private TextView mTextView;
    private EditText mEditText;
    private Button mButton;

    private SharedPreferences mPreferences;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = (TextView) findViewById(R.id.content);
        mEditText = (EditText) findViewById(R.id.input);
        mButton = (Button) findViewById(R.id.click);
        mButton.setOnClickListener(this);
        //操作当前APP本地的SharedPreferences文件local.xml
        mPreferences = getSharedPreferences("local", MODE_WORLD_READABLE | MODE_WORLD_WRITEABLE);
    }

    @Override
    public void onClick(View v) {
        mTextView.setText(mPreferences.getString("input", "default"));
        SharedPreferences.Editor editor = mPreferences.edit();
        editor.putString("input", mEditText.getText().toString());
        editor.commit();
    }
}
```

**客户端：**

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private TextView mTextView;
    private EditText mEditText;
    private Button mButton;

    private SharedPreferences mPreferences;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = (TextView) findViewById(R.id.content);
        mEditText = (EditText) findViewById(R.id.input);
        mButton = (Button) findViewById(R.id.click);
        mButton.setOnClickListener(this);

        mPreferences = getRemotePreferences("com.local.localapp", "local");
    }

    //获取其他APP的SharedPreferences文件local.xml，特别注意MODE_MULTI_PROCESS的mode
    private SharedPreferences getRemotePreferences(String pkg, String file) {
        try {
            Context remote = createPackageContext(pkg, Context.CONTEXT_IGNORE_SECURITY);
            return remote.getSharedPreferences(file, MODE_WORLD_READABLE | MODE_WORLD_WRITEABLE | MODE_MULTI_PROCESS);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }

    @Override
    public void onClick(View v) {
        mTextView.setText(mPreferences.getString("input", "default"));
        SharedPreferences.Editor editor = mPreferences.edit();
        editor.putString("input", mEditText.getText().toString());
        editor.commit();
    }
}
```

不解释，需求自己看情况吧，这就是一个经典的跨进程访问SharedPreferences，主要原理就是设置flag然后获取其他App的Context。

## **5 SharePreferences总结**

到此整个SharePreferences的使用及原理和进阶开发案例都已经全面剖析完毕，相信你对Android的SharePreferences会有一个全新的自己的认识，不再会被网上那些争议的评论而左右。下一篇文章会继续探讨Android的SharePreferences拓展控件，即Preference控件家族的使用及源码分析。
