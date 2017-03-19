### 1、简介
TextInputLayout 是android support design库里的一个控件，本文介绍的版本24.0.0-alpha2。该控件是一个容器，里面可以包含EditText用于输入内容(官方建议使用TextInputEditText)，当EditText获取焦点的时候，hint可以以动画的形式移动到EditText的上面位置，用于显示提示内容；edittext左下角可以显示错误的提示，例如密码输入错误的提示，右下角可以显示输入的内容长度，下面详细介绍该控件。

### 2、相关类：

相关的类我已经从design里面抽取出来了，具体可以看demo的内容：

![](class.png)

里面最主要的有两个类：

TextInputLayout：继承LinearLayout，用于盛放EditText，主要的容器。

CollapsingTextHelper：处理hint文字收起和展开动画。

### 3、结构和使用方法
演示如下：

![](demo.gif)

使用的时候是外层TextInputLayout包裹一个EditText如下：
```xml
    <demo.design.TextInputLayout
        android:id="@+id/inputlayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <demo.design.TextInputEditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="测试文字测速古" />
    </demo.design.TextInputLayout>
```
简单的结构图如下：

![](struct.png)

### 4、具体的源码分析
布局文件里TextInputLayout里包含了EditText，现在从加载EditText开始研究，TextInputLayout里面重写了addView，初始化的时候调用updateEditTextMargin设置上面需要预留的空间，用于hint做动画，再setEditText把EditText设置进去：
```java
    @Override
    public void addView(View child, int index, ViewGroup.LayoutParams params) {
        if (child instanceof EditText) {
            setEditText((EditText) child);//初始化EditText
            super.addView(child, 0, updateEditTextMargin(params));//
        } else {
            // Carry on adding the View...
            super.addView(child, index, params);
        }
    }
```
updateEditTextMargin里面所做的操作，由于上面显示的内容不是view，所以距离需要通过文字的高度计算，上面预留的位置为动画画笔的ascent高度，所以这里有点坑，这个高度没办法定制，而且外部没办法拿到，如果需要用到的话只能自己修改代码了。

```java
    private LayoutParams updateEditTextMargin(ViewGroup.LayoutParams lp) {
        // Create/update the LayoutParams so that we can add enough top margin
        // to the EditText so make room for the label
        LayoutParams llp = lp instanceof LayoutParams ? (LayoutParams) lp : new LayoutParams(lp);

        if (mHintEnabled) {
            if (mTmpPaint == null) {
                mTmpPaint = new Paint();
            }
            mTmpPaint.setTypeface(mCollapsingTextHelper.getCollapsedTypeface());
            mTmpPaint.setTextSize(mCollapsingTextHelper.getCollapsedTextSize());
            llp.topMargin = (int) -mTmpPaint.ascent();//这里是上面提示文字的区域的高度 llp.topMargin
        } else {
            llp.topMargin = 0;
        }

        return llp;
    }
```

在setEditText里面初始化EditText相关的东西，以及mCollapsingTextHelper动画相关的参数，字体，字体大小等等。而且设置了一个TextWatcher，用于监听字数的变化。
```java
private void setEditText(EditText editText) {
        // If we already have an EditText, throw an exception
        if (mEditText != null) {
            throw new IllegalArgumentException("We already have an EditText, can only have one");
        }

        if (!(editText instanceof TextInputEditText)) {//建议使用TextInputEditText
            Log.i(LOG_TAG, "EditText added is not a TextInputEditText. Please switch to using that"
                    + " class instead.");
        }

        mEditText = editText;

        // Use the EditText's typeface, and it's text size for our expanded text
        mCollapsingTextHelper.setTypefaces(mEditText.getTypeface());
        mCollapsingTextHelper.setExpandedTextSize(mEditText.getTextSize());

        final int editTextGravity = mEditText.getGravity();
        mCollapsingTextHelper.setCollapsedTextGravity(
                Gravity.TOP | (editTextGravity & GravityCompat.RELATIVE_HORIZONTAL_GRAVITY_MASK));
        mCollapsingTextHelper.setExpandedTextGravity(editTextGravity);

        // Add a TextWatcher so that we know when the text input has changed
        mEditText.addTextChangedListener(new TextWatcher() {
            @Override
            public void afterTextChanged(Editable s) {
                updateLabelState(true);
                if (mCounterEnabled) {
                    updateCounter(s.length());//监听内容数量
                }
            }

            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {}

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {}
        });

        // Use the EditText's hint colors if we don't have one set
        if (mDefaultTextColor == null) {//如果没有初始颜色，就是用hint的文字颜色
            mDefaultTextColor = mEditText.getHintTextColors();
        }

        // If we do not have a valid hint, try and retrieve it from the EditText, if enabled
        if (mHintEnabled && TextUtils.isEmpty(mHint)) {//初始化hint
            setHint(mEditText.getHint());
            // Clear the EditText's hint as we will display it ourselves
            mEditText.setHint(null);
        }

        if (mCounterView != null) {//更新计数器
            updateCounter(mEditText.getText().length());
        }

        if (mIndicatorArea != null) {//下面错误提示的view
            adjustIndicatorPadding();
        }

        // Update the label visibility with no animation
        updateLabelState(false);//更新上面提示
    }
```

addView结束后再看看onLayout里面做了些啥，之前说过动画是在CollapsingTextHelper里面完成的，这里onLayout初始化mCollapsingTextHelper里面做动画的两个状态区域的大小，一个是展开的，一个是收起的，因为做动画是一般都使用文字生成bitmap后再进行缩放的，为什么是一般呢，因为特殊情况是不使用bitmap的后面介绍。
```java
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);

        if (mHintEnabled && mEditText != null) {
            final int l = mEditText.getLeft() + mEditText.getCompoundPaddingLeft();
            final int r = mEditText.getRight() - mEditText.getCompoundPaddingRight();

            //初始化mCollapsingTextHelper里展开时的矩形区域ExpandedBounds
            mCollapsingTextHelper.setExpandedBounds(l,
                    mEditText.getTop() + mEditText.getCompoundPaddingTop(),
                    r, mEditText.getBottom() - mEditText.getCompoundPaddingBottom());

            // Set the collapsed bounds to be the the full height (minus padding) to match the
            // EditText's editable area
            //初始化mCollapsingTextHelper里收起时的矩形区域CollapsedBounds
            mCollapsingTextHelper.setCollapsedBounds(l, getPaddingTop(),
                    r, bottom - top - getPaddingBottom());

            mCollapsingTextHelper.recalculate();
        }
    }
```
onLayout后，基本上就完成了初始化的工作了，显示的时候都是比较正常的，就是一个EditTExt，来看看焦点改变的时候动画是怎么完成的，点击的时候hint文字会向上移动的动画，触发是在refreshDrawableState里面的updateLabelState：
```java
    @Override
    public void refreshDrawableState() {
        super.refreshDrawableState();
        // Drawable state has changed so see if we need to update the label
        updateLabelState(ViewCompat.isLaidOut(this));
    }
```
updateLabelState是更新上面提示文本的状态，animate参数是否有动画过渡，通过获取背景drawable的statelist判断当前的focus状态，再通过这个状态判断是否做动画。
```java
    private void updateLabelState(boolean animate) {
        final boolean hasText = mEditText != null && !TextUtils.isEmpty(mEditText.getText());
        final boolean isFocused = arrayContains(getDrawableState(), android.R.attr.state_focused);
        final boolean isErrorShowing = !TextUtils.isEmpty(getError());

        if (mDefaultTextColor != null) {
            mCollapsingTextHelper.setExpandedTextColor(mDefaultTextColor.getDefaultColor());
        }

        if (mCounterOverflowed && mCounterView != null) {
            mCollapsingTextHelper.setCollapsedTextColor(mCounterView.getCurrentTextColor());
        } else if (isFocused && mFocusedTextColor != null) {
            mCollapsingTextHelper.setCollapsedTextColor(mFocusedTextColor.getDefaultColor());
        } else if (mDefaultTextColor != null) {
            mCollapsingTextHelper.setCollapsedTextColor(mDefaultTextColor.getDefaultColor());
        }

        if (hasText || isFocused || isErrorShowing) {
            // We should be showing the label so do so if it isn't already
            collapseHint(animate);//收起动画
        } else {
            // We should not be showing the label so hide it
            expandHint(animate);//展开动画
        }
    }
```
动画实现的流程如下图：

![](anim.png)

collapseHint和expandHint基本上是一样的，只是最终的状态不一样，如果没有动画就直接调用mCollapsingTextHelper.setExpansionFraction()方法设置好最终状态；如果有动画，也是通过这个方法设置，只是在UpdateListener里面通过获取动画进行的百分比再设置对应的位置。

```java
    private void collapseHint(boolean animate) {
        if (mAnimator != null && mAnimator.isRunning()) {
            mAnimator.cancel();
        }
        if (animate && mHintAnimationEnabled) {
            animateToExpansionFraction(1f);
        } else {
            mCollapsingTextHelper.setExpansionFraction(1f);
        }
    }

    private void expandHint(boolean animate) {
        if (mAnimator != null && mAnimator.isRunning()) {
            mAnimator.cancel();
        }
        if (animate && mHintAnimationEnabled) {
            animateToExpansionFraction(0f);
        } else {
            mCollapsingTextHelper.setExpansionFraction(0f);
        }
    }

    private void animateToExpansionFraction(final float target) {
        if (mCollapsingTextHelper.getExpansionFraction() == target) {
            return;
        }
        if (mAnimator == null) {
            mAnimator = ViewUtils.createAnimator();
            mAnimator.setInterpolator(AnimationUtils.LINEAR_INTERPOLATOR);
            mAnimator.setDuration(ANIMATION_DURATION);
            mAnimator.setUpdateListener(new ValueAnimatorCompat.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimatorCompat animator) {
                    mCollapsingTextHelper.setExpansionFraction(animator.getAnimatedFloatValue());//动画过程调用相同的方法
                }
            });
        }
        mAnimator.setFloatValues(mCollapsingTextHelper.getExpansionFraction(), target);
        mAnimator.start();
    }
```
setExpansionFraction比较简单的,设置了当前动画的百分比。

```java
    /**
     * Set the value indicating the current scroll value. This decides how much of the
     * background will be displayed, as well as the title metrics/positioning.
     *
     * A value of {@code 0.0} indicates that the layout is fully expanded.
     * A value of {@code 1.0} indicates that the layout is fully collapsed.
     */
    void setExpansionFraction(float fraction) {
        fraction = MathUtils.constrain(fraction, 0f, 1f);//防止越界处理

        if (fraction != mExpandedFraction) {
            mExpandedFraction = fraction;
            calculateCurrentOffsets();
        }
    }
```

从上面setExpansionFraction一步步的调用过程： setExpansionFraction->calculateCurrentOffsets->calculateOffsets；
函数calculateOffsets通过传入的fraction计算当前画笔的textsize，color，ShadowLayer等参数。然后调用postInvalidateOnAnimation刷新界面。

```java
private void calculateOffsets(final float fraction) {
        interpolateBounds(fraction);
        mCurrentDrawX = lerp(mExpandedDrawX, mCollapsedDrawX, fraction,
                mPositionInterpolator);
        mCurrentDrawY = lerp(mExpandedDrawY, mCollapsedDrawY, fraction,
                mPositionInterpolator);

        setInterpolatedTextSize(lerp(mExpandedTextSize, mCollapsedTextSize,
                fraction, mTextSizeInterpolator));

        if (mCollapsedTextColor != mExpandedTextColor) {
            // If the collapsed and expanded text colors are different, blend them based on the
            // fraction
            mTextPaint.setColor(blendColors(mExpandedTextColor, mCollapsedTextColor, fraction));
        } else {
            mTextPaint.setColor(mCollapsedTextColor);
        }

        mTextPaint.setShadowLayer(
                lerp(mExpandedShadowRadius, mCollapsedShadowRadius, fraction, null),
                lerp(mExpandedShadowDx, mCollapsedShadowDx, fraction, null),
                lerp(mExpandedShadowDy, mCollapsedShadowDy, fraction, null),
                blendColors(mExpandedShadowColor, mCollapsedShadowColor, fraction));

        ViewCompat.postInvalidateOnAnimation(mView);//更新界面
    }
```

补充一下，上面计算颜色使用的是这个函数，可以用来做两个颜色之间的渐变，对A,R,G,B分别做处理，原生系统也有这个ArgbEvaluator，实现基本是一样的。
```java
    private static int blendColors(int color1, int color2, float ratio) {
        final float inverseRatio = 1f - ratio;
        float a = (Color.alpha(color1) * inverseRatio) + (Color.alpha(color2) * ratio);
        float r = (Color.red(color1) * inverseRatio) + (Color.red(color2) * ratio);
        float g = (Color.green(color1) * inverseRatio) + (Color.green(color2) * ratio);
        float b = (Color.blue(color1) * inverseRatio) + (Color.blue(color2) * ratio);
        return Color.argb((int) a, (int) r, (int) g, (int) b);
    }
```

上面刷新界面就是会触发draw绘制，动画的最终实现是在draw里面，绘制分两种情况 ：

1、hint文字收起和展开的文字大小差不多，即缩放比例为1，使用mTextPaint绘制文字即可。

2、缩放的比例不为1，则需要把hint文字生成bitmap再通过改变bitmap的区域大小进行缩放。：
```java
public void draw(Canvas canvas) {
        final int saveCount = canvas.save();

        if (mTextToDraw != null && mDrawTitle) {
            float x = mCurrentDrawX;
            float y = mCurrentDrawY;
            //是否使用Texture，其实是使用bitmap
            final boolean drawTexture = mUseTexture && mExpandedTitleTexture != null;

            final float ascent;
            final float descent;
            if (drawTexture) {
                ascent = mTextureAscent * mScale;
                descent = mTextureDescent * mScale;
            } else {
                ascent = mTextPaint.ascent() * mScale;
                descent = mTextPaint.descent() * mScale;
            }

            if (DEBUG_DRAW) {//debug打开后可以很明显地看到绘制的区域
                // Just a debug tool, which drawn a Magneta rect in the text bounds
                canvas.drawRect(mCurrentBounds.left, y + ascent, mCurrentBounds.right, y + descent,
                        DEBUG_DRAW_PAINT);
            }

            if (drawTexture) {
                y += ascent;
            }

            if (mScale != 1f) {
                canvas.scale(mScale, mScale, x, y);
            }

            if (drawTexture) {
                // If we should use a texture, draw it instead of text
                canvas.drawBitmap(mExpandedTitleTexture, x, y, mTexturePaint);
            } else {
                canvas.drawText(mTextToDraw, 0, mTextToDraw.length(), x, y, mTextPaint);
            }
        }

        canvas.restoreToCount(saveCount);
    }
```
到这里动画的部分就介绍完了。接着介绍错误提示框是怎么加载进去的,通过setErrorEnabled可以设置是否显示错误提示，但是如果直接调用setError(@Nullable final CharSequence error)，会默认调用setErrorEnabled(true)打开错误提示。当设置为true的时候会先new 一个TextView再把textView添加到底栏的LinearLayout里面。如果为false的话，会把ErrorView移除，移除。。所以如果true和false来回切，会导致布局跳动..这真是个大坑，视觉UI绝对不会允许这种跳跃：

```java
public void setErrorEnabled(boolean enabled) {
        if (mErrorEnabled != enabled) {
            if (mErrorView != null) {
                ViewCompat.animate(mErrorView).cancel();
            }

            if (enabled) {
                mErrorView = new TextView(getContext());//new一个textview显示错误内容
                try {
                    mErrorView.setTextAppearance(getContext(), mErrorTextAppearance);
                } catch (Exception e) {
                    // Probably caused by our theme not extending from Theme.Design*. Instead
                    // we manually set something appropriate
                    mErrorView.setTextAppearance(getContext(),
                            R.style.TextAppearance_AppCompat_Caption);
                    mErrorView.setTextColor(ContextCompat.getColor(
                            getContext(), R.color.design_textinput_error_color_light));
                }
                mErrorView.setVisibility(INVISIBLE);
                ViewCompat.setAccessibilityLiveRegion(mErrorView,
                        ViewCompat.ACCESSIBILITY_LIVE_REGION_POLITE);
                addIndicator(mErrorView, 0);//通过addIndicator添加ErrorView
            } else {
                mErrorShown = false;
                updateEditTextBackground();
                removeIndicator(mErrorView);//移除ErrorView
                mErrorView = null;
            }
            mErrorEnabled = enabled;
        }
```
addIndicator里面用添加view，index为位置，如果为错误提示View的话就加到前面，如果为计数器的话就加到后面。这里也是简单的LinearLayout加载View，不过如果设置了margin就会出现error文字偏移的问题，就像上面演示的图那种情况。所以这里可以改进，我这边的修改是TextView的layoutParams通过获取EditText的layoutParams来设置，让布局对齐。

```java
    private void addIndicator(TextView indicator, int index) {
        if (mIndicatorArea == null) {
            mIndicatorArea = new LinearLayout(getContext());
            mIndicatorArea.setOrientation(LinearLayout.HORIZONTAL);
            addView(mIndicatorArea, LayoutParams.MATCH_PARENT,
                    LayoutParams.WRAP_CONTENT);

            // Add a flexible spacer in the middle so that the left/right views stay pinned
            final Space spacer = new Space(getContext());
            final LayoutParams spacerLp = new LayoutParams(0, 0, 1f);
            //这里是添加下面错误提示view，在这里需要把edittext的layout_marginLeft或者layout_marginRight计算进去
            //希望之后的版本能够改进
            mIndicatorArea.addView(spacer, spacerLp);

            if (mEditText != null) {
                adjustIndicatorPadding();
            }
        }
        mIndicatorArea.setVisibility(View.VISIBLE);
        mIndicatorArea.addView(indicator, index);
        mIndicatorsAdded++;
    }
```

setCounterEnabled和上面setErrorEnable是一样的，这里就不再赘述了。至此，TextInputLayout大部分相关的东西基本都介绍完了。

### 4. 总结

TextInputLayout是一个比较简单的控件，不过动画的部分实现的比较复杂，该控价使用起来确实很方便，不过存在一些缺点，以下是我在使用时遇到的一些问题。

- 无法设置/获取上面文字的颜色，大小，间距等，下面的错误提示内容也是一样无法设置
- 无法设置动画时间，在代码里写死了
- 显示错误提示后高度会变化
- 显示文字个数，超出数量后闪退
- 如果包含的edittext有android:layout_marginLeft="10dp" ，这样布局有问题，这个修改在代码里有做备注
