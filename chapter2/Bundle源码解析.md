## Bundle源码解析 ##

### 1. Bundle的概念理解 ###
Bundle对于Android开发者来说肯定非常眼熟，它经常出现在以下场合：

- Activity状态数据的保存与恢复涉及到的两个回调：`void onSaveInstanceState (Bundle outState)`、`void onCreate (Bundle savedInstanceState)`
- Fragment的setArguments方法：`void setArguments (Bundle args)`
- 消息机制中的Message的setData方法：`void setData (Bundle data)`
- 其他场景不再列举

Bundle从字面上解释为“一捆、一批、一包”，结合上述几个应用场合，可以知道Bundle是用来传递数据的，我们暂将Bundle理解为Android中用来传递数据的一个容器。官方文档对Bundle的说明如下：

> A mapping from String values to various Parcelable types.

官方意为Bundle封装了String值到各种Parcelable类型数据的映射，可见跟我们上述理解是吻合的。

### 2. Bundle源码分析 ###

知道了Bundle的主要作用，再来看源码就容易理解了。

Bundle位于`android.os`包中，是一个final类，这就注定了Bundle不能被继承。Bundle继承自BaseBundle并实现了Cloneable和Parcelable两个接口，因此对Bundle源码的分析会结合着对BaseBundle源码进行分析。由于实现了Cloneable和Parcelable接口，因此以下几个重载是必不可少的：

-  `public Object clone()`
-  `public int describeContents()`
-  `public void writeToParcel(Parcel parcel, int flags)`
-  `public void readFromParcel(Parcel parcel)`
-  `public static final Parcelable.Creator<Bundle> CREATOR = new Parcelable.Creator<Bundle>()`

以上代码无需过多解释。

#### 2.1 Bundle的几个公有构造方法 ####

| 公有构造方法                             | 说明                                       |
| :--------------------------------- | :--------------------------------------- |
| public Bundle()                    | Constructs a new, empty Bundle           |
| public Bundle(ClassLoader loader)  | Constructs a new, empty Bundle that uses a specific ClassLoader for instantiating Parcelable and Serializable objects. |
| public Bundle(int capacity)        | Constructs a new, empty Bundle sized to hold the given number of elements. |
| public Bundle(Bundle b)            | Constructs a Bundle containing a copy of the mappings from the given Bundle. |
| public Bundle(PersistableBundle b) | Constructs a Bundle containing a copy of the mappings from the given PersistableBundle. |

第5个构造函数中的PersistableBundle也是继承自BaseBundle的，Bundle还提供了一个静态方法，用来返回只包含一个键值对的Bundle对象，具体源码如下：
```java
    public static Bundle forPair(String key, String value) {
        Bundle b = new Bundle(1);
        b.putString(key, value);
        return b;
    }
```

#### 2.2 Bundle的put与get方法族 ####

Bundle的功能是用来保存数据，那么必然提供了一系列存取数据的方法，这些方法太多了，几乎能够存取任何类型的数据，具体整理为下表：

| 相关保存方法                                   | 相关读取方法                                   |
| :--------------------------------------- | :--------------------------------------- |
| public void putBoolean(String key, boolean value) | public boolean getBoolean(String key)    |
| public void putByte(String key, byte value) | public byte getByte(String key)          |
| public void putChar(String key, char value) | public char getChar(String key)          |
| public void putShort(String key, short value) | public short getShort(String key)        |
| public void putFloat(String key, float value) | public float getFloat(String key)        |
| public void putCharSequence(String key, CharSequence value) | public CharSequence getCharSequence(String key) |
| public void putParcelable(String key, Parcelable value) | public <T extends Parcelable> T getParcelable(String key) |
| public void putSize(String key, Size value) | public Size getSize(String key)          |
| public void putSizeF(String key, SizeF value) | public SizeF getSizeF(String key)        |
| public void putParcelableArray(String key, Parcelable[] value) | public Parcelable[] getParcelableArray(String key) |
| public void putParcelableArrayList(String key, ArrayList<? extends Parcelable> value) | public <T extends Parcelable> ArrayList<T> getParcelableArrayList(String key) |
| public void putSparseParcelableArray(String key, SparseArray<? extends Parcelable> value) | public <T extends Parcelable> SparseArray<T> getSparseParcelableArray(String key) |
| public void putIntegerArrayList(String key, ArrayList<Integer> value) | public ArrayList<Integer> getIntegerArrayList(String key) |
| public void putStringArrayList(String key, ArrayList<String> value) | public ArrayList<String> getStringArrayList(String key) |
| public void putCharSequenceArrayList(String key, ArrayList<CharSequence> value) | public ArrayList<CharSequence> getCharSequenceArrayList(String key) |
| public void putSerializable(String key, Serializable value) | public Serializable getSerializable(String key) |
| public void putBooleanArray(String key, boolean[] value) | public boolean[] getBooleanArray(String key) |
| public void putByteArray(String key, byte[] value) | public byte[] getByteArray(String key)   |
| public void putShortArray(String key, short[] value) | public short[] getShortArray(String key) |
| public void putCharArray(String key, char[] value) | public char[] getCharArray(String key)   |
| public void putFloatArray(String key, float[] value) | public float[] getFloatArray(String key) |
| public void putCharSequenceArray(String key, CharSequence[] value) | public CharSequence[] getCharSequenceArray(String key) |
| public void putBundle(String key, Bundle value) | public Bundle getBundle(String key)      |
| public void putBinder(String key, IBinder value) | public IBinder getBinder(String key)     |

除了上述存取数据涉及到的方法外，Bundle还提供了一个clear方法：`public void clear()`，该方法可用于移除Bundle中的所有数据。

Bundle之所以能以键值对的方式存储数据，实质上是因为它内部维护了一个ArrayMap，具体定义是在其父类BaseBundle中：

> ArrayMap<String, ObjectmMap = null;

ArrayMap比HashMap更加高效、更加节省内存，它的初始化是在Bundle的构造方法中实现的，源码如下：

```java
    BaseBundle(ClassLoader loader, int capacity) {
        mMap = capacity > 0 ?
                new ArrayMap<String, Object>(capacity) : new ArrayMap<String, Object>();
        mClassLoader = loader == null ? getClass().getClassLoader() : loader;
    }
```

另外还可以发现，存储数据用的key都是String类型，值为各种数据类型，甚至可以存储Bundle、Binder等，如果ArrayMap中存在相同的key，则会替换掉之前的对应值。每一个put方法都对应一个get方法，对于基本数据类型还可以设置缺省值（上述表中未列出对应方法）。具体的存取则是在其父类BaseBundle中实现的，下面就以布尔类型数据为例来分析一下。

#### 2.3 Bundle存取数据的具体实现 ####

布尔数据的存储源码如下：
```java
    /**
     * Inserts a Boolean value into the mapping of this Bundle, replacing
     * any existing value for the given key.  Either key or value may be null.
     *
     * @param key a String, or null
     * @param value a Boolean, or null
     */
    void putBoolean(String key, boolean value) {
        unparcel();
        mMap.put(key, value);
    }
```

这里的mMap就是ArrayMap了，存储数据就是把键值对保存到ArrayMap里。

布尔类型数据的读取源码如下：
```java
    /**
     * Returns the value associated with the given key, or false if
     * no mapping of the desired type exists for the given key.
     *
     * @param key a String
     * @return a boolean value
     */
    boolean getBoolean(String key) {
        unparcel();
        if (DEBUG) Log.d(TAG, "Getting boolean in "+ Integer.toHexString(System.identityHashCode(this)));
        return getBoolean(key, false);
    }

getBoolean(String key, boolean defaultValue)的具体实现如下：

    /**
     * Returns the value associated with the given key, or defaultValue if
     * no mapping of the desired type exists for the given key.
     *
     * @param key a String
     * @param defaultValue Value to return if key does not exist
     * @return a boolean value
     */
    boolean getBoolean(String key, boolean defaultValue) {
        unparcel();
        Object o = mMap.get(key);
        if (o == null) {
            return defaultValue;
        }
        try {
            return (Boolean) o;
        } catch (ClassCastException e) {
            typeWarning(key, o, "Boolean", defaultValue, e);
            return defaultValue;
        }
    }
```

数据读取的逻辑也很简单，就是通过key从ArrayMap里读出保存的数据，并转换成对应的类型返回，当没找到数据或发生类型转换异常时返回缺省值。

注意到这里出现了一个方法：`unparcel()`，它的具体源码如下：
```java
    /**
     * If the underlying data are stored as a Parcel, unparcel them
     * using the currently assigned class loader.
     */

    /* package */ synchronized void unparcel() {
        if (mParcelledData == null) {
            if (DEBUG) Log.d(TAG, "unparcel " + Integer.toHexString(System.identityHashCode(this))
                    + ": no parcelled data");
            return;
        }

        if (mParcelledData == EMPTY_PARCEL) {
            if (DEBUG) Log.d(TAG, "unparcel " + Integer.toHexString(System.identityHashCode(this))
                    + ": empty");
            if (mMap == null) {
                mMap = new ArrayMap<String, Object>(1);
            } else {
                mMap.erase();
            }
            mParcelledData = null;
            return;
        }

        int N = mParcelledData.readInt();
        if (DEBUG) Log.d(TAG, "unparcel " + Integer.toHexString(System.identityHashCode(this))
                + ": reading " + N + " maps");
        if (N < 0) {
            return;
        }
        if (mMap == null) {
            mMap = new ArrayMap<String, Object>(N);
        } else {
            mMap.erase();
            mMap.ensureCapacity(N);
        }
        mParcelledData.readArrayMapInternal(mMap, N, mClassLoader);
        mParcelledData.recycle();
        mParcelledData = null;
        if (DEBUG) Log.d(TAG, "unparcel " + Integer.toHexString(System.identityHashCode(this)) + " final map: " + mMap);
    }
```

先来看下BaseBundle中mParcelledData的定义：
```java
    /*
     * If mParcelledData is non-null, then mMap will be null and the
     * data are stored as a Parcel containing a Bundle.  When the data
     * are unparcelled, mParcelledData willbe set to null.
     */
    Parcel mParcelledData = null;
```

在大部分情况下mParcelledData都是null，因此unparcel()直接返回。当使用构造函数`public Bundle(Bundle b)`创建Bundle时，会给mParcelledData赋值，具体实现如下：

```java
    /**
     * Constructs a Bundle containing a copy of the mappings from the given
     * Bundle.
     *
     * @param b a Bundle to be copied.
     */
    BaseBundle(BaseBundle b) {
        if (b.mParcelledData != null) {
            if (b.mParcelledData == EMPTY_PARCEL) {
                mParcelledData = EMPTY_PARCEL;
            } else {
                mParcelledData = Parcel.obtain();
                mParcelledData.appendFrom(b.mParcelledData, 0, b.mParcelledData.dataSize());
                mParcelledData.setDataPosition(0);
            }
        } else {
            mParcelledData = null;
        }

        if (b.mMap != null) {
            mMap = new ArrayMap<String, Object>(b.mMap);
        } else {
            mMap = null;
        }

        mClassLoader = b.mClassLoader;
    }
```

从上述代码片段可以知道mParcelledData的取值有3种情况：

- mParcelledData = EMPTY_PARCEL
- mParcelledData = Parcel.obtain()
- mParcelledData = null

在`unparcel()`方法中就对上述几种情况做了不同的处理，当mParcelledData为null时，直接返回；当mParcelledData为EMPTY_PARCEL时，会创建一个容量为1的ArrayMap对象；当mParcelledData为Parcel.obtain()时，则会将里面的数据读出，并创建一个ArrayMap，并将数据存储到ArrayMap对象里面，同时将mParcelledData回收并置为null，具体是由以下代码片段实现的：

```java
        int N = mParcelledData.readInt();
        if (DEBUG) Log.d(TAG, "unparcel " + Integer.toHexString(System.identityHashCode(this))
                + ": reading " + N + " maps");
        if (N < 0) {
            return;
        }
        if (mMap == null) {
            mMap = new ArrayMap<String, Object>(N);
        } else {
            mMap.erase();
            mMap.ensureCapacity(N);
        }
        mParcelledData.readArrayMapInternal(mMap, N, mClassLoader);
        mParcelledData.recycle();
        mParcelledData = null;
```

上面只是以布尔类型的数据为例分析了Bundle的存取过程，其他数据类型的存取原理相同，就不再赘述。

### 3. 小结

到此，Bundle的源码分析基本就结束了，其实Bundle比较简单，只是一个数据容器，不像Activity等有复杂的生命周期。对于开发者来说，只需要了解Bundle的功能、使用场景并掌握常用的数据存取方法即可。

在本人博客上也可以找到这篇文章：[Bundle源码解析](http://blog.csdn.net/ahence/article/details/51443722)
