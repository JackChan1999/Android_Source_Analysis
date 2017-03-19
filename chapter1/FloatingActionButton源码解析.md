## 背景

FloatingActionButton（下文以fab代替）是android support design组件库中提供的一个视图控件，是material design设计中fab的官方实现。

此控件的官方介绍如下：

> Floating action buttons are used for a promoted action. They are distinguished by a circled icon floating above the UI and have motion behaviors that include morphing, launching, and a transferring anchor point.

关于该控件的设计规范及使用场景请参考文档：
> http://www.google.com/design/spec/components/buttons-floating-action-button.html#

如果你还不了解design组件库，请参考官方博客:
> http://android-developers.blogspot.hk/2015/05/android-design-support-library.html


## 开始

源码版本:23.3.0

![类图](./01.png)

fab间接继承自`ImageView`（`ImageButton`是`ImageView`的子类），因而拥有`ImageView`的大部分特性。但是其内部还是做了很多定制，我们一一来看。

### 1. fab的自定义属性、背景着色相关

从构造器开始：

```java
public FloatingActionButton(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
		 //检查是否使用Theme.Appcompat主题
        ThemeUtils.checkAppCompatTheme(context);
        //拿到自定义属性并赋值
        TypedArray a = context.obtainStyledAttributes(attrs,
                R.styleable.FloatingActionButton, defStyleAttr,
                R.style.Widget_Design_FloatingActionButton);
       ...
        a.recycle();


        final int maxImageSize = (int) getResources().getDimension(R.dimen.design_fab_image_size);
        mImagePadding = (getSizeDimension() - maxImageSize) / 2;

		//背景着色
        getImpl().setBackgroundDrawable(mBackgroundTint, mBackgroundTintMode,
                mRippleColor, mBorderWidth);
      //绘制阴影
        getImpl().setElevation(elevation);
		...
    }
```
构造器中主要是拿到用户设置的自定义属性，比如着色、波纹颜色、大小等等,一共有以下几个属性可以定义。

```xml
<declare-styleable name="FloatingActionButton">
<attr name="backgroundTint"/>
<attr name="backgroundTintMode"/>
<attr format="color" name="rippleColor"/>
<attr name="fabSize">
	<enum name="normal" value="0"/>
	<enum name="mini" value="1"/>
</attr>
<attr name="elevation"/>
<attr format="dimension" name="pressedTranslationZ"/>
<attr format="dimension" name="borderWidth"/>
<attr format="boolean" name="useCompatPadding"/>
</declare-styleable>

```
属性的默认值定义如下：

```xml
 <style name="Widget.Design.FloatingActionButton" parent="android:Widget">

        <item name="android:background">@drawable/design_fab_background</item>
        <item name="backgroundTint">?attr/colorAccent</item>
        <item name="fabSize">normal</item>
        <item name="elevation">@dimen/design_fab_elevation</item>
        <item name="pressedTranslationZ">@dimen/design_fab_translation_z_pressed</item>
        <item name="rippleColor">?attr/colorControlHighlight</item>
        <item name="borderWidth">@dimen/design_fab_border_width</item>

    </style>
```

需要注意的是`android:background`属性，这里指定了background为`design_fab_background`,并且不允许改变:

```java
  @Override
    public void setBackgroundDrawable(Drawable background) {
        Log.i(LOG_TAG, "Setting a custom background is not supported.");
    }
```
那么我们来看下这个background长啥样：

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android"
        android:shape="oval">
    <solid android:color="@android:color/white" />
</shape>
```
很显然，fab的形状固定为圆形都是因为这个background。那么这里指定了背景色为白色，那是不是fab只能是白色背景呢？当然不是，还有我们牛逼的backgroundTint(即背景着色)，tint是android 5.x引进的一个新特性，可以动态地给drawable资源着色，其原理就是通过给控件设置colorFilter:

drawable.java

```java
public void setColorFilter(@ColorInt int color, @NonNull PorterDuff.Mode mode) {
        setColorFilter(new PorterDuffColorFilter(color, mode));
    }
```
默认的着色模式为SRC_IN(取交集、显示上层，故底层白色会被忽略)：

```java
    static final PorterDuff.Mode DEFAULT_TINT_MODE = PorterDuff.Mode.SRC_IN;

```


在fab构造的时候，会指定着色为`？attr/colorAccent`，即当前主题的`colorAccent`属性值。
然后执行如下代码，进行着色。

```java
 getImpl().setBackgroundDrawable(mBackgroundTint, mBackgroundTintMode,
                mRippleColor, mBorderWidth);
```

因为不同版本间的实现略有不同，所以这里会根据不同版本创建不同的`FloatingActionButtonImpl`实现类：

```java
private FloatingActionButtonImpl createImpl() {
        final int sdk = Build.VERSION.SDK_INT;
        if (sdk >= 21) {
            return new FloatingActionButtonLollipop(this, new ShadowDelegateImpl());
        } else if (sdk >= 14) {
            return new FloatingActionButtonIcs(this, new ShadowDelegateImpl());
        } else {
            return new FloatingActionButtonEclairMr1(this, new ShadowDelegateImpl());
        }
    }
```

以5.x为例，其setBackgroundDrawable实现代码如下:

先创建着色的背景drawable。

```java
 GradientDrawable createShapeDrawable() {
        GradientDrawable d = new GradientDrawable();
        d.setShape(GradientDrawable.OVAL);
        d.setColor(Color.WHITE);
        return d;
    }
```
再对此drawable设置tint：

```java
@Override
    void setBackgroundDrawable(ColorStateList backgroundTint,
            PorterDuff.Mode backgroundTintMode, int rippleColor, int borderWidth) {
        // Now we need to tint the shape background with the tint
        mShapeDrawable = DrawableCompat.wrap(createShapeDrawable());

        //着色，这里会其实就是设置了下colorFilter
        DrawableCompat.setTintList(mShapeDrawable, backgroundTint);

        if (backgroundTintMode != null) {
            DrawableCompat.setTintMode(mShapeDrawable, backgroundTintMode);
        }

        final Drawable rippleContent;
        if (borderWidth > 0) {
            mBorderDrawable = createBorderDrawable(borderWidth, backgroundTint);
            rippleContent = new LayerDrawable(new Drawable[]{mBorderDrawable, mShapeDrawable});
        } else {
            mBorderDrawable = null;
            rippleContent = mShapeDrawable;
        }

        mRippleDrawable = new RippleDrawable(ColorStateList.valueOf(rippleColor),
                rippleContent, null);

        mContentBackground = mRippleDrawable;

        mShadowViewDelegate.setBackgroundDrawable(mRippleDrawable);
    }
```
经过着色，fab就呈现出我们想要的颜色啦。

### 2. fab的大小

再来看fab的大小，fab有两种大小，一种是`NORMAL`，一种是`MINI`，实际大小分别是56dp和40dp，其定义可以在design库的values.xml中看到。

fab如何控制控件大小只有这两种规格呢(这样说不准确，事实上你可以通过设置fab的`layout_width`/`layout_height`指定为任意大小，但是我们最好按照MD规范来)?必然是通过复写`onMeasure`啦:

```java
  @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //我们希望的大小
        final int preferredSize = getSizeDimension();
		 //最终测量的大小
        final int w = resolveAdjustedSize(preferredSize, widthMeasureSpec);
        final int h = resolveAdjustedSize(preferredSize, heightMeasureSpec);

        //取小值，保证最后绘制的是圆形
        final int d = Math.min(w, h);

        // We add the shadow's padding to the measured dimension
        setMeasuredDimension(
                d + mShadowPadding.left + mShadowPadding.right,
                d + mShadowPadding.top + mShadowPadding.bottom);
    }
```

其中`getSizeDimension`方法计算出来的是我们期望的大小:

```java
final int getSizeDimension() {
        switch (mSize) {
            case SIZE_MINI:
                return getResources().getDimensionPixelSize(R.dimen.design_fab_size_mini);//40dp
            case SIZE_NORMAL:
            default:
                return getResources().getDimensionPixelSize(R.dimen.design_fab_size_normal);//56dp

       }
   }
```

但是最终的值还是得看我们设置的LayoutParams。关于控件测量相关内容不在此文介绍范围内，大家可以自行google。


### 3.fab的动画

fab还支持fab以动画的方式显现/隐藏，通常和AppBarLayout一起使用，可以通过`hide()`/`show()`两个方法控制。


那么动画是如何实现的呢:

```java
private void show(OnVisibilityChangedListener listener, boolean fromUser) {
        getImpl().show(wrapOnVisibilityChangedListener(listener), fromUser);
    }

private void hide(@Nullable OnVisibilityChangedListener listener, boolean fromUser) {
    getImpl().hide(wrapOnVisibilityChangedListener(listener), fromUser);
}
```

这里因为要兼容不同版本，所以具体实现也交给了不同的fab实现类。3.x之后很好办，直接使用属性动画，如果是3.x之前的话，那么只能使用传统的Animation了

以`hide()`为例，使用属性动画较为简单，直接使用`View#animate()`即可链式调用。

```java
@Override
    void hide(@Nullable final InternalVisibilityChangedListener listener, final boolean fromUser) {
        if (mIsHiding || mView.getVisibility() != View.VISIBLE) {
            // A hide animation is in progress, or we're already hidden. Skip the call
            if (listener != null) {
                listener.onHidden();
            }
            return;
        }

        if (!ViewCompat.isLaidOut(mView) || mView.isInEditMode()) {
            // If the view isn't laid out, or we're in the editor, don't run the animation
            mView.internalSetVisibility(View.GONE, fromUser);
            if (listener != null) {
                listener.onHidden();
            }
        } else {
            mView.animate().cancel();
            mView.animate()
                    .scaleX(0f)
                    .scaleY(0f)
                    .alpha(0f)
                    .setDuration(SHOW_HIDE_ANIM_DURATION)
                    .setInterpolator(AnimationUtils.FAST_OUT_LINEAR_IN_INTERPOLATOR)
                    .setListener(new AnimatorListenerAdapter() {
                        private boolean mCancelled;

                        @Override
                        public void onAnimationStart(Animator animation) {
                            mIsHiding = true;
                            mCancelled = false;
                            mView.internalSetVisibility(View.VISIBLE, fromUser);
                        }

                        @Override
                        public void onAnimationCancel(Animator animation) {
                            mIsHiding = false;
                            mCancelled = true;
                        }

                        @Override
                        public void onAnimationEnd(Animator animation) {
                            mIsHiding = false;
                            if (!mCancelled) {
                                mView.internalSetVisibility(View.GONE, fromUser);
                                if (listener != null) {
                                    listener.onHidden();
                                }
                            }
                        }
                    });
        }
    }
```

如果使用传统动画的话，则先在xml中定义好动画，然后构造`Animation`实例，启动动画。

```java
 @Override
    void hide(@Nullable final InternalVisibilityChangedListener listener, final boolean fromUser) {
        if (mIsHiding || mView.getVisibility() != View.VISIBLE) {
            // A hide animation is in progress, or we're already hidden. Skip the call
            if (listener != null) {
                listener.onHidden();
            }
            return;
        }

        Animation anim = android.view.animation.AnimationUtils.loadAnimation(
                mView.getContext(), R.anim.design_fab_out);
        anim.setInterpolator(AnimationUtils.FAST_OUT_LINEAR_IN_INTERPOLATOR);
        anim.setDuration(SHOW_HIDE_ANIM_DURATION);
        anim.setAnimationListener(new AnimationUtils.AnimationListenerAdapter() {
            @Override
            public void onAnimationStart(Animation animation) {
                mIsHiding = true;
            }

            @Override
            public void onAnimationEnd(Animation animation) {
                mIsHiding = false;
                mView.internalSetVisibility(View.GONE, fromUser);
                if (listener != null) {
                    listener.onHidden();
                }
            }
        });
        mView.startAnimation(anim);
    }
```


### 4. fab与CoordinatorLayout的交互

>这块内容因为与`CoordinatorLayout`/`CoordinatorLayout#Behavior`有很大关联，如果不熟悉，请先google相关资料。本文假设读者对这块内容已经有一定理解。


fab并不直接与`CoordinatorLayout`联系，而是通过`CoordinatorLayout#Behavior`作为桥梁。`CoordinatorLayout`类通过`CoordinatorLayout#Behavior`可以间接控制其直系子View的行为，能控制什么行为？View测量、布局、touch事件拦截、监听、NestedScroll等等。是不是很屌。

fab内部实现了`CoordinatorLayout#Behavior`抽象类。该抽象类有如下接口:

```java
public static abstract class Behavior<V extends View> {

		...

        public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
            return false;
        }

        public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
            return false;
        }

       ...
        /**
         * Determine whether the supplied child view has another specific sibling view as a
         * layout dependency.
         *
         * <p>This method will be called at least once in response to a layout request. If it
         * returns true for a given child and dependency view pair, the parent CoordinatorLayout
         * will:</p>
         * <ol>
         *     <li>Always lay out this child after the dependent child is laid out, regardless
         *     of child order.</li>
         *     <li>Call {@link #onDependentViewChanged} when the dependency view's layout or
         *     position changes.</li>
         * </ol>
         */
        public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) {
            return false;
        }

        /**
         * Respond to a change in a child's dependent view
         *
         * <p>This method is called whenever a dependent view changes in size or position outside
         * of the standard layout flow. A Behavior may use this method to appropriately update
         * the child view in response.</p>
         *
         * <p>A view's dependency is determined by
         * {@link #layoutDependsOn(CoordinatorLayout, android.view.View, android.view.View)} or
         * if {@code child} has set another view as it's anchor.</p>
         *
         * <p>Note that if a Behavior changes the layout of a child via this method, it should
         * also be able to reconstruct the correct position in
         * {@link #onLayoutChild(CoordinatorLayout, android.view.View, int) onLayoutChild}.
         * <code>onDependentViewChanged</code> will not be called during normal layout since
         * the layout of each child view will always happen in dependency order.</p>
         *
         * <p>If the Behavior changes the child view's size or position, it should return true.
         * The default implementation returns false.</p>
         *
         */
        public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
            return false;
        }

			...



        /**
         * Called when the parent CoordinatorLayout is about the lay out the given child view.
         *
         * <p>This method can be used to perform custom or modified layout of a child view
         * in place of the default child layout behavior. The Behavior's implementation can
         * delegate to the standard CoordinatorLayout measurement behavior by calling
         * {@link CoordinatorLayout#onLayoutChild(android.view.View, int)
         * parent.onLayoutChild}.</p>
         *
         * <p>If a Behavior implements
         * {@link #onDependentViewChanged(CoordinatorLayout, android.view.View, android.view.View)}
         * to change the position of a view in response to a dependent view changing, it
         * should also implement <code>onLayoutChild</code> in such a way that respects those
         * dependent views. <code>onLayoutChild</code> will always be called for a dependent view
         * <em>after</em> its dependency has been laid out.</p>
         *
         */
        public boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection) {
            return false;
        }

      ...

        public void onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,
                int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
            // Do nothing
        }


    }



```

看到这个抽象类，有两点需要注意:

 1. 此抽象类并无抽象方法，也即子类可选择任何想复写的方法进行复写。
 2. 此抽象类接受一个泛型。该泛型需要是View的子类。

fab实现此抽象类:

```java
public static class Behavior extends CoordinatorLayout.Behavior<FloatingActionButton> {}
```

有选择性地实现了三个方法:

```java
public boolean layoutDependsOn(CoordinatorLayout parent,
                FloatingActionButton child, View dependency);

public boolean onDependentViewChanged(CoordinatorLayout parent, FloatingActionButton child,
                View dependency);

 public boolean onLayoutChild(CoordinatorLayout parent, FloatingActionButton child,
                int layoutDirection);            

```

fab为啥要实现`Behavior`?主要是为了配合其他控件完成一些复杂的交互，比较经典的像这个:

[fab动画效果](http://material-design.storage.googleapis.com/publish/material_v_3/material_ext_publish/0B6Okdz75tqQsLWFucDNlYWEyeW8/components_snackbar_usage_fabdo_002.webm)

fab需要在`snackBar`弹出的时候自动向上平移，这就得知道SnackBar的状态了，实现`Behavior`让fab有机会监听到其他`CoordinatorLayout `子View的状态，并根据状态更新自己。

复写`layoutDependsOn `方法可以告诉`CoordinatorLayout `我对哪个View感兴趣，

这里当然是SnackBar了。（注意哦，SnackBar最终展现的是SnackbarLayout，SnackBar本身并不是View）

```java
private static final boolean SNACKBAR_BEHAVIOR_ENABLED = Build.VERSION.SDK_INT >= 11;

 @Override
        public boolean layoutDependsOn(CoordinatorLayout parent,
                FloatingActionButton child, View dependency) {
            // We're dependent on all SnackbarLayouts (if enabled)
            return SNACKBAR_BEHAVIOR_ENABLED && dependency instanceof Snackbar.SnackbarLayout;
        }
```

为什么API LEVEL要大于11呢？因为google偷懒想直接使用属性动画。

前面告诉了`CoordinatorLayout `fab对`SnackBar`比较感兴趣,那么当SnackBar状态改变的时候，`CoordinatorLayout`就会通过`onDependentViewChanged`回调通知fab:

fab就可以更新自己的UI拉（这里当然是平移喽）:

```java
@Override
        public boolean onDependentViewChanged(CoordinatorLayout parent, FloatingActionButton child,
                View dependency) {
            if (dependency instanceof Snackbar.SnackbarLayout) {
                updateFabTranslationForSnackbar(parent, child, dependency);
            } else if (dependency instanceof AppBarLayout) {
                // If we're depending on an AppBarLayout we will show/hide it automatically
                // if the FAB is anchored to the AppBarLayout
                updateFabVisibility(parent, (AppBarLayout) dependency, child);
            }
            return false;
        }
```

如果是SnackBar状态变化了，那么fab就会根据情况进行平移：

```java
private void updateFabTranslationForSnackbar(CoordinatorLayout parent,
                final FloatingActionButton fab, View snackbar) {
            final float targetTransY = getFabTranslationYForSnackbar(parent, fab);
            if (mFabTranslationY == targetTransY) {
                // We're already at (or currently animating to) the target value, return...
                return;
            }

            final float currentTransY = ViewCompat.getTranslationY(fab);

            // Make sure that any current animation is cancelled
            if (mFabTranslationYAnimator != null && mFabTranslationYAnimator.isRunning()) {
                mFabTranslationYAnimator.cancel();
            }

            if (fab.isShown()
                    && Math.abs(currentTransY - targetTransY) > (fab.getHeight() * 0.667f)) {
                // If the FAB will be travelling by more than 2/3 of it's height, let's animate
                // it instead
                if (mFabTranslationYAnimator == null) {
                    mFabTranslationYAnimator = ViewUtils.createAnimator();
                    mFabTranslationYAnimator.setInterpolator(
                            AnimationUtils.FAST_OUT_SLOW_IN_INTERPOLATOR);
                    mFabTranslationYAnimator.setUpdateListener(
                            new ValueAnimatorCompat.AnimatorUpdateListener() {
                                @Override
                                public void onAnimationUpdate(ValueAnimatorCompat animator) {
                                    ViewCompat.setTranslationY(fab,
                                            animator.getAnimatedFloatValue());
                                }
                            });
                }
                mFabTranslationYAnimator.setFloatValues(currentTransY, targetTransY);
                mFabTranslationYAnimator.start();
            } else {
                // Now update the translation Y
                ViewCompat.setTranslationY(fab, targetTransY);
            }

            mFabTranslationY = targetTransY;
        }
```
代码里的注释很多，我就不解释了。

前面说到AppBarLayout和fab一起使用可以完成另一个效果，即AppBarLayout伸缩时，fab也可以以动画的形式显现、隐藏，其实现如下：

```java
private boolean updateFabVisibility(CoordinatorLayout parent,
                AppBarLayout appBarLayout, FloatingActionButton child) {
            final CoordinatorLayout.LayoutParams lp =
                    (CoordinatorLayout.LayoutParams) child.getLayoutParams();
            //注意到我们必须为fab指定layout_anchor为appBarLayout                    
            if (lp.getAnchorId() != appBarLayout.getId()) {
                // The anchor ID doesn't match the dependency, so we won't automatically
                // show/hide the FAB
                return false;
            }

            if (child.getUserSetVisibility() != VISIBLE) {
                // The view isn't set to be visible so skip changing it's visibility
                return false;
            }

            if (mTmpRect == null) {
                mTmpRect = new Rect();
            }

            // First, let's get the visible rect of the dependency
            final Rect rect = mTmpRect;
            ViewGroupUtils.getDescendantRect(parent, appBarLayout, rect);

            if (rect.bottom <= appBarLayout.getMinimumHeightForVisibleOverlappingContent()) {
                // If the anchor's bottom is below the seam, we'll animate our FAB out
                child.hide(null, false);
            } else {
                // Else, we'll animate our FAB back in
                child.show(null, false);
            }
            return true;
        }
```


除此之外，`fab#Behavior`还实现了`onLayoutChild`,主要是为了根据AppBarLayout的当前状态来判断自己是否需要隐藏。

```java
 @Override
        public boolean onLayoutChild(CoordinatorLayout parent, FloatingActionButton child,
                int layoutDirection) {
            // First, lets make sure that the visibility of the FAB is consistent
            final List<View> dependencies = parent.getDependencies(child);
            for (int i = 0, count = dependencies.size(); i < count; i++) {
                final View dependency = dependencies.get(i);
                if (dependency instanceof AppBarLayout
                        && updateFabVisibility(parent, (AppBarLayout) dependency, child)) {
                    break;
                }
            }
            // Now let the CoordinatorLayout lay out the FAB
            parent.onLayoutChild(child, layoutDirection);
            // Now offset it if needed
            offsetIfNeeded(parent, child);
            return true;
        }
```

此方法会在`CoordinatorLayout`对孩子布局的时候进行调用(即`CoordinatorLayout#onLayout`)，`CoordinatorLayout `会检查所有的直系孩子，是否设置了Behavior，如果设置了，那么就执行其`onLayoutChild`方法:

CoordinatorLayout#onLayout

```java
 @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior behavior = lp.getBehavior();

            if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
                onLayoutChild(child, layoutDirection);
            }
        }
    }
```

如果该Behavior实现了OnLayoutChild，并且返回了true，那么将不会执行`CoordinatorLayout #onLayoutChild `,否则执行默认的布局方案。
最后一点，这里的Behavior如何生效的呢？通过注解：

```java
@CoordinatorLayout.DefaultBehavior(FloatingActionButton.Behavior.class)
public class FloatingActionButton extends VisibilityAwareImageButton {
```

`CoordinatorLayout `在解析孩子的`LayoutParams`时，会check有无注解：

```java
  LayoutParams getResolvedLayoutParams(View child) {
        final LayoutParams result = (LayoutParams) child.getLayoutParams();
        if (!result.mBehaviorResolved) {
            Class<?> childClass = child.getClass();
            DefaultBehavior defaultBehavior = null;
            while (childClass != null &&
                    (defaultBehavior = childClass.getAnnotation(DefaultBehavior.class)) == null) {
                childClass = childClass.getSuperclass();
            }
            if (defaultBehavior != null) {
                try {
                    result.setBehavior(defaultBehavior.value().newInstance());
                } catch (Exception e) {
                    Log.e(TAG, "Default behavior class " + defaultBehavior.value().getName() +
                            " could not be instantiated. Did you forget a default constructor?", e);
                }
            }
            result.mBehaviorResolved = true;
        }
        return result;
    }
```


至此`fab`解析完毕，谢谢观看！

如有疑惑，可以issue。

微博：[楚奕RX](http://weibo.com/u/2331178381?is_all=1)

## License

```
The MIT License (MIT)

Copyright (c) 2016 Rowandjj

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

```
