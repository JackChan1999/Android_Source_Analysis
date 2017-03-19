# TabLayout 源码解析
## 1. 功能介绍
### 1.1 TabLayout
Tabs跟随Actionbar在Android 3.0进入大家的视线，是一个很经典的设计。它也是Material Design 规范中提及的`Component`之一。Tabs or Bottom navigation？相信不少Android开发者与产品都撕过，就连微信在其中也有过抉择。Google在Google+以及Google Photo中相继采用Bottom navigation的设计把剧情推到向高潮，一度轰动整个社区。Google继而在Material Design 规范加入了Bottom navigation，表明了态度，也给这起争论画上了圆满的句号。

在 support desgin lib 发布前，大家基本都采用[PagerSlidingTabStrip](https://github.com/astuetz/PagerSlidingTabStrip)来实现tab效果。其实`TabLayout`在实现上和`PagerSlidingTabStrip`十分相似，今天我们来分析`TabLayout`。


### 1.2 TabLayout使用

`TabLayout`使用比较简单。既可以单独使用，也可以与`ViewPager`配合使用。
#### 1.2.1 TabLayout单独使用
在java代码中添加Tabs
```java
TabLayout tabLayout = (TabLayout) findViewById(R.id.tabLayout);
tabLayout.addTab(tabLayout.newTab().setText("Tab 1"));
tabLayout.addTab(tabLayout.newTab().setText("Tab 2"));
tabLayout.addTab(tabLayout.newTab().setText("Tab 3"));
```
也可以在xml中添加Tabs
```xml
<android.support.design.widget.TabLayout
    android:layout_height="wrap_content"
    android:layout_width="match_parent">

    <android.support.design.widget.TabItem
        android:text="@string/tab_text"/>

    <android.support.design.widget.TabItem
        android:icon="@drawable/ic_android"/>

</android.support.design.widget.TabLayout>
```

#### 1.2.2 与ViewPager搭配使用
```java
// find view
TabLayout tabLayout = ...;
ViewPager viewPager = ...;

PagerAdapter adapter = new PagerAdapter(){
    // ...Override some methods
    // TabLayout调用这个方法获取Tab的title
    @Override
    public CharSequence getPageTitle(int position) {
        return "Tab 1";
    }
}
viewPager.setAdapter(adapter);
tabLayout.setupWithViewPager(viewPager);
```

## 2. 总体设计

- `TabLayout`继承`HorizontalScrollView`天生就是一个可以横向滚动的`ViewGroup`. 我们知道, `HorizontalScrollView`与`ScrollView`一样, 最多只能包含一个子View.

- `SlidingTabStrip`继承于`LinearLayout`，是`TabLayout`的内部类。它是`TabLayout`唯一的子View. 所有的`TabView`都是它的子View.

- `TabView`继承于`LinearLayout`,以`Tab`为数据源，来展示Tab的样式。最终用for循环被add进`SlidingTabStrip`.

- `Tab`是一个简单的View Model实体类，控制`TabView`的title, icon, custom layout id等属性。

- `TabItem`继承于View. 用于在layout xml中来描述Tab. 需要注意的是，它不会add到`SlidingTabStrip`中去。它的作用是从xml中获取到text，icon，custom layout id等属性。TabLayout inflate到`TabItem`并获取属性到装配到`Tab`中，最终add到`SlidingTabStrip`中的还是`TabView`.

- `OnTabSelectedListener`是TabLayout中的内部接口，用于监听`SlidingTabStrip`中子`TabView`选中状态的改变。

- `Mode`是TabLayout滚动模式的描述，一共有两种状态。`MODE_FIXED`不可滚动模式，以及`MODE_SCROLLABLE`可以滚动模式。

- `Gravity`是`TabView`在`SlidingTabStrip`中layout方式的描述。分为：GRAVITY_FILL，GRAVITY_CENTER.

## 3. 详细设计
### 3.1 类关系图
![TabLayout](https://github.com/Aspsine/AndroidSdkSourceAnalysis/blob/master/img/class-uml.png)

### 3.2 分析

#### 3.2.1 TabLayout子View唯一性保证
前面介绍`TabLayout`继承于`HorizontalScrollView`最多只能有1个子View. 但`TabLayout`可以在layout中添加多个子View节点. 这是怎么回事呢？
```xml
<android.support.design.widget.TabLayout
    android:layout_height="wrap_content"
    android:layout_width="match_parent">

    <android.support.design.widget.TabItem
        android:text="@string/tab_text"/>

    <android.support.design.widget.TabItem
        android:icon="@drawable/ic_android"/>

</android.support.design.widget.TabLayout>
```
看过`LayoutInflater`源码的同学可能会知道这个过程：先inflate到生成View对象，再调用`ViewGroup#addView(...)`系列方法把view添加到ViewGroup中。我们发现TabLayout的`addView(...)`系列方法，都删去super调用，且调用了共同的一个方法，`addViewInternal(View view)`。

```xml
private void addViewInternal(final View child) {
    if (child instanceof TabItem) {
        addTabFromItemView((TabItem) child);
    } else {
        throw new IllegalArgumentException("Only TabItem instances can be added to TabLayout");
    }
}
```
可见，若child非`TabItem`对象会抛出异常。所以xml中给TabLayout添加tab时，只能添加`TabItem`对象。若想添加其它View类型怎么办？TabItem有`android:customView`这个属性。我们继续来看。
```java
private void addTabFromItemView(@NonNull TabItem item) {
    final Tab tab = newTab();
    if (item.mText != null) {
        tab.setText(item.mText);
    }
    if (item.mIcon != null) {
        tab.setIcon(item.mIcon);
    }
    if (item.mCustomLayout != 0) {
        tab.setCustomView(item.mCustomLayout);
    }
    addTab(tab);
}

public Tab newTab() {
    Tab tab = sTabPool.acquire();
    if (tab == null) {
        tab = new Tab();
    }
    tab.mParent = this;
    tab.mView = createTabView(tab);
    return tab;
}

private TabView createTabView(@NonNull final Tab tab) {
    TabView tabView = mTabViewPool != null ? mTabViewPool.acquire() : null;
    if (tabView == null) {
        tabView = new TabView(getContext());
    }
    tabView.setTab(tab);
    tabView.setFocusable(true);
    tabView.setMinimumWidth(getTabMinWidth());
    return tabView;
}

```

这里调`newTab()`方法创建了一个tab对象，并且用对象池把创建的tab对象缓存起来。然后将`TabItem`对象的属性都赋值给tab对象。在`createTabView(Tab tab)`这个方法中，首先从`TabView`池中获取`TabView`对象，如果不存在，则实例化一个对象，并调用`tabView.setTab(tab)`方法来进行了数据绑定。 `addTab(...)`有三个重载方法，最终都会调用如下方法：
```java
public void addTab(@NonNull Tab tab, boolean setSelected) {
    if (tab.mParent != this) {
        throw new IllegalArgumentException("Tab belongs to a different TabLayout.");
    }

    addTabView(tab, setSelected);
    configureTab(tab, mTabs.size());
    if (setSelected) {
        tab.select();
    }
}

private void addTabView(Tab tab, int position, boolean setSelected) {
    final TabView tabView = tab.mView;
    mTabStrip.addView(tabView, position, createLayoutParamsForTabs());
    if (setSelected) {
        tabView.setSelected(true);
    }
}

private void configureTab(Tab tab, int position) {
    tab.setPosition(position);
    mTabs.add(position, tab);

    final int count = mTabs.size();
    for (int i = position + 1; i < count; i++) {
        mTabs.get(i).setPosition(i);
    }
}
```
在`addView(Tab, int, boolean)`方法中，把`TabView`对象add进了`SlidingTabStrip`这个`ViewGroup`中。实际上`SlidingTabStrip`的对象`mTabStrip`才是`TabLayout`的唯一子View.在`TabLayout`的构造方法中:
```java
public TabLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    // 禁用横向滑动条
    setHorizontalScrollBarEnabled(false);

    // new 一个'SlidingTabStrip'的实例，并作为唯一的子View add进'TabLayout'.
    mTabStrip = new SlidingTabStrip(context);
    super.addView(mTabStrip, 0, new HorizontalScrollView.LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.MATCH_PARENT));

    // 省略下面的无关代码...
｝
```
至此，我们就明白了`TabLayout`中子View的一致性是如何保证的。也明白了`TabView`其实才是亲生的，`TabItem`其实是后娘养的! 这些代码都很简单，不过我们可以从中学习到很多有用的思想。

至此，一个清晰的`View`层级图应该就出现在了各位同学的眼前。
![TabLayout Hierarchy](https://github.com/Aspsine/AndroidSdkSourceAnalysis/blob/master/img/hierarchy.png)


#### 3.2.2 与ViewPager搭配使用
有了上面的的基础，我们再来看看`TabLayout`是如何和它的好基友`ViewPager`搭配使用的。
```java
public void setupWithViewPager(@Nullable final ViewPager viewPager) {
    //...
    //为理解简单起见，删掉边角性干扰代码，主要来看核心逻辑

    mViewPager = viewPager;

    // Add our custom OnPageChangeListener to the ViewPager
    if (mPageChangeListener == null) {
        mPageChangeListener = new TabLayoutOnPageChangeListener(this);
    }
    mPageChangeListener.reset();
    viewPager.addOnPageChangeListener(mPageChangeListener);

    // Now we'll add a tab selected listener to set ViewPager's current item
    setOnTabSelectedListener(new ViewPagerOnTabSelectedListener(viewPager));

    // Now we'll populate ourselves from the pager adapter
    setPagerAdapter(adapter, true);
}

public void setOnTabSelectedListener(OnTabSelectedListener onTabSelectedListener) {
    mOnTabSelectedListener = onTabSelectedListener;
}

private void setPagerAdapter(@Nullable final PagerAdapter adapter, final boolean addObserver) {
    if (mPagerAdapter != null && mPagerAdapterObserver != null) {
        // If we already have a PagerAdapter, unregister our observer
        mPagerAdapter.unregisterDataSetObserver(mPagerAdapterObserver);
    }

    mPagerAdapter = adapter;

    if (addObserver && adapter != null) {
        // Register our observer on the new adapter
        if (mPagerAdapterObserver == null) {
            mPagerAdapterObserver = new PagerAdapterObserver();
        }
        adapter.registerDataSetObserver(mPagerAdapterObserver);
    }

    // Finally make sure we reflect the new adapter
    populateFromPagerAdapter();
}
```
这里的`TabLayoutOnPageChangeListener`实现了`ViewPager.OnPageChangeListener`. 首先调用`ViewPager`对象`addOnPageChangeListener(OnPageChangeListener)`来监听`ViewPager`的滑动以及当前也的选中。然后设置`ViewPagerOnTabSelectedListener`对象，保证ViewPager的页面和TabLayout的item的选中状态保持一致，以及滚动的协同性。这里的监听在3.2.3中详细讲解。

我们一般调用`viewPager.getAdapter().notifyDataSetChanged()`来进行ViewPager的刷新. 现在我们在ViewPager的adapter中注册一个监听器，监听`ViewPager`的刷新行为。目的是为了刷新`ViewPager`的同时也可以刷新TabLayout. 我们来看看`PagerAdapterObserver`这个监听器是如何刷新TabLayout的。
```java
private class PagerAdapterObserver extends DataSetObserver {
    @Override
    public void onChanged() {
        populateFromPagerAdapter();
    }

    @Override
    public void onInvalidated() {
        populateFromPagerAdapter();
    }
}

private void populateFromPagerAdapter() {
    removeAllTabs();

    if (mPagerAdapter != null) {
        final int adapterCount = mPagerAdapter.getCount();
        for (int i = 0; i < adapterCount; i++) {
            addTab(newTab().setText(mPagerAdapter.getPageTitle(i)), false);
        }

        // Make sure we reflect the currently set ViewPager item
        if (mViewPager != null && adapterCount > 0) {
            final int curItem = mViewPager.getCurrentItem();
            if (curItem != getSelectedTabPosition() && curItem < getTabCount()) {
                selectTab(getTabAt(curItem));
            }
        }
    } else {
        removeAllTabs();
    }
}

public void removeAllTabs() {
    // Remove all the views
    for (int i = mTabStrip.getChildCount() - 1; i >= 0; i--) {
        removeTabViewAt(i);
    }

    for (final Iterator<Tab> i = mTabs.iterator(); i.hasNext();) {
        final Tab tab = i.next();
        i.remove();
        tab.reset();
        sTabPool.release(tab);
    }

    mSelectedTab = null;
}
```
刷新方式很简单粗暴，从`SlidingTabStrip`对象中移除所有的`TabView`，继而从View Model`mTabs`中移除所有`Tab`对象。然后从adapter中获取tab信息，循环调用`addTab(Tab, boolean)`方法重新添加`TabView`。最后调用`ViewPager`对象的`getCurrentItem()`方法，获取当前位置，然后调用selectTab(int position)恢复`TabView`的选中状态（针对TabView的选中，3.2.4中有详细介绍)。

#### 3.2.3 ViewPager与TabLayout的Tab及indicaotr协同滚动
```java
public static class TabLayoutOnPageChangeListener implements ViewPager.OnPageChangeListener {
    private final WeakReference<TabLayout> mTabLayoutRef;
    private int mPreviousScrollState;
    private int mScrollState;

    public TabLayoutOnPageChangeListener(TabLayout tabLayout) {
        mTabLayoutRef = new WeakReference<>(tabLayout);
    }

    @Override
    public void onPageScrollStateChanged(int state) {
        mPreviousScrollState = mScrollState;
        mScrollState = state;
    }

    @Override
    public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
        final TabLayout tabLayout = mTabLayoutRef.get();
        if (tabLayout != null) {
            // Only update the text selection if we're not settling, or we are settling after
            // being dragged
            final boolean updateText = mScrollState != SCROLL_STATE_SETTLING ||
                    mPreviousScrollState == SCROLL_STATE_DRAGGING;
            // Update the indicator if we're not settling after being idle. This is caused
            // from a setCurrentItem() call and will be handled by an animation from
            // onPageSelected() instead.
            final boolean updateIndicator = !(mScrollState == SCROLL_STATE_SETTLING
                    && mPreviousScrollState == SCROLL_STATE_IDLE);
            tabLayout.setScrollPosition(position, positionOffset, updateText, updateIndicator);
        }
    }

    @Override
    public void onPageSelected(int position) {
        final TabLayout tabLayout = mTabLayoutRef.get();
        if (tabLayout != null && tabLayout.getSelectedTabPosition() != position) {
            // Select the tab, only updating the indicator if we're not being dragged/settled
            // (since onPageScrolled will handle that).
            final boolean updateIndicator = mScrollState == SCROLL_STATE_IDLE
                    || (mScrollState == SCROLL_STATE_SETTLING
                    && mPreviousScrollState == SCROLL_STATE_IDLE);
            tabLayout.selectTab(tabLayout.getTabAt(position), updateIndicator);
        }
    }

    private void reset() {
        mPreviousScrollState = mScrollState = SCROLL_STATE_IDLE;
    }
}

```
用过`ViewPager`的同学对`OnPageChangeListener`不会陌生，不多赘述。`TabLayoutOnPageChangeListener`实现了`OnPageChangeListener`, 在`onPageScrolled(...)`方法中做协同滚动处理。滚动的条件是：
```java
final boolean updateIndicator = !(mScrollState == SCROLL_STATE_SETTLING && mPreviousScrollState == SCROLL_STATE_IDLE);
```
调用`TabLayout的setScrollPosition(...)`方法来控制`TabLayout`中`TabView`和indocator的协同滚动。
```java
private void setScrollPosition(int position, float positionOffset, boolean updateSelectedText, boolean updateIndicatorPosition) {
    final int roundedPosition = Math.round(position + positionOffset);
    if (roundedPosition < 0 || roundedPosition >= mTabStrip.getChildCount()) {
        return;
    }

    // Set the indicator position, if enabled
    if (updateIndicatorPosition) {
        mTabStrip.setIndicatorPositionFromTabPosition(position, positionOffset);
    }

    // Now update the scroll position, canceling any running animation
    if (mScrollAnimator != null && mScrollAnimator.isRunning()) {
        mScrollAnimator.cancel();
    }
    scrollTo(calculateScrollXForTab(position, positionOffset), 0);

    // Update the 'selected state' view as we scroll, if enabled
    if (updateSelectedText) {
        setSelectedTabView(roundedPosition);
    }
}
```
##### 3.2.3.1 TabLayout的Indicator协同滚动
indicator的滚动由SlidingTabStrip来处理：

```java
// Set the indicator position, if enabled
if (updateIndicatorPosition) {
    mTabStrip.setIndicatorPositionFromTabPosition(position, positionOffset);
}
```
这里的`position`是当前选中的位置。`positionOffset`是: `距当前Tab滑动的距离`／`从当前tab滑动到下一个tab的总距离` 这样一个范围在［0，1］间的小数。

SlidingTabStrip#setIndicatorPositionFromTabPosition(int, float)
```java
void setIndicatorPositionFromTabPosition(int position, float positionOffset) {
    if (mIndicatorAnimator != null && mIndicatorAnimator.isRunning()) {
        mIndicatorAnimator.cancel();
    }

    mSelectedPosition = position;
    mSelectionOffset = positionOffset;
    updateIndicatorPosition();
}
```
SlidingTabStrip#updateIndicatorPosition()
```java
private void updateIndicatorPosition() {
    final View selectedTitle = getChildAt(mSelectedPosition);
    int left, right;

    if (selectedTitle != null && selectedTitle.getWidth() > 0) {
        left = selectedTitle.getLeft();
        right = selectedTitle.getRight();

        if (mSelectionOffset > 0f && mSelectedPosition < getChildCount() - 1) {
            // Draw the selection partway between the tabs
            View nextTitle = getChildAt(mSelectedPosition + 1);
            left = (int) (mSelectionOffset * nextTitle.getLeft() +
                    (1.0f - mSelectionOffset) * left);
            right = (int) (mSelectionOffset * nextTitle.getRight() +
                    (1.0f - mSelectionOffset) * right);
        }
    } else {
        left = right = -1;
    }

    setIndicatorPosition(left, right);
}
```
通过`getChildAt(mSelectedPosition)`, 获取到到`mSelectedPosition`处的TabView。若滑动的`mSelectionOffset>0f`且当前选中的位置`mSelectedPosition`不是最后一个TabView. 获取到下一个TabView，并计算出indicator的left和right。

SlidingTabStrip＃setIndicatorPosition(int, int)
```java
private void setIndicatorPosition(int left, int right) {
    if (left != mIndicatorLeft || right != mIndicatorRight) {
        // If the indicator's left/right has changed, invalidate
        mIndicatorLeft = left;
        mIndicatorRight = right;
        ViewCompat.postInvalidateOnAnimation(this);
    }
}
```
非常简单的代码，在调用`ViewCompat.postInvalidateOnAnimation(this)`重绘View之前，去掉一些重复绘制的帧。


```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);

    // Thick colored underline below the current selection
    if (mIndicatorLeft >= 0 && mIndicatorRight > mIndicatorLeft) {
        canvas.drawRect(mIndicatorLeft, getHeight() - mSelectedIndicatorHeight,
                mIndicatorRight, getHeight(), mSelectedIndicatorPaint);
    }
}
```
绘制逻辑很简单。调用`canvas.drawRect(float left, float top, float right, float bottom, Paint paint)`来绘制indicator.这里：
```java
left = mIndicatorLeft;
top = getHeight() - mSelectedIndicatorHeight;
right = mIndicatorRight;
bottom = getHeight();
```

##### 3.2.3.2 TabLayout的TabView协同滚动
我们回头来看 3.2.3中`setScrollPosition(...)`方法
```java
private void setScrollPosition(int position, float positionOffset, boolean updateSelectedText, boolean updateIndicatorPosition) {
    final int roundedPosition = Math.round(position + positionOffset);
    if (roundedPosition < 0 || roundedPosition >= mTabStrip.getChildCount()) {
        return;
    }

    // Set the indicator position, if enabled
    if (updateIndicatorPosition) {
        mTabStrip.setIndicatorPositionFromTabPosition(position, positionOffset);
    }

    // Now update the scroll position, canceling any running animation
    if (mScrollAnimator != null && mScrollAnimator.isRunning()) {
        mScrollAnimator.cancel();
    }
    scrollTo(calculateScrollXForTab(position, positionOffset), 0);

    // Update the 'selected state' view as we scroll, if enabled
    if (updateSelectedText) {
        setSelectedTabView(roundedPosition);
    }
}
```
在3.2.3.1中我们知道indicator的滚动是通过`mTabStrip.setIndicatorPositionFromTabPosition(position, positionOffset)`实现的。那TabView的滚动呢？我们知道`TabLayout`是继承`HorizonScrollView`天生就是一个可以横行滚动的`View`，所以，我们只需要调用`scrollTo(int x, int y)`方法就可以实现横向滚动。
```java
scrollTo(calculateScrollXForTab(position, positionOffset), 0);
```
这里x方向的偏移量调用`calculateScrollXForTab(position, positionOffset)`实时计算得出，y方向的偏移量为0。
```java
private int calculateScrollXForTab(int position, float positionOffset) {
    if (mMode == MODE_SCROLLABLE) {
        final View selectedChild = mTabStrip.getChildAt(position);
        final View nextChild = position + 1 < mTabStrip.getChildCount()
                ? mTabStrip.getChildAt(position + 1)
                : null;
        final int selectedWidth = selectedChild != null ? selectedChild.getWidth() : 0;
        final int nextWidth = nextChild != null ? nextChild.getWidth() : 0;

        return selectedChild.getLeft()
                + ((int) ((selectedWidth + nextWidth) * positionOffset * 0.5f))
                + (selectedChild.getWidth() / 2)
                - (getWidth() / 2);
    }
    return 0;
}
```
至此，我们就明白了`TabLayout`是如何随`ViewPager`的滚动而滚动的。

### 3.2.4 Tab选中状态

```java
private void setSelectedTabView(int position) {
    final int tabCount = mTabStrip.getChildCount();
    if (position < tabCount && !mTabStrip.getChildAt(position).isSelected()) {
        for (int i = 0; i < tabCount; i++) {
            final View child = mTabStrip.getChildAt(i);
            child.setSelected(i == position);
        }
    }
}
```
调用View的`setSelected(boolean)`方法。

## 4. 开源项目中的使用
开源项目中使用TabLayout的例子特别多, 这里给出我写的一个项目：
- [SwipeToLoadLayout](https://github.com/Aspsine/SwipeToLoadLayout)的demo
