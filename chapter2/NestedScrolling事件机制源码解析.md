Android在发布 5.0（Lollipop）版本之后，Google为我们提供了嵌套滑动（NestedScrolling） 的特性，今天就由我带大家去看看嵌套滑动机制是怎样的原理？

首先，请随意瞄一瞄以下4个类：
[NestedScrollingChild](http://developer.android.com/reference/android/support/v4/view/NestedScrollingChild.html)
[NestedScrollingChildHelper](http://developer.android.com/reference/android/support/v4/view/NestedScrollingChildHelper.html)

[NestedScrollingParent](http://developer.android.com/reference/android/support/v4/view/NestedScrollingParent.html)
[NestedScrollingParentHelper](http://developer.android.com/reference/android/support/v4/view/NestedScrollingParentHelper.html)

有个大概印象就好，如果你一看就懂，那就不要浪费时间继续看下去了，啊哈哈哈！

从[NestedScrollingChild](http://developer.android.com/reference/android/support/v4/view/NestedScrollingChild.html)说起，它是个接口，定义如下：

```
public interface NestedScrollingChild {  
    /** 
     * 设置嵌套滑动是否能用
     * 
     *  @param enabled true to enable nested scrolling, false to disable
     */  
    public void setNestedScrollingEnabled(boolean enabled);  

    /** 
     * 判断嵌套滑动是否可用 
     * 
     * @return true if nested scrolling is enabled
     */  
    public boolean isNestedScrollingEnabled();  

    /** 
     * 开始嵌套滑动
     * 
     * @param axes 表示方向轴，有横向和竖向
     */  
    public boolean startNestedScroll(int axes);  

    /** 
     * 停止嵌套滑动 
     */  
    public void stopNestedScroll();  

    /** 
     * 判断是否有父View 支持嵌套滑动 
     * @return whether this view has a nested scrolling parent
     */  
    public boolean hasNestedScrollingParent();  

    /** 
     * 在子View的onInterceptTouchEvent或者onTouch中，调用该方法通知父View滑动的距离
     *
     * @param dx  x轴上滑动的距离
     * @param dy  y轴上滑动的距离
     * @param consumed 父view消费掉的scroll长度
     * @param offsetInWindow   子View的窗体偏移量
     * @return 支持的嵌套的父View 是否处理了 滑动事件 
     */  
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);  

    /** 
     * 子view处理scroll后调用
     *
     * @param dxConsumed x轴上被消费的距离（横向） 
     * @param dyConsumed y轴上被消费的距离（竖向）
     * @param dxUnconsumed x轴上未被消费的距离 
     * @param dyUnconsumed y轴上未被消费的距离 
     * @param offsetInWindow 子View的窗体偏移量
     * @return  true if the event was dispatched, false if it could not be dispatched.
     */  
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,  
          int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);  



    /** 
     * 滑行时调用 
     *
     * @param velocityX x 轴上的滑动速率
     * @param velocityY y 轴上的滑动速率
     * @param consumed 是否被消费 
     * @return  true if the nested scrolling parent consumed or otherwise reacted to the fling
     */  
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);  

    /** 
     * 进行滑行前调用
     *
     * @param velocityX x 轴上的滑动速率
     * @param velocityY y 轴上的滑动速率 
     * @return true if a nested scrolling parent consumed the fling
     */  
    public boolean dispatchNestedPreFling(float velocityX, float velocityY);  
}
```

以上方法数不多，请详细看看。
那它的作用是什么呢？让我们想想一种场景：CoordinatorLayout里嵌套着RecyclerView和Toolbar,我们上下滑动RecyclerView的时候，Toolbar会随之显现隐藏,这是典型的嵌套滑动机制情景。这里，RecyclerView作为嵌套的子View,我们猜测，它一定实现了NestedScrollingChild 接口（去看看它的定义就知道了，猜你个头）

```
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild {
..................................................................................
}
```

所以RecyclerView 实现了NestedScrollingChild 接口里的方法，我们在跟进去看看各个方法是怎么实现的？

```
    @Override
    public void setNestedScrollingEnabled(boolean enabled) {
        getScrollingChildHelper().setNestedScrollingEnabled(enabled);
    }

    @Override
    public boolean isNestedScrollingEnabled() {
        return getScrollingChildHelper().isNestedScrollingEnabled();
    }

    @Override
    public boolean startNestedScroll(int axes) {
        return getScrollingChildHelper().startNestedScroll(axes);
    }

    @Override
    public void stopNestedScroll() {
        getScrollingChildHelper().stopNestedScroll();
    }

    @Override
    public boolean hasNestedScrollingParent() {
        return getScrollingChildHelper().hasNestedScrollingParent();
    }

    @Override
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed, int[] offsetInWindow) {
        return getScrollingChildHelper().dispatchNestedScroll(dxConsumed, dyConsumed,
                dxUnconsumed, dyUnconsumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        return getScrollingChildHelper().dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
        return getScrollingChildHelper().dispatchNestedFling(velocityX, velocityY, consumed);
    }

    @Override
    public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
        return getScrollingChildHelper().dispatchNestedPreFling(velocityX, velocityY);
    }
```

从上面的代码可以看出，全部都交给`getScrollingChildHelper()`这个方法的返回对象处理了，看看这个方法是怎么实现的。

```
   private NestedScrollingChildHelper getScrollingChildHelper() {
        if (mScrollingChildHelper == null) {
            mScrollingChildHelper = new NestedScrollingChildHelper(this);
        }
        return mScrollingChildHelper;
    }
```

对，NestedScrollingChild 接口的方法都交给NestedScrollingChildHelper这个代理对象处理了。现在我们继续深入，随意挑个，分析下NestedScrollingChildHelper中开始嵌套滑动`startNestedScroll(int axes)`方法是怎么实现的。

*NestedScrollingChildHelper#startNestedScroll*

```
    public boolean startNestedScroll(int axes) {
        if (hasNestedScrollingParent()) {
             return true;
        }
        if (isNestedScrollingEnabled()) {//判断是否可以滑动
            ViewParent p = mView.getParent();
            View child = mView;
            while (p != null) {
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {//回调了父View的onStartNestedScroll方法
                    mNestedScrollingParent = p;
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
```

以上方法主要做了：

1. 判断是否有嵌套滑动的父View,返回值 true 表示找到了嵌套滑动的父View和同意一起处理 Scroll 事件。
2. 用While的方式寻找最近嵌套滑动的父View ，如果找到调用父view的`onNestedScrollAccepted`.

从这里至少可以得出 子view在调用某个方法都会回调嵌套父view相应的方法，比如子view开始了`startNestedScroll`，如果嵌套父view存在，就会回调父view的`onStartNestedScroll`、`onNestedScrollAccepted`方法。

有兴趣的朋友在去看看
*NestedScrollingChildHelper#dispatchNestedPreScroll*
*NestedScrollingChildHelper#dispatchNestedScroll*
*NestedScrollingChildHelper#stopNestedScroll*
的实现。

接下来，在来看看嵌套滑动父view *NestedScrollingParent*，定义如下

```
public interface NestedScrollingParent {

    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);

    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);

    public void onStopNestedScroll(View target);

    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed);

    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);

    public boolean onNestedFling(View target, float velocityX, 
                                                      float velocityY,boolean consumed);

    public boolean onNestedPreFling(View target, float velocityX, float velocityY);

    public int getNestedScrollAxes();
}
```

你会发现，其实和子view差不多的方法，大致一一对应关系，而且它的具体实现也交给了*NestedScrollingParentHelper*这个代理类，这和我们上文的方式是一样的，就不再重复了。

##### 捋一捋流程

1、当 NestedScrollingChild(下文用Child代替) 要开始滑动的时候会调用 `onStartNestedScroll` ,然后交给代理类NestedScrollingChildHelper（下文ChildHelper代替）的`onStartNestedScroll`请求给最近的NestedScrollingParent(下文Parent代替).

2、当ChildHelper的`onStartNestedScroll`方法 返回 true 表示同意一起处理 Scroll 事件的时候时候,ChildHelper会通知Parent回调`onNestedScrollAccepted` 做一些准备动作

3、当Child 要开始滑动的时候,会先发送`onNestedPreScroll`,交给ChildHelper的`onNestedPreScroll` 请求给Parent ,告诉它我现在要滑动多少距离,你觉得行不行,这时候Parent 根据实际情况告诉Child 现在只允许你滑动多少距离.然后 ChildHelper根据 onNestedPreScroll 中回调回来的信息对滑动距离做相对应的调整.

4、在滑动的过程中 Child 会发送`onNestedScroll`通知ChildeHelpaer的`onNestedScroll`告知Parent 当前 Child 的滑动情况.

5、当要进行滑行的时候,会先发送onNestedFling 请求给Parent,告诉它 我现在要滑行了,你说行不行, 这时候Parent会根据情况告诉 Child 你是否可以滑行.

6、Child 通过`onNestedFling` 返回的 Boolean 值来觉得是否进行滑行.如果要滑行的话,会在滑行的时候发送onNestedFling 通知告知 Parent 滑行情况.

7、当滑动事件结束就会child发送`onStopNestedScroll`通知 Parent 去做相关操作.

如果你此刻还是看得模模糊糊，没错，是我的责任，啊哈哈哈。要不我给个例子实践下，我们知道5.0之前的ListView是没有实现NestedScrollingChild这个接口的，如果要实现CoordinatorLayout里嵌套着ListView和Toolbar,我们上下滑动ListView的时候，Toolbar会随之显现隐藏,就必须重写ListView.给出我实现的代码吧！

```
package com.cjj.nestedlistview;

import android.content.Context;
import android.support.v4.view.MotionEventCompat;
import android.support.v4.view.NestedScrollingChild;
import android.support.v4.view.NestedScrollingChildHelper;
import android.support.v4.view.ViewCompat;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.widget.ListView;

public class NestedListView extends ListView implements NestedScrollingChild {

    private NestedScrollingChildHelper mChildHelper;
    private int mLastY;
    private final int[] mScrollOffset = new int[2];
    private final int[] mScrollConsumed = new int[2];
    private int mNestedOffsetY;

    public NestedListView(Context context) {
        super(context);
        init();
    }

    public NestedListView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public NestedListView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        mChildHelper = new NestedScrollingChildHelper(this);
        setNestedScrollingEnabled(true);
    }


    @Override
    public void setNestedScrollingEnabled(boolean enabled) {
        mChildHelper.setNestedScrollingEnabled(enabled);
    }

    @Override
    public boolean startNestedScroll(int axes) {
        return mChildHelper.startNestedScroll(axes);
    }

    @Override
    public void stopNestedScroll() {
        mChildHelper.stopNestedScroll();
    }

    @Override
    public boolean hasNestedScrollingParent() {
        return mChildHelper.hasNestedScrollingParent();
    }

    @Override
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
        return mChildHelper.dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        return mChildHelper.dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
        return mChildHelper.dispatchNestedFling(velocityX, velocityY, consumed);
    }

    @Override
    public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
        return mChildHelper.dispatchNestedPreFling(velocityX, velocityY);
    }


    @Override
    public boolean onTouchEvent(MotionEvent event) {
        final int action = MotionEventCompat.getActionMasked(event);

        int y = (int) event.getY();
        event.offsetLocation(0, mNestedOffsetY);
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mLastY = y;
                mNestedOffsetY = 0;
                break;
            case MotionEvent.ACTION_MOVE:

                int dy = mLastY - y;
                int oldY = getScrollY();

                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
                if (dispatchNestedPreScroll(0, dy, mScrollConsumed, mScrollOffset)) {
                    dy -= mScrollConsumed[1];
                    event.offsetLocation(0, -mScrollOffset[1]);
                    mNestedOffsetY += mScrollOffset[1];
                }
                mLastY = y - mScrollOffset[1];
                if (dy < 0) {
                    int newScrollY = Math.max(0, oldY + dy);
                    dy -= newScrollY - oldY;
                    if (dispatchNestedScroll(0, newScrollY - dy, 0, dy, mScrollOffset)) {
                        event.offsetLocation(0, mScrollOffset[1]);
                        mNestedOffsetY += mScrollOffset[1];
                        mLastY -= mScrollOffset[1];
                    }
                }
                stopNestedScroll();
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:

                stopNestedScroll();

                break;
        }
        return super.onTouchEvent(event);
    }
}
```

效果：

![img](http://upload-images.jianshu.io/upload_images/1204365-1d77b38824c07167.gif?imageMogr2/auto-orient/strip)