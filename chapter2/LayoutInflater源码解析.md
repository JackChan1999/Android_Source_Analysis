# LayoutInflater源码解析
> 本文分析版本: Android API 23，v4基于 23.2.1

## 1. 简介
实例化布局的XML文件成相应的View对象。它不能被直接使用，应该使用`getLayoutInflater（）`或`getSystemService（Class）`来获取已关联了当前`Context`并为你正在运行的设备正确配置的标准LayoutInflater实例对象。 例如：

`LayoutInflater inflater = (LayoutInflater)context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);`

为了创建一个对于你自己的View来说，附加了`LayoutInflater.Factory`的`LayoutInflater`，你需要使用`cloneInContext(Context)`来克隆一个已经存在`LayoutInflater`，然后调用`setFactory(LayoutInflater.Factory)`来替换成你自己的Factory。

由于性能原因，View的实例化很大程度上依赖对于xml文件在编译时候的预处理。因此，目前使用`LayoutInflater`不能使用直接通过原始xml文件获取的`XmlPullParser`，只能使用一个已编译的xml资源返回的`XmlPullParser`（(R.something file.）。

## 2. 获取LayoutInflater的三种方式：

- `LayoutInflater inflater = getLayoutInflater(); `//调用Activity的getLayoutInflater()
- `LayoutInflater inflater = LayoutInflater.from(context);`
- `LayoutInflater inflater =  (LayoutInflater)context.getSystemService(Context.LAYOUT_INFLATER_SERVICE); `

但是，这三种方式本质上还是一样的：

- `getLayoutInflater()`调用的还是Activity中的方法：

```java
    public LayoutInflater getLayoutInflater() {
        return getWindow().getLayoutInflater();
    }
```
其中`Window`是一个抽象类，它 **唯一** 实现类是`PhoneWindow`。

com/android/internal/policyPhoneWindow.java
```java
	public PhoneWindow(Context context) {
    	super(context);
    	mLayoutInflater = LayoutInflater.from(context);
		}
	/**
     * Return a LayoutInflater instance that can be used to inflate XML view layout
     * resources for use in this Window.
     *
     * @return LayoutInflater The shared LayoutInflater.
     */
    @Override
    public LayoutInflater getLayoutInflater() {
        return mLayoutInflater;
    }
```

可以看到，通过·`Activity`的`getLayoutInflater()`最终调用的还是第二种方法。
​
2.通过 `LayoutInflater.from(context);` 获取`LayoutInflater`对象。
​
android/view/LayoutInflater.java

```java
	/**
     * Obtains the LayoutInflater from the given context.
     */
    public static LayoutInflater from(Context context) {
		//通过获取系统服务的方式获取到LayoutInflater实例对象
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```

以上两种方法，最终还是调用了`context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)`获取LayoutInflater实例对象。

最终：不管以什么样的方式获取到LayoutInflater对象，最终都是[通过系统服务来获取实例对象](http://blog.csdn.net/chunqiuwei/article/details/50495686)。

## 3. 主要方法预览

*  `public static LayoutInflater from(Context context);`

获取LayoutInflater实例化对象。

* `public void setFactory(Factory factory);`

设置使用当前LayoutInflater创建View的自定义实例化工厂。通过设置自定义工厂，可以在系统实例化View的时候进行一些拦截操作，比如可以把本来的TextView拦截成Button、给TextView统一指定字体等。

* `public void setFactory2(Factory2 factory);`

同上，区别是工厂2多了对实例化View的时候Parent的支持。在API 11引入。

* `public void setFilter(Filter filter);`

给当前LayoutInflater设置过滤器，如果要被填充的View不被这个过滤器允许，则会抛出InflateException。这个过滤器会覆盖当前LayoutInflater之上的任何之前设置过的过滤器。

* `public View inflate(@LayoutRes int resource, @Nullable ViewGroup root);`
 > 把指定的布局资源填充成View。如果root不为空则把填充的View添加到root上，如果root为空则不添加。

* `public View inflate(XmlPullParser parser, @Nullable ViewGroup root);`

通过布局xml资源的解析器把布局资源填充成View。如果root不为空则把填充的View添加到root上，如果root为空则不添加。

* `public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot);`

把指定的布局资源填充成View。如果root不为空并且attachToRoot为true，则把填充的View添加到root上，否则不添加。

* `public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot);`

通过布局xml资源的解析器把布局资源填充成View。如果root不为空并且attachToRoot为true，则把填充的View添加到root上，否则不添加。

* `public final View createView(String name, String prefix, AttributeSet attrs);`

通过View的名称，前缀和attrs属性实例化View。

* `protected View onCreateView(String name, AttributeSet attrs);`

通过View的父View，View名称和attrs属性实例化View（最终调用的是`createView(String name, String prefix, AttributeSet attrs);`）。

* `protected View onCreateView(View parent, String name, AttributeSet attrs);`

通过View的父View，View名称、前缀和attrs属性实例化View（最终调用的是`createView(String name, String prefix, AttributeSet attrs);`）。

* `View createViewFromTag(View parent, String name,Context context, AttributeSet attrs);`

通过View的父View，View的名称、attrs属性实例化View（内部调用`onCreateView()`和`createView()`）。

* `void rInflate(XmlPullParser parser, View parent, Context context, final AttributeSet attrs, boolean finishInflate);`

解析Parent的子View并添加到Parent上（递归调用）。

## 4. 流程预览

```java
// 把xml布局资源或者通过资源解析器实例化View
inflate{
	if(merge标签){
		// 递归实例化根节点的子View
		rInflate();
		// 返回是父View（因为根节点是merge标签）
	}else{
		// 实例化根节点的View
		createViewFromTag();
		// 递归实例化跟节点的子View
		rInflateChildren()

		// 这个需要注意
		if(父View是空或者不把填充的View添加到父View){
			返回根节点View
		}else{
			返回父View
		}
	}
}
```

```java
// 通过View的名称实例化View
createViewFromTag{
	//各个工厂先onCreateView()
	onCreateView()或者createView()
}

// 递归实例化Parent的子View
rInflate{
	// 解析请求焦点
	parseRequestFocus()
	// 解析include标签
	parseInclude()
	// 通过View的名称实例化View
	createViewFromTag()
	rInflate
}
```

注意：在调用`inflate`方法的时候，传入的参数不一样，返回的View可是有区别的，总结起来就是：
​
1. 传入的父View为空或者不添加到父View上，则返回根节点的View。
2. 其他（也就是根节点是merge、父View不为空且添加到父View上，返回的是父View）。

## 5. 流程详情

关键字段：

```java
/************************字段定义区**********************/
// 反射调用构造方法的两个参数
final Object[] mConstructorArgs = new Object[2];
// 反射构造方法的参数类型
static final Class<?>[] mConstructorSignature = new Class[] {
        Context.class, AttributeSet.class};
protected final Context mContext;
```

PS：以下我们用 “当前View”来指代该操作的View，用“父View”指代“当前View”的父View。__

### 5.1 inflate方法解析

inflate方法主要是把布局资源实例化成View并返回。

通过获取LayoutInflater的三种方式，我们知道，通过布局文件填充成View对象最终调用的是下面两个方法：

```java
public View inflate(int resource, ViewGroup root, boolean attachToRoot) {
    if (DEBUG) System.out.println("INFLATING from resource: " + resource);
	//获取布局资源的xml解析器，注意：在开头的时候我们强调过，普通的xml是不被支持的，必须是经过编译器处理过的。
    XmlResourceParser parser = getContext().getResources().getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

最终调用的是此方法：

```java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
		// from传入的Context
		final Context inflaterContext = mContext;
		// 判断parser是否是AttributeSet，如果不是则用XmlPullAttributes去包装一下。
        final AttributeSet attrs = Xml.asAttributeSet(parser);
		// 保存之前的Context
        Context lastContext = (Context) mConstructorArgs[0];
		// 赋值为传入的Context
        mConstructorArgs[0] = inflaterContext;
		// 默认返回的是传入的Parent
        View result = root;

        try {
            // 查找开始标签
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

			//如果没找到有效的开始标签则抛出InflateException
            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }

			//获取控件的名称
            final String name = parser.getName();

            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }

			// 如果根节点是“merge”标签
            if (TAG_MERGE.equals(name)) {
				// 根节点为空或者不添加到根节点上，则抛出异常。
				// 因为“merge”标签必须是要被添加到父节点上的，不能独立存在。
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
				// 递归实例化root（也就是传入Parent）下所有的View
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
				// temp是当前xml的根节点的View。通过父View、View名、Context、属性，来实例化View。也即实例化根节点的View。
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

				// 如果传入Parent不为空
                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // 创建父View类型的LayoutParams参数
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
						// 如果不把填充的View 关联在父View上，则把父View的LayoutParams参数设置给它
                        // 如果把填充的View关联在父View上，则会走下面addView的逻辑
						temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }
                // 实例化根节点View下面的所有子View。
				// TODO ..................
                rInflate(parser, temp, attrs, true);
                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

				// Google建议关联所有找到的View
                // 如果根节点不为null，并且需要把根节点View关联到Parent上，则使用addView方法把布局填充成的View树添加到Parent上。
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

				// 决定返回的RootView（也即传入的Parent）还是xml中的根节点的View。
                // 如果传入的Parent为空 或 实例化的View不添加到Parent上，则返回布局文件的根节点的View
				// 否则，返回Parent
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            InflateException ex = new InflateException(e.getMessage());
            ex.initCause(e);
            throw ex;
        } catch (IOException e) {
            InflateException ex = new InflateException(
                    parser.getPositionDescription()
                    + ": " + e.getMessage());
            ex.initCause(e);
            throw ex;
        } finally {
            // Don't retain static reference on context.
			// 把这之前保存的Context从新放回全局变量中。
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return result;
    }
}
```

### 5.2 createViewFromTag方法解析
createViewFromTag方法主要通过要实例化View的父View、要实例化View的名称、Context上下文、属性值来实例化View。该方法主要做的操作有：

1. 先尝试用用户设置的Factory以及自己私有的Factory来实例化View。
2. 如果这几个Factory都没有实例化View，则调用`onCreateView`或者`createView`来实例化View。

代码分析：
```java
/*
 * 缺省方法可见性，好让BridgeInflater能重写它。
 * 根据父View、View名称、Context、属性实例化View。
 */
private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
    return createViewFromTag(parent, name, context, attrs, false);
}

View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {

	// 如果是View标签，则用class指向的类的完整名称来替换当前名称。(我们都知道，用Fragment的时候，可指定 class="Fragment完整路径名"，其他widget控件也类似)
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

	// 应用主题包装，如果允许并且已经被指定
    // Apply a theme wrapper, if allowed and one is specified.
    if (!ignoreThemeAttr) {
		// 获取Context中主题属性
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
		// 如果包含了主题，则用ContextThemeWrapper包装一下
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

	// 如果根标签是“1995”，则创建一个“BlinkLayout”（其实就是一个FrameLayout）
	// ps：这个没见过在哪里用到过。
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    try {
        View view;
		// 尝试通过 mFactory2 或者 mFactory来创建View，这两个是通过setFactory和setFactory2来设置的。
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

		// 如果没有设置自定义工厂并且LayoutInflater本身私有的View工厂不为空，则用私有View工厂创建View。
        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }

		// 如果View还为空，也即没有工厂，或者工厂未能正确创建View，则尝试通过自身的方法实例化View
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {

					// 如果View标签中没有"."，则代表是系统的widget，则调用onCreateView，这个方法会通过"createView"方法创建View
					// 不过前缀字段会自动补"android.view."。
                    view = onCreateView(parent, name, attrs);
                } else {

					// 非系统控件，则name本身就是控件的完整路径名。
					//通过widget完整路径名以及属性创建View。
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name);
        ie.initCause(e);
        throw ie;

    } catch (Exception e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name);
        ie.initCause(e);
        throw ie;
    }
}
```

### 5.3 onCreateView和createView方法解析

onCreateView有两个重载方法，最终调用的是`createView(String name, String prefix, AttributeSet attrs)`。

createView主要做的操作有：

1. 先通过Filter，看是否过滤。
2. 利用反射实例化View对象。

源码分析：

```java
protected View onCreateView(View parent, String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return onCreateView(name, attrs);
}

protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
	// 系统控件，前缀自动补上"android.view."
    return createView(name, "android.view.", attrs);
}

public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
	// 通过以View的name为key，查询构造函数的缓存map中时候已经有该View的构造函数。
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

		// 构造函数在缓存的map中没有，则尝试去创建并添加。
        if (constructor == null) {
			// 通过 类名去加载控件的字节码
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            //如果有自定义的过滤器并且加载到字节码，则通过过滤器判断是否允许加载该View。
            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
					// 如果不允许则抛出异常。
                    failNotAllowed(name, prefix, attrs);
                }
            }
			// 得到构造函数
            constructor = clazz.getConstructor(mConstructorSignature);
			constructor.setAccessible(true);
			// 缓存构造函数
            sConstructorMap.put(name, constructor);
        } else {
			// setFilter()可能会在类的构造函数被添加到map之后，所以获取到map中的构造函数后还需要判断是否过滤。
			if (mFilter != null) {
                 // 过滤的map中是否已经包含了此类名。
                Boolean allowedState = mFilterMap.get(name);
				// 当前类名没有被放到过滤的缓存map中
                if (allowedState == null) {
                    // 重新加载类的字节码
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);
                    // 重新通过过滤器判断是否过滤。
                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
					// 把过滤结果放到过滤的缓存map中。
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
						// 如果要过滤，则抛出异常。
                        failNotAllowed(name, prefix, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
					// 缓存构造函数的map中已经保存了当前要实例化的View的构造函数并且是要过滤的，抛出异常。
                    failNotAllowed(name, prefix, attrs);
                }
            }
        }

		// 实例化类的参数数组，0 是获取LayoutInflater传入的Context，1 是View的属性
        Object[] args = mConstructorArgs;
        args[1] = attrs;
		// 通过构造函数实例化View（看到此行就知道，为什么自定义View或者ViewGroup的时候，如果在布局中使用的话，必须重写两个参数的构造函数了。）
        final View view = constructor.newInstance(args);
		// 如果当前View是ViewStub，则把布局填充器设置给它。（因为ViewStub在此刻并不会填充期子View，而是等需要的时候由用户手动触发。）
        if (view instanceof ViewStub) {
            // 把当前LayoutInflater的克隆传递给ViewStub，让ViewStub实例化的时候用，因为ViewStub只是在需要的时候才会实例化View。
			final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        return view;

    } catch (NoSuchMethodException e) {
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class "
                + (prefix != null ? (prefix + name) : name));
        ie.initCause(e);
        throw ie;

    } catch (ClassCastException e) {
        // If loaded class is not a View subclass
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Class is not a View "
                + (prefix != null ? (prefix + name) : name));
        ie.initCause(e);
        throw ie;
    } catch (ClassNotFoundException e) {
        // If loadClass fails, we should propagate the exception.
        throw e;
    } catch (Exception e) {
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class "
                + (clazz == null ? "<unknown>" : clazz.getName()));
        ie.initCause(e);
        throw ie;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

### 5.4 rInflateChildren和rInflate方法解析

rInflate方法主要是遍历传入的Parent的子节点，实例化Parent的所有子View。

当遍历的时候发现当前View还有子View则递归调用此方法继续实例化当前View的子View。

这里面主要的操作有：

* parseRequestFocus()，处理请求焦点
* parseInclude()，处理include标签
* 实例化1995或一般View并添加到当前View的父View上

源码分析：

```java
/**
 * 循环方法用来深入xml的层级并且实例化内部的View（非根节点View），这个方法通过调用rInflate，并使用Parent的Context作为
 * 实例化View的Context。
 */
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}

/**
 * 循环方法用来深入xml的层级并且实例化view以及View的子View，
 * 最后调用onFinishInflate()方法
 */
void rInflate(XmlPullParser parser, View parent,Context context, final AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
	// 获取当前xml解析的深度
    final int depth = parser.getDepth();
    int type;
	// 循环遍历xml节点
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }
		// 获取节点的名称
        final String name = parser.getName();
        // 请求焦点
        if (TAG_REQUEST_FOCUS.equals(name)) {
            parseRequestFocus(parser, parent);
        } else if (TAG_TAG.equals(name)) {
			// 解析<tag>元素，并且设置键控标签在它包含的View上。最终调用的是View的view.setTag(key, value);方法
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
			//处理include标签
            if (parser.getDepth() == 0) {
				// 最外层使用include标签抛出异常。
                throw new InflateException("<include /> cannot be the root element");
            }
			// 解析include标签引入的布局
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
			// 如果是merge标签则抛出异常（因为此方法实例化的是xml根节点的子View，所以非根节点不能使用merge标签。）
            throw new InflateException("<merge /> must be the root element");
        } else {
			// 一般性的View
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
			// 获取父View的LayoutParams，并在把View添加到父View的时候带过去
			//（这里解释了，为什么自己手动new 一个View，添加到父View上的时候需要new父View的LayoutParams参数而不是自己的LayoutParams参数。）
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
			// 递归遍历实例化
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }
	// 如果父View下的所有View都完成填充，则调用父View的onFinishInflate()方法。
    if (finishInflate) parent.onFinishInflate();
}
```

### 5.5 parseInclude方法解析

```java
/**
 * 解析include标签
 */
private void parseInclude(XmlPullParser parser, Context context, View parent,
        AttributeSet attrs) throws XmlPullParserException, IOException {
    int type;

	// 如果include标签的父View是ViewGroup，则继续，否则抛出异常。
    if (parent instanceof ViewGroup) {
        // 使用主题包装。如果include中的View有自己的attr属性，则忽略。
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        final boolean hasThemeOverride = themeResId != 0;
        if (hasThemeOverride) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();

		// 获取Layout标签指向布局资源id
        int layout = attrs.getAttributeResourceValue(null, ATTR_LAYOUT, 0);
        if (layout == 0) {
			// 获取Layout的value值
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            if (value == null || value.length() <= 0) {
                throw new InflateException("You must specify a layout in the"
                        + " include tag: <include layout=\"@layout/layoutID\" />");
            }

			// 尝试解析"?attr/name"成id资源
            layout = context.getResources().getIdentifier(value.substring(1), null, null);
        }

        // include的布局可能引用了主题属性
        if (mTempValue == null) {
            mTempValue = new TypedValue();
        }
		// 尝试从主题中获取布局资源id，如果有的话。
        if (layout != 0 && context.getTheme().resolveAttribute(layout, mTempValue, true)) {
            layout = mTempValue.resourceId;
        }

		// 如果还是无法找到布局id，抛出异常。
        if (layout == 0) {
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            throw new InflateException("You must specify a valid layout "
                    + "reference. The layout ID " + value + " is not valid.");
        } else {

			// 获取include标签中布局的解析器
            final XmlResourceParser childParser = context.getResources().getLayout(layout);

            try {
                final AttributeSet childAttrs = Xml.asAttributeSet(childParser);
				// 查找根节点
                while ((type = childParser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty.
                }
				// 如果找不到根节点，抛出异常。
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(childParser.getPositionDescription() +
                            ": No start tag found!");
                }

				// 获取标签名
                final String childName = childParser.getName();
				// 实例化merge标签
                if (TAG_MERGE.equals(childName)) {
					// <merge>标签不支持android:theme，所以不需要其他处理
                    rInflate(childParser, parent, context, childAttrs, false);
                } else {
					// 创建View实例化对象
                    final View view = createViewFromTag(parent, childName,
                            context, childAttrs, hasThemeOverride);
                    final ViewGroup group = (ViewGroup) parent;

					// 获取View的id和可见性
                    final TypedArray a = context.obtainStyledAttributes(
                            attrs, R.styleable.Include);
                    final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
                    final int visibility = a.getInt(R.styleable.Include_visibility, -1);
                    a.recycle();

					// 尝试加载<include />标签中的布局参数，如果父View无法生成布局参数（比如include标签下没有宽高参数是要抛出运行时异常的）
					// 捕获运行时异常，然后用引用的Layout的attrs来创建布局参数
                    ViewGroup.LayoutParams params = null;
                    try {
						// 使用include标签的attrs来生成布局参数
                        params = group.generateLayoutParams(attrs);
                    } catch (RuntimeException e) {
                        // Ignore, just fail over to child attrs.
                    }
                    if (params == null) {
						// 如果include标签的attrs没有正确的生成布局参数，则使用Layout布局的attrs来生成布局参数
                        params = group.generateLayoutParams(childAttrs);
                    }
                    view.setLayoutParams(params);

                    // 实例化所有子View
                    rInflateChildren(childParser, view, childAttrs, true);

                    if (id != View.NO_ID) {
                        view.setId(id);
                    }

                    switch (visibility) {
                        case 0:
                            view.setVisibility(View.VISIBLE);
                            break;
                        case 1:
                            view.setVisibility(View.INVISIBLE);
                            break;
                        case 2:
                            view.setVisibility(View.GONE);
                            break;
                    }
					// 把布局实例化的View添加到父View上。
                    group.addView(view);
                }
            } finally {
                childParser.close();
            }
        }
    } else {
        throw new InflateException("<include /> can only be used inside of a ViewGroup");
    }

    LayoutInflater.consumeChildElements(parser);
}
```

## 6. LayoutInflaterCompat
LayoutInflater我们用的很多，一般都是用来把布局填充成View，它里面有两个方法：

```java
// api 1引入
setFactory(Factory factory)

// api 11 引入
setFactory2(Factory2 factory)
```
这里面需要传入接口的实现类，如下：

```java
public interface Factory {
    /**
     * Hook you can supply that is called when inflating from a LayoutInflater.
     * You can use this to customize the tag names available in your XML
     * layout files.
     *
     * <p>
     * Note that it is good practice to prefix these custom names with your
     * package (i.e., com.coolcompany.apps) to avoid conflicts with system
     * names.
     *
     * @param name Tag name to be inflated.
     * @param context The context the view is being created in.
     * @param attrs Inflation attributes as specified in XML file.
     *
     * @return View Newly created view. Return null for the default
     *         behavior.
     */
    public View onCreateView(String name, Context context, AttributeSet attrs);
}

public interface Factory2 extends Factory {
    /**
     * Version of {@link #onCreateView(String, Context, AttributeSet)}
     * that also supplies the parent that the view created view will be
     * placed in.
     *
     * @param parent The parent that the created view will be placed
     * in; <em>note that this may be null</em>.
     * @param name Tag name to be inflated.
     * @param context The context the view is being created in.
     * @param attrs Inflation attributes as specified in XML file.
     *
     * @return View Newly created view. Return null for the default
     *         behavior.
     */
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}
```

通过上述接口可以看出来，新的`setFactory2(Factory2 factory)`比老的`setFactory(Factory factory)`在构建View的时候多传入了一个Parent View。如果你想用`setFactory2(Factory factory)`需要实现带Parent View和不带Parent View的两个方法，比较复杂，所以v4包中的LayoutInflaterCompat就为我们提供了兼容性处理，先看用法：

```java    
LayoutInflater layoutInflater = getLayoutInflater();
LayoutInflaterCompat.setFactory(layoutInflater, new LayoutInflaterFactory() {

		@Override
		public View onCreateView(View parent, String name, Context context,
				AttributeSet attrs) {
			// name 是布局文件中View的名称，在这里可以坐很多操作，比如：
			// 给TextView设置字体。
			// 把TextView变成Button，如果需要的话。

			// 修改后的View，如果返回空则会调用LayoutInflater本身实例化View的方法，详情见createViewFromTag中try下面的逻辑。
			return null;
		}
	});
```

举个例子：现在我们用AS开发，一般默认是继承`AppCompatActivity`，在初始化的时候：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
	// 获取兼容包的委托类
    final AppCompatDelegate delegate = getDelegate();

	// 安装Factory
    delegate.installViewFactory();
    delegate.onCreate(savedInstanceState);
    if (delegate.applyDayNight() && mThemeId != 0) {
        // If DayNight has been applied, we need to re-apply the theme for
        // the changes to take effect. On API 23+, we should bypass
        // setTheme(), which will no-op if the theme ID is identical to the
        // current theme ID.
        if (Build.VERSION.SDK_INT >= 23) {
            onApplyThemeResource(getTheme(), mThemeId, false);
        } else {
            setTheme(mThemeId);
        }
    }
    super.onCreate(savedInstanceState);
}
```

在`AppCompatDelegate`的实现类`AppCompatDelegateImplV7`中，·`installViewFactory() `：

```java
@Override
public void installViewFactory() {
	// 使用LayoutInflaterCompat进行兼容性适配
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
		// 如果之前没有设置工厂，则自己实现接口然后传入，用来实现AppCompat特性，比如支持向低版本 tint着色等新特性。
		// 实际上它也是在工厂的实现类中用AppCompatXXX去替换XXX，比如用AppConpatTextView替换TextView等诸如此类。
        LayoutInflaterCompat.setFactory(layoutInflater, this);
    } else {
		// 如果已经设置了Factory并且不是当前类，则什么也不做，只打印日志。这样就会失去上述中tint等特性。
        if (!(LayoutInflaterCompat.getFactory(layoutInflater)
                instanceof AppCompatDelegateImplV7)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
}
```

上述方法实际上调用LayoutInflaterCompat的下面的方法：

```java
public static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
    IMPL.setFactory(inflater, factory);
}
```

IMPL是LayoutInflaterCompat中内部接口LayoutInflaterCompatImpl的实现类。这个接口有三个实现类：

- LayoutInflaterCompatImplBase
- LayoutInflaterCompatImplV11
- LayoutInflaterCompatImplV21

```java
static final LayoutInflaterCompatImpl IMPL;
static {
    final int version = Build.VERSION.SDK_INT;
    if (version >= 21) {
		// 5.0及其以上版本
        IMPL = new LayoutInflaterCompatImplV21();
    } else if (version >= 11) {
		// 3.0及其以上版本
        IMPL = new LayoutInflaterCompatImplV11();
    } else {
		// 低于3.0版本
        IMPL = new LayoutInflaterCompatImplBase();
    }
}
```

因此，调用LayoutInflaterCompat的setFactory方法，实际是调用对应版本的IMPL的setFactory方法。

1. 低于3.0版本

LayoutInflaterCompatImplBase中调用的是LayoutInflaterCompatBase的setFactory方法，

```java
static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
    inflater.setFactory(factory != null ? new FactoryWrapper(factory) : null);
}

static LayoutInflaterFactory getFactory(LayoutInflater inflater) {
    LayoutInflater.Factory factory = inflater.getFactory();
    if (factory instanceof FactoryWrapper) {
        return ((FactoryWrapper) factory).mDelegateFactory;
    }
    return null;
}
```

其中，`FactoryWrapper`是`LayoutInflaterCompatBase`的静态内部类：

```java
class LayoutInflaterCompatBase {

    static class FactoryWrapper implements LayoutInflater.Factory {

        final LayoutInflaterFactory mDelegateFactory;

        FactoryWrapper(LayoutInflaterFactory delegateFactory) {
            mDelegateFactory = delegateFactory;
        }

        @Override
        public View onCreateView(String name, Context context, AttributeSet attrs) {
            return mDelegateFactory.onCreateView(null, name, context, attrs);
        }

        public String toString() {
            return getClass().getName() + "{" + mDelegateFactory + "}";
        }
    }

    static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
        inflater.setFactory(factory != null ? new FactoryWrapper(factory) : null);
    }

    static LayoutInflaterFactory getFactory(LayoutInflater inflater) {
        LayoutInflater.Factory factory = inflater.getFactory();
        if (factory instanceof FactoryWrapper) {
            return ((FactoryWrapper) factory).mDelegateFactory;
        }
        return null;
    }

}
```

核心的方法就是`FactoryWrapper`的`onCreateView`,可以看到它调用的是带有Parent View参数的`onCreateView`方法，不过Parent View传的是null。

2.大于3.0小于5.0

LayoutInflaterCompatImplV11中调用的就是LayoutInflaterCompatHC的setFactory方法，LayoutInflaterCompatHC中：

```java
static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
	// 如果传入的Factory不为空则包装一下，否则传空
    final LayoutInflater.Factory2 factory2 = factory != null
            ? new FactoryWrapperHC(factory) : null;
	// 设置Factory2.
    inflater.setFactory2(factory2);
	// 获取当前Inflater的Factory。
    final LayoutInflater.Factory f = inflater.getFactory();
	// 如果属于Factory2则通过反射把mFactory赋值给mFactory2。
    if (f instanceof LayoutInflater.Factory2) {
        // The merged factory is now set to getFactory(), but not getFactory2() (pre-v21).
        // We will now try and force set the merged factory to mFactory2
        forceSetFactory2(inflater, (LayoutInflater.Factory2) f);
    } else {
		// 否则设置mFactory2为新创建的Factory2。
        // Else, we will force set the original wrapped Factory2
        forceSetFactory2(inflater, factory2);
    }
}

// 利用反射修改Factory2
/**
 * For APIs >= 11 && < 21, there was a framework bug that prevented a LayoutInflater's
 * Factory2 from being merged properly if set after a cloneInContext from a LayoutInflater
 * that already had a Factory2 registered. We work around that bug here. If we can't we
 * log an error.
 * 对于版本>- 11 并且 < 21,如果调用cloneInContext从LayoutInflater克隆一个LayoutInflater，在FrameWork层有一个bug阻止了LayoutInflater的Factory2的合并，因为已经有一个Factory2被注册了，所在在此通过反射的方式去修改Factory2。
 */
static void forceSetFactory2(LayoutInflater inflater, LayoutInflater.Factory2 factory) {
    if (!sCheckedField) {
        try {
            sLayoutInflaterFactory2Field = LayoutInflater.class.getDeclaredField("mFactory2");
            sLayoutInflaterFactory2Field.setAccessible(true);
        } catch (NoSuchFieldException e) {
            Log.e(TAG, "forceSetFactory2 Could not find field 'mFactory2' on class "
                    + LayoutInflater.class.getName()
                    + "; inflation may have unexpected results.", e);
        }
        sCheckedField = true;
    }
    if (sLayoutInflaterFactory2Field != null) {
        try {
            sLayoutInflaterFactory2Field.set(inflater, factory);
        } catch (IllegalAccessException e) {
            Log.e(TAG, "forceSetFactory2 could not set the Factory2 on LayoutInflater "
                    + inflater + "; inflation may have unexpected results.", e);
        }
    }
}
```

bug的具体产生原因见：[LayoutInflater在Api 21以下的setFactory2的bug是怎么产生的](https://github.com/peerless2012/AndroidBasis/blob/master/problem/Api21%E4%BB%A5%E4%B8%8B%E7%9A%84LayoutInflater%E4%B8%ADsetFactory2%E7%9A%84bug%E6%98%AF%E6%80%8E%E4%B9%88%E4%BA%A7%E7%94%9F%E7%9A%84.md)

3.大于5.0

```java
static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
    inflater.setFactory2(factory != null
            ? new LayoutInflaterCompatHC.FactoryWrapperHC(factory) : null);
}
```

这个其实没啥用，可以跟第二条合并的，但是最新的v4包没有改，但是在api >= 20的Android源码里面，其实已经这么做了：

```java
static final LayoutInflaterCompatImpl IMPL;
static {
    final int version = Build.VERSION.SDK_INT;
    if (version >= 11) {
        IMPL = new LayoutInflaterCompatImplV11();
    } else {
        IMPL = new LayoutInflaterCompatImplBase();
    }
}
```

## 7. 参考资料

[Android 源码解析 之 setContentView](http://blog.csdn.net/lmj623565791/article/details/41894125)

[context.getSystemService的简单说明](http://blog.csdn.net/chunqiuwei/article/details/50495686)

[Android LayoutInflater深度解析 给你带来全新的认识](http://blog.csdn.net/lmj623565791/article/details/38171465)
